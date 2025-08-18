---
weight: 4
title: "个人博客搭建之搭建篇-(Hugo + Github Pages + Github Action自动发布)"
date: 2025-08-18T15:35:59+08:00
lastmod: 2025-08-18T15:35:59+08:00
draft: false
author: "Stef"
authorLink: ""
description: "这篇文章利用Hugo + Github Pages搭建个人博客，并用Github Action实现自动发布."
images: []
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["Hugo", "Github Pages", "Github Action"]
categories: ["个人博客搭建"]

lightgallery: true
---

> 利用Hugo + Github Pages搭建个人博客，并用Github Action实现自动发布。

# 0. 使用到的工具
* Hugo: 静态页面生成工具。
* Github Pages: 静态网站托管平台。
* Github Action: 自动部署工作流，将`Hugo`生成的静态页面发布到`Github Pages`上。

# 1. 常见做法
* 做法1: `发布仓库（Pages站点仓库）` 和 `内容仓库（markdown源码）` 分为两个仓库，在`内容仓库（markdown源码）`提交文章后，自动触发`内容仓库`预先配置的Actions,执行对应的action构建打包并发布到`Github Pages站点仓库`,随后访问`https://<username>.github.io/`即可看到博客。
* 做法2(本文做法):	一个仓库，多个分支模式，`main`分支存放`内容源码(如markdown文章)`， `gh-pages`分支存放通过`Github Action`生成的静态页面。

# 