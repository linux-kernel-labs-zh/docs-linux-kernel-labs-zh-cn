=======
系统调用
=======

`查看幻灯片 <syscalls-slides.html>`_

.. slideconf::
   :autoslides: False
   :theme: single-level

课程目标：
===================

.. slide:: System Calls
   :inline-contents: True
   :level: 2

   * Linux 系统调用实现

   * VDSO and 虚拟系统调用

   * 通过系统调用访问用户空间



Linux 系统调用实现
=================================

从高层视角来看，系统调用是内核向用户应用程序提供的“服务”，它们类似于库 API，因为它们被描述为具有名称、参数和返回值的函数调用。

.. slide:: System Calls as Kernel services
   :inline-contents: True
   :level: 2

   |_|

   .. code-block:: text

           +-------------+           +-------------+
           |   应用程序   |           |   应用程序   |
           +-------------+           +-------------+
             |                           |
             |read(fd, buff, len)        |fork()
             |                           |
             v                           v
           +---------------------------------------+
           |                 内核                  |
           +---------------------------------------+


然而，从底层视角看的话，我们会发现系统调用实际上并不是函数调用，而是特定的汇编指令（与体系结构和内核相关），其功能如下：

.. slide:: System Call Setup
   :inline-contents: True
   :level: 2

   * 用于识别系统调用及其参数的设置信息

   * 触发内核模式切换

   * 获取系统调用的结果

在 Linux 中，系统调用使用数字进行标识，系统调用的参数为机器字大小（32位或64位）。最多可以有6个系统调用参数。系统调用编号和参数都存储在特定的寄存器中。

例如，在 32 位的 x86 架构中，系统调用标识符存储在 EAX 寄存器中，而参数存储在 EBX、ECX、EDX、ESI、EDI 和 EBP寄存器中。

.. slide:: Linux system call setup
   :inline-contents: False
   :level: 2

   * 系统调用通过数字进行标识

   * 系统调用的参数为机器字大小（32位或64位）并且最多可以有6个系统调用参数。

   * 使用寄存器同时存储它们（例如，对于 32 位 x86 架构：系统调用标识符使用 EAX 寄存器，参数使用 EBX、ECX、EDX、ESI、EDI 和 EBP 寄存器）。

系统库（例如 libc 库）提供系统库（例如 libc）提供了函数来实现实际的系统调用，以便应用程序更容易使用它们。

当用户到内核模式的转换发生时，执行流程会被中断，并传递到内核的入口点。这类似于中断和异常的处理方式（实际上，在某些架构上，这种转换正是由异常引起的）。

系统调用入口点会将寄存器（其中包含来自用户空间的值，包括系统调用标识符和系统调用参数）保存在堆栈上，然后继续执行系统调用分发器（system call dispatcher）。

.. note:: 在用户模式和内核模式之间的切换过程中，还会将堆栈从用户堆栈切换到内核堆栈。这一点在中断课程中有更详细的解释。

.. slide:: Example of Linux system call setup and handling
   :inline-contents: True
   :level: 2

   .. code-block:: text

           +-------------+   dup2    +-----------------------------+
           |   应用程序   |-----+     |  libc                       |
           +-------------+     |     |                             |
                               +---->| C7590 dup2:                 |
                                     | ...                         |
                                     | C7592 movl 0x8(%esp),%ecx   |
                                     | C7596 movl 0x4(%esp),%ebx   |
                                     | C759a movl $0x3f,%eax       |
      +------------------------------+ C759f int $0x80             |
      |                              | ...                         +<-----+
      |                              +-----------------------------+   	  |
      |								      	  |
      |						  		    	  |
      |								    	  |
      |								    	  |
      |    +------------------------------------------------------------+ |
      |    |                         内核                               | |
      |    |                                                            | |
      +--->|ENTRY(entry_INT80_32)                                       | |
           | ASM_CLAC                                                   | |
           | pushl   %eax                    # pt_regs->orig_ax         | |
           | SAVE_ALL pt_regs_ax=$-ENOSYS    # save rest                | |
           | ...                                                        | |
           | movl   %esp, %eax                                          | |
           | call   do_int80_syscall_32                                 | |
           | ....                                                       | |
           | RESTORE_REGS 4                  # skip orig_eax/error_code | |
           | ...                                                        | |
           | INTERRUPT_RETURN                                           +-+
           +------------------------------------------------------------+


系统调用分发器的作用是验证系统调用编号，并执行与该系统调用相关的内核函数。

