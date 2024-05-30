==========================
对称多处理
==========================

.. meta::
   :description: 介绍了对称多处理（SMP）在 Linux 内核中的相关概念和同步机制，包括内核并发源、原子操作、自旋锁、缓存抖动、进程和中断上下文同步、互斥锁、内存排序和 RCU 等内容。
   :keywords: Linux内核, 对称多处理, 同步, 原子操作, 自旋锁, 互斥锁, 内存排序, RCU

`查看幻灯片 <smp-slides.html>`_

.. slideconf::
   :autoslides: False
   :theme: single-level

课堂目标：
===================

.. slide:: 对称多处理
   :inline-contents: True
   :level: 2

   * 内核并发

   * 原子操作

   * 自旋锁

   * 缓存抖动

   * 优化自旋锁

   * 进程和中断上下文同步

   * 互斥锁

   * 每个 CPU 的数据

   * 内存排序和屏障

   * 读-拷贝 更新（RCU）


基本同步
======================

由于 Linux 内核支持对称多处理（SMP），它必须使用一组同步机制来保证结果可预测，并避免竟态条件。

.. note:: 在本讲座中，我们将交替使用核心、CPU 和处理器这些术语。

竟态条件在以下两个条件同时满足时可能会发生：

.. slide:: 竟态条件
   :inline-contents: True
   :level: 2

   * 至少有两个执行上下文同时运行：

     * 真正并行运行（例如，在不同处理器上运行的两个系统调用）

     * 其中一个上下文可以任意抢占另一个上下文（例如，中断抢占了系统调用）

   * 执行上下文对共享内存进行读写访问


竟态条件可能导致错误。这些错误结果很难调试，因为它们只在执行上下文以非常特定的顺序调度到 CPU 核心上时才会出现。

竟态条件的一个经典示例是资源计数器释放操作的不正确实现：

.. slide:: 竟态条件：资源计数器释放
   :inline-contents: True
   :level: 2

   .. code-block:: c

      void release_resource()
      {
           counter--;

	   if (!counter)
               free_resource();
      }


资源计数器用于确保共享资源持续可用，直至所有用户都将其释放。但上述实现存在一个竟态条件，可能导致资源被释放两次：


.. slide:: 竟态条件场景
   :inline-contents: True
   :level: 2

   |_|

   .. ditaa::

                             counter is 2

        Thread A                                Thread B

           *
           |
           |
     +---------------------+
     |  dec counter        |   counter is 1
     |  cEEE        	   |
     +---------------------+
           |
           |                 B preempts A
           +-----------------------------------------------+
                                                           |
	                                                   v
			   			    +----------------------+
                             counter is 0           | dec counter  	   |
                                                    | if (!counter)	   |
                           resource is freed        |     free_resource(); |
			   			    | cEEE    	       	   |
			   			    +----------------------+
                        B finishes, A continues            |
           +-----------------------------------------------+
           |
           v
    +----------------------+
    | if (!counter)	   |
    |     free_resource(); | resource is freed
    | cEEE     	       	   |
    +----------------------+

在大多数情况下，`release_resource()` 函数只会释放资源一次。然而，在上述情况中，如果线程 A 在递减 `counter` 后被立即抢占，而线程 B 调用了 `release_resource()`，导致资源被释放。当线程 A 恢复执行时，它也会释放资源，因为计数器的值为 0。

为了避免竞态条件，程序员首先必须确定可能产生竞态条件的临界区（critical section）。临界区是代码的一部分，它读取和写入共享内存，多个并行的上下文都可访问这段共享内存。

在上述示例中，最小的临界区从计数器递减开始，到检查计数器的值结束。

确定了临界区之后，可以使用以下方法之一避免竞态条件：

.. slide:: 避免竟态条件
   :inline-contents: True
   :level: 2

   * 使临界区 **原子化** (例如使用原子指令)

   * 在临界区期间 **禁用抢占** (例如禁用中断、后半部分处理程序或线程抢占)

   * **序列化访问** 临界区 (例如使用自旋锁或互斥锁，同一时间只允许有一个上下文或线程进入临界区)



Linux 内核并发源
================================

Linux 内核中存在多个并发源，具体多少个取决于内核配置和运行系统的类型：


