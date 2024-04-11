==========================
作业 2——驱动 UART
==========================

-  截止日期: :command:`2023 年 4 月 30 日，23:00`
-  此作业为个人完成

作业目标
=======================

*  巩固设备驱动的知识
*  阅读硬件文档并在文档中追踪所需功能
*  使用中断；在中断上下文中使用非阻塞函数
*  使用缓冲区；同步
*  带参数的内核模块

任务描述
=========

编写一个内核模块，实现串口 (`UART16550`) 的驱动程序。该设备驱动程序必须支持计算机中的两个标准串口, `COM1` 和 `COM2` (`0x3f8` 和 `0x2f8`，实际上是两个串口的整个 `8` 地址范围 `0x3f8-0x3ff` 和 `0x2f8-0x2ff`)。除了标准例程 (`open`, `read`, `write`, `close`) 外，驱动程序还必须支持使用 `ioctl` 操作 (`UART16550_IOCTL_SET_LINE`) 更改通信参数。

驱动程序必须使用中断来进行接收和发送，以减少延迟和 CPU 使用时间。 `read` 和 `write` 调用也必须是阻塞的。 :command:`不符合这些要求的作业将不予考虑。` 建议你为驱动程序中每个串行端口的读取例程使用一个缓冲区，并为写入例程使用另一个缓冲区。

当用户空间发起读取操作时，如果内核的读缓冲区为空，没有数据可读，那么阻塞读调用将会等待，直到 :command:`至少` 有一个字节被读取。同样，当用户空间发起写入操作时，如果内核的写缓冲区已满，没有空间可写，那么阻塞写调用也将会等待，直到 :command:`至少` 有一个字节被写入。

缓冲区方案
--------------

.. image:: ../img/buffers-scheme.png

各个缓冲区之间的数据传输是 `生产者-消费者 <https://zh.wikipedia.org/zh-cn/生产者消费者问题>`__ 问题。例如：

-   如果从进程写入设备，则进程是生产者，设备是消费者；如果消费者缓冲区没有剩余空间，则进程将被阻塞，直到消费者缓冲区至少有一个空闲空间为止。

-   如果从设备读取数据到进程中，则进程是消费者，设备是生产者；如果生产者缓冲区没有内容，进程将被阻塞，直到生产者的缓冲区中至少有一个元素为止。

实现细节
======================

-  该驱动程序以名为 :command:`uart16550.ko` 的内核模块形式实现。
-  该驱动程序是字符设备驱动，会根据传递给加载模块的参数，提供不同的功能操作。

   -  `major` 参数指定设备必须注册的主设备号。
   -  `option` 参数指定驱动程序的工作方式：

      -  OPTION_BOTH：还会注册 COM1 和 COM2，主设备号由 `major` 参数给出，次设备号为 0（对应 COM1）和 1（对应 COM2）；
      -  OPTION_COM1：仅注册 COM1，主设备号为 `major`，次设备号为 0。
      -  OPTION_COM2：仅注册 COM2，主设备号为 `major`，次设备号为 1。
   -  如果你不熟悉如何在 Linux 中传递参数，请参阅 `tldp <https://tldp.org/LDP/lkmpg/2.6/html/x323.html>`__。
   -  默认值为 `major=42` 和 `option=OPTION_BOTH`。
-  COM1 关联的中断号为 4 (`IRQ_COM1`)，COM2 关联的中断号为 3 (`IRQ_COM2`)
-  需要使用 `header <https://gitlab.cs.pub.ro/so2/2-uart/-/blob/master/src/uart16550.h>`__ 中的定义来进行特殊操作。
-  在实现读/写例程时，可以参考大写/小写字符设备驱动程序的 `示例 <https://ocw.cs.pub.ro/courses/so2/laboratoare/lab04?&#sincronizare_-_cozi_de_asteptare>`__；唯一的区别是你需要使用两个缓冲区，一个用于读取，另一个用于写入。
-  可以使用 `kfifo <https://lwn.net/Articles/347619/>`__ 作为缓冲区。
-  不需要使用延迟函数从端口读取/写入数据（可以在中断上下文中完成所有操作）。
-  需要在读/写例程与中断处理例程之间进行同步，以使例程成为阻塞例程；建议使用 `等待队列进行同步 <https://ocw.cs.pub.ro/courses/so2/laboratoare/lab04?&#sincronizare_-_cozi_de_asteptare>`__。
-  为了使作业正常工作，必须禁用 `默认串口驱动程序`：

   -  `cat /proc/ioports | grep serial` 可以检测到默认驱动程序是否存在于定义 COM1 和 COM2 的区域。
   - 为了停用它，必须重新编译内核，可以将串口驱动程序设置为模块，或者完全停用它（虚拟机上已经进行了这个修改）。

      -  `Device Drivers -> Character devices -> Serial driver -> 8250/16550 and compatible serial support.`