.. slide:: Linux System Call Dispatcher
   :inline-contents: True
   :level: 2

   .. code-block:: c

      /* 处理（handle）int $0x80 */
      __visible void do_int80_syscall_32(struct pt_regs *regs)
      {
	  enter_from_user_mode();
	  local_irq_enable();
	  do_syscall_32_irqs_on(regs);
      }

      /* Linux x86 32 位系统调用分发器的简化版本 */
      static __always_inline void do_syscall_32_irqs_on(struct pt_regs *regs)
      {
	  unsigned int nr = regs->orig_ax;

	  if (nr < IA32_NR_syscalls)
	      regs->ax = ia32_sys_call_table[nr](regs->bx, regs->cx,
	                                         regs->dx, regs->si,
	                                         regs->di, regs->bp);
	  syscall_return_slowpath(regs);
      }



为了向你展示系统调用的流程，我们会用虚拟机来模拟，然后用 gdb 工具来连接正在运行的内核，给 dup2 这个系统调用设置一个断点，再查看它的状态。

.. slide:: Inspecting dup2 system call
   :inline-contents: True
   :level: 2

   |_|

   .. asciicast:: ../res/syscalls-inspection.cast


总结一下，在系统调用过程中发生了以下情况：

.. slide:: System Call Flow Summary
   :inline-contents: True
   :level: 2

   * 应用程序设置系统调用编号和参数，并触发陷阱（trap）指令

   * 执行模式从用户模式切换到内核模式；CPU 切换到内核堆栈；用户堆栈和返回地址保存在内核堆栈中

   * 内核入口点将寄存器保存在内核堆栈中

   * 系统调用分发器识别系统调用函数并运行它

   * 恢复用户空间寄存器并切换回用户空间（例如，调用 IRET 指令）

   * 用户空间应用程序恢复执行


系统调用表
---------

系统调用表是系统调用分发器用于将系统调用编号映射到内核函数的数据结构。

.. slide:: System Call Table
   :inline-contents: True
   :level: 2

   .. code-block:: c

      #define __SYSCALL_I386(nr, sym, qual) [nr] = sym,

      const sys_call_ptr_t ia32_sys_call_table[] = {
        [0 ... __NR_syscall_compat_max] = &sys_ni_syscall,
        #include <asm/syscalls_32.h>
      };

   .. code-block:: c

      __SYSCALL_I386(0, sys_restart_syscall)
      __SYSCALL_I386(1, sys_exit)
      __SYSCALL_I386(2, sys_fork)
      __SYSCALL_I386(3, sys_read)
      __SYSCALL_I386(4, sys_write)
      #ifdef CONFIG_X86_32
      __SYSCALL_I386(5, sys_open)
      #else
      __SYSCALL_I386(5, compat_sys_open)
      #endif
      __SYSCALL_I386(6, sys_close)



系统调用参数处理
---------------

处理系统调用参数是棘手的。由于这些值是由用户空间设置的，内核不能假定其正确性，因此必须始终进行彻底的验证。

指针有一些重要的特殊情况需要进行检查：

.. slide:: 系统调用指针参数
   :inline-contents: True
   :level: 2

   * 绝不允许指向内核空间的指针

   * 检查无效指针


由于系统调用在内核模式下执行，它们可以访问内核空间，如果指针没有正确检查，用户应用程序可能会读取或写入内核空间。

例如，让我们考虑一种情况，即对于读取或写入系统调用没有进行此类检查。如果用户将一个指向内核空间的指针传递给写入系统调用，那么它稍后可以通过读取文件来访问内核数据。如果它将一个指向内核空间的指针传递给读取系统调用，那么它可以破坏内核内存。


.. slide:: 指向内核空间的指针
   :level: 2

   * 如果允许指向内核空间的指针存在，那么用户会通过写入系统调用访问内核数据

   * 如果允许指向内核空间的指针存在，那么用户会通过读取系统调用破坏内核数据


同样，如果应用程序传递的指针无效（例如，指针未映射或在需要进行写操作的情况下使用只读的指针），它可能会导致内核"崩溃"。可以采用两种方法来处理：

.. slide:: 处理无效指针的方法
   :inline-contents: True
   :level: 2

   * 在使用指针之前对照用户地址空间检查指针，或者

   * 避免检查指针，并依赖于内存管理单元（MMU）来检测指针是否无效，并使用页面故障处理程序确定指针是否无效


尽管第二种方法听起来很诱人，但实施起来并不那么容易。页面故障处理程序使用故障地址（被访问的地址）、引发故障的地址（执行访问的指令的地址）和用户地址空间的信息来确定原因：

.. slide:: 页面故障处理
   :inline-contents: True
   :level: 2


      * 写时复制（Copy on Write）、需求分页（demand paging）、交换（swapping）：故障地址和引发故障的地址都在用户空间；故障地址有效（在用户地址空间进行检查）。

      * 在系统调用中使用无效指针：引发故障的地址在内核空间；故障地址在用户空间且无效。

      * 内核错误（内核访问无效指针）：与上述情况相同。

