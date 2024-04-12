===========================
作业 3——软件 RAID
===========================

- 截止日期： :command:`周一，2023 年 5 月 15 日，23:00`
- 本作业可以由团队（最多 2 人）完成。只需由其中一人提交作业，并在 README 文件中列出学生的姓名。

实现一个软件 RAID 模块，该模块使用逻辑块设备从两个物理设备读取和写入数据，确保两个物理设备的数据一致性和同步。所实现的 RAID 类型类似于 `RAID 1`。

作业目标
=======================

* 深入理解 I/O 子系统的工作原理。
* 获得使用 `bio` 结构的进阶技能。
* 在 Linux 内核中使用块/磁盘设备。
* 获得在 Linux 中导航和理解专用于 I/O 子系统的代码和 API 的技能。


题目描述
=========

编写一个内核模块，实现 RAID 软件功能。 `软件 RAID <https://zh.wikipedia.org/zh-cn/RAID#实现方式>`__ 在逻辑设备和物理设备之间提供了一个抽象层。实现请使用 `RAID 方案 1 <https://zh.wikipedia.org/zh-cn/RAID#标准RAID>`__。

虚拟机具有两个硬盘，它们代表物理设备: `/dev/vdb` 和 `/dev/vdc`。操作系统有一个逻辑设备（块类型），其提供接口供用户空间访问。将请求写入逻辑设备将导致两个写操作，分别写入每个硬盘。硬盘未分区。可以认为每个硬盘都整个由单个分区覆盖。

每个分区存储一个扇区以及关联的校验和（CRC32），以确保错误恢复。在每次读取时，会读取两个分区的相关信息。如果第一个分区的扇区存在损坏的数据（CRC 值错误），则将读取第二个分区的扇区；同时，将修复第一个分区的扇区。在读取第二个分区发现其扇区损坏时也是类似的。如果两个分区的扇区都具有不正确的 CRC 值，则返回适当的错误代码。

重要提示
-----------------

为了确保错误恢复，每个扇区都与一个 CRC 码相关联。CRC 码存储在分区的 `LOGICAL_DISK_SIZE` 字节之后（在作业 `头文件 <https://gitlab.cs.pub.ro/so2/3-raid/-/blob/master/src/ssr.h>`__ 中定义的宏）。磁盘结构的布局如下所示：

.. code-block:: console

   +-----------+-----------+-----------+     +---+---+---+
   |  sector1  |  sector2  |  sector3  |.....|C1 |C2 |C3 |
   +-----------+-----------+-----------+     +---+---+---+

其中, ``C1``, ``C2``, ``C3`` 是扇区 ``sector1``, ``sector2``, ``sector3`` 的 CRC 值。CRC 区域位于分区的 `LOGICAL_DISK_SIZE` 字节之后。

CRC 的种子使用 0（零）。

实现细节
======================

- 内核模块的名称为 ``ssr.ko``。
- 逻辑设备将被作为块设备访问（使用主设备号 ``SSR_MAJOR`` 和次设备号 ``SSR_FIRST_MINOR``，用户空间可通过访问名称为 ``/dev/ssr`` 的文件来访问该逻辑设备）（通过宏 ``LOGICAL_DISK_NAME``）。
- 虚拟设备 (``LOGICAL_DISK_NAME`` —— ``/dev/ssr``) 的容量为 ``LOGICAL_DISK_SECTORS``（使用 ``struct gendisk`` 结构的 ``set_capacity`` 函数）。
- 两个硬盘由设备 ``/dev/vdb`` 和 ``/dev/vdc`` 表示，分别通过宏 ``PHYSICAL_DISK1_NAME`` 和 ``PHYSICAL_DISK2_NAME`` 定义。
- 若要使用与物理设备关联的 ``struct block_device`` 结构进行操作，可以使用 ``blkdev_get_by_path`` 和 ``blkdev_put`` 函数。
- 对于来自用户空间的请求处理，不建议使用 ``request_queue``，而是在 :c:type:`struct bio` 层面进行处理（使用 :c:type:`struct block_device_operations` 的 ``submit_bio`` 字段）。
- 由于数据扇区与 CRC 扇区是分开的，你需要为数据和 CRC 值构建单独的 ``bio`` 结构。
- 要为物理硬盘分配 :c:type:`struct bio`，可以使用 :c:func:`bio_alloc`；要向 bio 添加数据页，请使用 :c:func:`alloc_page` 和 :c:func:`bio_add_page`。
- 要释放为 :c:type:`struct bio` 分配的空间，你需要释放分配给该 bio 的页面（使用 :c:func:`__free_page` 宏），并调用 :c:func:`bio_put`。
- 生成 :c:type:`struct bio` 结构时，请注意其大小必须是磁盘扇区大小 (``KERNEL_SECTOR_SIZE``) 的倍数。
- 要发送请求到块设备并等待其结束，可以使用 :c:func:`submit_bio_wait` 函数。
- 使用 :c:func:`bio_endio` 来表示完成处理 ``bio`` 结构。
- 对于 CRC32 计算，可以使用内核提供的 :c:func:`crc32` 宏。
- 作业支持 `头文件 <https://gitlab.cs.pub.ro/so2/3-raid/-/blob/master/src/ssr.h>`__ 中可以找到一些有用的宏定义。
- 在调用堆栈中，块设备的单个请求处理函数可同时激活，有关更多详细信息，请参阅 `此处 <https://elixir.bootlin.com/linux/v5.10/source/block/blk-core.c#L1048>`__。
  你需要在内核线程中提交对物理设备的请求；我们建议使用 ``workqueues``。
