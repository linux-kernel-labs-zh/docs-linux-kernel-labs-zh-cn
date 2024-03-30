============================
文件系统驱动程序（第二部分）
============================

.. meta::
   :description: 提高对 inode、file 和 dentry 的了解，了解如何在 VFS（虚拟文件系统）中添加对常规文件和目录的支持，了解文件系统的内部实现

实验目标
==============

  * 提高对 inode、file 和 dentry 的了解。
  * 了解如何在 VFS（虚拟文件系统）中添加对常规文件和目录的支持。
  * 了解文件系统的内部实现。

Inode
=====

inode 是 UNIX 文件系统的重要组成部分，同时也是 VFS 的重要组成部分。inode 是元数据（它包含信息的信息）。inode 是磁盘上文件的唯一标识，并保存文件的信息（uid、gid、访问权限、访问时间以及指向数据块的指针等）。重要的一点是，inode 不保存文件名信息（文件名由相关的 :c:type:`struct dentry` 结构保存）。

inode 用于引用磁盘上的文件。要引用打开的文件（与进程内的文件描述符相关联），需要使用 :c:type:`struct file` 结构。一个 inode 可以关联任意数量的（零个或多个） ``file`` 结构（多个进程可以打开同一个文件，或者一个进程可以多次打开同一个文件）。

inode 既存在于 VFS 中（内存中），也存在于磁盘中（对于 UNIX、HFS 以及 NTFS 等）。VFS 中的 inode 由 :c:type:`struct inode` 结构表示。和 VFS 中的其他结构一样, :c:type:`struct inode` 是通用结构，涵盖了所有支持的文件类型的选项，甚至包括那些没有关联磁盘实体的文件类型（比如 FAT 文件系统）。

inode 结构
-------------------

inode 结构在所有文件系统中都是相同的。一般情况下，文件系统还有私有信息，这些信息通过结构的 ``i_private`` 字段引用。按照惯例，保存特定信息的结构被称为 ``<fsname>_inode_info``，其中 ``fsname`` 表示文件系统名称。例如，minix 和 ext4 文件系统将特定信息保存在 :c:type:`struct minix_inode_info` 或 :c:type:`struct ext4_inode_info` 结构中。

:c:type:`struct inode` 的一些重要字段包括：

  * ``i_sb``：inode 所属的文件系统的超级块结构。
  * ``i_rdev``：挂载的文件系统所在的设备
  * ``i_ino``：inode 的编号（在文件系统内唯一标识 inode）
  * ``i_blkbits``：块大小使用的比特数 == log\ :sub:`2`\ (块大小)
  * ``i_mode``, ``i_uid`` 以及 ``i_gid``：访问权限、uid 以及 gid

  * ``i_size``：文件/目录等的大小（以字节为单位）
  * ``i_mtime``, ``i_atime`` 以及 ``i_ctime``：修改、访问和创建时间
  * ``i_nlink``：使用此 inode 的名称条目（dentry）的数量；对于没有链接（既没有硬链接也没有符号链接）的文件系统，这个值总是设置为 1
  * ``i_blocks``：文件使用的块数（所有块，不仅仅是数据块）；这仅由配额子系统使用
  * ``i_op``, ``i_fop``：指向操作结构的指针：:c:type:`struct inode_operations` 和 :c:type:`struct file_operations`; ``i_mapping->a_ops`` 包含指向 :c:type:`struct address_space_operations` 的指针。
  * ``i_count``：inode 计数器，指示有多少内核组件在使用它。

一些可用于处理 inode 的函数包括：

  * :c:func:`new_inode`：创建新的 inode，将 ``i_nlink`` 字段设置为 1，并初始化 ``i_blkbits``, ``i_sb`` 和 ``i_dev``；
  * :c:func:`insert_inode_hash`：将 inode 添加到 inode 哈希表中；这个调用的一个有趣的效果是，如果 inode 被标记为脏，它将被写入磁盘；

    .. warning::

      使用 :c:func:`new_inode` 创建的 inode 不在哈希表中，除非你有充分的理由，否则你必须将其加入哈希表；

  * :c:func:`mark_inode_dirty`：将 inode 标记为脏；稍后它将被写入磁盘；
  * :c:func:`iget_locked`：从磁盘加载具有给定编号的 inode，如果它尚未加载。
  * :c:func:`unlock_new_inode`：与 :c:func:`iget_locked` 一起使用，释放对 inode 的锁定；
  * :c:func:`iput`：告诉内核对 inode 的操作已经完成；如果没有其他进程在使用它，它将被销毁（如果被标记为脏，那么写入磁盘后再销毁）；
  * :c:func:`make_bad_inode`：告诉内核该 inode 无法使用；通常在从磁盘读取 inode 时发现无法读取的情况下使用，表示该 inode 无效。

Inode 操作
----------------

获取 inode
^^^^^^^^^^^^^^^^

获取 inode（在 VFS 中的 :c:type:`struct inode`）是主要的 inode 操作之一。在 Linux 内核版本 ``2.6.24`` 之前，开发者定义了 ``read_inode`` 函数。从版本 ``2.6.25`` 开始，开发者必须定义 ``<fsname>_iget`` 函数，其中 ``<fsname>`` 是文件系统的名称。这个函数负责查找 VFS 中的 inode，如果存在则获取该 inode，否则创建一个新的 inode，并用磁盘中的信息填充它。

一般情况下，这个函数会调用 :c:func:`iget_locked` 从 VFS 中获取 inode 结构。如果 inode 是新创建的，则需要从磁盘中读取 inode（使用 :c:func:`sb_bread`），并填充有用的信息。

一个示例函数是 :c:func:`minix_iget`：

