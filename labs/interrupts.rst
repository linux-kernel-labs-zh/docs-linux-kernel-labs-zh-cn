==========================
I/O 访问和中断
==========================

实验目标
==============

* 与外围设备进行通信
* 实现中断处理程序
* 将中断与进程上下文同步

关键词：IRQ，I/O 端口，I/O 地址，基地址，UART，request_region，release_region，inb，outb

背景信息
======================

外围设备通过写入和读取其寄存器来进行控制。通常，设备具有多个寄存器，这些寄存器可以在内存地址空间或 I/O 地址空间的连续地址中访问。连接到 I/O 总线的每个设备都有一组 I/O 地址，称为 I/O 端口。I/O 端口可以映射到物理内存地址，以便处理器可以通过直接与内存交互的指令与设备进行通信。为了简化，我们将直接使用 I/O 端口（而不映射到物理内存地址）与物理设备进行通信。

每个设备的 I/O 端口是由一组专用寄存器构成的，以提供统一的编程接口。因此，大多数设备都具有以下类型的寄存器：

* **控制寄存器**：接收设备命令
* **状态寄存器**：包含有关设备内部状态的信息
* **输入寄存器**：从中读取设备的数据
* **输出寄存器**：将数据写入其中以传输到设备

物理端口根据位数进行区分：它们可以是 8 位、16 位或 32 位端口。

例如，并行端口具有从基地址 0x378 开始的 8 个 8 位 I/O 端口。数据日志位于基地址（0x378），状态寄存器位于基地址 + 1（0x379），控制寄存器位于基地址 + 2（0x37a）。数据日志既是输入日志也是输出日志。

虽然有些设备可以完全仅通过 I/O 端口或特殊内存区域进行控制，但在某些情况下，这是不够的。需要解决的主要问题是某些事件发生的时刻无法预期，对此如果处理器（CPU）反复查询设备的状态（轮询）的话，是很低效的。解决这个问题的方法是使用中断请求（IRQ），它是一种硬件通知，用于通知处理器发生了特定的外部事件。

为了使 IRQ 有用，设备驱动程序必须实现处理程序，即处理中断的特定代码序列。由于在许多情况下可用的中断数有限，设备驱动程序必须按顺序处理中断：在使用中断之前必须请求中断，在不再需要时必须释放中断。此外，在某些情况下，设备驱动程序必须共享中断或与中断同步。所有这些将进一步讨论。

当我们需要访问中断例程（A）和进程上下文或下半部分上下文（bottom-half context）（B）中运行的代码之间的共享资源时，我们必须使用特殊的同步技术。在（A）中，我们需要使用自旋锁原语，在（B）中必须禁用中断并使用自旋锁原语。仅禁用中断是不够的，因为中断例程可以在运行（B）的处理器之外的处理器上运行。

仅使用自旋锁可能会导致死锁。在这种情况下的典型死锁示例是：

1. 在 X 处理器上运行一个进程，并获取锁
2. 在释放锁之前，在 X 处理器上发生中断
3. 中断处理程序将尝试获取锁，并进入死循环


访问硬件
=============

在 Linux 中，I/O 端口的访问在所有体系结构上都有实现，并且有几个可以使用的 API。

请求访问 I/O 端口
---------------------------

在访问 I/O 端口之前，我们首先必须请求访问权限，这是为了确保只同时有一个用户使用。为了做到这一点，我们必须使用 :c:func:`request_region` 函数：

.. code-block:: c

   #include <linux/ioport.h>

   struct resource *request_region(unsigned long first, unsigned long n,
				   const char *name);

要释放一个已经持有的区域，可以使用 :c:func:`release_region` 函数：

.. code-block:: c

   void release_region(unsigned long start, unsigned long n);


例如，串口 COM1 的基地址是 0x3F8，它有 8 个端口，以下是请求访问这些端口的代码片段：

.. code-block:: c

   #include <linux/ioport.h>

   #define MY_BASEPORT 0x3F8
   #define MY_NR_PORTS 8

   if (!request_region(MY_BASEPORT, MY_NR_PORTS, "com1")) {
	/* 处理错误 */
	return -ENODEV;
   }

要释放端口，可以使用以下代码：

.. code-block:: c

   release_region(MY_BASEPORT, MY_NR_PORTS);

大多数情况下，端口请求是在驱动程序初始化或探测时进行的，端口释放是在设备或模块移除时进行的。

所有的端口请求可以在用户空间通过 :file:`/proc/ioports` 文件查看：

