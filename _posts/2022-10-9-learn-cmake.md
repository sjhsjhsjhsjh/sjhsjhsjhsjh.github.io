---
layout: post
title: CMake初级简明教程
categories: cmake
# banner:
#   image: /assets/images/banners/4.jpg
#   opacity: 0.618
#   background: "#000"
#   height: "100vh"
#   min_height: "38vh"
#   heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
#   subheading_style: "color: gold"
tags: cmake c++
---

> 我藏在人群中  然后失去晴空
> 像我的名字  从你的记忆清空。		---《你头顶的风》

本文仅仅介绍了一些常用函数的参数的**简单写法**以及**个人理解**，如果你想了解更多，可以去查找**专门介绍该函数**的博客。本文只能保证你读完之后对各个函数有基本的认识，以及最基本的应用。
个人认为，想要提高自己写`cmakelist`能力的最好方法有两个。其一者，自己编写更加复杂的工程文件。其二者，阅读、修改现有库的`cmakelist`。有的库的`cmakelist`有上百行（比如`PCL`库），但是其实看明白之后就是很简单的几个模块判断、引入。尤其是后者，小学写过的字帖都有三部分：描红，仿影，临帖。先看人家怎么写，学会笔法之后再写自己的，编程学习之法可谓相同。
笔者才疏学浅，知识均由自学得来，故可能多有错漏。本文仅作抛砖引玉之用，欢迎读者斧正。

## 1.cmake_minimum_required

含义：要求cmake的最小版本
示例：
```cpp
cmake_minimum_required(VERSION 2.8)
```
说明：

> cmake_minimum_required(VERSION)命令隐式调用cmake_policy(VERSION)命令，以指定为给定范围的cmake版本编写当前项目代码。

个人理解：
cmake各个版本在查找文件的策略等等上会有一些差异。通过这个函数指定最小版本。以保证当前的策略是你想要的策略。
对于初学者，无需管这个，随便指定一个就可以，例如`VERSION 2.8`.

## 2.project

含义：指定本工程名称
示例：
```cpp
project(test)
```
个人理解：
指定工程名称，随便起一个名字就可以。需要注意的是，这个指定的**不是**编译出来的`exe`或者是可执行文件的名字，那个是通过另一个函数指定的。这个就是工程的名字。

## 3.set

含义：设定变量的值
示例：
```cpp
set(PCL_ROOT "D:/program-tools/PCL/PCL 1.12.1") //设定PCL_ROOT值为后面的路径
set(_waiting_for_debug 0)	//设定值为0
set(DEFAULT_CXX_STANDARD 11)  //设定值为11
```
个人理解：
这个函数基本可以看作赋值用的，需要注意的是，编译器往往有一些预定义的值，例如示例里面的`DEFAULT_CXX_STANDARD `，这个是编译器预定义的值，通过将其置为11，可以设定编译规范为`C++11`.
这个函数另一个重要作用就是指定库的路径，例如示例中的第一行，指定了`PCL`库的`ROOT`根目录。这一点我们下面也会谈到。

## 4.find_package

> 这个函数**非常非常非常重要**，你编译的时候找不到包大部分都是因为这个函数**使用不当**引起的。而且有些问题非常隐秘。这一部分可能有点多，但是请**务必看下去**，不然到时候你debug一天也找不到问题的时候可不要后悔哦。

- 含义：寻找包
一般来说，包都是网上流行的一些库的合集，譬如PCL点云库，OpenCV图像库等等，都是很多功能、库的集合。cmake作为工程组织工具，怎么可能知道你把某库放在电脑的哪个犄角旮旯里了？所以它需要一个引导文件，来告诉它，这个包的库目录在哪，头文件在哪，怎么去查找后续的依赖库。下面有一段官方一点的说明。

> cmake本身**不提供**任何搜索库的便捷方法，所有搜索库并给变量赋值的操作必须由**cmake代码**完成，比如FindXXX.cmake和XXXConfig.cmake。只不过，库的作者通常会提供这两个文件，以方便使用者调用。

例如，以著名的点云库PCL为例，其部分文件夹结构如下：
```dark
---PCL
	|---PCL 1.12.1
		|---include
			|--- ...（略，一些头文件）
		|---lib
			|--- ...（略，一些库文件）
		|---cmake
			|---PCLConfig.cmake
			|---PCLConfigVersion.cmake
			|---Modles
				|---FindEigen.cmake
				|---FindFLANN.cmake
				|---FindQhull.cmake
				|---FindOpenMP.cmake
				|--- ...（略，其余的依赖cmake文件）
		|--- ...（其他文件夹）
```
观察上面的文件夹结构，作者特意把cmake文件独立一个文件夹，而其库引导文件`PCLConfig.cmake`就在cmake目录下，我们不妨称之为根引导文件。在这个根引导文件的同级目录下，有一个`Modles`文件夹，里面的文件都是诸如`Findxxx.cmake`之类，顾名思义，显然是找到PCL库的其他依赖库的引导文件。我们不用打开根引导文件也能猜到，想必是这个`PCLConfig.cmake`调用了文件夹里面的这些小的查找文件，去查找所需的依赖库。所以怎么办，你只要让cmake找到这个`PCLConfig.cmake`根引导文件就可以了。这个根文件会自动调用其他查找文件。

