---
title:  "Use Perforce Triggers for Submission Validation"
date:   2024-10-20 18:00
categories: [unrealtools]
tags: [Python, P4V]
image: /post-img/unrealtools/perforce-trigger-cover.png
published: true
---

In my post on Unreal validation, I mentioned an unresolved issue where I wanted to see if there was a way to trigger validation directly from Perforce instead of using Unreal’s validation class. This led me to research further, and I found a feature in the Perforce documentation:p4 triggers. Use ‘triggers’ can activate a command when certain actions, such as submitting content, are executed.

<br />

## Build a Personal P4 Server
To better work with perforce, the first thing I need is to have a server to make my own depot. Noticed that this is the server application, not the client application (p4v). 
### Install Helix Server 
- Select `Helix Server` and `Helix Command Line`:

![helix-install](/post-img/unrealtools/perforce-trigger/helix-install.png){: width="100%" .shadow}

- Port number is 1666 by default:

![helix-install](/post-img/unrealtools/perforce-trigger/helix-install-config1.png){: width="100%" .shadow}

- Server: `1666`
- User Name: mine is `tiancaoadmin`
- Text Editing Application: mine is Notepad.exe

![helix-install](/post-img/unrealtools/perforce-trigger/helix-install-config.png){: width="100%" .shadow}

### Install P4V
- Select all:

![p4v-install](/post-img/unrealtools/perforce-trigger/p4v-install.png){: width="100%" .shadow}

- This window will show again, fill the same info as before:

![p4v-install2](/post-img/unrealtools/perforce-trigger/p4v-install2.png){: width="100%" .shadow}


### Login
- Turn on P4V, create a new user:

![login](/post-img/unrealtools/perforce-trigger/login.png){: width="100%" .shadow}

- Fill the following info:

![new-user](/post-img/unrealtools/perforce-trigger/new-user.png){: width="100%" .shadow}

- Back to `Open Connection` window and click `OK`, then will enter the main window. Click `Tool->Administration`, it gonna ask if you would like to have super access, click yes. In the following window, I can see my account has super access level:

![admin](/post-img/unrealtools/perforce-trigger/admin.png){: width="100%" .shadow}

### Create a Depot and a Stream
- In the `Tool->Administration` window:

![create-depot](/post-img/unrealtools/perforce-trigger/create-depot.png){: width="100%" .shadow}


- Input the following info, for the `Depot Type` select `stream`:

![depot](/post-img/unrealtools/perforce-trigger/depot.png){: width="100%" .shadow}


- After created a depot, back to the main window, select the new created depot, then:

![stream](/post-img/unrealtools/perforce-trigger/stream.png){: width="100%" .shadow}


- Whithin the new popup window, for `Stream Type` select `mainline`:

![stream-config](/post-img/unrealtools/perforce-trigger/stream-config.png){: width="100%" .shadow}


### Create an End User
- To create an end user, would need to create a group firstly:

![create-group](/post-img/unrealtools/perforce-trigger/create-group.png){: width="100%" .shadow}


- Input a user name then click Add, the added user will appear in the above list:

![group-config](/post-img/unrealtools/perforce-trigger/group-config.png){: width="100%" .shadow}


- Then under Groups, that user just added can be found, next, right click on the user, find `Create User From...`:

![create-user](/post-img/unrealtools/perforce-trigger/create-user.png){: width="100%" .shadow}


- After, fill in corresponding info for this new user:

![user-config](/post-img/unrealtools/perforce-trigger/user-config.png){: width="100%" .shadow}


- New created user will appear under the users list:

![new-user2](/post-img/unrealtools/perforce-trigger/new-user2.png){: width="100%" .shadow}



### Use the New End User Account to Login 
- After created the new end user and grant access, I can use this user account to login from another machine to this server, but need to find the server's IP firstly by using cmd line `ipconfig`:

![ipconfig](/post-img/unrealtools/perforce-trigger/ipconfig.png){: width="100%" .shadow}


- Use the `IPv4 Address:` + 1666 as the server address:

![test-user-login](/post-img/unrealtools/perforce-trigger/test-user-login.png){: width="100%" .shadow}


