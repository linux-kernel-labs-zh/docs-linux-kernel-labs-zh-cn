============================
文件系统驱动程序（第一部分）
============================

.. meta::
   :description: 了解 Linux 中虚拟文件系统（VFS）的知识，理解有关“inode”、“dentry”、“文件”、“超级块”和数据块的概念，理解在 VFS 内挂载文件系统的过程了解各种文件系统类型，并理解具有物理支持（在磁盘上）和没有物理支持的文件系统之间的区别

实验目标
==============

  * 了解 Linux 中虚拟文件系统（VFS）的知识，理解有关“inode”、“dentry”、“文件”、“超级块”和数据块的概念。
  * 理解在 VFS 内挂载文件系统的过程。
  * 了解各种文件系统类型，并理解具有物理支持（在磁盘上）和没有物理支持的文件系统之间的区别。

虚拟文件系统（VFS）
========================

虚拟文件系统（也称为 VFS）是内核的组件，处理所有与文件和文件系统相关的系统调用。VFS 是用户与特定文件系统之间的通用接口。这种抽象简化了文件系统的实现，并使得多个文件系统更容易集成。这样，通过使用 VFS 提供的 API 来实现文件系统，通用硬件以及 I/O 子系统的通信部分由 VFS 处理。

从功能的角度来看，文件系统可以分为以下几类：

  * 磁盘文件系统（ext3、ext4、xfs、fat 以及 ntfs 等）
  * 网络文件系统（nfs、smbfs/cifs、ncp 等）
  * 虚拟文件系统（procfs、sysfs、sockfs、pipefs 等）

Linux 内核实例使用 VFS 来处理目录和文件的层次结构（一棵树）。通过挂载操作，新的文件系统将被添加为 VFS 子树。文件系统通常是从其所对应的环境中挂载的（从块类型设备、网络等）。然而，VFS 可以将普通文件作为虚拟块设备使用，因此可以将普通文件挂载为磁盘文件系统。这样，可以创建文件系统的堆叠。

VFS 的基本思想是提供可以表示任何文件系统文件的单一文件模型。文件系统驱动程序需要遵守公共的基准。这样，内核可以创建包含整个系统的单一目录结构。其中一个文件系统将作为根文件系统，其他文件系统将挂载在其各个目录下。

常见的文件系统模型
=============================

常见的文件系统模型（任何实现的文件系统都需要符合该模型）包括几种明确定义的实体: :c:type:`superblock`, :c:type:`inode`, :c:type:`file` 和 :c:type:`dentry`。这些实体是文件系统的元数据（包含有关数据或其他元数据的信息）。

模型实体间通过某些 VFS 子系统或内核子系统进行交互：dentry cache（目录项缓存）、inode cache（索引节点缓存）和 buffer cache（缓冲区缓存）。每个实体都被视为对象：它具有关联的数据结构和指向方法表的指针。通过替换关联的方法来为每个组件引入特定的行为。

超级块
----------

超级块存储了挂载文件系统所需的信息：

  * inode 和块的位置
  * 文件系统块大小
  * 最大文件名长度
  * 最大文件大小
  * 根 inode 的位置

本地化：
~~~~~~~~~~~~~

  * 对于磁盘文件系统，超级块在磁盘的第一个块中有对应项（文件系统控制块）。
  * 在 VFS 中，所有文件系统的超级块都保留在类型为 :c:type:`struct super_block` 的结构列表中，方法则保留在类型为 :c:type:`struct super_operations` 的结构中。

inode
-----

inode（索引节点）保存了有关文件的信息。注意这里的文件指的是泛指意义上的文件，常规文件、目录、特殊文件（管道、fifo）、块设备、字符设备、链接或可以抽象为文件的任何内容都包括在内。

inode 存储了以下信息：

  * 文件类型；
  * 文件大小；
  * 访问权限；
  * 访问或修改时间；
  * 数据在磁盘上的位置（指向包含数据的磁盘块的指针）。

.. note::
  通常，inode 不包含文件名。文件名由 :c:type:`dentry` 实体存储。这样，一个 inode 可以有多个名称（硬链接）。

本地化：
~~~~~~~~~~~~~

与 superblock 类似，:c:type:`inode` 也有磁盘对应项。磁盘上的 inodes 通常分组存储在一个专用区域（inode 区域）中，与数据块区域分开；在某些文件系统中，与 inodes 等效的内容分散在文件系统结构中（FAT）；作为 VFS 实体，inode 由 :c:type:`struct inode` 结构表示，并由 :c:type:`struct inode_operations` 结构定义与之相关的操作。

通常，每个 inode 都通过编号进行标识。在 Linux 上, ``ls`` 命令的 ``-i`` 实参显示与每个文件关联的 inode 编号：

.. code-block:: console

    razvan@valhalla:~/school/so2/wiki$ ls -i
    1277956 lab10.wiki  1277962 lab9.wikibak  1277964 replace_lxr.sh
    1277954 lab9.wiki   1277958 link.txt      1277955 homework.wiki

file
-------