.. code-block:: shell

   $ cat /proc/ioports
   0000-001f : dma1
   0020-0021 : pic1
   0040-005f : timer
   0060-006f : keyboard
   0070-0077 : rtc
   0080-008f : dma page reg
   00a0-00a1 : pic2
   00c0-00df : dma2
   00f0-00ff : fpu
   0170-0177 : ide1
   01f0-01f7 : ide0
   0376-0376 : ide1
   0378-037a : parport0
   037b-037f : parport0
   03c0-03df : vga+
   03f6-03f6 : ide0
   03f8-03ff : serial
   ...


访问 I/O 端口
-------------------

在驱动程序获取所需的 I/O 端口范围之后，可以对这些端口进行读取或写入操作。由于物理端口根据位数（8 位、16 位或 32 位）进行区分，因此根据其大小有不同的端口访问函数。在 asm/io.h 中定义了以下端口访问函数：

* *unsigned inb(int port)*，从端口读取一个字节（8 位）
* *void outb(unsigned char byte, int port)*，向端口写入一个字节（8 位）
* *unsigned inw(int port)*，从端口读取两个字节（16 位）
* *void outw(unsigned short word, int port)*，向端口写入两个字节（16 位）
* *unsigned inl (int port)*，从端口读取四个字节（32 位）
* *void outl(unsigned long word, int port)*，向端口写入四个字节（32 位）

端口参数指定进行读取或写入的端口地址，其类型取决于平台（可以是 unsigned long 或 unsigned short）。

如果处理器过快地传输数据到设备或从设备读取数据的话，某些设备可能会出现问题。为了避免这个问题，我们可能需要在 I/O 操作之后插入延迟。可以使用引入延迟的函数来实现这一点。它们的名称与上述函数类似，不同之处在于以 _p 结尾：inb_p、outb_p 等。

例如，以下序列在 COM1 串口上写入一个字节，然后读取它：

.. code-block:: c

   #include <asm/io.h>
   #define MY_BASEPORT 0x3F8

   unsigned char value = 0xFF;
   outb(value, MY_BASEPORT);
   value = inb(MY_BASEPORT);

5. 从用户空间访问 I/O 端口
------------------------

虽然上述描述的函数是为设备驱动程序定义的，但也可以通过包含 <sys/io.h> 头文件在用户空间中使用。为了使用它们，首先必须调用 ioperm 或 iopl 以获得执行端口操作的权限。ioperm 函数获取单个端口的权限，而 iopl 获取整个 I/O 地址空间的权限。要使用这些功能，用户必须具有 root 权限。

以下是在用户空间中使用的代码，用于获取串口的前 3 个端口的权限，然后释放它们：

.. code-block:: c

   #include <sys/io.h>
   #define MY_BASEPORT 0x3F8

   if (ioperm(MY_BASEPORT, 3, 1)) {
	/* 处理错误 */
   }

   if (ioperm(MY_BASEPORT, 3, 0)) {
	/* 处理错误 */
   }

ioperm 函数的第三个参数用于请求或释放端口权限：1 表示获取权限，0 表示释放权限。

中断处理
==================

请求中断
-----------------------

与其他资源一样，驱动程序在使用中断线之前必须获得对其的访问权限，并在执行结束时释放它。

在 Linux 中，使用 :c:func:`request_irq` 和 :c:func:`free_irq` 函数来请求和释放中断：

.. code-block:: c

   #include <linux/interrupt.h>

   typedef irqreturn_t (*irq_handler_t)(int, void *);

   int request_irq(unsigned int irq_no, irq_handler_t handler,
		   unsigned long flags, const char *dev_name, void *dev_id);

   void free_irq(unsigned int irq_no, void *dev_id);

请注意，要获取中断，开发者需调用 :c:func:`request_irq` 。调用此函数时，必须指定中断号 (*irq_no*)，在生成中断时调用的处理程序 (*handler*)，指示内核所需行为的标志 (*flags*)，使用此中断的设备的名称 (*dev_name*)，以及可以由用户配置成任何值的指针，并且没有全局意义 (*dev_id*)。大多数情况下, *dev_id* 是指向设备驱动程序的私有数据的指针。当使用 :c:func:`free_irq` 函数释放中断时，开发人员必须发送相同的指针值 (*dev_id*) 以及相同的中断号 (*irq_no*)。设备名称 (*dev_name*) 用于在 */proc/interrupts* 中显示统计信息。

:c:func:`request_irq` 返回的值为 0 表示成功，或者为负数错误代码，表示失败的原因。一个典型的值是 *-EBUSY*，表示中断已经被另一个设备驱动程序请求。

*handler* 函数在中断上下文中执行，这意味着我们不能调用阻塞的 API，如 :c:func:`mutex_lock` 或 :c:func:`msleep`。我们还必须避免在中断处理程序中执行大量工作，最好使用延迟工作。在中断处理程序中执行的操作包括读取设备寄存器以获取设备的状态并确认中断，这些操作大多数情况下可以使用非阻塞调用来执行。

