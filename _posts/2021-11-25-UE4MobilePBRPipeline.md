---
layout: post
title: "UE4 Mobile PBR Pipeline"
date:   2021-12-15 12:00
category: shaderposts
icon: sort-descending
keywords: tag1, tag2
image: pipeline.jpg
preview: 1
youtubeId: whFFbZq_geQ
---


1. [The Need](#the-need)
2. [The Mobile BasePass Shader](#the-mobile-base-pass-shader)
    - [Codes Without Macros](#codes-without-macros)
3. [To Visualize Each Piece](#to-visualize-each-piece)
4. [ComputeIndirect](#computeindirect)
5. [ENABLE_SKY_LIGHT](#enable_sky_light)
	- [SkyDiffuseLighting](#skydiffuselighting)
	- [DiffuseLookup](#diffuselookup)
	- [IndirectIrradiance](#indirectirradiance)
6. [FMobileDirectLighting](#fmobiledirectlighting)
	- [Lighting.Specular](#lightingspecular)
	- [Lighting.Diffuse](#lightingdiffuse)
	- [CalcSpecular()](#calcspecular)
	- [GGX_Mobile()](#ggxmobile)
7. [SpecularIBL](#specularibl)
8. [AccumulateLightingOfDynamicPointLight](#accumulatelightingofdynamicpointlight)
9. [DiffuseColor, SpecPreEnvBrdf, SpecularColor](#diffusecolor-specpreenvbrdf-specularcolor)
	- [ShadingModelContext.DiffuseColor](#shadingmodelcontextdiffusecolor)
	- [ShadingModelContext.SpecPreEnvBrdf](#shadingmodelcontextspecpreenvbrdf)
	- [ShadingModelContext.SpecularColor](#shadingmodelcontextspecularcolor)
10. [EnvBRDFApprox](#envbrdfapprox)
11. [Add Emission](#addemission;)
12. [Next Step](#next-step)



![pipeline](/post-img/shaderposts/ue4-pbr-pipeline/pipeline.jpg){: width="90%" }<br />

## The Need
No matter developing shader in NPR or PBR, a well-understanding of the default shading pipeline is important, especially in Unreal Engine where all the shader things are packed in their source codes. 
To manipulate the default pipeline or create your own lighting model, it needs a well understanding of the engine shading pipeline.<br />


## The Mobile BasePass Shader
### Codes Without Macros
`\Engine\Shaders\Private\MobileBasePassPixelShader.usf`
This is the path of the UE4 mobile PBR shader. There are a lot of macro there, such as `MATERIALBLENDING_...`, `MATERIAL_SHADINGMODEL_UNLIT`,  `FULLY_ROUGH`,  `NONMETAL`, etc.. Let's just remove all of them and see what the code is like.
{% highlight hlsl %}
void Main( 
	FVertexFactoryInterpolantsVSToPS Interpolants
	, FMobileBasePassInterpolantsVSToPS BasePassInterpolants
	, in float4 SvPosition : SV_Position
	OPTIONAL_IsFrontFace
	, out half4 OutColor	: SV_Target0
	)
{

	ResolvedView = ResolveView();

	SvPosition.y = ResolvedView.BufferSizeAndInvSize.y - SvPosition.y - 1;


	FMaterialPixelParameters MaterialParameters = GetMaterialPixelParameters(Interpolants, SvPosition);
	FPixelMaterialInputs PixelMaterialInputs;
	{
		float4 ScreenPosition = SvPositionToResolvedScreenPosition(SvPosition);
		float3 WorldPosition = BasePassInterpolants.PixelPosition.xyz;
		float3 WorldPositionExcludingWPO = BasePassInterpolants.PixelPosition.xyz;
		CalcMaterialParametersEx(MaterialParameters, PixelMaterialInputs, SvPosition, ScreenPosition, bIsFrontFace, WorldPosition, WorldPositionExcludingWPO);

	}


	// Store the results in local variables and reuse instead of calling the functions multiple times.
	FGBufferData GBuffer = (FGBufferData)0;
	GBuffer.WorldNormal = MaterialParameters.WorldNormal;
	GBuffer.BaseColor = GetMaterialBaseColor(PixelMaterialInputs);
	GBuffer.Metallic = GetMaterialMetallic(PixelMaterialInputs);
	GBuffer.Specular = GetMaterialSpecular(PixelMaterialInputs);
	GBuffer.Roughness = GetMaterialRoughness(PixelMaterialInputs);
	GBuffer.ShadingModelID = GetMaterialShadingModel(PixelMaterialInputs);
	half MaterialAO = GetMaterialAmbientOcclusion(PixelMaterialInputs);


	GBuffer.GBufferAO = MaterialAO;
	// The smallest normalized value that can be represented in IEEE 754 (FP16) is 2^-24 = 5.96e-8.
	// The code will make the following computation involving roughness: 1.0 / Roughness^4.
	// Therefore to prevent division by zero on devices that do not support denormals, Roughness^4
	// must be >= 5.96e-8. We will clamp to 0.015625 because 0.015625^4 = 5.96e-8.
	//
	// Note that we also clamp to 1.0 to match the deferred renderer on PC where the roughness is 
	// stored in an 8-bit value and thus automatically clamped at 1.0.
	GBuffer.Roughness = max(0.015625, GetMaterialRoughness(PixelMaterialInputs));
	
	FMobileShadingModelContext ShadingModelContext = (FMobileShadingModelContext)0;
	ShadingModelContext.Opacity = GetMaterialOpacity(PixelMaterialInputs);


	half3 Color = 0;

	half CustomData0 = GetMaterialCustomData0(MaterialParameters);
	half CustomData1 = GetMaterialCustomData1(MaterialParameters);
	InitShadingModelContext(ShadingModelContext, GBuffer, MaterialParameters.SvPosition, MaterialParameters.CameraVector, CustomData0, CustomData1);
	float3 DiffuseDir = MaterialParameters.WorldNormal;

	half IndirectIrradiance;
	half3 IndirectColor;
	ComputeIndirect(Interpolants, DiffuseDir, ShadingModelContext, IndirectIrradiance, IndirectColor);
	Color += IndirectColor;

        half3 SkyDiffuseLighting = GetSkySHDiffuseSimple(MaterialParameters.WorldNormal);
        half3 DiffuseLookup = SkyDiffuseLighting * ResolvedView.SkyLightColor.rgb;
        IndirectIrradiance += Luminance(DiffuseLookup);
			
	Color *= MaterialAO;
	IndirectIrradiance *= MaterialAO;

	// Shadow
	half Shadow = GetPrimaryPrecomputedShadowMask(Interpolants).r;

	half NoL = max(0, dot(MaterialParameters.WorldNormal, MobileDirectionalLight.DirectionalLightDirectionAndShadowTransition.xyz));
	half3 H = normalize(MaterialParameters.CameraVector + MobileDirectionalLight.DirectionalLightDirectionAndShadowTransition.xyz);
	half NoH = max(0, dot(MaterialParameters.WorldNormal, H));

	// Direct Lighting (Directional light) + IBL
	FMobileDirectLighting Lighting = MobileIntegrateBxDF(ShadingModelContext, GBuffer, NoL, MaterialParameters.CameraVector, H, NoH);
	// MobileDirectionalLight.DirectionalLightDistanceFadeMADAndSpecularScale.z saves SpecularScale for direction light.
	Color += (Shadow) * MobileDirectionalLight.DirectionalLightColor.rgb * (Lighting.Diffuse + Lighting.Specular * MobileDirectionalLight.DirectionalLightDistanceFadeMADAndSpecularScale.z);


	// Environment map has been prenormalized, scale by lightmap luminance
	half3 SpecularIBL = GetImageBasedReflectionLighting(MaterialParameters, GBuffer.Roughness, IndirectIrradiance, 1.0f);
	
	Color += SpecularIBL * ShadingModelContext.SpecularColor;

	// Local lights
	...
	
		AccumulateLightingOfDynamicPointLight(MaterialParameters, 
												ShadingModelContext,
												GBuffer,
												MobileMovablePointLight0.LightPositionAndInvRadius, 
												MobileMovablePointLight0.LightColorAndFalloffExponent, 
												MobileMovablePointLight0.SpotLightDirectionAndSpecularScale, 
												MobileMovablePointLight0.SpotLightAnglesAndSoftTransitionScaleAndLightShadowType, 
												#if SUPPORT_SPOTLIGHTS_SHADOW
												Settings,
												MobileMovablePointLight0.SpotLightShadowSharpenAndShadowFadeFraction,
												MobileMovablePointLight0.SpotLightShadowmapMinMax,
												MobileMovablePointLight0.SpotLightShadowWorldToShadowMatrix,
												#endif
												Color);
	...

	// Skylight
	//@mw todo
	// TODO: Also need to do specular.
	Color += SkyDiffuseLighting * half3(ResolvedView.SkyLightColor.rgb) * ShadingModelContext.DiffuseColor * MaterialAO;

	half4 VertexFog = half4(0, 0, 0, 1);

	VertexFog = BasePassInterpolants.VertexFog;

	// NEEDS_BASEPASS_PIXEL_FOGGING is not allowed on mobile for the sake of performance.
				 
	half3 Emissive = GetMaterialEmissive(PixelMaterialInputs);

	Color += Emissive;

	// On mobile, water (an opaque material) is rendered as trnaslucent with forced premultiplied alpha blending (see MobileBasePass::SetTranslucentRenderState)
	OutColor.rgb = Color * VertexFog.a + VertexFog.rgb;

{% endhighlight %}


## To Visualize Each Piece
It's hard to know what's going on by just watching the codes. So let's just go to the usf file return the specific parameter that you want to see, for instance:<br />
![to-see](/post-img/shaderposts/ue4-pbr-pipeline/to-see.jpg){: width="50%" }<br />
And go back to the engine, just make some minor changes, such as moving a node in the master material, then click apply, the shader will compile.<br />
Here I used a standard PBR material in a simple lighting environment, containing a directional light, a skylight, and a background plane. The complete Lit output is:<br />
![lit](/post-img/shaderposts/ue4-pbr-pipeline/lit.png){: width="70%" }<br />



## ComputeIndirect
The pixel shader starting from **Main()** function, and at line 643 `half3 Color = 0;` the color starts adding layer by layer. At first, it is added by `IndirectColor`, calculated from **ComputeIndirect**: 
{% highlight hlsl %}
half ComputeIndirect(FVertexFactoryInterpolantsVSToPS Interpolants, float3 DiffuseDir, FMobileShadingModelContext ShadingModelContext, out half IndirectIrradiance, out half3 Color)
{
	//To keep IndirectLightingCache conherence with PC, initialize the IndirectIrradiance to zero.
	IndirectIrradiance = 0;
	Color = 0;


	// Indirect Diffuse
#if LQ_TEXTURE_LIGHTMAP
	float2 LightmapUV0, LightmapUV1;
	uint LightmapDataIndex;
	GetLightMapCoordinates(Interpolants, LightmapUV0, LightmapUV1, LightmapDataIndex);

	half4 LightmapColor = GetLightMapColorLQ(LightmapUV0, LightmapUV1, LightmapDataIndex, DiffuseDir);
	Color += LightmapColor.rgb * ShadingModelContext.DiffuseColor * View.IndirectLightingColorScale;
	IndirectIrradiance = LightmapColor.a;
	...
{% endhighlight %}
This function samples the lightmap for static mesh with baked light. Notice that for mobile it is `LQ_TEXTURE_LIGHTMAP`. Here I don't have any lightmaps, the result is just black: <br />
![black.png](/post-img/shaderposts/ue4-pbr-pipeline/black.png){: width="50%" }<br />

<br />

## ENABLE_SKY_LIGHT
Then it comes **IndirectIrradiance**:<br />
![codes(3).png](/post-img/shaderposts/ue4-pbr-pipeline/codes(3).png){: width="80%" }<br />
The **IndirectIrradiance** is calculated under **ENABLE_SKY_LIGHT**:
{% highlight hlsl %}
#if ENABLE_SKY_LIGHT
		half3 SkyDiffuseLighting = GetSkySHDiffuseSimple(MaterialParameters.WorldNormal);
		half3 DiffuseLookup = SkyDiffuseLighting * ResolvedView.SkyLightColor.rgb;
		IndirectIrradiance += Luminance(DiffuseLookup);
#endif
{% endhighlight %}

Where,<br />
### SkyDiffuseLighting
`half3 SkyDiffuseLighting`<br />
![sky-diffuse-lighting](/post-img/shaderposts/ue4-pbr-pipeline/sky-diffuse-lighting.jpg){: width="40%" }<br />

### DiffuseLookup
`half3 DiffuseLookup`<br />
![diffuse-lookup](/post-img/shaderposts/ue4-pbr-pipeline/diffuse-lookup.jpg){: width="40%" }<br />

### IndirectIrradiance
`IndirectIrradiance`<br />
![indirect-irradiance](/post-img/shaderposts/ue4-pbr-pipeline/indirect-irradiance.png){: width="40%" }<br />
**IndirectIrradiance** will be used later in **SpecularIBL**

<br />

## FMobileDirectLighting
Continue scroll down, we will see some common parameters like `NoL`, `H`, `NoH`, after that we see **FMobileDirectLighting Lighting**. <br />
{% highlight hlsl %}
FMobileDirectLighting Lighting = MobileIntegrateBxDF(ShadingModelContext, GBuffer, NoL, MaterialParameters.CameraVector, H, NoH);
Color += (Shadow) * MobileDirectionalLight.DirectionalLightColor.rgb * (Lighting.Diffuse + Lighting.Specular * MobileDirectionalLight.DirectionalLightDistanceFadeMADAndSpecularScale.z);
{% endhighlight %}
Where Color += the `shadow`, directional light color, `Lighting.Diffuse` and `Lighting.Specular`and the directional light specular scale.
Let's output the result of it: <br />
![codes](/post-img/shaderposts/ue4-pbr-pipeline/codes.png){: width="80%" }<br />
You see that the shading is almost done except lacking of some sky light in the shadow area. So, what has been done in that two lines of code?<br />
Firstly, lets see the **FMobileDirectLighting Lighting** in `\Engine\Shaders\Private\MobileShadingModels.ush`.<br />

### Lighting.Specular
- Calculated in `MobileShadingModels.ush`, is in the struct of **FMobileDirectLighting**. 
- Which is the specular comes from the light, just like *Phong*.
- Looks like:

`Lighting.Specular`<br />
![lighting-specular](/post-img/shaderposts/ue4-pbr-pipeline/lighting-specular.png){: width="50%" }<br />

### Lighting.Diffuse
- Calculated in `MobileShadingModels.ush`, is in the struct of **FMobileDirectLighting**. 
- Looks like:

`Lighting.Diffuse`<br />
![lighting-diffuse](/post-img/shaderposts/ue4-pbr-pipeline/lighting-diffuse.png){: width="50%" }<br />

Below is the part **FMobileDirectLighting Lighting** in `\Engine\Shaders\Private\MobileShadingModels.ush`<br />
{% highlight hlsl %}
...
struct FMobileDirectLighting
{
	half3 Diffuse;
	half3 Specular;
};
...
Lighting.Specular = ShadingModelContext.SpecularColor * (NoL * CalcSpecular(GBuffer.Roughness, NoH));
Lighting.Diffuse = NoL * ShadingModelContext.DiffuseColor;
...
{% endhighlight %}
**Lighting.Specular + Lighting.Diffuse** will looks like:<br />
![diffuse-plus-specular](/post-img/shaderposts/ue4-pbr-pipeline/diffuse-plus-specular.jpg){: width="50%" }<br />


### CalcSpecular()
Below is the function **CalcSpecular()** that used above to calculate the specular.<br />
`\Engine\Shaders\Private\MobileShadingModels.ush`<br />
{% highlight hlsl %}
half CalcSpecular(half Roughness, half NoH)
{
	return (Roughness*0.25 + 0.25) * GGX_Mobile(Roughness, NoH);
}
{% endhighlight %}

### GGX_Mobile()
and **GGX_Mobile()** is in `\Engine\Shaders\Private\MobileGGX.ush`
{% highlight hlsl %}
half GGX_Mobile(half Roughness, float NoH)
{
    // Walter et al. 2007, "Microfacet Models for Refraction through Rough Surfaces"
	float OneMinusNoHSqr = 1.0 - NoH * NoH; 
	half a = Roughness * Roughness;
	half n = NoH * a;
	half p = a / (OneMinusNoHSqr + n * n);
	half d = p * p;
	// clamp to avoid overlfow in a bright env
	return min(d, 2048.0);
}
{% endhighlight %}

<br />

## SpecularIBL

Continue scrolling down, here it comes **SpecularIBL**. <br />
{% highlight hlsl %}
...
half3 SpecularIBL = GetImageBasedReflectionLighting(MaterialParameters, GBuffer.Roughness, IndirectIrradiance, 1.0f);
Color += SpecularIBL * ShadingModelContext.SpecularColor;
...
{% endhighlight %}

This is the specular created by image-based lighting, showing when your skylight sets to ![cubemap](/post-img/shaderposts/ue4-pbr-pipeline/cubemap.jpg){: width="30%" }<br />
Which looks like:<br />
`SpecularIBL`<br />
![specularIBL](/post-img/shaderposts/ue4-pbr-pipeline/SpecularIBL.jpg){: width="50%" }<br />

Later,  **SpecularIBL** is multiplied with [**ShadingModelContext.SpecularColor**](#diffusecolor-specpreenvbrdf-specularcolor) and add on to the `Color`.<br />
Which looks like:<br />
`SpecularIBL * ShadingModelContext.SpecularColor` <br />
![specularibl-specularcolor](/post-img/shaderposts/ue4-pbr-pipeline/specularibl-specularcolor.jpg){: width="50%" }<br />

Then add it on the `Color`:<br />
`Color += SpecularIBL * ShadingModelContext.SpecularColor;`<br />
![color+specularibl](/post-img/shaderposts/ue4-pbr-pipeline/color+specularibl.jpg){: width="50%" }<br />

See the metal necklace, it turns from black to silver with the light of cubemap after adding the `SpecularIBL`.<br />
``![necklace](/post-img/shaderposts/ue4-pbr-pipeline/necklace.jpg){: width="30%" }  ![necklace2](/post-img/shaderposts/ue4-pbr-pipeline/necklace2.jpg){: width="30%" }<br />


<br />

## AccumulateLightingOfDynamicPointLight
After the SpecularIBL, there is a function called **AccumulateLightingOfDynamicPointLight**, which is in charge of some dynamic lights except the directional light. This part calculated the attenuation of lights, NoL, and light color, then multiplied with `Lighting.Diffuse + Lighting.Specular`<br />
{% highlight hlsl %}
FMobileDirectLighting Lighting = MobileIntegrateBxDF(ShadingModelContext, GBuffer, PointNoL, MaterialParameters.CameraVector, PointH, PointNoH);
Color += min(65000.0, (Attenuation) * LightColorAndFalloffExponent.rgb * (1.0 / PI) * (Lighting.Diffuse + Lighting.Specular * SpotLightDirectionAndSpecularScale.w));		
{% endhighlight %}
`AccumulateLightingOfDynamicPointLight`<br />
![accumulateLight](/post-img/shaderposts/ue4-pbr-pipeline/accumulateLight.jpg){: width="50%" }<br />

Then scroll down, there is another **ENABLE_SKY_LIGHT** macro, which does this:
{% highlight hlsl %}
Color += SkyDiffuseLighting * half3(ResolvedView.SkyLightColor.rgb) * ShadingModelContext.DiffuseColor * MaterialAO;
{% endhighlight %}
This line uses of the `SkyDiffuseLighting` we got in the last [**ENABLE_SKY_LIGHT**](#enable_sky_light) macro part before, and multiplied with skylight color![skylight-color](/post-img/shaderposts/ue4-pbr-pipeline/skylight-color.jpg){: width="30%" }, also multiplied with `ShadingModelContext.DiffuseColor * MaterialAO`. <br />
We already seen [**ShadingModelContext.SpecularColor**](#shadingmodelcontextspecularcolor) in the `SpecularIBL` part, and here we see [**ShadingModelContext.DiffuseColor**](#shadingmodelcontextdiffusecolor). Read below to find what's their function is:
<br />

## DiffuseColor, SpecPreEnvBrdf, SpecularColor

{% highlight hlsl %}
struct FMobileShadingModelContext
{
	half Opacity;
	half3 DiffuseColor;
#if NONMETAL
	half SpecularColor;
#else
	half3 SpecularColor;
#endif
{% endhighlight %}

### ShadingModelContext.DiffuseColor
- Calculated in `MobileShadingModels.ush`. 
- `ShadingModelContext.DiffuseColor = GBuffer.BaseColor - GBuffer.BaseColor * GBuffer.Metallic;`
- Which indicates the effect of metallic value. The higher the metallic is, the darker the `DiffuseColor` is. 
- Looks like:<br />

`ShadingModelContext.DiffuseColor`<br />
![diffusecolor](/post-img/shaderposts/ue4-pbr-pipeline/diffusecolor.png){: width="60%" }<br />

### ShadingModelContext.SpecPreEnvBrdf
- Calculated in `MobileShadingModels.ush`. 
- `ShadingModelContext.SpecularColor = (DielectricSpecular - DielectricSpecular * GBuffer.Metallic) + GBuffer.BaseColor * GBuffer.Metallic;`
- Which makes  the nonmetal part black and metal part lighter, so that when it applied with `SpecularIBL`, it makes metal part receive more IBL lights:<br />

`ShadingModelContext.SpecPreEnvBrdf`<br />
![SpecPreEnvBrdf.png](/post-img/shaderposts/ue4-pbr-pipeline/SpecPreEnvBrdf.png){: width="60%" }<br />


### ShadingModelContext.SpecularColor
- Calculated in `MobileShadingModels.ush`. 
- Which is **ShadingModelContext.SpecPreEnvBrdf** calculated in [**EnvBRDFApprox**](#envbrdfapprox)
- Which 'means we have plausible Fresnel and roughness behavior for image based lighting', according to [Physically Based Shading on Mobile](https://www.unrealengine.com/en-US/blog/physically-based-shading-on-mobile)

`ShadingModelContext.SpecularColor`<br />
![specularcolor](/post-img/shaderposts/ue4-pbr-pipeline/specularcolor.png){: width="60%" }<br />

Below is the relative codes of **ShadingModelContext.SpecularColor** and **ShadingModelContext.DiffuseColor** in `\Engine\Shaders\Private\MobileShadingModels.ush`<br />

{% highlight hlsl %}
	half DielectricSpecular = 0.08 * GBuffer.Specular;
	ShadingModelContext.SpecularColor = (DielectricSpecular - DielectricSpecular * GBuffer.Metallic) + GBuffer.BaseColor * GBuffer.Metallic;	// 2 mad
	ShadingModelContext.SpecPreEnvBrdf = ShadingModelContext.SpecularColor;
	ShadingModelContext.DiffuseColor = GBuffer.BaseColor - GBuffer.BaseColor * GBuffer.Metallic;
	...
        ShadingModelContext.SpecularColor = GetEnvBRDF(ShadingModelContext.SpecularColor, GBuffer.Roughness, NoV);

{% endhighlight %}


### EnvBRDFApprox
Below is the function **GetEnvBRDF()** in `\Engine\Shaders\Private\MobileShadingModels.ush`<br />

{% highlight hlsl %}
half3 GetEnvBRDF(half3 SpecularColor, half Roughness, half NoV)
    {
        return EnvBRDFApprox(SpecularColor, Roughness, NoV);
    }
{% endhighlight %}

Below is **EnvBRDFApprox()** in `\Engine\Shaders\Private\BRDF.ush`. You can read [Physically Based Shading on Mobile](https://www.unrealengine.com/en-US/blog/physically-based-shading-on-mobile) to see about this approximate environment BRDF method. 
    
{% highlight hlsl %}
half3 EnvBRDFApprox( half3 SpecularColor, half Roughness, half NoV )
    {
        // [ Lazarov 2013, "Getting More Physical in Call of Duty: Black Ops II" ]
        // Adaptation to fit our G term.
        const half4 c0 = { -1, -0.0275, -0.572, 0.022 };
        const half4 c1 = { 1, 0.0425, 1.04, -0.04 };
        half4 r = Roughness * c0 + c1;
        half a004 = min( r.x * r.x, exp2( -9.28 * NoV ) ) * r.x + r.y;
        half2 AB = half2( -1.04, 1.04 ) * a004 + r.zw;

        // Anything less than 2% is physically impossible and is instead considered to be shadowing
        // Note: this is needed for the 'specular' show flag to work, since it uses a SpecularColor of 0
        AB.y *= saturate( 50.0 * SpecularColor.g );

        return SpecularColor * AB.x + AB.y;
    }
{% endhighlight %}

<br />

## Add Emission
The last step is to adding emission on the `Color` if the material has this input.
{% highlight hlsl %}
half3 Emissive = GetMaterialEmissive(PixelMaterialInputs);
Color += Emissive;
{% endhighlight %}

<br />

## Next Step
Once finished the disassembly of the codes, we know what each part looks like and what it functions, we can start to rebuild it on other platforms such as Maya, SP, 3dMax, etc. 
Also, if you want to make custom non physically based shading, it’s important to know the PBR pipeline first, so that you know where you should revise and write your own bits, and how to insert a custom lighting model. 



