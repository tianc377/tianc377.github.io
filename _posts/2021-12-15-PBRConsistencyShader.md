---
layout: post
title: "UE4 Consistency Shader in SubstancePainter/Maya"
date:   2021-12-15 12:00
category: shaderposts
icon: scale
keywords: tag1, tag2
image: pbr-consistency.jpg
preview: 1
youtubeId: whFFbZq_geQ
---


1. [The Need](#the-need)
2. [Substance Painter Unreal-like PBR Shader](#substance-painter-unreal-like-pbr-shader)
    - [SP Shader API](#sp-shader-api)
    - [SP PBR Metal Rough](#sp-pbr-metal-rough)
    - [The differences between SP and UE4](#the-differences-between-sp-and-ue4)
    - [My SP Consistency Shader](#my-sp-consistency-shader)
        - [fetch basic parameters](#fetch-basic-parameters)
        - [Lighting.Diffuse, Lighting.Specular](#lightingdiffuse-lightingspecular)
        - [SpecularIBL](#specularibl)
        - [SkyDiffuseLighting](#skydiffuselighting)
        - [ToneMapping](#tonemapping)

3. [Maya Unreal-like PBR Shader](#maya-unreal-like-pbr-shader)
4. [NPR Consistency Shader](#npr-consistency-shader)
5. [Marmoset Unreal-like PBR Shader](#marmoset-unreal-like-pbr-shader)


## The Need
It is important for artists to work in a consistent environment no matter in the engine or DCC. Especially when the outsourced team usually doesn’t have the authority to get the project’s engine, they couldn’t test and check the final visual result in the engine after they finished the art assets. Therefore consistency between engine and DCC rendering is essential. Although DCC software usually comes with its own PBR default shader, the different engine has different render pipeline and the different project has its own shading strategy besides shading is different on mobile and PC, a custom consistency shader is needed. <br />
To do that, it needs a well understanding of the engine shading pipeline, and I've already made a step by step disassembly of UE4 mobile basepass shader [*UE4 Mobile PBR Pipeline*](/shaderposts/2021/UE4MobilePBRPipeline.html). <br />


## Substance Painter Unreal-like PBR Shader
Below is the output comparison of UE4 Mobile PBR and my SP PBR.  
`Roughness 0 to 1. Metal top, nonmetal bottom. UE4` <br />
![ue4-metal-nonmetal](/post-img/shaderposts/pbr-consistency-shader/ue4-metal-nonmetal.png){: width="90%" }<br />
`Roughness 0 to 1. Metal top, nonmetal bottom. Substance Painter` <br />
![shading-sphere-sp](/post-img/shaderposts/pbr-consistency-shader/shading-sphere-sp.jpg){: width="90%s" }<br />

Applied on the asset of PUBG Mobile project, in UE4 and SP. <br />
![ue4-sp](/post-img/shaderposts/pbr-consistency-shader/ue4-sp.jpg){: width="80%" }<br />
### SP Shader API
Firstly, go to Substance Painter website to see their [Shader API](https://substance3d.adobe.com/documentation/spdoc/shader-api-89686018.html).<br />
As well as the [PBR Metal Rough](https://substance3d.adobe.com/documentation/spdoc/pbr-metal-rough-shader-api-188976094.html) shader that usually is taken as the default PBR shader in SP. ![PBR-metal-rough](/post-img/shaderposts/pbr-consistency-shader/PBR-metal-rough.png){: width="40%" }<br />
Generally, SP makes all their lib and functions in the [Shader API](https://substance3d.adobe.com/documentation/spdoc/shader-api-89686018.html), and there is no any actual .glsl files in directory anymore after 2018 version. <br />
Parameters are set in comment format, which you can find in [Paramters Page](https://substance3d.adobe.com/documentation/spdoc/parameters-shader-api-188976086.html). <br />
All the functions are in [Libraries Page](https://substance3d.adobe.com/documentation/spdoc/libraries-shader-api-188976070.html), where you can find BRDF functions, basic PBR parameters functions, samplers, etc.<br />

### SP PBR Metal Rough

Let's see SP's default PBR shader first:
{% highlight glsl %}
void shade(V2F inputs)
{
  // Apply parallax occlusion mapping if possible
  vec3 viewTS = worldSpaceToTangentSpace(getEyeVec(inputs.position), inputs);
  applyParallaxOffset(inputs, viewTS);

  // Fetch material parameters, and conversion to the specular/roughness model
  float roughness = getRoughness(roughness_tex, inputs.sparse_coord);
  vec3 baseColor = getBaseColor(basecolor_tex, inputs.sparse_coord);
  float metallic = getMetallic(metallic_tex, inputs.sparse_coord);
  float specularLevel = getSpecularLevel(specularlevel_tex, inputs.sparse_coord);
  vec3 diffColor = generateDiffuseColor(baseColor, metallic);
  vec3 specColor = generateSpecularColor(specularLevel, baseColor, metallic);
  // Get detail (ambient occlusion) and global (shadow) occlusion factors
  float occlusion = getAO(inputs.sparse_coord) * getShadowFactor();
  float specOcclusion = specularOcclusionCorrection(occlusion, metallic, roughness);

  LocalVectors vectors = computeLocalFrame(inputs);

  // Feed parameters for a physically based BRDF integration
  emissiveColorOutput(pbrComputeEmissive(emissive_tex, inputs.sparse_coord));
  albedoOutput(diffColor);
  diffuseShadingOutput(occlusion * envIrradiance(vectors.normal));
  specularShadingOutput(specOcclusion * pbrComputeSpecular(vectors, specColor, roughness));
  sssCoefficientsOutput(getSSSCoefficients(inputs.sparse_coord));
}
{% endhighlight %}

In this shader, it fetch material parameters from the texture input, through sampling and some manipulations*, then feed to 5 ShadingOutput. The most basic rendering equation for computing the fragment color is: `emissiveColor + albedo * diffuseShading + specularShading`.<br />
Well, I want to rewrite the shader so I'll just put my final result into the `diffuseShadingOutput` and ignore others. <br />
*You can find what the `generateDiffuseColor` `generateSpecularColor` `getRoughness` `getBaseColor`, etc. does in this page:[Lib Sampler](https://substance3d.adobe.com/documentation/spdoc/lib-sampler-shader-api-188976081.html).  <br />


### The differences between SP and UE4

`The asset in the standard Lookdev scene in UE4 Mobile ES3.1`<br />
![ue4-mat](/post-img/shaderposts/pbr-consistency-shader/ue4-mat.jpg){: width="50%" }<br />
`The asset in SP with default PBR shader`<br />
![sp-lit](/post-img/shaderposts/pbr-consistency-shader/sp-lit.jpg){: width="50%" }<br />
Very different, huh? So read the PBR-metal-rough shader above, lighting of SP is from the image-based lighting by **envIrradiance()** function, which return the irradiance for a given direction, the computation is based on environment's spherical harmonics projection. As well as a **pbrComputeSpecular()** works in the `specularShadingOutput`, which compute the microfacets specular reflection to the viewer's eye.
If I remove those two functions, the output is like: <br />
![no-env-spec](/post-img/shaderposts/pbr-consistency-shader/no-env-spec.jpg){: width="50%" }<br />
![pbrComputeSpec](/post-img/shaderposts/pbr-consistency-shader/pbrComputeSpec.jpg){: width="50%" }
![envIrradiance](/post-img/shaderposts/pbr-consistency-shader/envIrradiance.jpg){: width="50%" }

While for UE4, it has at least a directional light, SH SkyLight, and image-based lighting (IBL), as well as some approximate calculation I talked in the last post [*UE4 Mobile PBR Pipeline*](/shaderposts/2021/UE4MobilePBRPipeline.html). That's quite different.  


### My SP Consistency Shader

#### fetch basic parameters
Starting at `void shade(V2F inputs)`, fetch basic parameters from the meterial textures:
{% highlight glsl %}
  vec3 baseColor = getBaseColor(basecolor_tex, sparse_coord);
  float roughness = getRoughness(roughness_tex, sparse_coord);
  float metallic = getMetallic(metallic_tex, sparse_coord);
  float occlusion = getAO(inputs.sparse_coord);
{% endhighlight %}


{% highlight glsl %}
  vec3 N = vectors.normal;// computeWSBaseNormal(inputs.tex_coord, inputs.tangent, inputs.bitangent, inputs.normal);
  vec3 L = (getLightDir(inputs.position));
  vec3 V = getEyeVec(inputs.position);
  vec3 H = normalize(L + V);
  vec3 reflectDir = normalize(-reflect(V, N));
  float NoL = saturate(dot(N, L));
  float NoH = saturate(dot(N, H));
  float NoV = saturate(dot(N, V));
{% endhighlight %}

I have all basic parameters I need now, let's follow the pipeline of UE4 mobile basepass pixel shader. First I need ShadingModelContext.**SpecPreEnvBrdf**, ShadingModelContext.**SpecularColor**, ShadingModelContext.**DiffuseColor**. In SP glsl, they will be:
{% highlight glsl %}
  float dielectricSpecular = 0.08 * 0.5; //0.5 is the default GBuffer.Specular
  vec3 DiffuseColor = (baseColor - baseColor * metallic) * DiffuseCol;
  vec3 SpecPreEnvBrdf = (dielectricSpecular - dielectricSpecular * metallic) + baseColor * metallic;
  vec3 SpecularColor = EnvBRDFApprox(SpecPreEnvBrdf, roughness, NoV);
{% endhighlight %}

#### Lighting.Diffuse, Lighting.Specular
The second part we need is **Lighting.Diffuse** and **Lighting.Specular**:
{% highlight hlsl %}
//UE4 codes
Lighting.Specular = ShadingModelContext.SpecularColor * (NoL * CalcSpecular(GBuffer.Roughness, NoH));
Lighting.Diffuse = NoL * ShadingModelContext.DiffuseColor;
{% endhighlight %}
Just copy CalcSpecular() function from UE4 `MobileShadingModels.ush` file and `MobileGGX.ush`. Implement in SP:
{% highlight glsl %}
 vec3 Specular = SpecularColor * (NoL * CalcSpecular(roughness, NoH));
 vec3 Diffuse = NoL * DiffuseColor;
{% endhighlight %}
Next, add them on `Color`:
{% highlight glsl %}
  vec3 Color = vec3(0.0, 0.0, 0.0); 
  float shadow = getShadowFactor();
  Color += shadow * LightIntensity * LightColor.rgb * (Diffuse + Specular);
{% endhighlight %}
So far I got:  <br />
![specular](/post-img/shaderposts/pbr-consistency-shader/diffuse-specular.jpg){: width="50%" }

#### SpecularIBL
{% highlight hlsl %}
//UE4 codes
Color += SpecularIBL * ShadingModelContext.SpecularColor;
{% endhighlight %}
For the sampling of IBL in SP, if I go back to see UE4's codes of sampling the cubemap and delete all the macros but just see the actual lines:
{% highlight hlsl %}
//UE4 codes
void GatherSpecularIBL(....)
{
	....
    // Fetch from cubemap and convert to linear HDR
    half3 SpecularIBL = 0.0f;

    half AbsoluteSpecularMip = ComputeReflectionCaptureMipFromRoughness(Roughness, ResolvedView.ReflectionCubemapMaxMip);
    half4 SpecularIBLSample = ReflectionCube.SampleLevel(ReflectionSampler, ProjectedCaptureVector, AbsoluteSpecularMip);

    SpecularIBL = RGBMDecode(SpecularIBLSample, MaxValue); // rgbm.rgb * (rgbm.a * MaxValue);
    SpecularIBL = SpecularIBL * SpecularIBL;
    ...
}
{% endhighlight %}
The operation here is to sample the cubemap, RGBMDecode it and power it. 
<!-- RGBM: RGBM encoding stores a color in the RGB channels and a multiplier (M) in the alpha channel. The range of RGBM lightmaps goes from 0 to 34.49(52.2) in linear space, and from 0 to 5 in gamma space. -->
There are parameters such as `ProjectedCaptureVector`, `AbsoluteSpecularMip`, `MaxValue`, which are hardly able to get in SP, I just ignore them. It might affect the final result accuracy, but we just can't make it 100% the same, since the environment in engine is also not absolute.
In my SP code, I used SP's function `pbrComputeSpecular` from the lib but deleted the `computeLOD` part, leaving the `envSampleLOD`:
{% highlight glsl %}
  SpecularIBL = GetSpecIBL(roughness, reflectDir, V, H, N, vectors);
{% endhighlight %}
{% highlight glsl %}
  vec3 GetSpecIBL(float roughness, vec3 R, vec3 V, vec3 H, vec3 N, LocalVectors vectors)
    {
        vec3 SpecularIBL = vec3(0.0f);

        for(int i=0; i<256; ++i)
        {
            vec2 Xi = fibonacci2D(i, 256);
            vec3 Hn = importanceSampleGGX(Xi, vectors.tangent, vectors.bitangent, vectors.normal, roughness);
            vec3 Ln = normalize(R + Hn);// -reflect(vectors.eye,Hn);
            vec4 SpecularIBLSample = SampleEnvLOD(Ln, 1);
            SpecularIBL += SpecularIBLSample.rgb;
        }

        SpecularIBL /= float(256);
        return SpecularIBL* SpecularIBL;
    }
{% endhighlight %}
Add on Color: `Color += SpecularIBL * SpecularColor;`<br />
So far I got:<br />
![add-spec](/post-img/shaderposts/pbr-consistency-shader/add-spec.jpg){: width="50%" }


#### SkyDiffuseLighting
{% highlight hlsl %}
//UE4 codes
Color += SkyDiffuseLighting * half3(ResolvedView.SkyLightColor.rgb) * ShadingModelContext.DiffuseColor * MaterialAO;
{% endhighlight %}

`SkyDiffuseLighting` is the indirect irradiance from the skylight. So I just use SP's function **envIrradiance()** from the lib. 
{% highlight glsl %}
vec3 envIrradiance(vec3 dir)
{
  float rot = environment_rotation * M_2PI;
  float crot = cos(rot);
  float srot = sin(rot);
  vec4 shDir = vec4(dir.xzy, 1.0);
  shDir = vec4(
    shDir.x * crot - shDir.y * srot,
    shDir.x * srot + shDir.y * crot,
    shDir.z,
    1.0);
  return max(vec3(0.0), vec3(
      dot(shDir, irrad_mat_red * shDir),
      dot(shDir, irrad_mat_green * shDir),
      dot(shDir, irrad_mat_blue * shDir)
    )) * environment_exposure;
}
{% endhighlight %}
![envIrradiance](/post-img/shaderposts/pbr-consistency-shader/envIrradiance.jpg){: width="50%" }<br />

Add on together in addition with AO, <br />
{% highlight glsl %}
Color += envIrradianceCustom(N) * DiffuseColor;
Color *= occlusion;
{% endhighlight %}
I got:<br />
![after-ao](/post-img/shaderposts/pbr-consistency-shader/after-ao.jpg){: width="50%" }<br />

#### ToneMapping
The purpose of the Tone Mapping function is to map the wide range of high dynamic range (HDR) colors into low dynamic range (LDR) that a display can output. The Filmic tonemapper that is used in UE4 matches the industry standard set by the Academy Color Encoding System (ACES) for television and film. With Unreal Engine 4.15, the Filmic tonemapper using the ACES standard is enabled by default, you can't shut it down. Which means, if I want the color in SP shows almost same with UE4, I need to apply an ACES tonemapping. 
<div class="container">
    <!-- COMPARISON SLIDER CODE START -->
    <div class="comparison-slider-wrapper">
    <!-- Comparison Slider - this div contain the slider with the individual images captions -->
    <div class="comparison-slider">
        <div class="overlay">After ACES.</div>
        <img src="/post-img/shaderposts/pbr-consistency-shader/slider-aces.jpg" alt="marioPhoto 2">
        <!-- Div containing the image layed out on top from the left -->
        <div class="resize">
            <div class="overlay">Before ACES.</div>
        <img src="/post-img/shaderposts/pbr-consistency-shader/slider-no-aces.jpg" alt="marioPhoto 1">
        </div>
        <!-- Divider where user will interact with the slider -->
        <div class="divider"></div>
    </div>
    <!-- All global captions if exist can go on the div bellow -->
    <div class="caption"> Before and After ACES Tonemapping in SP.</div>
    </div>
    <!-- COMPARISON SLIDER CODE END -->
</div>
Where we can see that before tone mapping, the light area has no detials. 
{% highlight glsl %}
vec3 Tonemapping(vec3 x)
{

  float a = 2.51; 
  float b = 0.03; 
  float c = 2.43; 
  float d = 0.59; 
  float e = 0.53;

  vec3 result = saturate((x*(a*x+b))/(x*(c*x+d)+e));
  return result;
}
{% endhighlight %}
This is my ACES tonemapping function that referenced from [ACES Filmic Tone Mapping Curve](https://knarkowicz.wordpress.com/2016/01/06/aces-filmic-tone-mapping-curve/) by Krzysztof Narkowicz, and make a change on `e` which is the toe strength, to 0.53 that used in engine.<br />

After tonemapping, the output is:<br />
`Comparison between UE4 and SP in ES3.1`
![ue4-sp](/post-img/shaderposts/pbr-consistency-shader/ue4-sp.jpg){: width="80%" }<br />



## Maya Unreal-like PBR Shader
With same approach, it's easy to develop an Unreal-like Maya PBR shader as well. I'm not gonna talk it in details, as it is basically just transform the SP glsl shader above into hlsl. <br />
![maya-unreal-pbr](/post-img/shaderposts/pbr-consistency-shader/maya-unreal-pbr.jpg){: width="40%" }<br />
![pistol](/post-img/shaderposts/pbr-consistency-shader/pistol.png){: width="90%" }
![assets](/post-img/shaderposts/pbr-consistency-shader/assets.png){: width="60%" }



## NPR Consistency Shader
![apep](/post-img/shaderposts/pbr-consistency-shader/apep.png){: width="100%" } <br />
Cel shader cross platform. In UE4, SP, Maya, from left to right.<br />

## Marmoset Unreal-like PBR Shader
![marmoset](/post-img/shaderposts/pbr-consistency-shader/marmoset.png){: width="80%" } <br />