file 是文件系统模型中距离用户最近的组件。该结构体仅作为 VFS（虚拟文件系统）在内存中的实体存在，没有在磁盘上的物理对应物。

inode 抽象了磁盘上的文件，而 file 结构抽象了打开的文件。从进程的角度来看，file 实体抽象了文件。然而，从文件系统实现的角度来看，inode 才是抽象文件的那个实体。

file 结构维护了以下信息：

  * 文件游标位置；
  * 文件打开权限；
  * 指向关联 inode 的指针（最终是 inode 的索引）。

本地化：
~~~~~~~~~~~~~

  * 与之关联的 VFS 实体是 :c:type:`struct file` 结构，与之相关的操作由 :c:type:`struct file_operations` 结构表示。

目录项
------

目录项（dentry）将 inode 与文件名关联起来。

通常，dentry 结构包含两个字段：

  * 用于标识 inode 的整数；
  * 表示文件名的字符串。

dentry 是目录或文件路径的特定部分。例如，对于路径 ``/bin/vi``，将为 ``/``, ``bin`` 和 ``vi`` 创建 dentry 对象（总共 3 个 dentry 对象）。

  * dentry 在磁盘上有对应物，但对应关系不是直接的，因为每个文件系统都以特定方式维护 dentry。
  * 在 VFS 中，dentry 实体由 :c:type:`struct dentry` 结构表示，与之相关的操作在 :c:type:`struct dentry_operations` 结构中定义。

注册和注销文件系统
===================

在当前版本中，Linux 内核支持约 50 种文件系统，包括：

  * ext2/ext4
  * reiserfs
  * xfs
  * fat
  * ntfs
  * iso9660
  * 用于 CD 和 DVD 的 udf
  * hpfs

然而，在单个系统上，不太可能有超过 5-6 个文件系统。因此，文件系统（更准确地说，文件系统类型）被实现为模块，并可以随时加载或卸载。

为了能够动态加载/卸载文件系统模块，文件系统注册/注销 API 是不可或缺的。描述特定文件系统的结构是 :c:type:`struct file_system_type`：

	.. code-block:: c

	  #include <linux/fs.h>

	  struct file_system_type {
		   const char *name;
		   int fs_flags;
		   struct dentry *(*mount) (struct file_system_type *, int,
		                             const char *, void *);
		   void (*kill_sb) (struct super_block *);
		   struct module *owner;
		   struct file_system_type *next;
		   struct hlist_head fs_supers;
		   struct lock_class_key s_lock_key;
		   struct lock_class_key s_umount_key;
		   //...
	  };

  * ``name`` 是表示文件系统名称的字符串（传递给 ``mount -t`` 的参数）。
  * ``owner`` 对于以模块形式实现的文件系统来说是 ``THIS_MODULE``，如果直接编写在内核中，则 ``owner`` 为 ``NULL``。
  * ``mount`` 函数在加载文件系统时从磁盘中读取超级块到内存中。每种文件系统的函数都是独一无二的。
  * ``kill_sb`` 函数释放内存中的超级块。
  * ``fs_flags`` 指定文件系统必须以哪些标志挂载。例如, ``FS_REQUIRES_DEV`` 是一个标志，指定 VFS 文件系统需要一个磁盘（而不是虚拟文件系统）。
  * ``fs_supers`` 是一个列表，包含与该文件系统关联的所有超级块。由于同一文件系统可能会被多次挂载，因此每个挂载点都会有一个单独的超级块。

通常，在模块初始化函数中，将 *文件系统注册* 到内核。要进行注册，程序员需要：

  #. 使用名称、标志、实现超级块读取操作的函数以及对标识当前模块的结构的引用来初始化 :c:type:`struct file_system_type` 类型的结构体。
  #. 调用 :c:func:`register_filesystem` 函数。

在卸载模块时，必须调用 :c:func:`unregister_filesystem` 函数来注销文件系统。

在 ``ramfs`` 的代码中可以找到注册虚拟文件系统的示例：

.. code-block:: c

  static struct file_system_type ramfs_fs_type = {
          .name           = "ramfs",
          .mount          = ramfs_mount,
          .kill_sb        = ramfs_kill_sb,
          .fs_flags       = FS_USERNS_MOUNT,
  };

  static int __init init_ramfs_fs(void)
  {
          if (test_and_set_bit(0, &once))
                  return 0;
          return register_filesystem(&ramfs_fs_type);
  }

.. _FunctionsMountKillSBSection:

mount 和 kill_sb 函数
-----------------------

在挂载文件系统时，内核调用了在 :c:type:`struct file_system_type` 结构中定义的 mount 函数。该函数进行一系列的初始化操作，并返回表示挂载点目录的 dentry（:c:type:`struct dentry` 结构）。通常，:c:func:`mount` 是一个简单的函数，该函数调用以下函数之一：

  * :c:func:`mount_bdev`：挂载存储在块设备上的文件系统
  * :c:func:`mount_single`：挂载在所有挂载操作之间共享实例的文件系统
  * :c:func:`mount_nodev`：挂载不在物理设备上的文件系统
  * :c:func:`mount_pseudo`：用于伪文件系统的辅助函数（如 ``sockfs``, ``pipefs`` 等无法被挂载的文件系统）

