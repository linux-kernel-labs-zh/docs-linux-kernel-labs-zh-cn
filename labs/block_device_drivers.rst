====================
块设备驱动程序
====================

.. meta::
   :description: 了解 Linux 中 I/O 子系统的行为，在块设备的结构和函数上进行实际操作，通过解决练习，掌握块设备的 API 使用基础技能

实验目标
==============

  * 了解 Linux 中 I/O 子系统的行为
  * 在块设备的结构和函数上进行实际操作
  * 通过解决练习，掌握块设备的 API 使用基础技能

概述
========

块设备以数据通过固定大小的块来组织为特点，可以进行随机访问。这类设备的例子包括硬盘驱动器、CD-ROM 驱动器、RAM 磁盘等。块设备的速度通常比字符设备的速度快得多，并且它们的性能也很重要。这就是为什么 Linux 内核对这两种类型的设备处理方式不同（它使用一个专门的 API）。

因此，与字符设备相比，使用块设备更加复杂。字符设备只有当前位置，而块设备必须能够移动到设备上的任何位置，以提供对数据的随机访问。为了简化对块设备的操作，Linux 内核提供了一整个子系统，称为块 I/O（或块层）子系统。

从内核的角度来看，最小的逻辑寻址单元是块。虽然物理设备可以按扇区级别寻址，但内核使用块执行所有磁盘操作。由于最小的物理寻址单元是扇区，块的大小必须是扇区大小的倍数。此外，块的大小必须是 2 的幂，并且不能超过页面大小。块的大小可能因使用的文件系统而异，最常见的值为 512 B、1 KB 和 4 KB。


注册块 I/O 设备
===========================

要注册块设备，请使用函数 :c:func:`register_blkdev`。要注销一个块设备，可以使用函数 :c:func:`unregister_blkdev`。

从 Linux 内核的 4.9 版本开始，调用 :c:func:`register_blkdev` 不再是必须操作。该函数执行的唯一操作是动态分配一个主设备号（如果调用函数时主设备号参数为 0），并在 :file:`/proc/devices` 中创建一个条目。在未来的内核版本中，它可能被移除；然而，大多数驱动程序仍然调用它。

通常，在模块初始化函数中调用注册函数，在模块退出函数中调用注销函数。典型的场景如下所示：


.. code-block:: c

   #include <linux/fs.h>

   #define MY_BLOCK_MAJOR           240
   #define MY_BLKDEV_NAME          "mybdev"

   static int my_block_init(void)
   {
       int status;

       status = register_blkdev(MY_BLOCK_MAJOR, MY_BLKDEV_NAME);
       if (status < 0) {
                printk(KERN_ERR "unable to register mybdev block device\n");
                return -EBUSY;
        }
        //...
   }

   static void my_block_exit(void)
   {
        //...
        unregister_blkdev(MY_BLOCK_MAJOR, MY_BLKDEV_NAME);
   }


注册磁盘
===============

尽管 :c:func:`register_blkdev` 函数获取了主设备号，但它并没有向系统提供设备（磁盘）。为了创建和使用块设备（磁盘），我们使用在 :file:`linux/genhd.h` 中定义的专门接口。

在 :file:`linux/genhd.h` 中定义的有用函数是用于注册/分配磁盘、将其添加到系统中以及注销/卸载磁盘的函数。

:c:func:`alloc_disk` 函数用于分配磁盘，:c:func:`del_gendisk` 函数用于释放磁盘。使用 :c:func:`add_disk` 函数将磁盘添加到系统中。

通常在模块初始化函数中使用 :c:func:`alloc_disk` 和 :c:func:`add_disk` 函数，而在模块退出函数中使用 :c:func:`del_gendisk` 函数。

.. code-block:: c

   #include <linux/fs.h>
   #include <linux/genhd.h>

   #define MY_BLOCK_MINORS	 1

   static struct my_block_dev {
       struct gendisk *gd;
       //...
   } dev;

   static int create_block_device(struct my_block_dev *dev)
   {
       dev->gd = alloc_disk(MY_BLOCK_MINORS);
       //...
       add_disk(dev->gd);
   }

   static int my_block_init(void)
   {
       //...
       create_block_device(&dev);
   }

   static void delete_block_device(struct my_block_dev *dev)
   {
       if (dev->gd)
           del_gendisk(dev->gd);
       //...
   }

   static void my_block_exit(void)
   {
       delete_block_device(&dev);
       //...
   }

与字符设备一样，建议使用 :c:type:`my_block_dev` 结构来存储描述块设备的重要元素。

请注意，在调用 :c:func:`add_disk` 函数之后（实际上，甚至包括调用期间），磁盘是活动的，可以随时调用其方法。因此，在驱动程序完全初始化并准备好响应对注册磁盘的请求之前，不应调用此函数。


可以注意到，用于处理块设备（磁盘）的基本结构是 :c:type:`struct gendisk` 结构。

在调用 :c:func:`del_gendisk` 函数后，如果仍然有用户（对设备调用了打开操作，但关联的释放操作尚未被调用），则 :c:type:`struct gendisk` 结构可能继续存在（并且设备操作仍然可以调用）。一种解决方法是记录设备的用户数，并仅在设备没有剩余用户后调用 :c:func:`del_gendisk` 函数。

:c:type:`struct gendisk` 结构体
==================================

:c:type:`struct gendisk` 结构体存储关于磁盘的信息。如上所述，这样的结构体是通过 :c:func:`alloc_disk` 调用获得的，在将其作为参数传入 :c:func:`add_disk` 函数之前，必须填充其字段。

