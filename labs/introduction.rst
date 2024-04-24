====
介绍
====

.. meta::
   :description: 介绍操作系统 2 实验的规则和目标，介绍实验文档，介绍 Linux 内核及相关资源
   :keywords: 内核, 内核编程, Linux, cscope, LXR, gdb, addr2line, dump_stack

实验目标
=======

* 介绍操作系统 2 实验的规则和目标
* 介绍实验文档
* 介绍 Linux 内核及相关资源

关键词
======

* 内核，内核编程
* Linux，标准版（vanilla），http://www.kernel.org
* cscope，LXR
* gdb，/proc/kcore，addr2line，dump_stack

..
  _[SECTION-ABOUT-BEGIN]

关于本实验
=========

操作系统 2 实验是内核编程和驱动程序开发实验。本实验的目标是：

* 加深课程中介绍的概念
* 展示内核编程接口（内核 API）
* 获取在独立的环境中记录、开发和调试的技能
* 获取驱动程序开发的知识和技能

每个实验将呈现一组特定问题的概念、应用和命令。实验将以演示开始（每个实验都会有一组幻灯片）（15 分钟），其余时间将用于实验练习（80 分钟）。

为了获得最佳的实验效果，我们建议你阅读相关幻灯片。要完全理解实验，我们建议你查阅实验技术支持材料。如果需要深入学习，你可以使用辅助文档。

..
  _[SECTION-ABOUT-END]

..
  _[SECTION-REFERENCES-BEGIN]

参考资料
========

-  Linux

   -  `Linux 内核开发（第 3 版） <http://www.amazon.com/Linux-Kernel-Development-Robert-Love/dp/0672329468/>`__
   -  `Linux 设备驱动（第 3 版） <http://free-electrons.com/doc/books/ldd3.pdf>`__
   -  `精通 Linux 设备驱动程序 <http://www.amazon.com/Essential-Device-Drivers-Sreekrishnan-Venkateswaran/dp/0132396556>`__

-  通用

   -  `邮件列表 <http://cursuri.cs.pub.ro/cgi-bin/mailman/listinfo/pso>`__ (`检索邮件列表 <http://blog.gmane.org/gmane.education.region.romania.operating-systems-design>`__)

..
  _[SECTION-REFERENCES-END]

..
  _[SECTION-CODE-NAVIGATION-BEGIN]

源代码导航
=========

.. _cscope_intro:

cscope
------

`Cscope <http://cscope.sourceforge.net/>`__ 是一个用于高效导航 C 源代码的工具。要使用它，你必须从现有的源代码生成 cscope 数据库。在 Linux 树中，执行命令 :command:`make ARCH=x86 cscope` 就足够了。尽管不是必须通过 ARCH 变量指定架构，但建议这样做；否则，一些依赖于特定架构的函数会在数据库中出现多次。

你可以使用命令 :command:`make ARCH=x86 COMPILED_SOURCE=1 cscope` 来构建 cscope 数据库。这样，cscope 数据库中只会包含在编译过程中使用过的符号（symbol），从而在搜索符号时可以获得更好的性能。

Cscope 也可以作为独立工具使用，但与编辑器结合使用会更加有用。要在 :command:`vim` 中使用 cscope，你需要安装这两个软件包，并在文件 :file:`.vimrc` 中添加以下几行（实验中的机器已经进行了配置）：

.. code-block:: vim

    if has("cscope")
            " Look for a 'cscope.out' file starting from the current directory,
            " going up to the root directory.
            let s:dirs = split(getcwd(), "/")
            while s:dirs != []
                    let s:path = "/" . join(s:dirs, "/")
                    if (filereadable(s:path . "/cscope.out"))
                            execute "cs add " . s:path . "/cscope.out " . s:path . " -v"
                            break
                    endif
                    let s:dirs = s:dirs[:-2]
            endwhile

            set csto=0  " Use cscope first, then ctags
            set cst     " Only search cscope
            set csverb  " Make cs verbose

            nmap `<C-\>`s :cs find s `<C-R>`=expand("`<cword>`")`<CR>``<CR>`
            nmap `<C-\>`g :cs find g `<C-R>`=expand("`<cword>`")`<CR>``<CR>`
            nmap `<C-\>`c :cs find c `<C-R>`=expand("`<cword>`")`<CR>``<CR>`
            nmap `<C-\>`t :cs find t `<C-R>`=expand("`<cword>`")`<CR>``<CR>`
            nmap `<C-\>`e :cs find e `<C-R>`=expand("`<cword>`")`<CR>``<CR>`
            nmap `<C-\>`f :cs find f `<C-R>`=expand("`<cfile>`")`<CR>``<CR>`
            nmap `<C-\>`i :cs find i ^`<C-R>`=expand("`<cfile>`")`<CR>`$`<CR>`
            nmap `<C-\>`d :cs find d `<C-R>`=expand("`<cword>`")`<CR>``<CR>`
            nmap <F6> :cnext <CR>
            nmap <F5> :cprev <CR>

            " Open a quickfix window for the following queries.
            set cscopequickfix=s-,c-,d-,i-,t-,e-,g-
    endif

