---
layout: post
title:  "Stylized Character Rendering"
date:   2021-10-01 18:00
category: shaderposts
icon: brush-line
keywords: tag1, tag2
image: cel.jpg
preview: 1
youtubeId: vWOHPEhF99M
youtubeId2: atztUZg_GiY
youtubeId3: 9DwjalmyKbE
---

1. [Ace Force and Under One Person Projects](#ace-force-and-under-one-person-projects)
2. [Cel Pipeline](#cel-pipeline)
    - [Shadow](#shadow)
        - [Face Shadow by Normal](#face-shadow-by-normal)
        - [Face Shadow by Map](#face-shadow-by-map)
        - [Shadow Color](#shadow-color)
        - [Layer of Shadow](#layer-of-shadow)
        - [Exquisite Shadow](#exquisite-shadow)
        - [Shadow Hatching line](#shadow-hatching-line)
        - [Overflow Color](#overflow-color)
    - [Specular](#specular)
        - [Cel Specular](#cel-specular)
        - [GGX Mobile Specular](#ggx-mobile-specular)
        - [Hair Specular](#hair-specular)
    - [Dynamic Lights Integration](#dynamic-lights-integration)
        - [In UE4](#in-ue4)
        - [In Unity](#in-unity)
    - [Sky Diffuse Lighting](#sky-diffuse-lighting)
    - [The Outline](#the-outline)
        - [UE4 Outline](#ue4-outline)
        - [Unity Outline](#unity-outline)
    - [The Innerline](#the-innerline)
        - [The Fixed Innerline](#the-fixed-innerline)
        - [The Dynamic Innerline](#the-dynamic-innerline)
3. [Other Explorations](#other-explorations)
    

## Ace Force and Under One Person Projects
In *Ace Force* project and *Under One Person* project, I've been mainly in charge of the character shading. The character rendering pipeline developed from Unity to Unreal Engine, from pure cel style to half cel half PBR, with many explorations of the possibilities of stylization.<br />
![cel](/post-img/shaderposts/character-shading/cel.jpg){: width="100%" }<br />



## Cel Pipeline
### Shadow
![caro-ratio](/post-img/shaderposts/character-shading/caro-ratio.png){: width="30%" } <br />
Like every cel shading, the most important step is to make division on light and shadow area. We used `half lambert NoL` to calculate the ratio. The shape of the black white division is up to the mesh's topology, so it's strict for 3D artists when modelling. <br />

Well, the shadow does not always follow the light. Some particular areas such as under eyelids, under bangs, and under some narrow edges that can emphasize the structures of mesh, will be painted a certain grey value as a channel of a texture, and applied in the calculation of NoL, so that those area will always be in shadow. <br />
#### Face Shadow by Normal
While for parts need dynamic shadow, such as face, where you can't make it fixed, but meanwhile want to control the shadow shape, 3D artists manipulate the **normal direction** to get a smooth transition when light direction changes.<br />
![luca-face](/post-img/shaderposts/character-shading/luca-face.jpg){: width="30%" } ![luca-face-mesh](/post-img/shaderposts/character-shading/luca-face-mesh.jpg){: width="30%" } ![normal-dir](/post-img/shaderposts/character-shading/normal-dir.jpg){: width="30%" }<br />
#### Face Shadow by Map
Later on, in another project *Under One Person II*, we used the gradient map to make the face shadow, which was firstly created by **Genshin Impact**. 
This method uses a face mask map that contains two channels, R and G:<br />
![face-mask-map](/post-img/shaderposts/character-shading/face-mask-map.jpg){: width="20%" } ![face-r](/post-img/shaderposts/character-shading/face-r.jpg){: width="20%" } ![face-g](/post-img/shaderposts/character-shading/face-g.jpg){: width="20%" }<br />
![face-shadow-red](/post-img/shaderposts/character-shading/faceshadow-red.gif){: width="80%" }<br />
As the above image shows, the third pic is a value calculated the angle between the light vector and the starting vector and turns the angle to a radian value from 0-1. The middle pic is the R and G channel of the face mask stepping by the third value, so that it alternates once the light direction is parallel to the forward direction of the face.  <br />
The advantage of this method is the transition is very smooth, and artists can paint grey scales to control the desired light and shadow area.<br />
#### Shadow Color
For shadow area, we have a shadow color map that can multiplied on the basecolor map to make a hueshift on shadow area to mimic the manga aesthetics.<br />
![caro-dark](/post-img/shaderposts/character-shading/caro-dark.jpg){: width="30%" }<br />
#### Layer of Shadow
In later development, we also tried second layer of shadow to create volume effect <br />:
![second-shadow](/post-img/shaderposts/character-shading/second-shadow.gif){: width="50%" }<br />
#### Exquisite Shadow
Like I mentioned above, some self-shadowing such as bangs shadow cast on the face is fixed shadow painted on a textures. While for cinematic need, I made post-process material with custom stencil to create dynamic exquisite shadow of the bangs, for some close-up shots: <br />
![face-shadow](/post-img/shaderposts/character-shading/face-shadow.gif){: width="50%" }<br />
#### Shadow Hatching line
To expolre more on the stylization, I also tried handpainted hatching line effect on the character.<br />
![wy-hatching](/post-img/shaderposts/character-shading/hatching.gif){: width="70%" }<br />
![pistol-hatching](/post-img/shaderposts/character-shading/pistol-hatching.gif){: width="50%" }<br />
As well as a post-process hatching material:<br />
![scene-hatching](/post-img/shaderposts/character-shading/scene-hatching.png){: width="80%" }<br />

#### Overflow Color
This effect is the bounline between light and shadow area, like shown in the pic below<br /> ![overflow-eg](/post-img/shaderposts/character-shading/overflow-eg.jpg){: width="30%" }<br />
In the character shading, I used a `abs()` for the shifted ratio of shadow and light, then provided parameters to control the range, hardness and color. <br />
![overflow](/post-img/shaderposts/character-shading/overflow.png){: width="40%" }  ![overflow-luca](/post-img/shaderposts/character-shading/overflow-luca.jpg){: width="40%" }<br />


### Specular
The specular style has changed a lot, from pure cel hard edge specular to PBR GGX specular. 
#### Cel Specular
![cel-specular](/post-img/shaderposts/character-shading/cel-spec.jpg){: width="30%" }<br />
This character's hair is a typical manga style specular that calculated by `NoH` and with the influence of a texture that controls the range, the shape, the position of the specular. <br />
#### GGX Mobile Specular
Later on, in order to emphasize the realistic feeling of firearms, since it's an FPS game, we make the firearms merge with `GGX_Mobile` specular, while still keeps cel shadow on that (but make the edge soft gradient to merge in).  <br />
![M416](/post-img/shaderposts/character-shading/M416.gif){: width="70%" }<br />
#### Hair Specular
As I showed above, the old version of hair specular is pure hand-painted and controlled by textures. While in the new version, the hair specular revised to kajiya-kay anisotropy specular calculation. <br />
![hair-silver](/post-img/shaderposts/character-shading/hair-silver.gif){: width="40%" }<br />  
![anisotrophy](/post-img/shaderposts/character-shading/anisotropy.png){: width="40%" }{: width="90%" }<br />
<span style="font-size:0.8em;">Hair anisotropy specular exploration in <b>Blender</b>.</span><br /> 
As well as hair shading in another project *Under One Person* in **Unity**:<br /> 
![uc-hair](/post-img/shaderposts/character-shading/uc-hair.gif){: width="70%" }<br />  


### Dynamic Lights Integration
#### In UE4
For `Ace Force` project in UE4, dynamic lights integration was revised in `MobileBasePassPixelShader.usf`, as the cel-shading style needs the corresponding cel feeling attenuation such as point lights.<br />
![dynamiclight](/post-img/shaderposts/character-shading/dynamic-light.gif){: width="90%" }<br />  
#### In Unity
For `Under One Person` project, the project is using Unity URP pipeline, the character shader was un unlit shader originally, therefore how to receive and react to the scene lighting in this 3D game with full point of view is a significant problem. I cannot use realistic light attenuation calculation that comes with Unity URP, since that will create a realistic and volumetric effect on the flat cartoon character, which is not what I want to see:<br />  
![soft-attenuation](/post-img/shaderposts/character-shading/soft-attenuation.jpg){: width="30%" }<br />  
I also need to consider how to make the point lights combine with the character directional light that determined the light and shadow division on the character, therefore I need the lights attenuating with distance, but distributing evenly on the character. Below is the outcome.
{% include youtubePlayer.html id=page.youtubeId %}
[Distance Attenuation Cel Dynamic Lights In Unity](https://youtu.be/vWOHPEhF99M)<br />

### Sky Diffuse Lighting
In new version of `Ace Force` character shading, after increasing the volume of the shading, and adding GGX specular, we made another experiment on the shadow color, to merge with sky diffuse lighting and color. Below is a concept, see the blue diffuse light in the shadow, what's what the director wants to have:<br />
![luca-concept](/post-img/shaderposts/character-shading/luca-concept.png){: width="60%" }<br />  
So, I went for the `MobileBasePassPixelShader.usf` agian, looking for the calculation of the indirectIrradiance, took it out and used it as a lerp value, exposed two color parameter for the artists to control, one indicates sky color, another is ground color. <br /> 
![sky-diffuse-color](/post-img/shaderposts/character-shading/sky-diffuse-color.png){: width="30%" }<br />  
Then applied it on the dark area:<br />  
![Luca-skydiffuse](/post-img/shaderposts/character-shading/Luca-skydiffuse.jpg){: width="100%" }<br />  
![luca-compare](/post-img/shaderposts/character-shading/luca-comp.jpg){: width="70%" }<br />

### The Outline
#### UE4 Outline
In ue4 the outline is just make another copy of mesh, and give it an outline material. For the outline material, artists could use vertex color to control the different thickness of lines in different area.

#### Unity Outline
In unity the outline is to add a new pass, and cull front. Same, artists could use vertex color to control the different thickness of lines in different area.

### The Innerline
#### The Fixed Innerline
![line-tex](/post-img/shaderposts/character-shading/line-tex.jpg){: width="40%" }<br />
This line tex is multiplied on the final color, to make some inner line details such as expression lines on face or folds lines on clothing. 

#### The Dynamic Innerline
For `Under One Person` project, the main character's clothing is loose and soft. Besides, this is a first person view game, I tried to use a method based on camera distance to show and hide the fixed innerline to create a dynamic effect:<br />
![innerline-gif2](/post-img/shaderposts/character-shading/innerline-gif2.gif){: width="70%" }<br />
<span style="font-size:0.8em;">The blue part is the mask out area based on cam distance.</span>

![innerline-gif](/post-img/shaderposts/character-shading/innerline-gif.gif){: width="70%" }<br />


## Other Explorations
![luca-silver](/post-img/shaderposts/character-shading/luca-silver.png){: width="70%" }<br />

![nb-cg](/post-img/shaderposts/character-shading/nbcg-1.jpg){: width="90%" }<br />
<span style="font-size:0.8em;">Cel shading in official engine without adding new shading model</span>

![nb-cg](/post-img/shaderposts/character-shading/nbcg-2.png){: width="80%" }<br />
![nb-cg](/post-img/shaderposts/character-shading/nbcg-3.png){: width="90%" }<br />
<span style="font-size:0.8em;">half cel half PBR shading in official engine without adding new shading model</span>

![long](/post-img/shaderposts/character-shading/long.jpg){: width="100%" }<br />
<span style="font-size:0.8em;">new version of character shading</span>

![silver](/post-img/shaderposts/character-shading/silver-blender.png){: width="60%" }<br />
<span style="font-size:0.8em;">Style exploration in **Blender**</span>


{% include youtubePlayer.html id=page.youtubeId2 %}
[PBR Character Style Exploration in Blender](https://youtu.be/atztUZg_GiY)<br />

![wy-blender](/post-img/shaderposts/character-shading/wy-blender.png){: width="60%" }<br />
<span style="font-size:0.8em;">Style exploration in **Blender**</span>

{% include youtubePlayer.html id=page.youtubeId3 %}
[PBR Merged Cel Shading Character](https://youtu.be/9DwjalmyKbE)




<!--or this toon-style shader with PBR material tendency, we combined PBR lighting features (such as normal map, dynamic lighting, reflections, overflow color at the shadow line, sky light illumination etc.) with the stylish cartoon features (innerlines and outlines, cartoon shadow boundary, manipulated shadow color, customized sky light coloring etc.).-->
