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

<!-- ## Main Tab
![main-tab](/post-img/mayatools/hair-card-tool/main_tab.jpg){: width="80%" .shadow}
 -->

## Convert Strands to Curves

The first step is to retrieve the required curves. Maya XGen has a built-in feature that allows you to select a percentage of the desired strands. This process generates a lightweight MEL script containing all the point coordinate data of the curves. 
![get-curves](/post-img/mayatools/hair-card-tool/create_mel.jpg){: width="80%" .shadow}
Running this script in Maya will recreate the curves directly in the scene.
![get-curves](/post-img/mayatools/hair-card-tool/mel_file.jpg){: width="80%" .shadow}

## Filter Curves
![curves-filters](/post-img/mayatools/hair-card-tool/curve_filter.jpg){: width="90%" .shadow}

![curves-selection](/post-img/mayatools/hair-card-tool/curve_filter.gif){: width="100%" .shadow}

Once the curves are prepared, the first essential feature is curve filtering. I implemented a slider that allows users to select a percentage of unwanted curves—higher percentages result in more curves being selected. These selected curves can then be hidden using a `Hide Filtered Curves` button.
To make the filtering more controllable—for example, the top of the head typically requires denser hair—you can paint red vertex colors in the desired areas. Curves falling within the vertex color regions won't be selected or hidden, ensuring higher density, helping avoid exposing the scalp.

The `Mask Opacity` setting is used to adjust the transparency of the vertex color mask. If set to less than 1.0, curves within the vertex color coverage area may still be partially selected and hidden, allowing for finer control over which curves are kept visible and which are hidden, even in the designated areas.

Below is the function code for **randomly selecting curves** based on the percentage and checking if they are within **the vertex color mask range**:

```python
    # Get those faces of head mesh that are painted 
    def get_colored_face_dict(self):
        head_dp = om.MSelectionList().add(self.sel_head).getDagPath(0)

        colored_face_dict = {}
        head_face_len = cmd.polyEvaluate(self.sel_head, face = True)
        mesh = om.MFnMesh(head_dp)
        sourceColors = om.MColor()
        vert_colors = mesh.getVertexColors('colorSet1', sourceColors)
        m_vert_iter = om.MItMeshVertex(head_dp)
        for vert in m_vert_iter:
            if vert_colors[vert.index()][0]> 0.0:
                find_faces = vert.getConnectedFaces()
                for f in find_faces:
                    if f not in colored_face_dict.keys():
                        colored_face_dict.update({f : vert_colors[vert.index()][0]})
        return colored_face_dict
    
    # Select curves randomly based on a percentage 
    def random_curve_selection(self, input):

        try:
            head_dp = om.MSelectionList().add(self.sel_head).getDagPath(0)
        except:
            cmd.warning( "No Head Selected" )
            return

        # sel_curves = cmd.ls(long=True, selection=True)
        if not self.lock_curves:
            cmd.warning( "No Curves Selected" )
            return
       
        cmd.select(self.lock_curves)
        curve_visible_list = cmd.filterExpand(sm = 9) # filter only curves
        colored_face_dict = self.get_colored_face_dict()
        set_reduce_percent = input/100 #40
        reduce_percent = set_reduce_percent #40%
        vc_mask_opacity = self.update_vc_mask(self.le_vc_mask.text())
        keep_num = len(self.lock_curves) * (1-reduce_percent) # 100*0.4 keep 60
        cmd.select(clear = True)
        while(len(curve_visible_list)) > keep_num: # when keep curves > 60, continue
            for curv in curve_visible_list:    
                first_cv = f'{curv}.cv[0]'
                first_cv_pos = cmd.pointPosition(first_cv)
                closest_vert_pos, closest_face_id = om.MFnMesh(head_dp).getClosestPoint(om.MPoint(*first_cv_pos))

                if closest_face_id in colored_face_dict.keys():
                    reduce_factor = self.clamp(colored_face_dict[closest_face_id], 0.0, 1.0) * vc_mask_opacity
                    reduce_percent *= (1 - reduce_factor)  
                    
                if (reduce_percent < 0.1):
                    if curv in curve_visible_list:
                        curve_visible_list.remove(curv)
            
                if(random.random() < reduce_percent): # if whithin 0.4, delete
                    cmd.select(f'{curv}', af=True)  
                    if curv in curve_visible_list:
                        #cmd.hide(curv)
                        curve_visible_list.remove(curv)
                        cmd.select(f'{curv}', add=True)
                reduce_percent = set_reduce_percent

```

## Float Qt Slider 

Qt's Slider widget only supports integer values, meaning the slider's step size is limited to 1. To allow for decimal values, you would need to adjust the slider's behavior by manually scaling the values.

So most of my UI consists of a QLabel, a QLineEdit, and a QSlider. For example, with a width property, I want the slider's range to be `(0, 5.0)`. The actual range of my QSlider would be `(0, 500)`. When the slider value changes, it triggers function `update_line_edit_float` that updates the value displayed in the QLineEdit as `QSlider.value()/100`. Similarly, when the QLineEdit value changes, it triggers `update_line_edit_float` to synchronize both UI elements.

