================
内核分析
================

实验目标
==============

  * 熟悉 Linux 内核分析的基础知识
  * 理解基本的分析工具
  * 学习分析方法和良好实践

概述
========

到目前为止，我们已经学习了 Linux 内核不同组件的工作方式，以及如何编写与它们交互的驱动程序，以支持设备或协议。这帮助我们了解了 Linux 内核的工作原理，虽然大多数人并不会去编写内核驱动程序。

尽管如此，所学的技能将帮助我们使应用程序更好地与整个操作系统集成。为了做到这一点，我们必须对用户空间和内核空间有全面的认知。

本课程旨在将我们在内核空间的工作与实际应用场景相结合，我们在实际场景中不编写内核空间代码，而是使用分析工具查看内核，以便调试我们编写常规低级别应用程序时遇到的问题。

本课程的另一个重点是学习一些通用的调试软件问题的方法论，并介绍一些工具，帮助我们从内核中获得应用程序运行信息。

分析工具
===============

我们主要关注的工具是 ``perf``，它支持跟踪应用程序，并从一些通用层面检查系统。我们还将使用大多数人在日常生活中会用到的调试工具，如 ``htop``, ``ps`` 以及 ``lsof`` 等。

perf
----

``perf`` 是一个使用跟踪点、kprobes 和 uprobes 来对 CPU 进行插装的工具。该工具允许我们查看在给定点调用了哪些函数。这使我们能够了解内核在哪里花费了最多的时间，打印函数的调用栈，以及记录 CPU 正在运行的内容。

``perf`` 集成了以下模块：
* 静态跟踪
* 动态跟踪
* 资源监控

perf 提供的跟踪接口可以通过 ``perf`` 命令及其子命令来使用。

.. code-block:: bash

    root@qemux86:~# ./skels/kernel_profiling/perf

     usage: perf [--version] [--help] [OPTIONS] COMMAND [ARGS]

     The most commonly used perf commands are:
       annotate        Read perf.data (created by perf record) and display annotated code
       archive         Create archive with object files with build-ids found in perf.data file
       bench           General framework for benchmark suites
       buildid-cache   Manage build-id cache.
       buildid-list    List the buildids in a perf.data file
       c2c             Shared Data C2C/HITM Analyzer.
       config          Get and set variables in a configuration file.
       data            Data file related processing
       diff            Read perf.data files and display the differential profile
       evlist          List the event names in a perf.data file
       ftrace          simple wrapper for kernel's ftrace functionality
       inject          Filter to augment the events stream with additional information
       kallsyms        Searches running kernel for symbols
       kmem            Tool to trace/measure kernel memory properties
       kvm             Tool to trace/measure kvm guest os
       list            List all symbolic event types
       lock            Analyze lock events
       mem             Profile memory accesses
       record          Run a command and record its profile into perf.data
       report          Read perf.data (created by perf record) and display the profile
       sched           Tool to trace/measure scheduler properties (latencies)
       script          Read perf.data (created by perf record) and display trace output
       stat            Run a command and gather performance counter statistics
       test            Runs sanity tests.
       timechart       Tool to visualize total system behavior during a workload
       top             System profiling tool.
       version         display the version of perf binary
       probe           Define new dynamic tracepoints

     See 'perf help COMMAND' for more information on a specific command.

在上述输出中，我们可以看到 perf 的所有子命令以及它们的功能描述，其中最重要的是：

* ``stat`` ——显示统计信息，如上下文切换和缺页错误的次数；
* ``top`` ——一个交互式界面，可以查看最频繁的函数调用及其调用者。通过这个界面，我们在进行分析时可以直接提供反馈；
* ``list`` ——列出内核中可以插装的静态跟踪点。它可以帮助我们从内核内部获取信息；
* ``probe`` ——添加动态跟踪点，插装函数调用，以便被 perf 记录；
* ``record`` ——根据用户定义的跟踪点记录函数调用和堆栈跟踪。它还可以记录特定的函数调用及其堆栈跟踪。记录默认保存在一个名为 ``perf.data`` 的文件中；
* ``report`` ——显示 perf 记录中保存的信息。

另一种使用 perf 接口的方法是通过封装 perf 的脚本，这些脚本以更高级的方式查看事件或数据，而无需了解复杂的命令。其中一个例子是 ``iosnoop.sh`` 脚本，它显示正在进行的 I/O 传输。

ps
--

``ps`` 是 Linux 上的工具，我们借此可以监视机器在给定时间运行的进程，包括内核线程。这个工具简单易用，可以一目了然地查看 CPU 上正在运行的进程以及它们的 CPU 和内存使用情况。

为了列出所有正在运行的进程，我们使用 ``ps aux``命令，就像下面这样：

