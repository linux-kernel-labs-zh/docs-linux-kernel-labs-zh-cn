========
内核模块
========

.. meta::
   :description: 创建简单的模块，描述内核模块编译的过程，展示如何在内核中使用模块，简单的内核调试方法

实验目标
========

* 创建简单的模块
* 描述内核模块编译的过程
* 展示如何在内核中使用模块
* 简单的内核调试方法

..
  _[SECTION-OVERVIEW-BEGIN]

内核模块概述
===========

虽然单体内核比微内核更快，但缺乏模块化和可扩展性。在现代单体内核中，这个问题已经通过使用内核模块来解决。内核模块（或可加载内核模式）是一个包含代码的目标文件（object file），它可以在运行时扩展内核的功能（根据需要加载）；当不需要内核模块时，可以卸载它。大多数设备驱动程序以内核模块的形式使用。

为了开发 Linux 设备驱动程序，建议下载内核源代码，配置和编译它们，然后将编译后的版本安装在测试/开发工具机上。

..
  _[SECTION-OVERVIEW-END]

..
  _[SECTION-MODULE-EXAMPLE-BEGIN]

内核模块示例
============

以下是一个非常简单的内核模块示例。当加载到内核中时，它会生成消息 :code:`"Hi"`。当卸载内核模块时，将生成消息 :code:`"Bye"`。

.. code-block:: c

    #include <linux/kernel.h>
    #include <linux/init.h>
    #include <linux/module.h>

    MODULE_DESCRIPTION("My kernel module");
    MODULE_AUTHOR("Me");
    MODULE_LICENSE("GPL");

    static int dummy_init(void)
    {
            pr_debug("Hi\n");
            return 0;
    }

    static void dummy_exit(void)
    {
            pr_debug("Bye\n");
    }

    module_init(dummy_init);
    module_exit(dummy_exit);


生成的消息不会显示在控制台上，而是保存在专门预留的内存区域中，由日志守护程序（syslog）负责将其提取出来。要显示内核消息，你可以使用 `dmesg` 命令或检查日志文件。

.. code-block:: bash

   # cat /var/log/syslog | tail -2
   Feb 20 13:57:38 asgard kernel: Hi
   Feb 20 13:57:43 asgard kernel: Bye

   # dmesg | tail -2
   Hi
   Bye

..
  _[SECTION-MODULE-EXAMPLE-END]

..
  _[SECTION-COMPILE-MODULES-BEGIN]

编译内核模块
============

编译内核模块与编译用户程序不同。首先，需要使用另外的头文件。此外，模块不应链接到库。最重要的是，模块必须使用与加载模块的内核相同的选项进行编译。出于这些原因，有一种标准的编译方法（kbuild）。该方法需要使用两个文件： :file:`Makefile` 文件和 :file:`Kbuild` 文件。

以下是 :file:`Makefile` 文件的示例：

.. code-block:: bash

   KDIR = /lib/modules/`uname -r`/build

   kbuild:
           make -C $(KDIR) M=`pwd`

   clean:
           make -C $(KDIR) M=`pwd` clean

以下是用于编译模块的 :file:`Kbuild` 文件示例：

.. code-block:: bash

   EXTRA_CFLAGS = -Wall -g

   obj-m        = modul.o