有时候，即使设备使用中断，我们也无法以非阻塞模式读取设备的寄存器（例如连接到 I2C 或 SPI 总线的传感器，其驱动程序不能保证总线读/写操作是非阻塞的）。在这种情况下，我们必须在中断中规划一个正在进行的工作（工作队列、内核线程）来访问设备的寄存器。由于这种情况相对常见，内核提供了 :c:func:`request_threaded_irq` 函数来编写以两个阶段运行的中断处理程序：一个是进程阶段，另一个是中断上下文阶段：

.. code-block:: c

   #include <linux/interrupt.h>

   int request_threaded_irq(unsigned int irq, irq_handler_t handler,
			    irq_handler_t thread_fn,
			    unsigned long flags, const char *name, void *dev);

*handler* 函数在中断上下文中运行，并实现关键操作，而 thread_fn 函数在进程上下文中运行，并实现其余操作。

当进行中断请求时，可以传递的标志如下：

* *IRQF_SHARED* 通知内核该中断可以与其他设备共享。如果未设置此标志，并且请求的中断已经与其他处理程序关联，则中断请求将失败。内核以特殊方式处理共享中断：所有关联的中断处理程序将被执行，直到识别出生成中断的设备。但是，设备驱动程序如何知道中断处理例程是否是由其管理的设备生成的中断激活的呢？几乎所有支持中断的设备都有一个状态寄存器，可以在处理例程中查询该寄存器，以查看中断是否由设备生成（例如，在 8250 串口的情况下，该状态寄存器是 IIR（中断信息寄存器））。当请求共享中断时，dev_id 参数必须是唯一的，且不能为 NULL。通常将其设置为模块的私有数据。

* *IRQF_ONESHOT* 在进程上下文例程运行完成后，中断将重新激活；如果没有设置此标志，则中断将在中断上下文中的处理程序运行完成后重新激活。


可以在驱动程序的初始化（:c:func:`init_module`）时、设备探测时或设备使用时（例如设备打开时）执行中断请求。

以下示例执行了 COM1 串口的中断请求：

.. code-block:: c

   #include <linux/interrupt.h>

   #define MY_BASEPORT 0x3F8
   #define MY_IRQ 4

   static my_init(void)
   {
	[...]
	struct my_device_data *my_data;
	int err;

	err = request_irq(MY_IRQ, my_handler, IRQF_SHARED,
			  "com1", my_data);
	if (err < 0) {
	    /* 处理错误 */
	    return err;
	}
	[...]
   }

如你所见，串口 COM1 的中断号是 4，在共享模式（IRQF_SHARED）下使用。

.. attention:: 在请求共享中断（IRQF_SHARED）时，*dev_id* 参数不能为空。

要释放与串口关联的中断，需要执行以下操作：

.. code-block:: c

   free_irq (MY_IRQ, my_data);


在初始化函数（:c:func:`init_module`）或打开设备的函数中，必须激活设备的中断。具体操作取决于设备，但通常涉及设置控制寄存器的某个位。

以 8250 串口为例，要启用中断，必须执行以下操作：

.. code-block:: c

   #include <asm/io.h>
   #define MY_BASEPORT 0x3F8

   outb(0x08, MY_BASEPORT+4);
   outb(0x01, MY_BASEPORT+1);

在上面的示例中，执行了两个操作：

1. 通过设置 MCR 寄存器（调制解调控制寄存器）中的位 3（Aux Output 2），激活所有中断。
2. 通过在 IER 寄存器（中断使能寄存器）中设置适当的位，激活 RDAI（传输保持寄存器空中断）。


中断处理程序的实现
---------------------------

让我们来看一下中断处理程序函数的签名：

.. code-block:: c

   irqreturn_t (*handler)(int irq_no, void *dev_id);

该函数接收中断号 (*irq_no*) 和在请求中断时传递给 :c:func:`request_irq` 函数的指针作为参数。中断处理例程必须返回具有 :c:type:`typedef irqreturn_t` 类型的值。对于当前的内核版本，中断处理函数可以返回三种不同的值: *IRQ_NONE*, *IRQ_HANDLED* 或 *IRQ_WAKE_THREAD*。如果中断处理函数发现中断并不是由它所负责的设备触发的，它应该返回 *IRQ_NONE* 表示没有处理中断。如果中断处理函数能够在中断上下文中完成对中断的处理，它应该返回 *IRQ_HANDLED* 表示已经处理了中断。如果中断处理函数需要在进程上下文中继续处理中断，它应该返回 *IRQ_WAKE_THREAD* 表示需要调度执行进程上下文处理函数。

中断处理程序的基本框架如下：