该脚本会在当前目录或父目录中搜索名为 :file:`cscope.out` 的文件。如果 :command:`vim` 找到该文件，你可以使用快捷键 :code:`Ctrl + ]` 或 :code:`Ctrl+\ g` (按下 control-\\ 然后按 g) 直接跳转到光标所在单词的定义（函数、变量、结构等）。类似地，你可以使用 :code:`Ctrl+\ s` 前往光标所在单词的使用位置。

你可以从以下网址获取启用了 cscope 的 :file:`.vimrc` 文件（还包含其他好用的东西）：https://github.com/ddvlad/cfg/blob/master/\_vimrc。以下指南基于该文件，同时也展示了具有相同效果的基本 :command:`vim` 命令。

如果有多个结果（通常会有），你可以使用 :code:`F6` 和 :code:`F5` （:code:`:ccnext` 和 :code:`:cprev`）在它们之间切换。你还可以使用命令 :code:`:copen` 打开一个新的面板来显示结果。要关闭面板，可以使用 :code:`:cclose` 命令。

要返回到先前的位置，可以使用 :code:`Ctrl+o` (是字母 o，不是零)。该命令可以多次使用，即使 cscope 更改了你当前正在编辑的文件也有效。

要在 :command:`vim` 启动时直接跳转到符号定义，可以使用 :code:`vim -t <symbol_name>` (例如 :code:`vim -t task_struct`)。如果你已经启动了 :command:`vim` 并想按名称搜索符号，可以使用 :code:`cs find g <symbol_name>` (例如 :code:`cs find g task_struct`)。

如果你找到了多个结果，并且用 :code:`:copen` 命令打开了一个显示所有匹配项的面板，如果你想在面板中找到某种结构类型的符号，建议你用 :code:`/` ——斜杠命令在面板中搜索字符 :code:`{` (左花括号)。

.. important::
    你可以使用命令 :command:`:cs help` 获取所有 :command:`cscope` 命令的摘要。

    若要了解更多信息，请使用 :command:`vim` 内置的帮助命令：:command:`:h cscope` 或 :command:`:h copen`。

如果你使用 :command:`emacs`，请安装 :command:`xcscope-el` 包，并在 :file:`~/.emacs` 文件中添加以下行。

.. code-block:: vim

    (require ‘xcscope)
    (cscope-setup)

这些命令将自动为 C 和 C++ 模式激活 cscope。:code:`C-s s` 是按键绑定前缀，:code:`C-s s s` 用于搜索符号（如果光标位置在单词上，调用它时将使用该位置的单词）。有关详细信息，请查看 `https://github.com/dkogan/xcscope.el`。

clangd
------

`Clangd <https://clangd.llvm.org/>`__ 是一个语言服务器，提供了一些用于浏览 C 和 C++ 代码的工具。`语言服务器协议 <https://microsoft.github.io/language-server-protocol/>`__ 利用语义全项目分析，实现了诸如跳转到定义、查找引用、悬停提示、代码补全等功能。

Clangd 需要一个编译数据库来理解内核源代码。可以通过以下方式生成编译数据库：

.. code-block:: bash

    make defconfig
    make
    scripts/clang-tools/gen_compile_commands.py

LSP 客户端：

- Vim/Neovim（ `coc.nvim <https://github.com/neoclide/coc.nvim>`__、 `nvim-lsp <https://github.com/neovim/nvim-lspconfig>`__、 `vim-lsc <https://github.com/natebosch/vim-lsc>`__ 以及 `vim-lsp <https://github.com/prabirshrestha/vim-lsp>`__ ）
- Emacs（ `lsp-mode <https://github.com/emacs-lsp/lsp-mode>`__ ）
- VSCode（ `clangd extension <https://marketplace.visualstudio.com/items?itemName=llvm-vs-code-extensions.vscode-clangd>`__ ）

