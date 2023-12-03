=================
地址空间
=================

`查看幻灯片 <address-space-slides.html>`_

.. slideconf::
   :autoslides: False
   :theme: single-level

课程目标：
===================

.. slide:: 地址空间
   :inline-contents: True
   :level: 2

   * x86 MMU

     * 分段

     * 分页

     * TLB

   * Linux 地址空间

     * 用户空间

     * 内核空间

     * 高内存（high memory）


x86 MMU
=======

x86 内存管理单元（MMU）包括分段单元和分页单元。分段单元可用于定义由逻辑（虚拟）起始地址、基本线性（映射）地址和大小定义的逻辑内存段。段也可以根据访问类型（读取、执行、写入）或特权级别（例如，我们可以定义一些只能由内核访问的段）来限制访问。

当 CPU 进行内存访问时，它将使用分段单元根据段描述符中的信息将逻辑地址转换为线性地址。

如果启用了分页，线性地址将使用页表中的信息进一步转换为物理地址。

请注意，分段单元无法禁用，因此如果启用了 MMU，将始终使用分段。

.. slide:: x86 MMU
   :inline-contents: True
   :level: 2

   |_|

   .. ditaa::

                   +--------------+           +------------+
	 logical   |              |  linear   |            |  physical
        ---------> | Segmentation | --------> |   Paging   | ---------->
         address   |     Unit     |  address  |    Unit    |  address
	           |              |           |            |
                   +--------------+           +------------+

选择器
---------

程序可以使用多个段（segment），为了确定使用哪个段，使用了特殊寄存器（称为选择器）。常用的基本选择器有 CS——“代码选择器”，DS——“数据选择器”和 SS——“堆栈选择器”。

指令获取默认使用 CS，而数据访问默认使用 DS，除非使用了堆栈（例如，通过 pop 和 push 指令进行数据访问），这种情况下默认使用 SS。

选择器有三个主要字段：索引，表索引（TI）和运行特权级别（RPL）：


.. slide:: 选择器
   :inline-contents: True
   :level: 2

   |_|

   .. ditaa::
                              15           3    2 1    0
                               +------------+----+-----+
                               |            |    |     |
         Segment selectors     |   index    | TI | RPL |
     (CS, DS, SS, ES, FS, GS)  |            |    |     |
                               +------------+----+-----+

   .. ifslides::

      * 选择器: CS、DS、SS、ES、FS、GS

      * 索引: 用于索引段描述符表

      * TI: 选择 GDT 或 LDT

      * RPL: 仅对 CS 表示（当前）运行的特权级别

      * GDTR 和 LDTR 寄存器指向 GDT 和 LDT 的基址


索引用于确定应使用描述符表的哪个条目。 `TI` 用于选择全局描述符表（GDT）或局部描述符表（LDT）。这些表实际上是从特殊寄存器 `GDTR`（用于 GDT）和 `LDTR`（用于 LDT）指定的位置开始的数组。

.. note:: LDT 设计用于允许应用程序可以定义它们自己的特定段。尽管不是很多应用程序使用此功能，但 Linux（和 Windows）提供了系统调用，允许应用程序创建自己的段。

`RPL` 仅用于 CS，并表示当前特权级别。有 4 个特权级别，最高级别为 0（通常由内核使用），最低级别为 3（通常由用户应用程序使用）。


段描述符
------------------

CPU 使用选择器的 `index` 字段来访问一个 8 字节的描述符：

