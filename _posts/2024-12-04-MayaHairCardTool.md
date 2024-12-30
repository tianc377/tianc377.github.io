---
title:  "Maya Hair Card Tool"
date:   2024-12-04 21:00
category: [mayatools]
tags: [Maya, Python]
image: /post-img/mayatools/hair-card-tool.png
published: true
---


{% include embed/youtube.html id='KVyUu94NJuU' %}


## Intro
Recently, I encountered a requirement to develop a tool for converting HairStrands to HairCards. Unreal Engine already has a robust built-in feature for this, which works impressively well. However, we don’t use Unreal Engine, and developing a similar feature from scratch could take the development team up to a year. The prototype tool I’m tasked with creating focuses more on improving workflow efficiency, reducing manual adjustments, and streamlining the process. It’s not designed to be a magical, one-click solution for perfect hair cards but rather a helpful aid to optimize certain repetitive steps in the conversion workflow.

<br />

<!-- ![capture](/post-img/shaderposts/landscape-material/HighresScreenshot_2024.12.01-16.20.38.png){: width="100%" .shadow} -->


## DCC Choice

Currently, artists are using Maya's XGen to create HairStrands. While similar tools can be developed in Blender or Houdini, the workflow becomes cumbersome, requiring frequent switching and conversion between software. Therefore, I decided to develop this tool directly within Maya to ensure a smoother and more integrated workflow. Additionally, Maya’s Python API is robust and comprehensive, offering almost all the functionalities needed for geometric operations, making it an ideal choice for this task.

Maya has a built-in tool, previously part of the Bonus Tools and now located in a window named "Sweep," which allows converting curves into polygons with adjustable properties like quad count, width, and rotation. However, it has a significant limitation: when working with multiple curves, you can either create a single Sweep node for all curves, causing them to share attributes, or generate individual Sweep nodes for each. The latter option doesn't allow multi-selection or batch adjustments, making tweaking very inefficient.

So, part of my tool's functionality is integrating Maya's Sweep feature in a way that allows users to select multiple polygons and adjust them collectively through a single control panel, simplifying the process.


## Convert Strands to Curves

The first step is to retrieve the required curves. Maya XGen has a built-in feature that allows you to select a percentage of the desired strands. This process generates a lightweight MEL script containing all the point coordinate data of the curves. 
![get-curves](/post-img/mayatools/hair-card-tool/create_mel.jpg){: width="80%" .shadow}
Running this script in Maya will recreate the curves directly in the scene.
![get-curves](/post-img/mayatools/hair-card-tool/mel_file.jpg){: width="80%" .shadow}

## Filter Curves
![curves-selection](/post-img/mayatools/hair-card-tool/curves_selection.gif){: width="100%" .shadow}

The first 

## Rotation Solutions

TODO

## Live Attach

TODO

## Move Towards Center
TODO

## UV Editor
### QPainter
TODO

## WorkspaceControl

TODO

## Undo Chunk Customization

TODO

## PYD Conversion