:c:type:`struct gendisk` 结构体具有以下重要字段：

   * :c:member:`major`, :c:member:`first_minor` 以及 :c:member:`minor`：描述磁盘使用的标识符；磁盘必须至少有一个次设备号；如果磁盘允许分区操作，则必须为每个可能的分区分配一个次设备号
   * :c:member:`disk_name`：表示磁盘名称，如在 :file:`/proc/partitions` 和 sysfs (:file:`/sys/block`) 中显示
   * :c:member:`fops`：表示与磁盘关联的操作
   * :c:member:`queue`：表示请求队列
   * :c:member:`capacity`：表示磁盘容量（以 512 字节扇区为单位）；可以使用 :c:func:`set_capacity` 函数进行初始化
   * :c:member:`private_data`：指向私有数据的指针

下面是填充 :c:type:`struct gendisk` 结构体的示例：

.. code-block:: c

   #include <linux/genhd.h>
   #include <linux/fs.h>
   #include <linux/blkdev.h>
   
   #define NR_SECTORS              1024
   
   #define KERNEL_SECTOR_SIZE      512
   
   static struct my_block_dev {
       //...
       spinlock_t lock;                /* 互斥锁 */
       struct request_queue *queue;    /* 设备请求队列 */
       struct gendisk *gd;             /* gendisk 结构体 */
       //...
   } dev;

   static int create_block_device(struct my_block_dev *dev)
   {
       ...
       /* 初始化 gendisk 结构体 */
       dev->gd = alloc_disk(MY_BLOCK_MINORS);
       if (!dev->gd) {
           printk(KERN_NOTICE "alloc_disk failure\n");
           return -ENOMEM;
       }

       dev->gd->major = MY_BLOCK_MAJOR;
       dev->gd->first_minor = 0;
       dev->gd->fops = &my_block_ops;
       dev->gd->queue = dev->queue;
       dev->gd->private_data = dev;
       snprintf(dev->gd->disk_name, 32, "myblock");
       set_capacity(dev->gd, NR_SECTORS);
   
       add_disk(dev->gd);
   
       return 0;
   }

   static int my_block_init(void)
   {
       int status;
       //...
       status = create_block_device(&dev);
       if (status < 0)
           return status;
       //...
   }

   static void delete_block_device(struct my_block_dev *dev)
   {
       if (dev->gd) {
           del_gendisk(dev->gd);
       }
       //...
   }

   static void my_block_exit(void)
   {
       delete_block_device(&dev);
       //...
   }

如前所述，内核将磁盘视为一连串的 512 字节扇区。实际上，设备可能具有不同大小的扇区。为了与这些设备一起工作，内核需要了解实际扇区的大小，并且在所有操作中需要进行必要的转换。

要向内核通知设备的扇区大小，必须在分配请求队列后设置请求队列的参数，使用 :c:func:`blk_queue_logical_block_size` 函数完成设置。内核生成的所有请求都将是该扇区大小的倍数，并相应地对齐。但是，设备和驱动程序之间的通信仍将以 512 字节大小的扇区进行，因此每次都需要进行转换（上述代码中调用 :c:func:`set_capacity` 函数时就是一个例子）。

:c:type:`struct block_device_operations` 结构体
==================================================

就像对于字符设备，需要完成 :c:type:`struct file_operations` 中的操作一样，对于块设备，需要完成 :c:type:`struct block_device_operations` 中的操作。操作的关联是通过 :c:type:`struct gendisk` 结构体中的 :c:member:`fops` 字段完成的。

下面是 :c:type:`struct block_device_operations` 结构体的一些字段：

.. code-block:: c

   struct block_device_operations {
       int (*open) (struct block_device *, fmode_t);
       int (*release) (struct gendisk *, fmode_t);
       int (*locked_ioctl) (struct block_device *, fmode_t, unsigned,
                            unsigned long);
       int (*ioctl) (struct block_device *, fmode_t, unsigned, unsigned long);
       int (*compat_ioctl) (struct block_device *, fmode_t, unsigned,
                            unsigned long);
       int (*direct_access) (struct block_device *, sector_t,
                             void **, unsigned long *);
       int (*media_changed) (struct gendisk *);
       int (*revalidate_disk) (struct gendisk *);
       int (*getgeo)(struct block_device *, struct hd_geometry *);
       blk_qc_t (*submit_bio) (struct bio *bio);
       struct module *owner;
   }

:c:func:`open` 和 :c:func:`release` 操作可以直接由用户空间的程序调用，这些程序可能执行以下任务：分区、文件系统创建、文件系统验证。在 :c:func:`mount` 操作中，可以从内核空间直接调用 :c:func:`open` 函数，文件描述符由内核存储。块设备的驱动程序无法区分是从用户空间还是内核空间调用了 :c:func:`open` 函数。

下面是如何使用这两个函数的示例：

.. code-block:: c

   #include <linux/fs.h>
   #include <linux/genhd.h>

   static struct my_block_dev {
       //...
       struct gendisk * gd;
       //...
   } dev;

   static int my_block_open(struct block_device *bdev, fmode_t mode)
   {
       //...

       return 0;
   }

   static int my_block_release(struct gendisk *gd, fmode_t mode)
   {
       //...

       return 0;
   }

   struct block_device_operations my_block_ops = {
       .owner = THIS_MODULE,
       .open = my_block_open,
       .release = my_block_release
   };

   static int create_block_device(struct my_block_dev *dev)
   {
       //....
       dev->gd->fops = &my_block_ops;
       dev->gd->private_data = dev;
       //...
   }