- 要想快速运行，使用一个单独的 bio 批量发送相邻扇区的 CRC 值的读/写请求。例如，如果你需要发送扇区 0、1、...、7 的 CRC 请求，则使用一个单独的 bio，而不是 8 个 bio。
- 我们的建议不是强制性的（只要满足作业要求，任何解决方案都被接受）。

测试
=======
为了简化作业评估过程，减少提交作业时的错误，作业评估将通过一个名为 `_checker` 的 `测试脚本 <https://gitlab.cs.pub.ro/so2/3-raid/-/blob/master/checker/3-raid-checker/_checker>`__ 自动进行。测试脚本假定内核模块的名称为 `ssr.ko`。

如果在测试过程中，两个磁盘上的扇区都包含无效数据，导致读取错误，使得无法使用模块，则需要使用以下命令在虚拟机中重新创建两个磁盘：

.. code-block:: console

   $ dd if=/dev/zero of=/dev/vdb bs=1M
   $ dd if=/dev/zero of=/dev/vdc bs=1M

你还可以使用以下命令启动虚拟机以获得相同的结果：

.. code-block:: console

   $ rm disk{1,2}.img; make console # 或者 rm disk{1,2}.img; make boot

快速入门
==========

必须从 `src 目录 <https://gitlab.cs.pub.ro/so2/3-raid/-/tree/master/src>`__ 中的代码骨架开始实现作业。框架中只有一个名为 `ssr.h <https://gitlab.cs.pub.ro/so2/3-raid/-/blob/master/src/ssr.h>`__ 的头文件。你需要提供其余的实现。你可以添加任意数量的 `*.c`` 源文件和额外的 `*.h`` 头文件。你还应该提供一个名为 `ssr.ko` 的内核模块的 Kbuild 文件。请按照 `作业仓库 <https://gitlab.cs.pub.ro/so2/3-raid>`__ 的 `README.md 文件 <https://gitlab.cs.pub.ro/so2/3-raid/-/blob/master/README.md>`__ 中的说明进行操作。


提示
----

为了增加获得最高分的机会，请阅读并遵循 Linux 内核的编码风格，该风格在 `编码风格文档 <https://elixir.bootlin.com/linux/v4.19.19/source/Documentation/process/coding-style.rst>`__ 中有描述。

此外，请使用以下静态分析工具来验证代码：

- checkpatch.pl

.. code-block:: console

   $ linux/scripts/checkpatch.pl --no-tree --terse -f /path/to/your/file.c

- sparse

.. code-block:: console

   $ sudo apt-get install sparse
   $ cd linux
   $ make C=2 /path/to/your/file.c

- cppcheck

.. code-block:: console

   $ sudo apt-get install cppcheck
   $ cppcheck /path/to/your/file.c

扣分项
---------

有关作业扣分的信息可以在 `综合说明页面 <https://ocw.cs.pub.ro/courses/so2/teme/general>`__ 上找到。

在特殊情况下（作业通过测试，但不符合要求），以及如果作业未通过所有测试，则可能会降低更多分数。

提交作业
------------------------

作业将使用 `vmchecker-next <https://github.com/systems-cs-pub-ro/vmchecker-next/wiki/Student-Handbook>`__ 基础设施进行自动评分。提交将在 moodle 的 `课程页面 <https://curs.upb.ro/2022/course/view.php?id=5121>`__ 上与相关作业相关联。你可以在 `仓库 <https://gitlab.cs.pub.ro/so2/3-raid>`__ 的 `README.md 文件 <https://gitlab.cs.pub.ro/so2/3-raid/-/blob/master/README.md>`__ 中找到提交的详细信息。


资源
=========

- Linux 内核中 RAID 软件的实现 `RAID <https://elixir.bootlin.com/linux/v5.10/source/drivers/md>`__

建议使用 GitLab 存储你的作业。请按照 `README <https://gitlab.cs.pub.ro/so2/3-raid/-/blob/master/README.md>`__ 中的说明进行操作。


问题
=========

如果有相关问题，你可以查阅 `邮件列表归档 <http://cursuri.cs.pub.ro/pipermail/so2/>`__，或在专用的 Teams 频道上提问。

在提问之前，请确保：

- 你已经仔细阅读了作业说明
- 问题在 `FAQ 页面 <https://ocw.cs.pub.ro/courses/so2/teme/tema2/faq>`__ 中没有被提及
- 答案在 `邮件列表归档 <http://cursuri.cs.pub.ro/pipermail/so2/>`__ 中无法找到