.. slide:: Linux 内核并发源
   :inline-contents: True
   :level: 2

   * **单核系统**, **非抢占内核**：当前进程可以被中断抢占

   * **单核系统**, **抢占内核**：上述情况 + 当前进程可以被其他进程抢占

   * **多核系统**：上述情况 + 当前进程可以与在另一个处理器上运行的另一个进程或中断并行运行

.. note:: 我们只讨论内核并发问题，这就是为什么在单核系统上运行的非抢占内核只有中断作为并发源。


原子操作
===========

在某些情况下，我们可以通过使用硬件提供的原子操作来避免竞态条件。Linux 提供了统一的 API，用来访问原子操作：

.. slide:: 原子操作
   :inline-contents: True
   :level: 2

   * 基于整数：

     * 简单操作: :c:func:`atomic_inc` (原子递增), :c:func:`atomic_dec` (原子递减), :c:func:`atomic_add` (原子加法), :c:func:`atomic_sub` (原子减法)

     * 条件操作: :c:func:`atomic_dec_and_test` (原子递减并测试), :c:func:`atomic_sub_and_test` (原子减法并测试)

   * 基于位操作：

     * 简单操作: :c:func:`test_bit` (测试位), :c:func:`set_bit` (设置位), :c:func:`change_bit` (修改位)

     * 条件操作: :c:func:`test_and_set_bit` (测试并设置位), :c:func:`test_and_clear_bit` (测试并清除位), :c:func:`test_and_change_bit` (测试并修改位)

例如，我们可以使用 :c:func:`atomic_dec_and_test` 来实现资源计数器的递减和值检查的原子操作：

.. slide:: 使用 :c:func:`atomic_dec_and_test` 来实现资源计数器的释放
   :inline-contents: True
   :level: 2

   .. code-block:: c

      void release_resource()
      {
          if (atomic_dec_and_test(&counter))
               free_resource();
      }


原子操作的挑战之一是，在多核系统中，系统级别上不再是原子的，尽管在处理器核心级别仍然是原子的。

为了理解这一点，我们需要将原子操作分解为内存加载和存储操作。然后，我们可以构造出在多个 CPU 之间交错执行加载和存储操作的竞态条件场景。就像下面的示例中，两个处理器对同一个值进行递增将产生意外的结果：

.. slide:: 在 SMP 系统上原子操作可能不再是原子的
   :inline-contents: True
   :level: 2

   |_|

   .. ditaa::


                                   +------------+
                                   |   Memory   |
     +-------------+   LOAD (0)    |            |               +-------------+
     |    CPU 0    |<--------------|   v <- 0   |    LOAD (0)   |    CPU 1    |
     |             |   STORE (1)   |            |-------------->|             |
     |    inc v    |-------------->|   v <- 1   |    STORE (1)  |    inc v    |
     | cEEE        |               |   v <- 1   |<--------------| cEEE        |
     +-------------+               | cEEE       |               +-------------+
                                   +------------+


为了在 SMP 系统上提供原子操作，不同的体系结构采用不同的技术。例如，在 x86 上，使用 LOCK 前缀可以在执行带有前缀的操作时锁定系统总线。

.. slide:: 修复 SMP 系统上的原子操作（x86）
   :inline-contents: True
   :level: 2

   |_|

   .. ditaa::

                                   +------------+
     +-------------+   BUS LOCK    |   Memory   |
     |    CPU 1    |<------------->|            |
     |             |   LOAD (0)    |            |
     |    inc v    |<--------------|   v <- 0   |
     |             |   STORE (1)   |            |
     |             |-------------->|   v <- 1   |
     |             |  BUS UNLOCK   |            |
     | cEEE        |<------------->|            |   BUS LOCK    +-------------+
     +-------------+               |            |<------------->|    CPU 1    |
                                   |            |   LOAD (1)    |             |
                                   |            |<--------------|    inc v    |
                                   |   v <- 2   |   STORE (2)   |             |
                                   |            |-------------->|             |
                                   |            |  BUS UNLOCK   |             |
                                   | cEEE       |<------------->| cEEE        |
                                   +------------+               +-------------+


在 ARM 上，LDREX 和 STREX 指令一起使用，来实现原子访问：LDREX 加载一个值并通知独占监视器正在进行原子操作。STREX 尝试存储一个新值，但只有在独占监视器未检测到其他独占操作时才成功。因此，为了实现原子操作，程序员不断重试操作（包括 LDREX 和 STREX），直到独占监视器发出成功的信号。

