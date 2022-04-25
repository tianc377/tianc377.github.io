---
layout: post
title:  "Plant Grow Shader"
date:   2022-03-03 12:00
category: shaderposts
icon: flower-pink
keywords: tag1, tag2
image: plant-grow.gif
preview: 1
youtubeId: whFFbZq_geQ
youtubeId2: 9OnVEKCszqI
---

1. [The Task](#the-task)
2. [Houdini Pivot Painter](#houdini-pivot-painter)
    - [HDA](#hda)
    - [Index textures](#index-textures)  
3. [Vertex Color Processing](#vertex-color-processing)
    - [Maya Python](#maya-python)
4. [The Grow Material in Engine](#the-grow-material-in-engine)
    - [Individual Grow](#individual-grow)
    - [Spreading Grow](#spreading-grow)
    - [The Glittering](#the-glittering)


## The Task
This plant grow material was for a video project of PUBGM, it needed flowers to grow from leaves to leaves and with a controllable spreading direction.  

## Houdini Pivot Painter 

[Pivot Painter Tool 2.0](https://docs.unrealengine.com/4.27/en-US/AnimatingObjects/PivotPainter/PivotPainter2/) provided by Unreal is a MAX tool that you can use to make textures that store the pivot and rotation information, which you can use in the engine with shader, to either scale or morph. <br />
### HDA
While for this project, I used a houdini HDA that was developed by a houdini team, with custom UI, originally aimed at tree leaves wind effect. It is able to generate two textures, one is`x-axis and x-extent texture`, another one is `pivot and parent index texture`, shown below. <br />
### Index textures
![HDA-textures](/post-img/shaderposts/PUBGM/HDA-textures.png){: width="70%" }<br />
The `x-axis and x-extent texture` is for wind effect, which, the strength of wind will be affected by this texture, from the left to right, the bending degree will be different.

`pivot and parent index texture` is the key for scaling with hierarchy, the engine allows 3 levels scaling, it can separate the flower from stems to leaves and petals, and locate the pivot of every parts at the correct position. <br />

The HDA processing the mesh with several steps:
![HDA-steps](/post-img/shaderposts/PUBGM/HDA-steps.png){: width="100%" }<br />
1. Seperate trunk(stems) and leaves based on selection of materials.
2. Give the trunks different levels.
3. Separate the leaves.
4. Give leaves different lables, greens are able to rotate while reds are not.
After the processing, it's ready to generate the textures.


## Vertex Color Processing
### Maya Python
For the growing, a depth texture is also required, while I use vertex color to deal with this step.

To paint vertex color by hand won't be realistic, and also not accurate. The depth distinguishing depends on the height and also the layering (inner or outside) of the plant, that basically is the distance of every leaf/petal from the (0,0,0) point.  <br /> 

In this particular situation, I need to get every piece of the mesh based on the unit of leaf/petal, and then to make each vertice in the same leaf/petal has a uniform vertex color. If I just iterate based on each vertices' distance to the zero point, it will generate a very smooth gradient from bottom to top<br />
![boundingBoxZ](/post-img/shaderposts/PUBGM/boundingBoxZ.jpg){: width="20%" } <br />
that will cause the mesh growing with bending and a clear cutting line instead of growing each by each.  <br />

Theoretically, the `separate mesh` operation in Maya can achieve this goal, however, the mesh’s vertices index were processed in houdini, it will be disrupted if I separate the mesh. <br />

I get the max distance of all the vertices first, remap it as 1, since the color value is from 0-1, over that or below that will not work. <br />
![maya-paintVC](/post-img/shaderposts/PUBGM/maya-paintVC.gif){: width="80%" }<br />


After exporting, the mesh is ready for importing into the engine.


## The Grow Material in Engine
### Individual Grow
![nodes-1](/post-img/shaderposts/PUBGM/nodes-1.jpg){: width="100%" }<br />
Use the node `PivotPainter2_DecodePosition` provided by Unreal Engine to decode the pivot index texture. Then apply the vertex color on top of that. <br />
![white-flower](/post-img/shaderposts/PUBGM/white-flower.gif){: width="40%" }<br />

![wind](/post-img/shaderposts/PUBGM/wind.jpg){: width="100%" }<br />
For the wind part, you can download UE4's content examples ![content-examples](/post-img/shaderposts/PUBGM/content-examples.jpg){: width="10%" }, and find the demo examples of pivot painter 2.0, there is a foliage animation material that shows how to use the `x-axis and x-extent texture` to make wind effect, just copy it and make a function in your own project. <br />
![wind-flower](/post-img/shaderposts/PUBGM/wind-flower.gif){: width="40%" }<br />

### Spreading Grow
The shot of the film was designed as the flowers grow and spread following three light balls flying through a path. So it needs my foliages to grow with overall spreading range, direction and speed. 
The flower were be painted by foliage system, and the scale of the film map is very large. I use world position's XY channels to form a gradient mask below:
![landscape](/post-img/shaderposts/PUBGM/landscape.jpg){: width="80%" }<br />
In the shader, the mask can be rotate by lerping between X and Y in order to change the direction of the spreading. Also plug the parameters with material parameter collection asset so that I can control and key it in sequencer. 
![spreading-mask](/post-img/shaderposts/PUBGM/spreading-mask.jpg){: width="100%" }<br />

### The Glittering
The director asked for a magical representation to emphasize the growing process. As the growing and spreading effect is totally based on vertex animation in shader, there is no actual position information existing, which means it is hard for VFX artists to make glow particles that need to track the flowers. So I was thinking if making glitters directly on the flower shader is possible, as the lens is moving fast, it might create acceptable magical sparkles and glows. 

So I make manipulate on the spreading mask, to let it only mask the front line of the growing foliage, and add glow noises:<br />
![sparkles2](/post-img/shaderposts/PUBGM/sparkles2.jpg){: width="80%" }<br />
![sparkles](/post-img/shaderposts/PUBGM/sparkles.jpg){: width="80%" }<br />

The output:
![spreading](/post-img/shaderposts/PUBGM/spreading.gif){: width="100%" }<br />
