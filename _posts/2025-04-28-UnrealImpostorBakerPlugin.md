---
title:  "Unreal Impostor Baker Plugin"
date:   2025-04-28 21:00
categories: [unreal]
tags: [plugin, Unreal]
image: /post-img/unrealtools/impostor-baker/cover-impostor.png
published: false
---

## Intro 

Unreal官方有一个插件是用来制作植被impostor mesh的，这个插件全部使用blueprint和Editor Utility Widget，是一个很好的学习的案例。
这个插件在UE5.5以及更高版本才有完整的功能，我一开始用5.4打开它，没有所需的widget。
    

https://www.shaderbits.com/blog/octahedral-impostors

Dithering, Pixel Depth Offset and then Parallax Occlusion Mapping are progressively enabled in the video.
https://www.youtube.com/watch?v=JOL5e-J1btA&t=26s
<br />


## Generate Impostor Sprites and Materials

![how-to-bake](/post-img/unrealtools/impostor-baker/how-to-bake.png){: width="80%"} <br />

## Editor Utility Blueprint


choose parent class Asset Action Utitlity: 可以用纯蓝图（无代码、无 Python）来添加资源右键菜单。
![aau](/post-img/unrealtools/impostor-baker/asset-action-utility.png){: width="80%"} <br />
选这个才会在右键菜单中出现函数按钮。
![rightclick](/post-img/unrealtools/impostor-baker/right-click-menu.png){: width="80%"} <br />

add an input for the SetStaticMesh function, 这个函数会在assetAction里面使用，将selectedAsset传递给这个SetStaticMesh，然后它会将收到的asset cast为staticmesh然后设置给刚创建的variable StaticMesh。
![set-static-mesh](/post-img/unrealtools/impostor-baker/set-static-mesh.png){: width="80%"} <br />

对于Details View 这个Palette，想要让它只显示需要的variables，就要自己定义，并将新创建的category和details view里的category对应： 
![variable-cate](/post-img/unrealtools/impostor-baker/variable-cate.png){: width="90%"} <br />


  
Spawn and Register Tab node: 是一个 Editor Utility Widget 蓝图节点，用于在编辑器中动态创建并显示一个新的选项卡（Tab）面板。

Cast To 是为了把“通用 Editor 面板实例”转换成你自己定义的那个 Widget 类，这样你才能访问里面的变量和逻辑。


Actor类的蓝图可以添加Components：
![bp-actor](/post-img/unrealtools/impostor-baker/bp-actor.png){: width="80%"} <br />
