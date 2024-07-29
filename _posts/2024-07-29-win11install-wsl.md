---
layout: post
title: Windows11安装wsl与基本环境配置
subtitle: 相关命令备忘
author: 永清
categories: verilog
banner:
  image: /assets/images/banners/7.jpg
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: verilog
---

> 笔者默认看到此文的人都熟悉Ubuntu的常用命令和基本知识，如换源、ssh等知识，文中不再赘述。

## wsl子系统的安装和卸载
查看当前安装的子系统
```dark
wsl --list
```
注销（卸载）当前安装的Linux的Windows子系统
```dark
wsl --unregister Ubuntu
```
我还发现，命令行卸载完了需要进入设置->应用列表里面搜索ubuntu，卸载所有发行版，不然可能会出问题。

## 移动wsl到其它盘
这一步并不是必要的，甚至笔者是不推荐的。如果你有强迫症，晚上睡觉一想到有个wsl正在占用你宝贵的C盘空间你就睡不着觉，那么不妨按照这个做一下。本质是打包、移动、导入三步，但是有人提出，这样会损坏依赖目录和配置文件。所以，还是不动为好，不过动了之后其实也没啥问题，至少我没用出来，亦或是，也没人能证明某个问题是移动了wsl导致的。
```dark
// 导出wsl
wsl --export Ubuntu-20.04 D:\ubuntu20.04.tar
// 注销原本的
wsl --unregister Ubuntu-20.04
// 导入
wsl --import Ubuntu-20.04 D:\wsl\ubuntu D:\ubuntu20.04.tar --version 2
// 重新设置密码，这里就是我说的可能会损坏配置文件，不然怎么会需要重新配置这个呢
ubuntu2004 config --default-user Username
// 删除原本的包，手动删也行
del D:\ubuntu20.04.tar
```

## WslRegisterDistribution failed with error: 0x800706be
升级wsl。参考：[WSL issues](https://github.com/microsoft/WSL/issues/5768)
```
wsl.exe --update
```

## 安装wsl
我安装的是Ubuntu-24.04
```
wsl.exe --list --online
wsl.exe --install Ubuntu-24.04
```