.. slide:: 段描述符
   :inline-contents: True
   :level: 2

   |_|

   .. ditaa::

     63                           56                                              44              40                              32
    +-------------------------------+---+---+---+---+---------------+---+---+---+---+---------------+-------------------------------+
    |                               |   | D |   | A |    Segment    |   |   D   |   |               |                               |
    |     Base Address 31:24        | G | / | L | V |     Limit     | P |   P   | S |    Type       |     Base Address 23:16        |
    |                               |   | B |   | L |     19:16     |   |   L   |   |               |                               |
    +-------------------------------+---+---+---+---+---------------+---+---+---+---+---------------+-------------------------------+
    |                                                               |                                                               |
    |                    Base address 15:0                          |                       Segment Limit  15:0                     |
    |                                                               |                                                               |
    +---------------------------------------------------------------+---------------------------------------------------------------+
     31                                                              15                                                            0


   * Base: 段的起始线性地址

   * Limit: 段的大小

   * G: 粒度位：如果设置，则大小以字节为单位，否则以 4K 页面为单位

   * B/D: 数据/代码

   * Type: 代码段、数据/堆栈、TSS、LDT、GDT

   * Protection: 访问段所需的最低特权级别（RPL 与 DPL 进行比较）


一些描述符字段你应该比较熟悉。这是因为它们与我们之前讨论的中断描述符有一些相似之处。


Linux 中的分段
---------------------

在 Linux 中，段不用于定义堆栈、代码或数据。这些将使用分页单元进行设置，因为它允许更好的粒度，并且更重要的是，它允许 Linux 使用通用的方法，使得其在其他（不支持分段）的体系架构上也能工作。

然而，由于分段单元无法禁用，Linux 必须创建 4 个通用的 0-4GB 段，分别用于内核代码、内核数据、用户代码和用户数据。

除此之外，Linux 还使用段与 `set_thread_area` 系统调用一起来实现线程本地存储（TLS）。

它还使用 TSS 段来定义内核堆栈，以备在特权级别变化时（例如，在用户空间运行时的系统调用、中断发生时）使用。

