---
title:  "Substance Painter to Unreal Exporter"
date:   2024-10-25 21:00
categories: [unrealtools]
tags: [Python, Substance Painter]
image: /post-img/unrealtools/sp-export-cover.png
published: true
---

## Intro
In the last few posts, I experimented with methods of remote execution between Unreal and DCC tools, which gave me a solid foundation to start developing a tool. This tool will allow users to export textures from Substance Painter and import them into Unreal with a single click, as well as open the source Substance Painter project directly from Unreal's content browser with a right-click. It’s designed to save time on export/import actions and eliminate the hassle of searching for source files. Instead of asking around, artists can easily find the paths and files they need.

<br />

Below shows how the tool works:

`Export from SP to Unreal`:
![export-to-ue](/post-img/unrealtools/sp-to-unreal-tool/export-to-ue2.gif){: width="100%" .shadow}

<br />

`Open source file from Unreal`:
![open-with-sp](/post-img/unrealtools/sp-to-unreal-tool/open-with-sp.gif){: width="100%" .shadow}


<br />

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

`PopupDialog()`:
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

`create_last_selection_config()`:
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


## Export Button Clicked

Below the combobox, there’s an export button that displays the full export path, which updates as the combobox selections change.

In the `export_button_clicked` function, the script performs the following tasks:

- Saves the current combobox selections to the configuration file for easy retrieval next time.
- Finds or creates a Substance folder to save the current project file.
- Finds or creates a Textures folder for storing exported textures.
- Reads the export preset from a JSON file and checks the designated texture paths for existing textures. If textures already exist, they are deleted.
- Exports textures according to the selected preset and configuration.
- Calls `send_to_ue` to remotely execute the designated Unreal Engine script with the texture paths.

The code of this part is here:

`export_button_clicked()`:
```python
# Export Button Function
def export_button_clicked(self, button):
        button_display_path = button.text().replace('Export to: ', '')
        project_basepath = os.path.dirname(button_display_path).replace('/Textures','')
        project_basename = os.path.basename(button_display_path).replace('T_','').replace('_$.png', '')
        sp_project_folder = project_basepath + '/Substance'
        sp_project_path = project_basepath + f'/Substance/{project_basename}.spp'
        textures_folder = project_basepath + f'/Textures'


        self.create_last_selection_config(button_display_path)

        # Save project file
        if substance_painter.project.is_open():
            if not os.path.exists(sp_project_folder):
                os.mkdir(sp_project_folder)
            substance_painter.project.save_as(sp_project_path)
            print(f'The current project has been saved at: {sp_project_path}')
        else:
            substance_painter.logging.error("---------There's no project opened---------")
        
        # Export textures
        tex_source_paths = []
        texture_uedest_paths = []
        if substance_painter.project.is_open():
            if not os.path.exists(textures_folder):
                os.mkdir(textures_folder)

            with open(f'{plugin_path}' + '\my_export_preset.json') as file:
                preset = json.load(file)
                for map in preset['maps']:
                    tex_source_paths.append(textures_folder + '/' + map['fileName'].replace('$project', substance_painter.project.name()) + '.' + map['parameters']['fileFormat'])
                    texture_uedest_paths.append(textures_folder.replace(f'{self.root_path}', '/Game/Assets'))

                for tex in tex_source_paths:
                    if os.path.exists(tex):
                        os.remove(tex)
            # Export
            export_config = self.make_export_config(textures_folder) 
            export_result = substance_painter.export.export_project_textures(export_config)
            if export_result == substance_painter.export.ExportStatus.Success:
                print(export_result.message)
        else:
            substance_painter.logging.error("---------There's no project opened---------")

        # Send to UE
        for tex_source_path in tex_source_paths:
            self.send_to_ue(tex_source_path, sp_project_path, self.root_path)
```

<br />

### Substance Painter Export Preset Configuration

- From the above code, in **line 39**, `export_config` is created from a function named `make_export_config()`, then in **line 40**, `substance_painter.export.export_project_textures()` uses this `export_config` as an argument, applying the settings in the configuration to export textures:

`make_export_config()`:
```python

def make_export_config(self, textures_folder):

    with open(f'{plugin_path}' + '\my_export_preset.json') as file:
        preset = json.load(file)
        preset_name = preset['name']

    stack = substance_painter.textureset.get_active_stack()

    export_config = {
        "exportShaderParams": False,
        "exportPath": textures_folder,
        "defaultExportPreset" : preset_name,
        "exportPresets": [preset],
        "exportList" : [ { "rootPath" : str(stack) } ]
        }
    return export_config
```

- In this function I open and read a JSON file `my_export_preset.json`, which is basically a json version of Substance Painter's Output Template: 
![output-template](/post-img/unrealtools/sp-to-unreal-tool/output-template.png){: width="70%" .shadow} 
- This Json file allows you to specify how texture maps are combined and outputted, detailing settings like file naming conventions, format, resolution, and source channel, destination channel and suffixes required:

