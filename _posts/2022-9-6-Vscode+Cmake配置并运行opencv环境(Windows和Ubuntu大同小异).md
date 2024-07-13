---
layout: post
title: Vscode+Cmake配置并运行opencv环境(Windows和Ubuntu大同小异)
subtitle: vscode+cmake配置；封面：永清
author: 永清
categories: OpenCV
banner:
  image: /assets/images/banners/3.JPG
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: 环境配置 OpenCV
---

## 0.CMkae简介

CMake是一个开源免费并且跨平台的***构建工具***，可以用简单的语句来描述**所有平台**的编译过程。它能够根据当前所在平台输出对应的makefile或者project文件，并且主流的各种库均提供了.cmake文件提供调用支持，使用方便。在opencv上说，使用它可以使多个版本的CV环境共存，且随意调用任意切换互不影响。

如果你使用gcc/g++开发套件供cmake编译，那么你在各个平台上获得的报错信息都是一样的，不会随着使用开发平台的变化而变化(对，说的就是qt的阴间报错)

需要注意的就是cmake仅仅是**构建工具**，**连编译器都不是**。需要你**指定编译器**(开发套件)，一般windows下面指定的是MinGw，Ubuntu下指定自带的gcc/g++。

它和轻量美观的vs code结合可谓是双剑合璧，十分建议学习。

