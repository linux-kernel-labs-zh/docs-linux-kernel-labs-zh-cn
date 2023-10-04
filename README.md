# Linux 内核实验室中文教程

这是 linux kernel labs ([linux-kernel-labs/linux-kernel-labs.github.io](https://linux-kernel-labs.github.io/refs/heads/master/)) 教程的中文翻译版本，翻译后的版本托管在 Github pages 上，网址为 https://hanyujie2002.github.io/linux-kernel-labs-zh-cn/

## 介绍

这是一系列关于 Linux 内核主题的课程和实验。课程侧重于理论和 Linux 内核探索。

实验侧重于设备驱动程序主题，文档风格类似“howto”。每个主题分两部分：

主题概述，包含概述、主要抽象概念、简单示例和 API的指引。
实践部分，包含几个应由学生解决的练习；为了使学生专注于当下的主题，学生会得到一个起始编码框架和深入的技巧提示来解决练习。
这些内容基于布加勒斯特理工大学自动控制与计算机学院计算机科学与工程系的“操作系统 2”<http://ocw.cs.pub.ro/courses/so2>`_ 课程。

这个仓库包含了该教程的中文翻译版本。

## 翻译进度

- [x] [课程——系统调用](https://hanyujie2002.github.io/linux-kernel-labs-zh-cn/lectures/syscalls.html)
- [ ] WIP（施工中）[介绍](https://linux-kernel-labs.github.io/refs/heads/master/lectures/intro.html)

## 项目结构介绍：

- 本仓库是最*上游*的文档仓库，大家贡献提交请在这里完成
- 提交之后会自动通过 Github action 把文档放到 [linux-zh](https://github.com/hanyujie2002/linux-zh) 这个*中游*仓库里，这个仓库负责用文档生成网站
- 网站生成之后，会通过 Github action 把生成的静态网页内容托管在 [linux-kernel-labs-zh-cn](https://github.com/hanyujie2002/linux-kernel-labs-zh-cn) 这个*下游*仓库里，之后就可以在 https://hanyujie2002.github.io/linux-kernel-labs-zh-cn/ 看到自己提交的改动了

## 贡献指南

欢迎任何人参与翻译和校对!

如果你发现任何翻译问题或者想贡献新的翻译内容,请按以下步骤操作:

1. Fork 这个仓库
3. 提交 PR 请求合并到 main 分支

**翻译版权属于原作者**

欢迎大家一起来完善这份中文教程!

## 联系方式

如果有任何问题,欢迎通过以下方式联系我:

- Github Issue
- Email: yujiehan2002@outlook.com

欢迎大家的贡献和反馈!
