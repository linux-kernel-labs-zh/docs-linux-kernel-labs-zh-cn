=========================
作业 0——内核 API
=========================

-  截止日期: :command:`2023 年 3 月 26 日，23:00`

作业目标
=======================

*  熟悉 qemu 的设置
*  加载/卸载内核模块
*  熟悉内核中实现的列表 API
*  玩得开心 :)

说明
=========

编写名为 `list` 的内核模块（生成的文件必须命名为 `list.ko`），该模块在内部列表中存储数据（字符串）。

必须使用内核中实现的 `列表 API <https://github.com/torvalds/linux/blob/master/include/linux/list.h>`__。有关详细信息，请参阅 `实验 2 </so2/lab2-kernel-api.html>`__。

该模块向 procfs 导出名为 :command:`list` 的目录。该目录包含两个文件：

-   :command:`management`：只能写入；用于向内核模块传输命令的接口
-   :command:`preview`：只读；用于查看内核列表的内部内容的接口。

`代码骨架 <https://github.com/linux-kernel-labs/linux/blob/master/tools/labs/templates/assignments/0-list/list.c>`__ 实现了这两个 procfs 文件。你需要创建一个列表，并实现对数据的 `添加` 和 `读取` 支持。请按照代码中的 TODO 进行操作。

要与内核列表进行交互，你必须在 `/proc/list/management` 文件中编写命令（使用 `echo` 命令）：

- `addf name`：将 `name` 元素添加到列表的顶部
- `adde name`：将 `name` 元素添加到列表的末尾
- `delf name`：删除列表中第一个 `name` 项
- `dela name`：删除列表中所有的 `name` 元素

要想查看列表内容，可以查看 `/proc/list/preview` 文件的内容（使用 `cat` 命令）。格式为每行一个元素。

测试
=======

为了简化作业评估过程，同时也为了减少提交作业时的错误，作业评估将通过一个名为 `_checker` 的 `测试脚本 <https://github.com/linux-kernel-labs/linux/blob/master/tools/labs/templates/assignments/0-list/checker/_checker>`__ 自动进行。测试脚本假定内核模块名为 `list.ko`。

快速开始
==========

你必须从 `list.c <https://gitlab.cs.pub.ro/so2/0-list/-/blob/master/src/list.c>`__ 文件中找到的代码骨架开始实现作业。你应该按照 `任务仓库 <https://gitlab.cs.pub.ro/so2/0-list>`__ 的 `README.md 文件 <https://gitlab.cs.pub.ro/so2/0-list/-/blob/master/README.md>`__ 中的说明进行操作。

提示
----

要想增加获得最高分的机会，请阅读并遵循 Linux 内核编码风格，该风格在 `代码风格文档 <https://elixir.bootlin.com/linux/v4.19.19/source/Documentation/process/coding-style.rst>`__ 中有描述。

此外，使用以下静态分析工具来验证代码：

- checkpatch.pl

.. code-block:: console

   $ linux/scripts/checkpatch.pl --no-tree --terse -f /path/to/your/list.c

- sparse

.. code-block:: console

   $ sudo apt-get install sparse
   $ cd linux
   $ make C=2 /path/to/your/list.c

- cppcheck

.. code-block:: console

   $ sudo apt-get install cppcheck
   $ cppcheck /path/to/your/list.c

扣分项
---------

有关作业扣分项的信息可以在 `基本说明页面 <https://ocw.cs.pub.ro/courses/so2/teme/general>`__ 中找到。

在特殊情况下（作业通过测试，但不符合要求）以及如果作业未通过所有测试，成绩可能会降低更多。

提交作业
------------------------

使用 `vmchecker-next <https://github.com/systems-cs-pub-ro/vmchecker-next/wiki/Student-Handbook>`__ 基础设施自动对作业进行评分。提交将在 moodle 上的 `课程页面 <https://curs.upb.ro/2022/course/view.php?id=5121>`__ 上与相关作业相关联。你可以在 `仓库 <https://gitlab.cs.pub.ro/so2/0-list/-/blob/master>`__ 的 `README.md 文件 <https://gitlab.cs.pub.ro/so2/0-list/-/blob/master/README.md>`__ 中找到提交详细信息。

资源
=========

我们建议你使用 gitlab 存储作业。请按照 `README.md 文件 <https://gitlab.cs.pub.ro/so2/0-list/-/blob/master/README.md>`__ 中的说明进行操作。

问题
=========

如果你有相关的问题，你可以查阅邮件 `列表存档 <http://cursuri.cs.pub.ro/pipermail/so2/>`__，或在专用的 Teams 频道上提出问题。
