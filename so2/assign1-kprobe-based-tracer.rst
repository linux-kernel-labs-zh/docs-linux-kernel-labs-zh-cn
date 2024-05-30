==================================
作业 1——基于 Kprobe 的跟踪器
==================================

.. meta::
   :description: 介绍操作系统 2 课程的第一个实验任务，内容包括用 kprobe 机制实现一个内核层面的监视器, 监控内核常见的函数调用情况。
   :keywords: 内核编程, Linux, kprobe, kretprobe, 进程监控, 内存监控, 锁监控, procfs

-  截止日期： :command:`周一，2024 年 4 月 8 日，23:59`

作业目标
=======================

*  掌握与 Linux 内核中函数仪器化（instrumentation）相关的知识 (``kretprobes`` 机制)
*  掌握 Linux 内核中的 ``/proc`` 文件系统
*  熟悉 Linux 内核特定的数据结构 (``哈希表`` 和 ``链表``)

题目描述
=========

构建一个内核操作监视器（surveillant）。

通过这个监视器，我们的目标是拦截：

* ``kmalloc`` 和 ``kfree`` 调用
* ``schedule`` 调用
* ``up`` 和 ``down_interruptible`` 调用
* ``mutex_lock`` 和 ``mutex_unlock`` 调用

监视器将在进程级别上保存上述每个函数的调用次数。对于 ``kmalloc`` 和 ``kfree`` 调用，将显示已分配和释放的内存总量。

监视器需要以内核模块的形式实现，名称为 ``tracer.ko``。

实现细节
----------------------

拦截将通过记录每个上述函数的样本 (``kretprobe``) 来完成。监视器将保留一个包含被监视进程的列表/哈希表，并为这些进程记录上述信息。

为了控制包含被监视进程的列表/哈希表，我们需要使用名为 ``/dev/tracer`` 的字符设备，主设备号为 `10`，次设备号为 `42`。它将暴露一个带有两个参数的 ``ioctl`` 接口：

* 第一个参数是对监视子系统的请求：

    * ``TRACER_ADD_PROCESS``
    * ``TRACER_REMOVE_PROCESS``

* 第二个参数是要执行监视请求的进程的 PID

为了使用主设备号 `10` 创建字符设备，你需要在内核中使用 `miscdevice <https://elixir.bootlin.com/linux/latest/source/include/linux/miscdevice.h>`__ 接口。相关宏的定义可以在 `tracer.h 头文件 <https://gitlab.cs.pub.ro/so2/1-tracer/-/blob/master/src/tracer.h>`__ 中找到。

由于 ``kmalloc`` 函数是内联的，无法对分配的内存量进行仪器化，因此我们将检查 ``__kmalloc`` 函数：

* 我们需要使用 ``kretprobe``，它将保留已分配的内存量和已分配内存区域的地址。
* ``kretprobe`` 结构中的 ``.entry_handler`` 和 ``.handler`` 字段将用于保留已分配的内存量和分配的内存起始地址。

.. code-block:: C

    static struct kretprobe kmalloc_probe = {
       .entry_handler = kmalloc_probe_entry_handler, /* 进入处理程序 */
       .handler = kmalloc_probe_handler, /* 返回 probe 处理程序 */
       .maxactive = 32,
    };

由于 ``kfree`` 函数只接收要释放的内存区域的地址，为了确定释放的总内存量，我们需要根据该区域的地址确定其大小。这是可行的，因为系统在检查 ``__kmalloc`` 函数时进行了地址大小的关联。

对于其余的仪器化函数，使用 ``kretprobe`` 就足够了。

.. code-block:: C

    static struct kretprobe up_probe = {
       .entry_handler = up_probe_handler,
       .maxactive = 32,
    };

虚拟机内核已启用 ``CONFIG_DEBUG_LOCK_ALLOC`` 选项，其中 ``mutex_lock`` 符号是一个将展开为 ``mutex_lock_nested`` 的宏。因此，为了获取有关 ``mutex_lock`` 函数的信息，你需要对 ``mutex_lock_nested`` 函数进行仪器化。

添加到列表/哈希表的进程在执行结束时将从列表/哈希表中移除。此外，根据 ``TRACER_REMOVE_PROCESS`` 操作，系统将从调度列表/哈希表中移除一个进程。