但是，在最后两种情况下，我们没有足够的信息来确定故障的原因。

为了解决这个问题，Linux 使用特殊的 API（例如 c 语言函数 `copy_to_user`）来访问特别设计的用户空间：

.. slide:: 标记访问用户空间的内核代码
   :inline-contents: True
   :level: 2

   * 访问用户空间的确切指令被记录在一个表格中（异常表）

   * 当发生页故障时，会用该表格检查引发故障的地址


尽管故障处理情况可能在地址空间与异常表大小方面更加昂贵，而且更加复杂，但它针对常见情况进行了优化，这就是为什么在 Linux 中它更受欢迎且使用的更多。

.. slide:: 指针检查与故障处理的成本分析
   :inline-contents: True
   :level: 2

   +------------------+-----------------------+------------------------+
   | 成本             |  指针检查             | 故障处理               |
   +==================+=======================+========================+
   | 有效地址         | 地址空间搜索          | 可忽略的               |
   +------------------+-----------------------+------------------------+
   | 无效地址         | 地址空间搜索          | 异常表搜索             |
   +------------------+-----------------------+------------------------+


虚拟动态共享对象 (VDSO)
======================

VDSO（Virtual Dynamic Shared Object，虚拟动态共享对象）机制之所以诞生是为了优化系统调用的实现，以一种不需要 libc 跟踪 CPU 功能与内核版本的方式。

例如，x86 有两种触发系统调用的方式：int 0x80 和 sysenter。后者速度显著更快，因此如果可以的话应使用它。然而，它仅适用于 Pentium II 之后的处理器，并且仅适用于大于 2.6 内核版本的情况。

使用 VDSO 的话，系统调用接口由内核决定：

.. slide:: 虚拟动态共享对象（VDSO）
   :inline-contents: True
   :level: 2

   * 内核在一个特殊的内存区域生成一系列用来触发系统调用的指令（格式化为 ELF 共享对象）

   * 该内存区域映射到用户地址空间的末尾

   * libc 搜索 VDSO，如果存在，则使用它来发出系统调用


.. slide:: 检查 VDSO
   :inline-contents: True
   :level: 2

   |_|

   .. asciicast:: ../res/syscalls-vdso.cast



VDSO 的一个有趣发展产物是虚拟系统调用（vsyscalls），它们直接从用户空间运行。这些 vsyscall 也是 VDSO 的一部分，它们访问 VDSO 页面上的数据，该数据可以是静态的，也可以是由内核在 VDSO 页面的单独读写映射中修改的。作为 vsyscall 可以实现的系统调用的示例包括：getpid 或 gettimeofday。

.. slide:: 虚拟系统调用（vsyscalls）
   :inline-contents: True
   :level: 2

   * 直接从用户空间运行的“系统调用”，属于 VDSO 的一部分

   * 静态数据（例如，getpid（））

   * 内核通过VDSO的读写映射进行动态数据更新（例如，gettimeofday（），time（））


通过系统调用访问用户空间
=======================

正如我们之前提到的，必须使用特殊的 API（如 :c:func:`get_user`、:c:func:`put_user`、:c:func:`copy_from_user` 以及
:c:func:`copy_to_user`）来访问用户空间。这些 API 会检查指针是否位于用户空间，并在指针无效时处理错误。如果指针无效，它们将返回一个非零值。

.. slide:: 通过系统调用访问用户空间
   :inline-contents: True
   :level: 2

   .. code-block:: c

      /* 如果 user_ptr 无效，则返回 -EFAULT */
      if (copy_from_user(&kernel_buffer, user_ptr, size))
          return -EFAULT;

      /* 只有当 user_ptr 有效时才能工作，否则会导致内核崩溃 */
      memcpy(&kernel_buffer, user_ptr, size);


让我们来看一下最简单的 API，以 x86 为例的 `get_user` 实现：

.. slide:: `get_user` 实现
   :inline-contents: True
   :level: 2

   .. code-block:: c

      #define get_user(x, ptr)                                          \
      ({                                                                \
        int __ret_gu;                                                   \
        register __inttype(*(ptr)) __val_gu asm("%"_ASM_DX);            \
        __chk_user_ptr(ptr);                                            \
        might_fault();                                                  \
        asm volatile("call __get_user_%P4"                              \
                     : "=a" (__ret_gu), "=r" (__val_gu),                \
                        ASM_CALL_CONSTRAINT                             \
                     : "0" (ptr), "i" (sizeof(*(ptr))));                \
        (x) = (__force __typeof__(*(ptr))) __val_gu;                    \
        __builtin_expect(__ret_gu, 0);                                  \
      })