尽管它们经常被解释为“轻量级”或“高效”的同步机制（因为它们“不需要自旋或上下文切换”，或者因为它们“在硬件中实现，所以它们肯定更高效”，或者因为它们“只是指令，所以它们的效率应该与其他指令类似”），但从实现细节来看，原子操作实际上资源消耗巨大。


禁用抢占（中断）
==================

在单核系统和非抢占内核中，唯一的并发来源是通过中断抢占当前线程。要想防止并发，只需要禁用中断。

这可以通过特定于体系结构的指令来实现，但 Linux 提供了与体系结构无关的 API 来禁用和启用中断：

.. slide:: 与中断同步（x86）
   :inline-contents: True
   :level: 2

   .. code-block:: c

       #define local_irq_disable() \
           asm volatile („cli” : : : „memory”)

      #define local_irq_enable() \
          asm volatile („sti” : : : „memory”)

      #define local_irq_save(flags) \
          asm volatile ("pushf ; pop %0" :"=g" (flags)
                        : /* no input */: "memory") \
          asm volatile("cli": : :"memory")

      #define local_irq_restore(flags) \
          asm volatile ("push %0 ; popf"
                        : /* no output */
                        : "g" (flags) :"memory", "cc");


虽然中断可以通过 :c:func:`local_irq_disable` 和 :c:func:`local_irq_enable` 函数来显式禁用和启用，但这些 API 只应在当前状态和中断是已知的情况下使用。它们通常用于核心内核代码（如中断处理）。

如果是由于并发问题而希望避免中断的典型情况，建议使用 :c:func:`local_irq_save` 和 :c:func:`local_irq_restore` 变体。它们会保存和恢复中断状态，因此可以在重叠的临界区中自由调用它们，而无需担心因在临界区内意外启用中断而造成糟糕情况，只要调用是平衡的。

自旋锁
==========

自旋锁用于实现对关键区域的串行访问。在多核系统中，可以实现真正的执行并行性，因此自旋锁是必需的。以下是典型的自旋锁实现：

.. slide:: 自旋锁实现示例（x86）
   :inline-contents: True
   :level: 2

   .. code-block:: asm

      spin_lock:
          lock bts [my_lock], 0
	  jc spin_lock

      /* 临界区 */

      spin_unlock:
          mov [my_lock], 0

    **bts dts, src**——位测试并设置；它将来自 dts 内存地址的第 src 位复制到进位标志位（carry flag），然后将其设置为 1：

   .. code-block:: c

      CF <- dts[src]
      dts[src] <- 1


可以看出，自旋锁使用原子指令来确保同一时间只有一个核心能进入临界区。如果同时有多个核心试图进入，它们将不断地“自旋”，一直到锁被释放为止。

虽然自旋锁避免了竟态条件，但它可能对系统性能产生显著影响，这是由于“锁争用”（lock contention）引起的：

.. slide:: 锁争用
   :inline-contents: True
   :level: 2

   * 当至少有一个核心在自旋尝试进入临界区时，就会发生锁争用

   * 锁争用随着临界区大小、在临界区中花费的时间以及系统中处理器核心数量的增加而增加


自旋锁的另一个负面副作用是高速缓存抖动。

.. slide:: 高速缓存抖动
   :inline-contents: True
   :level: 2

   当多个处理器核心试图读写同一内存时，会发生高速缓存抖动，导致过多的高速缓存未命中。

   由于自旋锁在锁争用期间不断访问内存，高速缓存抖动很常见，这是由高速缓存一致性的实现方式造成的。


多处理器系统中的缓存一致性
==========================================

多处理器系统中的内存层次结构由本地 CPU 缓存（L1 缓存）、共享 CPU 缓存（L2 缓存）和内存组成。为了解释缓存一致性，我们暂且忽略 L2 缓存，只考虑 L1 缓存和主内存。

在下面的图中，我们展示了具有两个变量 A 和 B 的内存层次结构。两个变量位于不同的缓存行中，缓存和内存是同步的：