```python
    def update_slider_float(self, slider_widget, input):
        slider_widget.setSliderPosition(float(input) * 100)

    def update_line_edit_float(self, line_edit_widget, input):
        line_edit_widget.setText(str(input/ 100))
```

## Column, Row, Angle, Width, Rotate

For parameters like Column, Row, Angle, Width, and Rotate, I can directly link them to the corresponding parameters in the selected polygons' Sweep nodes, I can use `cmds.SetAttr()` to batch modify them. This operation is straightforward, so I won't go into further detail here.


## Taper Curves 
TODO

## Length 
TODO

## Filter Cards by Distance to the scalp 
TODO


## Rotate Cards
TODO

## Move Towards Center
TODO

## Live Attach
TODO

## Live Attach by Root and Weight
TODO


<br />
<br />

## UV Editor
### QPainter
TODO




## WorkspaceControl

TODO

[Simple_MayaDockingClass.py](https://gist.github.com/liorbenhorin/69da10ec6f22c6d7b92deefdb4a4f475)

## Undo Chunk Customization

[Undo Chunk Solution with QSlider](https://www.tech-artists.org/t/need-help-with-qslider-collating-as-1-undo/11534/7)
TODO



<br />
<br />

## Pyd format Conversion
Generally, when delivering scripts for use, it's important to prevent accidental modifications that could lead to errors. This is especially true when sending scripts to external teams. In such cases, encrypting the scripts can be helpful. One way to do this is by using Cython to convert Python files into the `.pyd` format. This format displays as unreadable garbage text when opened, thus protecting the script from being easily modified or understood.

### Install Cython
Normally we can use `[ maya root path ]\bin\mayapy.exe" setup.py install` to install through Cython's setup.py file that downloaded directly from [Cython](https://github.com/cython/cython), but because of my Maya is in C disk and the company pc would warn me that there's no access to write, so I have to install Cython from Pip and manually setup into Maya Python. 

Therefore I run the following command directly in the cmd window. 
`pip install cython`
`pip install setuptools`

This will install Cython into the default python folder, for me it is:
`C:\Users\xxxx\AppData\Local\Programs\Python\Python311`

and copy the new added Cython files into Maya Python's corresponding folder:
![install-cython](/post-img/mayatools/hair-card-tool/install-cython.jpg){: width="100%" .shadow}

### Setup Maya Python folder
There's also a necessary step in Maya's Python folder:
Create two *new folders* --> `include` and `libs`, under `Maya2023/Python`, and copy corresponding files into them from the folders that directly under Maya:
![new-folders](/post-img/mayatools/hair-card-tool/new-folders.jpg){: width="100%" .shadow}

Try `import Cython` in Maya script Editor, if there's no error then can move to the compilation. 


### Cython Compilation

#### setup.py
Will need a setup.py file that contains the python files that I need to convert into pyd files, and put it into the same folder of my python scripts:
![setup-folder](/post-img/mayatools/hair-card-tool/setup-folder.jpg){: width="80%" .shadow}

```python
    # setup.py
    import os
    from distutils.core import setup
    from Cython.Build import cythonize
    from distutils.extension import Extension

    extensions = ['ct_hair_tool_v_1_1.py', 'draw_uv_window.py']

    setup(
        ext_modules = cythonize(extensions)
    )
```
#### Run Compilation
Then in cmd window, cd to the folder of the python scripts that I want to convert, and use `mayapy.exe` to execute the `setup.py` file by:

`"C:\Program Files\Autodesk\Maya2023\bin\mayapy.exe" setup.py build_ext --inplace`

![cmd](/post-img/mayatools/hair-card-tool/cmd.jpg){: width="100%" .shadow}

After generation, there're new files added in the compile folder, then the last step is to **remove the suffix generated on the pyd files**:
![after-compile](/post-img/mayatools/hair-card-tool/pyd-files.jpg){: width="80%" .shadow}

Then we can use those pyd files just like a normal python script, for example:
```python
    import importlib
    import sys
    sys.path.append(r"F:\TempFiles\Maya_Xgen\maya_script\ToolPack")

    import ct_hair_tool_v_1_1
    importlib.reload(ct_hair_tool_v_1_1)

    my_dock = ct_hair_tool_v_1_1.dock_window(ct_hair_tool_v_1_1.MyDockingUI)
```
where `ct_hair_tool_v_1_1` is `ct_hair_tool_v_1_1.pyd`


### Potential issue
```python
    > # Error: DLL load failed: The specified module could not be found.
    > # Traceback (most recent call last):
    > #   File "<maya console>", line 1, in <module>
    > # ImportError: DLL load failed: The specified module could not be found.
```
This could be caused by the mismatch of Visual Studio version, so make sure you have the corresponding VS version that Maya needs, you can find it from this Maya's official site, where development team keeps updating:
Around the Corner: [Maya 2023 API Update guide](https://around-the-corner.typepad.com/adn/2022/03/maya-2023-api-update-guide.html)
Like here, I'm using Maya2023, I need at least Visual Studio 2019. 
![maya-build-env](/post-img/mayatools/hair-card-tool/maya-build-env.jpg){: width="80%" .shadow}