这些函数的其中一个参数是指向 :c:func:`fill_super` 函数的指针，该函数在超级块初始化之后被调用，以借助驱动程序完成超级块的初始化。在 ``fill_super`` 部分可以找到此类函数的示例。

在卸载文件系统时，内核调用 :c:func:`kill_sb` 函数，执行清理操作，并调用以下函数之一：

  * :c:func:`kill_block_super`：卸载块设备上的文件系统
  * :c:func:`kill_anon_super`：卸载虚拟文件系统（当请求时生成信息）
  * :c:func:`kill_litter_super`：卸载不在物理设备上的文件系统（信息保存在内存中）。

关于没有磁盘支持的文件系统，一个示例是 ``ramfs`` 文件系统的 :c:func:`ramfs_mount` 函数：

.. code-block:: c

  struct dentry *ramfs_mount(struct file_system_type *fs_type,
          int flags, const char *dev_name, void *data)
  {
          return mount_nodev(fs_type, flags, data, ramfs_fill_super);
  }

关于来自磁盘的文件系统，一个示例是 ``minix`` 文件系统的 :c:func:`minix_mount` 函数：

.. code-block:: c

  struct dentry *minix_mount(struct file_system_type *fs_type,
          int flags, const char *dev_name, void *data)
  {
           return mount_bdev(fs_type, flags, dev_name, data, minix_fill_super);
  }

VFS 中的超级块
=================

超级块既作为物理实体（磁盘上的实体）存在，也作为 VFS 实体（在 :c:type:`struct super_block` 结构中）存在。超级块仅包含元信息，并用于从磁盘中读取和写入元数据（如 inode、目录项）。超级块（以及隐式的 :c:type:`struct super_block` 结构）将包含有关所使用的块设备、inode 列表、文件系统根目录的 inode 指针以及超级块操作的指针的信息。

:c:type:`struct super_block` 结构的部分定义如下：

.. code-block:: c

  struct super_block {
          //...
          dev_t                   s_dev;              /* 标识符 */
          unsigned char           s_blocksize_bits;   /* 块大小（以位为单位） */
          unsigned long           s_blocksize;        /* 块大小（以字节为单位） */
          unsigned char           s_dirt;             /* 脏标志 */
          loff_t                  s_maxbytes;         /* 最大文件大小 */
          struct file_system_type *s_type;            /* 文件系统类型 */
          struct super_operations *s_op;              /* 超级块方法 */
          //...
          unsigned long           s_flags;            /* 挂载标志 */
          unsigned long           s_magic;            /* 文件系统的魔数 */
          struct dentry           *s_root;            /* 目录挂载点 */
          //...
          char                    s_id[32];           /* 信息标识符 */
          void                    *s_fs_info;         /* 文件系统私有信息 */
  };

超级块存储了文件系统实例的全局信息：
  * 所使用的物理设备
  * 块大小
  * 文件的最大大小
  * 文件系统类型
  * 支持的操作
  * 魔数（用于标识文件系统）
  * 根目录的 ``dentry``

此外，一个通用指针 (``void *``) 用于存储文件系统的私有数据。超级块可以被视为一个抽象对象，在具体实现时，会向其中添加自己的数据。

.. _SuperblockSection:

超级块操作
---------------------

超级块操作由 :c:type:`struct super_operations` 结构描述：

.. code-block:: c

  struct super_operations {
         //...
         int (*write_inode) (struct inode *, struct writeback_control *wbc);
         struct inode *(*alloc_inode)(struct super_block *sb);
         void (*destroy_inode)(struct inode *);
   
         void (*put_super) (struct super_block *);
         int (*statfs) (struct dentry *, struct kstatfs *);
         int (*remount_fs) (struct super_block *, int *, char *);
         //...
  };

该结构的字段是具有以下含义的函数指针：

* ``write_inode``, ``alloc_inode`` 与 ``destroy_inode`` 分别用于写入、分配和释放与 inode 相关的资源，将在下一个实验中进行详细描述。
* ``put_super`` 在卸载时调用，释放文件系统私有数据的任何资源（通常是内存）；
* ``remount_fs`` 在内核检测到重新挂载尝试（挂载标志 ``MS_REMOUNTM``）时调用；大部分情况下，需要检测是否尝试从只读切换到读写或反之；这可以简单地通过访问旧标志（在 ``sb->s_flags`` 中）和新标志 (``flags`` 参数) 来完成; ``data`` 是由 :c:func:`mount` 发送的表示文件系统特定选项的数据的指针；
* ``statfs`` 在执行 ``statfs`` 系统调用时调用（尝试 ``stat -f`` 或 ``df``）；此调用必须填充 :c:type:`struct kstatfs` 结构的字段，就像在 :c:func:`ext4_statfs` 函数中所做的那样。

.. _FillSuperSection:

