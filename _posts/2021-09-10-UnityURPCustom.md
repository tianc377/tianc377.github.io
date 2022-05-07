---
layout: post
title:  "Unity URP Custom Pipeline"
date:   2021-09-10 09:00
category: shaderposts
icon: subtask
keywords: tag1, tag2
image: unity-urp.jpg
preview: 1
youtubeId: vWOHPEhF99M
youtubeId2: atztUZg_GiY
youtubeId3: 9DwjalmyKbE
---

1. [Unity URP](#unity-urp)
    - [URP Shader Structure](#urp-shader-structure)
        - [.shader](#shader)
        - [_Inputs.hlsl](#_inputshlsl)
        - [_ForwardPass.hlsl](#_forwardpasshlsl)
2. [Custom Lit Shader](#custom-lit-shader)
    - [Custom Rim Light](#custom-rim-light)
3. [Custom Unlit Shader for Particles](#custom-unlit-shader-for-particles)
4. [URP Renderer Feature](#urp-renderer-feature)

## Unity URP 
[Unity URP](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@11.0/manual/index.html)
stands for Unity Universal Render Pipeline, which provided scriptable structures that can be easily customized. Also, it comes with the SRP Batcher draw call optimization that is very suitable for mobile game projects, which the Built-in Render Pipeline does not support. 

### URP Shader Structure
URP comes with some default shaders, and different with Cg, it using HLSL as the shader language. Usually, a valid shader comes with three files:
- `xxx.shader`
- `xxx_Inputs.hlsl`
- `xxx_ForwardPass.hlsl`

#### .shader
Where, `xxx.shader` is like the Cg shader file's part that list all the properties:<br />
![properties](/post-img/shaderposts/unity-urp/properties.jpg){: width="60%" }<br />
and subshader with passes, tags, pragmas, defines and includes:<br />
![subshader](/post-img/shaderposts/unity-urp/subshader.jpg){: width="50%" }<br />

#### _Inputs.hlsl
`xxx_Inputs.hlsl` is the file you state your variables and its data types. They should under the `CBUFFER_START(UnityPerMaterial)` macro otherwise the shader won't be compatible with the SRP Batcher: <br />
![inputs](/post-img/shaderposts/unity-urp/inputs.jpg){: width="60%" }<br />
You can also put the functions you want to use in this file.<br />

#### _ForwardPass.hlsl
`xxx_ForwardPass.hlsl` is the main part of your shader. It contains struct, vertex shader and fragment shader:<br />
![forwardpass](/post-img/shaderposts/unity-urp/forwardpass.jpg){: width="60%" }<br />

For syntax such as data types, samplers, math functions, etc., just search for DirectX HLSL syntax. <br />


## Custom Lit Shader
In *Under One Person* project, I made some modifications on the default lit shader for the environment artists. See the material inspector, I added custom selections to differentiate the surface type (snowy surface, damp surface, or the surface needs rim light), as well as the control of the original specular intensity and environment reflection intensity:  <br />
![scene-lit-exp](/post-img/shaderposts/unity-urp/scene-lit-exp.jpg){: width="100%" }<br />

So, if you want to revise the lighting behavior like I did, go to this lighting file:`#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"`<br />
If you need to revise the shaderGUI of the default shader like `Lit.shader`, you need to go to `Packages\com.unity.render-pipelines.universal\Editor\ShaderGUI\` directories to revise the shaderGUI C# files to make it work.

### Custom Rim Light
For the custom rim light, it's a request to achieve the clear cut edge handpainted rim light, for example: ![rimlight-eg](/post-img/shaderposts/unity-urp/rimlight-eg.jpg){: width="30%" }<br />
To achieve this, 3D artists use the tool developed in Maya to store the hard edge normal into vertex color RGB, and paint the rim mask and store it into vertex color Alpha channel. In the Lit shader I grab the vertex color and transform it into normal vectors:<br />

{% highlight hlsl %}
#if _USERIM
    output.color = input.color;
    //output.rimData.xyz = max(input.color.r, max(input.color.g, input.color.b)) > 0.001 ? TransformObjectToWorldNormal((input.color.rgb*2-1)) : normalInput.normalWS;
    output.rimData.xyz = TransformObjectToWorldNormal((input.color.rgb*2-1));
    output.rimData.w = 1 - saturate(distance(GetCameraPositionWS(), vertexInput.positionWS) / _rimFadeFar);
#endif
    ....
    //then apply it after half4 color = UniversalFragmentPBR(inputData, surfaceData);
SET_RIM_COLOR(color.rgb, input.rimData, inputData.normalWS, inputData.viewDirectionWS, inputData.positionWS, inputData.bakedGI, input.color.a, rimParam, _rimNormalMix);
{% endhighlight %}

I also developed corresponding maya shader and SP shader for out-source artists to create the assets without using engine to verify the output:<br />
![rim-sp](/post-img/shaderposts/unity-urp/rim-sp.png){: width="70%" }<br />
![rim-maya](/post-img/shaderposts/unity-urp/rim-maya.jpg){: width="80%" }<br />



## Custom Unlit Shader for Particles
![particle-unlit](/post-img/shaderposts/unity-urp/particle-unlit.jpg){: width="100%" }<br /> 
To ensure that the Particles/Unlit shader is able to use commonly in different situation, I added custom features based on VFX artists' request on the default particle shader. Features are such as distortion, UV rotation, UV flow, dissolve, ramp map creating, mask map, flow map, vertex distortion, etc. Also, for particles, as each particle will have self random value, that's why it needs to use Custom Data of the particle system as the dynamic parameters to plug into the shader, otherwise every particle will show same behavior at the same time. <br /> 
![custom-data](/post-img/shaderposts/unity-urp/customdata.jpg){: width="40%" }<br />

## URP Renderer Feature
A Renderer Feature is an asset that lets you add extra Render passes to a URP Renderer and configure their behavior. For example in the project, I added render feature to make red and blue channel pixels shifting for some fighting moments to increase the feeling of fast actions: <br />
![pixel-shift](/post-img/shaderposts/unity-urp/pixel-shift.jpg){: width="50%" }<br />
Rim light created through the traditional Fresnel effect has lots of problems, such as cannot precisely control the width and range in some perspectives, like the image shows:
![fresnel](/post-img/shaderposts/unity-urp/fresnel.png){: width="40%" }<br />
Therefore I wrote a custom render feature to get a pure white buffer that silhouetted the character figure:<br />
![buffer](/post-img/shaderposts/unity-urp/newbuffer.png){: width="70%" }<br />
then pass that buffer to my shader, then I could manipulated the screen uv with the influence of normal, view direction and pass that sample point to that texture, to get a constant, evenly rim light that will not appear in undesired area:<br />
![rimlight](/post-img/shaderposts/unity-urp/rimlight.gif){: width="70%" }<br />