.. code-block:: c

   irqreturn_t my_handler(int irq_no, void *dev_id)
   {
       struct my_device_data *my_data = (struct my_device_data *) dev_id;

      /* 如果中断不是针对该设备的（共享中断） */
      /* 返回 IRQ_NONE; */

      /* 清除中断挂起位 */
      /* 从设备读取或向设备写入数据 */

       return IRQ_HANDLED;
   }


通常，在中断处理程序中首先执行的操作，是确定中断是否由驱动程序所控制的设备触发的。这通常通过从设备的寄存器中读取信息来判断设备是否触发中断。其次，需要重置物理设备上的中断挂起位（interrupt pending bit），因为大多数设备在该位重置之前不会再触发中断（例如，对于 8250 串口，必须清除 IIR 寄存器中的第 0 位）。


锁定
-------

因为中断处理程序在中断上下文中运行，所以可以执行的操作很有限：无法访问用户空间内存，不能调用阻塞函数。此外，使用自旋锁进行同步是很棘手的事，如果某个进程获取了自旋锁，之后被运行处理程序中断，则可能导致死锁。

然而，设备驱动程序在某些情况下必须使用中断进行同步，例如当数据在中断处理程序和进程上下文或底半部处理程序之间共享时。在这些情况下，需要同时禁用中断和使用自旋锁。

有两种禁用中断的方法：在处理器级别禁用所有中断，或在设备或中断控制器级别禁用特定中断。处理器禁用更快，因此更受推荐。为此，有一些锁定函数可以在同时获取和释放自旋锁的同时禁用和启用中断: :c:func:`spin_lock_irqsave`, :c:func:`spin_unlock_irqrestore`, :c:func:`spin_lock_irq` 和 :c:func:`spin_unlock_irq`：

.. code-block:: c

   #include <linux/spinlock.h>

   void spin_lock_irqsave (spinlock_t * lock, unsigned long flags);
   void spin_unlock_irqrestore (spinlock_t * lock, unsigned long flags);

   void spin_lock_irq (spinlock_t * lock);
   void spin_unlock_irq (spinlock_t * lock);

:c:func:`spin_lock_irqsave` 函数在获取自旋锁之前禁用本地处理器上的中断；中断的先前状态保存在 *flags* 中。

如果你绝对确定当前处理器上的中断尚未被其他人禁用，并且确定在释放自旋锁时可以激活中断，可以使用 :c:`spin_lock_irq`。

对于读/写自旋锁，也有类似的函数可用：

* :c:func:`read_lock_irqsave`
* :c:func:`read_unlock_irqrestore`
* :c:func:`read_lock_irq`
* :c:func:`read_unlock_irq`
* :c:func:`write_lock_irqsave`
* :c:func:`write_unlock_irqrestore`
* :c:func:`write_lock_irq`
* :c:func:`write_unlock_irq`

如果我们想在中断控制器级别禁用中断（不推荐，因为禁用特定中断会更慢，而且无法禁用共享中断），可以使用 :c:func:`disable_irq`, :c:func:`disable_irq_nosync` 和 :c:func:`enable_irq`。使用这些函数将禁用所有处理器上的中断。调用可以嵌套：如果调用了两次 disable_irq，则需要调用相同次数的 enable_irq 才能启用中断。disable_irq 和 disable_irq_nosync 之间的区别在于前者将等待执行中的处理程序完成。因此, :c:func:`disable_irq_nosync` 通常更快，但可能与中断处理程序竞争，因此如果不确定请使用 :c:func:`disable_irq`。

以下代码禁用然后启用 COM1 串口的中断：

.. code-block:: c

   #define MY_IRQ 4

   disable_irq (MY_IRQ);
   enable_irq (MY_IRQ);

也可以在设备级别上禁用中断。这种方法同样比在处理器级别上禁用中断更慢，但它适用于共享中断。实现这个的方式因设备而异，通常需要从控制寄存器中清除一个位。

也可以独立于锁定操作，在当前处理器上禁用所有中断。出于同步目的而由设备驱动程序禁用所有中断是不合适的，因为如果中断在另一个 CPU 上处理，仍然可能出现竞态条件。用于在本地处理器上禁用/启用中断的函数是 :c:func:`local_irq_disable` 和 :c:func:`local_irq_enable`。

为了使用在进程上下文和中断处理例程之间共享的资源，可以按照以下方式使用上述函数：

.. code-block:: c

   static spinlock_t lock;

   /* IRQ 处理例程：中断上下文 */
   irqreturn_t kbd_interrupt_handle(int irq_no, void * dev_id)
   {
       ...
       spin_lock(&lock);
       /* 临界区——访问共享资源 */
       spin_unlock (&lock);
       ...
   }

   /* 进程上下文：在锁定时禁用中断 */
   static void my_access(void)
   {
       unsigned long flags;

       spin_lock_irqsave(&lock, flags);
       /* 临界区——访问共享资源 */
       spin_unlock_irqrestore(&lock, flags);

       ...
   }

   void my_init (void)
   {
       ...
       spin_lock_init (&lock);
       ...
   }