监视器保留的信息将通过 procfs 文件系统显示在 ``/proc/tracer`` 文件中。每个受监视的进程，在 ``/proc/tracer`` 文件中都会有一个条目，其第一个字段是进程的 PID。该条目是只读的，对其进行读操作将显示保留的结果。显示条目内容的示例：

.. code-block:: console

    $cat /proc/tracer
    PID   kmalloc kfree kmalloc_mem kfree_mem  sched   up     down  lock   unlock
    42    12      12    2048        2048        124    2      2     9      9
    1099  0       0     0           0           1984   0      0     0      0
    1244  0       0     0           0           1221   100   1023   1023   1002
    1337  123     99    125952      101376      193821 992   81921  7421   6392

测试
=======

为了简化作业评估过程，同时也为了减少提交作业时的错误，作业评估将通过一个名为 `_checker` 的 `测试脚本 <https://github.com/linux-kernel-labs/linux/blob/master/tools/labs/templates/assignments/1-tracer/checker/_checker>`__ 自动进行。测试脚本假定内核模块名为 `tracer.ko`。

快速开始
==========

你必须从 `src <https://gitlab.cs.pub.ro/so2/1-tracer/-/tree/master/src>`__ 目录中的代码模板开始实现作业。模板中只有一个名为 `tracer.h <https://gitlab.cs.pub.ro/so2/1-tracer/-/blob/master/src/tracer.h>`__ 的头文件。你需要提供其余的实现。你可以添加任意数量的 `*.c` 源文件和额外的 `*.h` 头文件。你还应该提供一个名为 `tracer.ko` 的 Kbuild 文件来编译内核模块。请按照 `作业仓库 <https://gitlab.cs.pub.ro/so2/1-tracer>`__ 的 `README.md 文件 <https://gitlab.cs.pub.ro/so2/1-tracer/-/blob/master/README.md>`__ 中的说明进行操作。


提示
----

要想增加获得最高分的机会，请阅读并遵循 Linux 内核中描述的 `编码风格规范 <https://elixir.bootlin.com/linux/v4.19.19/source/Documentation/process/coding-style.rst>`__。

此外，使用以下静态分析工具来验证代码：

- checkpatch.pl

.. code-block:: console

   $ linux/scripts/checkpatch.pl --no-tree --terse -f /path/to/your/tracer.c

- sparse

.. code-block:: console

   $ sudo apt-get install sparse
   $ cd linux
   $ make C=2 /path/to/your/tracer.c

- cppcheck

.. code-block:: console

   $ sudo apt-get install cppcheck
   $ cppcheck /path/to/your/tracer.c

扣分项
---------

关于作业扣分的信息可以在 `基本说明文件 <https://ocw.cs.pub.ro/courses/so2/teme/general>`__ 中找到。此外，以下因素还将被考虑：

* *-2*：未正确释放资源 (``kretprobes``, ``/proc`` 中的条目)
* *-2*：多个执行实例使用的数据的数据同步问题（例如，列表/哈希表）

在特殊情况下（作业通过了测试但不符合要求），以及如果作业未全部通过测试，成绩可能会降低得更多。

提交作业
------------------------

作业将由 `vmchecker-next <https://github.com/systems-cs-pub-ro/vmchecker-next/wiki/Student-Handbook>`__ 基础设施自动评分。提交作业将在 moodle 的 `课程页面 <https://curs.upb.ro/2022/course/view.php?id=5121>`__ 上与相关作业相关联。你可以在 `仓库 <https://gitlab.cs.pub.ro/so2/1-tracer>`__ 的 `README.md 文件 <https://gitlab.cs.pub.ro/so2/1-tracer/-/blob/master/README.md>`__ 中找到提交详细信息。

资源
=========

* `Documentation/kprobes.txt <https://www.kernel.org/doc/Documentation/kprobes.txt>`__ ——Linux 内核源代码中关于 ``kprobes`` 子系统的描述。
* `samples/kprobes/ <https://elixir.bootlin.com/linux/latest/source/samples/kprobes>`__ ——Linux 内核源代码中使用 ``kprobes`` 的一些示例。

我们建议你使用 gitlab 来存储你的作业。请按照 `README <https://gitlab.cs.pub.ro/so2/1-tracer/-/blob/master/README.md>`__ 中的说明操作。

问题
=========

如有相关问题，你可以参考邮件 `列表存档 <http://cursuri.cs.pub.ro/pipermail/so2/>`__ 或在专用的 Teams 频道上提问。