Kscope
------

如果想要更简单的界面的话，可以尝试 Kscope。`Kscope <http://sourceforge.net/projects/kscope/>`__ 是一个使用 QT 的 cscope 前端。它轻便、快速、易用。它支持使用正则表达式、调用图等方式进行搜索。Kscope 已经停止维护了。

还有一个适用于 Qt4 和 KDE 4 的 `移植版本 <https///opendesktop.org/content/show.php/Kscope4?content=156987>`__ ，其保留了与文本编辑器 Kate 的集成，并且比 SourceForge 上的最新版本更易于使用。

LXR Cross-Reference
-------------------

LXR（LXR Cross-Reference）是一种工具，允许使用 Web 界面来索引和引用程序源代码中的符号。Web 界面显示了符号在文件中定义或使用的位置的链接。LXR 的开发网站是 http://sourceforge.net/projects/lxr。类似的工具有 `OpenGrok <http://oracle.github.io/opengrok/>`__ 和 `Gonzui <http://en.wikipedia.org/wiki/Gonzui>`__。

尽管 LXR 最初是用于 Linux 内核源代码的，但也用于 `Mozilla <http://lxr.mozilla.org/>`__、 `Apache HTTP 服务器 <http://apache.wirebrain.de/lxr/>`__ 和 `FreeBSD <http://lxr.linux.no/freebsd/source>`__ 的源代码。

有许多网站使用 LXR 来进行 Linux 内核源代码的交叉引用，主要网站是 `开发原址 <http://lxr.linux.no/linux/>`__，然而该网站已不再运作。你可以使用 `https://elixir.bootlin.com/ <https://elixir.bootlin.com/>`__。

LXR 允许在任意文本或文件名上搜索标识符（符号）。它提供的主要特点和优势是可以轻松地找到任何全局标识符的声明。这样，它便于快速访问函数声明、变量、宏定义，以及轻松地浏览代码。此外，它还能够检测当变量或函数发生变化时，哪些代码区域会受到影响，这对于开发和调试阶段是一个真正的优势。

SourceWeb
---------

`SourceWeb <http://rprichard.github.io/sourceweb/>`__ 是一个用于 C 和 C++ 的源代码索引器。它使用 Clang 编译器提供的 `框架 <http://clang.llvm.org/docs/IntroductionToTheClangAST.html>`__ 来索引代码。

cscope 和 SourceWeb 之间的主要区别在于，SourceWeb 在某种程度上是一个编译器插件。SourceWeb 不会索引所有的代码，而只会索引实际被编译器编译的代码。这样的话，一些问题就没有了，例如在多个位置定义的函数变体中的的哪个被使用的歧义。这也意味着索引需要更多的时间，因为编译后的文件必须再次通过索引器生成引用。

使用示例：

.. code-block:: bash

    make oldconfig
    sw-btrace make -j4
    sw-btrace-to-compile-db
    sw-clang-indexer --index-project
    sourceweb index

:file:`sw-btrace` 是一个添加 :file:`libsw-btrace.so` 库到 :code:`LD_PRELOAD` 的脚本。这样，该库将被 :code:`make` 启动的每个进程（基本上是编译器）加载， 注册用于启动进程的命令，并生成一个名为 :file:`btrace.log` 的文件。然后，:code:`sw-btrace-to-compile-db` 使用该文件将其转换为 clang 定义的格式： `JSON Compilation Database <http://clang.llvm.org/docs/JSONCompilationDatabase.html>`__ 。 然后上述步骤生成的 JSON 编译数据库由索引器使用，索引器通过已编译的源文件再进行一次遍历，生成 GUI 使用的索引。

建议：不要对正在使用的源代码进行索引，而是使用其副本，因为 SourceWeb 目前没有单独重新生成单个文件的索引的功能，你将不得不重新生成完整的索引。

..
  _[SECTION-CODE-NAVIGATION-END]

..
  _[SECTION-DEBUGGING-BEGIN]

内核调试
========

与调试程序相比，调试内核更加困难，因为操作系统没有提供支持。这就是为什么通常使用两台通过串行接口相互连接的计算机进行此过程。

