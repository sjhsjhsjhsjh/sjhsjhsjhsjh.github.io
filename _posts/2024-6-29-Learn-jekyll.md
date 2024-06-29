---
layout: post
title: Jekyll 学习笔记
subtitle: 记录目前理解的部分
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
top: 1
sidebar: []
---

# 文章头结构
典型的文章头结构具有下面的信息：
- `layout`: 文章类型
- `title`: 文章大标题
- `subtitle`: 文章小标题
- `author`: 作者
- `banner`: 横幅
  - `image`: 横幅图片。注意的是，所有资源都可以通过 `/assets/images/...` 这样的形式引用
- `categories`: 归类的文件夹
- `tags`: 标签，和文件夹不同。此二者均可在网站的右上角直接访问.