---
layout: post
title:  "Add Custom Render Pass In UE5"
date:   2023-09-17 21:00
category: unrealtools
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
    - [Add Module to the Project](#add-module-to-the-project)
    - [Shader Files](#shader-files)
        - [Usf File](#usf-file)
        - [Shader Header File](#shader-header-file)
        - [Shader Cpp File](#shader-cpp-file)
    - [Insert Into the Render Pipeline](#insert-into-the-render-pipeline)
        - [SceneViewExtension](#sceneviewextension)
        - [Cpp File](#cpp-file)
        - [Header File](#header-file)
    - [Use Custom Module](#use-custom-module)
3. [What's Next](#whats-next)

         

       
    

## Extending From Last Post
In my last post, I tried to write a plugin to use `ush` file as a library file to write shader codes and include it in *custom node* in material editor: <br />
**[Use HLSL in UE5 by C++ Plugin](https://tianc377.github.io/unrealtools/2023/ShaderPlugin.html)**<br />
This was a good start to set up the dependencies of modules and mapping the shader directory into the correct place. <br />
With this structure then, I tried to extend it's function and added an actual <span style="color: #0fc2aa">global shader</span> as a custom render pass to Unreal's render pipeline, more importantly, without editing the engine codes. <br />
In this post I will note how I manage to do that.


## How to
### The Plugin Structure


![ctlib-struct](/post-img/unrealtools/add-renderpass/ctlib-struct.png)<br />
Compared with last post, there are several files added. <br />
* First group has Plugin cpp file `CTLib.cpp` and Plugin header file `CTLib.h`. In last post I've already got these two files, `CTLib.cpp` is the one has shader directory mapping. <br /> 
* Second group has shader file `CTGlobalShader.usf`, shader header file `CTGlobalShader.h` and shader cpp file `CTGlobalShader.cpp`. <br />
* Third group has ViewExtension cpp file `CTSceneViewExtension.cpp` and ViewExtension header file `CTSceneViewExtension.h`. <br />


### Add Module to the Project

Compared with last post, there're 4 project files needs to modified in addition in order to add a custom module into the project: <br />
![addmodule3](/post-img/unrealtools/add-renderpass/addmodule3.png) <br />

* xxx.uproject <br />
    ![add-module](/post-img/unrealtools/add-renderpass/add-module.png) <br />

* xxx.Build.cs <br />
    ![add-module2](/post-img/unrealtools/add-renderpass/add-module2.png) <br />

* xxx.Target.cs & xxxEditor.Target.cs <br />
    ![target-cs.png](/post-img/unrealtools/add-renderpass/target-cs.png) <br />

<br />
<br />

### Shader Files

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


#### Usf File 

`CTGlobalShader.usf` 

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
        float2 UV = ApplyScreenTransform(SvPosition.xy, SvPositionToInputTextureUV); //*1
        OutColor = Texture2DSample(InputTexture, InputSampler, UV)* InputColor;
        
    }
    {% endhighlight %}
* I wrote a very simple shader as a start, it simply multiplies the scene color with an Input Color.    
* `ApplyScreenTransform()` is a function from `"/Engine/Private/ScreenPass.ush"`, to transform position to texture UV.  
* `FScreenTransform SvPositionToInputTextureUV` is a parameter needs to be transfer into when shader is used. 
* *1 `ApplyScreenTransform()`: <br /> 
![apply-st](/post-img/unrealtools/add-renderpass/apply-st.png) <br /> 
<!--当SV_POSITION值作为像素着色器的输入时，其xy的值表示当前像素在屏幕空间下的像素坐标加上0.5的偏移量。为什么SV_POSITION.xy相比像素坐标会存在0.5的偏移呢？这是因为像素着色器的SV_POSITION.xy的值实际上是像素中心点对应的屏幕空间坐标，例如800*600分辨率的应用，下标为(0, 0)像素的中心点对应的坐标即为(0.5, 0.5)，下标为(799, 599)像素的中心点对应的坐标即为(799.5, 599.5)。
DirectX的NDC空间取值范围为x：[-1, 1]，y：[-1, 1]，z：[0, 1]。-->

This is used for transforming `SV_POSITION` from (0 ~ bufferwith) x (0 ~ bufferheight) to 0 ~ 1 as a texture UV. <br /> 
It will multiply `SV_POSITION.xy` by a Scale and add a Bias. <br /> 
`FScreenTransform SvPositionToInputTextureUV` defined as a float4 composed with FVector2f Scale and FVector2f Bias, need to get this argument in my viewextension.cpp later<br /> 
This way of transform coordinates is from most postprocessing shaders in Unreal, but I can also simply use: <br />
`float2 UV = float4(SvPosition.xy * (1 / ViewportUV.xy), 0.0f, 0.0f);` like below: <br />

{% highlight hlsl %}

    // ----------CTGlobalShader.usf-------------
    #include "/Engine/Private/Common.ush"
    #include "/Engine/Private/ScreenPass.ush"
    #define FVector2f float2
    Texture2D InputTexture;
    SamplerState InputSampler;
    float4 InputColor;
    FVector2f ViewportUV;


    void MainPS(
        float4 SvPosition : SV_POSITION,
        out float4 OutColor : SV_Target0) 
    {
        float2 UV = float4((SvPosition.xy - 0.5) * (1 / ViewportUV.xy), 0.0f, 0.0f); 
        OutColor = Texture2DSample(InputTexture, InputSampler, UV) * InputColor;
        
    }
{% endhighlight %}

Where `ViewportUV` is from `FScreenPassTextureViewport(SceneColor).Rect.Width(), FScreenPassTextureViewport(SceneColor).Rect.Height()` later in the `CTSceneViewExtension.cpp` file. 

* **SV_POSITION** in Pixel Shader is the center position of a pixel, so that it has 0.5 offset for each pixel, when we scale SV_POSITION to 0 ~ 1 range, we need to translate pixel position back to (0,0), which is minus 0.5 firstly, then multiply its xy with an inverse scale factor `float4((SvPosition.xy - 0.5) * (1 / ViewportUV.xy), 0.0f, 0.0f)`.<br /> 
![pixel](/post-img/unrealtools/add-renderpass/pixel.png)
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
![ctlib-struct](/post-img/unrealtools/add-renderpass/renderthread.png){: width="90%" }<br />
Some documents saying it only works in post-processing stages and no mobile pipeline, but in the header file there's after basepass option and mobile option, dk yet, need to give a try. <br />
Anyway, I use `PrePostProcessPass_RenderThread` that is the one called right before Post Processing rendering begins. <br />
* In its usage description above, the first step is to create a class that inherits from **FSceneViewExtensionBase** and first argument needs to be **const FAutoRegister& AutoRegister**. 
* Second step is to inherit some virtual functions to set up **FSceneView** and **FSceneViewFamily** and type of *RenderThread* from **ISceneViewExtension** class. 
    * **FSceneView** is a class that manage projection from scene space into a 2D screen region, things like **view matrix**, actor being viewd, **FOV**, **view frustum**, **near/far clipping plane** and etc are also inside it. Check it out in `Engine\Source\Runtime\Engine\Public\SceneView.h`. 
    * **FSceneViewFamily** is a set of SceneViews into a scene which only have different view transforms and owner actors.
* Then the third step is to register and initialize this class as using :

    > TSharedRef<FMyExtension,ESPMode::ThreadSafe> MyExtension;<br />
    > MyExtension = FSceneViewExtensions::NewExtension<FMyExtension>(Param1, Param2);






#### Cpp File

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
        const FIntRect ViewPort = static_cast<const FViewInfo&>(View).ViewRect; // *1
        FGlobalShaderMap* GlobalShaderMap = GetGlobalShaderMap(GMaxRHIFeatureLevel);	//GetShader for certain feature level

        RDG_EVENT_SCOPE(GraphBuilder, "CTRenderPass"); // *2

        FRHISamplerState* PointClampSampler = TStaticSamplerState<SF_Point, AM_Clamp, AM_Clamp, AM_Clamp>::GetRHI(); // *3
        Inputs.Validate();

        FScreenPassTexture SceneColor((*Inputs.SceneTextures)->SceneColorTexture, ViewPort); // *4
        
        TShaderMapRef<FCTGlobalShaderPS> CTGlobalShaderPS(GlobalShaderMap);
        FCTGlobalShaderPS::FParameters* CTGlobalShaderParameters = GraphBuilder.AllocParameters<FCTGlobalShaderPS::FParameters>(); 
        // Allocate pass parameters with GraphBuilder.AllocParameters assign all relevant RDG resourses used in the execution lambda. 

        CTGlobalShaderParameters->InputTexture = SceneColor.Texture;
        CTGlobalShaderParameters->SvPositionToInputTextureUV = (
        FScreenTransform::ChangeTextureBasisFromTo( // *5
            FScreenPassTextureViewport(SceneColor), // *6
            FScreenTransform::ETextureBasis::TexelPosition, // *7
            FScreenTransform::ETextureBasis::ViewportUV)  // *8
            * // multipy by
        FScreenTransform::ChangeTextureBasisFromTo(
            FScreenPassTextureViewport(SceneColor), 
            FScreenTransform::ETextureBasis::ViewportUV, 
            FScreenTransform::ETextureBasis::TextureUV));
        CTGlobalShaderParameters->InputSampler = PointClampSampler;
        CTGlobalShaderParameters->InputColor = TintColor;
        CTGlobalShaderParameters->RenderTargets[0] = FRenderTargetBinding(SceneColor.Texture, ERenderTargetLoadAction::ELoad); //(FRDGTexture, ELoadAction)
        
        FPixelShaderUtils::AddFullscreenPass(  // *9
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

*1 `const FIntRect ViewPort = ...` Specify the viewport rect, it is retrieved from the FSceneView View object by casting it as an FViewInfo object and retrieving the view rect:<br />

*2 `RDG_EVENT_SCOPE(GraphBuilder, "CTRenderPass");` Use **RDG_EVENT_SCOPE** to add a GPU profile scope around passes. These are consumed by external profilers like RenderDoc, as well as RDG Insights.<br />

<!-- >
    //FRDGTextureDesc OutputDesc = SceneColor.Texture->Desc;
	 //const FScreenPassRenderTarget Output(GraphBuilder.CreateTexture(OutputDesc, TEXT("CT OUTPUT RT")), ViewPort, ERenderTargetLoadAction::ELoad);
    *3 `FRDGBuilder::CreateTexture` Create a texture <br />
    *4 `OutputDesc` and render target `Output` are established for `SvPositionToInputTextureUV`. <br />
<-->
*3 Create a Point sampler. <br />

*4 Scene Color is updated incrementally through the post process pipeline: **FPostProcessingInputs** <br />
If we dont have the **FPostProcessingInputs** argument, for example in **PostRenderBasePassDeferred_RenderThread**, can alternatively use `const FSceneTextures& SceneTextures = FSceneTextures::Get(GraphBuilder);` to retrieve the scene textures.<br />

*5 `ChangeTextureBasisFromTo(TextureViewport, SrcBasis, DestBasis)`: <br />
    ![definition2](/post-img/unrealtools/add-renderpass/definition2.png)<br />

*6 The first argument:<br />
    `FScreenPassTextureViewport`: <br />
    ![definition1](/post-img/unrealtools/add-renderpass/definition1.png)<br />
    // 描述包含在纹理范围内的视图矩形。用于导出纹理坐标变换。<br />
    This will let us get `TextureViewport.Extent` and `TextureViewport.Rect`.<br />

*7 The second argument:<br />
    `ETextureBasis::TexelPosition` <br />
    ![texture-basis](/post-img/unrealtools/add-renderpass/texture-basis.png) <br />

*8 The third argument:<br />
    `ETextureBasis::ViewportUV` :point_up_2: <br />



From 5 to 8, these are aming to get a proper scale and bias value to do the transform later in shader to get a proper scene texture uv, this part of code is from postprocess shaders in Unreal such as `Engine\Source\Runtime\Renderer\Private\PostProcess\PostProcessBloomSetup.cpp`. 

I tried to understand what `ChangeTextureBasisFromTo` function does, it's comparing the last two arguments **SrcBasis** and **DestBasis**, and both of them are from a enum class **ETextureBasis**, having 4 different texture coordinate basis: <br />
> ScreenPosition, ViewportUV, TexelPosition, TextureUV <br />

These 4 texture coordinate basis have different range as in the comments above each of them (in the screenshot above), it looks like the four enums are arranged in a progressive order, then in `ChangeTextureBasisFromTo` function, by comparing the source and destination enum, it makes a corresponding calculation that returns a vector4 value composed by FVector2f Scale and FVector2f Bias. <br />
Take codes from `PostProcessBloomSetup.cpp` as example:<br />
![bloom](/post-img/unrealtools/add-renderpass/bloom.png){: width="90%" }<br />
The `Output` is the render target where pixels will draw on, and `SceneColor` has the viewportUV we want, the first *ChangeTextureBasisFromTo* turn from TexelPosition to ViewportUV, the second call turn from ViewportUV to TextureUV, multiply the two calls results together we get a scale and bias transfrom from TexelPosition to TextureUV. Then this vector4 can be used in shader. <br />


In my cpp code above, I put *SceneColor* as FScreenPassTextureViewport argument in both calls, cuz my output render target is also SceneColor. But in my particular case I know what exact coordinates I need, so I just use another simply way to get screen texture UV, it is: <br />

> CTGlobalShaderParameters->ViewportUV = FVector2f(FScreenPassTextureViewport(SceneColor).Rect.Width(),FScreenPassTextureViewport(SceneColor).Rect.Height());




*9 
`FPixelShaderUtils::AddFullscreenPass` : Dispatch a pixel shader to render graph builder with its parameters. 
![addfullscreenpass](/post-img/unrealtools/add-renderpass/addfullscreenpass.png) <br />


<br />



#### Header File

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

`MYMODULE_API` makes the entire class or functions to be callable outside of module dll. 
This header fille I took this file from a plugin named Color Correct Regions as a reference: `Engine\Plugins\Experimental\ColorCorrectRegions\Source\ColorCorrectRegions\Public\ColorCorrectRegionsSceneViewExtension.h`


<br />

<br />

### Use Custom Module

I simply call the module in an Actor c++ class which built from the engine, then drag that actor into a scene and in play mode it will be called and run.  <br />
![myactor](/post-img/unrealtools/add-renderpass/myactor.png) <br />

This is the result that shader applied on screen: <br />

![result](/post-img/unrealtools/add-renderpass/result.png){: width="80%" } <br />

If you capture in RenderDoc, the pass is under **PostProcessing** pass:  <br />
![renderdoc](/post-img/unrealtools/add-renderpass/renderdoc.png) <br />


## What's Next

So at this point I've managed to finish a frame of adding a custom render pass and custom shaders, then I can start to add more interesting stuff in my global shader to make more complex effect such as a post processing shader or a debug tool shader or etc. It will be a good start to know deeper about Unreal render pipeline and graphics API.

<br />


<br />


<br />
<!-- > References
    https://docs.unrealengine.com/5.2/en-US/render-dependency-graph-in-unreal-engine/
    https://docs.unrealengine.com/5.0/en-US/graphics-programming-overview-for-unreal-engine/
    https://blog.csdn.net/u010281174/article/details/123806725
    <br /> 
    a <span style="color: #0fc2aa">global shader</span> (shaders that are not created using the Material Editor, operate on fixed geometry) can have more advanced functionality like post-processing effects or a custom shader pass, etc. And it's able to created in a plugin which makes it easy to implement in other projects. 
<-->
