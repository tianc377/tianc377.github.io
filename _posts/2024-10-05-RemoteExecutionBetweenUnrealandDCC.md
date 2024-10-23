---
title:  "Remote Execution Between Unreal and DCC"
date:   2024-10-10 21:00
categories: [unrealtools]
tags: [Python, Substance Painter, Maya]
image: /post-img/unrealtools/ue-remote-control-cover.png
published: false
---

Since my last post, when I encountered an [unresolved issue](https://tianc377.github.io/posts/UnrealPythonAssetValidator/#unresolved-issues) while writing the validator, I’ve started working more frequently with remote connections between Unreal, DCC tools, and even Perforce. This is an essential topic in pipeline and tool scripting. Although I had some experience with socket communication in Maya long time ago, I hadn’t tried anything similar between Unreal and other applications. After a few days of exploration, I thought it would be a good time to write everything down before I forget the details.     


## Socket
The first thing I think about remote communication is Socket. I tried with a very simple server script and client script:


```python
# TCP server 
import socket

HOST = '127.0.0.1'
PORT = 65432

def process_message(message):
    string = '----------UE response: get your messages---------------' #message sent to client
    return string

              
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.bind((HOST,PORT))
    s.listen()
    
    conn,addr = s.accept()
    with conn:
        
        while True:
           data = conn.recv(1024)
           print(f'-------Message from client :{data.decode()}') #print message from client
           if not data:
               break
               
           response = process_message(data.decode())
           conn.sendall(response.encode())
```

```python
# TCP client
import socket

HOST = '127.0.0.1'
PORT = 65432

def socketConnection():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((HOST,PORT))
         
        message = "--------Maya's response: I'm Maya!-----------" # message sent to server
        s.sendall(message.encode())
        response = s.recv(1024).decode()
        print(f'--------Message from server: {response}--------') # message from server

socketConnection()
```


## Remote Execution .py




## HTTP signal 

![asset-fails](/post-img/unrealtools/ue-validator/asset-fails.png){: width="80%" .shadow}