:c:func:`fill_super` 函数
===================================

如前所述, :c:func:`fill_super` 函数用于超级块初始化的最后一段。此初始化包括填充 :c:type:`struct super_block` 结构字段和根目录 inode 的初始化。

一个实现示例是 :c:func:`ramfs_fill_super` 函数，它用于初始化超级块中的其余字段：

.. code-block:: c

  #include <linux/pagemap.h>
  
  #define RAMFS_MAGIC     0x858458f6
  
  static const struct super_operations ramfs_ops = {
    .statfs         = simple_statfs,
    .drop_inode     = generic_delete_inode,
    .show_options   = ramfs_show_options,
  };
  
  static int ramfs_fill_super(struct super_block *sb, void *data, int silent)
  {
    struct ramfs_fs_info *fsi;
    struct inode *inode;
    int err;
  
    save_mount_options(sb, data);
  
    fsi = kzalloc(sizeof(struct ramfs_fs_info), GFP_KERNEL);
    sb->s_fs_info = fsi;
    if (!fsi)
      return -ENOMEM;
  
    err = ramfs_parse_options(data, &fsi->mount_opts);
    if (err)
      return err;
  
    sb->s_maxbytes          = MAX_LFS_FILESIZE;
    sb->s_blocksize         = PAGE_SIZE;
    sb->s_blocksize_bits    = PAGE_SHIFT;
    sb->s_magic             = RAMFS_MAGIC;
    sb->s_op                = &ramfs_ops;
    sb->s_time_gran         = 1;
  
    inode = ramfs_get_inode(sb, NULL, S_IFDIR | fsi->mount_opts.mode, 0);
    sb->s_root = d_make_root(inode);
    if (!sb->s_root)
      return -ENOMEM;
  
    return 0;
  }


内核提供了实现文件系统结构的操作的通用函数。上面代码中使用的 :c:func:`generic_delete_inode` 和 :c:func:`simple_statfs` 函数就是这种函数，如果它们的功能足够，可以用于实现驱动程序。

上面代码中的 :c:func:`ramfs_fill_super` 函数填充了超级块中的一些字段，然后读取根 inode 并分配根 dentry。读取根 inode 在 :c:func:`ramfs_get_inode` 函数中完成，它包括使用 :c:func:`new_inode` 函数分配新的 inode 并进行初始化。为了释放 inode，使用了 :c:func:`iput`，并使用 :c:func:`d_make_root` 函数分配根 dentry。

一个用于磁盘文件系统的示例实现是 minix 文件系统中的 :c:func:`minix_fill_super` 函数。磁盘文件系统的功能与虚拟文件系统类似，唯一的区别是使用了缓冲区缓存。此外，minix 文件系统使用 :c:type:`struct minix_sb_info` 结构来保存私有数据。这个函数的很大一部分工作是初始化这些私有数据。私有数据使用 :c:func:`kzalloc` 函数进行分配，并存储在超级块结构的 ``s_fs_info`` 字段中。

VFS 函数通常以超级块、索引节点和/或包含指向超级块的指针的目录项作为实参，以便能够轻松访问这些私有数据。

.. _BufferCacheSection:

缓冲区缓存
============

缓冲区缓存是处理块设备读写缓存的内核子系统。缓冲区缓存使用的基本实体是 :c:type:`struct buffer_head` 结构。该结构中最重要的字段包括：

  * ``b_data``，指向读取数据或写入数据的内存区域的指针
  * ``b_size``，缓冲区大小
  * ``b_bdev``，块设备
  * ``b_blocknr``，已加载或需要保存在磁盘上的设备的块号
  * ``b_state``，缓冲区的状态

以下是与这些结构一起使用的一些重要函数：

  * :c:func:`__bread`：读取具有给定编号和给定大小的块到一个 ``buffer_head`` 结构中；如果成功，则返回指向 ``buffer_head`` 结构的指针，否则返回 ``NULL``；
  * :c:func:`sb_bread`：与前一个函数相同，但读取的块的大小从超级块中获取，读取的设备也从超级块中获取；
  * :c:func:`mark_buffer_dirty`：将缓冲区标记为脏（设置 ``BH_Dirty`` 位）；缓冲区将在稍后的时间写入磁盘 (``bdflush`` 内核线程会定期唤醒并将缓冲区写入磁盘)；
  * :c:func:`brelse`：在先前将缓冲区写入磁盘（如果需要）后，释放缓冲区使用的内存；
  * :c:func:`map_bh`：将 buffer-head 与相应的扇区关联。

函数和有用的宏
===========================

超级块通常包含以位图（位向量）形式表示的占用块的映射（由索引节点、目录条目、数据占用）。为了处理这种映射，建议使用以下功能：

  * :c:func:`find_first_zero_bit`，用于在内存区域中查找第一个为零的位。size 参数表示搜索区域中的位数；
  * :c:func:`test_and_set_bit`，设置位并获取旧值；
  * :c:func:`test_and_clear_bit`，删除位并获取旧值；
  * :c:func:`test_and_change_bit`，取反位的值并获取旧值。