请注意，没有读取或写入操作。这些操作是由与磁盘的请求队列相关联的 :c:func:`request` 函数执行的。

请求队列——多队列块层
=======================

块设备的驱动程序使用队列来存储将要处理的块输入/输出请求。请求队列由 :c:type:`struct request_queue` 结构体表示。请求队列由一系列请求及其关联的控制信息组成，这些请求通过双向链表链接在一起。请求通过更高层次的内核代码（例如文件系统）添加到队列中。

块设备驱动程序将每个队列与处理函数关联起来，对于某队列中的每个请求（通过 :c:type:`struct request` 结构体表示），都将调用该队列对应处理函数。

在早期的 Linux 内核版本中，每个设备驱动程序关联了一个或多个请求队列 (:c:type:`struct request_queue`)，任何客户端都可以向其添加请求，并能够对其进行重新排序。这种方法的问题在于每个队列都需要锁，在分布式系统中效率低下。

`多队列块队列机制 <https://www.kernel.org/doc/html/latest/block/blk-mq.html>`_ 通过将设备驱动程序队列分为两部分来解决了这个问题：
 1. 软件分段队列（software staging queue）
 2. 硬件调度队列（hardware dispatch queue）

软件分段队列
-----------------------

分段队列在将请求发送给块设备驱动程序之前，保存来自客户端的请求。为了避免每个队列都有一把锁，为每个 CPU 或节点分配一个分段队列。一个软件队列只与一个硬件队列关联。

在这个队列中，根据 I/O 调度程序，请求可以合并或重新排序，以最大化性能。这意味着只有来自相同 CPU 或节点的请求可以进行优化。

分段队列通常不被块设备驱动程序使用，而只在 I/O 子系统内部使用，以在将请求发送给设备驱动程序之前对其进行优化。

硬件调度队列
------------------------

硬件队列 (:c:type:`struct blk_mq_hw_ctx`) 用于将请求从分段队列发送到块设备驱动程序。一旦进入此队列，请求就无法合并或重新排序。

根据底层硬件的不同，块设备驱动程序可以创建多个硬件队列，以提高并行性和最大化性能。

标签集
--------

块设备驱动程序可以在前一个请求完成之前接受另一个请求。因此，上层需要一种方式来知道请求何时完成。为此，在提交时为每个请求添加一个“标签”，并在请求完成后使用完成通知将其发送回来。

这些标签是标签集 (:c:type:`struct blk_mq_tag_set`) 的一部分，每个设备的标签集都是唯一的。在分配和初始化请求队列之前，会分配和初始化标签集结构，并且还存储一些队列的属性。

.. code-block:: c

    struct blk_mq_tag_set {
      ...
      const struct blk_mq_ops   *ops;
      unsigned int               nr_hw_queues;
      unsigned int               queue_depth;
      unsigned int               cmd_size;
      int                        numa_node;
      void                      *driver_data;
      struct blk_mq_tags       **tags;
      struct list_head           tag_list;
      ...
    };

:c:type:`struct blk_mq_tag_set` 结构中的一些字段如下：

 * ``ops``——队列操作，特别是请求处理函数
 * ``nr_hw_queues``——为设备分配的硬件队列数量
 * ``queue_depth``——硬件队列的大小
 * ``cmd_size``——在设备末尾额外分配的字节数，供块设备驱动程序使用（如果需要）
 * ``numa_node``——在 NUMA 系统中，存储设备连接的节点的索引
 * ``driver_data``——驱动程序私有的数据（如果需要）
 * ``tags``——指向包含 ``nr_hw_queues`` 个标签集的数组的指针
 * ``tag_list``——使用该标签集的请求队列的列表

创建和删除请求队列
---------------------------------

我们使用 :c:func:`blk_mq_init_queue` 函数创建请求队列，使用 :c:func:`blk_cleanup_queue` 函数删除请求队列。第一个函数同时创建硬件队列和软件队列，并初始化它们的结构。

队列属性，包括硬件队列的数量、容量和请求处理函数，使用上述所述的 :c:type:`blk_mq_tag_set` 结构进行配置。

以下是使用这些函数的示例：

.. code-block:: c

   #include <linux/fs.h>
   #include <linux/genhd.h>
   #include <linux/blkdev.h>

   static struct my_block_dev {
       //...
       struct blk_mq_tag_set tag_set;
       struct request_queue *queue;
       //...
   } dev;

   static blk_status_t my_block_request(struct blk_mq_hw_ctx *hctx,
                                        const struct blk_mq_queue_data *bd)
   //...

   static struct blk_mq_ops my_queue_ops = {
      .queue_rq = my_block_request,
   };

   static int create_block_device(struct my_block_dev *dev)
   {
       /* 初始化标签集 */
       dev->tag_set.ops = &my_queue_ops;
       dev->tag_set.nr_hw_queues = 1;
       dev->tag_set.queue_depth = 128;
       dev->tag_set.numa_node = NUMA_NO_NODE;
       dev->tag_set.cmd_size = 0;
       dev->tag_set.flags = BLK_MQ_F_SHOULD_MERGE;
       err = blk_mq_alloc_tag_set(&dev->tag_set);
       if (err) {
           goto out_err;
       }

       /* 分配队列 */
       dev->queue = blk_mq_init_queue(&dev->tag_set);
       if (IS_ERR(dev->queue)) {
           goto out_blk_init;
       }

       blk_queue_logical_block_size(dev->queue, KERNEL_SECTOR_SIZE);

        /* 为队列结构分配私有数据。 */
       dev->queue->queuedata = dev;
       //...

   out_blk_init:
       blk_mq_free_tag_set(&dev->tag_set);
   out_err:
       return -ENOMEM;
   }

   static int my_block_init(void)
   {
       int status;
       //...
       status = create_block_device(&dev);
       if (status < 0)
           return status;
       //...
   }

   static void delete_block_device(struct block_dev *dev)
   {
       //...
       blk_cleanup_queue(dev->queue);
       blk_mq_free_tag_set(&dev->tag_set);
   }

   static void my_block_exit(void)
   {
       delete_block_device(&dev);
       //...
   }