.. slide:: 同步的缓存和内存
   :inline-contents: True
   :level: 2

   |_|

   .. ditaa::

        +-------+             +-------+
        | CPU 0 |             | CPU 1 |
        +-------+             +-------+
          cache                 cache
        +-------+             +-------+
      A |   1   |             |   1   | A
        +-------+             +-------+
      B |   2   |             |   2   | B
        +-------+             +-------+
                     memory
        +-----------------------------+
      A |              1              |
        +-----------------------------+
      B |              2              |
        +-----------------------------+


如果缓存和内存之间缺乏同步机制的话，当 CPU 0 执行 `A = A + B`，而 CPU 1 执行 `B = A + B` 时，会得到以下内存视图：

.. slide:: 缺乏同步的缓存和内存
   :inline-contents: True
   :level: 2

   |_|

   .. ditaa::

        +-------+             +-------+
        | CPU 0 |             | CPU 1 |
        +-------+             +-------+
        A <- A + B            B <- A + B

        +-------+             +-------+
      A |   3   |             |   1   | A
        +-------+             +-------+
      B |   2   |             |   3   | B
        +-------+             +-------+
	       write back caches
        +-----------------------------+
      A |              1              |
        +-----------------------------+
      B |              2              |
        +-----------------------------+


为了避免上述情况，多处理器系统使用了缓存一致性（cache coherence）协议。缓存一致性协议主要分为两种类型：

.. slide:: 缓存一致性协议
   :inline-contents: True
   :level: 2

   * 总线嗅探（Bus snooping）：缓存监视内存总线事务，并采取行动以保持一致性。

   * 目录（Directory）协议：有一个单独的实体（目录）来维护缓存的状态；缓存与目录交互以保持一致性。

   缓存嗅探协议较为简单，但当核心数超过 32-64 时性能表现较差。

   目录协议的缓存一致性协议能够更好地扩展（可达数千个核心），非一致性存储访问（NUMA）系统中通常使用的就是目录协议。


实际应用中常用的简单缓存一致性协议是 MESI（根据缓存行状态的首字母缩写命名: **Modified（已修改）**, **Exclusive（独占）**, **Shared（共享）** 和 **Invalid（已失效）**）。其主要特点包括：

.. slide:: MESI 缓存一致性协议
   :inline-contents: True
   :level: 2

   * 缓存策略：写回（write back）

   * 缓存行状态

     * 已修改：由单个核心拥有且为脏数据

     * 独占：由单个核心拥有且为干净数据

     * 共享：由多个核心共享且为干净数据

     * 已失效：该行未被缓存

CPU 核心发出读取或写入请求将触发状态转换，如下所示：

.. slide:: MESI 状态转换
   :inline-contents: True
   :level: 2

   * 已失效 -> 独占：读取请求，所有其他核心中的该行处于已失效状态；从内存加载该行

   * 已失效 -> 共享 ：读取请求，至少一个核心中的该行处于共享或独占状态；从兄弟缓存加载该行

   * 已失效/共享/独占 -> 已修改：写入请求; **所有其他** 核心 **将自身的该行状态设为已失效**

   * 已修改 -> 已失效 ：来自其他核心的写入请求；将该行刷新到内存


.. note:: MESI 协议最重要的特性是它是一种写失效（write-invalidate）的缓存协议。当对共享位置进行写操作时，所有其他缓存都会失效。

这对于某些访问模式具有重要的性能影响，其中一个模式是争用简单的自旋锁，就像我们上面讨论的那样。

为了说明这个问题，让我们假设某个三核系统，其中第一个核心已经获取了自旋锁，并且正在运行临界区，而其他两个核心正在自旋等待进入临界区：

.. slide:: 自旋锁争用导致的缓存抖动
   :inline-contents: True
   :level: 2

   |_|

   .. ditaa::

      +-------+                     +-------+                  +-------+
      | CPU 0 |<---------------+    | CPU 1 |   Invalidate     | CPU 0 |
      | cache |<-------------+ |    | cache |<---+ +---------->| cache |
      +-------+  Invalidate  | |    +-------+    | |           +-------+
                             | |                 | |
                             | |                 +----------------------------+
      spin_lock(&lock);      | |                   |                          |
                             | |     READ lock     |                          |
                             | +---- WRITE lock ---+                          |
                             |                                                |
                             |                                 READ lock      |
                             +-------------------------------- WRITE lock ----+

         ...                            ...                       ...
      READ data                      READ lock                 READ lock
          |                              |                         |
          |                              |                         |
          |                              |                         |
          +------------------------------+-------------------------+
                                         |
                                         v

                                    cache miss