以下宏定义可用于验证索引节点的类型：

  * ``S_ISDIR`` (``inode->i_mode``)，用于检查索引节点是否为目录；
  * ``S_ISREG`` (``inode->i_mode``)，用于检查索引节点是否为普通文件（非链接或设备文件）。

进一步阅读
===============

#. Robert Love——Linux 内核开发，第二版——第 12 章 虚拟文件系统
#. 《深入理解 Linux 内核》，第 3 版——第 12 章 虚拟文件系统
#. `Linux 虚拟文件系统（演示）`_
#. `理解 Unix/Linux 文件系统`_
#. `创建 Linux 虚拟文件系统`_
#. `Linux 文档项目——VFS`_
#. `Linux 中的“虚拟文件系统”`_
#. `Linux 文件系统教程`_
#. `Linux 虚拟文件系统`_
#. `Documentation/filesystems/vfs.txt`_
#. `文件系统源代码`_

.. _Linux 虚拟文件系统（演示）: http://www.coda.cs.cmu.edu/doc/talks/linuxvfs/
.. _理解 Unix/Linux 文件系统: http://www.cyberciti.biz/tips/understanding-unixlinux-file-system-part-i.html
.. _创建 Linux 虚拟文件系统: http://lwn.net/Articles/57369/
.. _Linux文档项目——VFS: http://www.tldp.org/LDP/tlk/fs/filesystem.html
.. _Linux 中的“虚拟文件系统”: http://www.linux.it/~rubini/docs/vfs/vfs.html
.. _Linux 文件系统教程: http://inglorion.net/documents/tutorials/tutorfs/
.. _Linux 虚拟文件系统: http://www.win.tue.nl/~aeb/linux/lk/lk-8.html
.. _Documentation/filesystems/vfs.txt: http://lxr.free-electrons.com/source/Documentation/filesystems/vfs.txt
.. _文件系统源代码: http://lxr.free-electrons.com/source/fs/

练习
=========

.. include:: ../labs/exercises-summary.hrst
.. |LAB_NAME| replace:: 文件系统

..
  _[SURVEY-LABEL]

myfs
----------

首先，我们计划熟悉 Linux 内核和虚拟文件系统（VFS）组件所提供的接口。为此，我们将使用一个简单的虚拟文件系统（即没有物理磁盘支持）。该文件系统名为 ``myfs``。

我们将在实验框架的 ``myfs/`` 子目录中进行操作。我们将在此实验中实现超级块操作，下一个实验将继续进行索引节点操作。

1. 注册和注销 myfs 文件系统
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

处理文件系统的第一步是注册和注销它。我们要为 ``myfs.c`` 中描述的文件系统执行此操作。查看文件内容并按照标记为 ``TODO 1`` 的指示进行操作。

在 :ref:`RegisterUnregisterSection` 部分描述了需要执行的步骤。使用字符串 ``"myfs"`` 作为文件系统名称。

.. note::
  在文件系统结构中，使用代码框架中的 ``myfs_mount`` 函数填充超级块（在挂载时完成）。在 ``myfs_mount`` 中调用专用于没有磁盘支持的文件系统的函数。作为特定挂载函数的参数，使用代码框架中定义的 ``fill_super`` 类型的函数。你可以查看 :ref:`FunctionsMountKillSBSection` 部分。

  要销毁超级块（在卸载时完成），请使用 ``kill_litter_super``，这也是特定于没有磁盘支持的文件系统的函数。该函数已经实现，你需要在 :c:type:`struct file_system_type` 结构中填充它。


完成标记为 ``TODO 1`` 的部分后，编译模块，将其复制到 QEMU 虚拟机中，并启动虚拟机。加载内核模块，然后检查 ``/proc/filesystems`` 文件中是否存在 ``myfs`` 文件系统。

目前，文件系统只是注册了，它没有暴露可以使用的操作。如果我们尝试挂载它，操作将失败。为了尝试挂载，我们创建挂载点 ``/mnt/myfs/``。

.. code-block:: console

  # mkdir -p /mnt/myfs

然后我们使用 ``mount`` 命令：

.. code-block:: console

  # mount -t myfs none /mnt/myfs

我们得到的错误消息显示我们还没有实现在超级块上的操作。我们将需要实现超级块上的操作并初始化根索引节点。我们将在后续步骤中完成这些操作。

.. note::

  发送给 ``mount`` 命令的 ``none`` 实参表示我们没有要挂载的设备，因为文件系统是虚拟的。类似地，这也是 Linux 系统上挂载 ``procfs`` 或 ``sysfs`` 文件系统的方式。

2. 完成 myfs 的超级块
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

为了能够挂载文件系统，我们需要填充其超级块的字段，即类型为 :c:type:`struct super_block` 的通用 VFS 结构。我们将在 :c:func:`myfs_fill_super` 函数内填充该结构；超级块由作为函数实参传递的变量 ``sb`` 表示。请按照标记为 ``TODO 2`` 的提示进行操作。