在初始化标签集结构后，使用 :c:func:`blk_mq_alloc_tag_set` 函数分配标签列表。将处理请求的函数的指针（:c:func:`my_block_request`）填充到 ``my_queue_ops`` 结构中，然后将该结构的指针添加到标签集中。

基于添加到标签集中的信息，使用 :c:func:`blk_mq_init_queue` 函数创建队列。

作为请求队列初始化的一部分，你可以配置 :c:member:`queuedata` 字段，该字段相当于其他结构中的 :c:member:`private_data` 字段。

用于处理请求队列的有用函数
----------------------------------------------

:c:type:`struct blk_mq_ops` 中的 ``queue_rq`` 函数用于处理对块设备的请求。该函数相当于在字符设备中遇到的读取和写入函数。该函数接收对设备的请求作为参数，并可以使用各种函数来处理这些请求。

下面描述了在处理程序中用于处理请求的函数：

   * :c:func:`blk_mq_start_request` ——在开始处理请求之前必须调用；
   * :c:func:`blk_mq_requeue_request` ——重新发送队列中的请求；
   * :c:func:`blk_mq_end_request` ——结束请求处理并通知上层。

块设备的请求
==========================

块设备的请求由 :c:type:`struct request` 结构描述。

:c:type:`struct request` 结构的字段包括：

   * :c:member:`cmd_flags`：一系列标志，包括方向（读取或写入）；要确定方向，使用宏定义 :c:macro:`rq_data_dir`，如果是读取请求它返回 0，如果是写入请求则返回 1；
   * :c:member:`__sector`：传输请求的第一个扇区；如果设备扇区大小不同，应进行适当的转换。要访问此字段，请使用宏 :c:macro:`blk_rq_pos`；
   * :c:member:`__data_len`：要传输的总字节数；要访问此字段，请使用宏 :c:macro:`blk_rq_bytes`；
   * 通常，将传输当前 :c:type:`struct bio` 中的数据；可以使用宏 :c:macro:`blk_rq_cur_bytes` 来获取数据大小；
   * :c:member:`bio`：动态列表，其中包含一组与请求相关联的 :c:type:`struct bio` 结构，它们是与请求关联的缓冲区集合；如果存在多个缓冲区，则使用宏定义 :c:macro:`rq_for_each_segment` 访问该字段，如果只有一个关联的缓冲区，则使用 :c:macro:`bio_data` 宏定义访问该字段；

我们将在 :ref:`bio_structure` 部分中对 :c:type:`struct bio` 结构及其相关操作进行更详细的讨论。

创建请求
----------------

读取/写入请求是由位于内核 I/O 子系统上方的代码层创建的。通常，为块设备创建请求的子系统是文件管理子系统。I/O 子系统充当文件管理子系统和块设备驱动程序之间的接口。I/O 子系统的主要责任是将请求添加到特定块设备的队列中，并根据性能考虑对请求进行排序和合并。

处理请求
-----------------

块设备驱动程序的核心部分是请求处理函数 (``queue_rq``)。在前面的示例中，扮演这个角色的函数是 :c:func:`my_block_request`。如在 `创建和删除请求队列`_ 部分所述，该函数在创建标签集结构时与驱动程序相关联。

当内核认为驱动程序应该处理 I/O 请求时，将调用该函数。该函数必须开始处理队列中的请求，但不必完成它们，因为请求可能由驱动程序的其他部分完成。

请求函数在原子上下文中运行，并且必须遵循原子代码的规则（不能调用可能导致睡眠的函数等）。

调用处理请求的函数与任何用户空间进程的操作是异步的，并且不应对运行相应函数的进程作出任何假设。此外，不应假设请求提供的缓冲区是来自内核空间还是用户空间，任何访问用户空间的操作都是错误的。

下面是一个简单的请求处理函数示例：

.. code-block:: c

    static blk_status_t my_block_request(struct blk_mq_hw_ctx *hctx,
                                         const struct blk_mq_queue_data *bd)
    {
        struct request *rq = bd->rq;
        struct my_block_dev *dev = q->queuedata;

        blk_mq_start_request(rq);

        if (blk_rq_is_passthrough(rq)) {
            printk (KERN_NOTICE "Skip non-fs request\n");
            blk_mq_end_request(rq, BLK_STS_IOERR);
            goto out;
        }

        /* 做任务 */
        ...

        blk_mq_end_request(rq, BLK_STS_OK);

    out:
        return BLK_STS_OK;
    }

函数 :c:func:`my_block_request` 执行以下操作：

   * 从 ``bd`` 实参获取指向请求结构的指针，并使用 :c:func:`blk_mq_start_request` 函数开始处理请求。
   * 块设备可能会接收不传输数据块的调用（例如，对磁盘的低级操作，涉及特殊的设备访问方式的指令）。大多数驱动程序不知道如何处理这些请求，并返回错误。
   * 调用 :c:func:`blk_mq_end_request` 函数返回错误，第二个参数为 ``BLK_STS_IOERR``。
   * 根据关联设备的需求处理请求。
   * 请求结束。在这种情况下，调用 :c:func:`blk_mq_end_request` 函数以完成请求。