测试
=======

为了简化作业评估过程，同时也为了减少提交作业时的错误，作业评估将通过一个名为 `_checker` 的 `测试脚本 <https://github.com/linux-kernel-labs/linux/blob/master/tools/labs/templates/assignments/2-uart-checker/checker/_checker>`__ 自动进行。测试脚本假定内核模块名为 `uart16550.ko`。

快速开始
==========

你必须从 `src <https://gitlab.cs.pub.ro/so2/2-uart/-/tree/master/src>`__ 目录中的代码模板开始实现作业。模板中只有一个名为 `uart16550.h <https://gitlab.cs.pub.ro/so2/1-tracer/-/blob/master/src/uart16550.h>`__ 的头文件。你需要提供其余的实现。你可以添加任意数量的 `*.c` 源文件和额外的 `*.h` 头文件。你还应该提供一个名为 `uart16550.ko` 的 Kbuild 文件来编译内核模块。请按照 `作业仓库 <https://gitlab.cs.pub.ro/so2/2-uart>`__ 的 `README.md 文件 <https://gitlab.cs.pub.ro/so2/2-uart/-/blob/master/README.md>`__ 中的说明进行操作。


提示
----

要想增加获得最高分的机会，请阅读并遵循 Linux 内核中描述的 `编码风格规范 <https://elixir.bootlin.com/linux/v4.19.19/source/Documentation/process/coding-style.rst>`__。

此外，使用以下静态分析工具来验证代码：

- checkpatch.pl

.. code-block:: console

   $ linux/scripts/checkpatch.pl --no-tree --terse -f /path/to/your/list.c

- sparse

.. code-block:: console

   $ sudo apt-get install sparse
   $ cd linux
   $ make C=2 /path/to/your/list.c

- cppcheck

.. code-block:: console

   $ sudo apt-get install cppcheck
   $ cppcheck /path/to/your/list.c

扣分规则
----------

有关作业扣分的信息可以在“基本说明页面 <https://ocw.cs.pub.ro/courses/so2/teme/general>`__ 上找到。

在特殊情况下（作业通过了测试但不符合要求），以及如果作业未全部通过测试，成绩可能会降低得更多。

提交作业
------------------------

作业将由 `vmchecker-next <https://github.com/systems-cs-pub-ro/vmchecker-next/wiki/Student-Handbook>`__ 基础设施自动评分。提交作业将在 moodle 的 `课程页面 <https://curs.upb.ro/2022/course/view.php?id=5121>`__ 上与相关作业相关联。你可以在 `仓库 <https://gitlab.cs.pub.ro/so2/2-uart>`__ 的 `README.md 文件 <https://gitlab.cs.pub.ro/so2/2-uart/-/blob/master/README.md>`__ 中找到提交详细信息。


资源
=========

-  串口文档可以在 `tldp <https://tldp.org/HOWTO/Serial-HOWTO-19.html>`__ 上找到。
-  `寄存器表 <http://www.byterunner.com/16550.html>`__
-  `16550 数据手册 <https://pdf1.alldatasheet.com/datasheet-pdf/view/9301/NSC/PC16550D.html>`__
-  `备选文档 <https://en.wikibooks.org/wiki/Serial_Programming/8250_UART_Programming>`__

我们建议你使用 GitLab 存储作业。请按照 `README <https://gitlab.cs.pub.ro/so2/2-uart/-/blob/master/README.md>`__ 中的说明操作。


问题
=========

如有相关问题，你可以参考邮件 `列表存档 <http://cursuri.cs.pub.ro/pipermail/so2/>`__ 或在专用的 Teams 频道上提问。
