---
layout: post
title: "PBR/NPR Consistency Shader in UE4/SP/Maya/Marmoset"
date:   2021-12-15 12:00
category: shaderposts
icon: scale
keywords: tag1, tag2
image: shader-balls-1.png
preview: 1
youtubeId: whFFbZq_geQ
---


1. [The Need](#the-need)
2. [First of All, UE4 PBR Pipeline](#first-of-all-ue4-pbr-pipeline)
    - [The Mobile BasePass Shader](#the-mobile-basepass-shader)
3. [Maya Unreal-like PBR Shader](#maya-unreal-like-pbr-shader)

    - [For the Project](#for-the-project)
4. [Substance Painter Unreal-like PBR Shader](#substance-painter-unreal-like-pbr-shader)
    - [Template](#template)
5. [Marmoset Unreal-like PBR Shader](#marmoset-unreal-like-pbr-shader)
6. [Maya NPR Shader](#maya-npr-shader)
7. [Substance NPR Shader](#substance-npr-shader)

![shader-balls-1](/post-img/shaderposts/pbr-consistency-shader/shader-balls-1.png){: width="60%" }<br />

## The Need
It is important for artists to work in a consistent environment no matter in the engine or DCC. Especially when the outsourced team usually doesn’t have the authority to get the project’s engine, they couldn’t test and check the final visual result in the engine after they finished the art assets. Therefore consistency between engine and DCC rendering is essential. Although DCC software usually comes with its own PBR default shader, the different engine has different render pipeline and the different project has its own shading strategy besides shading is different on mobile and PC, a custom consistency shader is needed. <br />
To do that, it needs a well understanding of the engine shading pipeline. <br />


## First of All, UE4 PBR Pipeline
Here I’m only talking about the mobile version PBR shader in UE4.<br />

### The Mobile BasePass Shader
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

Look at the `Color` variable, it starts from `half3 Color = 0;`, and then continues adding on layer by layer.<br />
`BaseColor`,`Metallic`,`Specular`,`Roughness` are parameters from the material panel input. <br />

There are some special parameters in the above calculation that need to know:
- **ShadingModelContext.DiffuseColor**
    - Calculated in `MobileShadingModels.ush`. 
    - `ShadingModelContext.DiffuseColor = GBuffer.BaseColor - GBuffer.BaseColor * GBuffer.Metallic;`
    - Which indicates the effect of metallic value. The higher the metallic is, the darker the `DiffuseColor` is. 
    - Looks like:

    ![diffusecolor](/post-img/shaderposts/pbr-consistency-shader/diffusecolor.png){: width="70%" }<br />

- **ShadingModelContext.SpecPreEnvBrdf**
    - Calculated in `MobileShadingModels.ush`. 
    - `ShadingModelContext.SpecularColor = (DielectricSpecular - DielectricSpecular * GBuffer.Metallic) + GBuffer.BaseColor * GBuffer.Metallic;`
    - Looks like:
    
    ![SpecPreEnvBrdf.png](/post-img/shaderposts/pbr-consistency-shader/SpecPreEnvBrdf.png){: width="70%" }<br />


- **ShadingModelContext.SpecularColor**
    - Calculated in `MobileShadingModels.ush`. 
    - Which is **ShadingModelContext.SpecPreEnvBrdf** calculated in `GetEnvBRDF()`
    - Looks like:
    
    ![specularcolor](/post-img/shaderposts/pbr-consistency-shader/specularcolor.png){: width="70%" }<br />

Below is the calculation of **ShadingModelContext.SpecularColor** and **ShadingModelContext.DiffuseColor** in `\Engine\Shaders\Private\MobileShadingModels.ush`<br />

{% highlight hlsl %}
	half DielectricSpecular = 0.08 * GBuffer.Specular;
	ShadingModelContext.SpecularColor = (DielectricSpecular - DielectricSpecular * GBuffer.Metallic) + GBuffer.BaseColor * GBuffer.Metallic;	// 2 mad
	ShadingModelContext.SpecPreEnvBrdf = ShadingModelContext.SpecularColor;
	ShadingModelContext.DiffuseColor = GBuffer.BaseColor - GBuffer.BaseColor * GBuffer.Metallic;
	...
        ShadingModelContext.SpecularColor = GetEnvBRDF(ShadingModelContext.SpecularColor, GBuffer.Roughness, NoV);

{% endhighlight %}

Below is the function **GetEnvBRDF()** in `\Engine\Shaders\Private\MobileShadingModels.ush`<br />

{% highlight hlsl %}
half3 GetEnvBRDF(half3 SpecularColor, half Roughness, half NoV)
    {
        return EnvBRDFApprox(SpecularColor, Roughness, NoV);
    }
{% endhighlight %}

Below is **EnvBRDFApprox()** in `\Engine\Shaders\Private\BRDF.ush`
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


- **Lighting.Specular**
    - Calculated in `MobileShadingModels.ush`, is in the struct of **FMobileDirectLighting**. 
    - Which is the specular comes from the light, just like *Phong*.
    - Looks like:
    
    ![lighting-specular](/post-img/shaderposts/pbr-consistency-shader/lighting-specular.png){: width="70%" }<br />

- **Lighting.Diffuse**
    - Calculated in `MobileShadingModels.ush`, is in the struct of **FMobileDirectLighting**. 
    - Looks like:
    
    ![lighting-diffuse](/post-img/shaderposts/pbr-consistency-shader/lighting-diffuse.png){: width="70%" }<br />

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

Below is the function **CalcSpecular()** in `\Engine\Shaders\Private\MobileShadingModels.ush`<br />
{% highlight hlsl %}
half CalcSpecular(half Roughness, half NoH)
{
	return (Roughness*0.25 + 0.25) * GGX_Mobile(Roughness, NoH);
}
{% endhighlight %}

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

The complete Lit output is:<br />
![lit](/post-img/shaderposts/pbr-consistency-shader/lit.png){: width="70%" }<br />





## Maya Unreal-like PBR Shader

### none




## Substance Painter Unreal-like PBR Shader
### Template
Template of posts setting is in `_drafts/template.md`. `Layout` is always named `post`. `Title` is a title of post, writing in quotation marks. `Date` written in the following format: `yyyy-mm-dd hh:mm`. In `category` specifies one category. In `icon` written the name of icon (its in the folder `images`). In `tags` is possible to write multiple tags using a comma. In `image` specify the path to image preview (can not fill). And in `preview` you can write `0` to on the main page didn't show the announcement of the post. 