.. note::

  要填充 ``myfs_fill_super`` 函数，你可以从 :ref:`FillSuperSection` 部分中的示例开始。

  对于超级块结构字段，请尽可能使用代码框架中定义的宏。


超级块结构中的 ``s_op`` 字段必须初始化为超级块操作结构（类型为 :c:type:`struct super_operations`）。你需要定义这样的结构。

有关定义 :c:type:`struct super_operations` 结构和填充超级块的信息，请参阅 :ref:`SuperblockSection` 部分。

.. note::

  初始化 :c:type:`struct super_operations` 结构的 ``drop_inode`` 和 ``statfs`` 字段。


尽管此时超级块将被正确初始化，但挂载操作仍将失败。为了成功完成挂载操作，还需要初始化根索引节点，这将在下一个练习中进行操作。


3. 初始化 myfs 根索引节点
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

根索引节点是文件系统根目录（即 ``/``）的索引节点。初始化是在文件系统挂载时完成的。在挂载时调用的 ``myfs_fill_super`` 函数会调用 ``myfs_get_inode`` 函数来创建并初始化索引节点。通常，所有索引节点都是由此函数创建和初始化；但是，在本练习中，我们只创建根索引节点。

:c:type:`inode` 在 ``myfs_get_inode`` 函数内进行分配（调用 :c:func:`new_inode` 函数，将返回的结果分配给局部变量 ``inode``）。

为了成功完成文件系统的挂载，你需要填充 ``myfs_get_inode`` 函数。按照标记为 ``TODO 3`` 的指示进行操作。可以参考 `ramfs_get_inode <https://elixir.bootlin.com/linux/latest/source/fs/ramfs/inode.c#L63>`_ 函数。

.. note::

  要初始化 ``uid``, ``gid`` 和 ``mode``，可以使用 :c:func:`inode_init_owner` 函数，就像在 :c:func:`ramfs_get_inode` 中那样。调用 :c:func:`inode_init_owner` 时，应将 ``NULL`` 作为第二个参数，因为创建的索引节点没有父目录。

  将 VFS 索引节点的 ``i_atime``, ``i_ctime`` 和 ``i_mtime`` 初始化为 :c:func:`current_time` 函数返回的值。

  你需要为目录类型的索引节点初始化操作。执行以下步骤：

    #. 使用 ``S_ISDIR`` 宏检查这是否是目录类型的索引节点。
    #. 对于 ``i_op`` 和 ``i_fop`` 字段，请使用已经实现的内核函数：

       * 对于 ``i_op``：使用 :c:type:`simple_dir_inode_operations`。
       * 对于 ``i_fop``：使用 :c:type:`simple_dir_operations`。

    #. 使用 :c:func:`inc_nlink` 函数增加目录的链接数。

4. 测试 myfs 的挂载和卸载
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

现在我们可以挂载文件系统了。按照上述步骤编译内核模块，将其复制到虚拟机中，启动虚拟机，然后插入内核模块，创建挂载点 ``/mnt/myfs/``，并挂载文件系统。我们可以通过检查 ``/proc/mounts`` 文件来验证文件系统是否已挂载。

``/mnt/myfs``目录的索引节点号是多少？为什么？

.. note::

  要显示目录的索引节点号，请使用以下命令：

  .. code-block:: console

    ls -di /path/to/directory

  其中 ``/path/to/directory/`` 是要显示其索引节点号的目录的路径。

我们使用以下命令检查 myfs 文件系统的统计信息：

.. code-block:: console

  stat -f /mnt/myfs

我们想查看挂载点 ``/mnt/myfs`` 的内容以及是否可以创建文件。为此，我们运行以下命令：

.. code-block:: console

  # ls -la /mnt/myfs
  # touch /mnt/myfs/a.txt

我们可以看到我们无法在文件系统上创建 ``a.txt`` 文件。这是因为我们尚未在 :c:type:`struct super_operations` 结构中实现与索引节点相关的操作。我们将在下一个实验中实现这些操作。

使用以下命令卸载文件系统：

.. code-block:: console

  umount /mnt/myfs

同时卸载对应的内核模块。

.. note::

  要测试整个功能，你可以使用 ``test-myfs.sh`` 脚本：

  .. code-block:: console

    ./test-myfs.sh

  该脚本将使用 ``make copy`` 将其复制到虚拟机，但前提是它具有可执行权限：

  .. code-block:: console

    student@workstation:~/linux/tools/labs$ chmod +x skels/filesystems/myfs/test-myfs.sh


.. note::

  显示的文件系统统计信息很简单，因为这些信息是由 simple_statfs 函数提供的。

minfs
-----

接下来，我们将实现一个非常简单的文件系统，名为 ``minfs``，这个文件系统支持磁盘。我们将使用虚拟机中的一个磁盘，格式化并挂载 ``minfs`` 文件系统。

为此，我们需要从实验框架中访问 ``minfs/kernel`` 目录，并处理 ``minfs.c`` 中的代码。与 ``myfs`` 类似，我们不会实现与索引节点相关的操作，只限于处理超级块和挂载。其他操作将在下一个实验中实现。