`my_export_preset.json`:
```json 

{
    "name": "CTPreset",
    "maps": [
                {
                    "fileName": "T_$project_D",
                    "parameters": {
                        "bitDepth": "8",
                        "dithering": false,
                        "fileFormat": "png",
                        "paddingAlgorithm": "diffusion",
                        "dilationDistance": 16
                        },
                    "channels": [
                        {
                            "destChannel": "R",
                            "srcChannel": "R",
                            "srcMapType": "documentMap",
                            "srcMapName": "basecolor"
                        },
                        {
                            "destChannel": "G",
                            "srcChannel": "G",
                            "srcMapType": "documentMap",
                            "srcMapName": "basecolor"
                        },
                        {
                            "destChannel": "B",
                            "srcChannel": "B",
                            "srcMapType": "documentMap",
                            "srcMapName": "basecolor"
                        }
                    ]
                },
                {
                    "fileName": "T_$project_N",
                    "parameters": {
                        "bitDepth": "8",
                        "dithering": false,
                        "fileFormat": "png",
                        "paddingAlgorithm": "diffusion",
                        "dilationDistance": 16
                        },
                    "channels": [
                        {
                            "destChannel": "R",
                            "srcChannel": "R",
                            "srcMapType": "virtualMap",
                            "srcMapName": "Normal_DirectX"
                        },
                        {
                            "destChannel": "G",
                            "srcChannel": "G",
                            "srcMapType": "virtualMap",
                            "srcMapName": "Normal_DirectX"
                        },
                        {
                            "destChannel": "B",
                            "srcChannel": "B",
                            "srcMapType": "virtualMap",
                            "srcMapName": "Normal_DirectX"
                        }
                    ]
                },
                {
                    "fileName": "T_$project_AORM",
                    "parameters": {
                        "bitDepth": "8",
                        "dithering": false,
                        "fileFormat": "png",
                        "paddingAlgorithm": "diffusion",
                        "dilationDistance": 16
                        },
                    "channels": [
                        {
                            "destChannel": "R",
                            "srcChannel": "R",
                            "srcMapType": "documentMap",
                            "srcMapName": "ambientOcclusion"
                        },
                        {
                            "destChannel": "G",
                            "srcChannel": "G",
                            "srcMapType": "documentMap",
                            "srcMapName": "roughness"
                        },
                        {
                            "destChannel": "B",
                            "srcChannel": "B",
                            "srcMapType": "documentMap",
                            "srcMapName": "metallic"
                        }
                    ]
                }
            ]
}
```
- Basically, the `config` argument that `substance_painter.export.export_project_textures()` take has to include `exportPath`, `exportPresets` which I saved in a json file, and `exportList` for project that has multiple texture sets. 


<br />

### Send to UE

I've already covered the details of remote execution from DCC to UE in another post [Remote Execution Between Unreal and DCC](https://tianc377.github.io/posts/RemoteExecutionBetweenUnrealandDCC/).
In that `export_button_clicked()` function, the `send_to_ue()` function used at line 48 is shown below:

`send_to_ue()`:
```python
def send_to_ue(self, tex_source_path, sp_proj_path, root_path):
        remote_exec = remote.RemoteExecution()
        remote_exec.start()
        time.sleep(2) # sleep 2 to let start() work, otherwise no remote node available
        # Ask for a remote node ID
        remote_node_id = remote_exec.remote_nodes
        # Connect to it
        if remote_node_id:
            remote_exec.open_command_connection(remote_node_id)
            exec_mode = 'ExecuteFile' # use this mode to execute a file directly
            rec = remote_exec.run_command("import import_textures_from_rawdata", exec_mode=exec_mode)    
            rec = remote_exec.run_command("import importlib", exec_mode=exec_mode)    
            rec = remote_exec.run_command("importlib.reload(import_textures_from_rawdata)", exec_mode=exec_mode)    
            if rec['success'] == True:
                rec = remote_exec.run_command(f"import_textures_from_rawdata.import_texture_to_ue('{tex_source_path}','{sp_proj_path}','{self.root_path}')", exec_mode=exec_mode)  
        
                remote_exec.stop() # Other wise will have infinite 'Unhandled remote execution message type "ping"'
                substance_painter.logging.warning("---------Successfully sent command to Unreal!---------")
            else:
                remote_exec.stop()
                substance_painter.logging.warning("--------Failed send command to Unreal---------")
        else:        
            substance_painter.logging.error("---------There's no available editor---------")
            remote_exec.stop()
```
- In the code above, I pass the texture paths to another Python script, `import_textures_from_rawdata.py`. At this point, the script begins executing **within Unreal**.

## In Unreal

### Import Textures

