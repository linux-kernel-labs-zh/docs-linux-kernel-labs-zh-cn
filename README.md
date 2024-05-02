[![Stars](https://img.shields.io/github/stars/linux-kernel-labs-zh/docs-linux-kernel-labs-zh-cn.svg)](https://github.com/linux-kernel-labs-zh/docs-linux-kernel-labs-zh-cn/stargazers)
[![Forks](https://img.shields.io/github/forks/linux-kernel-labs-zh/docs-linux-kernel-labs-zh-cn.svg)](https://github.com/linux-kernel-labs-zh/docs-linux-kernel-labs-zh-cn/network/members)
[![Watchers](https://img.shields.io/github/watchers/linux-kernel-labs-zh/docs-linux-kernel-labs-zh-cn.svg)](https://github.com/linux-kernel-labs-zh/docs-linux-kernel-labs-zh-cn/watchers)

# Linux 内核实验中文教程

本文档是 linux kernel labs ([linux-kernel-labs/linux-kernel-labs.github.io](https://linux-kernel-labs.github.io/refs/heads/master/)) 教程的中文翻译版本，翻译后的版本托管在 Vercel 上，网址为 [https://linux-kernel-labs-zh.xyz](https://linux-kernel-labs-zh.xyz)。

## 介绍

本内容是布加勒斯特理工大学的 Linux 内核教学课程。该课程通过动手实践设备驱动编写，使学习者深入理解 Linux 内核，适合所有对 Linux 内核原理感兴趣的人阅读。

本文档主要分为两个模块，一个是“课程”，还有一个是“实验”。“课程”部分写得不甚详细，更适合有经验的教师上课时使用。而“实验”部分则是本文档最有价值的部分，写的非常的详细而且由浅入深，Linux 内核零基础的同学也可以来学习。注意“实验”模块学习之前，并不需要学习“课程”模块。

## 实验环境配置

实验在基于 QEMU 的虚拟机中进行。首先在主机上开发和构建内核代码，然后将其部署至虚拟机执行。配置实验环境有两种方法：

1. **使用 Docker（推荐方式）**：请参照 [so2-labs 仓库](https://github.com/linux-kernel-labs-zh/so2-labs)中的指南进行配置。
2. **手动配置**：克隆或下载 [Linux-zh 仓库](https://github.com/linux-kernel-labs-zh/linux-zh)至你的 Linux 系统（推荐使用 Debian/Ubuntu 发行版，支持物理机、虚拟机或 WSL），并根据[虚拟机配置指南](https://linux-kernel-labs-zh.xyz/info/vm.html#section-2)配置环境。

## 项目结构介绍：

- 本仓库是最*上游*的文档仓库，提交在这里完成
- 提交之后会自动通过 Github action 把文档放到 [linux-zh](https://github.com/linux-kernel-labs-zh/linux-zh) 这个*中游*仓库里，这个仓库负责用文档生成网站
- 网站生成之后，会通过 Github action 把生成的静态网页内容托管在 [linux-kernel-labs-zh.github.io](https://github.com/linux-kernel-labs-zh/linux-kernel-labs-zh.github.io) 这个*下游*仓库里，之后同步到 Vercel 就可以在 https://linux-kernel-labs-zh.xyz 看到提交的改动了。

## 版权声明

**翻译版权属于原作者**

## 联系方式

如果有任何问题,欢迎通过以下方式联系我:

- Github Issue
- Email: linux-kernel-labs-zh@hotmail.com

欢迎大家的反馈！