.. slide:: Linux 中的分段
   :inline-contents: True
   :level: 2

   .. code-block:: c

      /*
       * Linux 中每个 CPU 的 GDT 布局：
       *
       *   0——空（null）                                            <=== 缓存行 #1
       *   1——保留
       *   2——保留
       *   3——保留
       *
       *   4——未使用                                                <=== 缓存行 #2
       *   5——未使用
       *
       *  ------- TLS（线程本地存储）段的开始：
       *
       *   6——TLS 段 #1                   [ glibc 的 TLS 段 ]
       *   7——TLS 段 #2                   [ Wine 的 %fs Win32 段 ]
       *   8——TLS 段 #3                                             <=== 缓存行 #3
       *   9——保留
       *  10——保留
       *  11——保留
       *
       *  ------- 内核段的开始：
       *
       *  12——内核代码段                                             <=== 缓存行 #4
       *  13——内核数据段
       *  14——默认用户 CS
       *  15——默认用户 DS
       *  16——TSS                                                   <=== 缓存行 #5
       *  17——LDT
       *  18——PNPBIOS 支持（16->32 门）
       *  19——PNPBIOS 支持
       *  20——PNPBIOS 支持                                          <=== 缓存行 #6
       *  21——PNPBIOS 支持
       *  22——PNPBIOS 支持
       *  23——APM BIOS 支持
       *  24——APM BIOS 支持                                         <=== 缓存行 #7
       *  25——APM BIOS 支持
       *
       *  26——ESPFIX 小型 SS
       *  27——每个 CPU                 [ 指向每个 CPU 数据区的偏移量 ]
       *  28——stack_canary-20          [ 用于栈保护 ]                <=== 缓存行 #8
       *  29——未使用
       *  30——未使用
       *  31——用于双重故障处理的 TSS
       */

       DEFINE_PER_CPU_PAGE_ALIGNED(struct gdt_page, gdt_page) = { .gdt = {
       #ifdef CONFIG_X86_64
               /*
                * 在长模式下，我们也需要有效的内核数据和代码段
                * IRET 将检查段类型  kkeil 2000/10/28
                * 同样，sysret 需要特殊的 GDT 布局
                *
                * 目前，TLS 描述符与 i386 上的位置不同。
                * 希望没有人期望它们位置固定（Wine？）
                */
               [GDT_ENTRY_KERNEL32_CS]         = GDT_ENTRY_INIT(0xc09b, 0, 0xfffff),
               [GDT_ENTRY_KERNEL_CS]           = GDT_ENTRY_INIT(0xa09b, 0, 0xfffff),
               [GDT_ENTRY_KERNEL_DS]           = GDT_ENTRY_INIT(0xc093, 0, 0xfffff),
               [GDT_ENTRY_DEFAULT_USER32_CS]   = GDT_ENTRY_INIT(0xc0fb, 0, 0xfffff),
               [GDT_ENTRY_DEFAULT_USER_DS]     = GDT_ENTRY_INIT(0xc0f3, 0, 0xfffff),
               [GDT_ENTRY_DEFAULT_USER_CS]     = GDT_ENTRY_INIT(0xa0fb, 0, 0xfffff),
       #else
               [GDT_ENTRY_KERNEL_CS]           = GDT_ENTRY_INIT(0xc09a, 0, 0xfffff),
               [GDT_ENTRY_KERNEL_DS]           = GDT_ENTRY_INIT(0xc092, 0, 0xfffff),
               [GDT_ENTRY_DEFAULT_USER_CS]     = GDT_ENTRY_INIT(0xc0fa, 0, 0xfffff),
               [GDT_ENTRY_DEFAULT_USER_DS]     = GDT_ENTRY_INIT(0xc0f2, 0, 0xfffff),
               /*
                * 用于调用 PnPBIOS 的段具有字节粒度。
                * 代码段和数据段具有固定的 64K 限制，
                * 传输段的大小在运行时设置。
                */
               /* 32 位代码 */
               [GDT_ENTRY_PNPBIOS_CS32]        = GDT_ENTRY_INIT(0x409a, 0, 0xffff),
               /* 16 位代码 */
               [GDT_ENTRY_PNPBIOS_CS16]        = GDT_ENTRY_INIT(0x009a, 0, 0xffff),
               /* 16 位数据 */
               [GDT_ENTRY_PNPBIOS_DS]          = GDT_ENTRY_INIT(0x0092, 0, 0xffff),
               /* 16 位数据 */
               [GDT_ENTRY_PNPBIOS_TS1]         = GDT_ENTRY_INIT(0x0092, 0, 0),
               /* 16 位数据 */
               [GDT_ENTRY_PNPBIOS_TS2]         = GDT_ENTRY_INIT(0x0092, 0, 0),
               /*
                * APM 段具有字节粒度，并且它们的基址在运行时设置。
                * 所有段的限制都是 64K。
                */
               /* 32 位代码 */
               [GDT_ENTRY_APMBIOS_BASE]        = GDT_ENTRY_INIT(0x409a, 0, 0xffff),
               /* 16 位代码 */
               [GDT_ENTRY_APMBIOS_BASE+1]      = GDT_ENTRY_INIT(0x009a, 0, 0xffff),
               /* 数据 */
               [GDT_ENTRY_APMBIOS_BASE+2]      = GDT_ENTRY_INIT(0x4092, 0, 0xffff),

               [GDT_ENTRY_ESPFIX_SS]           = GDT_ENTRY_INIT(0xc092, 0, 0xfffff),
               [GDT_ENTRY_PERCPU]              = GDT_ENTRY_INIT(0xc092, 0, 0xfffff),
               GDT_STACK_CANARY_INIT
       #endif
       } };
       EXPORT_PER_CPU_SYMBOL_GPL(gdt_page);


检查选择器和段
---------------------------------

.. slide:: 检查选择器和段
   :inline-contents: True
   :level: 2

   |_|

   .. asciicast:: ../res/selectors-and-segments.cast


x86 分页
----------

x86 分页单元支持两种类型的分页：常规分页和扩展分页。

常规分页有 2 级，并且固定的页面大小为 4KB。线性地址被分割为三个字段：

* 目录（directory）（最高的 10 位）

* 表（table）（接下来的 10 位）

