==================
体系结构层
==================

.. meta::
   :description: 这篇文档介绍了 Linux 内核的体系结构层，包括内核的引导过程、内存管理、MMU 管理、线程管理、时间管理、中断和异常管理、系统调用以及平台驱动程序等内容。
   :keywords: Linux 内核, 体系结构, 引导, 内存管理, MMU, 线程管理, 时间管理, 中断, 异常, 系统调用, 平台驱动程序

`查看幻灯片 <arch-slides.html>`_

.. slideconf::
   :autoslides: False
   :theme: single-level

课程目标：
===================

.. slide:: 简介
   :inline-contents: True
   :level: 2

   * 体系结构层概述

   * 引导过程概述


体系结构层概述
==========================

.. slide:: 体系结构层概述
   :level: 2
   :inline-contents: True

   .. ditaa::
      :height: 100%

      +---------------+  +--------------+      +---------------+
      | Application 1 |  | Application2 | ...  | Application n |
      +---------------+  +--------------+      +---------------+
              |                 |                      |
              v                 v                      v
      +--------------------------------+------------------------+
      |   Kernel core & subsystems     |    Generic Drivers     |
      +--------------------------------+------------------------+
      |             Generic Architecture Code                   |
      +---------------------------------------------------------+
      |              Architecture Specific Code                 |
      |                                                         |
      | +-----------+  +--------+  +---------+  +--------+      |
      | | Bootstrap |  | Memory |  | Threads |  | Timers |      |
      | +-----------+  +--------+  +---------+  +--------+      |
      | +------+ +----------+ +------------------+              |
      | | IRQs | | Syscalls | | Platform Drivers |              |
      | +------+ +----------+ +------------------+              |
      | +------------------+  +---------+     +---------+       |
      |	| Platform Drivers |  | machine | ... | machine |       |
      | +------------------+  +---------+     +---------+       |
      +---------------------------------------------------------+
              |                 |                      |
              v                 v                      v
      +--------------------------------------------------------+
      |                         Hardware                       |
      +--------------------------------------------------------+


引导（boot）过程
----------

.. slide:: 引导过程
   :level: 2
   :inline-contents: True

   * 第一个运行的内核代码

   * 通常在 MMU 禁用的情况下运行

   * 移动/重新定位内核代码


引导过程
----------

.. slide:: 引导过程
   :level: 2
   :inline-contents: True

   * 第一个运行的内核代码

   * 通常在 MMU 禁用的情况下运行

   * 复制引导加载程序（bootloader）参数并确定内核运行位置

   * 将内核代码移动/重新定位到最终位置

   * 初始 MMU 设置——映射内核



内存设置
------------

.. slide:: 内存设置
   :level: 2
   :inline-contents: True

   * 确定可用内存并设置引导内存分配器

   * 在页面分配器建立之前管理内存区域

   * Bootmem——使用位图跟踪空闲块

   * Memblock——取代 bootmem 并支持内存范围

     * 同时支持物理地址和虚拟地址

     * 支持 NUMA 架构


MMU 管理
--------------

.. slide:: MMU 管理
   :level: 2
   :inline-contents: True

   * 实现通用的页表操作 API：类型、访问器、标志

   * 实现 TLB 管理 API：刷新、使无效


线程管理
-----------------

.. slide:: 线程管理
   :level: 2
   :inline-contents: True

   * 定义线程类型（struct thread_info）并实现分配线程的函数（如果需要）

   * 实现 :c:func:`copy_thread` 和 :c:func:`switch_context`


时间管理
----------------

.. slide:: 时间管理
   :level: 2
   :inline-contents: True

   * 设置定时器节拍并提供时间源

   * 大部分转移到平台驱动程序

     * clock_event_device——用于调度定时器

     * clocksource——用于读取时间


中断和异常管理
-----------------------------

.. slide:: 中断和异常管理
   :level: 2
   :inline-contents: True

   * 定义中断和异常处理程序/入口点

   * 设置优先级

   * 为中断控制器提供平台驱动程序


系统调用
------------

.. slide:: 系统调用
   :level: 2
   :inline-contents: True

   * 定义系统调用入口点

   * 实现用户空间访问原语（例如，copy_to_user）


平台驱动程序
----------------

.. slide:: 平台驱动程序
   :level: 2
   :inline-contents: True

   * 平台和体系结构特定的驱动程序

   * 与平台设备枚举方法绑定（例如，设备树或 ACPI）

机器特定代码
---------------------

.. slide:: 机器特定代码
   :level: 2
   :inline-contents: True

   * 一些体系结构使用“机器”/“平台”抽象

   * 在使用大量不同种类的嵌入式系统中很常见（例如，ARM、powerPC）


引导过程概述
============================


.. slide:: 引导流程检查
   :level: 2
   :inline-contents: True


   .. asciicast:: ../res/boot.cast
