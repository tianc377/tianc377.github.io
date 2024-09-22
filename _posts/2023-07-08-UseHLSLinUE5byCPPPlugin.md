---
title:  "Use HLSL in UE5 by C++ Plugin"
date:   2023-07-08 09:00
category: [unrealtools]
tags: [C++, HLSL]
image: /post-img/unrealtools/shader-plugin.jpg
---

<!-- 1. [Why](#why)
2. [How To](#how-to)
    - [Create Your Plugin](#create-your-plugin)
    - [Create a Shader Folder](#created-a-shader-folder)
    - [Virtual Directory Mapping](#virtual-directory-mapping)
    - [Create a usf and ush file](#create-a-usf-and-ush-file)
    - [Include it in a custom node](#include-it-in-a-custom-node)
3. [Tips](#tips) -->

       
    

## Why

For everyone used to use code based shader like hlsl or glsl or shaderlab, it could be painful to use the node based material editor especially when using complex calculation, nodes are hard to read and organize. 
Moreover some method is not able to use in material editor such as for loop and array index. 
I am working on a texture array shader and I need to pair index of UV region to each corresponding UV tilling parameters, I really need a for loop instead of using many `if` nodes to make that work like...↓ <br />
![if-nodes](/post-img/unrealtools/shader-plugin/if-nodes.png){: width="30%" } <br />

In addition, a <span style="color: #0fc2aa">global shader</span> (shaders that are not created using the Material Editor, operate on fixed geometry) can have more advanced functionality like post-processing effects or a custom shader pass, etc. And it's able to created in a plugin which makes it easy to implement in other projects.

In this post I'am using a rather easy way to include the shader file in custom node in the Material Editor to use the function in the shader file. This could be a good start to get to know how to setup full global shader later.

## How to
### Create Your Plugin

First I created a plugin from the engine. <br />
![create-plugin](/post-img/unrealtools/shader-plugin/create-plugin.jpg){: width="950px" } <br />

Unreal will create a directory for you with a directory comes with C++ files. <br />
![blank-plugin](/post-img/unrealtools/shader-plugin/blank-plugin.png) <br />


### Create a Shader Folder
Afterwards, create a shader folder named `Shaders`. Then your directory structure will looks like this ↓ <br />
![shader-folder](/post-img/unrealtools/shader-plugin/shader-folder.png) <br />

### Virtual Directory Mapping

*The source files of any new shaders created need to be stored in the `Engine/Shaders`  folder. If a shader is part of a plugin, it should be stored in the `Plugin/Shaders`  folder instead.*

Unreal actually sets the directory where you need to save your sourece shader files.<br />
In this case there is a function named `AddShaderSourceDirectoryMapping`:<br />
This function maps a real shader directory existing on disk to a virtual shader directory<br />
![virtual-dir-mapping](/post-img/unrealtools/shader-plugin/virtual-dir-mapping.png) <br />
If I search this function, it is defined in `ShaderCore.h`:<br />
![rg-search](/post-img/unrealtools/shader-plugin/rg-search.png){: width="550px" } <br />
`ShaderCore.h` is a headerfile from `RenderCore` module, therefore to use it we need to append a module in the `.Build.cs` file in your plugin folder. <br />
![build-cs](/post-img/unrealtools/shader-plugin/build-cs.png) <br />
![add-module](/post-img/unrealtools/shader-plugin/add-module.png){: width="950px" } <br />
I added the `RenderCore` module as well as the `Projects` module which we need for the `IPluginManager` class, where we can use `IPluginManager::FindPlugin` and use `IPlugin::GetBaseDir` to get the plugin dir, therefore get the `Shaders` folder. <br />
In `YourPluginFolder/Sourece/Private/YourPluginName.cpp`, adding codes like below:<br />
![cpp-file](/post-img/unrealtools/shader-plugin/cpp-file.png)<br />

At this point if you can have a successful build then can create the shader file.<br />

### Create a usf and ush file
In your `Shaders` folder, created a text file and rename the extension to `.usf`. <br />
If you want have syntax highlight and hlsl extension option in Visual Studio, what I did is to download an extension: <br />
![hlsl-vs](/post-img/unrealtools/shader-plugin/hlsl-vs.png)<br />

I'm using this usf file as a custom node in the material editor, so that I can use for loop, but I wanna create a lib for my shader functions for future use, it won't work if I write the function directly in the usf and include it in the custom node, because this will be a function inside another function that HLSL not allows:<br />
![function-error](/post-img/unrealtools/shader-plugin/function-erro.png)<br />
Therefore I nested the function inside a struct in a `ush` header file: <br />
![ush-file](/post-img/unrealtools/shader-plugin/ush-file.png)<br />
Afterwards it's good to go for the `usf` file, call that function:<br />
![usf-file](/post-img/unrealtools/shader-plugin/usf-file.png)<br />
Notice that the `#include` hearder file path is the virtual path.<br />

### Include it in a custom node
Finally, press F5 to build and open the project, in custom node I included the `usf` file and give corresponding input parameters:<br />
![custom-node](/post-img/unrealtools/shader-plugin/custom-node.png)<br />

Then it just works perfectly, and free to add more HLSL functions into the header file.<br /> 

<!--or etc.).-->


## Tips

*Enable `r.ShaderDevelopmentMode` by setting it to 1. The easiest way is to edit `ConsoleVariables.ini` (in YourEngineFolder/Engine/Config) so it is done every time you load. This will enable retry-on-error and shader development related logs and warnings.*<br /> 

I also recommend `r.DumpShaderDebugInfo = 1`, this will save temp combined shader files to `YourProject\Saved\ShaderDebugInfo\...` <br /> 


<br /> 

<br /> 

<br /> 