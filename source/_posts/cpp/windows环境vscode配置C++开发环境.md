
---
title: windows环境vscode配置C++开发环境
date: 2023-04-15 10:54:08
tags: 
- cmake
- vscode
categories:  
- cplusplus
---
## 安装GCC

GCC全称是GNU C Compiler ，最早的时候就是一个C编译器。但是后来因为这个项目里边集成了更多其他不同语言的编译器，GCC就代表 the GNU Compiler Collection，所以表示一堆编译器的合集。
MinGW 的全称是：Minimalist GNU on Windows 。它实际上是将经典的开源 C语言 编译器 GCC 移植到了 Windows 平台下，并且包含了 Win32API ，因此可以将源代码编译为可在 Windows 中运行的可执行程序。而且还可以使用一些 Windows 不具备的，Linux平台下的开发工具。一句话来概括：MinGW 就是 GCC 的 Windows 版本 。
### MinGW版本命名方式
MinGW 可以适应不同系统开发环境，因此有几大参数需要进行选择： Version、Architecture、Threads、Exception

Version：指的是你选择的 GCC 编译器的版本，一般建议选择最新的版本；

Architecture：指的是你的电脑的系统类型，i686 表示的是 32 位的系统类型，x86_64 表示的是 64 位的系统类型；

Threads：指的是线程模型，posix 或 win32；如果在 Windows 下开发 Linux 应用程序，则选择 posix；
如果开发 Windows 平台下的应用程序，就需要选择 Win32；

Exception：指的是异常处理模型。i686 系统架构有两种选择：dwarf 和 sjlj；x86_64 系统架构也有两种选择：seh 和 sjlj。其中i686开发32位的应用程序推荐使用dwarf速度更快,x86_64开发64位应用程序使用seh。

MSVCRT 或 UCRT 运行时库：minGW-w64 编译器使用 MSVCRT 作为运行时库，它在所有版本的 Windows 上都可用。由于 Windows 10 通用 C 运行时 (UCRT) 可作为 MSVCRT 的替代品。通用 C 运行时也可以安装在早期版本的 Windows 上（请参阅：Windows中通用 C 运行时的更新）。除非您针对的是旧版本的 Windows，否则 UCRT 作为运行时库是更好的选择，因为它是为了更好地支持最新的 Windows 版本以及提供更好的标准一致性而编写的。

最终MinGw的版本推荐x86_64-posix-seh-urct，下载地址：https://github.com/niXman/mingw-builds-binaries/releases?page=1

### GCC对CPP版本支持情况
![](/img/gcc-cpp-version-support.png)

### 配置MinGW
在windows系统变量中配置MinGW安装目录/bin/目录，既可以在终端中使用gcc.exe,g++.exe编译工具。在bin目录下复制一份mingw32-make.exe为make.exe，这样make工具同样可以使用了。

## 安装Cmake
在https://github.com/Kitware/CMake/releases?page=1下载cmake包。解压安装后，同样windows的系统变量PATH，配置CMake bin目录地址。

