==============================================================
SO2 课程 01——课程概要以及 Linux 内核介绍
==============================================================

`查看幻灯片 <lec1-intro-slides.html>`_

.. slideconf::
   :autoslides: False
   :theme: single-level

.. slide:: SO2 课程 01——课程概要以及 Linux 内核介绍
   :inline-contents: False
   :level: 1


团队
======

.. slide:: 团队
   :inline-contents: True
   :level: 2

   * 丹尼尔·巴卢塔（丹尼尔），拉兹万·迪亚科内斯库（拉兹万，RD），克劳迪乌吉奥克（克劳迪乌），瓦伦丁·吉塔（瓦利），谢尔久·魏斯（谢尔久），奥克塔维安·普尔迪拉（塔维）

   * 亚历山德鲁·米利塔鲁（亚历克斯），特奥多拉·舍尔巴内斯库（特奥），斯特凡特奥多雷斯库（斯特凡，范内），米哈伊·波普斯库（米哈伊，米苏），康斯坦丁·拉杜卡努，丹尼尔·丁卡，劳伦丁·斯特凡

   * 祝你在新学期一切顺利！

课程定位
================

.. slide:: 课程定位
   :inline-contents: True
   :level: 2

   .. code-block:: text

      +---------------------------------------------------------+
      |        应用程序编程 (EGC, SPG, PP, SPRC, IOC 等)         |
      +---------------------------------------------------------+

           +----------------------------------+
           |       系统编程 (PC, SO, CPL)     |
           +----------------------------------+
                                                     用户空间
      ----------------------------------------------------------=-
                                                     内核空间
             +--------------------------+
             |       内核编程 (SO2)      |
             +--------------------------+

      ----------------------------------------------------------=-

           +----------------------------------+
           |     硬件 (PM, CN1, CN2, PL )     |
           +----------------------------------+


资源
=======

.. slide:: 资源
   :inline-contents: True
   :level: 2

   * Linux 内核实验: https://linux-kernel-labs-zh.xyz/
   * 邮件列表: so2@cursuri.cs.pub.ro
   * Facebook
   * vmchecker
   * Google 目录，Google 日历
   * LXR: https://elixir.bootlin.com/linux/v5.10.14/source
   * cs.curs.pub.ro——作为门户的角色
   * 积分奖励


社区
==========

.. slide:: 社区
   :inline-contents: True
   :level: 2

   * 贡献教程: https://linux-kernel-labs-zh.xyz/info/contributing.html
   * 修正、调整、澄清、有用的信息
   * 讨论列表
   * 回答同学们的问题
   * 提出与课程相关的讨论主题
   * Facebook
   * 提供建议、提案和反馈
   * 获得积分


评分
=======

.. slide:: 评分
   :inline-contents: True
   :level: 2

   * 实验室活动 2 分
   * “考试”期间评分 3 分
   * 家庭作业 5 分
   * “额外”活动
   * 家庭作业 + 额外活动得分超过 5 分
     与考试成绩成正比
   * 作业 0——0.5 分
   * 作业 1、2、3——每项 1.5 分
   * 通过条件：最终成绩 4.5，考试最低成绩 3

课程目标
====================

.. slide:: 课程目标
   :inline-contents: True
   :level: 2

   * 展示操作系统内部结构
   * 目标：通用操作系统
   * 单体内核结构和组件
   * 进程、文件系统、网络
   * 内存管理
   * 以 Linux 为例


实验和作业目标
================

.. slide:: 实验和作业目标
   :inline-contents: True
   :level: 2

   * 掌握实现设备驱动程序所需的知识

   * 通过解决练习题深入理解知识

必修课程
========

.. slide:: 必修课程
   :inline-contents: True
   :level: 2

   * 编程：C 语言
   * 数据结构：哈希表，平衡树
   * IOCLA：寄存器和基本指令操作（加法，比较，跳转）
   * 计算机网络：TLB/CAM，内存，处理器，I/O
   * PC，RL：以太网，IP，套接字
   * 操作系统：进程，文件，线程，虚拟内存

关于课程
========

.. slide:: 关于课程
   :inline-contents: True
   :level: 2

   * 12 堂课
   * 互动性
   * 参与讨论
   * 当你不理解时请提问
   * 相当“密集”，强烈建议在课前和课后阅读参考资料
   * 1 小时 20 分钟的演讲 + 20 分钟的测试和讨论


课程列表
=========

.. slide:: 课程列表
   :inline-contents: True
   :level: 2

   .. hlist::
      :columns: 2

      * 介绍
      * 系统调用
      * 进程
      * 中断
      * 同步
      * 内存寻址
      * 内存管理
      * 文件管理
      * 内核调试
      * 网络管理
      * 虚拟化
      * 内核性能分析


关于实验
================

.. slide:: 关于实验
   :inline-contents: True
   :level: 2

   * 内核模块和设备驱动程序
   * 15 分钟演示 / 80 分钟工作时间
   * 活动将被评分
   * 边做边学

关于主题
===========

.. slide:: 关于主题
   :inline-contents: True
   :level: 2

   * 必需：深入了解 API（实验）和概念（课程）
   * 公开测试
   * 测试支持（vmchecker）
   * 虽然要写的代码不多，但难度相对较大
   * 难度在于适应新环境

主题列表
==========

.. slide:: 主题列表
   :inline-contents: True
   :level: 2

   * 主题 0——内核 API
   * 基于 Kprobe 的追踪器
   * 串行端口驱动程序
   * 软件 RAID
   * SO2 传输协议


课程参考书目
=================

.. slide:: 课程参考书目
   :inline-contents: True
   :level: 2

   * 《Linux 内核开发》第三版，Robert Love，Addison Wesley，2010 年

   * 《理解 Linux 内核》第三版，Daniel P. Bovet & Marco Cesati，O'Reilly，2005 年

   * 《Linux 网络架构》，Klaus Wehrle，Frank Pahlke，Hartmut Ritter，Daniel Muller，Marc Bechler，Prentice Hall，2004 年

   * 《理解 Linux 网络内部结构》，Christian Benvenuti，O'Reilly，2005 年

实验参考书目
======================

.. slide:: 实验参考书目
   :inline-contents: True
   :level: 2

   * 《Linux 设备驱动程序》第三版，Alessandro Rubini & Jonathan Corbet，O'Reilly，2006 年

   * 《Linux 内核简明教程》，Greg Kroah-Hartman，O'Reilly，2005 年


.. include:: ../lectures/intro.rst
   :start-line: 6
