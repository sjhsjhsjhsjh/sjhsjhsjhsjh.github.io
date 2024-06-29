---
layout: post
title: Verilated model didn‘t converge报错原因及解决
subtitle: “一生一芯”项目笔记
author: 永清
categories: verilog
banner:
  image: /assets/images/banners/baiheliang.jpg
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: verilog
---

>还留着你的回忆，分不清南北东西
>我真的不想从此迷失在这幻境 
>----《迷失幻境》

想必你急于知道原因，不想听笔者解决问题的过程，所以我先放结论，再说我发现的过程。

## 1.错误原因
一句话概括，根本原因是：
**有一个量在极快的变化**

verilog官网原图：
![在这里插入图片描述](/assets/images/5/1.png)

verilog官网上的原文是：

This is because the signals keep toggling even with out time passing. Thus to prevent an infinite loop, the Verilated executable gives the DIDNOTCONVERGE error.

**翻译：这是因为*信号在不停的变换即使时间没有流逝*。所以，为了防止无限循环，verilog可执行文件给出了DIDNOTCONVERGE 错误。**

To debug this, first review any UNOPT or UNOPTFLAT warnings that were ignored. Though typically it is safe to ignore UNOPTFLAT (at a performance cost), at the time of issuing a UNOPTFLAT Verilator did not know if the logic would eventually converge and assumed it would.

**翻译:若要对此进行调试，请首先查看忽略的任何UNOPT或UNOPTFLAT警告。尽管通常忽略UNOPTFLAT（以性能为代价）是安全的，但在发布UNOPTFLAT时，验证器不知道逻辑是否最终会收敛，并假设它会收敛。**

## 2. 解决方案
官网方案：
![在这里插入图片描述](/assets/images/5/2.png)
run Verilator with --prof-cfuncs -CFLAGS -DVL_DEBUG. Rerun the test. Now just before the convergence error you should see additional output similar to this

翻译：使用--prof-cfuncs-CFLAGS-DVL_DEBUG运行Verilator。重新运行测试。现在，就在收敛错误之前，您应该会看到类似的输出

分析：例如你编译.v的命令是
```bash
verilator -Wno-fatal top.v main.cpp --top-module top --cc --trace --exe
```
则添上 --prof-cfuncs -CFLAGS -DVL_DEBUG即可，即
```bash
verilator -Wno-fatal top.v main.cpp --top-module top --cc --trace --exe --prof-cfuncs -CFLAGS -DVL_DEBUG
```
再次运行，会发现有类似于这样的出错：
![错误](/assets/images/5/3.png)
注意看，其指明了是top模块的变量b出的问题，并且是在第16行
根据这个修改就可以了。

## 3.笔者的解决过程
### 3.1 笔者的问题背景
我需要写一个ALU，ALU当然需要有减法，而且我的ALU输入是补码。
众所周知，补码的减法是
```cpp
A - B = A + ((~B) + 1)
```
所以我verilog是这么写的
```cpp
b = ~b + 1;
ans = a + b;
```
这样写看起来没什么问题（实际上就是没有问题）
问题出在我的仿真上

### 3.2 问题发现与解决
我测试代码的方法是，给a、b赋初值，所以每次测试只需要运行一次，也没什么敏感变量，所以always的括号里面是空的，代码如下
```cpp
reg [2:0] command = 3'b001;
reg [3:0] a = 3'b1001;
reg [3:0] b = 3'b0001;

    always @() begin
        case (command)
            3'b000:begin
                 {temp_overflow_flag, temp_ans} = a + b;
            end
            3'b001:begin
                b = ~b + 1;
                 {temp_overflow_flag, temp_ans} = a + b;
            end
·············下略
```
这样会导致always无间隙的无限执行，所以**b被无限执行取反操作**，导致verilog以为出问题了，报了这个错

然后我加了个clk，敏感变量设置成posedge clk就可以了

因为笔者是新手，并不知道always括号里面空着会这样，所以犯了这样愚蠢的错误，尤其是不读入时钟，确实很蠢。

## 4.后记
出现这个问题的**直接原因**就是**变量的值极其快速的改变**，
但是根本原因必然各不相同，我是因为时钟没设置，你的必须对症下药，这个变量是在哪里改变的，为什么会被快速改变，就可以解决问题了。