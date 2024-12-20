---
title:  "Remote Execution Between Unreal and DCC"
date:   2024-10-10 21:00
categories: [unrealtools]
tags: [Python, Substance Painter, Maya]
image: /post-img/unrealtools/ue-remote-control-cover.png
published: true
---

Since my last post, when I encountered an [unresolved issue](https://tianc377.github.io/posts/UnrealPythonAssetValidator/#unresolved-issues) while writing the validator, I’ve started working more frequently with remote connections between Unreal, DCC tools, and even Perforce. This is an essential topic in pipeline and tool scripting. Although I had some experience with socket communication in Maya long time ago, I hadn’t tried anything similar between Unreal and other applications. After a few days of exploration, I thought it would be a good time to write everything down before I forget the details.     


## Socket
The first thing I think about remote communication is Socket. I tried with a very simple server script and client script:

```python
# TCP server
import socket

# Local ip and port
HOST = '127.0.0.1'
PORT = 65432

# Create socket object 
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as server_socket:
    # Bind the host and port with the socket object
    server_socket.bind((HOST,PORT))
    # Start to listen
    server_socket.listen(1)
    print('waiting for client to connect...')
    # Wait for connections
    client_socket, client_address = server_socket.accept()
    print('connection is from: ', client_address)

    # Receive data from the client
    data = client_socket.recv(1024)
    print('data received is: ', data.decode())
    # Send response to client
    message_to_client = 'you have connected the server'
    client_socket.sendall(message_to_client.encode())
    # Clost client connection
    client_socket.close()

```

```python
# TCP client
import socket

HOST = '127.0.0.1'
PORT = 65432

# Create socket object 
client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Connect the to server
client_socket.connect((HOST,PORT))

# Sent message to server
message_to_server = 'Hello server'
client_socket.sendall(message_to_server.encode())

# Receive data from server
data = client_socket.recv(1024)
print('received data from server is: ', data.decode())

# Close client connection
client_socket.close()
```

I executed the server script in Unreal, the client script in Maya. As in below screenshot, Maya got the massage send from Unreal `Hello server`, and Unreal got the message send from Maya `you have connected the server`:
![maya-socket](/post-img/unrealtools/ue-remote-execution/maya-ue-socket2.png){: width="100%" .shadow}

<br />

## remote_execution.py

While exploring socket communication, I discovered that Unreal already provides a Python script for performing remote operations using sockets, with ready-to-use functions. To use Python in Unreal, you usually need to enable the `Python Editor Script Plugin`. Once it's enabled, a folder named `...\Engine\Plugins\Experimental\PythonScriptPlugin\Content\Python` will appear in the engine directory. In this folder, there're two files very useful, one is `debugpy_unreal` which I'll show how to use it to be able to debug Python in Unreal, another one is `remote_execution.py`.

`remote_execution.py` This file encapsulates various functions for Unreal Engine's sub-thread to listen and transmit data with Python, and exposes several functions for users to call. It also provides a simple example at the end of the file:

```python
# Usage example
if __name__ == '__main__':
    set_log_level(_logging.DEBUG)
    remote_exec = RemoteExecution()
    remote_exec.start()
    # Ask for a remote node ID
    _sys.stdout.write('Enter remote node ID to connect to: ')
    remote_node_id = _sys.stdin.readline().rstrip()
    # Connect to it
    remote_exec.open_command_connection(remote_node_id)
    # Process commands remotely
    _sys.stdout.write('Connected. Enter commands, or an empty line to quit.\n')
    exec_mode = MODE_EXEC_FILE
    while True:
        input = _sys.stdin.readline().rstrip()
        if input:
            if input.startswith('set mode '):
                exec_mode = input[9:]
            else:
                print(remote_exec.run_command(input, exec_mode=exec_mode))
        else:
            break
    remote_exec.stop()
```
In this example, it shows usage of some functions:
- Use `RemoteExecution()` to instance a remote execution object
- `.start()` can start the remote execution session. This will begin the discovey process for remote "nodes" (Unreal Editor instances running Python).
- `.open_command_connection()` can open a command connection to the given remote "node".
- In this example provided, it didn't show you how to get the node, but it commented in the file: "emote_node_id (string): The ID of the remote node (this can be obtained by querying `remote_nodes`)". So we can get the node, which is a Unreal Editor instance running Python, by **`remote_node_id = remote_exec.remote_nodes`**
- **`.run_command()`** this function is the one that you can run a command or even a script, and pass parameter into it, it can also return output. It has 3 arguments: command, unattended, exec_mode. 

### Run command and get output

So, how exactly do you get a result or response using the `.run_command()` function? For more details, there’s additional information in a C++ file named `PythonScriptRemoteExecution.cpp`, which is referenced in the first few lines of the `remote_execution.py` script. In the comment of `.run_command()` function, it also notes that:
![return](/post-img/unrealtools/ue-remote-execution/return.png){: width="100%" .shadow}

In `PythonScriptRemoteExecution.cpp`, there's a comment about the `command_result`:
![re-exe-cpp](/post-img/unrealtools/ue-remote-execution/re-exe-cpp.png){: width="100%" .shadow}
By calling the `.run_command()` function, it not only returns whether the command was successfully executed, but it can also return an output dictionary. This dictionary indicates the type of output and includes the log of the execution. This means you can format the information you want to pass as a string, giving you more flexibility in handling responses.


### command_result

I tested printing the command in Perforce since I'm currently writing a script for it:

![output-test](/post-img/unrealtools/ue-remote-execution/output-test.png){: width="100%" .shadow}

In the script, at the top-left corner, I simply used `.run_command` to execute another function, `error_window()`, from a separate Python file, using `unreal.log_warning()`. On the right side, the command_result displayed keys like `success`, `command`, `result`, and `output`. Inside the `output`, it provided the type as info and returned the warning message I had written in the script.

You may have noticed that the second Python file `perforce_validator_in_ue.py`, which is called by the `run_command()` function, is executed inside Unreal since I was able to use `import unreal` without any issues. This is super useful because the file that executes the remote command is usually within a DCC tool, where Unreal Python modules aren’t accessible. With this remote execution setup, any code that needs to run in Unreal’s Python environment can be placed in a separate script, called inside of Unreal later after the remote connection, and the results can be sent back to the DCC tool. This makes communication between Unreal and external applications much easier.







<br />

## HTTP signal 

I also explored another method commonly used in Unreal's Remote Control feature. Unreal offers a functionality called [Remote Control](https://dev.epicgames.com/documentation/en-us/unreal-engine/remote-control-for-unreal-engine), which allows you to for example create a standalone web page with a user interface, such as sliders or color pickers, then can use them to control light intensity or adjust the material albedo, that, properties inside of engine. 
There're some official tutorials from Unreal, showing how to use the HTTP remote control: [Getting Started with Unreal Engine and Remote Control API | Unreal Fest 2022](https://www.youtube.com/watch?v=XapljWLWnsk&t=837s)

Below is the code of my file: 

```python
import requests 
url = "http://localhost:30010/remote/object/call"
payload = {
    "objectPath": "/Engine/PythonTypes.Default__HTTPConnectUnreal",
    "functionName": "get_input",
    "parameters": {"string": f"{ue_files_paths}"}
}
headers = {
    "Content-Type": "application/json",
    "User-Agent": "insomnia/10.1.0"
}
response = requests.request("PUT", url, json=payload, headers=headers)
print(response.text)
```

- So the above block of code can be generated from API platform like Insomnia or Postman: 
    ![insomnia](/post-img/unrealtools/ue-remote-execution/insomnia.png){: width="90%" .shadow}
    
- The `url` line after `PUT` follows a fixed format. The variable part is the localhost port number, which you need to check in your Unreal Project settings:
    
    ![port-number](/post-img/unrealtools/ue-remote-execution/port-number.png){: width="90%" .shadow}

- In the body, the object path is `/Engine/PythonTypes.Default__` + `class name`. To ensure the class is found, use Unreal decorators `@unreal.uclass()` and `@unreal.ufunction()` to label them (refer to the code above the Insomnia window).

- Then `functionName` is simply the function name you gonna call.
- The parameters correspond to the function’s arguments; in my example, I used a string.
- Then the rest `headers = {}`, is generated from Insomnia:

    ![insomnia2](/post-img/unrealtools/ue-remote-execution/insomnia2.png){: width="90%" .shadow}

- Simply copy this code and run it in any DCC that supports Python, and it will send the parameters into Unreal.
- You’ll need to install the requests module if you don’t already have it.


<br />

## Tips: Scripting with Unreal Python

### Plugins
There are several plugins and features that need to be enabled to access the full functionality related to Python, remote control, and remote executions in Unreal:
![plugins](/post-img/unrealtools/ue-remote-execution/plugins.png){: width="100%" .shadow}
![project-setting](/post-img/unrealtools/ue-remote-execution/project-setting.png){: width="100%" .shadow}

<br />

### Auto complete 
- Find the `setting.json` file of Python extension setting:
![auto-complete](/post-img/unrealtools/ue-remote-execution/auto-complete.png){: width="100%" .shadow}
- Add the project python folder (where the `unreal.py` located) into the `extraPaths`: 
![vscode-settings-json](/post-img/unrealtools/ue-remote-execution/vscode-settings-json.png){: width="100%" .shadow}
- Voila, I have auto completion when I type unreal API:
![auto-complete](/post-img/unrealtools/ue-remote-execution/auto-complete.gif){: width="100%" .shadow}

<br />

### debugpy_unreal.py
- In `...\Engine\Plugins\Experimental\PythonScriptPlugin\Content\Python` there's a file named `debugpy_unreal.py`. This file has some instructions to setup the debug feature in VSCode:

![debugpy](/post-img/unrealtools/ue-remote-execution/debugpy.png){: width="90%" .shadow}

- However, what this file actually does is use pip to install the `debugpy` module in the engine project. I didn't want to create or change anything in the engine project, so I installed it inside my CTLib plugin folder:

![install-debugpy](/post-img/unrealtools/ue-remote-execution/install-debugpy.png){: width="100%" .shadow}

- After installation, the module appeared inside of my plugin's `Python` folder: 

![plugin-folder](/post-img/unrealtools/ue-remote-execution/plugin-folder.png){: width="100%" .shadow}

- Once installed and Unreal restarted, follow the instructions from the `debugpy_unreal.py` file:

![debugpy-step1](/post-img/unrealtools/ue-remote-execution/debugpy-step1.png){: width="90%" .shadow}

- After input `import debugpy_unreal` and `debugpy_unreal.start()`, Unreal will be frozen, 

- Next, use the configuration code from that `debugpy_unreal.py` file in `launch.json`, which you can find by clicking on `Run and Debug` in VSCode:

![launch-json](/post-img/unrealtools/ue-remote-execution/launch-json.png){: width="100%" .shadow}

Inside `pathMappings`, I used the path of the Python folder in my plugin where the debugpy module and all my Python scripts are saved.

After saving the `launch.json` file, run the debugger and set breakpoints in the file you need to debug. Then, import that file inside Unreal, and it will stop at the breakpoint:

![debug-start](/post-img/unrealtools/ue-remote-execution/debug-start.png){: width="100%" .shadow}