上图中可以看到，由于在锁上自旋的核心发出的写操作，我们会看到频繁的缓存行失效操作，这意味着两个等待的核心将在等待锁时刷新和加载缓存行，从而在内存总线上创建不必要的流量并减慢第一个核心的内存访问速度。

另一个问题是，很有可能由第一个 CPU 在临界区访问的数据存储在与锁相同的缓存行中（在获取锁后将数据准备在缓存中是一种常见优化）。这意味着两个自旋核心触发的缓存失效将减慢临界区的执行，从而触发更多的缓存失效操作。

优化的自旋锁
====================

由于缓存抖动，简单的自旋锁实现可能会出现性能问题，且随着核心数量的增加问题会不断加剧。要避免这个问题，有两种可行的策略：

* 减少写操作的次数，从而减少缓存失效操作的次数

* 避免其他处理器在相同的缓存行上自旋，从而避免缓存失效操作


下面是使用第一种方法的优化自旋锁实现：

.. slide:: 优化的自旋锁（KeAcquireSpinLock）
   :inline-contents: True
   :level: 2

   |_|

   .. code-block:: asm

      spin_lock:
          rep ; nop
          test lock_addr, 1
          jnz spin_lock
          lock bts lock_addr
          jc spin_lock


   * 我们首先只测试（读）锁，并且使用非原子指令来操作，以避免写入进而导致在自旋时产生失效操作

   * 仅当锁 *可能* 是空闲的时候，我们才尝试获取它

该实现还使用 **PAUSE** 指令来避免由于（错误的）内存顺序违规而引起的流水线刷新，并添加一个小延迟（与内存总线频率成比例）以降低功耗。

在许多体系结构版本的 Linux 内核中，还使用了类似的实现，其实现支持公平性（基于到达时间判断允许哪个 CPU 内核进入临界区），这种实现的名称是 `票据自旋锁（ticket spin lock） <https://lwn.net/Articles/267968/>`_。

然而，当前 x86 体系结构使用自旋锁队列来实现自旋锁，CPU 核心在不同的锁上旋转（如果分布在不同的缓存行中那最好），以避免缓存失效操作。

.. slide:: 自旋锁队列
   :inline-contents: True
   :level: 2

   |_|

   .. ditaa::

           +-------------------------------------------+
           |              Queued Spin Lock        cEEE |
           |                                           |
           |   +---+      +---+      +---+      +---+  |
           |   |   |----->|   |----->|   |----->|   |  |
           |   +---+      +---+      +---+      +---+  |
           |     ^          ^          ^          ^    |
           |     |          |          |          |    |
           +-------------------------------------------+
                 |          |	       |          |
               CPU10      CPU17       CPU99     CPU0
              owns the   spins on    spins on  spins on
               lock      private     private   private
                          lock        lock      lock



理论上，当一个 CPU 核心尝试获取锁并失败时，它会将自己的私有锁添加到等待 CPU 核心的列表中。当锁的持有者退出临界区时，它会解除列表中的下一个锁（如果有的话）。

即使使用读取自旋优化的自旋锁，也无法完全避免缓存失效操作。因为锁的持有者在修改与锁同一缓存行中的其他数据结构时，仍然会产生缓存失效操作。这反过来会在自旋核心的后续读取中产生内存流量。

因此，如果有大量核心（如 NUMA 系统），队列自旋锁由于其更好的扩展性因此更适合。由于它们具有与 ticket 锁类似的公平性属性，因此在 x86 架构上是首选的实现。


进程和中断上下文同步
=============================================

同时从进程和中断上下文访问共享数据是相对常见的场景。在单核系统中，我们可以通过禁用中断来解决这一问题，但在多核系统中，这种方法行不通，因为进程和中断上下文可能在不同的 CPU 核心上运行。

使用为多处理器系统设计的自旋锁似乎是正确的解决方案，但这样做可能会产生常见的死锁条件，下面的场景详述这一问题：

.. slide:: 进程和中断处理程序同步死锁
   :inline-contents: True
   :level: 2

   * 在进程上下文中，我们获取自旋锁

   * 发生中断，并在同一 CPU 核心上调度

   * 中断处理程序运行并尝试获取自旋锁

   * 当前 CPU 将发生死锁

要避免这个问题，可以双管齐下：