.. code-block:: c

  static struct inode *V1_minix_iget(struct inode *inode)
  {
  	struct buffer_head * bh;
  	struct minix_inode * raw_inode;
  	struct minix_inode_info *minix_inode = minix_i(inode);
  	int i;

  	raw_inode = minix_V1_raw_inode(inode->i_sb, inode->i_ino, &bh);
  	if (!raw_inode) {
  		iget_failed(inode);
  		return ERR_PTR(-EIO);
  	...
  }

  struct inode *minix_iget(struct super_block *sb, unsigned long ino)
  {
  	struct inode *inode;

  	inode = iget_locked(sb, ino);
  	if (!inode)
  		return ERR_PTR(-ENOMEM);
  	if (!(inode->i_state & I_NEW))
  		return inode;

  	if (INODE_VERSION(inode) == MINIX_V1)
  		return V1_minix_iget(inode);
      ...
  }

minix_iget 函数使用 :c:func:`iget_locked` 函数获取 VFS inode。如果该 inode 已经存在（非新建，即 ``I_NEW`` 标志未设置），则函数返回。否则，函数调用 :c:func:`V1_minix_iget` 函数，该函数将使用 :c:func:`minix_V1_raw_inode` 从磁盘读取 inode，然后使用读取的信息完成 VFS inode 的初始化。

超级块操作
^^^^^^^^^^^^^^^

许多超级块使用的超级块操作（:c:type:`struct super_operations` 结构的组成部分）在处理 inode 时使用。下面描述了这些操作：

  * ``alloc_inode``: 分配 inode。通常，此函数会分配一个 :c:type:`struct <fsname>_inode_info` 结构，并执行基本的 VFS inode 初始化（使用 :c:func:`inode_init_once`）；minix 使用 :c:func:`kmem_cache_alloc` 函数进行分配，该函数与 SLAB 子系统交互。对于每个分配，都会调用缓存构造函数，在 minix 的情况下是 :c:func:`init_once` 函数。或者，也可以使用 :c:func:`kmalloc`，在这种情况下，应调用 :c:func:`inode_init_once` 函数。:c:func:`alloc_inode` 函数将由 :c:func:`new_inode` 和 :c:func:`iget_locked` 函数调用。
  * ``write_inode``：将作为参数接收的 inode 保存/更新到磁盘；要更新 inode，尽管效率不高，但对于初学者来说，建议使用以下操作：

    * 使用 :c:func:`sb_bread` 函数从磁盘加载 inode；
    * 根据保存的 inode 修改缓冲区；
    * 使用 :c:func:`mark_buffer_dirty` 将缓冲区标记为脏；内核将处理其在磁盘上的写入；
    * 一个示例是 ``minix`` 文件系统中的 :c:func:`minix_write_inode` 函数。

  * ``evict_inode``：从磁盘和内存中移除通过 ``i_ino`` 字段接收的 inode 的任何信息（包括磁盘上的 inode 和相关的数据块）。这涉及执行以下操作：

    * 从磁盘中删除 inode；
    * 更新磁盘位图（如果有）；
    * 通过调用 :c:func:`truncate_inode_pages` 从 page cache 中删除 inode；
    * 通过调用 :c:func:`clear_inode` 从内存中删除 inode；
    * 一个示例是 minix 文件系统中的 :c:func:`minix_evict_inode` 函数。

  * ``destroy_inode`` 释放 inode 占用的内存

inode_operations
^^^^^^^^^^^^^^^^

索引节点操作由 :c:type:`struct inode_operations` 结构描述。

索引节点分为多种类型：文件、目录、特殊文件（管道、FIFO）、块设备、字符设备以及链接等。因此，每种类型的索引节点需要实现的操作都不同。下面详细介绍了对 :ref:`文件类型的索引节点 <FileInodes>` 和 :ref:`目录类型的索引节点 <DirectoryInodes>` 的操作。

对索引节点的操作通过 :c:type:`struct inode` 结构中的 ``i_op`` 字段进行初始化和访问。

file 结构
==================

``file`` 结构对应于由进程打开的文件，仅存在于内存中，并与索引节点关联。它是最接近用户空间的 VFS 实体；结构字段包含用户空间文件的熟悉信息（访问模式、文件位置等），与之相关的操作由已知的系统调用 (``read``, ``write`` 等)执行。

文件操作由 :c:type:`struct file_operations` 结构描述。

文件系统的文件操作使用 :c:type:`struct inode` 结构中的 ``i_fop`` 字段进行初始化。在打开文件时，VFS 使用 ``inode->i_fop`` 的地址初始化 :c:type:`struct file` 结构的 ``f_op`` 字段，因此后续的系统调用使用存储在 ``file->f_op`` 中的值。

.. _FileInodes:

常规文件索引节点
====================

要使用索引节点，必须填充索引节点结构的 ``i_op`` 和 ``i_fop`` 字段。索引节点的类型决定了它需要实现的操作。

.. _FileOperations:

常规文件索引节点操作
------------------------------

``minix`` 文件系统为索引节点操作定义了 ``minix_file_inode_operations`` 结构，而对于文件操作，则定义了 ``minix_file_operations`` 结构：

.. code-block:: c

  const struct file_operations minix_file_operations = {
           .llseek         = generic_file_llseek,
           .read_iter      = generic_file_read_iter,
           //...
           .write_iter     = generic_file_write_iter,
           //...
           .mmap           = generic_file_mmap,
           //...
  };

  const struct inode_operations minix_file_inode_operations = {
          .setattr        = minix_setattr,
          .getattr        = minix_getattr,
  };

          //...
          if (S_ISREG(inode->i_mode)) {
                  inode->i_op = &minix_file_inode_operations;
                  inode->i_fop = &minix_file_operations;
          }
          //...



内核中实现了 :c:func:`generic_file_llseek`, :c:func:`generic_file_mmap`, :c:func:`generic_file_read_iter` 和 :c:func:`generic_file_write_iter` 函数。

对于简单的文件系统，只需要实现截断操作 (``truncate`` 系统调用)。尽管最初有一个专用的操作，但从 3.14 版本开始，该操作已嵌入到 ``setattr`` 中：如果粘贴大小与索引节点的当前大小不同，则必须执行截断操作。在 :c:func:`minix_setattr` 函数中，有实现此验证的示例：

.. code-block:: c

  static int minix_setattr(struct dentry *dentry, struct iattr *attr)
  {
          struct inode *inode = d_inode(dentry);
          int error;

          error = setattr_prepare(dentry, attr);
          if (error)
                  return error;

          if ((attr->ia_valid & ATTR_SIZE) &&
              attr->ia_size != i_size_read(inode)) {
                  error = inode_newsize_ok(inode, attr->ia_size);
                  if (error)
                          return error;

                  truncate_setsize(inode, attr->ia_size);
                  minix_truncate(inode);
          }

          setattr_copy(inode, attr);
          mark_inode_dirty(inode);
          return 0;
  }

截断操作涉及以下内容：

  * 释放磁盘上多余的数据块（如果新尺寸小于旧尺寸），或者分配新的数据块（当新尺寸较大时）；
  * 更新磁盘位图（如果使用）；
  * 更新索引节点；
  * 使用 :c:func:`block_truncate_page` 函数，将上一个块中未使用的空间填充为零。

在 ``minix`` 文件系统中，有一个实现截断操作的示例是 :c:func:`minix_truncate` 函数。

.. _AddressSpaceOperations:

地址空间操作
------------------------

进程的地址空间与文件之间有着密切的联系：程序的执行几乎完全是通过将文件映射到进程的地址空间中进行的。由于这种方法非常有效且相当通用，因此也可以用于常规的系统调用，如 ``read`` 和 ``write``。

描述地址空间的结构是 :c:type:`struct address_space`，与之相关的操作由结构 :c:type:`struct address_space_operations` 描述。要初始化地址空间操作，请填充文件类型索引节点的 ``inode->i_mapping->a_ops``。

一个示例是 minix 文件系统中的 ``minix_aops`` 结构：

.. code-block:: c

  static const struct address_space_operations minix_aops = {
         .readpage = minix_readpage,
         .writepage = minix_writepage,
         .write_begin = minix_write_begin,
         .write_end = generic_write_end,
         .bmap = minix_bmap
  };

  //...
  if (S_ISREG(inode->i_mode)) {
        inode->i_mapping->a_ops = &minix_aops;
  }
  //...

:c:func:`generic_write_end` 函数已经实现。大多数特定函数非常容易实现，如下所示：

.. code-block:: c

  static int minix_writepage(struct page *page, struct writeback_control *wbc)
  {
           return block_write_full_page(page, minix_get_block, wbc);
  }

  static int minix_readpage(struct file *file, struct page *page)
  {
           return block_read_full_page(page, minix_get_block);
  }

  static void minix_write_failed(struct address_space *mapping, loff_t to)
  {
          struct inode *inode = mapping->host;

          if (to > inode->i_size) {
                  truncate_pagecache(inode, inode->i_size);
                  minix_truncate(inode);
          }
  }

  static int minix_write_begin(struct file *file, struct address_space *mapping,
                          loff_t pos, unsigned len, unsigned flags,
                          struct page **pagep, void **fsdata)
  {
          int ret;

          ret = block_write_begin(mapping, pos, len, flags, pagep,
                                  minix_get_block);
          if (unlikely(ret))
                  minix_write_failed(mapping, pos + len);

          return ret;
  }

  static sector_t minix_bmap(struct address_space *mapping, sector_t block)
  {
           return generic_block_bmap(mapping, block, minix_get_block);
  }

只需实现 :c:type:`minix_get_block` 函数，该函数将文件的一个数据块转换为设备上的一个数据块。如果接收到的 ``create`` 标志被设置，那么必须分配一个新的数据块。在创建新的数据块时，必须相应地更新位图。为了通知内核不要从磁盘中读取该数据块，必须使用 :c:func:`set_buffer_new` 函数标记 ``bh``。通过 :c:func:`map_bh` 函数，将缓冲区与数据块关联起来。

Dentry 结构体
================

目录操作使用 :c:type:`struct dentry` 结构体。它的主要任务是在索引节点和文件名之间建立链接。该结构体的重要字段如下所示：

.. code-block:: c

  struct dentry {
          //...
          struct inode             *d_inode;     /* 关联的索引节点 */
          //...
          struct dentry            *d_parent;    /* 父目录的 dentry 对象 */
          struct qstr              d_name;       /* dentry 名称 */
          //...

          struct dentry_operations *d_op;        /* dentry 操作表 */
          struct super_block       *d_sb;        /* 文件的超级块 */
          void                     *d_fsdata;    /* 文件系统特定的数据 */
          //...
  };

字段含义：

  * ``d_inode``：由该 dentry 引用的索引节点；
  * ``d_parent``：与父目录相关联的 dentry；
  * ``d_name``：:c:type:`struct qstr` 结构，包含字段 ``name`` 和 ``len``（名称和名称的长度）。
  * ``d_op``：与 dentry 相关的操作，由 :c:type:`struct dentry_operations` 结构表示。内核实现了默认操作，因此无需（重新）实现它们。某些文件系统可以根据 dentry 的特定结构进行优化。
  * ``d_fsdata``：保留给实现 dentry 操作的文件系统特定的字段；

Dentry 操作
-----------------

应用于 dentry 的最常见操作包括：

  * ``d_make_root``：分配根 dentry。通常在读取超级块的函数 (``fill_super``) 中使用，该函数必须初始化根目录。因此，我们从超级块获取根索引节点，并将其作为实参传递给此函数，以填充 :c:type:`struct super_block` 结构的 ``s_root`` 字段。
  * ``d_add``：将 dentry 与索引节点关联起来；在上述讨论中，作为参数传递的 dentry 表示需要创建的条目（名称、长度）。在创建/加载尚未与任何 dentry 关联并尚未添加到索引节点哈希表中的新索引节点时，将使用此函数（在 ``lookup`` 中）。
  * ``d_instantiate``：上述调用的轻量级版本，其中 dentry 先前已添加到哈希表中。

.. warning::

  ``d_instantiate`` 必须用于实现创建调用 (``mkdir``, ``mknod``, ``rename`` 以及  ``symlink``)，而不是 ``d_add``。

.. _DirectoryInodes:

目录索引节点操作
===========================

目录类型的索引节点相关的操作比文件类型的索引节点操作复杂得多。开发人员必须定义索引节点的操作和文件的操作。在 ``minix`` 中，这些操作定义在 :c:type:`minix_dir_inode_operations` 和 :c:type:`minix_dir_operations` 中：

.. code-block:: c

  struct inode_operations minix_dir_inode_operations = {
        .create = minix_create,
        .lookup = minix_lookup,
        .link = minix_link,
        .unlink = minix_unlink,
        .symlink = minix_symlink,
        .mkdir = minix_mkdir,
        .rmdir = minix_rmdir,
        .mknod = minix_mknod,
        //...
  };

  struct file_operations minix_dir_operations = {
        .llseek = generic_file_llseek,
        .read = generic_read_dir,
        .iterate = minix_readdir,
        //...
  };

          //...
  	if (S_ISDIR(inode->i_mode)) {
  		inode->i_op = &minix_dir_inode_operations;
  		inode->i_fop = &minix_dir_operations;
  		inode->i_mapping->a_ops = &minix_aops;
  	}
         //...

我们唯一已经实现的函数是 :c:func:`generic_read_dir`。

实现目录索引节点操作的函数如下所述。

创建索引节点
-----------------

索引节点创建函数由 ``inode_operations`` 结构体中的 ``create`` 字段指示。在 minix 的例子中，该函数是 :c:func:`minix_create`。此函数由 ``open`` 和 ``creat`` 系统调用调用。该函数执行以下操作：

  #. 在磁盘上的物理结构中引入新条目；不要忘记更新磁盘上的位图。
  #. 使用传入函数的访问权限配置访问权限。
  #. 使用 :c:func:`mark_inode_dirty` 函数将索引节点标记为脏。
  #. 使用 ``d_instantiate`` 函数实例化目录条目 (``dentry``)。

创建目录
--------------------

目录创建函数由 ``inode_operations`` 结构体中的 ``mkdir`` 字段指示。在 minix 的例子中，该函数是 :c:func:`minix_mkdir`。此函数由 ``mkdir`` 系统调用调用。该函数执行以下操作：

  #. 调用 :c:func:`minix_create`。
  #. 为目录分配一个数据块。
  #. 创建 ``"."`` 和 ``".."`` 条目。

创建链接
---------------

链接创建函数（硬链接）由 ``inode_operations`` 结构体中的 ``link`` 字段指示。在 minix 的例子中，该函数是 :c:func:`minix_link`。此函数由 ``link`` 系统调用调用。该函数执行以下操作：

  * 将新的 dentry 绑定到索引节点。
  * 递增索引节点的 ``i_nlink`` 字段。
  * 使用 :c:func:`mark_inode_dirty` 函数将索引节点标记为脏。

创建符号链接
------------------------

符号链接创建函数由 ``inode_operations`` 结构体中的 ``symlink`` 字段指示。在 minix 的例子中，该函数是 :c:func:`minix_symlink`。要执行的操作与 ``minix_link`` 类似，区别在于创建了一个符号链接。

删除链接
---------------

链接删除函数（硬链接）由 ``inode_operations`` 结构体中的 ``unlink`` 字段指示。在 minix 的例子中，该函数是 :c:func:`minix_unlink`。此函数由 ``unlink`` 系统调用调用。该函数执行以下操作：

  #. 从物理磁盘结构中删除作为参数给出的 dentry
  #. 将条目指向的索引节点的 ``i_nlink`` 计数器减一（否则该索引节点将永远不会被删除）

删除目录
--------------------

目录删除函数由 ``inode_operations`` 结构体中的 ``rmdir`` 字段指示。在 minix 的例子中，该函数是 :c:func:`minix_rmdir`。此函数由 ``rmdir`` 系统调用调用。该函数执行以下操作：

  #. 执行 ``minix_unlink`` 完成的操作
  #. 确保目录为空；否则，返回 ``ENOTEMPTY``
  #. 还删除数据块

在目录中搜索索引节点
-------------------------------------

在目录中搜索条目并提取索引节点的函数由 ``inode_operations`` 结构体中的 ``lookup`` 字段指示。在 minix 的例子中，该函数是 ``minix_lookup``。当需要有关与目录中条目关联的索引节点的信息时，会间接调用此函数。该函数执行以下操作：

  #. 在由 ``dir`` 指示的目录中搜索具有名称 ``dentry->d_name.name`` 的条目
  #. 如果找到条目，则返回 ``NULL`` 并使用 :c:func:`d_add` 函数将索引节点与名称关联
  #. 否则，返回 ``ERR_PTR``

遍历目录中的条目
----------------------------------------

在目录中遍历条目（列出目录内容）的函数由 ``struct file_operations`` 结构体中的 ``iterate`` 字段指示。在 minix 的例子中，该函数是 ``minix_readdir``。此函数由 ``readdir`` 系统调用调用。

该函数返回目录中的所有条目，或者当为其分配的缓冲区不可用时，仅返回部分条目。此函数的调用可能返回：

  * 如果对应的用户空间缓冲区有足够的空间，则返回与现有条目数相等的数字；
  * 小于实际条目数的数字，对应的用户空间缓冲区中有多少空间，就返回多少；
  * ``0``，表示没有更多条目可读取。

该函数将连续调用，直到读取完所有可用的条目。该函数至少会被调用两次。

  * 在以下情况下仅调用两次：

    * 第一次调用读取所有条目并返回它们的数量；
    * 第二次调用返回 0，表示没有其他条目可读取。

  * 如果第一次调用未返回总条目数，则会多次调用该函数。

该函数执行以下操作：

  #. 遍历当前目录中的条目（dentry）。
  #. 对于找到的每个 dentry，递增 ``ctx->pos``。
  #. 对于每个有效的 dentry（例如，除了 ``0`` 之外的索引节点），调用 :c:func:`dir_emit` 函数。
  #. 如果 :c:func:`dir_emit` 函数返回非零值，表示用户空间的缓冲区已满，函数将返回。

``dir_emit`` 函数的参数包括：

  * ``ctx`` 是目录遍历上下文，其作为参数传递给 ``iterate`` 函数；
  * ``name`` 是条目的名称（字符串）；
  * ``name_len`` 是条目名称的长度；
  * ``ino`` 是与条目关联的索引节点号；
  * ``type`` 标识条目类型: ``DT_REG``（文件）, ``DT_DIR``（目录）, ``DT_UNKNOWN`` 等。当条目类型未知时，可以使用 ``DT_UNKNOWN``。

.. _BitmapOperations:

位图操作
=================

在处理文件系统时，管理信息（哪个块是空闲的或忙碌的，哪个索引节点是空闲的或忙碌的）使用位图存储。为此，我们经常需要使用位操作。这些操作包括：

  * 搜索第一个为 0 的位：表示一个空闲的块或索引节点
  * 将位标记为 1：标记忙碌的块或索引节点

位图操作可以在 ``include/asm-generic/bitops`` 目录下的头文件中找到，特别是在 ``find.h`` 和 ``atomic.h`` 中。常见的函数（它们的名称指示其作用）包括：

  * :c:func:`find_first_zero_bit`
  * :c:func:`find_first_bit`
  * :c:func:`set_bit`
  * :c:func:`clear_bit`
  * :c:func:`test_and_set_bit`
  * :c:func:`test_and_clear_bit`

这些函数通常接收位图的地址，可能还有其大小（以字节为单位），如果需要，还要指定需要激活（设置）或停用（清除）的位的索引。

下面列出了一些使用示例：

.. code-block:: c

  unsigned int map;
  unsigned char array_map[NUM_BYTES];
  size_t idx;
  int changed;

  /* 在 32 位整数中找到第一个为 0 的位。 */
  idx = find_first_zero_bit(&map, 32);
  printk (KERN_ALERT "第 %zu 位是第一个为 0 的位。\n", idx);

  /* 在 NUM_BYTES 字节的数组中找到第一个为 1 的位。 */
  idx = find_first_bit(array_map, NUM_BYTES * 8);
  printk (KERN_ALERT "第 %zu 位是第一个为 1 的位。\n", idx);

  /*
   * 清除整数中的第 idx 位。
   * 假设 idx 小于整数的位数。
   */
  clear_bit(idx, &map);

  /*
   * 测试并设置数组中的第 idx 位。
   * 假设 idx 小于数组的位数。
   */
  changed = __test_and_set_bit(idx, &sbi->imap);
  if (changed)
  	printk(KERN_ALERT "%zu 位已更改\n", idx);

进一步阅读
===============

#. Robert Love《Linux 内核开发》，第二版——第 12 章 虚拟文件系统
#. 了解 Linux 内核，第 3 版——第 12 章 虚拟文件系统
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
.. _Linux 文档项目——VFS: http://www.tldp.org/LDP/tlk/fs/filesystem.html
.. _Linux 中的“虚拟文件系统”: http://www.linux.it/~rubini/docs/vfs/vfs.html
.. _Linux 文件系统教程: http://inglorion.net/documents/tutorials/tutorfs/
.. _Linux 虚拟文件系统: http://www.win.tue.nl/~aeb/linux/lk/lk-8.html
.. _Documentation/filesystems/vfs.txt: http://lxr.free-electrons.com/source/Documentation/filesystems/vfs.txt
.. _文件系统源代码: http://lxr.free-electrons.com/source/fs/

练习
=========

.. include:: ../labs/exercises-summary.hrst
.. |LAB_NAME| replace:: 文件系统

.. important::

  在本实验中，我们将继续实现之前实验中的文件系统。为此，我们将使用以下命令生成实验的框架：

  .. code-block:: console

    TODO=5 LABS=filesystems make skels

  之后，我们将从 ``TODO 5`` 开始实现。

myfs
----

在下面的练习中，我们将使用 ``myfs`` 文件系统，这是我们在上一个实验中开始实现的。我们之前挂载了文件系统，现在我们将继续进行常规文件和目录的操作。在完成这些练习之后，我们将能够创建、修改和删除常规目录和文件。

我们将主要使用 ``inode`` 和 ``dentry`` VFS 结构。 ``inode`` 结构定义了文件（可以是任何类型：常规文件、目录、链接），而 ``dentry`` 结构定义了名称，即目录中的条目。

为此，我们将访问实验框架中的 ``myfs`` 目录。之前生成的框架包含了上一个实验的解决方案；我们将从这里开始。与前一个实验一样，我们将使用 ``ramfs`` 文件系统作为起点。

1. 目录操作
^^^^^^^^^^^^^^^^^^^^^^^

首先，我们将实现用于处理目录的操作。创建文件或删除文件的操作也是目录操作；这些操作会导致添加或删除目录条目 (*dentry*)。

到本练习结束时，我们将能够在文件系统中创建和删除条目。我们还不能读取和写入常规文件；我们将在下一个练习中进行常规文件的读取和写入。

按照标有 ``TODO 5`` 的指示进行操作，这将指导你完成所需的步骤。

你需要指定以下目录操作：

  * 创建文件 (``create`` 函数)
  * 搜索 (``lookup`` 函数)
  * 链接 (``link`` 函数)
  * 创建目录 (``mkdir`` 函数)
  * 删除 (``rmdir`` 和 ``unlink`` 函数)
  * 创建节点 (``mknod``)
  * 重命名 (``rename`` 函数)

为此，请在标有 ``TODO 5`` 的位置的代码中定义 ``myfs_dir_inode_operations`` 结构。首先，只需定义 ``myfs_dir_inode_operations`` 结构；你将在下一个练习中定义 ``myfs_file_operations``, ``myfs_file_inode_operations`` 和 ``myfs_aops`` 结构。

.. tip::

  请阅读 :ref:`DirectoryInodes` 部分。

  作为参考，你可以查看 ``ramfs_dir_inode_operations`` 结构。

在 ``myfs_mkdir``, ``myfs_mknod`` 和 ``myfs_create`` 中实现 ``mkdir``, ``mknod`` 和 ``create`` 操作。这些操作将允许你在文件系统中创建目录和文件。

.. tip::

  我们建议使用 ``mknod`` 函数使代码模块化，你也可以在下一个练习中使用它。对于 inode 的读取和分配，请使用已经实现的 ``myfs_get_inode`` 函数。

  请按规范，按照文件系统 ``ramfs`` 中已实现的下列函数操作：

    * :c:func:`ramfs_mknod`
    * :c:func:`ramfs_mkdir`
    * :c:func:`ramfs_create`

  对于其他函数，请使用已在 VFS 中定义的通用调用 (``simple_*``)。

  在 ``myfs_get_inode`` 函数中，初始化目录 inode 的操作字段：

  * ``i_op`` 必须初始化为结构体 ``myfs_dir_inode_operations`` 的地址；
  * ``i_fop`` 必须初始化为在 VFS 中定义的结构体 ``simple_dir_operations`` 的地址。

.. note::

  ``i_op`` 是指向类型为 :c:type:`struct inode_operations` 的结构体指针，其中包含与 inode 相关的操作，对于目录来说，包括创建新条目、列出条目以及删除条目等。

  ``i_fop`` 是指向类型为 :c:type:`struct file_operations` 的结构体指针，其中包含和与 inode 关联的 ``file`` 结构有关的操作，例如 ``read``, ``write`` 和 ``lseek``。

测试
"""""""

完成模块后，我们可以测试文件和目录的创建。为此，我们编译内核模块（使用 ``make build``），并将生成的文件 (``myfs.ko``) 和测试脚本 (``test-myfs-{1,2}.sh``) 复制到虚拟机目录中（使用 ``make copy``）。

.. note::

  只有当测试脚本可执行时，它们才会被 ``make copy`` 复制到虚拟机中：

  .. code-block:: console

    student@workstation:~/linux/tools/labs$ chmod +x skels/filesystems/myfs/test-myfs-*.sh

启动虚拟机后，插入模块，创建挂载点并挂载文件系统：

.. code-block:: console

  # insmod myfs.ko
  # mkdir -p /mnt/myfs
  # mount -t myfs none /mnt/myfs

现在我们可以在挂载的目录 (``/mnt/myfs``) 中创建文件层次结构和子目录。我们可以使用以下类似的命令：

.. code-block:: console

  # touch /mnt/myfs/peanuts.txt
  # mkdir -p /mnt/myfs/mountain/forest
  # touch /mnt/myfs/mountain/forest/tree.txt
  # rm /mnt/myfs/mountain/forest/tree.txt
  # rmdir /mnt/myfs/mountain/forest

此时，我们无法读取或写入文件。当运行以下类似的命令时，我们将收到错误消息。

.. code-block:: console

  # echo "chocolate" > /mnt/myfs/peanuts.txt
  # cat /mnt/myfs/peanuts.txt

这是因为我们尚未实现用于处理文件的操作；我们将在后续实现。

要卸载内核模块，请使用以下命令：

.. code-block:: console

  umount /mnt/myfs
  rmmod myfs

要测试内核模块提供的功能，可以使用专用脚本 ``test-myfs-1.sh``。如果实现正确，将不会显示任何错误消息。

2. 文件操作
^^^^^^^^^^^^^^^^

我们想要实现用于处理文件的操作，这些操作用于访问文件的内容：读取、写入以及截断等。为此，你需要指定在结构体 :c:type:`struct inode_operations`、:c:type:`struct file_operations` 和 :c:type:`struct address_space_operations` 中描述的操作。

按照标记为 ``TODO`` 6 的指示进行操作，这将引导你完成所需的步骤。

首先定义 ``myfs_file_inode_operations`` 和 ``myfs_file_operations``。

.. tip::

  请阅读 :ref:`FileOperations` 部分。

  使用 VFS 提供的通用函数。

  ``ramfs`` 文件系统是一个实现示例。请参考 ``ramfs_file_inode_operations`` 和 ``ramfs_file_operations`` 的实现。

在函数 ``myfs_get_inode`` 中，为常规文件 inode 初始化操作字段：

 * ``i_op`` 必须初始化为 ``myfs_file_inode_operations``；
 * ``i_fop`` 必须初始化为 ``myfs_file_operations``。

接下来定义 ``myfs_aops`` 结构体。

.. tip::

  请阅读 :ref:`AddressSpaceOperations` 部分。

  使用 VFS 提供的通用函数。

  ``ramfs`` 文件系统是一个实现示例: ``ramfs_aops`` 结构体。

  你不需要定义 ``set_page_dirty`` 类型的函数。

将 inode 结构体的 ``i_mapping->a_ops`` 字段初始化为 ``myfs_aops``。

测试
"""""""

为了测试，我们使用前面练习中描述的步骤。除了那些步骤之外，我们现在可以使用类似以下的命令来读取、写入和修改文件：

.. code-block:: console

  # echo "chocolate" > /mnt/myfs/peanuts.txt
  # cat /mnt/myfs/peanuts.txt

要测试模块提供的功能，可以使用专用脚本：

.. code-block:: console

  # ./test-myfs-2.sh

如果实现正确，在运行上述脚本时将不会显示任何错误消息。

minfs
-----

在下面的练习中，我们将使用在上一个实验中开始开发的 minfs 文件系统。这是带有磁盘支持的文件系统。我们之前在挂载文件系统后止住脚步，现在我们将继续进行常规文件和目录的操作。在完成这些练习后，我们将能够在文件系统中创建和删除条目。

我们将主要使用 :c:type:`inode` 和 :c:type:`dentry` VFS 结构。inode 结构定义了文件（可以是任何类型：常规文件、目录、链接），而 dentry 结构定义了名称，即目录条目。

为此，我们将访问实验框架中的 ``minfs/kernel`` 目录。生成的实验框架包含了上一个实验的最终结果；我们将从这里开始。与上一个实验一样，我们将 ``minix`` 文件系统作为起点。

我们将使用 ``minfs/user`` 目录中的格式化工具 ``mkfs.minfs``，该工具通过运行 ``make build`` 来自动编译并通过 ``make copy`` 将其复制到虚拟机中的目录中。

格式化工具使用类似下面的命令来准备虚拟机磁盘：

.. code-block:: console

  # ./mkfs.minfs /dev/vdb

格式化后，磁盘的结构如下图所示：

.. image:: ../res/minfs_arch.png

如图所示, ``minfs`` 是极简的文件系统。 ``minfs`` 包含最多 32 个 inode，每个 inode 有一个数据块（文件大小限制为块大小）。超级块包含 32 位的位图 (``imap``)，每位表示一个 inode 的使用情况。

.. note::

  在开始工作之前，请仔细阅读 ``minfs/kernel/minfs.h`` 头文件。此文件包含了这些练习中将使用的结构体和宏。这些结构体和宏定义了上面图表中描述的文件系统。

1. 迭代操作
^^^^^^^^^^^^^^^^^^^^

首先，我们希望能够列出根目录的内容。为此，我们必须能够读取根目录中的条目，这意味着要实现 ``iterate`` 操作。 ``iterate`` 操作是 ``minfs_dir_operations`` 结构体（类型为 ``file_operations``）中的一个字段，由函数 ``minfs_readdir`` 实现。我们需要实现这个函数。

按照标记为 ``TODO 5`` 的位置进行操作，这将引导你完成所需的步骤。

.. tip::

  请阅读 :ref:`DirectoryInodes` 部分。

  作为起点，请参考 :c:func:`minix_readdir` 函数。该函数相当复杂，但它可以帮助你了解需要执行的步骤。

  接下来，在 ``minfs.c`` 和 ``minfs.h`` 中，查看结构体 ``struct minfs_inode_info``, ``struct minfs_inode`` 和 ``struct minfs_dir_entry`` 的定义。你将在 ``minfs_readdir`` 实现中使用它们。

获取与目录关联的 inode 和结构体 ``struct minfs_inode_info``。结构体 ``struct minfs_inode_info`` 可以帮助我们查找目录的数据块。从这个结构体中，你可以获取 ``data_block`` 字段，表示磁盘上的数据块索引。

.. tip::

  要获取结构体 ``struct minfs_inode_info``，请使用 :c:func:`list_entry` 或 :c:func:`container_of`。

使用 :c:func:`sb_bread` 读取目录的数据块。

.. tip::

  目录的数据块由与目录对应的结构体 ``struct minfs_inode_info`` 的 ``data_block`` 字段指示。

  块中的数据由 ``buffer_head`` 结构体的 ``b_data`` 字段引用（通常的代码将是 ``bh->b_data``）。该块（作为目录的数据块）包含一个数组，该数组具有最多 ``MINFS_NUM_ENTRIES`` 个条目，类型为 ``struct minfs_dir_entry`` (``minfs`` 特定的目录条目)。可以通过强制转换为 ``struct minfs_dir_entry *`` 来处理块中的数据。

在数据块中迭代所有条目，并在 ``for`` 循环中填充用户空间缓冲区。

.. tip::

  对于每个索引，通过在 ``bh->b_data`` 字段上进行指针运算，获取相应的 ``struct minfs_dir_entry`` 条目。请忽略 ``ino`` 字段等于 0 的目录条目。这样的目录条目是目录条目列表中的空槽。

  对于每个有效的条目，都用适当参数调用了 :c:func:`dir_emit`。这个调用将把 dentry 发送给调用者（然后发送到用户空间）。

  在 :c:func:`qnx6_readdir` 和 :c:func:`minix_readdir` 中查看调用示例。

测试
"""""""

完成模块后，我们可以测试列出根目录内容的功能。为此，我们编译内核模块 (``make build``)，将结果与测试脚本 (``minfs/user/test-minfs-{0,1}.sh``) 和格式化工具 (``minfs/user/mkfs.minfs``) 一起复制到虚拟机中（使用 ``make copy`` 命令），然后启动虚拟机。

.. note::

  只有当测试脚本具有可执行权限时，它们才会被复制到虚拟机中：

  .. code-block:: console

    student@eg106:~/src/linux/tools/labs$ chmod +x skels/filesystems/minfs/user/test-minfs*.sh

启动虚拟机后，我们格式化 ``/dev/vdb`` 磁盘，创建挂载点并挂载文件系统：

.. code-block:: console

  # ./mkfs.minfs /dev/vdb
  # mkdir -p /mnt/minfs
  # mount -t minfs /dev/vdb /mnt/minfs

现在，我们可以列出根目录的内容：

.. code-block:: console

  # ls -l /mnt/minfs

我们注意到已经有一个文件 (``a.txt``)；它是由格式化工具创建的。

我们还注意到，我们不能使用 ``ls`` 命令显示文件的信息。这是因为我们还没有实现 ``lookup`` 函数。我们将在下一个练习中实现它。

为了测试模块提供的功能，我们可以使用专用脚本：

.. code-block:: console

  # ./test-minfs-0.sh
  # ./test-minfs-1.sh

2. 查找操作
^^^^^^^^^^^^^^^^^^^

为了正确列出目录的内容，我们需要实现搜索功能，即 ``lookup`` 操作。 ``lookup`` 操作是 ``minfs_dir_inode_operations`` 结构体（类型为 ``inode_operations``）中的字段，由 ``minfs_lookup`` 函数实现。我们需要实现函数 ``minfs_lookup``。实际上，我们需要实现 ``minfs_lookup`` 函数调用的 ``minfs_find_entry`` 函数。

按照标记为 ``TODO 6`` 的位置提示的步骤进行操作。

.. tip::

  请阅读 :ref:`DirectoryInodes` 部分。

  作为起点，请阅读函数 :c:func:`qnx6_find_entry` 和 :c:func:`minix_find_entry`。

在 ``minfs_find_entry`` 函数中，迭代包含目标 dentry 的目录： ``dentry->d_parent->d_inode``。迭代意味着遍历目录数据块（类型为 ``struct minfs_dir_entry``）中的条目，并定位（如果存在）所请求的条目。

.. tip::

  从与目录对应的类型为 ``struct minfs_inode_info`` 的结构体中，找出数据块索引并读取它 (``sb_read``)。你需要使用 ``bh->b_data`` 访问块内容。目录数据块包含一个条目数组，该数组最多包含 ``MINFS_NUM_ENTRIES`` 个类型为 ``struct minfs_dir_entry`` 的条目。使用指针运算从数据块 (``bh->b_data``) 中获取类型为 ``struct minfs_dir_entry`` 的条目。

  检查目录中是否存在指定名称（存储在局部变量 ``name`` 中）的条目（数据块中存在一个名称等于给定名称的条目）。使用 :c:func:`strcmp` 进行验证。

  忽略 ``ino`` 字段等于 ``0`` 的目录条目。这些目录条目是目录条目列表中的空槽。

  将找到的 dentry 存储在变量 ``final_de`` 中。如果没有找到任何 dentry，则变量 ``final_de`` 的值将为 ``NULL``，即其初始化值。

在 ``minfs_lookup`` 函数中注释掉 ``simple_lookup`` 调用，以调用 ``minfs_readdir`` 的实现。

测试
"""""""

为了进行测试，我们使用前面练习中描述的步骤。列出目录（根目录）的长文件列表 (``ls -l``) 将显示权限和其他文件特定信息：

.. code-block:: console

  # ls -l /mnt/minfs

为了测试模块提供的功能，我们可以使用专用脚本：

.. code-block:: console

  # ./test-minfs-0.sh
  # ./test-minfs-1.sh

如果实现正确，运行上面的脚本时将不会显示任何错误消息。

.. note::

  在使用以下命令挂载文件系统后：

  .. code-block:: console

    # mount -t minfs /dev/vdb /mnt/minfs

  我们尝试使用以下命令创建一个文件：

  .. code-block:: console

    # touch /mnt/minfs/peanuts.txt

  这时我们遇到了错误，因为我们还没有实现创建文件的目录操作。我们将在下一个练习中实现这个功能。

3. 创建操作
^^^^^^^^^^^^^^^^^^^

为了能够在目录中创建文件，我们必须实现 ``create`` 操作。 ``create`` 操作是 ``minfs_dir_inode_operations`` 结构体（类型为 ``inode_operations``）中的字段，由 ``minfs_create`` 函数实现。我们需要实现这个函数。实际上，我们将实现 ``minfs_new_inode``（创建和初始化 inode）和 ``minfs_add_link``（为创建的 inode 添加一个链接、名称或 *dentry*）函数。

按照标记为 ``TODO 7`` 的指示进行操作。

.. tip::

  请阅读 :ref:`DirectoryInodes` 部分。

  查看 ``minfs_create`` 函数的代码和 ``minfs_new_inode``, ``minfs_add_link`` 函数的骨架。

实现函数 ``minfs_new_inode``。在这个函数中，你将使用 :c:func:`new_inode` 函数创建并初始化 inode。初始化是使用磁盘上的数据完成的。

.. tip::

  以函数 :c:func:`minix_new_inode` 为模型。在 imap (``sbi->imap``) 中找到第一个空闲的 inode。使用位操作 (``find_first_zero_bit`` 和 ``set_bit``)。请阅读 :ref:`BitmapOperations` 部分。

  将超级块的缓冲区 (``sbi->sbh``) 标记为脏。

  你必须像 ``myfs`` 文件系统中做的那样初始化常规字段。在调用 ``inode_init_owner`` 时，将 ``i_mode`` 字段初始化为 ``0``。稍后在调用者中进行初始化。

实现函数 ``minfs_add_link``。该函数将新的 dentry (``struct minfs_dir_entry``) 添加到父目录数据块 (``dentry->d_parent->d_inode``) 中。

.. tip::

  以函数 ``minix_add_link`` 为模型。

在 ``minfs_add_link`` 中，我们希望找到 dentry 的第一个空闲位置。为此，你需要迭代目录数据块，并找到第一个空闲条目。空闲的 dentry 的 ``ino`` 字段等于 ``0``。

.. tip::

  为了处理目录，获取与父目录对应的类型为 ``struct minfs_inode_info`` 的 inode（即 **dir** inode）。不要使用变量 ``inode`` 来获取 ``struct minfs_inode_info``；该 inode 属于文件，而不是你想要向其中添加链接/目录项的父目录。要获取 ``struct minfs_inode_info`` 结构体，请使用 :c:func:`container_of`。

  结构体 ``struct minfs_inode_info`` 对于查找目录数据块（由 ``dentry->d_parent->d_inode`` 指示的块，即 ``dir`` 变量）非常有用。从这个结构体中获取 ``data_block`` 字段，表示磁盘上的数据块的索引。该块包含目录中的条目。使用 :c:func:`sb_bread` 读取块，然后使用 ``bh->b_data`` 引用数据。该块最多包含 ``MINFS_NUM_ENTRIES`` 个类型为 ``struct minfs_dir_entry`` 的条目。

  如果所有条目都被占用，则返回 ``-ENOSPC``。

  使用变量 ``de`` 迭代数据块中的条目，并提取第一个空闲条目 (``ino`` 字段为 ``0``)。

  当找到空闲位置时，填充相应的条目：

    * 将 ``inode->i_ino`` 字段填写到 ``de->ino``
    * 将 ``dentry->d_name.name`` 字段填写到 ``de->name``

  然后将缓冲区标记为脏。


测试
"""""""

为了进行测试，我们使用前面练习中描述的步骤。现在我们可以在文件系统中创建文件：

.. code-block:: console

  # touch /mnt/minfs/peanuts.txt

为了测试模块提供的功能，我们可以使用专用脚本：

.. code-block:: console

  # ./test-minfs-2.sh

如果实现正确，运行上面的脚本时将不会显示任何错误消息。

.. note::

  目前的 ``minfs`` 文件系统的实现还不完整。要完善实现，还需要添加删除文件的功能、创建和删除目录的功能、重命名条目的功能以及修改文件内容的功能。
