=============
合作
=============

在开源世界中，合作是至关重要的，我们鼓励你选择一个团队合作伙伴来共同完成选定的任务。

以下是一个简单的指南，帮助你入门：

1. 使用 Github / Gitlab
----------------------

在团队内部共享你的工作成果的最佳方式是使用版本控制系统（VCS）来跟踪每一次更改。请注意，你必须将你的存储库设置为私有，并仅授予团队成员读/写访问权限。

2. 从任务的骨架开始
-------------------------------------------

添加 `init`/`exit` 函数、驱动程序操作和可能需要的全局结构。

.. code-block:: c

  // SPDX-License-Identifier: GPL-2.0
  /*
   * uart16550.c——UART16550 驱动程序
   *
   * 作者：John Doe <john.doe@mail.com>
   * 作者：Ionut Popescu <ionut.popescu@mail.com>
   */
  struct uart16550_dev {
     struct cdev cdev;
     /*TODO */
  };

  static struct uart16550_dev devs[MAX_NUMBER_DEVICES];

  static int uart16550_open(struct inode *inode, struct file *file)
  {
      /*TODO */
      return 0;
  }

  static int uart16550_release(struct inode *inode, struct file *file)
  {
     /*TODO */
     return 0;
  }

  static ssize_t uart16550_read(struct file *file,  char __user *user_buffer,
                                size_t size, loff_t *offset)
  {
        /*TODO */
  }

  static ssize_t uart16550_write(struct file *file,
                                 const char __user *user_buffer,
                                 size_t size, loff_t *offset)
  {
       /*TODO */
  }

  static long
  uart16550_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
  {
        /*TODO */
        return 0;
  }

  static const struct file_operations uart16550_fops = {
         .owner		 = THIS_MODULE,
         .open		 = uart16550_open,
         .release	 = uart16550_release,
         .read		 = uart16550_read,
         .write		 = uart16550_write,
         .unlocked_ioctl = uart16550_ioctl
  };

  static int __init uart16550_init(void)
  {
    /* TODO: */
  }

  static void __exit uart16550_exit(void)
  {
     /* TODO: */
  }

  module_init(uart16550_init);
  module_exit(uart16550_exit);

  MODULE_DESCRIPTION("UART16550 Driver");
  MODULE_AUTHOR("John Doe <john.doe@mail.com");
  MODULE_AUTHOR("Ionut Popescu <ionut.popescu@mail.com");

3. 为每个单独的更改添加提交
---------------------------------------

首个提交必须是骨架文件。而其余的代码应该在骨架文件的基础上进行添加。请认真写提交消息。简要解释该提交的内容以及 *为什么* 它是必要的。

遵循良好提交消息的七个规则：[点击这里](https://cbea.ms/git-commit/#seven-rules)

.. code-block:: console

  Commit 3c92a02cc52700d2cd7c50a20297eef8553c207a (HEAD -> tema2)
  Author: John Doe <john.doe@mail.com>
  Date:   Mon Apr 4 11:54:39 2022 +0300

    uart16550：为任务 #2 添加初始骨架

    这个提交添加了 uart16550 任务的简单骨架文件。请注意模块的 init/exit 回调和 open/release/read/write/ioctl 的 file_operation 的虚拟实现。

    Signed-off-by: John Doe <john.doe@mail.com>

4. 在团队内拆分工作
----------------------------

将每个团队成员的任务添加进 `TODO`。尽量平均地拆分工作。

在开始编码之前，制定一个计划。在骨架文件的顶部，将每个成员的任务添加进 `TODO`。就全局结构和整体驱动程序设计达成一致。然后开始编码。

5. 进行代码审查
------------------------

创建包含你的提交的 PR，并与团队成员进行审查。你可以参考 `创建拉取请求教程` 的 `视频 <https://www.youtube.com/watch?v=YvoHJJWvn98>`_。

6. 合并工作
--------------------

最终的工作是合并所有 PR 的结果。根据提交消息，他人应该能够清楚地了解代码的进展以及在团队内工作是如何管理的。

.. code-block:: console

  f5118b873294 uart16550: Add uart16550_interrupt implementation
  2115503fc3e3 uart16550: Add uart16550_ioctl implementation
  b31a257fd8b8 uart16550: Add uart16550_write implementation
  ac1af6d88a25 uart16550: Add uart16550_read implementation
  9f680e8136bf uart16550: Add uart16550_open/release implementation
  3c92a02cc527 uart16550: Add skeleton for SO2 assignment #2