.. bio_structure:

:c:type:`struct bio` 结构
==============================

每个 :c:type:`struct request` 结构是一个 I/O 块请求，但可能来自于更高级别的多个独立请求的组合。要传输的扇区可以分散在主存中，但它们总是对应于设备上的一组连续扇区。请求被表示为一系列段，每个段对应于内存中的一个缓冲区。内核可以合并引用相邻扇区的请求，但不会将读取请求与写入请求合并到一个单独的 :c:type:`struct request` 结构中。

:c:type:`struct request` 结构的底层实现为 :c:type:`struct bio` 结构组成的链表，同时包含一些信息，这些信息使驱动程序在处理请求时保留其当前位置。

:c:type:`struct bio` 结构是块 I/O 请求的某个部分的低级表示。

.. code-block:: c

   struct bio {
       //...
       struct gendisk          *bi_disk;
       unsigned int            bi_opf;         /* 低位是请求标志位，高位是 REQ_OP。使用访问器。 */
       //...
       struct bio_vec          *bi_io_vec;     /* 实际向量列表 */
       //...
       struct bvec_iter        bi_iter;
       /...
       void                    *bi_private;
       //...
   };

反过来，:c:type:`struct bio` 结构包含 :c:type:`struct bio_vec` 结构的 :c:member:`bi_io_vec` 向量。它由要传输的物理内存中的单个页面，页面内的偏移和缓冲区的大小组成。要遍历 :c:type:`struct bio` 结构，需要遍历 :c:type:`struct bio_vec` 向量，并从每个物理页面传输数据。为了简化向量遍历，请使用 :c:type:`struct bvec_iter` 结构。该结构保持有关在遍历过程中使用了多少个缓冲区和扇区的信息。请求类型被编码在 :c:member:`bi_opf` 字段中；要确定请求类型，请使用 :c:func:`bio_data_dir` 函数。

创建 :c:type:`struct bio` 结构
---------------------------------------

可以使用两个函数来创建 :c:type:`struct bio` 结构：

   * :c:func:`bio_alloc`：为新结构分配空间；结构必须进行初始化；
   * :c:func:`bio_clone`：复制现有的 :c:type:`struct bio` 结构；新获得的结构将使用克隆结构字段的值进行初始化；缓冲区与已克隆的 :c:type:`struct bio` 结构共享，因此必须谨慎访问缓冲区，以避免两个克隆体访问同一内存区域；

这两个函数都返回新的 :c:type:`struct bio` 结构。

提交 :c:type:`struct bio` 结构
---------------------------------------

通常，:c:type:`struct bio` 结构是由内核的更高层级（通常是文件系统）创建的。因此，创建的结构随后会传递给 I/O 子系统，该子系统将多个 :c:type:`struct bio` 结构聚合成一个请求。

要将 :c:type:`struct bio` 结构提交给关联的 I/O 设备驱动程序，可以使用 :c:func:`submit_bio` 函数。该函数接收已初始化的 :c:type:`struct bio` 结构作为实参，该结构将被添加到 I/O 设备的请求队列中的一个请求中。从该队列中，可以使用专门的函数由 I/O 设备驱动程序处理该请求。


.. _bio_completion:

等待 :c:type:`struct bio` 结构的完成
-----------------------------------------------------------

将 :c:type:`struct bio` 结构提交给驱动程序会将其添加到请求队列中的一个请求中，然后进一步进行处理。因此，当 :c:func:`submit_bio` 函数返回时，并不能保证结构的处理已经完成。如果希望等待请求的处理完成，可以使用 :c:func:`submit_bio_wait` 函数。

要在对 :c:type:`struct bio` 结构的处理结束时得到通知（当未使用 :c:func:`submit_bio_wait` 函数时），应使用结构的 :c:member:`bi_end_io` 字段。该字段指定在 :c:type:`struct bio` 结构处理结束时将调用的函数。可以使用结构的 :c:member:`bi_private` 字段将信息传递给该函数。

初始化 :c:type:`struct bio` 结构
-------------------------------------------

一旦分配了 :c:type:`struct bio` 结构，在传输之前，必须对其进行初始化。

初始化结构涉及填充其重要字段。如上所述，:c:member:`bi_end_io` 字段用于指定在结构处理完成时调用的函数。:c:member:`bi_private` 字段用于存储可以在 :c:member:`bi_end_io` 指向的函数中访问的有用数据。

:c:member:`bi_opf` 字段指定操作的类型。

.. code-block:: c

   struct bio *bio = bio_alloc(GFP_NOIO, 1);
   //...
   bio->bi_disk = bdev->bd_disk;
   bio->bi_iter.bi_sector = sector;
   bio->bi_opf = REQ_OP_READ;
   bio_add_page(bio, page, size, offset);
   //...

在上面的代码片段中，我们指定了块设备以及发送给块设备的以下内容：:c:type:`struct bio` 结构、起始扇区、操作 (:c:data:`REQ_OP_READ` 或 :c:data:`REQ_OP_WRITE`) 和内容。:c:type:`struct bio` 结构的内容是由一个物理页面、页面中的偏移量和缓冲区大小描述的缓冲区。可以使用 :c:func:`alloc_page` 调用来分配页面。

.. note:: :c:func:`bio_add_page` 调用中的 :c:data:`size` 字段必须是设备扇区大小的倍数。