上面的 *my_access* 函数在进程上下文中运行。为了同步对共享数据的访问，我们禁用中断并使用自旋锁 *lock*，即 :c:func:`spin_lock_irqsave` 和 :c:func:`spin_unlock_irqrestore` 函数。

在中断处理例程中，我们使用 :c:func:`spin_lock` 和 :c:func:`spin_unlock` 函数来访问共享资源。

.. note:: :c:func:`spin_lock_irqsave` 和 :c:func:`spin_unlock_irqrestore` 的 *flags* 参数是一个值而不是指针，但请记住 :c:func:`spin_lock_irqsave` 函数会更改标志的值，因为这实际上是一个宏。

中断统计
--------------------

关于系统中断的信息和统计数据可以在 */proc/interrupts* 或 */proc/stat* 中找到。只有具有关联中断处理程序的系统中断才会显示在 */proc/interrupts* 中：

.. code-block:: shell

   # cat /proc/interrupts
		   CPU0
   0:           7514294       IO-APIC-edge   timer
   1:              4528       IO-APIC-edge   i8042
   6:                 2       IO-APIC-edge   floppy
   8:                 1       IO-APIC-edge   rtc
   9:                 0       IO-APIC-level  acpi
   12:             2301       IO-APIC-edge   i8042
   15:               41       IO-APIC-edge   ide1
   16:             3230       IO-APIC-level  ioc0
   17:             1016       IO-APIC-level  vmxnet ether
   NMI:               0
   LOC:         7229438
   ERR:               0
   MIS:               0

第一列指定了与中断相关联的 IRQ。接下来的列显示了系统中每个处理器生成的中断次数；最后两列提供了有关中断控制器和注册该中断处理程序的设备名称的信息。

*/proc/stat* 文件提供了有关系统活动的信息，包括自上次（重新）启动系统以来生成的中断次数：

.. code-block:: shell

   # cat /proc/stat | grep in
   intr 7765626 7754228 4620 0 0 0 0 2 0 1 0 0 0 2377 0 0 41 3259 1098 0 0 0 0 0 0 0 0 0
   0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
   0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0

*/proc/stat* 文件中的每一行都以关键字开头，指定了该行信息的含义。对于中断信息，这个关键字是 intr。行上的第一个数字表示中断的总数，其他数字表示从 0 开始的每个 IRQ 的中断次数。计数器包括系统中所有处理器的中断次数。


进一步阅读
===============

串口
-----------

* `串口 <http://en.wikipedia.org/wiki/Serial_port>`_
* `串口/RS232 端口接口 <http://www.beyondlogic.org/serial/serial.htm>`_


并行端口
-------------

* `标准并行端口接口 <http://www.beyondlogic.org/spp/parallel.htm>`_
* `并行端口中心 <http://www.lvr.com/parport.htm>`_

键盘控制器
-------------------

* `Intel 8042 <http://en.wikipedia.org/wiki/Intel_8042>`_
* drivers/input/serio/i8042.c
* drivers/input/keyboard/atkbd.c

Linux设备驱动程序
--------------------

* `Linux 设备驱动程序，第 3 版，第 9 章——与硬件通信 <http://lwn.net/images/pdf/LDD3/ch09.pdf>`_
* `Linux 设备驱动程序，第 3 版，第 10 章——中断处理 <http://lwn.net/images/pdf/LDD3/ch10.pdf>`_
* `中断处理程序 <http://tldp.org/LDP/lkmpg/2.6/html/x1256.html>`_


练习
=========

.. include:: ../labs/exercises-summary.hrst
.. |LAB_NAME| replace:: 中断

0. 简介
--------

使用 |LXR|_，在 Linux 内核中查找以下符号的定义：

* :c:type:`struct resource`
* :c:func:`request_region` 和 :c:func:`__request_region`
* :c:func:`request_irq` 和 :c:func:`request_threaded_irq`
* :c:func:`inb`（适用于 x86 架构）

分析以下 Linux 代码：

* 键盘初始化函数 :c:func:`i8042_setup_kbd`
* AT 或 PS/2 键盘中断函数 :c:func:`atkbd_interrupt`

键盘驱动程序
------------------

下一个练习的目标是创建一个使用键盘 IRQ 的驱动程序，检查传入的按键代码并将其存储在缓冲区中。通过字符设备驱动程序，用户空间可以访问该缓冲区。

1. 请求 I/O 端口
------------------------

首先，我们的目标是在 I/O 空间中为硬件设备分配内存。我们看到，我们无法为键盘分配空间，因为指定的区域已经被分配。然后，我们将为未使用的端口分配 I/O 空间。