.. _gdb_intro:

gdb（Linux）
-----------

在 Linux 上，一种更简单但也具有许多缺点的调试方法是使用 `gdb <http://www.gnu.org/software/gdb/>`__ 进行本地调试，其中涉及到未压缩的内核镜像（:file:`vmlinux` ）和文件：:file:`/proc/kcore` （实时内核镜像）。这种方法通常用于检查内核并在其运行时检测特定的不一致性。特别是如果内核是使用 :code:`-g` 选项编译的（该选项会保留调试信息）这种方法就非常有用。但是，这种方法无法使用一些常用的调试技术，例如数据修改的断点。

.. note:: 因为 :file:`/proc` 是一个虚拟文件系统，:file:`/proc/kcore` 在磁盘上并不存在。当程序尝试访问 :file:`/proc/kcore` 时，内核会即时生成它。它用于调试目的。

          根据 :command:`man proc` 的说明：

          ::

              /proc/kcore
              此文件代表系统的物理内存，并以 ELF 核心文件格式存储。借助这个伪文件（pseudo-file）和未剥离（unstripped）的内核（/usr/src/linux/vmlinux）二进制文件，可以使用 GDB 来检查任何内核数据结构的当前状态。

未压缩的内核镜像提供关于其中所包含的数据结构和符号的信息。

.. code-block:: bash

    student@eg106$ cd ~/src/linux
    student@eg106$ file vmlinux
    vmlinux: ELF 32-bit LSB executable, Intel 80386, ...
    student@eg106$ nm vmlinux | grep sys_call_table
    c02e535c R sys_call_table
    student@eg106$ cat System.map | grep sys_call_table
    c02e535c R sys_call_table

:command:`nm` 程序用于显示对象或可执行文件中的符号。在我们的例子中，:file:`vmlinux` 是一个 ELF 文件。或者，我们可以使用文件 :file:`System.map` 来查看内核中的符号信息。

然后，我们使用 :command:`gdb` 来使用未压缩的内核镜像检查这些符号。一个简单的 :command:`gdb` 会话如下所示：

.. code-block:: bash

    student@eg106$ cd ~/src/linux
    stduent@eg106$ gdb --quiet vmlinux
    Using host libthread_db library "/lib/tls/libthread_db.so.1".
    (gdb) x/x 0xc02e535c
    0xc02e535c `<sys_call_table>`:    0xc011bc58
    (gdb) x/16 0xc02e535c
    0xc02e535c `<sys_call_table>`:    0xc011bc58      0xc011482a      0xc01013d3     0xc014363d
    0xc02e536c `<sys_call_table+16>`: 0xc014369f      0xc0142d4e      0xc0142de5     0xc011548b
    0xc02e537c `<sys_call_table+32>`: 0xc0142d7d      0xc01507a1      0xc015042c     0xc0101431
    0xc02e538c `<sys_call_table+48>`: 0xc014249e      0xc0115c6c      0xc014fee7     0xc0142725
    (gdb) x/x sys_call_table
    0xc011bc58 `<sys_restart_syscall>`:       0xffe000ba
    (gdb) x/x &sys_call_table
    0xc02e535c `<sys_call_table>`:    0xc011bc58
    (gdb) x/16 &sys_call_table
    0xc02e535c `<sys_call_table>`:    0xc011bc58      0xc011482a      0xc01013d3     0xc014363d
    0xc02e536c `<sys_call_table+16>`: 0xc014369f      0xc0142d4e      0xc0142de5     0xc011548b
    0xc02e537c `<sys_call_table+32>`: 0xc0142d7d      0xc01507a1      0xc015042c     0xc0101431
    0xc02e538c `<sys_call_table+48>`: 0xc014249e      0xc0115c6c      0xc014fee7     0xc0142725
    (gdb) x/x sys_fork
    0xc01013d3 `<sys_fork>`:  0x3824548b
    (gdb) disass sys_fork
    Dump of assembler code for function sys_fork:
    0xc01013d3 `<sys_fork+0>`:        mov    0x38(%esp),%edx
    0xc01013d7 `<sys_fork+4>`:        mov    $0x11,%eax
    0xc01013dc `<sys_fork+9>`:        push   $0x0
    0xc01013de `<sys_fork+11>`:       push   $0x0
    0xc01013e0 `<sys_fork+13>`:       push   $0x0
    0xc01013e2 `<sys_fork+15>`:       lea    0x10(%esp),%ecx
    0xc01013e6 `<sys_fork+19>`:       call   0xc0111aab `<do_fork>`
    0xc01013eb `<sys_fork+24>`:       add    $0xc,%esp
    0xc01013ee `<sys_fork+27>`:       ret
    End of assembler dump.