.. _bio_content:

如何使用 :c:type:`struct bio` 结构的内容
----------------------------------------------------------

要使用 :c:type:`struct bio` 结构的内容，必须将结构的支持页面映射到内核地址空间，从那里可以访问它们。要进行映射/取消映射，可以使用 :c:macro:`kmap_atomic` 和 :c:macro:`kunmap_atomic` 宏。

以下是一个典型的使用示例：

.. code-block:: c

   static void my_block_transfer(struct my_block_dev *dev, size_t start,
                                 size_t len, char *buffer, int dir);


   static int my_xfer_bio(struct my_block_dev *dev, struct bio *bio)
   {
       struct bio_vec bvec;
       struct bvec_iter i;
       int dir = bio_data_dir(bio);

       /* 独立完成每个段 */
       bio_for_each_segment(bvec, bio, i) {
           sector_t sector = i.bi_sector;
           char *buffer = kmap_atomic(bvec.bv_page);
           unsigned long offset = bvec.bv_offset;
           size_t len = bvec.bv_len;

           /* 处理映射后的缓冲 */
           my_block_transfer(dev, sector, len, buffer + offset, dir);

           kunmap_atomic(buffer);
       }

       return 0;
   }

如上面的示例所示，遍历 :c:type:`struct bio` 需要遍历其所有的段（segment）。每个段 (:c:type:`struct bio_vec`) 由物理地址页面、页面中的偏移量和大小定义。

为了简化对 :c:type:`struct bio` 的处理，可以使用 :c:macro:`bio_for_each_segment` 宏定义。它将遍历所有的段，并更新存储在迭代器（:c:type:`struct bvec_iter`）中的全局信息，例如当前扇区以及其他内部信息（段向量索引，剩余待处理的字节数等）。

你可以在映射的缓冲区中存储信息或提取信息。

如果使用请求队列并且需要在 :c:type:`struct bio` 级别处理请求，则应使用 :c:macro:`rq_for_each_segment` 宏而不是 :c:macro:`bio_for_each_segment` 宏。该宏遍历 :c:type:`struct request` 结构中的每个 :c:type:`struct bio` 结构的每个段，并更新 :c:type:`struct req_iterator` 结构。:c:type:`struct req_iterator` 包含当前的 :c:type:`struct bio` 结构和遍历其段的迭代器。

以下是一个典型的使用示例：

.. code-block:: c

   struct bio_vec bvec;
   struct req_iterator iter;

   rq_for_each_segment(bvec, req, iter) {
       sector_t sector = iter.iter.bi_sector;
       char *buffer = kmap_atomic(bvec.bv_page);
       unsigned long offset = bvec.bv_offset;
       size_t len = bvec.bv_len;
       int dir = bio_data_dir(iter.bio);

       my_block_transfer(dev, sector, len, buffer + offset, dir);

       kunmap_atomic(buffer);
   }

释放 :c:type:`struct bio` 结构
-------------------------------------

一旦内核子系统使用了 :c:type:`struct bio` 结构，就需要释放对它的引用。这可以通过调用 :c:func:`bio_put` 函数来实现。

在 :c:type:`struct bio` 级别设置请求队列
----------------------------------------------------

我们之前已经介绍了如何指定一个函数来处理发送给驱动程序的请求。该函数接收请求作为实参，并在 :c:type:`struct request` 级别进行处理。

如果要获得更高的灵活性，我们需要指定在 :c:type:`struct bio` 结构级别进行处理的函数，而不再使用请求队列，那么我们将需要在与驱动程序关联的 :c:type:`struct block_device_operations` 结构中填充 ``submit_bio`` 字段。

下面是一个典型的初始化在 :c:type:`struct bio` 结构级别执行处理的函数的示例：

.. code-block:: c

    // 执行处理的函数的声明
    // :c:type:`struct bio` 结构级别
    static blk_qc_t my_submit_bio(struct bio *bio);

    struct block_device_operations my_block_ops = {
       .owner = THIS_MODULE,
       .submit_bio = my_submit_bio
       ...
    };

进一步阅读
===============

* `Linux 设备驱动程序第 3 版，第 16 章 块设备驱动程序 <http://static.lwn.net/images/pdf/LDD3/ch16.pdf>`_
* Linux 内核开发第 2 版——第 13 章 块 I/O 层
* `一个简单的块设备驱动程序 <https://lwn.net/Articles/58719/>`_
* `gendisk 接口 <https://lwn.net/Articles/25711/>`_
* `bio 结构 <https://lwn.net/Articles/26404/>`_
* `请求队列 <https://lwn.net/Articles/27055/>`_
* `Documentation/block/request.txt——请求结构文档 <https://elixir.bootlin.com/linux/v4.15/source/Documentation/block/request.txt>`_
* `Documentation/block/biodoc.txt——通用块层注释 <https://elixir.bootlin.com/linux/v4.15/source/Documentation/block/biodoc.txt>`_
* `drivers/block/brd/c——基于 RAM 的块磁盘驱动程序 <https://elixir.bootlin.com/linux/v4.15/source/drivers/block/brd.c>`_
* `I/O 调度器 <https://www.linuxjournal.com/article/6931>`_


练习
=========

.. include:: ../labs/exercises-summary.hrst
.. |LAB_NAME| replace:: block_device_drivers

0. 简介
--------

使用 |LXR|_ 在 Linux 内核中查找以下符号的定义：

   * :c:type:`struct bio`
   * :c:type:`struct bio_vec`
   * :c:macro:`bio_for_each_segment`
   * :c:type:`struct gendisk`
   * :c:type:`struct block_device_operations`
   * :c:type:`struct request`

