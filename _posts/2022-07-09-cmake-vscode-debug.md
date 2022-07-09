---
layout: post
title: "Debugging CMake based projects"
author: "Shtille"
categories: journal
tags: [CMake,VSCode,VisualStudio,C++]
#image: cards.jpg
---

To debug CMake based project I've chosen Visual Studio Code (VS Code) - crossplatform IDE with many useful plugins.
Each platform has its specifics.

## Platforms
### Mac OS X
1. Install Command line tools (part of Xcode).
2. Install VSCode from its site.
3. Install following plugins:
	- C/C++
	- CMake Tools
4. Open project.
5. Create _launch.json_ in _.vscode_ folder with following content:
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(lldb) Launch",
            "type": "cppdbg",
            "request": "launch",
            "program": "${command:cmake.launchTargetPath}",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "lldb"
        }
    ]
}
```
Where "cwd" is current working directory.
6. Select "(lldb) Launch" configuration either in "Run and Debug" tab and "CMake Tools" panel downside.

### Windows
1. Install latest [Visual Studio Community](https://visualstudio.microsoft.com/en/vs/community/).
2. Install VSCode from its site.
3. Install following plugins:
	- C/C++
	- CMake Tools
4. Open project from Developer Command Prompt:
```
command line: <PathToVSCodeExe> <PathToProjectDirectory>
```
5. Create _launch.json_ in _.vscode_ folder with following content:
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(VS) Launch",
            "type": "cppvsdbg",
            "request": "launch",
            "program": "${command:cmake.launchTargetPath}",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}${pathSeparator}build",
            "environment": []
        }
    ]
}
```
Where "cwd" is current working directory. Choose the one you want.
6. Select "(VS) Launch" configuration either in "Run and Debug" tab and "CMake Tools" panel downside.

Hence Visual Studio Code allows to debug the code its debugging has less options than original Visual Studio has (breakpoint on memory change, memory consumption overview).