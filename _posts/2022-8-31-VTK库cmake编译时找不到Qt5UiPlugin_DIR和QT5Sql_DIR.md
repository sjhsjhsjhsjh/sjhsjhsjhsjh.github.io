---
layout: post
title: VTK库cmake编译时找不到Qt5UiPlugin_DIR和QT5Sql_DIR
subtitle: 一个偶然发现的问题；封面：贡橙神兽
author: 永清
categories: 环境配置
banner:
  image: /assets/images/banners/2.jpg
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: 环境配置
sidebar: []
---

> 封面：贡橙神兽

ubuntu使用cmake-gui编译VTK时，Qt5UiPlugin_DIR和QT5Sql_DIR是红的，怎么办？

答：安装libqt5x11extras5-dev和qt5-default两个包。
打开终端，输入：
```bash
sudo apt-get install libqt5x11extras5-dev
sudo apt-get install qt5-default
```