可以注意到未压缩的内核映像被用作 :command:`gdb` 的参数。在编译后，可以在内核源代码的根目录中找到该映像。

使用 :command:`gdb` 进行调试的几个命令如下：

- :command:`x` （examine）——用于显示指定地址的内存区域的内容（该地址可以是物理地址的值、符号或符号的地址）。它可以接受以下参数（以 :code:`/` 开头）：要显示数据的格式（:code:`x` 表示十六进制，:code:`d` 表示十进制，等等）、要显示的内存单元（memory unit）数量以及单个内存单元的大小。

- :command:`disassemble` ——用于反汇编函数。

- :command:`p` （print）——用于评估并显示表达式的值。可以通过参数指定要显示数据的格式（:code:`/x` 表示十六进制，:code:`/d` 表示十进制，等等）。

对内核映像的分析是一种静态分析方法。如果我们想进行动态分析（分析内核的运行情况，而不仅仅是静态映像），我们可以使用 :file:`/proc/kcore`；这是内核的动态映像（存储在内存中）。

.. code-block:: bash

    student@eg106$ gdb ~/src/linux/vmlinux /proc/kcore
    Core was generated by `root=/dev/hda3 ro'.
    #0  0x00000000 in ?? ()
    (gdb) p sys_call_table
    $1 = -1072579496
    (gdb) p /x sys_call_table
    $2 = 0xc011bc58
    (gdb) p /x &sys_call_table
    $3 = 0xc02e535c
    (gdb) x/16 &sys_call_table
    0xc02e535c `<sys_call_table>`:    0xc011bc58      0xc011482a      0xc01013d3     0xc014363d
    0xc02e536c `<sys_call_table+16>`: 0xc014369f      0xc0142d4e      0xc0142de5     0xc011548b
    0xc02e537c `<sys_call_table+32>`: 0xc0142d7d      0xc01507a1      0xc015042c     0xc0101431
    0xc02e538c `<sys_call_table+48>`: 0xc014249e      0xc0115c6c      0xc014fee7     0xc0142725

使用内核的动态镜像有助于检测 `rootkit <http://zh.wikipedia.org/wiki/Rootkit>`__ 。

- `Linux设备驱动程序第 3 版——调试器和相关工具 <http://linuxdriver.co.il/ldd3/linuxdrive3-CHP-4-SECT-6.html>`__
- `在 Linux 中检测 Rootkit 和内核级入侵 <http://www.securityfocus.com/infocus/1811>`__
- `用户模式 Linux <http://user-mode-linux.sf.net/>`__

获取堆栈跟踪
-----------

有时，你需要获取有关执行路径到达某个特定点的信息。你可以使用 :command:`cscope` 或 LXR 来确定这些信息，但某些函数从许多执行路径调用，这使得这种方法变得困难。

在这些情况下，使用函数 :code:`dump_stack()` 获取堆栈跟踪非常有用。

..
  _[SECTION-DEBUGGING-END]

..
  _[SECTION-DOCUMENTATION-BEGIN]

文档
====

与用户空间编程相比，内核开发是一个困难的过程。内核的 API 和用户空间不同，内核子系统的复杂性也更高，因此需要额外的准备工作。相关的文档比较零散，有时候需要查阅多个来源才能对某个方面有较全面的了解。

Linux 内核的主要优势是可以访问源代码和其开放式开发系统。因此，互联网上存在大量的内核相关文档。

以下是与 Linux 内核相关的一些链接：

- `KernelNewbies <http://kernelnewbies.org>`__
- `KernelNewbies——内核编程 <http://kernelnewbies.org/KernelHacking>`__
- `内核分析——HOWTO <http://www.tldp.org/HOWTO/KernelAnalysis-HOWTO.html>`__
- `Linux 内核编程 <http://web.archive.org/web/20090228191439/http://www.linuxhq.com/lkprogram.html>`__
- `Linux 内核——Wikibooks <http://en.wikibooks.org/wiki/Linux_kernel>`__

