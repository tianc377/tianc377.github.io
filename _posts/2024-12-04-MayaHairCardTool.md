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

Qt's Slider widget only supports integer values, meaning the slider's minimum step size is limited to 1. To allow for decimal values, you would need to adjust the slider's behavior by manually scaling the values.

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
![cmd-curve](/post-img/mayatools/hair-card-tool/cmd-curve.gif){: width="100%" .shadow}
For the taper functionality, I used Maya's cmds API function `falloffCurveAttr`. This function requires first creating an attribute group node, then add necessary attributes to the ramp node using `addAttr`; and the keys on that curve can be referenced as `xxx.xxxcurve[i]`to add keys on the curve.
The taper feature code is below:
```python
    # Taper Curve
    if cmd.ls('ramp_node'):
        cmd.delete('ramp_node')
    cmd.group(em=True, n = 'ramp_node')
    cmd.addAttr('ramp_node', attributeType='compound', ln='taper_curve',numberOfChildren = 3, multi=True)
    cmd.addAttr('ramp_node', at='float', ln='taper_curve_pos', p='taper_curve', defaultValue= 1) #position
    cmd.addAttr('ramp_node', at='float', ln='taper_curve_val', p='taper_curve', defaultValue= 1) #value
    cmd.addAttr('ramp_node', at='enum', ln='taper_curve_type', p='taper_curve', enumName = 'None:Linear:Smooth:Spline', defaultValue= 1, min = 0, max = 3) #interp
    cmd.setAttr('ramp_node.taper_curve[0]', 0, 1, 2) #position, value, interp
    cmd.setAttr('ramp_node.taper_curve[1]', 0.5, 1, 2) #position, value, interp
    cmd.setAttr('ramp_node.taper_curve[2]', 0.75, 1, 2) #position, value, interp
    cmd.setAttr('ramp_node.taper_curve[3]', 1, 1, 2) #position, value, interp

    self.curve_attr = cmd.falloffCurveAttr( 'Taper Curve', h=90, attribute = 'ramp_node.taper_curve', changeCommand = functools.partial(self.update_current_curve_sliders))
    curve_attr_qwidget = omui.MQtUtil.findControl(self.curve_attr) # Wrap Maya cmd UI to Qt Widget
    curve_attr_qwidget = wrapInstance(int(curve_attr_qwidget), QtWidgets.QWidget)
```

## Length 
![card-length](/post-img/mayatools/hair-card-tool/card_length.gif){: width="100%" .shadow}
The SweepTool lacks a built-in function to control length. Initially, I tried simply stretching the polygons along the Y-axis, but this caused deformation and didn’t work well for horizontally placed polygons. Later, I used the Extrude function to extend polygons by adding new faces at the ends, and automatically adjusts the division count based on the extended length.

First, the indices of the edges at the very end must be identified. From the sweep attributes, I can obtain the number of polygon columns (e.g., 3). Thus, the last 3 faces correspond to the last row of the card. Among these faces, I locate the third edge, which represents the bottom edge of each quad. These three edges are then stored in a dictionary, preparing them for the next extrusion step.

```python
    def find_card_edge(self, undo_name):
        self.undo_on(undo_name)
        # input = input/100
        self.sel_cards = cmd.ls(sl = True, type = 'transform')
        self.extrude_card_edge_dict = {}

        for card in self.sel_cards:
            if 'sweep' in str(card):
                card_dp = om.MSelectionList().add(card).getDagPath(0)
                poly_iter = om.MItMeshPolygon(card_dp)
                # creator = self.find_sweep_creators() #[['sweepMeshCreator2'], ['sweepMeshCreator3']]
                shape = cmd.listRelatives(card)
                creator = cmd.findType(shape, deep=True, type='sweepMeshCreator')[0]
                segment = cmd.getAttr(f'{creator}.profileArcSegments') # Num of the column of the card
                face_index_list = []
                edge_index_list = []
                for f in poly_iter:
                    face_index_list.append(f.index())
                    edge_index_list.append(f.getEdges())
                last_few_edges = edge_index_list[-segment:]
                extrude_edges = []
                for e in last_few_edges:
                    edge_name = f'{card}.e[{e[2]}]' #the third edge of every face
                    extrude_edges.append(edge_name)
                self.extrude_card_edge_dict.update({card:extrude_edges})
```

In the `exec_extrude_edge` function, I use `MItMeshEdge` to find the length of edge #1 (one of the side edges of the first face), which represents the height of each row. When a length value is entered via the slider, it is divided by the row's height, and the integer result determines the number of divisions. This ensures that divisions are automatically added based on the extended length:

