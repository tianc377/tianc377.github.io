---
layout: post
title:  "Song of the Sea Project"
date:   2022-04-01 11:00
category: shaderposts
icon: ship-fill
keywords: tag1, tag2
image: theIsland.jpg
preview: 0
youtubeId: whFFbZq_geQ
youtubeId2: 9OnVEKCszqI
---



![the third shot](/post-img/shaderposts/song-of-the-sea/theIsland.jpg){: width="800" }



1. [The Idea](#the-idea)
2. [Firstly...](#firstly)
3. [Unreal Engine Layered Material](#unreal-engine-layered-material)
    - [Structure](#structure)
    - [Utilize](#utilize)
    - [Pros and Cons](#pros-and-cons)
    - [Flexibility and Consistency](#flexibility-and-consistency)
    - [Unfortunately, More samplers](#unfortunately-more-samplers)
    - [Texture Share](#texture-share)
4. [Custom Shading Model: CTCel](#custom-shading-model-ctcel)
    - [If it's beyond the scope...](#if-its-beyond-the-scope)
    - [C++ files](#c++-files)
    - [Shader files](#shader-files)


## The Idea
:ocean:*Song of the Sea*:ocean: is one of my favorite animation films. The special art style with watercolor is so interesting and appealing, which really motivates me to make a 3D scene to build the iconic landscape of the film in the game engine. 

## Firstly... 

Firstly I downloaded the movie, watched it again, took screenshots and studied the details of the style. Basically, it is flat, overlaying multiple layers of colour, strokes, splatters, dots, and mixing randomly.

![the rock detail](/post-img/shaderposts/song-of-the-sea/theRockDetail.png){: width="400" }

I started to figure out how to make the handpainted watercolour effect of the texture, that dominates the overall impression. So here we go, Substance Designer! I did some research on stylizing textures, and videos from [Stylized Station](https://www.youtube.com/channel/UC7cmH--tFhYduIshTKzQUJQ) really inspired me and were helpful. The key step is <span style="color: #0fc2aa">slope blur</span>, which really gives a very watercolour/guache feeling. Slope blur is available in both SD and SP, and I chose SD to explore the style since it has a clearer structure.  

At the first, I planned to make the whole texture pipeline in SD, but during exploring in SD, I found it was better to mix in the engine directly. I had the experience of using UE layered material workflow for a project to make landscapes, and that approach is very flexible and efficient, which is perfect for this layering artistic style. 

![the SD blend](/post-img/shaderposts/song-of-the-sea/SDblend.jpg){: width="1000" }



![the rock detail breakdown](/post-img/shaderposts/song-of-the-sea/EgBreakDown.jpg){: width="800" }


## Unreal Engine Layered Material 
### Structure

The structure of Unreal Engine layered material is like, not lerp the effect in a base material all together but to mix it in the material instance with their provided UI, which is clear, light, and very flexible. It's like you drawing in Photoshop, with all the image layers.

1. **Base Material** with layer setting up.<br />
![Base Material](/post-img/shaderposts/song-of-the-sea/BaseMaterial.png){: width="800" }<br />

2. **Material Layer Blend asset** creating.<br />
![blend asset](/post-img/shaderposts/song-of-the-sea/MaterialLayerBlend.png){: width="800" }<br />
So this asset decides how your current layer and layer below blend together, for example, the checker map here I use, will blend the current layer in the white area, and layer/layers underneath it into the black area.

3. **Material Layer**.<br />
![material layer](/post-img/shaderposts/song-of-the-sea/MaterialLayer.png){: width="800" }<br />
Material layer asset is where you write your features and the attributes you want to output.



### Utilize

![stone MI](/post-img/shaderposts/song-of-the-sea/stoneMI.jpg){: width="800" }<br />
Take, one of my stone material instance for example, you can see there are 7 layers, each one in charge of different colors and patterns. The material layer blend asset I used most for this project is: <br />
![noise asset](/post-img/shaderposts/song-of-the-sea/noiseAsset.jpg){: width="400" }<br />
which using one noise texture as the main lerp value and with three additional mask modes, almost cover all ways of masking: <br />
![mask toggle](/post-img/shaderposts/song-of-the-sea/maskToggle.jpg){: width="400" }<br />
![blend noise](/post-img/shaderposts/song-of-the-sea/blend_noise.jpg){: width="800" }<br />
And these are my noise textures using to create those handpainted style materials, except the grunge images, others were created and exported from Substance Designer.<br />
![noise tex](/post-img/shaderposts/song-of-the-sea/noiseTex.jpg){: width="600" }<br />
Below are blend assets and layer assets I created for this project:<br />
![blend_assets](/post-img/shaderposts/song-of-the-sea/blend_assets.jpg){: width="400" }<br />
![layer_assets](/post-img/shaderposts/song-of-the-sea/layer_assets.jpg){: width="400" }<br />


### Flexibility and Consistency
For me, specifically in this project, the layered material pipeline provides more pros. The most beneficial part is flexibility and meanwhile consistency. As it allows me to layer dozens of effects with complete material tuning control, I can skip the bulky assets importing steps, and preview right away in the engine viewport. In the past studio experience, especially in two NPR projects that we build specialized shading models, consistency is always the most painful thing in the production for either character or environment artists. I had written shaders across platforms from game engines (Unity or UE), Substance Painter, Maya, 3DMax, and Marmoset, to allow artists to contain consistency on any DCC software, and that costs time to keep updating all together everytime once modification happens. <br />

### Unfortunately, More samplers
Obviously, stacking in real-time will produce more samples compared to baking texture all together. In [*Gears of War 4: Creating a Layered Material System for 60fps*](https://dl.acm.org/doi/10.1145/3084363.3085026), the team developed a material cooker that was implemented with the engine material layer system, allowing to cook the stacking material down to its simplest form. Well, what if when we just make a small project and don’t really need an extremly efficient run-time performance? Unreal provided the `Texture Share` feature.<br />

### Texture Share
[Texture Share](https://docs.unrealengine.com/5.0/en-US/texture-share-in-unreal-engine/) efficiently sends and receives GPU data between processes by bypassing the CPU and its expensive memory-copy operation by keeping the data stored in GPU memory.<br />
![textureShare2](/post-img/shaderposts/song-of-the-sea/textureShare2.jpg){: width="200" }<br />
The below image shows after I turn on the shared texture sampler, a stacking material that 4 layers using the same noise mask, whose sampler reduces from 4 to 1.<br />
![textureShare](/post-img/shaderposts/song-of-the-sea/textureShare.jpg){: width="400" }<br />






## Custom Shading Model: CTCel
### If it's beyond the scope...
I have to say the most painful experience with Unreal Engine is you cannot add a custom lighting model or custom shader like Unity does, I personally think that is very restricted for technical artists.:face_with_head_bandage: <br />
I’ve done that in Unreal Engine 4 in the previous studio, I guessed it won't be very differerent in UE5 <br />
At first, I was hesitant to create a custom shading model or not. I used a blend layer that grabs light direction from the source code into a custom node to make a NOL mask:<br />
![NoLblend](/post-img/shaderposts/song-of-the-sea/NoLblend.jpg){: width="800" }<br />
you can find these short code in `BasePassPixelShader.usf` and use it in custom code node to get the directional light direction and light color with an unlit master material.<br />
`ResolvedView.DirectionalLightDirection.xyz` <br />
`ResolvedView.DirectionalLightColor.rgb`<br />
However, in this way the object is not able to cast a shadow, like the image below, you see there is no shadow on the ground:<br />
![no shadow](/post-img/shaderposts/song-of-the-sea/no_shadow.png){: width="500" }<br />
Thus, I need to add a shading model...:triumph:


### C++ files
Below is the list of C++ and header files that you need to modify:<br />
`EngineTypes.h`
`MaterialShader.cpp`
`HLSLMaterialTranslator.cpp`
`Material.cpp`
`MaterialShared.cpp`
`ShaderMaterial.h`
`ShaderMaterialDerivedHelpers.cpp`
`ShaderGenerationUtil.cpp`

Read this article, and you will get 90% done with the new shading model: [*Unreal Engine 4 Rendering Part 6: Adding a new Shading Model*](https://medium.com/@lordned/ue4-rendering-part-6-adding-a-new-shading-model-e2972b40d72d)<br />
However, just one file not mentioned in the article by Matt Hoffman: `ShaderGenerationUtil.cpp`<br />
You need to change three places in this file:<br />
![shaderGenerationUtil1](/post-img/shaderposts/song-of-the-sea/shaderGenerationUtil.png){: width="600" }<br />
![shaderGenerationUtil2](/post-img/shaderposts/song-of-the-sea/shaderGenerationUtil(3).png){: width="600" }<br />
![shaderGenerationUtil2](/post-img/shaderposts/song-of-the-sea/shaderGenerationUtil(2).png){: width="600" }<br />
Otherwise, after build, you will see your shading model and able to choose, but it just showing solid black.<br />


### Shader files
Below is the list of shader files that you need to modify:<br />
`ShadingCommon.ush`
`Definitions.usf`
`BasePassCommon.ush`
`DeferredShadingCommon.ush`
`ShadingModelsMaterial.ush`
`ShadingModels.ush`
`DeferredLightingCommon.ush`

You can just follow Matt Hoffman's article to write the shader files. While for me, I add some additional codes to allow my new shading model available for **Translucent**.  
![maskedCTCel](/post-img/shaderposts/song-of-the-sea/maskedCTCel.png){: width="600" }<br />

{% include youtubePlayer.html id=page.youtubeId %}














{% include youtubePlayer.html id=page.youtubeId2 %}