请按照下面的图表来理解 ``minfs`` 文件系统中各个结构的作用。

.. image:: ../res/minfs.png

1. 注册和注销 minfs 文件系统
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::

  在解决本练习之前，我们需要在虚拟机中添加一个磁盘。你可以使用以下命令生成一个文件作为磁盘镜像：

  .. code-block:: console

    dd if=/dev/zero of=mydisk.img bs=1M count=100

  并且在 ``qemu/Makefile`` 文件中的 ``qemu`` 命令中添加 ``-drive file=mydisk.img,if=virtio,format=raw`` 参数（在 ``QEMU_OPTS`` 变量中）。新的 ``qemu`` 命令参数必须在现有磁盘参数 (``YOCTO_IMAGE``) 之后添加。

要注册和注销文件系统，你需要在 ``minfs.c`` 中填写 ``minfs_fs_type`` 和 ``minfs_mount`` 函数。按照标有 ``TODO 1`` 的指引进行操作。

.. note::

  在文件系统结构中，对于挂载，请使用代码框架中的 ``minfs_mount`` 函数。在此函数中，调用带有磁盘支持的文件系统挂载函数（请参见 :ref:`FunctionsMountKillSBSection` 部分。使用 :c:func:`mount_bdev`）。选择最合适的函数来销毁超级块（在卸载时完成）；请记住这是带有磁盘支持的文件系统。使用 :c:func:`kill_block_super` 函数。

  使用适当的值初始化 :c:type:`minfs_fs_type` 结构的 ``fs_flags`` 字段，以适应带有磁盘支持的文件系统。请参阅 :ref:`RegisterUnregisterSection` 部分。

  填充超级块的函数是 ``minfs_fill_super``。

完成标有 ``TODO 1`` 的部分后，编译模块，将其复制到 QEMU 虚拟机中，并启动虚拟机。加载内核模块，然后检查 ``/proc/filesystems`` 文件中是否存在 ``minfs`` 文件系统。

为了测试 ``minfs`` 文件系统的挂载，我们需要使用其结构对磁盘进行格式化。格式化需要使用 ``minfs/user`` 目录下的 ``mkfs.minfs`` 格式化工具。该工具在运行 ``make build`` 时会自动编译，并在 ``make copy`` 时复制到虚拟机中。

编译、复制和启动虚拟机后，使用格式化工具对 ``/dev/vdd`` 进行格式化：

.. code-block:: console

  # ./mkfs.minfs /dev/vdd

加载内核模块：

.. code-block:: console

  # insmod minfs.ko

创建挂载点 ``/mnt/minfs/``：

.. code-block:: console

  # mkdir -p /mnt/minfs/

并挂载文件系统

.. code-block:: console

  # mount -t minfs /dev/vdd /mnt/minfs/

操作失败，因为根索引节点未初始化。

2. 完善 minfs 超级块
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

为了能够挂载文件系统，你需要在 ``minfs_fill_super`` 函数中填充超级块（即类型为 :c:type:`struct super_block` 的结构体），超级块对应该函数的 ``s`` 实参。操作超级块的结构已经定义好了: ``minfs_ops``。按照标有 ``TODO 2`` 的指引操作。你还可以参考 `minix_fill_super <https://elixir.bootlin.com/linux/latest/source/fs/minix/inode.c#L153>`_ 函数的实现。

.. note::

  一些结构可以在头文件 ``minfs.h`` 中找到。

  有关使用缓冲区的信息，请参阅 :ref:`BufferCacheSection` 部分。

  读取磁盘上的第一个块（索引为 0 的块）。要读取块，请使用 :c:func:`sb_bread` 函数。将读取的数据（:c:type:`struct buffer_head` 结构中的 ``b_data`` 字段）转换为存储磁盘上的 ``minfs`` 超级块的信息的结构体：在源代码文件中定义的 :c:type:`struct minfs_super_block`。

  结构 :c:type:`struct minfs_super_block` 包含了文件系统特定的信息，这些信息在 :c:type:`struct super_block` 通用结构中找不到（在这种情况下只有版本号）。这些附加信息（在磁盘上的 :c:type:`struct minfs_super_block` 中找到，但在 :c:type:`struct super_block`（VFS）中找不到）将存储在 :c:type:`struct minfs_sb_info` 结构中。

为了检查功能，我们需要用于读取根索引节点的函数。这里暂时使用 ``myfs`` 文件系统练习中的 ``myfs_get_inode`` 函数。将该函数复制到源代码中，并像处理 myfs 时一样调用它。调用 ``myfs_get_inode`` 函数时的第三个参数是索引节点的创建权限，与虚拟文件系统练习（myfs）中的类似。

执行上一个练习中的命令来验证实现。

3. 创建和销毁 minfs 索引节点
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