这些链接并不全面。使用 `互联网 <http://www.google.com>`__ 和 `内核源代码 <http://lxr.free-electrons.com/>`__ 是必不可少的。

..
  _[SECTION-DOCUMENTATION-END]

练习
====

..
  _[SECTION-EXERCISES-REMARKS-BEGIN]

备注
----

.. note::

  -  通常，开发内核模块的步骤如下：

     -  编辑模块源代码（在物理机上）；
     -  编译模块（在物理机上）；
     -  生成用于虚拟机的最小镜像；该镜像包含内核、你的模块、busybox 以及测试程序；
     -  使用 QEMU 启动虚拟机；
     -  在虚拟机中运行测试。

  -  当使用 cscope 时，请使用文件 :file:`~/src/linux`。如果没有文件 :file:`cscope.out`，可以使用命令 :command:`make ARCH=x86 cscope` 来生成它。

  -  你可以在 :ref:`vm_link` 找到有关虚拟机的更多详细信息。

.. important::
    在解决练习之前, 请 **仔细** 阅读所有要点。

..
  _[SECTION-EXERCISES-REMARKS-END]

..
  _[EXERCISE1-BEGIN]

启动虚拟机
---------

虚拟机基础设施摘要：

-  :file:`~/src/linux` ——Linux 内核源代码，用于编译模块。该目录包含文件 :file:`cscope.out`，用于在源代码树中导航。

-  :file:`~/src/linux/tools/labs/qemu` ——用于生成和运行 QEMU 虚拟机的脚本和辅助文件。

要启动虚拟机，请在目录 :file:`~/src/linux/tools/labs` 中运行 :command:`make boot`：

.. code-block:: shell

    student@eg106:~$ cd ~/src/linux/tools/labs
    student@eg106:~/src/linux/tools/labs$ make boot

默认情况下，你不会获得提示符或任何图形界面，但你可以使用 :command:`minicom` 或 :command:`screen` 连接到虚拟机提供的控制台。

.. code-block:: shell

    student@eg106:~/src/linux/tools/labs$ minicom -D serial.pts

    <按回车键>

    qemux86 login:
    Poky (Yocto Project Reference Distro) 2.3 qemux86 /dev/hvc0

另外，也可以使用命令 :command:`QEMU_DISPLAY=gtk make boot` 启动虚拟机，这种情况下虚拟机带有图形界面支持。

.. note::
    要访问虚拟机，请在登录提示符处输入用户名 :code:`root`；无需输入密码。虚拟机将以 root 帐户的权限启动。

..
  _[EXERCISE1-END]

..
  _[EXERCISE2-BEGIN]

添加和使用虚拟磁盘
-----------------

.. note:: 如果你没有文件 :file:`mydisk.img`，你可以从地址 http://elf.cs.pub.ro/so2/res/laboratoare/mydisk.img 下载它。该文件必须放在 :file:`tools/labs` 目录下。

在 :file:`~/src/linux/tools/labs` 目录下，有一个新的虚拟机磁盘，文件名为 :file:`mydisk.img`。我们需要将该磁盘添加到虚拟机并在虚拟机中使用它。

编辑 :file:`qemu/Makefile` 文件，在 :code:`QEMU_OPTS` 变量中添加 :code:`-drive file=mydisk.img,if=virtio,format=raw`。

.. note:: qemu 中已经添加了两个磁盘（disk1.img 和 disk2.img）。你需要在它们之后添加新的磁盘。在这种情况下，新的磁盘可以通过 :file:`/dev/vdd` 访问（vda 是根分区，vdb 是 disk1，vdc 是 disk2）。

.. hint:: 你不需要在 :file:`/dev` 中手动创建新磁盘的条目，因为虚拟机使用的是 :command:`devtmpfs`。

在 :file:`tools/labs` 目录下运行 :code:`make` 命令以启动虚拟机。创建 :file:`/test` 目录，并尝试挂载新的磁盘：

.. code-block:: bash

    mkdir /test
    mount /dev/vdd /test

我们无法挂载该虚拟磁盘，因为内核不支持 :file:`mydisk.img` 的文件系统。你需要识别出 :file:`mydisk.img` 的文件系统类型，并在编译内核时在内核中添加对该文件系统的支持。

关闭虚拟机（关闭 QEMU 窗口，无需使用其他命令）。在物理机上使用 :command:`file` 命令查看 :file:`mydisk.img` 文件的文件系统类型。可以识别出它是 :command:`btrfs` 文件系统。