* 偏移量（offset）（最低的 12 位）


.. slide:: 常规分页
   :inline-contents: True
   :level: 2

   |_|

   .. ditaa::
      :--no-separation:

                                         Virtual Address
      +------------+     +------------------+----------------+---------------+
      |    CR3     |     |    DIRECTORY cEEE|     TABLE cDDD |    OFFSET cCCC|
      +------------+     +------------------+----------------+---------------+
            |                     |                 |            |
            |                     |                 |            |             PAGE
            |                     |                 |            |    /-----------------------\
            |                     |                 |            |    |                       |
            |                     |                 |            |    |                       |
            |                     |     +-----------+            |    +-----------------------+
            |                     |     |                        +--->|  Physical Address cCCC|
            |                     |     |                             +-----------------------+
            |   +-----------------+     |                             |                       |
            |   |                       |         PAGE                |                       |
            |   |                       |         TABLE               |                       |
            |   |          PAGE         |     /------------\          |                       |
            |   |        DIRECTORY      |     |            |          |                       |
            |   |      /------------\   |     |            |          |                       |
            |   |      |            |   |     +------------+   +----> \-----------------------/
            |   |      |            |   +---->| cDDD       |---+
            |   |      |            |         +------------+
            |   |      |            |         |            |
            |   |      |            |         |            |
            |   |      +------------+         |            |
            |   +----->|cEEE        |---+     |            |
            |          +------------+   |     |            |
            |          |            |   +---->\------------/
            |          |            |
            +--------->\------------/


当启用扩展分页时，使用单级分页，页面大小为 4MB。线性地址被分割为两个字段：

* 目录（最高的10位）

* 偏移量（最低的22位）

.. slide:: 扩展分页
   :inline-contents: True
   :level: 2

   .. ditaa::
      :--no-separation:

                                         Virtual Address
      +------------+     +-------------------+-----------------------------+
      |    CR3     |     |    DIRECTORY cEEE |          OFFSET cDDD        |
      +------------+     +-------------------+-----------------------------+
            |                      |                              |
            |                      |                              |             PAGE
            |                      |                              |    /----------------------\
            |                      |                              |    |                      |
            |                      |                              |    |                      |
            |                      |                              |    +----------------------+
            |                      |                              +--->| Physical Address cDDD|
            |                      |                                   +----------------------+
            |    +-----------------+                                   |                      |
            |    |                                                     |                      |
            |    |                                                     |                      |
            |    |          PAGE                                       |                      |
            |    |        DIRECTORY                                    |                      |
            |    |      /------------\                                 |                      |
            |    |      |            |             +------------------>\----------------------/
            |    |      |            |             |
            |    |      |            |             |
            |    |      |            |             |
            |    |      |            |             |
            |    |      +------------+             |
            |    +----->| cEEE       |-------------+
            |           +------------+
            |           |            |
            |           |            |
            +---------->\------------/


页表
------------

我们可以混合使用常规分页和扩展分页，目录页有一个位来指定是使用扩展分页还是常规分页。特殊的 CR3 寄存器指向页目录的基址，而页目录项指向页表的基址。

页目录和页表都有 1024 个条目，每个条目占用 4 字节。

所有表都存储在内存中，页表地址是物理地址。


.. slide:: 页表
   :inline-contents: False
   :level: 2

   * 页目录和页表都有 1024 个条目

   * 每个条目占用 4 字节

   * 特殊的 CR3 寄存器指向页目录的基址

   * 页目录项指向页表的基址

   * 所有表都存储在内存中

   * 所有表地址都是物理地址


页表项字段：

.. slide:: 页表项字段
   :inline-contents: True
   :level: 2

   * 存在/不存在

   * PFN（页面帧号）：物理地址的最高 20 位

   * 访问位——不由硬件更新（可由操作系统用于管理）

   * 脏位——不由硬件更新（可由操作系统用于管理）

   * 访问权限：读/写

   * 特权级别：用户/特权级

   * 页面大小——仅适用于页目录；如果设置，将使用扩展分页

   * PCD（页面缓存禁用），PWT（页面写穿透）