.. slide:: 用于 SMP 的中断同步
   :inline-contents: True
   :level: 2

   * 在进程上下文中：禁用中断并获取自旋锁；这将保护免受中断或其他 CPU 核心竟态条件的影响（:c:func:`spin_lock_irqsave` 和 :c:func:`spin_lock_restore` 结合了这两个操作）

   * 在中断上下文中：获取自旋锁；这将保护免受运行在不同处理器上的其他中断处理程序或进程上下文的竟态条件的影响


对于其他中断上下文处理程序（如 softirqs、tasklets 或定时器），我们也面临相同的问题，尽管禁用中断可能会起作用，但建议使用专用的 API：

.. slide:: SMP 的软中断同步
   :inline-contents: True
   :level: 2

   * 在进程上下文中使用 :c:func:`spin_lock_bh` (将 :c:func:`local_bh_disable` 和 :c:func:`spin_lock` 结合起来) 和 :c:func:`spin_unlock_bh` (将 :c:func:`spin_unlock` 和 :c:func:`local_bh_enable` 结合起来)

   * 在软中断上下文中使用：:c:func:`spin_lock` 和 :c:func:`spin_unlock` (如果与中断处理程序共享数据，则使用 :c:func:`spin_lock_irqsave` 和 :c:func:`spin_lock_irqrestore`)


如前所述，考虑到抢占，Linux 内核中的并发源还可以是其他进程。

.. slide:: 抢占
   :inline-contents: True
   :level: 2

   |_|

   抢占是可配置的：如果激活，它提供更低的延迟和响应时间，而如果停用，它提供更好的吞吐量。

   抢占被自旋锁和互斥锁禁用，但也可以手动禁用（通过核心内核代码）。


至于本地中断启用和禁用的 API，软中断和抢占 API 允许它们在嵌套的临界区中使用。计数器用来跟踪软中断和抢占的状态。事实上，它们使用同一个计数器，但是增量不同：