.. code-block:: c

   TODO
   root@qemux86:~/skels/kernel_profiling/0-demo# cd
    root@qemux86:~# ps aux
    USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
    root         1  0.0  0.5   2004  1256 ?        Ss   12:06   0:12 init [5]
    root         2  0.0  0.0      0     0 ?        S    12:06   0:00 [kthreadd]
    [...]
    root       350  4.5  4.4  11132 10688 hvc0     T    12:07  17:21 ./io-app
    root      1358  0.0  0.0      0     0 ?        I    14:30   0:00 [kworker/u2:1-e
    root      2293  0.1  1.5   5516  3704 ?        Ss   18:18   0:00 sshd: root@pts/
    root      2295  0.0  1.3   3968  3232 pts/0    Ss+  18:19   0:00 -sh
    root      2307  0.0  0.0      0     0 ?        I    18:19   0:00 [kworker/u2:2-e
    root      2350  0.0  0.7   3032  1792 hvc0     R+   18:26   0:00 ps aux
    root      2392  2.6  0.0      0     0 ?        D    18:31   0:00 test-script

一个需要注意的信息是，第 7 列表示进程的状态, ``S`` 表示挂起, ``D`` 表示因 I/O 而挂起, ``R`` 表示正在运行。

time
----

我们可以通过 ``time`` 命令检查进程在 I/O、运行应用程序代码或运行内核空间代码上所花费的时间。这可以帮助我们找出应用程序问题的来源（是因为在内核空间中运行太多，因此在进行系统调用时有一些开销，还是问题出在用户代码中）。

.. code-block:: c

    root@qemux86:~# time dd if=/dev/urandom of=./test-file bs=1K count=10
    10+0 records in
    10+0 records out
    10240 bytes (10 kB, 10 KiB) copied, 0.00299749 s, 3.4 MB/s

    real	0m0.020s
    user	0m0.001s
    sys	0m0.015s

在上述输出中，我们测量了使用 ``dd`` 生成文件所用的时间。计时结果显示在输出的底部。工具输出的值如下：

* ``real`` ——从应用程序启动到结束经过的时间；
* ``user`` ——运行 ``dd`` 代码所花费的时间；
* ``sys`` ——代表进程运行过程中内核代码所花费的时间。

我们可以看到 ``user`` 和 ``sys`` 值的总和并不等于 ``real`` 值。这种情况可能发生在应用程序在多个核心上运行（总和可能较高），或者应用程序休眠（总和较低）的情况下。

top
---

``top`` 是一个应用程序，在大多数系统中都可以找到，它实时列出正在系统上运行的应用程序。 ``top`` 以交互方式运行，并自动刷新其输出，这一点与 ``ps`` 不同。当我们需要进行高级别的持续监控时，我们可以使用该工具。

性能分析方法论
=====================

在进行性能分析时，我们的目标是找出问题的原因。通常，这个问题是在应用程序无法按预期工作时被观察到的。当我们说应用程序没有按预期工作时，对于不同的人来说可能意味着不同的事情。例如，一个人可能抱怨应用程序变慢了，而另一个人可能说应用程序在 CPU 上运行，但没有输出任何内容。

在任何问题解决的环境中，第一步是了解我们要调试的应用程序的默认行为，并确保它目前没有在预期的参数范围内运行。

练习
=========

.. include:: ../labs/exercises-summary.hrst
.. |LAB_NAME| replace:: 内核分析

.. note::

    本任务中我们需要使用 ``perf`` 跟踪工具。当在我们的系统上本地运行时，我们需要使用包管理器安装 ``linux-tools-<version>-generic`` 软件包才能运行它。由于我们的虚拟机没有访问包管理器的权限，我们需要从 `这个 <http://swarm.cs.pub.ro/~sweisz/perf>`_ 链接下载 ``perf`` 二进制文件。将应用程序下载到 ``skels/kernel_profiling`` 目录，并授予执行权限。

.. warning::

    运行 ``perf`` 时，请确保运行的是下载的版本，而不是``PATH``变量中的版本。

.. note::

    在完成本课程的练习时，我们将需要同时运行多个命令。为此，我们需要使用 SSH 连接到虚拟机。我们建议使用 ``core-image-sato-sdk-qemu`` 镜像，因为它具备我们所需的工具。要使用 ``core-image-sato-sdk-qemu`` 文件系统运行虚拟机，请取消 ``qemu/Makefile`` 文件中的第 16 行的注释。

.. note::

    如果你希望运行我们在仓库中包含的基于 ``perf-tools`` 的脚本，例如 ``iosnoop.sh``，你需要授予其执行权限，以便将其复制到虚拟机的文件系统中。

.. note::

    你的意见对我们有很大帮助，可以帮助我们改进 SO2 课程的组成部分和开展方式。请填写 `curs.upb.ro 平台上的反馈表格 <https://curs.upb.ro/2022/mod/feedbackadm/view.php?id=15292>`_。

    该表格是匿名的，活动时间为 2023 年 5 月 22 日至 6 月 2 日。在所有成绩评定完成后，SO2 团队将能够查看结果。

    我们邀请你评估 SO2 团队的活动，并说明其优点和缺点，以及改进课程的建议。你的反馈对我们来说非常重要，我们借此可以提高未来几年的课程质量。

    我们特别关注以下问题：

      * 有哪些内容你不喜欢以及你认为做得不好的地方有哪些？
      * 为什么你不喜欢它，为什么你认为它做得不好？
      * 我们应该做些什么来改善情况？

0. 演示：I/O 问题的性能分析
===============================

在处理 I/O 时，我们必须记住，与内存相比，I/O 是操作系统中速度最慢的系统之一，与之相比内存和进程调度（处理运行在 CPU 上的内容）的速度要快上一个数量级。

因此，我们必须谨慎考虑 I/O 操作，因为如果系统忙于处理 I/O 请求的话，应用程序可能就无法正常运行。另一个可能遇到的问题是，如果应用程序需要等待 I/O 操作完成的话，I/O 可能因其速度慢而影响应用程序的响应性。

让我们来看一个应用程序并调试其中的问题。

我们将运行 ``0-demo`` 目录的 ``io-app`` 应用程序。

为了检查运行在 CPU 上的内容，并查看进程的堆栈，我们可以使用 ``perf record``子命令，就像下面这样：

.. code-block:: bash

    root@qemux86:~# ./perf record -a -g
    Couldn't synthesize bpf events.
    ^C[ perf record: Woken up 7 times to write data ]
    [ perf record: Captured and wrote 1.724 MB perf.data (8376 samples) ]


perf 将无限期记录值，但我们可以使用 ``Ctrl+c`` 快捷键关闭它。我们使用了 ``-a`` 选项来探测所有 CPU，并使用 ``-g`` 选项记录整个调用堆栈。

为了可视化记录的信息，我们需要使用 ``perf report`` 命令，它将打开一个页面，显示在 CPU 上的最频繁的函数调用及其调用堆栈。

.. code-block:: bash

    root@qemux86:~# ./perf report --header -F overhead,comm,parent
    # Total Lost Samples: 0
    #
    # Samples: 8K of event 'cpu-clock:pppH'
    # Event count (approx.): 2094000000
    #
    # Overhead  Command          Parent symbol
    # ........  ...............  .............
    #
        58.63%  io-app           [other]
                |
                 --58.62%--__libc_start_main
                           main
                           __kernel_vsyscall
                           |
                            --58.61%--__irqentry_text_end
                                      do_SYSENTER_32
                                      do_fast_syscall_32
                                      __noinstr_text_start
                                      __ia32_sys_write
                                      ksys_write
                                      vfs_write
                                      |
                                       --58.60%--ext4_file_write_iter
                                                 ext4_buffered_write_iter
    [...]

我们使用 ``--header`` 选项打印表头，并使用 ``-F overhead,comm,parent`` 选项打印调用堆栈、命令和调用者的时间百分比。

我们可以看到 ``io-app`` 命令在文件系统中进行了一些写操作，这在很大程度上增加了系统的负载。

有了这些信息，我们知道了应用程序正在进行许多 I/O 调用。为了查看这些请求的大小，我们可以使用 ``iosnoop.sh`` 脚本。

.. code-block:: bash

    root@qemux86:~/skels/kernel_profiling# ./iosnoop.sh 1
    Tracing block I/O. Ctrl-C to end.
    COMM         PID    TYPE DEV      BLOCK        BYTES     LATms
    io-app       889    WS   254,0    4800512      1310720     2.10
    io-app       889    WS   254,0    4803072      1310720     2.04
    io-app       889    WS   254,0    4805632      1310720     2.03
    io-app       889    WS   254,0    4808192      1310720     2.43
    io-app       889    WS   254,0    4810752      1310720     3.48
    io-app       889    WS   254,0    4813312      1310720     3.46
    io-app       889    WS   254,0    4815872      524288     1.03
    io-app       889    WS   254,0    5029888      1310720     5.82
    io-app       889    WS   254,0    5032448      786432     5.80
    jbd2/vda-43  43     WS   254,0    2702392      8192       0.22
    kworker/0:1H 34     WS   254,0    2702408      4096       0.40
    io-app       889    WS   254,0    4800512      1310720     2.60
    io-app       889    WS   254,0    4803072      1310720     2.58
    [...]

根据这个输出，我们可以看到 ``io-app`` 在进行循环读取，因为第一个块 ``4800512`` 重复出现，并且它进行了大块读取，因为它每次读取一兆字节。这个不断循环的过程增加了我们所经历的系统负载。

1. 调查降低响应性的问题
---------------------------------------

``io.ko`` 模块位于 ``kernel_profiling/1-io`` 目录中，在插入后会降低系统的响应性。我们注意到在输入命令时命令行会出现卡顿，但是在运行 ``top`` 命令时，我们发现系统的负载并不高，并且没有任何占用资源的进程。

找出 ``io.ko`` 模块在做什么，以及为什么它会导致卡顿。

.. hint::

    跟踪所有被调用的函数，并检查 CPU 在哪里花费了大部分时间。为此，你可以运行 ``perf record`` 和 ``perf report`` 并查看输出，或者使用 ``perf top`` 命令。

2. 启动新线程
------------------------

我们希望以并行方式将同一个函数循环运行 100 次。我们在 ``kernel_profiling/2-scheduling`` 目录下的 ``scheduling`` 可执行文件中实现了两种解决方案。

当执行 ``scheduling`` 可执行文件时，它会并行打印 100 个运行实例的消息。我们可以通过将第一个参数设为 ``0`` 或 ``1`` 来调整执行方式。

找出哪种解决方案更好，以及原因是什么。

3. 调整 ``cp``
----------------

我们的目标是编写一个集成在 Linux 中的 ``cp`` 工具的副本，该工具已经由 ``kernel_profiling/3-memory`` 目录中的 ``memory`` 可执行文件实现。它对复制操作支持两种实现：

* 使用 ``read()`` 系统调用将源文件的内容读入内存缓冲区，并使用 ``write()`` 系统调用将该缓冲区写入目标文件；
* 使用 ``mmap`` 系统调用将源文件和目标文件映射到内存中，然后在内存中将源文件的内容复制到目标文件中。

另一个我们可以使用的可调参数是复制操作的块大小，无论是通过读取/写入还是在内存中进行都支持这个参数。

1) 调查这两种复制机制中哪一种更快。在这一步中，你需要使用 1024 字节的块大小。

2) 一旦找到哪种复制机制更快，更改块大小参数，看看哪个值能给出最好的复制效果。为什么？