你需要在内核中启用 :command:`btrfs` 支持并重新编译内核镜像。

.. warning:: 如果在执行 :command:`make menuconfig` 命令时收到错误提示，可能是因为你没有安装 :command:`libncurses5-dev` 包。使用以下命令安装它：

             ::

                 sudo apt-get install libncurses5-dev

.. hint:: 进入 :file:`~/src/linux/` 子目录。运行 :command:`make menuconfig` 命令，进入 *File systems* 部分。启用 *Btrfs filesystem support* 选项。你需要使用内置选项（而不是模块），即 :command:`<*>` 必须出现在选项旁边（**不是** :command:`<M>`）。

          保存你所做的配置。使用默认配置文件（:file:`config`）。

          在内核源代码子目录（:file:`~/src/linux/`）中使用以下命令重新编译：

          ::

              make

          为了加快速度，你可以使用 :command:`-j` 选项并行运行多个任务。通常建议使用 :command:`CPU 数量+1`：

          ::

              make -j5

内核重新编译完成后，**重新启动** QEMU 虚拟机：也就是在子目录中执行 :command:`make` 命令。你无需复制任何内容，因为 :file:`bzImage` 文件是符号链接，指向你刚刚重新编译完成的内核映像。

在 QEMU 虚拟机内部，再次执行 :command:`mkdir` 和 :command:`mount` 操作。有了对 :command:`btrfs` 文件系统的支持，现在 :command:`mount` 命令将成功完成。

.. note:: 在做作业时，无需重新编译内核，因为你只需要使用内核模块。然而，熟悉配置和重新编译内核很重要。

如果你仍然想重新编译内核，请备份 :file:`bzImage` 文件（在 ~/src/linux 的链接中有完整路径）。借此你可以返回到初始配置，拥有与 vmchecker 完全相同的环境。

..
  _[EXERCISE2-END]

..
  _[EXERCISE3-BEGIN]

GDB 和 QEMU
------------

我们可以实时对 QEMU 虚拟机进行调查和排除问题 。

.. note:: 你还可以使用 :command:`GDB Dashboard` 插件，以获得友好的界面。系统必须有安装 Python，才能成功编译 :command:`gdb`。

          要想安装它，你只需运行：
          ::

              wget -P ~ git.io/.gdbinit

为此，我们首先启动 QEMU 虚拟机。然后，我们可以使用以下命令通过 :command:`gdb` 连接到 **正在运行的 QEMU 虚拟机**：

::

    make gdb

我们在 QEMU 命令中使用了 :command:`-s` 参数，这意味着 QEMU 会监听 :code:`1234` 端口，等待 :command:`gdb` 的连接。我们可以使用 :command:`gdb` 的 **远程目标** 功能来进行调试。现有的 :file:`Makefile` 已经帮我们处理了相关细节。

当你附加调试器到一个进程时，该进程会暂停。你可以添加断点并检查进程的当前状态。

附加 gdb 到 QEMU 虚拟机（使用 :command:`make gdb` 命令）并在 :command:`gdb` 控制台中使用以下命令在 :code:`sys_access` 函数中设置断点：

::

    break sys_access

此时，虚拟机已暂停。要继续执行（直到调用 :code:`sys_access` 函数），请在 :command:`gdb` 控制台中使用命令：

::

    continue

此时，虚拟机处于活动状态并具有可用的控制台。要进行 :code:`sys_access` 调用，我们可以使用 :command:`ls` 命令。请注意，此时虚拟机再次被 :command:`gdb` 暂停，并且在 :command:`gdb` 控制台中出现了相应的 :code:`sys_access` 回调消息。

使用 :command:`step` 、:command:`continue` 或 :command:`next` 指令逐步跟踪代码执行。你可能不完全理解整个执行过程，所以可以使用 :command:`list` 和 :command:`backtrace` 等命令来跟踪执行流程。

.. hint:: 在 :command:`gdb` 提示符处，你可以按 :command:`Enter` (不输入其他内容) 来重新运行上一条命令。

..
  _[EXERCISE3-END]

..
  _[EXERCISE4-BEGIN]

4. GDB 探索
-----------

使用 :command:`gdb` 命令显示创建内核线程（`kernel_thread`）的函数的源代码。

