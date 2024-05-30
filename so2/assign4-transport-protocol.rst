=====================================
作业 4——SO2 传输协议
=====================================

.. meta::
   :description: 这个实验文档介绍了操作系统 2 课程中的第 4 个实验任务，内容包括 I/O 访问和中断机制
   :keywords: SO2, 操作系统 2, I/O, 中断, Linux 内核

- 截止日期: :command:`星期一，2023 年 5 月 29 日，23:00`
- 这个作业可以由团队完成（最多 2 人）。只需其中一人提交作业，学生姓名应列在 README 文件中。

实现一个简单的数据报传输协议——STP (*SO2 传输协议)。

作业目标
=======================

* 了解 Linux 内核中的网络子系统运行原理
* 掌握在 Linux 中使用网络子系统基本结构的技能
* 通过在现有协议栈中实现一个协议，加深与通信和网络协议相关的概念

任务描述
=========

在 Linux 内核中实现一个名为 STP (*SO2 传输协议*) 的协议，该协议在网络和传输层使用数据报（它不面向连接，也不使用流控制元素）。

STP 协议在作用层面上是传输层协议（基于端口的多路复用），但在 `OSI 模型 <http://zh.wikipedia.org/zh-cn/OSI模型>`__ 的第 3 层（网络层）上运行，位于数据链路层之上。

STP 头部由 ``struct stp_header`` 结构定义：

.. code-block:: c

  struct stp_header {
          __be16 dst;
          __be16 src;
          __be16 len;
          __u8 flags;
          __u8 csum;
  };


其中：

  * ``len`` 是数据包的大小（包括头部）（以字节为单位）；
  * ``dst`` 和 ``src`` 分别是目标端口和源端口；
  * ``flags`` 包含各种标志，目前未使用（标记为 *reserved*）；
  * ``csum`` 是整个包（包括头部）的校验和；校验和通过对所有字节进行异或（XOR）计算得出。

使用该协议的套接字将使用 ``AF_STP`` 协议族。

该协议必须直接在以太网上工作。使用的端口范围为 ``1`` 到 ``65535``。端口 ``0`` 未使用。

STP 相关结构和宏的定义可以在 `作业支持头文件 <https://gitlab.cs.pub.ro/so2/4-stp/-/blob/master/src/stp.h>`__ 中找到。

实现细节
======================

内核模块需要命名为 **af_stp.ko**。

你必须定义一个类型为 `net_proto_family <http://elixir.free-electrons.com/linux/v5.10/source/include/linux/net.h#L211>`__ 的结构，该结构可创建 STP 套接字。新创建的套接字不与任何端口或接口相关联，无法接收/发送数据包。你必须使用特定于 STP 协议族的操作来初始化 `socket ops 字段 <http://elixir.free-electrons.com/linux/v5.10/source/include/linux/net.h#L125>`__。该字段指向结构 `proto_ops <http://elixir.free-electrons.com/linux/v5.10/source/include/linux/net.h#L139>`__，其中必须包含以下函数：

* ``release``：释放一个 STP 套接字
* ``bind``：将套接字与端口关联（可能还包括接口），以便接收/发送数据包：

  * 只能在一个端口上绑定套接字（不能在接口上绑定）
  * 与一个端口关联的套接字将能够接收发送到该端口的所有接口上的数据包（类似于仅与一个端口关联的 UDP 套接字）；这些套接字不能发送数据包，因为无法指定可以通过标准套接字 API 发送的接口
  * 两个套接字不能绑定到相同的端口——接口组合：

    * 如果已经有一个与端口和接口绑定的套接字，则第二个套接字不能绑定到相同的端口和相同的接口或相同端口接口不指定
    * 如果已经有一个绑定到端口但没有指定接口的套接字，则不能将第二个套接字绑定到相同的端口（无论是否指定接口）

  * 我们建议在绑定操作中使用哈希表而不是其他数据结构（如列表、数组）；在内核 `hashtable.h 头文件中 <http://elixir.free-electrons.com/linux/v4.9.11/source/include/linux/hashtable.h>`__ 中有一个哈希表实现

* ``connect``：将套接字与远程端口和硬件地址（MAC 地址）关联，从而可以发送/接收数据包：

  * 这使得套接字可以执行 ``send`` / ``recv`` 操作而不是 ``sendmsg`` / ``recvmsg`` 或 ``sendto`` / ``recvfrom``
  * 一旦连接到主机，套接字将只接受来自该主机的数据包
  * 一旦连接，套接字将无法断开连接

