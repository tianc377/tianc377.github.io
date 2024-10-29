---
title:  "Substance Painter to Unreal Exporter"
date:   2024-10-25 21:00
categories: [unrealtools]
tags: [Python, Substance Painter]
image: /post-img/unrealtools/sp-export-cover.png
published: true
---




In the last few posts, I experimented with methods of remote execution between Unreal and DCC tools, which gave me a solid foundation to start developing a tool. This tool will allow users to export textures from Substance Painter and import them into Unreal with a single click, as well as open the source Substance Painter project directly from Unreal's content browser with a right-click. It’s designed to save time on export/import actions and eliminate the hassle of searching for source files. Instead of asking around, artists can easily find the paths and files they need.


Before starting the scripting process, I created a simple flowchart to map out the workflow involved in this tool.
![flowchart](/post-img/unrealtools/sp-to-unreal-tool/flowchart.png){: width="100%"}


## Add a Toolbar in Substance Painter
First, I need to add a button for my plugin. Substance Painter has its own Python API, and within the [substance_painter.ui](https://helpx.adobe.com/substance-3d-painter-python/api/substance-painter/ui.html) module, there’s a function called `add_toolbar` that allows you to add a toolbar beneath the menu row:
![toolbar](/post-img/unrealtools/sp-to-unreal-tool/toolbar.png){: width="100%" .shadow}
I can then add a `QToolButton` for my plugin within this toolbar:

```python
# Create a new class for my plugin's UI
class main_window():
    def __init__(self):
        pass

    def button_clicked(self):
        main_dlg = PopupDialog()

    def create_button(self):
        icon_path = plugin_path + '/icon.png'
        toolbar = substance_painter.ui.add_toolbar("Export To Unreal", "Export To Unreal")
        self.button = PySide6.QtWidgets.QToolButton()
        self.button.setText('UE')
        self.button.setIcon(PySide6.QtGui.QIcon(icon_path))
        self.button.setToolTip('Export To UE')
        toolbar.addWidget(self.button)

        self.button.clicked.connect(functools.partial(self.button_clicked))

        plugin_widgets.append(toolbar)
```

## Popup Dialog Window

![popup-dialog](/post-img/unrealtools/sp-to-unreal-tool/popup-dialog.gif){: width="100%" .shadow}

Next, I want a dialog window to open when I click the plugin’s button. In the code above, I use `self.button.clicked.connect()` to call another method, `button_clicked`, which instantiates a class called `PopupDialog()`. It’s worth noting that I used `functools.partial()` because, for some reason, the `.connect() `function doesn’t work without it, the dialog doesn’t pop up. I’m wondering if this might be related to Substance Painter itself.

Anyway, next, I’ll create another class to handle the dialog window.

In my opinion, a project's raw data depot should mirror the directory structure within the engine, creating a clean and organized data repository that’s easy to navigate. In the current project that I'm working on, however, the art content directories and the engine directories are disorganized, making it nearly impossible to locate an asset’s source file without asking around. 

Following this approach, I structured my raw data depot to mirror the engine directory. Rather than having users select folders from an explorer window, I’m considering using comboboxes to navigate through each directory level step-by-step, with each combobox populated by the folders within that directory level. Only a TA or TD would have the permission to create new folders in the depot.


### Config Root Path
Before moving further, there’s another issue to address: the root path of the raw data workspace. Since all asset paths rely on this root path, which can vary across different machines, we need a consistent solution. In a real project, we could use the Windows **`subst`** command to map the directory to a uniform location (e.g., P:/), so everyone shares the same root path. 

For my personal project, though, I’ll simply use a config file to store the root path:

![config-setup](/post-img/unrealtools/sp-to-unreal-tool/config-setup.gif){: width="100%" .shadow}

The `config_setup.py` script will run if the exporter script doesn’t detect the `rootpath_config` file in the plugin folder. It opens a dialog prompting users to select their raw data workspace root folder, then saves this path to a config file.

Below is my `config_setup.py`:
```python
import sys
import configparser
import os
import pathlib
import tkinter
import tkinter.filedialog
import PySide6
import functools 
import substance_painter
import substance_painter.ui
import substance_painter.logging
from PySide6.QtWidgets import QToolButton, QDialog, QComboBox, QHBoxLayout, QLabel, QVBoxLayout, QGridLayout, QLineEdit, QPushButton


class ConfigSetup():
    def __init__(self):
        self.current_path = str(pathlib.Path(__file__).parent.absolute())
    def create_config(self, input):
        
        config = configparser.ConfigParser()

        config['Default'] = {'root_path': input}

        with open(f'{self.current_path}/rootpath_config.ini', 'w') as configfile:
            config.write(configfile)

    def read_config(self, keyword):
        config = configparser.ConfigParser()
        config.read(f'{self.current_path}/rootpath_config.ini')
        root_path = config['Default'][keyword]
        print('from read_config:'+root_path)
        return root_path

    def choose_folder_dialog(self, button_sel_dir):
        dlg = tkinter.filedialog.askdirectory()
        button_sel_dir.setText(dlg)
        print('from button:' + button_sel_dir.text())

        self.create_config(dlg)
    
    def ok_clicked(self, dialog, input):
        output = self.read_config(input)
        substance_painter.logging.warning(f"Root path of your workspace has been set: {output}")
        substance_painter.ui.delete_ui_element(dialog)

    def select_root_path_dialog(self):
        
        dlg = QDialog()
        dlg.setWindowTitle('Set rawdata workspace root path')
        self.button_sel_dir = QPushButton(text = 'Select the directory')
        self.button_sel_dir.clicked.connect(functools.partial(self.choose_folder_dialog, self.button_sel_dir))

        button_ok = QPushButton(text = 'OK')
        button_ok.clicked.connect(functools.partial(self.ok_clicked, dlg, 'root_path'))

        layout = QGridLayout()
        layout.setColumnMinimumWidth(1, 350)
        layout.addWidget(self.button_sel_dir, 0,1)
        layout.addWidget(button_ok, 1,1)


        dlg.setLayout(layout)
        dlg.exec_()



def start_plugin():

    obj = ConfigSetup()
    config_path = str(pathlib.Path(__file__).parent.absolute()) + '/rootpath_config.ini'
    if(os.path.exists(config_path)):
        substance_painter.logging.warning(f"Root path of your workspace has been set: {obj.read_config('root_path')}, if need to change, delete rootpath_config.ini and restart.")
    else:
        config_window = obj.select_root_path_dialog()

    

def close_plugin():
    pass

if __name__ == "__main__":
    start = start_plugin()

```

<br />


#### No Module Named tkinter
One point worth noting: I used `tkinter` to provide an explorer window for users to select the root path. However, I encountered issues when trying to use `tkinter` within Substance Painter, as it kept returning an error saying "No module named tkinter", even though it was already installed. It turns out this is because Substance Painter uses its own Python installation located at `C:\Program Files\Adobe\Adobe Substance 3D Painter\resources\pythonsdk`, which doesn’t include `tkinter`. To resolve this, I went to my own Python installation directory,

- copy and paste the whole `tcl` folder:
![tcl](/post-img/unrealtools/sp-to-unreal-tool/tcl.png){: width="100%" .shadow}

- copy and paste required dlls:
![tkinter-dll](/post-img/unrealtools/sp-to-unreal-tool/tkinter-dll.png){: width="100%" .shadow}

- copy and paste the `_tkinter.lib`:
![tkinter-lib](/post-img/unrealtools/sp-to-unreal-tool/tkinter-lib.png){: width="100%" .shadow}

Then back to Substance Painter, `tkinter` module should be able to use. 


<br />


### Populate Combobox 

As mentioned, each directory level will be stored in a specific combobox. After selecting an option from one combobox, the next combobox will refresh and display items based on the previous selection. For example, if "CHR" is selected in the first combobox, the second combobox will show options like "Humanoid" and "Outfit," rather than options like "Building," "Terrain," or "Vegetation," which belong to different categories:
![combobox](/post-img/unrealtools/sp-to-unreal-tool/combobox.gif){: width="100%" .shadow}

To achieve this, every combobox except the first (which isn’t dependent on a prior selection) needs to repopulate whenever the previous combobox’s selection changes. This means that each combobox can use the path of the last selected item as a starting point to list the folders under that path, as the folder names will serve as the current combobox item names.

So, I created two populate functions that are triggered when a combobox’s item index changes. For instance, when the selected item index in combobox 2 changes, the connected function repopulates combobox 3 accordingly.

Additionally, I'll store each folder's full path in its corresponding **`itemData()`**. This allows easy retrieval of the current selected folder's path when needed.

```python
...
class PopupDialog(QDialog):
    def __init__(self):
        super().__init__()

        try:
            self.root_path = config_setup.ConfigSetup().read_config('root_path')
        except:
            root_config = config_setup.start_plugin()
            self.root_path = config_setup.ConfigSetup().read_config('root_path')
        
        dlg = QDialog()
        dlg.setWindowTitle('Select Export Directory')
        dlg.resize(600, 100)
        
        ...
        ...


        export_button = QPushButton(f'Export to: {self.read_last_selection_config(0)[1]}')
        self.populate_directory(combobox1)
        combobox1.currentIndexChanged.connect(functools.partial(self.populate_sub_directory, combobox1, combobox2, export_button, '1'))
        combobox2.currentIndexChanged.connect(functools.partial(self.populate_sub_directory, combobox2, combobox3, export_button, '2'))
        combobox3.currentIndexChanged.connect(functools.partial(self.populate_sub_directory, combobox3, combobox4, export_button, '3'))
        combobox4.currentIndexChanged.connect(functools.partial(self.populate_sub_directory, combobox4, combobox5, export_button, '4'))
        combobox5.currentIndexChanged.connect(functools.partial(self.populate_sub_directory, combobox5, combobox6, export_button, '5'))
        
        export_button.clicked.connect(functools.partial(lambda: self.export_button_clicked(export_button)))

        row_layout = QVBoxLayout()
        row_layout.addLayout(grid_layout)
        row_layout.addWidget(export_button)

        dlg.setLayout(row_layout)

        dlg.exec_()

    def populate_directory(self, combobox): # For the first combobox
        paths = [p.path for p in os.scandir(self.root_path) if p.is_dir()]

        for p in paths:
            name = pathlib.Path(p).name
            path = pathlib.PureWindowsPath(p).as_posix()
            combobox.addItem(name, path)
            

    def populate_sub_directory(self, current_combobox, next_combobox, export_button, level, current_index): # For the rest of comboboxes

        current_item_data = current_combobox.itemData(current_index)

        if current_item_data != None:       
            current_path = current_item_data
            paths = [p.path for p in os.scandir(str(current_path)) if p.is_dir()]
            print(*paths)
            file_name = current_path.replace(f'{self.root_path}/', '')
            file_name = file_name.replace('/', '_')
            file_name = 'T_' + file_name + '_$.png'
            export_button.setText(f'Export to: {current_path}/Textures/{file_name}')
            print('export label: ' + export_button.text()) #Export to: D:/ctrawdata/ENV/Building/Pipes/Hole/Rusty_01/Textures/ENV_Building_Pipes_Hole_Rusty_01_$.png

            next_combobox.clear()
            next_combobox.addItem('..')
            for p in paths:     
                name = pathlib.Path(p).name
                path = pathlib.PureWindowsPath(p).as_posix()
                next_combobox.addItem(name, path)     
...

```


<br />

### Remember the Last Selection

![remember-last](/post-img/unrealtools/sp-to-unreal-tool/remember-last.gif){: width="100%" .shadow}


To make the dialog window remember the last selection, I created two additional functions: one to save the current selection to a config file, and another to read this config when populating the comboboxes. This way, I don’t have to reselect items each time I export. The line `export_button = QPushButton(f'Export to: {self.read_last_selection_config(0)[1]}')` in the above code is reading the last selection path and displaying on the export button. 

```python
    ...
    def create_last_selection_config(self, last_path):
         if os.path.exists(saved_path_config):
             os.remove(saved_path_config)
         new_config = configparser.ConfigParser()
         new_config['Default'] = {'last_path': last_path}  
         with open(saved_path_config, 'w') as configfile:
            new_config.write(configfile)  


    def read_last_selection_config(self, level):
        config = configparser.ConfigParser()
        last_path = f'{self.root_path}/../../../../../../..'
        if not os.path.exists(saved_path_config):
            self.create_last_selection_config(last_path)

        config.read(saved_path_config)
        try:
            last_path = config['Default']['last_path']
        except:
            self.create_last_selection_config(last_path)
        make_items = os.path.dirname(last_path)
        make_items = os.path.dirname(make_items)
        make_items = make_items.replace(f'{self.root_path}/','')
        make_items = make_items.replace('/', ',')
        print('make_items:' + make_items)
        items = make_items.split(',')
        return [items[level], last_path]
    ...
```



<br />


## Export


<br />

### Substance Painter Export Preset Configuration


<br />

### Export Button Clicked


<br />

#### Send to UE



<br />
<br />

## Open Texture Source SP Project from UE

### Add Custom UI into Menu of UE




<br />

<br />


## Result
![export-to-ue](/post-img/unrealtools/sp-to-unreal-tool/export-to-ue2.gif){: width="100%" .shadow}
![open-with-sp](/post-img/unrealtools/sp-to-unreal-tool/open-with-sp.gif){: width="100%" .shadow}












<!-- - SP export tool 
- SP 添加plugin tool bar， 以及只有functool才起作用的问题
- 关于遍历文件夹的方法 
- 关于Qt信号
- tkinter无法使用的问题
- SP export preset 的使用
- 记录上次所选内容
- 导出到UE并记录UE路径+SP文件路径
- UE内add new button
- UE内自动打开SP


- create p4v personal server 
- p4 timeout error
- laggy unreal editor and p4ignore file

- unreal python editor validator base

- p4 trigger doesn't work, because need to use unreal.py module 

- get source code and disable that button

- Error: Unreal Unhandled exception: System.UnauthorizedAccessException: Access to the path...: untick 'read only' from the Binary folder's property setting

- in order to fix redirector right after renaming: manually create a C++ function that Python can call from.



- create p4v personal server 
- Error: Unreal Unhandled exception: System.UnauthorizedAccessException: Access to the path...: untick 'read only' from the Binary folder's property setting
- p4 timeout error
- laggy unreal editor and p4ignore file
- p4 trigger doesn't work, because need to use unreal.py module  -->