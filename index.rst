=============
Linux 内核教学
=============

这是一系列关于 Linux 内核主题的课程和实验。课程侧重于理论和 Linux 内核探索。

实验侧重于设备驱动程序主题，文档风格类似“howto”。每个主题分两部分：

* 主题概述，包含概述、主要抽象概念、简单示例和 API的指引。

* 实践部分，包含几个应由学生解决的练习；为了使学生专注于当下的主题，学生会得到一个起始编码框架和深入的技巧提示来解决练习。

这些内容基于布加勒斯特理工大学自动控制与计算机学院计算机科学与工程系的“操作系统 2”<http://ocw.cs.pub.ro/courses/so2>`_ 课程。

你可以在 https://github.com/linux-kernel-labs 获取最新版本。

在你的主机上安装 docker-compose 后，可以从源代码构建文档:

.. code-block:: bash

   cd tools/labs && make docker-docs

然后用你的浏览器中打开 **Documentation/output/labs/index.html**。

或者，你可以直接在主机上构建(参见 tools/labs/docs/Dockerfile 中的依赖项)：

.. code-block:: bash

   cd tools/labs && make docs

.. toctree::

   so2/index.rst

.. toctree::
   :caption: 课程

   lectures/intro.rst
   lectures/syscalls.rst
   lectures/processes.rst
   lectures/interrupts.rst
   lectures/smp.rst
   lectures/address-space.rst
   lectures/memory-management.rst
   lectures/fs.rst
   lectures/debugging.rst
   lectures/networking.rst
   lectures/arch.rst
   lectures/virt.rst

.. toctree::
   :caption: 实验

   labs/infrastructure.rst
   labs/introduction.rst
   labs/kernel_modules.rst
   labs/kernel_api.rst
   labs/device_drivers.rst
   labs/interrupts.rst
   labs/deferred_work.rst
   labs/block_device_drivers.rst
   labs/filesystems_part1.rst
   labs/filesystems_part2.rst
   labs/networking.rst
   labs/arm_kernel_development.rst
   labs/memory_mapping.rst
   labs/device_model.rst
   labs/kernel_profiling.rst

.. toctree::
   :caption: 有用信息

   info/vm.rst
   info/extra-vm.rst
   info/contributing.rst