* ``sendmsg``, ``recvmsg``：在 STP 套接字上发送或接收数据报：

  * 对于 *接收* 部分，有关发送数据包的主机的元信息可以存储在 `sk_buff 中的 cb 字段 <http://elixir.free-electrons.com/linux/v5.10/source/include/linux/skbuff.h#L742>`__ 中

* ``poll``：必须使用默认函数 ``datagram_poll``
* 对于其他操作，必须使用内核中预定义的 stub 函数 (``sock_no_*``)。

.. code-block:: c

    static const struct proto_ops stp_ops = {
            .family = PF_STP,
            .owner = THIS_MODULE,
            .release = stp_release,
            .bind = stp_bind,
            .connect = stp_connect,
            .socketpair = sock_no_socketpair,
            .accept = sock_no_accept,
            .getname = sock_no_getname,
            .poll = datagram_poll,
            .ioctl = sock_no_ioctl,
            .listen = sock_no_listen,
            .shutdown = sock_no_shutdown,
            .setsockopt = sock_no_setsockopt,
            .getsockopt = sock_no_getsockopt,
            .sendmsg = stp_sendmsg,
            .recvmsg = stp_recvmsg,
            .mmap = sock_no_mmap,
            .sendpage = sock_no_sendpage,
    };

套接字操作使用一种称为 ``sockaddr_stp`` 的地址类型，这种类型在 `作业支持头文件 <https://github.com/linux-kernel-labs/linux/blob/master/tools/labs/templates/assignments/4-stp/stp.h>`__ 中定义。对于 *bind* 操作，只需要考虑套接字绑定的端口和接口索引。对于 *receive* 操作，只需要使用该结构中的 ``addr`` 和 ``port`` 字段填充发送数据包的主机的 MAC 地址以及发送数据包的端口。此外，在发送数据包时，目标主机将从此结构的 ``addr`` 和 ``port`` 字段获取。

你需要使用 `dev_add_pack <http://elixir.free-electrons.com/linux/v5.10/source/net/core/dev.c#L521>`__ 调用，注册一个 `packet_type <http://elixir.free-electrons.com/linux/v5.10/source/include/linux/netdevice.h#L2501>`__ 结构，来能够从网络层接收 STP 数据包。

该协议需要通过 *procfs* 文件系统提供一个接口，用于统计发送/接收的数据包。该文件必须命名为 ``/proc/net/stp_stats``，该名称由 `作业支持头文件 <https://gitlab.cs.pub.ro/so2/4-stp/-/blob/master/src/stp.h>`__ 中的 ``STP_PROC_FULL_FILENAME`` 宏指定。格式必须是简单表格类型，具有 ``2`` 行：第一行是表格的标题，第二行是对应列的统计数据。表格的列必须按顺序排列：

.. code::

    RxPkts HdrErr CsumErr NoSock NoBuffs TxPkts

其中：

* ``RxPkts`` ——接收到的数据包数量
* ``HdrErr`` ——接收到的带有头部错误的数据包数量（数据包过短或源/目标端口为 0）
* ``CsumErr`` ——接收到的有校验和错误的数据包数量
* ``NoSock`` ——未找到目标套接字的接收数据包数量
* ``NoBuffs`` ——接收队列已满而无法接收的数据包数量
* ``TxPkts`` ——发送的数据包数量

我们建议使用 `proc_create <http://elixir.free-electrons.com/linux/v5.10/source/include/linux/proc_fs.h#L108>`__ 和 `proc_remove <http://elixir.free-electrons.com/linux/v5.10/source/fs/proc/generic.c#L772>`__ 函数来创建或删除由 ``STP_PROC_FULL_FILENAME`` 指定的项。

示例协议实现
-------------------------------

关于协议实现的示例，我们推荐参考 `PF_PACKET <http://elixir.free-electrons.com/linux/v5.10/source/net/packet/af_packet.c>`__ 套接字的实现以及 `UDP 实现 <http://elixir.free-electrons.com/linux/v5.10/source/net/ipv4/udp.c>`__ 或 `IP 实现 <http://elixir.free-electrons.com/linux/v5.10/source/net/ipv4/af_inet.c>`__ 中的各种函数。

测试
=======

为了简化作业评估过程，同时减少提交作业的错误，作业评估将使用一个名为 `_checker` 的 `测试脚本 <https://gitlab.cs.pub.ro/so2/3-raid/-/blob/master/checker/4-stp-checker/_checker>`__ 进行自动评估。测试脚本假设内核模块命名为 `af_stp.ko`。