```python
    def exec_extrude_edge(self, slider, undo_name):

        input = slider.value()/100
        # Extrude
        for k in self.extrude_card_edge_dict.keys():
            print(f'k = {k}')
            # get edge length:
            card_dp = om.MSelectionList().add(k).getDagPath(0)
            edge_iter = om.MItMeshEdge(card_dp)

            edge_length = 1.0  # init
            for e in edge_iter:
                if e.index() == 1:
                    edge_length = e.length()
                    print(f'edge_length = {edge_length}')
                    break
  
            shape = cmd.listRelatives(k)[0]
            print(f'shape = {shape}')
            extrude_node = cmd.findType(shape, deep=True, type = 'polyExtrudeEdge')
            print(f'extrude_node = {extrude_node}')
            if extrude_node == None:
                extrude_node = cmd.polyExtrudeEdge(self.extrude_card_edge_dict[k], lty = 1.0, divisions = 0)
                print(f'extrude_node2 = {extrude_node}')

            cmd.setAttr(f'{extrude_node[0]}.localTranslateY', input)
            cmd.setAttr(f'{extrude_node[0]}.smoothingAngle', 60)
            print(f'{extrude_node[0]}.localTranslateY')

            # Find Division 
            div = math.floor(input / edge_length)
            if div < 1:
                div = 1
            cmd.setAttr(f'{extrude_node[0]}.divisions', div)
            print(f'div = {div}')
            
        self.undo_off(undo_name)
```



## Filter Cards by Distance to the scalp 

![card-filter](/post-img/mayatools/hair-card-tool/card-filter.gif){: width="100%" .shadow}

This feature is designed to filter out polygons closer to the scalp. I calculate the distance between the center of each card and the center of the head mesh, then sort them into a list based on this distance.

```python
    def make_card_list_by_distance(self):
        #self.sel_cards = cmd.ls(sl = True)
        self.dist_dict = {}
        for card in self.lock_cards:
            card_center = cmd.objectCenter(card)
            dist = math.dist(card_center, self.head_center)
            self.dist_dict.update({card: dist})

        self.dist_dict = dict(sorted(self.dist_dict.items(), key=lambda item: item[1]))
        #return self.dist_dict
    
    def slider_inner_cards_select(self, input):
        self.make_card_list_by_distance()
        self.cards_num = len(self.lock_cards)

        cmd.select(clear = 1)
        input = int(input)
        if input > self.cards_num:
            input = self.cards_num
        for i in range(input):
            cmd.select(f'{list(self.dist_dict.keys())[i]}', af= True)
```



## Move Towards the Scalp
This feature adjusts the distance between hair cards and the scalp. I find the center point of each card, then by using the `om.MFnMesh.getClosestPoint` function to find the closest point of head scalp to that center point. The cards then move along this vector.

Initially, I calculated the direction vector using head point - card center, but this approach caused inconsistent movement directions for cards positioned on the left and right sides of the head:

![mismatch-direction](/post-img/mayatools/hair-card-tool/mismatch_directions.gif){: width="100%" .shadow}

Ideally, regardless of their location, the cards should always move directly toward the scalp. So, I added a directional check in the function. If the product of the `card_to_head` vector and the `direction` vector is negative, it indicates the moving direction is reversed. In this case, the `direction` vector is multiplied by `-1` to correct it, see the `Line 26-28` in the following code:

```python
    def slider_lock_move_direction(self):
        self.new_point_dict = {}

        try:
            head_dp = om.MSelectionList().add(self.sel_head).getDagPath(0)
        except:
            cmd.warning( "No Head Selected" )
            return

        inner_cards = cmd.ls(sl = True, type = 'transform')
        if not inner_cards:
            cmd.warning( "There's no Card Selected" )
            return
        
        head_mesh = om.MFnMesh(head_dp)
        head_center = cmd.objectCenter(self.sel_head)

        for card in inner_cards:
            curve = cmd.filterExpand(card, sm = 9)
            if not curve:
                #print(f'card:{card}')
                card_center = cmd.objectCenter(card)
                card_to_head = om.MVector(head_center) - om.MVector(card_center)
                closest_head_vert = head_mesh.getClosestPoint(om.MPoint(*card_center))
                direction = om.MVector(closest_head_vert[0]) - om.MVector(card_center)
                for i in range(3):
                    if card_to_head[i] * direction[i] < 0: # Correct the moving direction
                        direction[i] *= -1                   


                direction = om.MVector(*direction).normalize()
                card_position = cmd.xform(card, query = True, scalePivot = True)
                card_position = om.MVector(*card_position)
                
                self.new_point_dict.update({card: [card_position, direction]})

    # Move function
    def slider_move_cards_to_closest_face(self, input): 
        self.undo_on('move dist')       
        if not self.new_point_dict:
            return
        
        for item in self.new_point_dict.items():
            new_point =  item[1][0] + item[1][1] *(input/100)
            cmd.xform(item[0], translation = new_point)
```
![match-direction](/post-img/mayatools/hair-card-tool/match_directions.gif){: width="100%" .shadow}




