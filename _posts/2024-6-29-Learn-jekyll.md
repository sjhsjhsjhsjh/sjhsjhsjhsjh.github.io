---
layout: post
title: Jekyll 学习笔记
subtitle: 随笔，记录容易忘但是有用的部分
author: 永清
categories: 环境配置
banner:
  image: /assets/images/banners/xishi.jpg
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: jekyll
sidebar: []
---

## 文章头结构
典型的文章头结构具有下面的信息：
- `layout`: 文章类型
- `title`: 文章大标题
- `subtitle`: 文章小标题
- `author`: 作者
- `banner`: 横幅
  - `image`: 横幅图片。注意的是，所有资源都可以通过 `/assets/images/...` 这样的形式引用
- `categories`: 归类的文件夹
- `tags`: 标签，和文件夹不同。此二者均可在网站的右上角直接访问.
- `published`: 布尔值，令其为false则不发布。
- `top`: 布尔值，令其为1则置顶

## 启动服务器的命令
cd到文件夹根目录下，`bundle exec jekyll serve`即可

## 注意文件格式
md文件格式必须是utf-8，不能是gbk

## 高级调试
使用浏览器的debug工具，找到希望更改的动画元素，找到其css样式位置，更改即可。