正如你所见，在示例中调用 :command:`make` 命令对 :file:`Makefile` 文件进行编译将导致在内核源代码目录 (``/lib/modules/`uname -r`/build``) 中调用 :command:`make` 并引用当前目录 (``M = `pwd``)。该过程最终会读取当前目录中的 :file:`Kbuild` 文件，并按照该文件中的指示编译模块。

.. note:: 对于实验，我们将根据虚拟机的规格配置不同的 :command:`KDIR`：

          .. code-block:: bash

              KDIR = /home/student/src/linux
              [...]

:file:`Kbuild` 文件包含一个或多个指令，用于编译内核模块。其中一个最简单的指令示例是 ``obj-m = module.o``。根据该指令，将从 ``module.o`` 文件开始创建一个内核模块（ :code:`ko` ，即 :code:`kernel object`，也就是内核对象）。``module.o`` 将基于 ``module.c`` 或 ``module.S`` 创建。所有这些文件都可以在 :file:`Kbuild` 所在的目录中找到。

下面是一个使用多个子模块的 :file:`Kbuild` 文件示例：

.. code-block:: bash

   EXTRA_CFLAGS = -Wall -g

   obj-m        = supermodule.o
   supermodule-y = module-a.o module-b.o

对于上面的示例，编译步骤如下：

   * 编译 :file:`module-a.c` 和 :file:`module-b.c` 源文件，生成 module-a.o 和 module-b.o 对象文件（object）
   * 然后将 :file:`module-a.o` 和 :file:`module-b.o` 链接到 :file:`supermodule.o`
   * 基于 :file:`supermodule.o` 创建 :file:`supermodule.ko` 模块


:file:`Kbuild` 中目标（target）的后缀决定了它们的用途，如下所示：

   * M（modules） 标示目标为可加载内核模块

   * Y（yes） 表示目标是用于编译并链接到模块（``$(模块名称)-y``）或内核（``obj-y``）的对象文件

   * 其他任何目标后缀都将被 :file:`Kbuild` 忽略，不会被编译


.. note:: 这些后缀使得可以通过运行 :command:`make menuconfig` 命令或直接编辑 :file:`.config` 文件来轻松配置内核。该文件设置了一系列变量，用于确定在构建时向内核添加哪些特性。例如，使用 :command:`make menuconfig` 命令添加 BTRFS 支持时，在 :file:`.config` 文件中添加行 :code:`CONFIG_BTRFS_FS = y`。BTRFS kbuild 包含了一行 ``obj-$(CONFIG_BTRFS_FS):= btrfs.o``，它会转变成 ``obj-y:= btrfs.o``。这将编译 :file:`btrfs.o` 对象，并将其链接到内核。如果没有设置变量，该行会转变成 ``obj:=btrfs.o``，然后被忽略，进而内核构建时不会包含 BTRFS 支持。

要了解更多详细信息，请参阅内核源代码中的 :file:`Documentation/kbuild/makefiles.txt` 和 :file:`Documentation/kbuild/modules.txt` 文件。

..
  _[SECTION-COMPILE-MODULES-END]

..
  _[SECTION-LOAD-MODULES-BEGIN]

加载/卸载内核模块
================

要加载内核模块，请使用 :command:`insmod` 程序。该程序接收用于编译和链接模块的 :file:`*.ko` 文件的路径作为参数。要从内核中卸载模块请使用 :command:`rmmod` 命令，该命令接收模块名称作为参数。

.. code-block:: bash

   $ insmod module.ko
   $ rmmod module.ko

加载内核模块时，将执行 ``module_init`` 宏（macro）参数指定的函数。同样，当卸载模块时，将执行 ``module_exit`` 宏参数指定的函数。

下面是一个完整的编译、加载和卸载内核模块的示例：

.. code-block:: bash

   faust:~/lab-01/modul-lin# ls
   Kbuild  Makefile  modul.c

   faust:~/lab-01/modul-lin# make
   make -C /lib/modules/`uname -r`/build M=`pwd`
   make[1]: Entering directory `/usr/src/linux-2.6.28.4'
     LD      /root/lab-01/modul-lin/built-in.o
     CC [M]  /root/lab-01/modul-lin/modul.o
     Building modules, stage 2.
     MODPOST 1 modules
     CC      /root/lab-01/modul-lin/modul.mod.o
     LD [M]  /root/lab-01/modul-lin/modul.ko
   make[1]: Leaving directory `/usr/src/linux-2.6.28.4'

   faust:~/lab-01/modul-lin# ls
   built-in.o  Kbuild  Makefile  modul.c  Module.markers
   modules.order  Module.symvers  modul.ko  modul.mod.c
   modul.mod.o  modul.o

   faust:~/lab-01/modul-lin# insmod modul.ko

   faust:~/lab-01/modul-lin# dmesg | tail -1
   Hi

   faust:~/lab-01/modul-lin# rmmod modul

   faust:~/lab-01/modul-lin# dmesg | tail -2
   Hi
   Bye

可以使用 :command:`lsmod` 命令或查看 :file:`/proc/modules` 、 :file:`/sys/module` 目录来获取有关加载到内核中的模块的信息。

..
  _[SECTION-LOAD-MODULES-END]

..
  _[SECTION-DEBUG-MODULES-BEGIN]

内核模块调试
===========

与调试常规程序相比，调试内核模块要复杂得多。首先，内核模块中的错误可能导致整个系统阻塞。因此，故障排除的速度会大大降低。为了避免重新启动，建议使用虚拟机（qemu、virtualbox 或者 vmware）。

当插入包含错误的模块到内核中时，它最终会生成一个 `内核 oops <https://zh.wikipedia.org/wiki/Oops_(Linux内核)>`_ 。内核 oops 是内核检测到的无效操作，只可能由内核生成。对于稳定的内核版本，这几乎可以肯定意味着模块含有错误。在 oops 出现后，内核仍将继续工作。

出现内核 oops 时，保存生成的消息非常重要。如上所述，内核生成的消息保存在日志中，并可使用 :command:`dmesg` 命令显示。为了确保不丢失任何内核消息，建议直接从控制台插入/测试内核，或定期检查内核消息。值得注意的是，oops 不止可能是由于编程错误，也有可能是硬件错误引起的。

如果发生致命错误，系统无法恢复到稳定状态，将造成 `内核错误（kernel panic） <https://zh.wikipedia.org/wiki/内核错误>`_ 。

以下是一个包含错误并会造成 oops 的内核模块示例：

.. code-block:: c

    /*
     * 造成 oops 的内核模块
     */

    #include <linux/kernel.h>
    #include <linux/module.h>
    #include <linux/init.h>

    MODULE_DESCRIPTION ("Oops");
    MODULE_LICENSE ("GPL");
    MODULE_AUTHOR ("PSO");

    #define OP_READ         0
    #define OP_WRITE        1
    #define OP_OOPS         OP_WRITE

    static int my_oops_init (void)
    {
            int *a;

            a = (int *) 0x00001234;
    #if OP_OOPS == OP_WRITE
            *a = 3;
    #elif OP_OOPS == OP_READ
            printk (KERN_ALERT "value = %d\n", *a);
    #else
    #error "Unknown op for oops!"
    #endif

            return 0;
    }

    static void my_oops_exit (void)
    {
    }

    module_init (my_oops_init);
    module_exit (my_oops_exit);

.. **

将此模块插入内核将造成 oops：

.. code-block:: bash

   faust:~/lab-01/modul-oops# insmod oops.ko
   [...]

   faust:~/lab-01/modul-oops# dmesg | tail -32
   BUG: unable to handle kernel paging request at 00001234
   IP: [<c89d4005>] my_oops_init+0x5/0x20 [oops]
     *de = 00000000
   Oops: 0002 [#1] PREEMPT DEBUG_PAGEALLOC
   last sysfs file: /sys/devices/virtual/net/lo/operstate
   Modules linked in: oops(+) netconsole ide_cd_mod pcnet32 crc32 cdrom [last unloaded: modul]

   Pid: 4157, comm: insmod Not tainted (2.6.28.4 #2) VMware Virtual Platform
   EIP: 0060:[<c89d4005>] EFLAGS: 00010246 CPU: 0
   EIP is at my_oops_init+0x5/0x20 [oops]
   EAX: 00000000 EBX: fffffffc ECX: c89d4300 EDX: 00000001
   ESI: c89d4000 EDI: 00000000 EBP: c5799e24 ESP: c5799e24
    DS: 007b ES: 007b FS: 0000 GS: 0033 SS: 0068
   Process insmod (pid: 4157, ti=c5799000 task=c665c780 task.ti=c5799000)
   Stack:
    c5799f8c c010102d c72b51d8 0000000c c5799e58 c01708e4 00000124 00000000
    c89d4300 c5799e58 c724f448 00000001 c89d4300 c5799e60 c0170981 c5799f8c
    c014b698 00000000 00000000 c5799f78 c5799f20 00000500 c665cb00 c89d4300
   Call Trace:
    [<c010102d>] ? _stext+0x2d/0x170
    [<c01708e4>] ? __vunmap+0xa4/0xf0
    [<c0170981>] ? vfree+0x21/0x30
    [<c014b698>] ? load_module+0x19b8/0x1a40
    [<c035e965>] ? __mutex_unlock_slowpath+0xd5/0x140
    [<c0140da6>] ? trace_hardirqs_on_caller+0x106/0x150
    [<c014b7aa>] ? sys_init_module+0x8a/0x1b0
    [<c0140da6>] ? trace_hardirqs_on_caller+0x106/0x150
    [<c0240a08>] ? trace_hardirqs_on_thunk+0xc/0x10
    [<c0103407>] ? sysenter_do_call+0x12/0x43
   Code: <c7> 05 34 12 00 00 03 00 00 00 5d c3 eb 0d 90 90 90 90 90 90 90 90
   EIP: [<c89d4005>] my_oops_init+0x5/0x20 [oops] SS:ESP 0068:c5799e24
   ---[ end trace 2981ce73ae801363 ]---

虽然相对晦涩，但内核在出现 oops 时提供了有关错误的宝贵信息。第一行：

.. code-block:: bash

   BUG: unable to handle kernel paging request at 00001234
   EIP: [<c89d4005>] my_oops_init + 0x5 / 0x20 [oops]

告诉我们错误的原因和生成错误的指令的地址。在我们的例子中，这是对内存的无效访问。

接下来的一行是：

   ``Oops: 0002 [# 1] PREEMPT DEBUG_PAGEALLOC``

告诉我们这是第一个 oops（#1）。在这个上下文中，这很重要，因为一个 oops 可能会导致其他 oops。通常只有第一个 oops 是相关的。此外，oops 代码（ ``0002`` ）提供了有关错误类型的信息（参见 :file:`arch/x86/include/asm/trap_pf.h` ）：

   * Bit 0 == 0 表示找不到页面，1 表示保护故障
   * Bit 1 == 0 表示读取，1 表示写入
   * Bit 2 == 0 表示内核模式，1 表示用户模式

在这种情况下，我们有一个写入访问导致了 oops（bit 1 为 1）。

下面是寄存器的转储（dump）。它解码了指令指针 (``EIP``) 的值，并指出错误出现在 :code:`my_oops_init` 函数中，偏移为 5 个字节（``EIP: [<c89d4005>] my_oops_init+0x5``）。该消息还显示了堆栈内容和到目前为止的调用回溯。

如果发生了无效的读取调用 (``#define OP_OOPS OP_READ``)，消息将是相同的，但是 oops 代码将不同，现在将是 ``0000``:

.. code-block:: bash

   faust:~/lab-01/modul-oops# dmesg | tail -33
   BUG: unable to handle kernel paging request at 00001234
   IP: [<c89c3016>] my_oops_init+0x6/0x20 [oops]
     *de = 00000000
   Oops: 0000 [#1] PREEMPT DEBUG_PAGEALLOC
   last sysfs file: /sys/devices/virtual/net/lo/operstate
   Modules linked in: oops(+) netconsole pcnet32 crc32 ide_cd_mod cdrom

   Pid: 2754, comm: insmod Not tainted (2.6.28.4 #2) VMware Virtual Platform
   EIP: 0060:[<c89c3016>] EFLAGS: 00010292 CPU: 0
   EIP is at my_oops_init+0x6/0x20 [oops]
   EAX: 00000000 EBX: fffffffc ECX: c89c3380 EDX: 00000001
   ESI: c89c3010 EDI: 00000000 EBP: c57cbe24 ESP: c57cbe1c
    DS: 007b ES: 007b FS: 0000 GS: 0033 SS: 0068
   Process insmod (pid: 2754, ti=c57cb000 task=c66ec780 task.ti=c57cb000)
   Stack:
    c57cbe34 00000282 c57cbf8c c010102d c57b9280 0000000c c57cbe58 c01708e4
    00000124 00000000 c89c3380 c57cbe58 c5db1d38 00000001 c89c3380 c57cbe60
    c0170981 c57cbf8c c014b698 00000000 00000000 c57cbf78 c57cbf20 00000580
   Call Trace:
    [<c010102d>] ? _stext+0x2d/0x170
    [<c01708e4>] ? __vunmap+0xa4/0xf0
    [<c0170981>] ? vfree+0x21/0x30
    [<c014b698>] ? load_module+0x19b8/0x1a40
    [<c035d083>] ? printk+0x0/0x1a
    [<c035e965>] ? __mutex_unlock_slowpath+0xd5/0x140
    [<c0140da6>] ? trace_hardirqs_on_caller+0x106/0x150
    [<c014b7aa>] ? sys_init_module+0x8a/0x1b0
    [<c0140da6>] ? trace_hardirqs_on_caller+0x106/0x150
    [<c0240a08>] ? trace_hardirqs_on_thunk+0xc/0x10
    [<c0103407>] ? sysenter_do_call+0x12/0x43
   Code: <a1> 34 12 00 00 c7 04 24 54 30 9c c8 89 44 24 04 e8 58 a0 99 f7 31
   EIP: [<c89c3016>] my_oops_init+0x6/0x20 [oops] SS:ESP 0068:c57cbe1c
   ---[ end trace 45eeb3d6ea8ff1ed ]---

objdump
-------

可以使用 :command:`objdump` 程序找到造成 oops 的指令（instruction）的详细信息。常用的选项有 :command:`-d` 用于反汇编代码， :command:`-S` 用于将 C 代码与汇编语言代码交错显示。然而，为了进行高效的解码，我们需要找到内核模块被加载到的地址。这可以在 :file:`/proc/modules` 中找到。

以下是在上述模块上使用 :command:`objdump` 的示例，以确定造成 oops 的指令：

.. code-block:: bash

   faust:~/lab-01/modul-oops# cat /proc/modules
   oops 1280 1 - Loading 0xc89d4000
   netconsole 8352 0 - Live 0xc89ad000
   pcnet32 33412 0 - Live 0xc895a000
   ide_cd_mod 34952 0 - Live 0xc8903000
   crc32 4224 1 pcnet32, Live 0xc888a000
   cdrom 34848 1 ide_cd_mod, Live 0xc886d000

   faust:~/lab-01/modul-oops# objdump -dS --adjust-vma=0xc89d4000 oops.ko

   oops.ko:     file format elf32-i386


   Disassembly of section .text:

   c89d4000 <init_module>:
   #define OP_READ         0
   #define OP_WRITE        1
   #define OP_OOPS         OP_WRITE

   static int my_oops_init (void)
   {
   c89d4000:       55                      push   %ebp
   #else
   #error "Unknown op for oops!"
   #endif

           return 0;
   }
   c89d4001:       31 c0                   xor    %eax,%eax
   #define OP_READ         0
   #define OP_WRITE        1
   #define OP_OOPS         OP_WRITE

   static int my_oops_init (void)
   {
   c89d4003:       89 e5                   mov    %esp,%ebp
           int *a;

           a = (int *) 0x00001234;
   #if OP_OOPS == OP_WRITE
           *a = 3;
   c89d4005:       c7 05 34 12 00 00 03    movl   $0x3,0x1234
   c89d400c:       00 00 00
   #else
   #error "Unknown op for oops!"
   #endif

           return 0;
   }
   c89d400f:       5d                      pop    %ebp
   c89d4010:       c3                      ret
   c89d4011:       eb 0d                   jmp    c89c3020 <cleanup_module>
   c89d4013:       90                      nop
   c89d4014:       90                      nop
   c89d4015:       90                      nop
   c89d4016:       90                      nop
   c89d4017:       90                      nop
   c89d4018:       90                      nop
   c89d4019:       90                      nop
   c89d401a:       90                      nop
   c89d401b:       90                      nop
   c89d401c:       90                      nop
   c89d401d:       90                      nop
   c89d401e:       90                      nop
   c89d401f:       90                      nop

   c89d4020 <cleanup_module>:

   static void my_oops_exit (void)
   {
   c89d4020:       55                      push   %ebp
   c89d4021:       89 e5                   mov    %esp,%ebp
   }
   c89d4023:       5d                      pop    %ebp
   c89d4024:       c3                      ret
   c89d4025:       90                      nop
   c89d4026:       90                      nop
   c89d4027:       90                      nop

请注意，生成 oops 的指令（先前确定为 ``c89d4005`` ）是：

  ```C89d4005: c7 05 34 12 00 00 03 movl $ 0x3,0x1234``

这正是预期的结果 - 将值 3 存储在地址 0x0001234 上。

:file:`/proc/modules` 用于查找加载的内核模块的地址。:command:`--adjust-vma` 选项允许你相对于 ``0xc89d4000`` 位置显示指令。:command:`-l` 选项将显示源代码中每行的编号，源代码与汇编语言代码交错显示。

addr2line
---------

寻找造成 oops 的代码的一种更简单的方法是使用 :command:`addr2line` 实用程序：

.. code-block:: bash

   faust:~/lab-01/modul-oops# addr2line -e oops.o 0x5
   /root/lab-01/modul-oops/oops.c:23

其中 ``0x5`` 是生成 oops 的程序计数器的值（``EIP = c89d4005``），减去根据 :file:`/proc/modules` 的信息得出的模块的基地址（``0xc89d4000``）。

minicom
-------

:command:`Minicom` (或其他等效程序，例如 :command:`picocom` 以及 :command:`screen`) 是一种用于与串行端口（serial port）连接和交互的程序。串行端口是在开发阶段分析内核消息（kernel message）或与嵌入式系统进行交互的基本方法。有两种常见的连接方式：

* 使用串行端口，设备路径为 :file:`/dev/ttyS0`

* 使用串行 USB 端口（FTDI），在这种情况下，设备路径为 :file:`/dev/ttyUSB`

对于实验中使用的虚拟机，在虚拟机启动后，我们需要使用的设备路径将显示在屏幕上：

.. code-block:: bash

    char device redirected to /dev/pts/20 (label virtiocon0)

使用 Minicom：

.. code-block:: bash

   # 使用 COM1 连接，速率为 115,200 字符/秒
   minicom -b 115200 -D /dev/ttyS0

   # 使用 USB 串行端口连接
   minicom -D /dev/ttyUSB0

   # 连接到虚拟机的串行端口
   minicom -D /dev/pts/20

netconsole
----------

:command:`Netconsole` 是允许通过网络记录内核调试消息的程序。当磁盘日志系统不起作用、串行端口不可用或终端不响应命令时，这非常有用。:command:`Netconsole` 是内核模块。

要想正常工作，它需要以下参数：

   * 调试站点的端口、IP 地址和源接口名称
   * 将调试消息发送到的机器的端口、MAC 地址和 IP 地址

这些参数可以在将模块插入内核时进行配置，甚至在模块插入后也可以配置，如果模块在编译时配置了 ``CONFIG_NETCONSOLE_DYNAMIC`` 选项。

插入 :command:`netconsole` 内核模块时的示例配置如下：

.. code-block:: bash

   alice:~# modprobe netconsole netconsole=6666@192.168.191.130/eth0,6000@192.168.191.1/00:50:56:c0:00:08

因此，在具有地址 ``192.168.191.130`` 的站点上，调试消息将被发送到 ``eth0`` 接口，源端口为 ``6666``。消息将被发送到 ``192.168.191.1``，使用 MAC 地址 ``00:50:56:c0:00:08``，至端口 ``6000`` 上。

可以在目标站点上使用 :command:`netcat` 显示消息：

.. code-block:: bash

   bob:~ # nc -l -p 6000 -u

或者，目标站点可以配置 :command:`syslogd` 来拦截这些消息。更多信息可以在 :file:`Documentation/networking/netconsole.txt` 中找到。

Printk 调试
-----------

``最古老且最有用的两种调试辅助工具是你的大脑和 Printf``。

在调试过程中，通常会使用一种原始但非常有效的方法：:code:`printk` 调试。尽管也可以使用调试器，但通常并不是非常有用：简单的错误（未初始化的变量、内存管理问题等）可以通过控制消息和内核解码的 oops 消息轻松找到。

对于更复杂的错误，即使是调试器也无法提供太多帮助，除非你对操作系统的结构非常熟悉。在调试内核模块时，其中存在许多未知因素：多个上下文（我们同时运行多个进程和线程）、中断以及虚拟内存等等。

你可以使用 :code:`printk` 将内核消息显示到用户空间。它类似于 :code:`printf` 的功能；唯一的区别是传输的消息可以用 :code:`"<n>"` 字符串为前缀，其中 :code:`n` 表示错误级别（日志级别），取值范围为 ``0`` 到 ``7`` 。除了 :code:`"<n>"`，级别也可以用符号常量编码：

.. code-block:: c

    KERN_EMERG——n = 0
    KERN_ALERT——n = 1
    KERN_CRIT——n = 2
    KERN_ERR——n = 3
    KERN_WARNING——n = 4
    KERN_NOTICE——n = 5
    KERN_INFO——n = 6
    KERN_DEBUG——n = 7


所有日志级别的定义都可以在 :file:`linux/kern_levels.h` 中找到。基本上，系统凭借这些日志级别将消息发送到各种输出：控制台、位于 :file:`/var/log` 中的日志文件等等。

.. note:: 要在用户空间显示 :code:`printk` 消息，:code:`printk` 日志级别必须比 `console_loglevel` 变量的优先级高。可以从 :file:`/proc/sys/kernel/printk` 配置默认的控制台日志级别。

例如，以下命令：

.. code-block:: bash

    echo 8 > /proc/sys/kernel/printk

将使所有内核日志消息在控制台上显示。也就是说，日志级别必须严格小于 :code:`console_loglevel` 变量。例如，如果 :code:`console_loglevel` 的值为 ``5``（指定于 :code:`KERN_NOTICE`），只有比 ``5`` 更严格的日志级别的消息（即 :code:`KERN_EMERG` 、:code:`KERN_ALERT` 、:code:`KERN_CRIT` 、:code:`KERN_ERR` 以及 :code:`KERN_WARNING`）将显示。

想要快速查看执行内核代码的效果的话，控制台重定向的消息可能对你很有帮助，但如果内核遇到不可修复的错误并且系统冻结，则不再那么有用。在这种情况下，必须查看系统的日志，因为它们保留从一次系统启动到下一次系统重新启动之间的信息。这些日志文件位于 :file:`/var/log` 中，是由内核运行期间的 :code:`syslogd` 和 :code:`klogd` 填充的文本文件。:code:`syslogd` 和 :code:`klogd` 从挂载在 :file:`/proc` 中的虚拟文件系统中获取信息。原则上，打开 :code:`syslogd` 和 :code:`klogd` 后，来自内核的所有消息都将发送到 :file:`/var/log/kern.log`。

调试的更简单的方法是使用 :file:`/var/log/debug` 文件。它只包含具有 :code:`KERN_DEBUG` 日志级别的内核的 :code:`printk` 消息。

给定一个生产内核（production kernel）（类似于我们可能正在运行的内核），其只包含发布代码，我们的模块是少数几个带有以 KERN_DEBUG 为前缀的消息的模块之一。通过查找与我们的模块的调试会话对应的消息，我们可以轻松浏览 :file:`/var/log/debug` 中的信息。

一个示例如下：

.. code-block:: bash

    # 清除先前信息的调试文件（或可能是备份文件）
    $ echo "新调试会话" > /var/log/debug
    # 运行测试
    # 如果没有导致内核崩溃的关键错误，检查输出
    # 如果发生关键错误且机器只能通过重新启动来响应，请重新启动系统并检查 /var/log/debug。

消息的格式显然必须包含所有相关信息，以便检测错误，但插入代码 :code:`printk` 以提供详细信息可能会花费与编写代码解决问题一样多的时间。通常在使用 :code:`printk` 显示的调试消息的完整性与将这些消息插入文本中所需的时间之间需要有权衡。

一种非常简单、插入 :code:`printk` 更省时并使我们能够分析测试指令流的方法是使用预定义的常量 :code:`__FILE__` 、:code:`__LINE__` 和 :code:`__func__`：

    * ``__FILE__`` 会被编译器替换为当前正在编译的源文件的名称。
    * ``__LINE__`` 会被编译器替换为当前源文件中当前指令所在的行号。
    * ``__func__`` / ``__FUNCTION__`` 会被编译器替换为当前指令所在的函数的名称。

.. note::
    :code:`__FILE__` 和 :code:`__LINE__` 是 ANSI C 规范的一部分，:code:`__func__` 是 C99 规范的一部分；:code:`__FUNCTION__` 是 GNU 的一个 C 扩展，不具有可移植性；然而，由于我们编写的代码是针对 Linux 内核的，所以可以毫无问题地使用它们。

可以在这种情况下使用以下宏定义：

.. code-block:: c

   #define PRINT_DEBUG \
          printk (KERN_DEBUG "[% s]: FUNC:% s: LINE:% d \ n", __FILE__,
                  __FUNCTION__, __LINE__)

然后，在每个想要查看执行是否“到达”的位置，插入 PRINT_DEBUG；这是一种简单快捷的方法，通过仔细分析输出可以得出结果。

:command:`dmesg` 命令用于查看在控制台上不显示，需要使用 :code:`printk` 来打印的消息。

要删除日志文件中的所有先前消息，请运行：

.. code-block:: bash

    cat /dev/null > /var/log/debug

要删除 :command:`dmesg` 命令显示的消息，请运行：

.. code-block:: bash

    dmesg -c


动态调试
--------

动态调试（ `dyndbg <https://www.kernel.org/doc/html/v4.15/admin-guide/dynamic-debug-howto.html>`_ ）技术可以动态地激活/停用调试。与 :code:`printk` 不同，它提供了更高级的 :code:`printk` 选项，可以用于仅显示我们想要的消息；其对于复杂模块或故障排除子系统非常有用。这显著减少了显示的消息数量，只留下与调试上下文相关的消息。要启用 ``dyndbg`` ，内核必须编译时启用 ``CONFIG_DYNAMIC_DEBUG`` 选项。一旦配置了这个选项，就可以每次调用时动态启用 :code:`pr_debug()` 、 :code:`dev_dbg()` 和 :code:`print_hex_dump_debug()` 、 :code:`print_hex_dump_bytes()`。

debugfs 中的 :file:`/sys/kernel/debug/dynamic_debug/control` 文件可以用于过滤消息或查看现有过滤器。

.. code-block:: c

   mount -t debugfs none /debug

`Debugfs <http://opensourceforu.com/2010/10/debugging-linux-kernel-with-debugfs/>`_ 是个简单的文件系统，用作内核空间接口和用户空间接口，以配置不同的调试选项。任何调试工具都可以在 debugfs 中创建和使用自己的文件/文件夹。

例如，要显示 ``dyndbg`` 中的现有过滤器，可以使用以下命令：

.. code-block:: bash

   cat /debug/dynamic_debug/control

要启用 :file:`svcsock.c` 文件中第 ``1603`` 行的调试消息：

.. code-block:: bash

   echo 'file svcsock.c line 1603 +p' > /debug/dynamic_debug/control

:file:`/debug/dynamic_debug/control` 文件不是普通文件。它显示了过滤器的 ``dyndbg`` 设置。使用 echo 在其中写入会更改这些设置（实际上不会进行写入）。请注意，该文件包含了 ``dyndbg`` 调试消息的设置。不要在该文件中进行日志记录。

Dyndbg 选项
~~~~~~~~~~

* ``func`` ——只显示与过滤器中定义的函数名称相同的函数的调试消息。

  .. code-block:: bash

      echo 'func svc_tcp_accept +p' > /debug/dynamic_debug/control

* ``file`` ——要显示调试消息的文件名。可以只是源文件名，也可以是绝对路径或内核树路径。

  .. code-block:: bash

    file svcsock.c
    file kernel/freezer.c
    file /usr/src/packages/BUILD/sgi-enhancednfs-1.4/default/net/sunrpc/svcsock.c

* ``module`` ——显示模块名称。

  .. code-block:: bash

     module sunrpc

* ``format`` ——只显示显示格式包含指定字符串的消息。

  .. code-block:: bash

     format "nfsd: SETATTR"

* ``line`` - 显示调试调用的行号或行号范围。

  .. code-block:: bash

     # 在 svcsock.c 文件的第 1603 行到第 1605 行之间触发调试消息
     $ echo 'file svcsock.c line 1603-1605 +p' > /sys/kernel/debug/dynamic_debug/control
     # 从文件开头到第 1605 行启用调试消息
     $ echo 'file svcsock.c line -1605 +p' > /sys/kernel/debug/dynamic_debug/control

除了上述选项外，还可以使用操作符 ``+``、 ``-`` 或 ``=`` 来添加、删除或设置一系列标志：

   * ``p`` 激活 pr_debug()。
   * ``f`` 在打印的消息中包含函数名。
   * ``l`` 在打印的消息中包含行号。
   * ``m`` 在打印的消息中包含模块名称。
   * ``t`` 如果不是从中断上下文调用，则包括线程 ID。
   * ``_`` 不设置标志。

KDB：内核调试器
--------------

内核调试器已被证明在开发和调试过程中非常有用。其主要优势之一是可以进行实时调试。这使得我们能够实时监视对内存的访问，甚至在调试过程中修改内存。内核调试器从版本 2.6.26-rc1 开始，已集成到主线内核中。KDB 不是一个 *源代码调试器*，但在进行完整分析时，可以与 gdb 和符号文件并行使用——请参见 :ref:`GDB调试部分 <gdb_intro>`

要使用 KDB，你有以下选项：

 * 非 USB 键盘 + VGA 文本控制台
 * 串口控制台
 * USB EHCI 调试端口

在实验中，我们将使用连接到主机的串口接口。以下命令将在串口上激活 GDB：

.. code-block:: bash

  echo hvc0 > /sys/module/kgdboc/parameters/kgdboc

KDB 是一种 *停止模式调试器*，这意味着在其活动期间，所有其他进程都将停止。可以使用 `SysRq <http://zh.wikipedia.org/wiki/Magic_SysRq組合鍵>`__ 命令强制内核在执行过程中进入 KDB

.. code-block:: bash

  echo g > /proc/sysrq-trigger

或者在连接到串口的终端中使用键盘组合键 ``Ctrl+O g`` (例如使用 :command:`minicom`)。

KDB 具有各种命令来控制和定义被调试系统的上下文：

 * lsmod, ps, kill, dmesg, env, bt（backtrace，回溯）
 * 转储跟踪日志
 * 硬件断点
 * 修改内存

要获取有关可用命令的更详细描述，可以在 KDB shell 中使用 ``help`` 命令。在下一个示例中，你可以看到一个简单的 KDB 使用示例，它设置了一个硬件断点来监视 ``mVar`` 变量的更改。

.. code-block:: bash

  # 触发 KDB
  echo g > /proc/sysrq-trigger
  # 或者如果我们连接到串口，使用以下命令
  Ctrl-O g
  # 在对 mVar 变量进行写访问时设置断点
  kdb> bph mVar dataw
  # 从KDB返回
  kdb> go

..
  _[SECTION-DEBUG-MODULES-END]

练习
====

.. _exercises_summary:

.. include:: ../labs/exercises-summary.hrst
.. |LAB_NAME| replace:: kernel_modules

0. 引言
--------

使用 :command:`cscope` 或 |LXR|_ 在 Linux 内核源代码中查找以下符号的定义：

* :c:func:`module_init` 和 :c:func:`module_exit`

  - 这两个宏的作用是什么？ ``init_module`` 和 ``cleanup_module`` 是什么？

* :c:data:`ignore_loglevel`

  - 这个变量用于什么？

.. warning::
  如果使用 :command:`cscope` 时遇到问题，可能是数据库没有生成。要生成数据库，请在内核目录中使用以下命令：

  .. code-block:: bash

    make ARCH=x86 cscope

.. note::
  在使用 :command:`cscope` 搜索结构时，只使用结构名（不包括 :code:`struct`）。所以，要搜索结构 :c:type:`struct module`，可以使用以下命令：

   .. code-block:: bash

     vim -t module

  或者在 :command:`vim` 中使用命令：

   .. code-block:: bash

     :cs f g module

.. note::
  有关使用 :command:`cscope` 的更多信息，请阅读上一个实验的 :ref:`cscope 章节 <cscope_intro>`。

..
  _[EXERCISE1-BEGIN]

1. 内核模块
-----------

为了使用内核模块，我们将按照 :ref:`上述 <exercises-summary>` 步骤进行操作。

首先在 `tools/labs` 目录下运行以下命令生成名为 **1-2-test-mod** 的任务骨架，然后构建并复制模块到虚拟机中。

.. code-block:: bash

  $ LABS=kernel_modules make skels
  $ make build
  $ make copy

这些命令将构建并复制当前实验骨架中的所有模块。

.. warning::
  在解决练习 3 之前，编译 ``3-error-mod`` 时会出现编译错误。为了避免此问题，删除 :file:`skels/kernel_modules/3-error-mod/` 目录，并从 ``skels/Kbuild`` 中删除相应的行。

使用 :command:`make boot` 启动虚拟机，使用 `minicom -D serial.pts` 连接到串行控制台，并执行以下任务：

* 加载内核模块。

* 列出内核模块并检查当前模块是否存在。

* 卸载内核模块。

* 使用 :command:`dmesg` 命令查看加载/卸载内核模块时显示的消息。

.. note:: 请阅读 `加载/卸载内核模块`_ 部分。在卸载内核模块时，只需指定模块名称（不包括扩展名）。

..
  _[EXERCISE1-END]

..
  _[EXERCISE2-BEGIN]


2. Printk
---------

观察虚拟机控制台。为什么消息直接显示在虚拟机控制台上？

配置系统，使消息不直接显示在串行控制台上，只能使用 ``dmesg`` 命令来查看。

.. hint:: 一种方法是通过将所需级别写入 ``/proc/sys/kernel/printk`` 来设置控制台日志级别。使用的值应小于模块源代码中用于打印消息的级别。

重新加载/卸载该模块。消息不应该打印到虚拟机控制台上，但是在运行 ``dmesg`` 命令时应该可见。

..
  _[EXERCISE2-END]

..
  _[EXERCISE3-BEGIN]

3. 错误
--------

生成名为 **3-error-mod** 的任务的框架。编译源代码并得到相应的内核模块。

为什么会出现编译错误? **提示:** 这个模块与前一个模块有什么不同？

修改该模块以解决这些错误的原因，然后编译和测试该模块。

..
  _[EXERCISE3-END]

..
  _[EXERCISE4-BEGIN]

4. 子模块
---------

查看 :file:`4-multi-mod/` 目录中的 C 源代码文件 ``mod1.c`` 和 ``mod2.c``。模块 2 仅包含模块 1 使用的函数的定义。

修改 :file:`Kbuild` 文件，从这两个 C 源文件创建 ``multi_mod.ko`` 模块。

.. hint:: 阅读实验室中的 `编译内核模块`_ 部分。

编译、复制、启动虚拟机、加载和卸载内核模块。确保消息在控制台上正确显示。

..
  _[EXERCISE4-END]

..
  _[EXERCISE5-BEGIN]

5. 内核 oops
--------------

进入任务目录 **5-oops-mod** 并检查 C 源代码文件。注意问题将在哪里发生。在 Kbuild 文件中添加编译标记 ``-g``。

.. hint:: 阅读实验中的 `编译内核模块`_ 部分。

编译相应的模块并将其加载到内核中。识别 oops 出现的内存地址。

.. hint:: 阅读实验中的 `调试`_ 部分。要识别地址，请遵循 oops 消息并提取指令指针 (``EIP``) 寄存器的值。

确定是哪条指令触发了 oops。

.. hint:: 使用 :file:`proc/modules` 信息获取内核模块的加载地址。在物理机上使用 objdump 和/或 addr2line。Objdump 需要编译时开启调试支持！请阅读实验中的 `objdump`_ 和 `addr2line`_ 部分。

尝试卸载内核模块。请注意，该操作无法成功，因为自 oops 发生以来，内核模块内部仍然存在对内核的引用；在释放这些引用之前（在 oops 的情况下几乎不可能），模块无法卸载。

..
  _[EXERCISE5-END]

..
  _[EXERCISE6-BEGIN]

6. 模块参数
-----------

进入任务目录 **6-cmd-mod** 并检查 C 源代码文件 ``cmd_mod.c``。编译并复制相关的模块，然后加载内核模块以查看 printk 消息。然后从内核中卸载该模块。

在不修改源代码的情况下，加载内核模块以显示消息 ``Early bird gets tired``。

.. hint:: 可以通过向模块传递参数来更改 str 变量。在 `这里 <http://tldp.org/LDP/lkmpg/2.6/html/x323.html>`_ 找到更多相关信息。

.. _proc-info:

..
  _[EXERCISE6-END]

..
  _[EXERCISE7-BEGIN]

7. 进程信息
------------

检查名为 **7-list-proc** 的任务的框架。添加代码来显示当前进程的进程 ID（ ``PID`` ）和可执行文件名。

按照标记为 ``TODO`` 的命令进行操作。在加载和卸载模块时，必须显示这些信息。

.. note::
          * 在Linux内核中，进程由 :c:type:`struct task_struct` 描述。使用 |LXR|_ 或 ``cscope`` 来查找 :c:type:`struct task_struct` 的定义。

          * 要找到包含可执行文件名的结构字段，请查找“executable”的注释。

          * 内核中给定时间运行的当前进程的结构指针由 :c:macro:`current` 变量（类型为 :c:type:`struct task_struct*`）给出。

.. hint:: 要使用 :c:macro:`current`，你需要包含定义 :c:type:`struct task_struct` 的头文件，即 ``linux/sched.h``。

编译、复制、启动虚拟机并加载模块。卸载内核模块。

重复加载/卸载操作。注意显示的进程 PID 是不同的。这是因为在加载模块时，从可执行文件 :file:`/sbin/insmod` 创建了一个进程，而在卸载模块时，从可执行文件 :file:`/sbin/rmmod` 创建了一个进程。

..
  _[EXERCISE7-END]

..
  _[EXTRA-EXERCISE-BEGIN]

Extra Exercises
===============

额外练习
=======

1. KDB
------

进入 **8-kdb** 目录。使用 :command:`SysRq` 命令通过串口激活 KDB 并进入 KDB 模式。使用 :command:`minicom` 连接到与 virtiocon0 相链接的伪终端，配置 KDB 使用 hvc0 串口：

.. code-block:: bash

    echo hvc0 > /sys/module/kgdboc/parameters/kgdboc

然后使用 SysRq 命令启用 KDB (:command:`Ctrl + O g`)。查看当前系统状态（使用 :command:`help` 命令查看可用的 KDB 命令）。使用 :command:`go` 命令继续内核执行。

加载 :file:`hello_kdb` 模块。该模块在写入 :file:`/proc/hello_kdb_bug` 文件时会模拟一个错误。使用以下命令模拟错误：

.. code-block:: bash

    echo 1 > /proc/hello_kdb_bug

运行上述命令后，每次出现 oops/内核崩溃 时，内核会停止执行并进入调试模式。

分析堆栈跟踪并确定导致错误的代码。我们如何从 KDB 中找到模块加载的地址？

同时，在一个新窗口中使用 GDB 并根据 KDB 提供的信息查看代码。

.. hint::
    加载符号文件。使用 :command:`info line` 命令。

当写入 :file:`/proc/hello_kdb_break` 时，模块将递增 :c:data:`kdb_write_address` 变量。进入 KDB 并设置每次对 :c:data:`kdb_write_address` 变量进行写入访问的断点。返回内核以触发写入，使用以下命令：

.. code-block:: bash

    echo 1 > /proc/hello_kdb_break

2. PS模块
----------

更新在 :ref:`proc-info` 处创建的内核模块，以便在插入内核模块时显示有关系统中所有进程的信息，而不仅仅是当前进程的信息。然后，将获得的结果与 :command:`ps` 命令的输出进行比较。

.. hint::
    * 系统中的进程以循环列表的形式组织。

    * :c:macro:`for_each_...` 宏（例如 :c:macro:`for_each_process`）在你希望遍历列表中的项目时非常有用。

    * 要了解如何使用某个功能或宏，请使用 |LXR|_ 或 Vim 和 :command:`cscope` 进行搜索并查找使用场景。

3. 内存信息
-----------

创建一个内核模块，显示当前进程的虚拟内存区域；对于每个内存区域，它将显示起始地址和结束地址。

.. hint::
    * 从现有的内核模块开始。

    * 研究结构 :c:type:`struct task_struct`、:c:type:`struct mm_struct` 和 :c:type:`struct vm_area_struct`。内存区域由类型为 :c:type:`struct vm_area_struct` 的结构表示。

    * 不要忘记包含定义必要结构的头文件。

4. 动态调试
----------

进入 **9-dyndbg** 目录并编译 :code:`dyndbg.ko` 模块。

熟悉挂载在 :file:`/debug` 中的 :code:`debugfs` 文件系统，并分析文件 :file:`/debug/dynamic_debug/control` 的内容。插入 :code:`dyndbg.ko` 模块并注意 :file:`dynamic_debug/control` 文件的新内容。

在相应的文件中出现了什么额外内容？运行以下命令：

.. code-block:: bash

    grep dyndbg /debug/dynamic_debug/control

配置 :command:`dyndbg`，以便在卸载模块时仅显示来自 :c:func:`my_debug_func` 的标记为“Important”的消息，本练习仅过滤掉 :c:func:`pr_debug` 调用；始终显示 :c:func:`printk` 调用。

请指定两种过滤方式。

.. hint::
    阅读 `动态调试`_ 部分并查看 :command:`dyndbg` 的选项（例如 :command:`line` 、:command:`format`）。

执行过滤操作并检查 :file:`dynamic_debug/control` 文件。发生了什么变化？如何知道哪些调用被激活了？

.. hint::
    检查 :command:`dyndbg` 标志。卸载内核模块并观察日志消息。

5. 初始化期间的动态调试
------------------------

正如你注意到的那样，只有在插入模块后才能激活/过滤 :c:func:`pr_debug` 调用。在某些情况下，查看模块初始化期间的消息可能会很有帮助。可以通过使用一个名为 :command:`dyndbg` 的默认（伪）参数作为初始化模块的参数来实现。使用此参数，你可以添加/删除 :command:`dyndbg` 标志。

.. hint::
    阅读 `动态调试`_ 部分的最后一部分，查看可用的标志（例如： :command:`+/- p`）。

阅读 `模块初始化时的调试消息部分 <https://01.org/linuxgraphics/gfx-docs/drm/admin-guide/dynamic-debug-howto.html#debug-messages-at-module-initialization-time>`_ ，并插入模块以便在初始化期间显示 :c:func:`my_debug_func`（即 :c:func:`dyndbg_init`）中的消息。

.. warning::
    在实验的虚拟机中，你需要使用 :command:`insmod` 而不是 :command:`modprobe`。

在不卸载模块的情况下，停用 :c:func:`pr_debug` 调用。

.. hint::
    你可以删除设置的标志。卸载内核模块。

..
  _[EXTRA-EXERCISE-END]