Linux 分页
------------

Linux 分页使用 4 个级别以支持 64 位架构。下图显示了如何使用多个虚拟地址块来索引页表并计算物理地址。


.. slide:: Linux 分页
   :inline-contents: True
   :level: 2

   .. ditaa::
      :--no-separation:

                                                                 Virtual Address
      +------------+     +------------------+-----------------+------------------+-------------------+---------------+
      |    CR3     |     |   GLOBAL DIR cEEE|   UPPER DIR cDDD|  MIDDLE DIR  cCCC|     TABLE     cBBB|   OFFSET cAAA |
      +------------+     +------------------+-----------------+------------------+-------------------+---------------+
            |                     |                 |            |                       |                   |
            |                     |                 |            |                       |                   |                  PAGE
            |                     |                 |            |                       |                   |         /----------------------\
            |                     |                 |            |                       |                   |         |                      |
            |                     |                 |            |                       |                   |         |                      |
            |                     |     +-----------+            |                       |     PAGE GLOBAL   |         +----------------------+
            |                     |     |                        |                       |      DIRECTORY    +-------->| Physical Address cAAA|
            |                     |     |                        |     PAGE MIDDLE       |    /------------\           +----------------------+
            |   +-----------------+     |                        |      DIRECTORY        |    |            |           |                      |
            |   |                       |       PAGE UPPER       |    /------------\     |    |            |           |                      |
            |   |                       |       DIRECTORY        |    |            |     |    |            |           |                      |
            |   |       PAGE GLOBAL     |     /------------\     |    |            |     |    |            |           |                      |
            |   |        DIRECTORY      |     |            |     |    +------------+     |    |            |           |                      |
            |   |      /------------\   |     |            |     +--->| cCCC       |---+ |    +------------+           |                      |
            |   |      |            |   |     |            |          +------------+   | +--->| cBBB       |---------->\----------------------/
            |   |      |            |   |     |            |          |            |   |      +------------+
            |   |      |            |   |     +------------+   +----->\------------/   |      |            |
            |   |      |            |   +---->| cDDD       |---+                       |      |            |
            |   |      |            |         +------------+                           +----->\------------/
            |   |      +------------+         |            |
            |   +----->| cEEE       |--+      |            |
            |          +------------+  |      |            |
            |          |            |  +----->\------------/
            |          |            |
            +--------->\------------/


Linux 具有用于创建和遍历页表的通用 API。借助于此我们可以实现使用相同的通用代码来创建和修改内核和进程的地址空间，该代码依赖于宏和函数，将这些通用操作转换为在不同体系结构上运行的代码。

以下是使用 Linux 页表 API 将虚拟地址转换为物理地址的示例：

.. slide:: 用于页表处理的 Linux API
   :inline-contents: True
   :level: 2

   .. code-block:: c

      struct * page;
      pgd_t pgd;
      pmd_t pmd;
      pud_t pud;
      pte_t pte;
      void *laddr, *paddr;

      pgd = pgd_offset(mm, vaddr);
      pud = pud_offet(pgd, vaddr);
      pmd = pmd_offset(pud, vaddr);
      pte = pte_offset(pmd, vaddr);
      page = pte_page(pte);
      laddr = page_address(page);
      paddr = virt_to_phys(laddr);


为了支持具有少于 4 级分页的体系结构（例如32位 x86），某些宏和/或函数可以为 0/空：

.. slide:: 如果平台支持的分页少于 4 级怎么办？
   :inline-contents: True
   :level: 2

   .. code-block:: c

      static inline pud_t * pud_offset(pgd_t * pgd,unsigned long address)
      {
          return (pud_t *)pgd;
      }

      static inline pmd_t * pmd_offset(pud_t * pud,unsigned long address)
      {
          return (pmd_t *)pud;
      }