*kbd.c* 文件中包含了键盘驱动程序的框架。浏览源代码并检查 :c:func:`kbd_init` 函数。注意我们需要的 I/O 端口是 I8042_STATUS_REG 和 I8042_DATA_REG。

按照骨架中标有 **TODO 1** 的部分进行操作。在 :c:func:`kbd_init` 函数中请求 I/O 端口，并确保检查错误并在出现错误时进行适当的清理。在请求时，使用 ``MODULE_NAME`` 宏设置调用者的 ID 字符串（``name``）设置为该宏的值。此外，在 :c:func:`kbd_exit` 函数中添加代码以释放 I/O 端口。

.. note:: 在继续之前，你可以回顾一下 `请求访问 I/O 端口`_ 部分。

现在构建模块并将其复制到虚拟机镜像中：

.. code-block:: shell

   tools/labs $ make build
   tools/labs $ make copy


现在启动虚拟机并插入模块：

.. code-block:: shell

   root@qemux86:~# insmod skels/interrupts/kbd.ko
   kbd: loading out-of-tree module taints kernel.
   insmod: can't insert 'skels/interrupts/kbd.ko': Device or resource busy

注意，在尝试请求 I/O 端口时会出现错误。这是因为我们已经有了一个请求此 I/O 端口的驱动程序。为了验证，请查看 :file:`/proc/ioports` 文件以查找 ``STATUS_REG`` 和 ``DATA_REG`` 的值：

.. code-block:: shell

   root@qemux86:~# cat /proc/ioports | egrep "(0060|0064)"
   0060-0060 : keyboard
   0064-0064 : keyboard


让我们找出是哪个驱动程序注册了这些端口，并尝试移除与之关联的模块。