4. I/O 延迟
--------------

我们编写了一个模块来读取磁盘的内容。插入位于 ``4-bio`` 目录中的 ``bio.ko`` 模块后，系统负载急剧上升（可以在 ``top`` 命令中看到），但我们发现系统仍然响应良好。

调查是什么导致系统负载增加。是 I/O 问题还是调度问题？

.. hint::

    尝试使用 ``perf`` 跟踪I/O操作，或使用 ``iosnoop.sh`` 脚本来检查在特定时间点发生了哪些 I/O 操作。

5. 错误的 ELF 文件
----------

.. note::

    这是一个额外的练习，已在原生 Linux 系统上进行了测试。它可以在 QEMU 虚拟机中运行，但在我们的测试中行为很奇怪。我们建议你使用原生（也可以是 VirtualBox 或者 VMware）的 Linux 系统。

我们成功构建了一个 ELF文件（作为 `Unikraft <https://github.com/unikraft/unikraft>`__ 构建的一部分），在静态分析中该文件是有效的，但无法执行。该文件名为 ``bad_elf``，位于 ``5-bad-elf/`` 文件夹中。

运行它会触发“段错误（segmentation fault）”消息。如果使用 ``strace`` 运行它则会显示 ``execve()`` 出现错误。

.. code::

    ... skels/kernel_profiling/5-bad-elf$ ./bad_elf
    Segmentation fault

    ... skels/kernel_profiling/5-bad-elf$ strace ./bad_elf
    execve("./bad_elf", ["./bad_elf"], 0x7ffc3349ba50 /* 70 vars \*/) = -1 EINVAL (Invalid argument)
    --- SIGSEGV {si_signo=SIGSEGV, si_code=SI_KERNEL, si_addr=NULL} ---
    +++ killed by SIGSEGV +++
    Segmentation fault (core dumped)

ELF 文件本身是有效的：

.. code::

    ... skels/kernel_profiling/5-bad-elf$ readelf -a bad_elf

问题可以在内核中检测到。

使用 ``perf`` 或更好的 `ftrace <https://jvns.ca/blog/2017/03/19/getting-started-with-ftrace/>`__ 来检查程序调用的内核函数。确定发送 ``SIGSEGV`` 信号的函数调用。找出问题的原因。在 `elf(5)手册页 <https://linux.die.net/man/5/elf>`__ 中找到该原因。