tcpdump
-------

你可以使用 ``tcpdump`` 实用程序来排查发送的数据包。测试使用回环接口；要跟踪发送的数据包，可以使用以下形式的命令行：

.. code:: console

    tcpdump -i lo -XX

你可以使用静态版本的 `tcpdump <http://elf.cs.pub.ro/so2/res/teme/tcpdump>`__。要将其添加到虚拟机的 ``PATH`` 环境变量中，请将该文件复制到 ``/linux/tools/labs/rootfs/bin``。如果该目录不存在，请创建该目录。记得给 ``tcpdump`` 文件赋予执行权限：

.. code:: console

    # 使用 ./local.sh docker interactive 连接到 docker
    cd /linux/tools/labs/rootfs/bin
    wget http://elf.cs.pub.ro/so2/res/teme/tcpdump
    chmod +x tcpdump

快速入门
==========

必须从 `src <https://gitlab.cs.pub.ro/so2/4-stp/-/tree/master/src>`__ 目录中找到的代码骨架开始实现作业。骨架中只有一个名为 `stp.h <https://gitlab.cs.pub.ro/so2/4-stp/-/blob/master/src/stp.h>`__ 的头文件。你需要提供其余的实现。你可以添加任意数量的 `*.c`` 源文件和额外的 `*.h`` 头文件。你还应该提供一个名为 `af_stp.ko` 的内核模块的 Kbuild 文件。请按照 `作业仓库 <https://gitlab.cs.pub.ro/so2/4-stp>`__ 的 `README.md 文件 <https://gitlab.cs.pub.ro/so2/4-stp/-/blob/master/README.md>`__ 中的说明进行操作。



提示
----

为了增加获得最高评分的机会，请阅读并遵循 `Linux 内核编码风格 <https://elixir.bootlin.com/linux/v5.10/source/Documentation/process/coding-style.rst>`__。

此外，使用以下静态分析工具验证代码：

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

有关作业扣分的信息可以在 `通用指导页面 <https://ocw.cs.pub.ro/courses/so2/teme/general>`__ 上找到。

在特殊情况下（作业通过测试，但不符合要求），以及如果作业未通过所有测试，分数可能会降低得更多。

提交作业
------------------------

作业将使用 `vmchecker-next <https://github.com/systems-cs-pub-ro/vmchecker-next/wiki/Student-Handbook>`__ 基础设施进行自动评分。提交将在 moodle 的 `课程页面 <https://curs.upb.ro/2022/course/view.php?id=5121>`__ 上与相关作业相关联。你可以在 `仓库 <https://gitlab.cs.pub.ro/so2/4-stp>`__ 的 `README.md 文件 <https://gitlab.cs.pub.ro/so2/4-stp/-/blob/master/README.md>`__ 中找到提交详细信息。


资源
=========

* `课程 10 ——网络 </so2/lec10-networking.html>`__
* `实验 10 ——网络 </so2/lab10-networking.html>`__
* Linux 内核源代码

  * `PF_PACKET 套接字实现 <http://elixir.free-electrons.com/linux/v5.10/source/net/packet/af_packet.c>`__
  * `UDP 协议实现 <http://elixir.free-electrons.com/linux/v5.10/source/net/ipv4/udp.c>`__
  * `IP 协议实现 <http://elixir.free-electrons.com/linux/v5.10/source/net/ipv4/af_inet.c>`__

* 理解 Linux 网络内部的工作原理

  * 第 8-13 章

* `作业支持头文件 <https://gitlab.cs.pub.ro/so2/4-stp/-/blob/master/src/stp.h>`__

我们建议你使用 GitLab 存储你的作业。请按照 `README <https://gitlab.cs.pub.ro/so2/4-stp/-/blob/master/README.md>`__ 中的说明进行操作。

问题
=========

如果有相关的问题，请查阅邮件 `列表归档 <http://cursuri.cs.pub.ro/pipermail/so2/>`__ 或在专用 Teams 频道上提问。

在提问之前，请确保：

   - 你已经仔细阅读了作业说明
   - 问题在 `FAQ 页面 <https://ocw.cs.pub.ro/courses/so2/teme/tema2/faq>`__ 上没有被提出
   - 答案无法在 `邮件列表归档 <http://cursuri.cs.pub.ro/pipermail/so2/>`__ 中找到