- 函数基本示例：
```cpp
find_package(OpenCV REQUIRED)
```
- 参数说明：
前者是包的名字，后者是附加描述。
`REQUIRED`：这个包**必须**要找到，找不到就会报错，下面的都不会执行。
例如下图，我命令去找DNN的包，但是它说：
“找不到一个由下列名字提供的DNN包的配置：
`DNNConfig.cmake`
`dnn-config.cmake`”
![dnnrequired](/assets/images/7/1.png)
各位读者务必记住，这就是没有找到DNN这个包的意思。那么如何令其找到呢？
答：通过使用上面介绍的`set()`函数，设定库DNN的路径，从而找到其配置文件`DNNconfig.cmake`。
例如，你要引入的库是`PCL`库，那么怎么办，你需要找到你PCL库源代码所在位置，找到`PCLConfig.cmake`，设定`PCL_DIR`值为该文件路径即可。
例如：
```cpp
set(PCL_DIR  "D:/program-tools/PCL/PCL 1.12.1/cmake")
find_package(PCL REQUIRED)
```
这样就不会再报错了。
再比如，我要引入Opencv，怎么办：
```cpp
set(OpenCV_DIR  D:/opencv/opencv3.4.6/opencv/build/x64/vc15/lib/)
find_package(OpenCV REQUIRED)
```
你可能注意到，这两个DIR一个是字符串，一个不是，这个不重要，两个写法都是可以的。

- 总结：
该函数往往与`set()`函数配套使用，例如你所需要的库名为ABC，则你需要定义`ABC_DIR`到你的库引导文件下，再调用`find_package()`
```cpp
set(ABC_DIR "your path")
find_package(ABC REQUIRED)
```

- 扩展阅读资料：
描述子不止有`REQUIRED`，还有一些其他的。不过对于初学者而言，一般你所需的库肯定是必须的，而不是可选的。所以如果你有需要，请自行查阅其他资料，这里仅介绍`REQUIRED`。

- `find_package`采用两种模式搜索库：
1. `Module`模式：搜索`CMAKE_MODULE_PATH`指定路径下的`FindXXX.cmake`文件，执行该文件从而找到XXX库。其中，具体查找库并给`XXX_INCLUDE_DIRS`和`XXX_LIBRARIES`两个变量赋值的操作由`FindXXX.cmake`模块完成（先搜索当前项目里面的Module文件夹里面提供的`FindXXX.cmake`，然后再搜索系统路径`/usr/local/share/cmake-x.y/Modules/FindXXX.cmake`）
2. `Config`模式：搜索`XXX_DIR`指定路径下的`XXXConfig.cmake`文件，执行该文件从而找到XXX库。其中具体查找库并给`XXX_INCLUDE_DIRS`和`XXX_LIBRARIES`两个变量赋值的操作由`XXXConfig.cmake`模块完成。

## 5. include_directories
- 含义：**添加**头文件搜索路径
顾名思义，你可以通过多个`include_directories`函数来添加大量的搜索路径，这些函数之间不是互相覆盖的关系，而是添加到搜索列表的关系。

- 示例：
```cpp
include_directories(${OpenCV_INCLUDE_DIRS})
include_directories("D:/program-tools/PCL/PCL 1.12.1/include/pcl-1.12/")
include_directories("D:/program-tools/PCL/PCL 1.12.1/3rdParty/FLANN/include/")
```
示例讲解：
第一行：`${OpenCV_INCLUDE_DIRS}`是一个变量，里面的值显而易见是个路径。通过`include_directories`引入搜索路径。
第二、三行：引入指定头文件搜索路径。

个人讲解：
关于`${OpenCV_INCLUDE_DIRS}`这类函数怎么来，一般是`OpencvConfig.cmake`这种自带的引导文件自带的，你自己读就可以看到，一般都在注释里面注明了。![opencvconfig](/assets/images/7/2.png)
这种库提供的变量，其实一般已经被库自带的引导文件自动包含了，但是为了vs code的自动补全，你不妨再手动添加一下，反正也没什么坏处。

## 6.add_executable
含义：添加可执行文件

示例：
```cpp
add_executable(main 
${PROJECT_SOURCE_DIR}/DNN/dnn.cpp
${PROJECT_SOURCE_DIR}/main.cpp
)
```
关于什么是`${PROJECT_SOURCE_DIR}`，自己去查，到这都四千字了懒得写了。

