===========================
向 Linux 内核实验项目做贡献
===========================

.. meta::
   :description: 介绍如何为 Linux 内核实验项目做出贡献的指南，包括如何 fork 仓库、创建拉取请求以及修改拉取请求等步骤
   :keywords: Linux, 内核, 实验, 贡献, fork, pull request, GitHub

``Linux 内核实验`` 是一个开放的平台。你可以通过对文档、练习或基础设施的贡献来帮助它变得更好。无论是对错别字的修正还是在文档中添加新的部分，所有的贡献我们都欢迎。

所有需要进行贡献的信息都可以在 `Linux 内核实验 Linux 仓库 <https://github.com/linux-kernel-labs/linux>`_ 中找到。如果要改变任何东西，你需要从你自己的 fork 创建一个拉取请求（``Pull Request``，``PR``）到这个仓库。PR 将由团队成员进行审核，并在解决任何可能的问题后合并。

********
仓库结构
********

`Linux 内核实验仓库 <https://github.com/linux-kernel-labs/linux>`_ 是 Linux 内核仓库的一个 fork，增加了以下内容：

  * ``/tools/labs``: 包含实验室和:ref:`虚拟机（VM）基础设施<vm_link>`

    * ``tools/labs/templates``: 包含骨架（skeleton）源码
    * ``tools/labs/qemu``: 包含 qemu 虚拟机配置

  * ``/Documentation/teaching``: 包含用于生成此文档的源码

********
构建文档
********

要构建文档，请导航到 ``tools/labs`` 并运行以下命令：

.. code-block:: bash

  make docs

.. note::
  该命令会安装所有需要的包。在某些情况下，包的安装或文档的构建可能会因为依赖版本不兼容而失败。

  与其费力去修复依赖，构建文档最简单的方法是使用 `Docker <https://www.docker.com/>`_。 首先，在你的主机上安装 ``docker`` 和 ``docker-compose``，然后运行：

  .. code-block:: bash

     make docker-docs

  第一次运行可能需要一些时间，但后续的构建会更快一些。

********
做出贡献
********

Fork 仓库
======================

1. 如果你还没有做过，请把 `linux-kernel-labs repo <https://github.com/linux-kernel-labs/linux>`_ 仓库克隆到本地：

   .. code-block:: bash

     $ mkdir -p ~/src
     $ git clone git@github.com:linux-kernel-labs/linux.git ~/src/linux

2. 前往 https://github.com/linux-kernel-labs/linux，确保你已经登录并点击页面右上角的 ``Fork``。

3. 把 fork 的仓库作为一个新的远程仓库添加到本地仓库：

   .. code-block:: bash

     $ git remote add my_fork git@github.com:<your_username>/linux.git

现在，你可以用 ``my_fork`` 来代替 ``origin`` 推送到你的分叉（例如 ``git push my_fork master``）。

创建拉取请求
===========

.. warning::

  拉取请求必须从它们自己的分支创建，这些分支是从``master``开始的。

1. 切换到 master 分支，确保没有本地更改:

  .. code-block:: bash

    student@eg106:~/src/linux$ git checkout master
    student@eg106:~/src/linux$ git status
    On branch master
    Your branch is up-to-date with 'origin/master'.
    nothing to commit, working directory clean


2. 确保本地 master 分支与 linux-kernel-labs 同步:

  .. code-block:: bash

    student@eg106:~/src/linux$ git pull origin master

  .. note::

    你也可以将最新的 master 推送到你 fork 的仓库:

    .. code-block:: bash

      student@eg106:~/src/linux$ git push my_fork master

3. 为你的更改创建一个新分支：

  .. code-block:: bash

    student@eg106:~/src/linux$ git checkout -b <your_branch_name>

4. 做一些更改并提交。在这个例子中，我们将更改 ```Documentation/teaching/index.rst``：

  .. code-block:: bash

    student@eg106:~/src/linux$ vim Documentation/teaching/index.rst
    student@eg106:~/src/linux$ git add Documentation/teaching/index.rst
    student@eg106:~/src/linux$ git commit -m "<commit message>"

  .. warning::

    提交信息必须包含对更改的相关描述以及已更改组件的位置。

    示例:

      * ``documentation: index: 修正第一节错别字``
      * ``labs: block_devices: 更改 printk 日志级别``

5. 将本地分支推送到你 fork 的仓库:

  .. code-block:: bash

    student@eg106:~/src/linux$ git push my_fork <your_branch_name>

6. 打开拉取请求

  * 转到 https://github.com 并打开你 fork 的仓库页面
  * 点击 ``New pull request``。
  * 确保基础仓库（左侧）是 ``linux-kernel-labs/linux``，基础是 master。
  * 确保头部仓库（右侧）是你的 fork 仓库，比较分支是你推送的分支。
  * 点击 ``Create pull request``。

修改拉取请求
===========

在收到对你的更改的反馈后，你可能需要更新拉取请求。你应该对同一分支进行新的推送。为此，请按照以下步骤操作：

1. 确保你的分支仍然与 ``linux-kernel-labs`` 仓库的 ``master`` 分支同步。

  .. code-block:: bash

    student@eg106:~/src/linux$ git fetch origin master
    student@eg106:~/src/linux$ git rebase FETCH_HEAD

  .. note::

    如果你遇到冲突，这意味着其他人修改了与你相同的文件/行，并且在你打开拉取请求后已经合并了更改。

    在这种情况下，你需要通过手动编辑冲突文件来修复冲突（运行 ``git status`` 查看这些文件）。
    修复冲突后，使用 ``git add`` 添加它们，然后运行 ``git rebase --continue``。


2. 将更改应用于本地文件
3. 提交更改。我们希望所有更改都在同一提交中，所以我们将更改合并到初始提交中。

  .. code-block:: bash

    student@eg106:~/src/linux$ git add Documentation/teaching/index.rst
    student@eg106:~/src/linux$ git commit --amend

4. 强制推送更新的提交:

  .. code-block:: bash

    student@eg106:~/src/linux$ git push my_fork <your_branch_name> -f

  此步骤之后，拉取请求将被更新。现在由 linux-kernel-labs 小组来审查拉取请求并将你的贡献集成到主项目中。