.. code-block:: shell

   $ find -name \*.c | xargs grep \"keyboard\"

   find -name \*.c | xargs grep \"keyboard\" | egrep '(0x60|0x64)'
   ...
   ./arch/x86/kernel/setup.c:{ .name = "keyboard", .start = 0x60, .end = 0x60,
   ./arch/x86/kernel/setup.c:{ .name = "keyboard", .start = 0x64, .end = 0x64

看起来这些 I/O 端口是由内核在启动期间注册的，我们将无法移除与之关联的模块。相反，让我们欺骗内核并注册端口 0x61 和 0x65。

在 :c:func:`kbd_init` 函数中使用函数 :c:func:`request_region` 来分配这些端口，并在 :c:func:`kbd_exit` 函数中使用函数 :c:func:`release_region` 来释放分配的内存。

这次我们可以加载模块，并且 */proc/ioports* 显示这些端口的所有者是我们的模块：

.. code-block:: shell

   root@qemux86:~# insmod skels/interrupts/kbd.ko
   kbd: loading out-of-tree module taints kernel.
   Driver kbd loaded
   root@qemux86:~# cat /proc/ioports | grep kbd
   0061-0061 : kbd
   0065-0065 : kbd

让我们移除模块并检查 I/O 端口是否已释放：

.. code-block:: shell

   root@qemux86:~# rmmod kbd
   Driver kbd unloaded
   root@qemux86:~# cat /proc/ioports | grep kbd
   root@qemux86:~#

2. 中断处理例程
-----------------------------

对于这个任务，我们将实现并注册一个键盘中断的中断处理例程。在继续之前，你可以先回顾一下 `请求中断`_ 一节。

请按照骨架中标有 **TODO 2** 的部分进行操作。

首先，定义一个名为 :c:func:`kbd_interrupt_handler` 的空中断处理例程。

.. note:: 由于我们已经有一个使用该中断的驱动程序，我们应该将中断报告为未处理（即返回 :c:type:`IRQ_NONE`），以便原始驱动程序仍有机会进行处理。

然后，使用 :c:type:`request_irq` 注册中断处理例程。中断号由 `I8042_KBD_IRQ` 宏定义。中断处理例程必须使用 :c:type:`IRQF_SHARED` 进行请求，以与键盘驱动程序（i8042）共享中断线。

.. note:: 对于共享中断, *dev_id* 不能为 NULL。请使用 ``&devs[0]``，即 :c:type:`struct kbd` 的指针。此结构包含了设备管理所需的所有信息。为了在 */proc/interrupts* 中看到该中断，请不要使用 NULL 作为 *dev_name* 。你可以使用 MODULE_NAME 宏。

          如果中断请求失败，请确保通过跳转到正确的标签（label）来进行适当的清理，即释放 I/O 端口并注销字符设备驱动程序。

编译、复制并加载模块到内核中。通过查看 */proc/interrupts*，检查中断线是否已注册。从源代码中确定 IRQ 号码（参见 `I8042_KBD_IRQ`）并验证该中断线上有两个注册的驱动程序（这表示我们有一个共享中断线）：i8042 初始驱动程序和我们的驱动程序。

.. ::note:: 关于 */proc/interrupts* 格式的更多详细信息可以在 `中断统计`_ 一节中找到。

在例程内部打印一条消息，以确保它被调用。将模块编译并重新加载到内核中。使用 :command:`dmesg` 检查在虚拟机上按键时是否调用了中断处理例程。还要注意，当使用串口时不会触发键盘中断。

.. attention:: 要访问虚拟机上的键盘，请使用“QEMU_DISPLAY=gtk make boot”启动。

3. 将 ASCII 键存储到缓冲区
-----------------------------

接下来，我们希望收集按键的输入到缓冲区里，并将其内容发送到用户空间。为此，我们将在中断处理中添加以下内容：

* 捕获按下的键（只捕获按下的键，忽略释放的键）
* 识别 ASCII 字符
* 将与按键对应的 ASCII 字符复制并存储在设备的缓冲区中

请按照骨架中标记为 **TODO 3** 的部分进行操作。

读取数据寄存器
.........................

首先，填写 :c:func:`i8042_read_data` 函数，以读取键盘控制器的 ``I8042_DATA_REG`` 寄存器。该函数只需要返回寄存器的值。寄存器的值也称为扫描码（scancode），它在每次按键时生成。

.. hint:: 使用 :c:func:`inb` 读取 ``I8042_DATA_REG`` 寄存器，并将值存储在局部变量 :c:type:`val` 中。请参阅 `访问 I/O 端口`_ 部分。

在 :c:func:`kbd_interrupt_handler` 中调用 :c:func:`i8042_read_data` 并打印读取的值。

按以下格式打印有关按键的信息：

.. code-block:: c

   pr_info("IRQ:% d, scancode = 0x%x (%u,%c)\n",
      irq_no, scancode, scancode, scancode);


其中，scancode，即扫描码，是使用 :c:func:`i8042_read_data` 函数读取的寄存器的值。

请注意，扫描码（读取的寄存器的值）不是按下键的 ASCII 字符。我们需要理解扫描码。

解释扫描码
.........................

请注意，寄存器值是扫描码，而不是按下的字符的 ASCII 值。还要注意，中断在按键按下和释放时都会发送。我们只需要在按键按下时获取扫描码，然后解码 ASCII 字符。

.. note:: 要检查扫描码，可以使用 showkey 命令（showkey -s）。

      命令将在按下键后显示 10 秒钟的键扫描码，然后停止。如果按下并释放一个键，你将获得两个扫描码：一个对应按下的键，一个对应释放的键。例如：

      * 如果按下回车键，你将获得 0x1c（0x1c）和 0x9c（释放键）。
      * 如果按下键 a，你将获得 0x1e（按下的键）和 0x9e（释放键）。
      * 如果按下键 b，你将获得 0x30（按下的键）和 0xb0（释放键）。
      * 如果按下键 c，你将获得 0x2e（按下的键）和 0xae（释放键）。
      * 如果按下 Shift 键，你将获得 0x2a（按下的键）和 0xaa（释放键）。
      * 如果按下 Ctrl 键，你将获得 0x1d（按下的键）和 0x9d（释放键）。

        正如在 `这篇文章 <http://www.linuxjournal.com/article/1080>`_ 中所指出的，释放键的扫描码比按下键的扫描码高 128（0x80）。这是我们区分按下键的扫描码和释放键的扫描码的方法。

        扫描码被转换为与键匹配的键码（keycode）。按下的扫描码和释放的扫描码具有相同的键码。对于上面显示的键，我们有以下表格：

        .. flat-table::

           * - 键
             - 按下的扫描码
             - 释放的扫描码
             - 键码

           * - 回车
             - 0x1c
             - 0x9c
             - 0x1c（28）

           * - a
             - 0x1e
             - 0x9e
             - 0x1e（30）

           * - b
             - 0x30
             - 0xb0
             - 0x30（48）

           * - c
             - 0x2e
             - 0xae
             - 0x2e（46）

           * - Shift
             - 0x2a
             - 0xaa
             - 0x2a（42）

           * - Ctrl
             - 0x1d
             - 0x9d
             - 0x1d（29）

        按键按下/释放操作在 is_key_press() 函数中执行，获取扫描码的 ASCII 字符在 get_ascii() 函数中进行。

在中断处理程序中，先检查扫描码以确定按键是按下还是释放，然后确定相应的 ASCII 字符。

.. hint:: 要检查按下/释放，请使用 :c:func:`is_key_press` 函数。使用 :c:func:`get_ascii` 函数获取相应的 ASCII 码。这两个函数都以扫描码作为参数。

.. hint:: 要显示接收到的信息，请使用以下格式。

   .. code-block:: c

      pr_info("IRQ %d: scancode=0x%x (%u) pressed=%d ch=%c\n",
              irq_no, scancode, scancode, pressed, ch);

   其中，scancode 是数据寄存器的值，ch 是 get_ascii() 函数返回的值。

将字符存储到缓冲区
...............................

我们希望将按下的字符（而不是其他键）收集到一个循环缓冲区（circular buffer）中，以便可以从用户空间中使用。

更新中断处理程序，将按下的 ASCII 字符添加到设备缓冲区的末尾。如果缓冲区已满，则将丢弃该字符。

.. hint:: 设备缓冲区是设备的 :c:type:`struct kbd` 中的字段 :c:type:`buf`。要从中断处理程序中获取设备数据，请使用以下结构：

	  .. code-block:: c

	    struct kbd *data = (struct kbd *) dev_id;

	  缓冲区的大小位于 :c:type:`struct kbd` 的字段 :c:type:`count` 中。:c:type:`put_idx` 和 :c:type:`get_idx` 字段指定下一个写入和读取的索引。查看 :c:func:`put_char` 函数的实现，了解数据是如何添加到循环缓冲区中的。

.. attention:: 使用自旋锁对缓冲区和辅助索引进行同步访问。在设备结构体 :c:type:`struct kbd` 中定义自旋锁，并在 :c:func:`kbd_init` 中进行初始化。

	       使用 :c:func:`spin_lock` 和 :c:func:`spin_unlock` 函数来保护中断处理程序中的缓冲区。

	       请参阅 `锁定`_ 小节。

4. 读取缓冲区
----------------------

为了访问键盘记录器的数据，我们需要将其发送到用户空间。我们将使用 */dev/kbd* 字符设备来实现这一点。当从该设备读取数据时，我们将从内核空间的缓冲区中获取按键数据。

在这一步中，请按照 :c:func:`kbd_read` 函数中标有 **TODO 4** 的部分进行操作。

:c:func:`get_char` 的实现类似于 :c:func:`put_char` 。在实现循环缓冲区时要小心。

在 :c:func:`kbd_read` 函数中，将数据从缓冲区复制到用户空间缓冲区。

.. hint:: 使用 :c:func:`get_char` 从缓冲区中读取一个字符，并使用 :c:func:`put_user` 将其存储到用户缓冲区中。

.. attention:: 在读取函数中，使用 :c:func:`spin_lock_irqsave` 和 :c:func:`spin_unlock_irqrestore` 进行加锁。

	       请参阅 `锁定`_ 部分。

.. attention:: 我们不能在持有锁的情况下使用 :c:func:`put_user` 或 :c:func:`copy_to_user`，因为在原子上下文中不允许访问用户空间。

	       有关更多信息，请阅读前面实验中的 :ref:`访问进程地址空间 <_access_to_process_address_space>`。

要进行测试，你需要在读取之前使用 mknod 创建 */dev/kbd* 字符设备驱动程序。设备的主设备号和次设备号定义为 ``KBD_MAJOR`` 和 ``KBD_MINOR``：

.. code-block:: c

   mknod /dev/kbd c 42 0

构建、复制和启动虚拟机，并加载该模块。使用以下命令进行测试：

.. code-block:: c

   cat /dev/kbd


5. 重置缓冲区
-------------------

如果对设备进行写操作，则重置缓冲区。在这一步中，请按照骨架中标有 **TODO 5** 的部分进行操作。

实现 :c:func:`reset_buffer` 并将写操作添加到 *kbd_fops* 中。

.. attention:: 在写函数中，当重置缓冲区时，请使用 :c:func:`spin_lock_irqsave` 和 :c:func:`spin_unlock_irqrestore` 进行加锁。

         请参阅 `锁定`_ 部分。

为了进行测试，你需要在读取之前使用 mknod 创建 */dev/kbd* 字符设备驱动程序。设备的主设备号和次设备号定义为 ``KBD_MAJOR`` 和 ``KBD_MINOR``：

.. code-block:: c

   mknod /dev/kbd c 42 0

构建、复制和启动虚拟机，并加载该模块。使用以下命令进行测试：

.. code-block:: c

   cat /dev/kbd

按下一些键，然后运行命令 :command:`echo "clear" > /dev/kbd`。再次检查缓冲区的内容。它应该重置了。

额外练习
==========

1. kfifo
---------

使用 `kfifo API <https://elixir.bootlin.com/linux/v4.15/source/include/linux/kfifo.h>`_ 实现一个键盘记录器。

.. hint:: 参考内核代码中的 `API 调用示例 <https://elixir.bootlin.com/linux/v4.15/source/samples/kfifo>`_。例如，文件 `bytestream-examples.c <https://elixir.bootlin.com/linux/v4.15/source/samples/kfifo/bytestream-example.c>`_。