`import_textures_from_rawdata.py`:
```python
import unreal
import os
import json
from pathlib import Path
ue_python_location = os.path.dirname(__file__)#D:/Projects/github/real-playground/RealPlayground/Plugins/CTLib/Content/Python/       

def write_json(data):
    with open(f'{ue_python_location}/sp_data.json', 'w') as f:
        json.dump(data, f)

def read_json():
    with open(f'{ue_python_location}/sp_data.json') as f:
        data = json.load(f)
    return data

def import_texture_to_ue(tex_source_path, sp_proj_path, root_path):
        task = unreal.AssetImportTask()
        task.automated = True
        task.destination_name = Path(tex_source_path).stem
        task.destination_path = os.path.dirname(tex_source_path).replace(f'{root_path}', '/Game/Assets')
        task.filename = tex_source_path
        task.replace_existing = True
        task.save = False

        unreal.AssetToolsHelpers.get_asset_tools().import_asset_tasks([task])

        try:
            data = read_json()
        except:
            write_json({'ue_dest_path':'sp_proj_path'})
            data = read_json()
        ue_dest_path = f'{task.destination_path}/{task.destination_name}'
        if ue_dest_path not in data:
            data.update({f'{ue_dest_path}': f'{sp_proj_path}'})
            write_json(data)
```
- Within this script, I used `unreal.AssetImportTask()` class, where `unreal.AssetImportTask().filename` is the source path, and `unreal.AssetImportTask().destination_path` is the target path. Then pass the `task` into `unreal.AssetToolsHelpers.get_asset_tools().import_asset_tasks()` to complete the import action. 

- After import, both the source paths and their corresponding Unreal destination paths are recorded in a dictionary saved to a JSON file, `sp_data.json`. This setup allows the source Substance Painter project to be easily located later from Unreal.

<br />

### Add a Custom Widget into Unreal's Menu

At this point, I’ve completed the Substance Painter plugin for exporting textures and importing them into Unreal. Now, I want to add a right-click option on Texture2D assets in Unreal to open the source Substance Painter project file directly.

The first step is to add the button.

In Unreal, the `unreal.ToolMenuEntryScript` class allows you to create a new entry and register it in an existing menu. This class also has an `execute()` function that can be overridden with custom code to execute specific actions you defined. 

`UEtoSPMenu()`:
```python
@unreal.uclass()
class UEtoSPMenu(unreal.ToolMenuEntryScript):
    @unreal.ufunction(override=True)
    def execute(self, context):
        data = read_json()
        sel_obj = unreal.EditorUtilityLibrary.get_selected_assets()[0]
        sel_obj_path = sel_obj.get_path_name().split('.',1)[0]
        if sel_obj_path in data:
            print(f'Find the file at {data[sel_obj_path]}')
            os.startfile(f'{data[sel_obj_path]}')
        else:
            unreal.log_warning("Open in Substance Painter: the texture doesn't have Substance Painter source file")

```
After defining this class, I can add its instance to a menu using `register_menu_entry()`. Before that, though, I need to find the exact menu name where I want to insert it. To identify existing menu names in Unreal, I used a script to iterate through and print all available menu names:

`list_all_menu_names.py`:
```python
def list_all_menu_names(num=1000):
    menu_list = []
    for i in range(num):
        obj = unreal.find_object(None, "/Engine/Transient.ToolMenus_0:RegisteredMenu_%s"%i)
        if not obj:
            continue
    
        menu_name = str(obj.menu_name)

        menu_list.append(menu_name)
    return menu_list
```
This script iterates through objects named in the format `/Engine/Transient.ToolMenus_0:RegisteredMenu_i`, because all menu objects in Unreal follow this naming convention, with the only difference being the suffix number.

The list of names I got looks like this:
![list-menu-name](/post-img/unrealtools/sp-to-unreal-tool/list-menu-name.png){: width="100%" .shadow}

The one I need for the textures is `ContentBrowser.AssetContextMenu.Texture`. After I know the name of the entry, I can write the function `insert_menu()`:

`insert_menu()`:
```python
def insert_menu():
    menus = unreal.ToolMenus.get()
    asset_menu = menus.find_menu('ContentBrowser.AssetContextMenu.Texture')
    ue_to_sp_obj = UEtoSPMenu()
    ue_to_sp_obj.init_entry(
        owner_name = asset_menu.menu_name,
        menu = asset_menu.menu_name,
        section = "GetAssetActions",
        name = "Open in Substance Painter",
        label = "Open in Substance Painter",
        tool_tip = "Open the texture in Substance Painter"
    )
    ue_to_sp_obj.register_menu_entry()

    ue_to_sp_obj_data = ue_to_sp_obj.data
    ue_to_sp_obj_icon = unreal.ScriptSlateIcon(style_set_name="UMGStyle", small_style_name="Designer.Icon.Small", style_name="Designer.Icon")
    ue_to_sp_obj_data.icon = ue_to_sp_obj_icon

    menus.refresh_all_widgets()
```
In the code above, the parameters within `.init_entry()`, specifically `label`, `name`, and `section`, serve distinct purposes, as illustrated in the image below:

![entry](/post-img/unrealtools/sp-to-unreal-tool/entry.png){: width="100%" .shadow}

p.s. You can turn on the green debug text by **Editor Preferences -> Display UI Extension Points**

<br />

### Open Source Painter Project from Unreal


For opening the source project file, I’ve already stored the source project path and the Unreal destination path in a dictionary within a JSON file when importing the textures. In the `execute()` function of the `UEtoSPMenu()` class, the script reads the path of the currently selected texture and checks if it’s in the dictionary. If it is, it uses`os.startfile()` to open the Substance Painter project directly.

<br />














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