=====================================================
作业 7——使用 KVM 的 SO2 虚拟机管理器
=====================================================

- 截止日期: :command:`2023 年 5 月 29 日，23:00`
- 这个作业可以由团队完成（最多 2 人）。只需由其中一人提交作业，并在 README 文件中列出学生的姓名。

在本作业中，我们将使用 Linux 内核中的 KVM API 来开发一个简单的虚拟机管理器（VMM）。

本作业分为两个部分：虚拟机（VM）代码和 VMM 代码。我们将使用一个非常简单的协议来实现这两个组件之间的通信。该协议名为 SIMVIRTIO。


I. 虚拟机管理器
==========================

通常情况下，要从零开始构建一个 VMM，我们需要实现三个主要功能：初始化 VMM、初始化虚拟 CPU 和运行客户机代码。我们将把 VMM 的实现分为这三个阶段。

1. 初始化 VMM
-------------------------

一个 VM 通常由三个元素表示，一个是用于与 KVM API 进行交互的文件描述符，一个是用于配置 VM 文件描述符（每个 VM 对应一个）（例如设置其内存），还有一个是指向 VM 内存的指针。我们为你提供了以下结构，用于在使用 VM 时进行参考。

.. code-block:: c

	typedef struct vm {
		int sys_fd;
		int fd;
		char *mem;
	} virtual_machine;