.. slide:: 抢占和软中断屏蔽
   :inline-contents: True
   :level: 2

   .. code-block:: c

      #define PREEMPT_BITS      8
      #define SOFTIRQ_BITS      8
      #define HARDIRQ_BITS      4
      #define NMI_BITS          1

      #define preempt_disable() preempt_count_inc()

      #define local_bh_disable() add_preempt_count(SOFTIRQ_OFFSET)

      #define local_bh_enable() sub_preempt_count(SOFTIRQ_OFFSET)

      #define irq_count() (preempt_count() & (HARDIRQ_MASK | SOFTIRQ_MASK))

      #define in_interrupt() irq_count()

      asmlinkage void do_softirq(void)
      {
          if (in_interrupt()) return;
          ...


互斥锁（Mutexes）
====================

互斥锁用于防止其他 CPU 核心的竟态条件，但它们只能在 **进程上下文** 中使用。与自旋锁相反，如果一个线程等待进入临界区，它不会占用 CPU 时间，而是会被添加到一个等待队列中，直到临界区被释放。

由于互斥锁和自旋锁的使用存在交集，因此在此比较一下它们：

.. slide:: 互斥锁
   :inline-contents: True
   :level: 2

   * 如果上下文切换开销低于自旋时间平均值，则系统吞吐量比自旋锁好，因为它们不会"浪费" CPU 周期

   * 不能在中断上下文中使用

   * 比起自旋锁具有更高的延迟

从概念上讲, :c:`mutex_lock` 操作相对简单：如果互斥锁未被获取，我们可以通过原子交换操作走捷径：


.. slide:: :c:func:`mutex_lock` 捷径
   :inline-contents: True
   :level: 2

   .. code-block:: c

      void __sched mutex_lock(struct mutex *lock)
      {
        might_sleep();

        if (!__mutex_trylock_fast(lock))
          __mutex_lock_slowpath(lock);
      }

      static __always_inline bool __mutex_trylock_fast(struct mutex *lock)
      {
        unsigned long curr = (unsigned long)current;

        if (!atomic_long_cmpxchg_acquire(&lock->owner, 0UL, curr))
          return true;

        return false;
      }


否则，我们将采取缓慢的路径，将自己添加到互斥锁的等待列表中并进入睡眠状态。

.. slide:: :c:func:`mutex_lock` 慢路径
   :inline-contents: True
   :level: 2

   .. code-block:: c

      ...
        spin_lock(&lock->wait_lock);
      ...
        /* 添加等待的任务到等待队列尾部 (FIFO): */
        list_add_tail(&waiter.list, &lock->wait_list);
      ...
        waiter.task = current;
      ...
        for (;;) {
	  if (__mutex_trylock(lock))
	    goto acquired;
        ...
	  spin_unlock(&lock->wait_lock);
	...
          set_current_state(state);
      	  spin_lock(&lock->wait_lock);
        }
        spin_lock(&lock->wait_lock);
      acquired:
        __set_current_state(TASK_RUNNING);
        mutex_remove_waiter(lock, &waiter, current);
        spin_lock(&lock->wait_lock);
      ...

完整的实现稍微复杂一些：它不会立即进入睡眠状态，而是在检测到锁的拥有者当前在不同的 CPU 上运行时进行乐观自旋，因为很有可能拥有者很快就会释放锁。它还检查信号并处理锁依赖引擎调试功能的互斥调试。

:c:func:`mutex_unlock` 操作是对称的：如果互斥锁没有等待者，我们可以通过原子交换操作走快速路径：

.. slide:: :c:func:`mutex_unlock` 快速路径
   :inline-contents: True
   :level: 2

   .. code-block:: c

      void __sched mutex_unlock(struct mutex *lock)
      {
	if (__mutex_unlock_fast(lock))
	  return;
	__mutex_unlock_slowpath(lock, _RET_IP_);
      }

      static __always_inline bool __mutex_unlock_fast(struct mutex *lock)
      {
	unsigned long curr = (unsigned long)current;

	if (atomic_long_cmpxchg_release(&lock->owner, curr, 0UL) == curr)
	  return true;

	return false;
      }

      void __mutex_lock_slowpath(struct mutex *lock)
      {
      ...
        if (__mutex_waiter_is_first(lock, &waiter))
		__mutex_set_flag(lock, MUTEX_FLAG_WAITERS);
      ...


.. note:: 由于 :c:type:`struct task_struct` 被缓存对齐，owner 字段的低 7 位可用于各种标志，例如 :c:type:`MUTEX_FLAG_WAITERS`。

否则，我们会选择从列表中选取第一个等待者并唤醒它的慢速路径：

.. slide:: :c:func:`mutex_unlock` 慢速路径
   :inline-contents: True
   :level: 2

   .. code-block:: c

      ...
      spin_lock(&lock->wait_lock);
      if (!list_empty(&lock->wait_list)) {
        /* 获得等待队列的第一个条目 */
        struct mutex_waiter *waiter;
        waiter = list_first_entry(&lock->wait_list, struct mutex_waiter,
                                  list);
	next = waiter->task;
	wake_q_add(&wake_q, next);
      }
      ...
      spin_unlock(&lock->wait_lock);
      ...
      wake_up_q(&wake_q);



CPU 独占数据
============

每个 CPU 独占数据通过避免使用共享数据来避免竟态条件。相反，使用一个大小为最大可能的 CPU 核心数的数组，并且每个核心将使用自己的数组条目来读取和写入数据。这种方法当然有优势：

.. slide:: 每个 CPU 独占的数据
   :inline-contents: True
   :level: 2

   * 无需同步即可访问数据

   * 没有争用，没有性能影响

   * 非常适合分布式处理，其中只偶尔需要聚合（例如统计计数器）


内存顺序和屏障
============================

现代处理器和编译器采用乱序执行来提高性能。例如，处理器可以在等待当前指令数据从内存中获取时执行“未来”指令。

以下是乱序编译器生成的代码示例：

.. slide:: 乱序编译器生成的代码
   :inline-contents: True
   :level: 2

   +-------------------+-------------------------+
   | C code            | Compiler generated code |
   +-------------------+-------------------------+
   |.. code-block:: c  |.. code-block:: asm      |
   |		       |			 |
   |   a = 1;          |  MOV R10, 1		 |
   |   b = 2;          |  MOV R11, 2		 |
   |                   |  STORE R11, b		 |
   |                   |  STORE R10, a		 |
   +-------------------+-------------------------+


.. note:: 当执行乱序指令时，处理器会确保数据依赖关系，即不会执行那些输入依赖于尚未执行的先前指令的输出的指令。

在大多数情况下，乱序执行不是一个问题。然而，在某些情况下（例如，在处理器之间或处理器与硬件之间通过共享内存进行通信），我们必须在没有数据依赖关系的情况下执行一些指令之前执行另一些指令。

为此，我们可以使用屏障来对内存操作进行排序：

.. slide:: 屏障
   :inline-contents: True
   :level: 2

   * 读屏障 (:c:func:`rmb()`，:c:func:`smp_rmb()`) 用于确保没有读操作越过屏障；也就是说，在执行屏障之后的第一条指令之前，所有的读操作都已经完成

   * 写屏障 (:c:func:`wmb()`，:c:func:`smp_wmb()`) 用于确保没有写操作越过屏障

   * 简单屏障（:c:func:`mb()`，:c:func:`smp_mb()`）用于确保没有读操作或写操作越过屏障


读-复制-更新（Read Copy Update，RCU）
======================

读-复制-更新是一种特殊的同步机制，类似于读写锁，但在某些方面有显著的改进（以及一些限制）：

.. slide:: 读-复制-更新（RCU）
   :level: 2
   :inline-contents: True

   * **只读**：同时进行无锁访问和写访问

   * 写访问仍然需要锁，以避免写者之间的竞争

   * 需要读者进行单向遍历


实际上，在 Linux 内核中，读写锁已经被弃用并移除，取而代之的是 RCU。

为新的数据结构实现 RCU 是困难的，但是一些常见的数据结构（如列表、队列、树）具有可以使用的 RCU API。

RCU 将数据结构的删除更新分为两个阶段：

.. slide:: 移除和回收
   :inline-contents: True
   :level: 2

   * **移除**：删除对元素的引用。一些旧的读者仍然可以看到旧的引用，因此我们不能释放该元素。

   * **消除**：释放元素。此操作被推迟直到所有现有的读者完成遍历（静默周期）。新的读者不会影响静默周期。


例如，让我们看一下如何使用 RCU 从列表中删除一个元素的示例：

.. slide:: RCU 列表删除
   :inline-contents: True
   :level: 2

   |_|

   .. ditaa::

         (1) List Traversal                          (2) Removal
                                                    +-----------+
      +-----+     +-----+     +-----+      +-----+  |  +-----+  |  +-----+
      |     |     |     |     |     |      |     |  |  |     |  |  |     |
      |  A  |---->|  B  |---->|  C  |      |  A  |--+  |  B  |--+->|  C  |
      |     |     |     |     |     |      |     |     |     |     |     |
      +-----+     +-----+     +-----+      +-----+     +-----+     +-----+
         ^           ^           ^            ^           ^           ^
         |           |           |            |           |           |







         (3) Quiescent cycle over                 (4) Reclamation
               +-----------+
      +-----+  |  +-----+  |  +-----+      +-----+                 +-----+
      |     |  |  |     |  |  |     |      |     |                 |     |
      |  A  |--+  |  B  |  +->|  C  |      |  A  |---------------->|  C  |
      |     |     |     |     |     |      |     |                 |     |
      +-----+     +-----+     +-----+      +-----+                 +-----+
         ^                       ^            ^                       ^
         |                       |            |                       |


在第一步中，可以看到在读者遍历列表时，所有元素都被引用。在第二步中，写者移除了元素 B。由于仍然有读者持有对其的引用，回收被推迟。在第三步中，静默周期刚刚过去，可以注意到没有对元素 B 的引用了。其他元素仍然有来自在元素被移除后开始列表遍历的读者的引用。在第四步中，我们最终执行回收（释放元素）。

现在我们已经介绍了 RCU 在高层次上的工作原理，让我们看一下用于遍历列表以及向列表中添加和删除元素的 API：


.. slide:: 列表 RCU API 速查表
   :inline-contents: True
   :level: 2

   .. code-block:: c

      /* 列表遍历 */
      rcu_read_lock();
      list_for_each_entry_rcu(i, head) {
        /* 不允许休眠、阻塞调用或上下文切换 */
      }
      rcu_read_unlock();


      /* 列表元素删除  */
      spin_lock(&lock);
      list_del_rcu(&node->list);
      spin_unlock(&lock);
      synchronize_rcu();
      kfree(node);

      /* 列表元素添加  */
      spin_lock(&lock);
      list_add_rcu(head, &node->list);
      spin_unlock(&lock);