## Rotate Cards
![rotate-card](/post-img/mayatools/hair-card-tool/card-rotate.gif){: width="100%" .shadow}

Although the Sweep Tool includes a polygon rotation feature, I created a one-click rotation functionality. This tool calculates the center point and normal vector of each card's face, finds the closest point on the scalp to the center point, and determines its normal vector. Using these pairs of normals, it calculates Euler values for rotation, averages them, and applies the average Euler rotation to the card using its center point as the pivot.

`Get Euler between two vectors:`
```python
    def get_euler(self, head_closest_normal, card_poly_normal, head_xform, card_xform, card_poly, card_poly_center):
        head_closest_normal = om.MVector(*head_closest_normal)
        card_poly_normal = om.MVector(*card_poly_normal)
        head_xform_mtx = cmd.xform(head_xform, query = True, worldSpace= True, matrix = True)
        card_xform_mtx = cmd.xform(card_xform, query = True, worldSpace= True, matrix = True)
        head_matrix = om.MMatrix(head_xform_mtx)
        card_matrix = om.MMatrix(card_xform_mtx)
        head_closest_normal *= head_matrix
        card_poly_normal *= card_matrix

        rotation = card_poly_normal.rotateTo(head_closest_normal)
        euler = rotation.asEulerRotation().asVector()
        euler = [x * (180/math.pi) for x in euler]

        return list(euler)
```
`Get the two normal vectors from the card and the scalp, average the result and execute rotation`
```python
    def rotate_card(self):
        try:
            head_dp = om.MSelectionList().add(self.sel_head).getDagPath(0)
        except:
            cmd.warning( "No Head Selected" )
            return

        inner_cards = cmd.ls(sl = True, type = 'transform')
        if not inner_cards:
            cmd.warning( "There's no Card Selected" )
            return

        
        head_mesh = om.MFnMesh(head_dp)
        euler= [0,0,0]

        if self.sel_head in inner_cards:
            inner_cards.remove(f'{self.sel_head}') # Otherwise rotating head causes Maya crash

        for card in inner_cards:
            curve = cmd.filterExpand(card, sm = 9)
            if not curve:
                card_dp = om.MSelectionList().add(card).getDagPath(0)

                card_mesh = om.MFnMesh(card_dp)
                card_mesh_vert_iter = om.MItMeshVertex(card_dp)
                card_mesh_poly_iter = om.MItMeshPolygon(card_dp)

                for poly in card_mesh_poly_iter:
                    card_poly_center = poly.center()
                    card_poly_normal = poly.getNormal()  
                    card_poly_xform = f'{card}.f[{poly.index()}]'  

                    head_closest_point, head_closest_normal, head_closest_face_id = head_mesh.getClosestPointAndNormal(card_poly_center) # Only pick the first vert

                    cmd.select(f'{head_dp}.f[{head_closest_face_id}]')

                    # Get the sum up euler from each face
                    for i in range(3):
                        euler[i] += self.get_euler(head_closest_normal, card_poly_normal, self.sel_head, card, card_poly_xform, list(poly.center())[:-1])[i]
                
                # Get average euler
                for i in range(3):
                    euler[i] /= cmd.polyEvaluate(card, face = True)

                # Exec rotate
                card_poly_center = cmd.objectCenter(card) # use card center as rotate pivot
                cmd.select(card)
                cmd.rotate(euler[0], euler[1], euler[2], card, relative = True, p = card_poly_center, os = True, fo = True) 

                cmd.move(0,0,0, f'{card}.scalePivot', f'{card}.rotatePivot')
                cmd.makeIdentity(card, apply=True)

```


## Live Attach

![live-attach](/post-img/mayatools/hair-card-tool/live-attach.gif){: width="100%" .shadow}

Considering the need to closely align the cards to the scalp to create a shell that covers the head, I added a "Live Attach" button. This feature utilizes Maya's Live Surface functionality, allowing each face of the card to snap and conform to the scalp model.


## Live Attach by Root and Weight
TODO

<br />
<br />

## Undo Chunk Customization

[Undo Chunk Solution with QSlider](https://www.tech-artists.org/t/need-help-with-qslider-collating-as-1-undo/11534/7)
TODO



<br />
<br />


## UV Editor
### QPainter
TODO




## WorkspaceControl

TODO

[Simple_MayaDockingClass.py](https://gist.github.com/liorbenhorin/69da10ec6f22c6d7b92deefdb4a4f475)




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