## 1. 安装VS Code
[VS code官网](https://code.visualstudio.com/Download)
点此下载vs code，需要梯子才能全速下载，没有的自行想办法。

## 2.安装cmake
### windows

打开[CMake官网](https://cmake.org/download/)，下载windows版本(.msi文件)
![cmake版本选择](/assets/images/2/1.png)
下载完毕后，运行之。
Next->I accept the License & Next ->选择Add CMake to the PATH for all users，不然你还得自己去添->自己设定一个喜欢的安装路径 & next->Install 弹出的请求更改对话框选“是”

安装完毕后，查看环境变量
win11直接在设置里面搜索“环境变量”就可以了，win10进入windows设置，点击系统，高级系统设置，环境变量。
![环境变量](/assets/images/2/2.png)
看看你的PATH环境变量里面有没有这个，当然路径可能不一样，但是最终要指向你的cmake的bin文件夹下。

如果确认无误，那么就算安装好了

### Ubuntu
```bash
sudo apt-get install cmake cmake-gui
```
完事

## 3.vs code初始设置
在左侧的侧边栏中找到插件图标，一般是个小拼图或者是四个小方块之类，安装如下插件：
c++开发基础插件，提供c++基础的开发能力
<br>
![c++插件](/assets/images/2/3.png)
<br>
CMake插件，提供cmake调用能力
<br>
![cmake插件](/assets/images/2/4.png)
<br>
其他插件，如：code runner（运行器）、Chinese(简体中文包)
可选个人推荐插件：background-cover（换背景）markdown all in one（提供markdown编辑预览能力）polacode-2022（能够制作漂亮的代码片段截图）

安装完毕后，叉掉code重新打开。
检查你的侧边栏是否有cmake这一栏，如果有，请保持打勾状态
<br>
![cmake菜单](/assets/images/2/5.png)
<br>

## 4.cmake插件设定

使用快捷键ctrl+shift+p调出命令面板，输入cmake:kit，选择**扫描工具包**，
![扫描工具包](/assets/images/2/6.png)
过一会应该会弹出扫描到的工具包，如果没有，请你手动点选择工具包。

![选择工具包](/assets/images/2/7.png)
我这里是因为安装了visual studio所有有这么多套件，新生的电脑上如果没有安装visual studio，是没有这么多的。总之，选择一个你喜欢的开发套件即可。需要注意的是，如果不是visual studio的套件，需要看它显示的using compliers后面是不是c和CXX都有，不然你不能编译c++。一般来说都有。单击喜欢的套件使用之。

## 5.安装opencv

[opencv下载链接](https://opencv.org/releases/page/3/)
拉到底，看到3.4.6版本，点击windows，下载下来即可。

3.4.6是个人目前来说遇到问题最少的版本。之前电脑上的opencv4有问题，同样的代码在3.4.6里编译能过，cv4就有一个函数链接的时候报错。而且cv4还有solvepnp的异常问题，不建议安装。如果你非要尝鲜，建议cv3、4同时安装，选择调用。

点击打开，设置解压缩路径。
![cv解压缩](/assets/images/2/8.png)
完事。不需要配置什么环境变量。至于为什么待会说。

### ubuntu
ubuntu环境下不能这样直接安装，需要编译安装。编译安装的教程网上非常多我就不再赘述，但是大致步骤为：
安装依赖包、编译、安装、添加环境变量、更改bash文件这几步。
编译完毕之后，记住你安装库文件的位置，做下面的步骤：

## 6.实际工程调用
实际工程调用的时候，只需要在cmakelist里面添加如下语句即可指定版本并链接、包含相关的库、头文件。大小写请严格遵守！

> cmake查找库的原理是：调用库自带的config文件，根据其指引添加头文件、库文件列表。所以我们只需要指定其config文件的地址就可以。config文件一般都是库名+config.cmake命名，如：若为opencv库，则为：OpenCVConfig.cmake，你set一个dir到它所在位置就可以让cmake自行调用

```c
set(OpenCV_DIR  D:/opencv/opencv3.4.6/opencv/build/x64/vc15/lib/)
find_package(OpenCV REQUIRED)
```
需要注意的是，set的opencv dir是set到你需要的库目录下面，并且务必存在.cmake引导文件，例如：
![查找cvconfig](/assets/images/2/9.png)
这俩opencv_world346.lib就是opencv的库文件，下面的OpenCVConfig.cmake就是cmake指引文件，而版本是x64、vc15，一般来说你x64文件夹下应该还有一个vc14，如果你有需要，可以设定到那里。一般来说设定在vc15下面就可以。

OpenCVConfig.cmake为我们制作好了头文件、库文件列表，调用方式如下：
```c
#添加头文件
include_directories(${OpenCV_INCLUDE_DIRS})
#链接库文件
target_link_libraries(main  ${OpenCV_LIBS})
```
至于为什么是这两个变量，你可以自己去看opencv的cmake引导文件，里面写的明明白白，代码之下无秘密。
![opencvconfigcmake](/assets/images/2/10.png)
## 7.引例
> 虽然配置到上面就已经结束了，但是还是想做一个例子工程，以免有人不会用。如果你目前还不会cmakelist的语法，那赶紧去学。不要指望着什么都有人来教。C站上教程多的是。

1.新建文件夹
2.打开vscode， ctrl+k ctrl+o打开刚刚创建的文件夹。
3.创建如下文件：
main.cpp
```cpp
#include <iostream>
#include <opencv2/opencv.hpp>

int main()
{
    cv::Mat img;
    img = cv::imread("D:\\c++\\showCV\\1.jpg"); //这里放你图片的绝对路径，注意是双斜杠

    cv::imshow("show", img);
    cv::waitKey(0);

    return 0;
}
```

CMakeLists.txt
```c
cmake_minimum_required(VERSION 2.8)

project(test)

set(DEFAULT_CXX_STANDARD 11)

set(OpenCV_DIR  D:/opencv/opencv3.4.6/opencv/build/x64/vc15/lib/)#库路径你自己换成你电脑上的

find_package(OpenCV REQUIRED)

include_directories(${OpenCV_INCLUDE_DIRS})

add_executable(main ${PROJECT_SOURCE_DIR}/main.cpp)

target_link_libraries(main  ${OpenCV_LIBS})
```
**拷贝一张你喜欢的图片进去，命名为1.jpg**

文件夹结构目录如下：(build是cmake自动生成的，如果你没有请不要急)
![folder](/assets/images/2/11.png)
确定你是否已经选择开发套件，如果没有，打开cmakelist，ctrl+s保存，它会弹出让你选择开发套件。

下图为vscode底边栏。第一个是选择优化模式，可以选择debug、release等，第二个是开发套件，点击更改或者选择。第三个是build，点击编译，第四个是选择目前编译的目标，如果你有多个生成，可以选择你想要生成的而不是编译全部。小臭虫是启动debug，这个很好用，自己去查怎么配。小三角是运行，后面那个main是选择当前你想要运行的目标。
![underbar](/assets/images/2/12.png)

在cmakelist内每次更改完**一定ctrl+s保存**，cmake会自动更新构建方式。下面自动弹出输出框，你看有没有类似这样的提示：
![cmakeoutput](/assets/images/2/13.png)
如果输出显示configure down和generating down，代表构建成功。点击底边栏的小齿轮build即可编译。
![complier](/assets/images/2/14.png)
无误后点击小三角即可运行，弹出你橙的图片：
<br>
![run](/assets/images/2/15.png)
<br>
2022.10.17 补充一点。你们应该注意到了，我这个代码里面读取图片写的是绝对路径。因为有一点诸君一定要注意，就是CMake编译文件夹是这么组织的：
<br>
![目录](/assets/images/2/16.png)
<br>
就是，你编译出来的可执行文件在`build`文件夹下面的`Debug`文件夹下（如果你选择的优化级别是`Release`，那么就是在`Release`文件夹下）。所以你的**执行路径不是在工程文件夹根目录，而是在`工程根目录\build\Debug\`文件夹下**。所以你如果读图写的是
```cpp
img = imread("1.jpg");
```
那么你**必须把图复制一份放到`工程根目录\build\Debug\`文件夹下**，否则必然会报错。比如我上面截的那张图，只有`工程根目录\build\Debug\`文件夹下的那个`map.jpg`是有效的，其他几个都是没有用的，这一点务必要注意。如果你觉得拷图片进去麻烦你就写绝对路径，免得生事。

> 结语：配置教程到这里就结束了，其实还有很多相关知识，我没有写。你们读完也肯定不是立即就都会写`cmakelist`，但是请自己去查一查，真的不难。而我推荐学习cmake的原因是，它比集成IDE如visual studio、Qt要底层，但是又比`makefile`要简洁，`makefile`繁琐至极，visual studio、Qt之类虽然好用，但是作为程序员，编译器怎么找到头文件、怎么找到库文件，什么时候链接库的，都应该知道。而且这个是跨平台的，我这个示例工程拿到unix上一样好用，直接就可以编译，你在visual studio里面配的工程放到ubuntu上面，如果不重新配置，你看能跑不能？

2023/5/24更新：
由于鄙人先前学识所限，说明以下常见问题：
1. 为什么我用vs的编辑器可以build，用装的gcc就不可以呢
在opencv官网下载的windows安装包，也就是exe那个可以一键安装的，是专供vs编译器使用的，但是其中附带源码，如果你想要用GCC编译器来调用OpenCV也可以，要用GCC编译出静态库方可调用，我这里简述一下：
- 下载MinGW-posix版本编译器，这个posix是什么呢，就是windows版本下附带多线程库的MinGW，普通版本是不附带多线程库的，编译源代码的时候会报错，这一点一定要注意。
- 把MinGW套件路径加入环境变量，用CMAKE指定该套件进行编译。
- 编译出来的库目录加入环境变量，这样MinGW编译你的工程的时候才能链接到库文件，据我试验，CMAKE使用set OpenCV_DIR设定路径对MinGW编译器的target_link_libraries无效，所以需要加入环境变量。

1. 如果你不放心，你可以把OpenCV_DIR添加到环境变量里面，也就是文中提到的opencv库（cmake引导文件）的路径
2. 如果你配置遇到问题，请把问题粘贴详细，报错信息、环境信息，不要只说我遇到错了，别人想帮你也帮不了