## VSCODE配置
1. VSCODE的扩展中搜索一下插件进行安装
- C/C++；
- C/C++ Extension Pack；
### task.json
在.vscode下创建tasks.json文件，它的作用是告诉 VS Code 如何构建（编译）程序，将调用 g++编译器从源代码创建一个可执行文件。
```json
{
    "tasks": [
        {
            "type": "cppbuild",
            "label": "C/C++: g++.exe 生成活动文件",
            "command": "D:\\soft\\mingw64\\bin\\g++.exe",//这里同上面文件，请换成自己存储的g++所在路径    
            "args": [
                "-std=c++11",
                "-O0",
                "-Wl,--stack=268435456",
                "-fdiagnostics-color=always",
                "-g",
                "${file}",
                "-o",
                "${fileDirname}\\hello.exe",
                "-g",
                "-fexec-charset=GBK"
            ],
            "options": {
                "cwd": "${fileDirname}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "detail": "调试器生成的任务。"
        }
    ],
    "version": "2.0.0"
}
```
### launch.json
launch.json是用于运行 ( run ) 和调试 ( debug ) 的配置文件，可以指定语言环境，指定调试类型
```json
// https://code.visualstudio.com/docs/cpp/launch-json-reference
{
    "version": "0.2.0",
    "configurations": [{
        "name": "(gdb) Launch", // 配置名称，将会在启动配置的下拉菜单中显示
        "type": "cppdbg", // 配置类型，对于C/C++可认为此处只能是cppdbg，由cpptools提供；不同编程语言不同
        "request": "launch", // 可以为launch（启动）或attach（附加）
        "program": "${fileDirname}/hello.exe", // 将要进行调试的程序的路径
        "args": [], // 程序调试时传递给程序的命令行参数，一般设为空
        "stopAtEntry": false, // 设为true时程序将暂停在程序入口处，相当于在main上打断点
        "cwd": "${workspaceFolder}", // 调试程序时的工作目录，此为工作区文件夹；改成${fileDirname}可变为文件所在目录
        "environment": [], // 环境变量
        "externalConsole": true, // 使用单独的cmd窗口，与其它IDE一致；为false时使用内置终端
        "internalConsoleOptions": "neverOpen", // 如果不设为neverOpen，调试时会跳到“调试控制台”选项卡
        "MIMode": "gdb", // 指定连接的调试器，可以为gdb或lldb。但我没试过lldb
        "miDebuggerPath": "gdb.exe", // 调试器路径，Windows下后缀不能省略，Linux下则不要
        "setupCommands": [
            { // 模板自带，好像可以更好地显示STL容器的内容，具体作用自行Google
                "description": "Enable pretty-printing for gdb",
                "text": "-enable-pretty-printing",
                "ignoreFailures": false
            }
        ],
        "preLaunchTask": "C/C++: g++.exe 生成活动文件" // 调试前执行的任务，一般先编译程序。与tasks.json的label相对应
    }]
}
```

### c_cpp_properties.json
这个文件用于源代码的语法高亮等，比如有预处理器预定义宏，像WIN32这样的，可以在里面写一下，从而VSCode编辑器能识别出来并对相关代码进行语法高亮或者灰色显示(比如条件编译下在Linux环境中才会进行编译的代码段)

## VSCODE使用Cmake开发CPP工程项目(推荐)
使用task.json配置编译C++一般比较繁琐，这里使用CMake进行编译项目。这里需要自行编写CMakeLists.txt文件进行工程项目编译，然后还是使用Vscode launch.json进行项目调试。修改launsh.json项目程序地址。
```json
// https://code.visualstudio.com/docs/cpp/launch-json-reference
{
    "version": "0.2.0",
    "configurations": [{
        "name": "(gdb) Launch", // 配置名称，将会在启动配置的下拉菜单中显示
        "type": "cppdbg", // 配置类型，对于C/C++可认为此处只能是cppdbg，由cpptools提供；不同编程语言不同
        "request": "launch", // 可以为launch（启动）或attach（附加）
        "program": "${fileDirname}/build/hello.exe", // 将要进行调试的程序的路径
        "args": [], // 程序调试时传递给程序的命令行参数，一般设为空
        "stopAtEntry": false, // 设为true时程序将暂停在程序入口处，相当于在main上打断点
        "cwd": "${workspaceFolder}", // 调试程序时的工作目录，此为工作区文件夹；改成${fileDirname}可变为文件所在目录
        "environment": [], // 环境变量
        "externalConsole": true, // 使用单独的cmd窗口，与其它IDE一致；为false时使用内置终端
        "internalConsoleOptions": "neverOpen", // 如果不设为neverOpen，调试时会跳到“调试控制台”选项卡
        "MIMode": "gdb", // 指定连接的调试器，可以为gdb或lldb。但我没试过lldb
        "miDebuggerPath": "gdb.exe", // 调试器路径，Windows下后缀不能省略，Linux下则不要
        "setupCommands": [
            { // 模板自带，好像可以更好地显示STL容器的内容，具体作用自行Google
                "description": "Enable pretty-printing for gdb",
                "text": "-enable-pretty-printing",
                "ignoreFailures": false
            }
        ]
    }]
}
```
使用CMake编译项目可以得到可执行文件进行调试。
```shell
mkdir build
cd build
cmake -G "MinGW Makefiles" ..
make
```