该实现使用内联汇编，允许在 C 代码中插入汇编序列，并处理对汇编代码中的变量的访问或者来自汇编代码的变量的访问。

根据变量 x 的类型大小，将调用 __get_user_1、__get_user_2 或 __get_user_4 中的一个函数。此外，在执行汇编调用之前，将把 `ptr` 移动到第一个寄存器 EAX，而在汇编部分完成后，将把 EAX 的值移动到 `__ret_gu`，将 EDX 寄存器的值移动到 `__val_gu`。

以下是表示该过程的伪代码：


.. slide:: get_user 伪代码
   :inline-contents: True
   :level: 2

   .. code-block:: c

      #define get_user(x, ptr)                \
          movl ptr, %eax                      \
	  call __get_user_1                   \
	  movl %edx, x                        \
	  movl %eax, result                   \



__get_user_1 在 x86 上的实现如下所示：

.. slide:: get_user_1 实现
   :inline-contents: True
   :level: 2

   .. code-block:: none

      .text
      ENTRY(__get_user_1)
          mov PER_CPU_VAR(current_task), %_ASM_DX
          cmp TASK_addr_limit(%_ASM_DX),%_ASM_AX
          jae bad_get_user
          ASM_STAC
      1:  movzbl (%_ASM_AX),%edx
          xor %eax,%eax
          ASM_CLAC
          ret
      ENDPROC(__get_user_1)

      bad_get_user:
          xor %edx,%edx
          mov $(-EFAULT),%_ASM_AX
          ASM_CLAC
          ret
      END(bad_get_user)

      _ASM_EXTABLE(1b,bad_get_user)

前两个语句使用当前任务（进程）描述符的 addr_limit 字段与存储在 EDX 中的指针进行比较，以确保我们没有指向内核空间的指针。

然后，禁用 SMAP（Supervisor Mode Access Prevention，监管模式访问防护），以允许内核从用户空间访问，并使用标签 1: 处的指令访问用户空间。然后将 EAX 清零以表示成功，启用 SMAP，之后调用返回。

movzbl 指令是执行对用户空间访问的指令，并且其地址通过标签 1: 捕获并存储在一个特殊的部分中：

.. slide:: 异常表条目
   :inline-contents: True
   :level: 2

   .. code-block:: c

      /* 异常表条目 */
      # define _ASM_EXTABLE_HANDLE(from, to, handler)           \
        .pushsection "__ex_table","a" ;                         \
        .balign 4 ;                                             \
        .long (from) - . ;                                      \
        .long (to) - . ;                                        \
        .long (handler) - . ;                                   \
        .popsection

      # define _ASM_EXTABLE(from, to)                           \
        _ASM_EXTABLE_HANDLE(from, to, ex_handler_default)


对于每个访问用户空间的地址，我们在异常表中都有一个条目，它由以下内容组成：引发故障的地址（from）、在出现错误时跳转到的位置（to）以及处理跳转逻辑的处理函数。所有这些地址都以相对格式存储为相对于异常表地址的 32 位值，因此适用于 32 位和 64 位内核。


所有的异常表条目都由链接器脚本收集在 __ex_table 部分中：

.. slide:: 异常表构建
   :inline-contents: True
   :level: 2

   .. code-block:: c

      #define EXCEPTION_TABLE(align)					\
	. = ALIGN(align);						\
	__ex_table : AT(ADDR(__ex_table) - LOAD_OFFSET) {		\
		VMLINUX_SYMBOL(__start___ex_table) = .;			\
		KEEP(*(__ex_table))					\
		VMLINUX_SYMBOL(__stop___ex_table) = .;			\
	}


该部分由 __start___ex_table 和 __stop___ex_table 符号保护，以便在 C 代码中轻松找到数据。此表由错误处理程序访问：

.. slide:: 异常表处理
   :inline-contents: True
   :level: 2

   .. code-block:: c

      bool ex_handler_default(const struct exception_table_entry *fixup,
                              struct pt_regs *regs, int trapnr)
      {
          regs->ip = ex_fixup_addr(fixup);
          return true;
      }

      int fixup_exception(struct pt_regs *regs, int trapnr)
      {
          const struct exception_table_entry *e;
          ex_handler_t handler;

          e = search_exception_tables(regs->ip);
          if (!e)
              return 0;

          handler = ex_fixup_handler(e);
          return handler(e, regs, trapnr);
      }

它的作用仅仅是将返回地址设置为异常表条目字段中的地址，对于 get_user 异常表条目而言，返回地址是 bad_get_user，它会向调用者返回 -EFAULT。
