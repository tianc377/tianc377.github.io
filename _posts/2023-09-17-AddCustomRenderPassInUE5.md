---
layout: post
title:  "Add Custom Render Pass In UE5"
date:   2023-09-17 21:00
category: shaderposts
icon: layers
keywords: tag1, tag2
image: add-renderpass.png
preview: 1
youtubeId: vWOHPEhF99M
youtubeId2: atztUZg_GiY
---

1. [Extending From Last Post](#extending-from-last-post)
2. [How To](#how-to)
    - [The Plugin Structure](#the-plugin-structure)
    - [Shader Files](#shader-files)
        - [Usf File](#usf-file)
        - [Shader Header File](#shader-header-file)
        - [Shader Cpp File](#shader-cpp-file)
    - [Insert into the render pipeline](#insert-into-the-render-pipeline)
        - [SceneViewExtension](#SceneViewExtension)
            - [Cpp File](#cpp-file)
            - [Header File](#header-file)
         

       
    

## Extending From Last Post
In my last post, I tried to write a plugin to use `ush` file as a library file to write shader codes and include it in *custom node* in material editor: <br />
**[Use HLSL in UE5 by C++ Plugin](https://tianc377.github.io/shaderposts/2023/ShaderPlugin.html)**<br />
This was a good start to set up the dependencies of modules and mapping the shader directory into the correct place. <br />
With this structure then, I tried to extend it's function and added an actual <span style="color: #0fc2aa">global shader</span> as a custom render pass to Unreal's render pipeline, more importantly, without editing the engine codes. <br />
In this post I will note how I manage to do that.


## How to
### The Plugin Structure


![ctlib-struct](/post-img/shaderposts/add-renderpass/ctlib-struct.png)<br />
Compared with last post, there are several files added. <br />
* First group has Plugin cpp file `CTLib.cpp` and Plugin header file `CTLib.h`. <br />
* Second group has shader file `CTGlobalShader.usf`, shader header file `CTGlobalShader.h` and shader cpp file `CTGlobalShader.cpp`. <br />
* Third group has ViewExtension cpp file `CTSceneViewExtension.cpp` and ViewExtension header file `CTSceneViewExtension.h`. <br />

In last post I've already got the Plugin files in first group, `CTLib.cpp` is the one has shader directory mapping. <br /> 

### Shader Files

#### Usf File 

`CTGlobalShader.usf` 
* I wrote a very simple shader as a start, it simply multiplies the scene color with an Input Color.    
* `ApplyScreenTransform()` is a function from `"/Engine/Private/ScreenPass.ush"`, to transform position to texture UV.  
* `FScreenTransform SvPositionToInputTextureUV` is a parameter needs to be transfer into when shader is used. 
* I found above UV transform function from `Engine\Source\Runtime\Renderer\Private\PostProcess\PostProcessBloomSetup.cpp`
    {% highlight hlsl %}
    // ----------CTGlobalShader.usf-------------
    #include "/Engine/Public/Platform.ush"  //need to include this otherwise compilation error: cant found common.ush
    #include "/Engine/Private/Common.ush"
    #include "/Engine/Private/ScreenPass.ush"

    // Shader Parameters
    Texture2D InputTexture;
    SamplerState InputSampler;
    float4 InputColor;
    FScreenTransform SvPositionToInputTextureUV;

    // Fragment Shader
    void MainPS(
        float4 SvPosition : SV_POSITION,
        out float4 OutColor : SV_Target0) 
    {
        float2 UV = ApplyScreenTransform(SvPosition.xy, SvPositionToInputTextureUV); 
        OutColor = Texture2DSample(InputTexture, InputSampler, UV)* InputColor;
        
    }
    {% endhighlight %}
    

<br /> 


#### Shader Header File

`CTGlobalShader.h`
* In header file, it is a fixed format to setup shader type, shader, shader parameters in Unreal. I still use codes from `Engine\Source\Runtime\Renderer\Private\PostProcess\PostProcessBloomSetup.cpp` as reference to write my own. 

    {% highlight cpp %}
    // ----------CTGlobalShader.h-------------
    #pragma once

    #include "GlobalShader.h"
    #include "Runtime/Renderer/Private/ScreenPass.h"

    class FCTGlobalShaderPS : public FGlobalShader {
    public:
        DECLARE_SHADER_TYPE(FCTGlobalShaderPS, Global,);
        SHADER_USE_PARAMETER_STRUCT(FCTGlobalShaderPS, FGlobalShader);

        BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
            SHADER_PARAMETER(FScreenTransform, SvPositionToInputTextureUV) // Parameter for UV transform
            SHADER_PARAMETER_RDG_TEXTURE(Texture2D, InputTexture) // Declares read access to an FRDGTexture* which maps to 'InputTexture' in HLSL code.
            SHADER_PARAMETER_SAMPLER(SamplerState, InputSampler) 
            SHADER_PARAMETER(FLinearColor, InputColor) 
            RENDER_TARGET_BINDING_SLOTS()
        END_SHADER_PARAMETER_STRUCT()
    };
    
    {% endhighlight %}

<br /> 


#### Shader Cpp File

`CTGlobalShader.cpp`  
* in cpp file, it implements the shader type by a macro `IMPLEMENT_SHADER_TYPE()` from `Engine\Source\Runtime\RenderCore\Public\Shader.h`.
    {% highlight cpp %}
    // ----------CTGlobalShader.cpp-------------
    #include "CTGlobalShader.h"

    IMPLEMENT_SHADER_TYPE(, FCTGlobalShaderPS, TEXT("/Plugin/CTLib/CTGlobalShader.usf"), TEXT("MainPS"), SF_Pixel);
    // arguments are: IMPLEMENT_SHADER_TYPE(TemplatePrefix,ShaderClass,SourceFilename,FunctionName,Frequency);
    {% endhighlight %}


<br /> 


### Insert into the render pipeline

#### SceneViewExtension
There's very few documentation about global shader in Unreal, one can be found is 
[Adding Global Shaders to Unreal Engine](https://docs.unrealengine.com/5.2/en-US/adding-global-shaders-to-unreal-engine/), which is super outdated and need to modify the engine, have no idea how to implement it from this documentation  :poop: <br />
Well there is another interface provided by Unreal that can let us insert a pass in the post process passes: [SceneViewExtensions](https://docs.unrealengine.com/5.1/en-US/API/Runtime/Engine/FSceneViewExtensions/)<br />
In its header file `\Engine\Source\Runtime\Engine\Public\SceneViewExtension.h`, there's surprisingly a large block of description: <br />
{% highlight cpp %}
 /*  
        SCENE VIEW EXTENSIONS
    *  -----------------------------------------------------------------------------------------------
    *
    *  This system lets you hook various aspects of UE rendering.
    *  To create a view extension, it is advisable to inherit
    *  from FSceneViewExtensionBase, which implements the
    *  ISceneViewExtension interface.
    *
    *
    *
    *  INHERITING, INSTANTIATING, LIFETIME
    *  -----------------------------------------------------------------------------------------------
    *
    *  In order to inherit from FSceneViewExtensionBase, do the following:
    *
    *      class FMyExtension : public FSceneViewExtensionBase
    *      {
    *          public:
    *          FMyExtension( const FAutoRegister& AutoRegister, FYourParam1 Param1, FYourParam2 Param2 )
    *          : FSceneViewExtensionBase( AutoRegister )
    *          {
    *          }
    *      };
    *
    *  Notice that your first argument must be FAutoRegister, and you must pass it
    *  to FSceneViewExtensionBase constructor. To instantiate your extension and register
    *  it, do the following:
    *
    *      FSceneViewExtensions::NewExtension<FMyExtension>(Param1, Param2);
    *
    *  You should maintain a reference to the extension for as long as you want to
    *  keep it registered.
    *
    *      TSharedRef<FMyExtension,ESPMode::ThreadSafe> MyExtension;
    *      MyExtension = FSceneViewExtensions::NewExtension<FMyExtension>(Param1, Param2);
    *
    *  If you follow this pattern, the cleanup of the extension will be safe and automatic
    *  whenever the `MyExtension` reference goes out of scope. In most cases, the `MyExtension`
    *  variable should be a member of the class owning the extension instance.
    *
    *  The engine will keep the extension alive for the duration of the current frame to allow
    *  the render thread to finish.
    *
    *
    *
    *  OPTING OUT of RUNNING
    *  -----------------------------------------------------------------------------------------------
    *
    *  Each frame, the engine will invoke ISceneVewExtension::IsActiveThisFrame() to determine
    *  if your extension wants to run this frame. Returning false will cause none of the methods
    *  to be called this frame. The IsActiveThisFrame() method will be invoked again next frame.
    *
    *  If you need fine grained control over individual methods, your IsActiveThisFrame should
    *  return `true` and gate each method as needed.
    *
    *
    *
    *  PRIORITY
    *  -----------------------------------------------------------------------------------------------
    *  Extensions are executed in priority order. Higher priority extensions run first.
    *  To determine the priority of your extension, override ISceneViewExtension::GetPriority();
 */
{% endhighlight %}

So for this tool, 
* it only supports deferred shading pipeline
* it can insert before or after different passes, in header file there're functions end with _RenderThread:<br />
![ctlib-struct](/post-img/shaderposts/add-renderpass/renderthread.png){: width="90%" }<br />
Some documents saying it only works in post-processing stages and no mobile pipeline, but in the header file there's after basepass option and mobile option, dk yet, need to give a try. <br />
Anyway, I use `PrePostProcessPass_RenderThread` that is the one called right before Post Processing rendering begins. <br />

##### Cpp File

{% highlight cpp %}

    // ----------CTSceneViewExtension.cpp-------------
    #include "CTSceneViewExtension.h"
    #include "CTGlobalShader.h"
    #include "PixelShaderUtils.h"
    #include "PostProcess/PostProcessing.h"

    FCTSceneViewExtension::FCTSceneViewExtension(const FAutoRegister& AutoRegister, FLinearColor ColorInput) : FSceneViewExtensionBase(AutoRegister) 
    {
        TintColor = ColorInput;
    }

    void FCTSceneViewExtension::PrePostProcessPass_RenderThread(FRDGBuilder& GraphBuilder, const FSceneView& View, const FPostProcessingInputs& Inputs)
    {	
        const FIntRect ViewPort = static_cast<const FViewInfo&>(View).ViewRect;
        FGlobalShaderMap* GlobalShaderMap = GetGlobalShaderMap(GMaxRHIFeatureLevel);	//GetShader for certain feature level

        RDG_EVENT_SCOPE(GraphBuilder, "CTRenderPass");  //Use RDG_EVENT_SCOPE to add a GPU profile scope around passes. These are consumed by external profilers like RenderDoc, as well as RDG Insights.

        FRHISamplerState* PointClampSampler = TStaticSamplerState<SF_Point, AM_Clamp, AM_Clamp, AM_Clamp>::GetRHI();
        Inputs.Validate();

        FScreenPassTexture SceneColor((*Inputs.SceneTextures)->SceneColorTexture, ViewPort); 
        FRDGTextureDesc OutputDesc = SceneColor.Texture->Desc;
        const FScreenPassRenderTarget Output(GraphBuilder.CreateTexture(OutputDesc, TEXT("CT OUTPUT RT")), ViewPort, ERenderTargetLoadAction::ELoad);
        // Create a texture with FRDGBuilder::CreateTexture
        
        TShaderMapRef<FCTGlobalShaderPS> CTGlobalShaderPS(GlobalShaderMap);
        FCTGlobalShaderPS::FParameters* CTGlobalShaderParameters = GraphBuilder.AllocParameters<FCTGlobalShaderPS::FParameters>(); 
        // Allocate pass parameters with GraphBuilder.AllocParameters assign all relevant RDG resourses used in the execution lambda. 

        CTGlobalShaderParameters->InputTexture = SceneColor.Texture;//SceneColorCopyRenderTarget.Texture;//(*Inputs.SceneTextures)->SceneColorTexture; // Output.Texture;
        CTGlobalShaderParameters->SvPositionToInputTextureUV = (
        FScreenTransform::ChangeTextureBasisFromTo(FScreenPassTextureViewport(Output), FScreenTransform::ETextureBasis::TexelPosition, FScreenTransform::ETextureBasis::ViewportUV) *
        FScreenTransform::ChangeTextureBasisFromTo(FScreenPassTextureViewport(SceneColor), FScreenTransform::ETextureBasis::ViewportUV, FScreenTransform::ETextureBasis::TextureUV));
        CTGlobalShaderParameters->InputSampler = PointClampSampler;
        CTGlobalShaderParameters->InputColor = TintColor;
        CTGlobalShaderParameters->RenderTargets[0] = FRenderTargetBinding(SceneColor.Texture, ERenderTargetLoadAction::ELoad); //(FRDGTexture, ELoadAction)
        
        FPixelShaderUtils::AddFullscreenPass(
            GraphBuilder,
            GlobalShaderMap, 
            FRDGEventName(TEXT("CTPass")), 
            CTGlobalShaderPS, 
            CTGlobalShaderParameters,
            ViewPort); //GraphBuilder, Shader, Name, PS, Parameters, Viewport
        UE_LOG(LogTemp, Warning, TEXT("-----------------------AFTER AddFullscreenP--------------------------"));
        //FPixelShaderUtils::AddFullscreenPass for a full screen pixel shader pass, from Engine\Source\Runtime\RenderCore\Private\RenderGraphUtils.cpp
    }

{% endhighlight %}


##### Header File

{% highlight cpp %}
    // ----------CTSceneViewExtension.h-------------
    #pragma once

    #include "SceneViewExtension.h"


    class CTLIB_API FCTSceneViewExtension : public FSceneViewExtensionBase {
        FLinearColor TintColor;
    public:
        FCTSceneViewExtension(const FAutoRegister& AutoRegister, FLinearColor ColorInput); //declare a constructor 

        //~ Begin FSceneViewExtensionBase Interface
        virtual void SetupViewFamily(FSceneViewFamily& InViewFamily) override {};
        virtual void SetupView(FSceneViewFamily& InViewFamily, FSceneView& InView) override {};
        virtual void BeginRenderViewFamily(FSceneViewFamily& InViewFamily) override {};

        virtual void PrePostProcessPass_RenderThread(FRDGBuilder& GraphBuilder, const FSceneView& View, const FPostProcessingInputs& Inputs) override;
        //~ End FSceneViewExtensionBase Interface
    };    


{% endhighlight %}












<br />


<br />


<br />
References

https://docs.unrealengine.com/5.2/en-US/render-dependency-graph-in-unreal-engine/




<br /> 
a <span style="color: #0fc2aa">global shader</span> (shaders that are not created using the Material Editor, operate on fixed geometry) can have more advanced functionality like post-processing effects or a custom shader pass, etc. And it's able to created in a plugin which makes it easy to implement in other projects.

<br /> 