.. note:: 你可以使用 GDB 进行静态内核分析，在内核源代码目录中执行类似以下命令：

          ::

              gdb vmlinux

          请参阅实验中的 `gdb (Linux) <#gdb-linux>`__ 部分。

使用 `gdb` 命令找到 `jiffies` 变量在内存中的地址和内容。:code:`jiffies` 变量保存了系统启动以来的时钟节拍数。

.. hint:: 要跟踪 jiffies 变量的值，可以在 :command:`gdb` 中使用动态分析，运行以下命令：

          ::

              make gdb

          就像前面的练习一样。

          请参阅实验室中的 `gdb (Linux) <#gdb-linux>`__ 部分。

.. hint:: :code:`jiffies` 是一个 64 位变量。
          可以发现它的地址与 :code:`jiffies_64` 变量相同。

          要查看 64 位变量的内容，请在 :command:`gdb` 控制台中使用以下命令：

          ::

              x/gx & jiffies

          如果要显示 32 位变量的内容，可以在 :command:`gdb` 控制台中使用以下命令：

          ::

              x/wx & jiffies

..
  _[EXERCISE4-END]

..
  _[EXERCISE5-BEGIN]
```


5. Cscope 探索
--------------------

使用 LXR 或 cscope 在 :file:`~/src/linux/` 目录下查找特定结构或函数的位置。

Cscope 索引文件已生成。使用 :command:`vim` 和其他相关命令来滚动浏览源代码。例如，使用以下命令：

::

    vim

打开 :command:`vim` 编辑器。然后，在编辑器内部使用以下命令：

:command:`:cs find g task\_struct`

找到定义以下数据类型的文件：

-    ``struct task_struct``

-    ``struct semaphore``

-    ``struct list_head``

-    ``spinlock_t``

-    ``struct file_system_type``

.. hint:: 对于特定结构，只需搜索其名称。

          例如，在 :command:`struct task_struct` 的情况下，搜索 :command:`task_struct` 字符串。

通常，你会得到更多匹配项。要找到你感兴趣的匹配项，请执行以下操作：

#.    使用 :command:`vim` 的 :command:`:copen` 命令列出所有匹配项。

#.    通过查找左括号（:command:`{`），即结构定义行上的单个字符，找到正确的匹配项，要搜索左括号，可以在 :command:`vim` 中使用 :command:`/{`。

#.    在相应的行上，按下 :command:`Enter` 键进入定义变量的源代码。

#.    使用命令 :command:`:cclose` 关闭次要窗口。

找到声明以下全局内核变量的文件：

-    ``sys_call_table``

-    ``file_systems``

-    ``current``

-    ``chrdevs``

.. hint:: 要做到这一点，使用带有以下语法的 :command:`vim` 命令：

          :command:`:cs f g <symbol>`

          其中 :command:`<symbol>` 是要搜索的符号的名称。

找到声明以下函数的文件：

-    ``copy_from_user``

-    ``vmalloc``

-    ``schedule_timeout``

-    ``add_timer``

.. hint:: 要做到这一点，使用带有以下语法的 :command:`vim` 命令：

          :command:`:cs f g <symbol>`

          其中:command:`<symbol>`是要搜索的符号的名称。

顺序浏览以下的数据结构：

-   ``struct task_struct``

-   ``struct mm_struct``

-   ``struct vm_area_struct``

-   ``struct vm_operations_struct``

也就是说，你访问一个结构，然后找到其中具有下一个结构数据类型的字段，访问相应的字段，依此类推。注意这些结构定义在哪些文件中；这将对接下来的实验有用。


.. hint:: 要在 :command:`vim` 中搜索符号（:command:`vim` 带有 :command:`cscope` 支持），可以将光标放在符号上，并使用键盘快捷键 :command:`Ctrl+]`。

          要返回到上一个匹配项（在搜索/跳转之前的匹配项），请使用键盘快捷键 :command:`Ctrl+o`。

          要向前进行搜索（返回到 :command:`Ctrl+o` 之前的匹配项），请使用键盘快捷键 :command:`Ctrl+i`。

按照上述说明，找到并浏览以下函数调用序列：

-   ``bio_alloc``

-   ``bio_alloc_bioset``

-   ``bvec_alloc``

-   ``kmem_cache_alloc``

-   ``slab_alloc``

.. note:: 阅读实验中的 `cscope <#cscope>`__ 或 `LXR 交叉引用 <#lxr-cross-reference>`__ 部分。