挂载操作中，我们需要初始化根索引节点，并且为了获得根索引节点，我们需要实现与索引节点相关的函数。也就是说，你需要实现 ``minfs_alloc_inode`` 和 ``minfs_destroy_inode`` 函数。按照标有 ``TODO 3`` 的指示进行操作。你可以将 :c:func:`minix_alloc_inode` 和 :c:func:`minix_destroy_inode` 函数作为参考。

为了实现，请查看 ``minfs.h`` 头文件中的宏和结构。

.. note::

  要想实现在 ``minfs_alloc_inode`` 和 ``minfs_destroy_inode`` 中的内存分配/释放，建议使用 :c:func:`kzalloc` 和 :c:func:`kfree`。

  在 ``minfs_alloc_inode`` 中，分配类型为 :c:type:`struct minfs_inode_info` 的结构体，但只返回类型为 :c:type:`struct inode` 的结构体，即返回 ``vfs_inode`` 字段对应的结构体。

  在 ``minfs_alloc_inode`` 函数中，调用 :c:func:`inode_init_once` 来初始化索引节点。

  在 ``destroy_inode`` 函数中，你可以使用 ``container_of`` 宏访问 :c:type:`struct minfs_inode_info` 结构体。

.. note::

  在本练习中，你已经实现了 ``minfs_alloc_inode`` 和 ``minfs_destroy_inode`` 函数，但尚未调用它们。实现的正确性将在下一个练习的最后进行检查。

4. 初始化 minfs 根索引节点
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

为了挂载文件系统，需要初始化根索引节点。为此，你需要完成 ``minfs_ops`` 结构体，包括 ``minfs_alloc_inode`` 和 ``minfs_destroy_inode`` 函数，并填充 ``minfs_iget`` 函数。

``minfs_iget`` 函数用于分配 VFS 索引节点（即 :c:type:`struct inode`），并用磁盘中的 minfs 索引节点特定信息（即 ``struct minfs_inode``）来填充它。

按照标有 ``TODO 4`` 的指示进行操作。在 :c:type:`struct super_operations` 结构体的 ``alloc_inode`` 和 ``destroy_inode`` 字段中填写在上一步中实现的函数。

根索引节点的信息存储在磁盘上的第二个块中（索引为 1 的索引节点）。使 ``minfs_iget`` 从磁盘中读取根 minfs 索引节点（:c:type:`struct minfs_inode`）并填充 VFS 索引节点（:c:type:`struct inode`）。

在 ``minfs_fill_super`` 函数中，用 ``minfs_iget`` 函数调用替换 ``myfs_get_inode`` 的调用。

.. note::
  要实现 ``minfs_iget`` 函数，请参考 `V1_minix_iget <https://elixir.bootlin.com/linux/v4.15/source/fs/minix/inode.c#L460>`_ 的实现。要读取块，请使用 :c:func:`sb_bread` 函数。将读取的数据（:c:type:`struct buffer_head` 结构中的 ``b_data`` 字段）强制转换为磁盘上的 minfs 索引节点（:c:type:`struct minfs_inode`）。

  使用从磁盘读取的 minfs 索引节点结构中的值填充 VFS 索引节点中的 ``i_uid``, ``i_gid``, ``i_mode`` 以及 ``i_size`` 字段。要初始化 ``i_uid`` 和 ``i_gid`` 字段，请使用函数 :c:func:`i_uid_write` 和 :c:func:`i_gid_write`。

  将 VFS 索引节点的 ``i_atime``, ``i_ctime`` 和 ``i_mtime`` 字段初始化为 :c:func:`current_time` 函数的返回值。

  你需要为目录类型的索引节点初始化操作。请按照以下步骤进行操作：

    #. 使用 ``S_ISDIR`` 宏检查是否为目录类型的索引节点。
    #. 对于 ``i_op`` 和 ``i_fop`` 字段，使用已实现的内核函数：

       * 对于 ``i_op``：:c:func:`simple_dir_inode_operations`。
       * 对于 ``i_fop``：:c:func:`simple_dir_operations`。

    #. 使用 :c:func:`inc_nlink` 函数增加目录的链接数。

5. minfs 挂载和卸载的测试
~~~~~~~~~~~~~~~~~~~~~~~~~~

现在我们可以挂载文件系统了。按照上述步骤编译内核模块，将其复制到虚拟机中，启动虚拟机，然后插入内核模块，创建挂载点 ``/mnt/minfs/`` 并挂载文件系统。我们通过查看 ``/proc/mounts`` 文件来验证文件系统是否已挂载。

通过列出挂载点内容 ``/mnt/minfs/`` 来检查是否一切正常：

.. code-block:: console

  # ls /mnt/minfs/

挂载和验证完成后，卸载文件系统并从内核中卸载模块。

.. note::
  或者，你可以使用 ``test-minfs.sh`` 脚本来完整测试功能：

  .. code-block:: console

    # ./test-minfs.sh

  仅当该脚本是可执行文件，在运行 ``make copy`` 命令时该脚本会被复制到虚拟机中。

  .. code-block:: console

    student@workstation:~/linux/tools/labs$ chmod +x skels/filesystems/minfs/user/test-minfs.sh