- If you get an connection error like `WSAETIMEDOUT`, try to add the port 1666 into the `Inbound Rules` of Windows Firewall setting:

![firewall](/post-img/unrealtools/perforce-trigger/firewall.png){: width="100%" .shadow}


<br />

## Perforce Triggers

Triggers are customized scripts that are called when certain operations are performed. 
The [Triggers](https://www.perforce.com/manuals/p4sag/Content/P4SAG/chapter.scripting.triggers.html) documentation also mentions that ‘If the script returns a value of **0**, the operation **continues**. If the script returns **any other value**, the operation **fails**.’ 
On [Sample Trigger](https://www.perforce.com/manuals/p4sag/Content/P4SAG/scripting.trigger.sample.html) page, it says 'To define this trigger, use the p4 triggers command, and add a line like the following to the triggers form: `bypassad auth-check auth "/home/perforce/bypassad.sh %user%" `'

Try use `p4 trigger` command from the depot, there's a txt file popped up:
![triggers-txt](/post-img/unrealtools/perforce-trigger/triggers-txt.gif){: width="100%" .shadow}

You can see in this txt file, under those command lines, there's a `Triggers:` line that you can fill your command whithin it.

### Trigger Form
Use my code as an example, the trigger form has a format:

![trigger-cmd](/post-img/unrealtools/perforce-trigger/trigger-cmd.png){: width="100%" .shadow}

- `ChangelistValidator` is this trigger's name, you can name it whatever you want;
- `change-content` is the `Type`, there're many different Type can be used:

![type](/post-img/unrealtools/perforce-trigger/type.png){: width="90%" .shadow}


- `//ctproject/...` is a `Path`, here I just used my depot root path;
- Next, the actual command that this trigger will execute is quoted: `C:\Users\Jovian\AppData\Local\Programs\Python\Python311\python.exe D:\Projects\github\real-playground\RealPlayground\Plugins\CTLib\Content\Python\perforce_validator.py `, here in my example, I let the trigger use python.exe to execute a python script. 

- The last, and very important element is **`%changelist%`**, this is actually a [**trigger script variable**](https://www.perforce.com/manuals/p4sag/Content/P4SAG/scripting.triggers.variables.html) that can be used later in the python script, can be used later in the Python script. In my case, I can retrieve the current changelist number being submitted from this `%changelist%` label. 


<br />

## Trigger Script
After saving the newly added trigger in the trigger file, it's time to look at the Python script that I want the trigger to execute.

I want to write a script that remotely connects to Unreal Engine to execute another script within it. This Unreal script will check the size of the textures being submitted in the changelist and return the validation result to the trigger script, allowing Perforce to either pass or fail the submission.

### Connect to P4 Server
First, within this script, I need to use P4 commands to create a P4 object and provide necessary details like the port number, username, client name, and password. This information can be found using the command `p4 info`:
```python
p4= P4.P4()
p4.port = "1666"
p4.user = "tiancaoadmin"
p4.client = "tiancaoadmin_DESKTOP_634"
p4.password = "********"
p4.connect()
p4.run_login()
```
<!-- ![p4-script-login](/post-img/unrealtools/perforce-trigger/p4-script-login.png){: width="100%" .shadow} -->
![p4info](/post-img/unrealtools/perforce-trigger/p4info.png){: width="100%" .shadow}

### Use argparse to Get the Argument
Remember when adding the command line in the trigger file, I included the trigger variable `%changelist%` at the end? In this script, I need to parse it to extract the changelist number:

```python
import argparse

global ARGS
parser = argparse.ArgumentParser()
parser.add_argument('changelist',
                    help="current # of changelist being submitted")
ARGS = parser.parse_args()
cl = ARGS.changelist
```

### Get File Paths 
Since I’ll need to check files within Unreal Engine later, I first need to know their paths in Unreal to locate them. To do this, I’ll use the changelist number of the current submission to retrieve the files in this changelist. Perforce has a command -- [**opened**](https://www.perforce.com/manuals/cmdref/Content/CmdRef/p4_opened.html) which lists files that are open in pending changelists.

I also skipped files marked as delete since there’s no need to validate the sizes of files being deleted.

To be noted, for Perforce commands like `opened`, `changes`, `files`, you can always use `p4.run()` to execute them in Python.
```python
info = p4.run( "info")
client_stream = info[0]['clientStream']
ue_files_paths = []
current_cl = p4.run("opened", "-c", f"{cl}")
for item in current_cl:
    if (item['action'] != 'delete'): # delete files don't need to be validated
        p4v_file_path = str(item['depotFile'])
        ue_file_path = p4v_file_path.replace(f'{client_stream}' + '/Content', '/Game') 
        ue_file_path = ue_file_path.split('.', 1)[0]
        ue_files_paths.append(ue_file_path)
```

### Remotely Connect to Unreal
Next, I’ll send the gathered paths to another script that will run in Unreal. This script will validate the sizes of the textures being submitted. I’ve already covered the details of Unreal’s `remote_execution` module, check it out in this post. [Remote Execution Between Unreal and DCC](https://tianc377.github.io/posts/RemoteExecutionBetweenUnrealandDCC/).

```python
remote_exec = remote.RemoteExecution()
remote_exec.start()
time.sleep(2) # sleep 2 to let start() work, otherwise no remote node available
# Ask for a remote node ID
remote_node_id = remote_exec.remote_nodes

# Connect to it
if remote_node_id:
    remote_exec.open_command_connection(remote_node_id)
    exec_mode = 'ExecuteFile' # use this mode to execute a file directly
    # input = sys.stdin.readline().rstrip()
    remote_exec.run_command('import perforce_validator_in_ue', exec_mode=exec_mode)  
    remote_exec.run_command('import importlib', exec_mode=exec_mode)
    remote_exec.run_command('importlib.reload(perforce_validator_in_ue)', exec_mode=exec_mode)
    rec = remote_exec.run_command(f"perforce_validator_in_ue.error_window({ue_files_paths})", exec_mode=exec_mode, raise_on_failure=True)    
    validation_result = rec['output'][0]['output']
    if validation_result == [] : #if the list is not empty
        print('----Validation Passes-----')
        exit(1)
    else:
        print(f'{validation_result}')
        print('----Validation Failed-----')
        exit(1)
else:
    print('----Validation Failed with invalid remote node id-----')
```

### The Validation Script Executed in Unreal

In the code above, on line 12, `perforce_validator_in_ue` is my separate script that will be executed in Unreal. This script loads the files in the specified changelist, checks if they’re textures, and then verifies their sizes. If any textures exceed 2048, their names are added to a list called `invalid_assets = []`, and the script will return a `unreal.log_warning()` listing those textures.

```python
import unreal 

def error_window(ue_files_paths):
    invalid_assets = []
    for obj in ue_files_paths:
        asset = unreal.EditorAssetLibrary.load_asset(obj)
        if asset.get_class().get_class_path_name().asset_name == 'Texture2D':
            texture_width = asset.blueprint_get_size_x()
            texture_height = asset.blueprint_get_size_y()
            if (texture_width> 2048) or (texture_height> 2048):
                # print(asset.get_name())
                if asset.get_name() not in invalid_assets:
                    invalid_assets.append(asset.get_name())
            else:
                pass
        else:
            pass
    # unreal.log_error(f'{flag}')       
    if invalid_assets:
        data = ', '.join(invalid_assets)
        unreal.log_warning(f'--- {data} is/are larger than 4096 ---')    
```

<br />


## Result
This is how the script looks like:
![p4-validator](/post-img/unrealtools/perforce-trigger/p4-validator.gif){: width="100%" .shadow}

<br />


## Unresolved Issue
Yeah yeah, there's still an issue with this approach. 

Initially, I wanted to execute the validation directly from Perforce because Unreal has a submit button that can bypass the Unreal Validator system. After completing this Perforce trigger validator, I tested submissions directly in Perforce, and it worked perfectly. 

However, it didn’t work when using the source control UI in Unreal; the scripts caused Unreal to freeze. Upon investigation, I discovered that when Unreal's source control submission window is open, it does not receive any remote execution signals, causing that thread to stop. As a result, the script gets stuck waiting for a connection.

This is why I eventually returned to using Unreal's validation system. However, the Perforce validation is still quite useful such as validating raw data submitted from DCC, ensuring correct naming conventions, and checking directory structures, among other things.