个人讲解：
第一个参数就是可执行文件的名字，Windows下面就是`main.exe`，后面若干个参数都是说明了，这个可执行文件是由哪几个源代码编译过来的。当你的工程很简单的时候，可能一个cpp就可以编译出来一个exe可执行文件，但是一些稍微复杂一点的就不是这样的了。
需要注意的是，这个函数其实老版cmake是不支持一下添加好几个源文件的，因为其实源文件里面也有依赖关系，例如你在`A.cpp`里面定义了类A，在`main.cpp`里面使用了类A，那么如果你先编译`main.cpp`就会报错。但是我看现在的cmake好像都支持自动排序编译，很实用。不过仍然建议，按照你源文件的依赖顺序去写，对你自己也不无好处。

一个项目可以不止生成一个目标文件。


## 7.target_link_libraries
含义：对该目标链接库

示例：
```cpp
target_link_libraries(main  ${OpenCV_LIBS})
target_link_libraries(test ${Boost_LIBRARIES} ${Qhull_LIBRARIES} ${PCL_LIBRARIES} ${PTHREAD_LIB})

```

个人理解：
1. 需要注意的是，先生成目标再链接库。链接库是在编译成可执行文件后面的，不要把顺序搞反了。
2. 后面的其实是库的列表，这个`${OpenCV_LIBS}`大家有兴趣可以打印出来看看是什么。

## 8.message
含义：输出，打印
示例：
```cpp
message(root: "111 ${PCL_ROOT} 1111")
```
效果：
![message](/assets/images/7/3.png)
可以看到，`${}`括起来的自动替换为变量的值，而`root:`即使没有被套在双引号内，仍然是作为字符打印的。这个常用做调试，查看变量的值。这个是vscode终端，没有颜色。如果是ubuntu的terminal，通过改变参数，可以输出红色、黄色等颜色的错误、警告。

## 9.简易示例1
这个是我写的一个DNN库的cmakelist，这里没有把它编译成lib。可以作为参考。
```cpp
cmake_minimum_required(VERSION 2.8)

project(main)

# SET(CMAKE_BUILD_TYPE "Release")	#可以设定优化为Release

# set(CMAKE_CXX_FLAGS_RELEASE "-O3")    #可以设定优化级别为O3

set(DEFAULT_CXX_STANDARD 11)	#C++11规范

set(OpenCV_DIR  D:/opencv/opencv3.4.6/opencv/build/x64/vc15/lib/)

find_package(OpenCV REQUIRED)

include_directories(${OpenCV_INCLUDE_DIRS})

include_directories(${PROJECT_SOURCE_DIR}/DNN/)

add_executable(main 
${PROJECT_SOURCE_DIR}/main.cpp
${PROJECT_SOURCE_DIR}/DNN/dnn.cpp
)

target_link_libraries(main  ${OpenCV_LIBS})
```

## 10.简易示例2（含add_library）
这个是一个叫做wintoast的库，做出来的效果是这样的：
![wintoast](/assets/images/7/4.png)
可以以windows系统通知的形式，横幅通知在屏幕右下方，文字、图标都可以换，有点好玩。这里我又自己二次封装了一下，注意看`add_library`函数。这个文件写的很丑，不过既然是自己写着玩，我也就没管了。
```cpp
cmake_minimum_required(VERSION 3.17)
project(wintoasttest)
add_definitions(-DCOMPILEDWITHC11)
include_directories(${PROJECT_SOURCE_DIR}/lib-wintoast/)

add_library(wintoastlib STATIC 
${PROJECT_SOURCE_DIR}/lib-wintoast/wintoastlib.cpp  
${PROJECT_SOURCE_DIR}/lib-wintoast/wintoastlib.h)

add_library(my_wintoastlib STATIC 
${PROJECT_SOURCE_DIR}/sjh/my-wintoast/mywintoast.cpp  
${PROJECT_SOURCE_DIR}/sjh/my-wintoast/mywintoast.h)

add_executable(main main.cpp)
add_executable(test test.cpp)
add_executable(test2 test2.cpp)

target_link_libraries(main ${PROJECT_SOURCE_DIR}/lib-wintoast/wintoastlib.lib)
target_link_libraries(test ${PROJECT_SOURCE_DIR}/lib-wintoast/wintoastlib.lib)
target_link_libraries(test2 ${PROJECT_SOURCE_DIR}/lib-wintoast/wintoastlib.lib
                            ${PROJECT_SOURCE_DIR}/sjh/my-wintoast/my_wintoastlib.lib)
```

后续将会填充其他函数。写了六千七百字今晚不想写了。

> 我终于平庸  终于化成了
> 路过你头顶的风
> --《你头顶的风》