初始化 KVM 虚拟机的第一步是与 [KVM_API](https://www.kernel.org/doc/html/latest/virt/kvm/api.html) 进行交互。KVM API 通过 ``/dev/kvm`` 进行公开。我们需要使用 ioctl 调用来调用该 API。

下面的代码片段展示了如何调用 ``KVM_GET_API_VERSION`` 来获取 KVM API 版本。

.. code-block:: c

	int kvm_fd = open("/dev/kvm", O_RDWR);
	if (kvm_fd < 0) {
	    perror("open /dev/kvm");
	    exit(1);
	}

	int api_ver = ioctl(kvm_fd, KVM_GET_API_VERSION, 0);
	if (api_ver < 0) {
	    perror("KVM_GET_API_VERSION");
	    exit(1);
	}

现在让我们简要介绍 VMM 如何初始化 VM。以下只是最基本的步骤，实际上在 VM 初始化过程中，VMM 可能会执行许多其他操作。

1. 首先，使用 KVM_GET_API_VERSION 检查我们是否运行了预期的 KVM 版本，即 ``KVM_API_VERSION``。
2. 然后，使用 ``KVM_CREATE_VM`` 创建虚拟机。请注意，调用 ``KVM_CREATE_VM`` 会返回一个文件描述符。我们将在后续的设置阶段使用这个文件描述符。
3. （可选）在基于 Intel 的 CPU 上，我们需要调用 ``KVM_SET_TSS_ADDR``，并将地址设置为 ``0xfffbd000``。
4. 接下来，为虚拟机分配内存。我们需要使用 ``mmap`` 函数进行分配，调用时需要使用 ``PROT_WRITE``, ``MAP_PRIVATE``, ``MAP_ANONYMOUS`` 和 ``MAP_NORESERVE`` 参数。我们建议为虚拟机分配 0x100000 字节的内存。
5. 使用 ``madvise`` 函数将内存标记为 ``MADV_MERGEABLE``。
6. 最后，使用 ``KVM_SET_USER_MEMORY_REGION`` 将内存分配给虚拟机。

**请确保你理解在何时使用哪个文件描述符，在调用 KVM_CREATE_VM 时我们使用 KVM 文件描述符，但在与虚拟机交互（例如调用 KVM_SET_USER_MEMORY_REGION）时我们使用虚拟机的文件描述符**

简而言之，用于虚拟机初始化的 API 包括：

* KVM_GET_API_VERSION
* KVM_CREATE_VM
* KVM_SET_TSS_ADDR
* KVM_SET_USER_MEMORY_REGION。

2. 初始化虚拟 CPU
___________________________

我们需要虚拟 CPU（VCPU）来存储寄存器值。

.. code-block:: c

	typedef struct vcpu {
		int fd;
		struct kvm_run *kvm_run;
	} virtual_cpu;

要创建虚拟 CPU，我们需要执行以下操作：
1. 调用 ``KVM_CREATE_VCPU`` 创建虚拟 CPU。此调用将返回一个文件描述符。
2. 使用 ``KVM_GET_VCPU_MMAP_SIZE`` 获取共享内存的大小。
3. 使用 ``mmap`` 分配所需的 VCPU 内存大小。我们需要把 VCPU 文件描述符传递给 ``mmap`` 调用。我们可以将结果存储在 ``kvm_run`` 中。


简而言之，用于虚拟机的 API 包括：

* KVM_CREATE_VCPU
* KVM_GET_VCPU_MMAP_SIZE

**我们建议使用 2MB 页面以简化翻译过程。**

运行虚拟机
==============


设置实模式（real mode）
----------------------------

首先，CPU 会以保护模式启动。要想运行任何有意义的代码，我们需要把 CPU 切换到[实模式](https://wiki.osdev.org/Real_Mode)。为此，我们需要配置几个 CPU 寄存器。

1. 首先，我们需要使用 ``KVM_GET_SREGS`` 获取寄存器的值。我们可以使用 ``struct kvm_regs`` 结构来完成这个任务。
2. 我们需要将 ``cs.selector`` 和 ``cs.base`` 设置为 0。我们可以使用 ``KVM_SET_SREGS`` 来设置这些寄存器。
3. 接下来，我们需要通过 ``rflags`` 寄存器清除所有 ``FLAGS`` 位。我们需要将 ``rflags`` 设置为 2，因为第一位必须始终为 1。我们还需要将 ``RIP`` 寄存器设置为 0。

设置长模式（long mode）
-------------------------

实模式适用于非常简单的客户机，比如在 `guest_16_bits` 文件夹中的客户机。但是，大多数现代程序需要 64 位地址，因此我们需要切换到长模式。OSDev 上的以下文章提供了[设置长模式](https://wiki.osdev.org/Setting_Up_Long_Mode)所需的所有必要信息。

在 ``vcpu.h`` 中，你可以找到有用的宏，例如 CR0_PE、CR0_MP 以及 CR0_ET 等。

由于我们需要运行更复杂的程序，我们还需要为我们的程序创建一个小的堆栈 ``regs.rsp = 1 << 20;``。不要忘记设置 RIP 和 RFLAGS 寄存器。

运行
-------

在设置了实模式或长模式的 VCPU 之后，我们终于可以在虚拟机上运行代码了。

1. 我们将客户机代码复制到虚拟机内存中, `memcpy(vm->mem, guest_code, guest_code_size)`。客户机代码将在下面两个将讨论的变量中提供。
2. 在无限循环中，我们执行以下操作：
   * 调用 VCPU 文件描述符上的 ``KVM_RUN`` 来运行 VCPU。
   * 通过 VCPU 的共享内存，检查 ``exit_reason`` 参数，以查看客户机是否发出了任何请求：
   * 处理以下 VMEXIT: ``KVM_EXIT_MMIO``, ``KVM_EXIT_IO`` 以及 ``KVM_EXIT_HLT``。当 VM 写入 MMIO 地址时，会触发 ``KVM_EXIT_MMIO``。当 VM 调用 ``inb`` 或 ``outb`` 时，会调用 ``KVM_EXIT_IO``。当用户执行 ``hlt`` 指令时，会调用 ``KVM_EXIT_HLT``。

客户机代码
----------

正在运行的虚拟机也被称为客户机。我们将使用客户机来测试我们的实现。

1. 在实现 SIMVIRTIO 之前进行测试。客户机会在地址 400 和 RAX 寄存器中写入值 42。
2. 为了测试更复杂的实现，我们需要扩展上述程序，使用 `outb` 指令在端口 `0xE9` 上也写入“Hello, world!\n”。
3. 为了测试 `SIMVIRTIO` 的实现，我们需要

如何获取客户机代码？客户机代码在以下静态指针 guest16、guest16_end-guest16 中可用。链接器脚本会填充它们。


## SIMVIRTIO：
从客户机和 VMM 之间的通信中，我们将实现一种非常简单的协议称为 ``SIMVIRTIO``。它是现实世界中使用的名为 virtio 的真实协议的简化版本。

配置空间：

＋－－－－－－－＋－－－－－－－－－＋－－－－－－－－＋－－－－－－－－－＋－－－－－－－－－－＋－－－－－－－＋－－－－－－－＋
｜　u32　　　 　｜　u16　　　　　　｜　u8　　　　　　｜　u8　　　　　　　｜　u8　　　　　　　　｜　u8　　　　　｜　u8　　　　　｜
＋＝＝＝＝＝＝＝＋＝＝＝＝＝＝＝＝＝＋＝＝＝＝＝＝＝＝＋＝＝＝＝＝＝＝＝＝＋＝＝＝＝＝＝＝＝＝＝＋＝＝＝＝＝＝＝＋＝＝＝＝＝＝＝＋
｜　魔术值　　　｜　最大队列长度　　｜　设备状态　　　｜　驱动程序状态　　｜　队列选择器　　　　｜　Q0(TX)　CTL｜　Q1(RX)　CTL｜
｜　R　　　　 　｜　R　　　　 　　　｜　R　　　　 　　｜　R/W　　　　　　｜　R/W　　 　　　　　｜　R/W　　　　｜　R/w　　 　　｜
＋－－－－－－－＋－－－－－－－－－＋－－－－－－－－＋－－－－－－－－－＋－－－－－－－－－－＋－－－－－－－＋－－－－－－－＋


控制器队列
-----------------

对于 ``SIMVIRTIO`` 实现，我们为你提供了以下结构和方法。

.. code-block:: c

	typedef uint8_t q_elem_t;
	typedef struct queue_control {
	    // 指向‘buffer’中当前可用的头部/生产者索引的指针。
	    unsigned head;
	    // 指向消费者使用的‘buffer’中最后一个索引的指针。
	    unsigned tail;
	} queue_control_t;
	typedef struct simqueue {
	    // MMIO 队列控制。
	    volatile queue_control_t *q_ctrl;
	    // 队列缓冲区/数据的大小。
	    unsigned maxlen;
	    // 队列数据缓冲区。
	    q_elem_t *buffer;
	} simqueue_t;
	int circ_bbuf_push(simqueue_t *q, q_elem_t data)
	{
	}
	int circ_bbuf_pop(simqueue_t *q, q_elem_t *data)
	{
	}


设备结构
-----------------

.. code-block:: c

	#define MAGIC_VALUE 0x74726976
	#define DEVICE_RESET 0x0
	#define DEVICE_CONFIG 0x2
	#define DEVICE_READY 0x4
	#define DRIVER_ACK 0x0
	#define DRIVER 0x2
	#define DRIVER_OK 0x4
	#define DRIVER_RESET 0x8000
	typedef struct device {
	    uint32_t magic;
	    uint8_t device_status;
	    uint8_t driver_status;
	    uint8_t max_queue_len;
	} device_t;
	typedef struct device_table {
	    uint16_t count;
	    uint64_t device_addresses[10];
	 } device_table_t;
 

我们需要执行以下处理程序：
* MMIO (read/write) VMEXIT
* PIO (read/write) VMEXIT

使用骨架
==================

调试
=========


任务
=====
1. 30分 实现一个简单的 VMM，运行来自 `guest_16_bits` 的代码。在此任务中，我们需要以读模式运行 VCPU。
2. 20分 扩展前面的实现，以在实模式下运行 VCPU。我们需要运行 `guest_32_bits` 示例。
3. 30分 实现 `SIMVIRTIO` 协议。
4. 10分 实现池化而不是 VMEXIT。我们需要使用宏 `USE_POOLING` 来打开和关闭此选项。
5. 10分 添加性能分析代码。测量 VMM 触发的 VMEXIT 次数。

提交作业
------------------------

作业存档需要根据 `规则页面 <https://ocw.cs.pub.ro/courses/so2/reguli-notare#reguli_de_trimitere_a_temelor>`__ 上的规定，通过 **Moodle** 提交。


提示
----

为了增加获得最高分的机会，请阅读并遵循 Linux 内核编码风格，该风格在 `编码风格文档 <https://elixir.bootlin.com/linux/v4.19.19/source/Documentation/process/coding-style.rst>`__ 中有描述。

此外，使用以下静态分析工具来验证代码：

* checkpatch.pl

  .. code-block:: console

     $ linux/scripts/checkpatch.pl --no-tree --terse -f /path/to/your/file.c

* sparse

  .. code-block:: console

     $ sudo apt-get install sparse
     $ cd linux
     $ make C=2 /path/to/your/file.c

* cppcheck

  .. code-block:: console

     $ sudo apt-get install cppcheck
     $ cppcheck /path/to/your/file.c

扣分项
---------

关于作业扣分项的信息可以在 `常规说明页面<https://ocw.cs.pub.ro/courses/so2/teme/general>`__ 中找到。

在特殊情况下（作业通过测试但不符合要求），以及如果作业未通过所有测试，成绩可能会比上述提到的降低更多。

参考资料
------------------------------------

我们建议你在开始做作业之前阅读以下内容：
* [用几行代码创建 KVM 主机](https://zserge.com/posts/kvm/)


太长不看
------------

1. 虚拟机管理程序（VMM）创建并初始化虚拟机和虚拟 CPU。
2. 切换到实模式并运行 `guest_16_bits` 中的简单客户机代码。
3. 切换到长模式并运行 `guest_32_bits` 中更复杂的客户机代码。
4. 实现 SIMVIRTIO 协议。我们将在以下子任务中描述其行为。
5. 客户机在 TX 队列（队列 0）中写入 `R` 的 ASCII 码，这将导致 `VMEXIT`。
6. VMM 需要处理由前面在队列中的写入引起的 VMEXIT。当客户端接收到 `R` 字母时，它将启动设备的复位过程，并将设备状态设置为 `DEVICE_RESET`。
7. 在处理复位后，客户机必须将设备状态设置为 `DRIVER_ACK`。在此之后，客户机需要向 TX 队列中写入字母 `C`。
8. 在收到字母 `C` 时，VMM 需要初始化配置过程。它会将设备状态设置为 `DEVICE_CONFIG` 并在 device_table 中添加新条目。
9. 配置过程完成后，客户机会将驱动程序状态设置为 `DRIVER_OK`。
10. 接下来，VMM 会将设备状态设置为 `DEVICE_READY`。
11. 客户机需要在 TX 队列中写入“Ana are mere”并执行停机指令。
12. VMM 需要向 STDOUT 打印接收到的消息并执行停机请求。
13. 最后，VMM 需要验证地址 0x400 处和寄存器 RAX 中是否存储了值 42。