1. 块设备
---------------

创建一个内核模块，允许你注册或取消注册块设备。从实验骨架的 :file:`1-2-3-6-ram-disk/kernel` 目录中的文件开始。

按照实验骨架中标记为 **TODO 1** 的注释进行操作。使用现有的宏定义 (:c:macro:`MY_BLOCK_MAJOR`, :c:macro:`MY_BLKDEV_NAME`)。检查注册函数返回的值，在出现错误的情况下返回错误代码。

编译模块，将其复制到虚拟机并插入内核。验证你的设备是否成功创建在 :file:`/proc/devices` 内。你将看到一个主设备号为 240 的设备。

卸载内核模块，并检查设备是否已注销。

.. hint:: 查看 `注册块 I/O 设备`_ 部分。

将 :c:macro:`MY_BLOCK_MAJOR` 的值更改为 7。编译模块，将其复制到虚拟机并插入内核。注意到插入失败，因为已经有另一个驱动程序/设备在内核中注册了主设备号 7。

将 :c:macro:`MY_BLOCK_MAJOR` 宏的值恢复为 240。

2. 磁盘注册
--------------------

修改前面的模块以添加与驱动程序关联的磁盘。分析宏定义、:c:type:`my_block_dev` 结构以及 :file:`ram-disk.c` 文件中的现有函数。

按照标记为 **TODO 2** 的注释进行操作。使用 :c:func:`create_block_device` 和 :c:func:`delete_block_device` 函数。

.. hint:: 查看 `注册磁盘`_ 和 `处理请求`_ 部分。

填充 :c:func:`my_block_request` 函数以处理请求，但实际上不处理你的请求：显示“request received”消息以及以下信息：来自当前 :c:type:`struct bio` 结构的起始扇区、总大小、数据大小和方向。要验证请求类型，请使用 :c:func:`blk_rq_is_passthrough`（该函数在请求由文件系统生成的情况下返回 0）。

.. hint:: 要找到所需的信息，查看 `块设备的请求`_ 部分。

使用 :c:func:`blk_mq_end_request` 函数来完成请求的处理。

将模块插入内核，并检查模块打印的消息。当添加设备时，会向设备发送一个请求。检查 :file:`/dev/myblock` 是否存在，如果不存在，使用以下命令创建设备：

.. code-block:: shell

   mknod /dev/myblock b 240 0

要生成写入请求，请使用以下命令：

.. code-block:: shell

   echo "abc"> /dev/myblock

注意，写入请求之前会有一个读取请求。该请求用于从磁盘中读取块并使用用户提供的数据“更新”其内容，而不会覆盖其他部分。在读取和更新之后，写入操作发生。

3. RAM 磁盘
-----------

修改前面的模块以创建一个 RAM 磁盘：对设备的请求将导致在一个内存区域中进行读写操作。

内存区域 :c:data:`dev->data` 已经在模块的源代码中使用 :c:func:`vmalloc` 进行了分配，并使用 :c:func:`vfree` 进行了释放。

.. note:: 查看 `处理请求`_ 部分。

按照标记为 **TODO 3** 的注释完成 :c:func:`my_block_transfer` 函数，将请求信息写入内存区域/从内存区域中读取。该函数将在队列处理函数 :c:func:`my_block_request` 中为每个请求调用。要写入/读取内存区域，请使用 :c:func:`memcpy`。要确定写入/读取的信息，请使用 :c:type:`struct request` 结构的字段。

.. hint:: 要了解请求数据的大小，请使用 :c:macro:`blk_rq_cur_bytes` 宏。不要使用 :c:macro:`blk_rq_bytes` 宏。

.. hint:: 要找到与请求相关联的缓冲区，请使用 :c:data:`bio_data`(:c:data:`rq->bio`)。

.. hint:: 有关有用的宏的描述，请参阅 `块设备的请求`_ 部分。

.. hint:: 你可以在 `Linux 设备驱动程序 <http://lwn.net/Kernel/LDD3/>`_ 的 `块设备驱动程序示例 <https://github.com/martinezjavier/ldd3/blob/master/sbull/sbull.c>`_ 中找到有用的信息。

为了进行测试，使用测试文件 :file:`user/ram-disk-test.c`。测试程序在 ``make build`` 编译时会自动编译，之后使用 ``make copy`` 复制到虚拟机，可以在 QEMU 虚拟机上使用以下命令运行：

.. code-block:: shell

   ./ram-disk-test

无需将模块插入内核，它将由 ``ram-disk-test`` 命令插入。

由于传输数据缺乏同步（刷新），一些测试可能会失败。

4. 从磁盘读取数据
--------------------------

本练习的目的是从内核直接读取 :c:macro:`PHYSICAL_DISK_NAME` 磁盘（:file:`/dev/vdb`）的数据。

.. attention:: 在解决此练习之前，我们需要确保将磁盘添加到虚拟机中。

               检查 :file:`qemu/Makefile` 中的变量 ``QEMU_OPTS``。应该已经使用 ``-drive ...`` 添加了两个额外的磁盘。

               如果没有，请使用以下命令生成我们将用作磁盘镜像的文件：:command:`dd if=/dev/zero of=qemu/mydisk.img bs=1024 count=1` 并将以下选项添加到 :file:`qemu/Makefile`（在 :c:data:`QEMU_OPTS` 变量之中，root 盘之后）：
               :command:`-drive file=qemu/mydisk.img,if=virtio,format=raw`

按照目录 :file:`4-5-relay/` 中标记为 **TODO 4** 的注释，实现 :c:func:`open_disk` 和 :c:func:`close_disk` 函数。使用 :c:func:`blkdev_get_by_path` 和 :c:func:`blkdev_put` 函数。设备必须以独占的读写模式打开 (:c:macro:`FMODE_READ` | :c:macro:`FMODE_WRITE` | :c:macro:`FMODE_EXCL`)，并且当前模块必须作为 holder（:c:macro:`THIS_MODULE`）。

实现 :c:func:`send_test_bio` 函数。你将需要创建新的 :c:type:`struct bio` 结构并填充它，然后提交它并等待它。读取磁盘的第一个扇区。要等待，请调用 :c:func:`submit_bio_wait` 函数。

.. hint:: 磁盘的第一个扇区是索引为 0 的扇区。这个值必须用于初始化 :c:type:`struct bio` 的 :c:member:`bi_iter.bi_sector` 字段。

          对于读操作，使用 :c:macro:`REQ_OP_READ` 宏来初始化 :c:type:`struct bio` 的 :c:member:`bi_opf` 字段。

操作完成后，显示 :c:type:`struct bio` 结构读取的前 3 个字节数据。使用 ``"% 02x"`` 格式作为 :c:func:`printk` 的参数来显示数据以及 :c:macro:`kmap_atomic` 和 :c:macro:`kunmap_atomic` 宏。

.. hint:: 对于 :c:func:`kmap_atomic` 函数的实参，只需使用代码中分配的页面，即 :c:data:`page` 变量。

.. hint:: 请查看 :ref:`bio_content` 和 :ref:`bio_completion` 部分。

为了进行测试，使用 :file:`test-relay-disk` 脚本，在运行 :command:`make copy` 时会将其复制到虚拟机中。如果未复制，请确保该脚本可执行：

.. code-block:: shell

   chmod +x test-relay-disk

无需将模块加载到内核中，它将由 :command:`test-relay-disk` 加载。

使用以下命令运行脚本：

.. code-block:: shell

   ./test-relay-disk

脚本会将“abc”写入到 :c:macro:`PHYSICAL_DISK_NAME` 指定的磁盘的开头。运行后，模块将显示 61 62 63（字母“a”、“b”和“c”的十六进制值）。

5. 将数据写入磁盘
------------------

按照标有 **TODO 5** 的注释，在磁盘上写入消息（:c:macro:`BIO_WRITE_MESSAGE`）。

函数 :c:func:`send_test_bio` 接收操作类型（读取或写入）作为实参。在函数 :c:func:`relay_init` 中调用读取函数，在函数 :c:func:`relay_exit` 中调用写入函数。建议使用 :c:macro:`REQ_OP_READ` 和 :c:macro:`REQ_OP_WRITE` 宏。

在 :c:func:`send_test_bio` 函数中，如果操作是写入，使用消息 :c:macro:`BIO_WRITE_MESSAGE` 填充与 :c:type:`struct bio` 结构相关联的缓冲区。使用 :c:macro:`kmap_atomic` 和 :c:macro:`kunmap_atomic` 宏来处理与 :c:type:`struct bio` 结构相关联的缓冲区。

.. hint:: 需要通过相应地设置 :c:member:`bi_opf` 字段来更新与 :c:type:`struct bio` 结构相关联的操作类型。

为了测试，请使用以下命令运行 :file:`test-relay-disk` 脚本：

.. code-block:: shell

   ./test-relay-disk

该脚本将在标准输出中显示 ``"read from /dev/sdb: 64 65 66"`` 消息。

6. 在 :c:type:`struct bio` 级别处理请求队列中的请求
------------------------------------------------

在练习 3 的实现中，我们只处理了请求的当前 :c:type:`struct bio` 的 :c:type:`struct bio_vec`。我们希望处理来自请求队列中所有 :c:type:`struct bio` 结构的所有 :c:type:`struct bio_vec` 结构（也称为段）。

在 ramdisk 的实现（:file:`1-2-3-6-ram-disk/` 目录）中，添加一些支持，使得其可以在 :c:type:`struct bio` 级别处理请求队列中的请求。按照标有 **TODO 6** 的注释进行操作。

将 :c:macro:`USE_BIO_TRANSFER` 宏设置为 1。

实现 :c:func:`my_xfer_request` 函数。使用 :c:macro:`rq_for_each_segment` 宏遍历请求中每个 :c:type:`struct bio` 的 :c:type:`bio_vec` 结构。

.. hint:: 请查阅 :ref:`bio_content` 部分中的指示和代码片段。

.. hint:: 使用 :c:type:`struct bio` 的段迭代器获取当前扇区（:c:member:`iter.iter.bi_sector`）。

.. hint:: 使用请求迭代器获取对当前 :c:type:`struct bio` 的引用（:c:member:`iter.bio`）。

.. hint:: 使用 :c:macro:`bio_data_dir` 宏查找读取或写入的方向。

使用 :c:macro:`kmap_atomic` 或 :c:macro:`kunmap_atomic` 宏映射每个 :c:type:`struct bio` 结构的页面并访问其关联的缓冲区。为了进行实际的传输，调用前面练习中实现的 :c:func:`my_block_transfer` 函数。

为了进行测试，请使用 :file:`ram-disk-test.c` 测试文件：

.. code-block:: shell

   ./ram-disk-test

无需将模块插入到内核中，它将由 :command:`ram-disk-test` 可执行文件插入。

某些测试可能会由于传输数据的缺乏同步（刷新）而崩溃。
