=============
延迟工作
=============

实验目标
==============

* 理解延迟工作（即在稍后时间执行的代码）
* 实现使用延迟工作的常见任务
* 理解延迟工作的同步特性

关键词：softirq、tasklet、struct tasklet_struct、下半部处理程序、jiffies、HZ、timer、struct timer_list、spin_lock_bh、spin_unlock_bh、workqueue、struct work_struct、内核线程、events/x

背景信息
======================

延迟工作是一类内核功能，允许我们安排代码在稍后的时间执行。这些安排的代码可以在进程上下文或中断上下文中运行，具体取决于延迟工作的类型。延迟工作用于补充中断处理程序的功能，因为中断具有重要的要求和限制：

* 中断处理程序的执行时间必须尽可能短
* 在中断上下文中，我们不能使用阻塞调用

使用延迟工作，我们可以在中断处理程序中执行最小所需的工作，并安排一个异步操作在稍后的时间运行，以执行其余的操作。

在中断上下文中运行的延迟工作也称为下半部（bottom-half），因为其目的是执行中断处理程序（top-half）之外所剩余的操作。

定时器（timer）是另一种类型的延迟工作，用于调度在经过一定时间后未来操作的执行。

内核线程本身并不是延迟工作，但可以用来补充延迟工作机制。通常，内核线程用作处理包含阻塞调用的事件的“工作线程（workers）”。

所有类型的延迟工作都使用三种典型的操作：

1. **初始化**。每种类型都由一个结构描述，其字段需要进行初始化。在此时还设置要调度的处理程序。
2. **调度**。尽快安排处理程序的执行（或在超时后）。
3. **屏蔽** 或 **取消**。禁用处理程序的执行。此操作可以是同步的（可保证在取消完成后不再运行处理程序）或异步的。

.. attention:: 在进行延迟工作的清理工作（例如释放与延迟工作相关的结构或从内核中删除模块及其处理程序代码），始终使用同步类型的延迟工作取消。

主要的延迟工作类型包括内核线程和软中断（softirq）。工作队列是在内核线程之上实现的，而 tasklet 和定时器是在软中断之上实现的。下半部（bottom-half）处理程序是 Linux 中最早的延迟工作实现，但后来被软中断所取代。这就是某些函数名称中含有 *bh* 的原因（译注：bh 即 bottom-half 的首字母缩写）。

软中断（Softirqs）
==================

设备驱动程序不能使用软中断，软中断专门为各种内核子系统保留。因此，在编译时定义的软中断的数量是固定的。针对当前内核版本，定义了以下类型：

.. code-block:: c

   enum {
       HI_SOFTIRQ = 0,
       TIMER_SOFTIRQ,
       NET_TX_SOFTIRQ,
       NET_RX_SOFTIRQ,
       BLOCK_SOFTIRQ,
       IRQ_POLL_SOFTIRQ,
       TASKLET_SOFTIRQ,
       SCHED_SOFTIRQ,
       HRTIMER_SOFTIRQ,
       RCU_SOFTIRQ,
       NR_SOFTIRQS
   };


每种类型都有特定的用途：

* *HI_SOFTIRQ* 和 *TASKLET_SOFTIRQ* ——运行任务（tasklet）
* *TIMER_SOFTIRQ* ——运行定时器
* *NET_TX_SOFTIRQ* 和 *NET_RX_SOFTIRQ* ——由网络子系统使用
* *BLOCK_SOFTIRQ* ——由 IO 子系统使用
* *BLOCK_IOPOLL_SOFTIRQ* ——当调用 iopoll 处理程序时，由 IO 子系统使用以提高性能
* *SCHED_SOFTIRQ* ——负载均衡
* *HRTIMER_SOFTIRQ* ——高精度定时器的实现
* *RCU_SOFTIRQ* ——RCU 类型机制的实现 [1]_

.. [1] RCU 是一种机制，用于按照两个步骤执行破坏性操作（例如从链表中删除元素）：（1）移除对已删除数据的引用（2）释放元素的内存。只有在确保没有人再使用该元素后，才执行第二个步骤。此机制的优点是可以无需同步地读取数据。有关更多信息，请参阅 Documentation/RCU/rcu.txt。

*HI_SOFTIRQ* 类型的软中断优先级最高，其次是其他定义的软中断。*RCU_SOFTIRQ* 具有最低优先级。

软中断在中断上下文中运行，这意味着它们不能调用阻塞函数。如果软中断处理程序需要调用此类函数，可以调度工作队列来执行这些阻塞调用。

tasklet
--------

与软中断类似，任务（tasklet）是一种在中断上下文中运行的延迟工作。任务（tasklet）与软中断之间的主要区别在于，任务（tasklet）可以动态分配，并且因此可以被设备驱动程序使用。任务（tasklet）由 :c:type:`struct tasklet` 表示，与许多其他内核结构一样，需要在使用之前进行初始化。预初始化的任务（tasklet）可以以如下方式定义：

.. code-block:: c

   void handler(unsigned long data);

   DECLARE_TASKLET(tasklet, handler, data);
   DECLARE_TASKLET_DISABLED(tasklet, handler, data);


如果我们想手动初始化任务（tasklet），可以使用以下方法：

.. code-block:: c

   void handler(unsigned long data);

   struct tasklet_struct tasklet;

   tasklet_init(&tasklet, handler, data);

当执行任务（tasklet）时，*data* 参数将发送给处理程序。

可以使用调度操作来安排任务（tasklet）的运行。任务（tasklet）是在软中断的基础上执行的。可以使用以下函数进行任务（tasklet）的调度：

.. code-block:: c

   void tasklet_schedule(struct tasklet_struct *tasklet);

   void tasklet_hi_schedule(struct tasklet_struct *tasklet);

使用 *tasklet_schedule* 函数，将调度一个 *TASKLET_SOFTIRQ* 软中断，并运行所有调度的任务（tasklet）。对于 *tasklet_hi_schedule* 函数，将调度一个 *HI_SOFTIRQ* 软中断。

如果一个任务（tasklet）被多次调度，并且在多个调度之间这个任务（tasklet）没有运行，它将只运行一次。任务（tasklet）运行后，可以重新调度它，以便在稍后的时间再次运行。任务（tasklet）可以被其处理程序重新安排。

任务（tasklet）可以被屏蔽，可以使用以下函数：

.. code-block:: c

   void tasklet_enable(struct tasklet_struct *tasklet);
   void tasklet_disable(struct tasklet_struct *tasklet);

请记住，由于任务（tasklet）是在软中断的基础上执行的，因此不能在处理程序函数中使用阻塞调用。

定时器（Timer）
--------------

定时器是一种特殊类型的延迟工作。它们由 :c:type:`struct timer_list` 定义，并在中断上下文中运行，是基于软中断实现的。

要使用定时器，首先必须调用 :c:func:`timer_setup` 函数进行初始化：

.. code-block:: c

   #include <linux/sched.h>

   void timer_setup(struct timer_list *timer,
          void (*function)(struct timer_list *),
          unsigned int flags);

上述函数初始化了结构体的内部字段，并将 *function* 关联为定时器处理程序。由于定时器是通过软中断计划的，因此在与处理函数相关的代码中不能使用阻塞调用。

使用 :c:func:`mod_timer` 函数进行定时器的调度：

.. code-block:: c

   int mod_timer(struct timer_list *timer, unsigned long expires);

其中 *expires* 是要运行处理函数的时间（未来的时间）。该函数可用于调度或重新调度定时器。

时间单位为 *jiffie*。一 jiffie 的绝对值取决于平台，并且可以使用 :c:type:`HZ` 宏找到，该宏定义了 1 秒内的 jiffies 数。要在 jiffies (*jiffies_value*) 和秒 (*seconds_value*) 之间进行转换，使用以下公式：

.. code-block:: c

   jiffies_value = seconds_value * HZ ;
   seconds_value = jiffies_value / HZ ;

内核维护一个计数器，其中包含自上次引导（boot）以来的 jiffies 数，可以通过全局变量或宏 :c:macro:`jiffies` 访问。我们可以使用它来为定时器计算未来的时间：

.. code-block:: c

   #include <linux/jiffies.h>

   unsigned long current_jiffies, next_jiffies;
   unsigned long seconds = 1;

   current_jiffies = jiffies;
   next_jiffies = jiffies + seconds * HZ;

要停止定时器，请使用 :c:func:`del_timer` 和 :c:func:`del_timer_sync` 函数：

.. code-block:: c

   int del_timer(struct timer_list *timer);
   int del_timer_sync(struct timer_list *timer);

这些函数可以用于已调度的定时器和未计划的定时器。:c:func:`del_timer_sync` 用于消除在多处理器系统上可能出现的竞态条件，因为在调用结束时，可以保证定时器处理函数不会在任何处理器上运行。

在使用定时器时，常见的错误是忘记关闭定时器。例如，在移除模块之前，我们必须停止定时器，因为如果定时器在模块被移除后过期，处理函数将不再加载到内核中，从而导致内核出错。

通常用于初始化和调度一秒钟超时的代码是：

.. code-block:: c

   #include <linux/sched.h>

   void timer_function(struct timer_list *);

   struct timer_list timer;
   unsigned long seconds = 1;

   timer_setup(&timer, timer_function, 0);
   mod_timer(&timer, jiffies + seconds * HZ);

停止定时器的方法如下：

.. code-block:: c

   del_timer_sync(&timer);

锁定（Locking）
------------

为了在运行在进程上下文（A）的代码和运行在软中断上下文（B）的代码之间进行同步，我们需要使用特殊的锁原语。我们必须在（A）中使用自旋锁操作，并禁用底半部处理程序，在（B）中只使用基本的自旋锁操作。使用自旋锁可以确保在禁用软中断后，多个 CPU 之间不会发生竞争，而禁用软中断可以确保在已经获取自旋锁的 CPU 上调度软中断时不会发生死锁。

我们可以使用 :c:func:`local_bh_disable` 和 :c:func:`local_bh_enable` 来禁用和启用软中断处理程序（并且由于定时器和任务（tasklet）在软中断之上运行，还包括它们）：

.. code-block:: c

   void local_bh_disable(void);
   void local_bh_enable(void);

允许嵌套调用，当所有的 local_bh_disable() 调用都有相应的 local_bh_enable() 调用时，才会实际重新启用软中断：

.. code-block:: c

   /* 假设软中断已启用 */
   local_bh_disable();  /* 现在禁用了软中断 */
   local_bh_disable();  /* 软中断仍处于禁用状态 */

   local_bh_enable();  /* 软中断仍处于禁用状态 */
   local_bh_enable();  /* 现在启用了软中断 */

.. attention:: 上述调用只会在本地处理器上禁用软中断，通常不安全，必须与自旋锁配合使用。

大多数情况下，设备驱动程序将使用用于同步的特殊版本的自旋锁调用，如 :c:func:`spin_lock_bh` 和 :c:func:`spin_unlock_bh`：

.. code-block:: c

   void spin_lock_bh(spinlock_t *lock);
   void spin_unlock_bh(spinlock_t *lock);


工作队列
======================

工作队列（workqueue）用于在进程上下文中调度要执行的操作。它们所处理的基本单元称为工作项（work）。有两种类型的工作项：

* :c:type:`struct work_struct` ——它安排一个任务在稍后的时间运行
* :c:type:`struct delayed_work` ——它安排一个任务在至少给定的时间间隔之后运行

延迟工作项使用定时器在指定的时间间隔后运行。这种类型的工作项的调用方式与 :c:type:`struct work_struct` 类似，但在函数名称中有 **_delayed**。

在使用工作项之前，必须对其进行初始化。有两种可以使用的宏类型，一种在同时声明和初始化工作项，另一种仅初始化工作项（声明必须单独进行）：

.. code-block:: c

   #include <linux/workqueue.h>

   DECLARE_WORK(name , void (*function)(struct work_struct *));
   DECLARE_DELAYED_WORK(name, void(*function)(struct work_struct *));

   INIT_WORK(struct work_struct *work, void(*function)(struct work_struct *));
   INIT_DELAYED_WORK(struct delayed_work *work, void(*function)(struct work_struct *));

:c:func:`DECLARE_WORK` 和 :c:func:`DECLARE_DELAYED_WORK` 声明并初始化工作项，而 :c:func:`INIT_WORK` 和 :c:func:`INIT_DELAYED_WORK` 则初始化已经声明的工作项。

以下代码声明并初始化工作项：

.. code-block:: c

   #include <linux/workqueue.h>

   void my_work_handler(struct work_struct *work);

   DECLARE_WORK(my_work, my_work_handler);

或者，如果我们想要单独初始化工作项：

.. code-block:: c

   void my_work_handler(struct work_struct * work);

   struct work_struct my_work;

   INIT_WORK(&my_work, my_work_handler);

一旦声明并初始化完成，我们就可以使用 :c:func:`schedule_work` 和 :c:func:`schedule_delayed_work` 来安排任务：

.. code-block:: c

   schedule_work(struct work_struct *work);

   schedule_delayed_work(struct delayed_work *work, unsigned long delay);

:c:func:`schedule_delayed_work` 可以用于计划在给定延迟后执行工作项。延迟时间的单位是 jiffies。

工作项无法被屏蔽，但可以通过调用 :c:func:`cancel_delayed_work_sync` 或 :c:func:`cancel_work_sync` 来取消它们：

.. code-block:: c

   int cancel_work_sync(struct delayed_work *work);
   int cancel_delayed_work_sync(struct delayed_work *work);

这些调用只会停止工作项的后续执行。如果在调用时工作项已经在运行，它将继续运行。无论如何，当这些调用返回时，可以确保该任务不再运行。

.. attention:: 尽管这些函数也有非同步版本（例如 :c:func:`cancel_work`），但在执行清理工作时不要使用它们，否则可能会出现竞态条件。

我们可以通过调用 :c:func:`flush_scheduled_work` 来等待工作队列完成所有工作项的运行：

.. code-block:: c

   void flush_scheduled_work(void);

此函数是阻塞的，因此不能在中断上下文中使用。该函数将等待所有工作项完成。对于延迟工作项，在调用 :c:func:`flush_scheduled_work` 之前必须调用 :c:type:`cancel_delayed_work`。

最后，以下函数可用于在特定处理器上调度工作项 (:c:func:`schedule_delayed_work_on`)，或在所有处理器上调度工作项 (:c:func:`schedule_on_each_cpu`)：

.. code-block:: c

   int schedule_delayed_work_on(int cpu, struct delayed_work *work, unsigned long delay);
   int schedule_on_each_cpu(void(*function)(struct work_struct *));

初始化和调度工作项的常用代码如下：

.. code-block:: c

   void my_work_handler(struct work_struct *work);

   struct work_struct my_work;

   INIT_WORK(&my_work, my_work_handler);

   schedule_work(&my_work);

等待工作项终止的方法如下：

.. code-block:: c

   flush_scheduled_work();

正如你所见，*my_work_handler* 函数接收任务项作为参数。为了能够访问模块的私有数据，可以使用 :c:func:`container_of`：

.. code-block:: c

   struct my_device_data {
       struct work_struct my_work;
       // ...
   };

   void my_work_handler(struct work_struct *work)
   {
      struct my_device_data * my_data;

      my_data = container_of(work, struct my_device_data,  my_work);
      // ...
   }

使用上述函数调度工作项将在内核线程的上下文中运行处理程序，该线程称为 *events/x*，其中 x 是处理器编号。内核将为系统中每个处理器初始化一个内核线程（或工作池）：

.. code-block:: shell

   $ ps -e
   PID TTY TIME CMD
   1?  00:00:00 init
   2 ?  00:00:00 ksoftirqd / 0
   3 ?  00:00:00 events / 0 <--- 运行工作项的内核线程
   4 ?  00:00:00 khelper
   5 ?  00:00:00 kthread
   7?  00:00:00 kblockd / 0
   8?  00:00:00 kacpid

上述函数使用预定义的工作队列（称为 events），它们在 *events/x* 线程的上下文中运行，如上所述。尽管在大多数情况下这已经足够，但它是一个共享资源，在工作项处理程序中出现较长的延迟可能会导致其他队列使用者的延迟。因此，有一些函数用于创建额外的队列。

工作队列由 :c:type:`struct workqueue_struct` 表示。可以使用以下函数创建一个新的工作队列：

.. code-block:: c

   struct workqueue_struct *create_workqueue(const char *name);
   struct workqueue_struct *create_singlethread_workqueue(const char *name);

:c:func:`create_workqueue` 为系统中的每个处理器使用一个线程，而 :c:func:`create_singlethread_workqueue` 则使用单个线程。

要将任务添加到新队列中，请使用 :c:func:`queue_work` 或 :c:func:`queue_delayed_work`：

.. code-block:: c

   int queue_work(struct workqueue_struct *queue, struct work_struct *work);

   int queue_delayed_work(struct workqueue_struct *queue,
                          struct delayed_work *work, unsigned long delay);

:c:func:`queue_delayed_work` 可以用于计划延迟执行的工作项。延迟的时间单位是 jiffies。

要等待所有工作项完成，请调用 :c:func:`flush_workqueue`：

.. code-block:: c

   void flush_workqueue(struct workqueue_struct *queue);

要销毁工作队列，请调用 :c:func:`destroy_workqueue`：

.. code-block:: c

   void destroy_workqueue(struct workqueue_struct *queue);

下面的示例代码声明并初始化一个额外的工作队列，声明并初始化一个工作项，并将其添加到队列中：

.. code-block:: c

   void my_work_handler(struct work_struct *work);

   struct work_struct my_work;
   struct workqueue_struct *my_workqueue;

   my_workqueue = create_singlethread_workqueue("my_workqueue");
   INIT_WORK(&my_work, my_work_handler);

   queue_work(my_workqueue, &my_work);

下面的代码示例显示了如何移除工作队列：

.. code-block:: c

   flush_workqueue(my_workqueue);
   destroy_workqueue(my_workqueue);

使用这些函数计划的工作项将在一个名为 *my_workqueue* 的新内核线程的上下文中运行，该名称是传递给 :c:func:`create_singlethread_workqueue` 函数的参数。

内核线程
==============

内核线程之所以出现，是为了在进程上下文中运行内核代码。内核线程是工作队列机制的基础。实质上，内核线程是一种只在内核态下运行，并且没有用户地址空间或其他用户属性的线程。

要创建内核线程，请使用函数 :c:func:`kthread_create`：

.. code-block:: c

   #include <linux/kthread.h>

   struct task_struct *kthread_create(int (*threadfn)(void *data),
                void *data, const char namefmt[], ...);

* *threadfn* 是将由内核线程运行的函数
* *data* 是要传递给函数的参数
* *namefmt* 表示内核线程的名称，如在 ps/top 中显示的那样；可以包含 %d、%s 等序列，它们将根据标准 printf 语法进行替换。

例如，以下调用：

.. code-block:: c

   kthread_create(f, NULL, "%skthread%d", "my", 0);

将创建一个名为 mykthread0 的内核线程。

使用此函数创建的内核线程将被停止（处于 *TASK_INTERRUPTIBLE* 状态）。要启动内核线程，请调用 :c:func:`wake_up_process`：

.. code-block:: c

   #include <linux/sched.h>

   int wake_up_process(struct task_struct *p);

或者，你可以使用 :c:func:`kthread_run` 来创建并运行内核线程：

.. code-block:: c

   struct task_struct *kthread_run(int (*threadfn)(void *data),
                void *data, const char namefmt[], ...);

尽管在内核线程中运行的函数的编程限制更宽松，并且调度更接近用户空间的调度，但仍然有一些限制需要考虑。下面列出可以或不能从内核线程中执行的操作：

* 不能访问用户地址空间（即使使用 copy_from_user、copy_to_user），因为内核线程没有用户地址空间
* 不能实现长时间运行的忙等待代码；如果内核没有启用抢占选项，那么该代码将在不会被其他内核线程或用户进程抢占的情况下运行，从而占用系统资源
* 可以调用阻塞操作
* 可以使用自旋锁，但如果锁的保持时间很长，建议使用互斥锁（mutex）

内核线程的终止是在内核线程中运行的函数自愿进行的，通过调用 :c:func:`do_exit`：

.. code-block:: c

   fastcall NORET_TYPE void do_exit(long code);

大多数内核线程处理程序的实现都使用相同的模型，建议开始使用相同的模型以避免常见错误：

.. code-block:: c

   #include <linux/kthread.h>

   DECLARE_WAIT_QUEUE_HEAD(wq);

   // 列出内核线程要处理的事件
   struct list_head events_list;
   struct spin_lock events_lock;


   // 描述要处理的事件的结构体
   struct event {
       struct list_head lh;
       bool stop;
       // ...
   };

   struct event* get_next_event(void)
   {
       struct event *e;

       spin_lock(&events_lock);
       e = list_first_entry(&events_list, struct event*, lh);
       if (e)
           list_del(&e->lh);
       spin_unlock(&events_lock);

       return e;
   }

   int my_thread_f(void *data)
   {
       struct event *e;

       while (true) {
           wait_event(wq, (e = get_next_event()));

           /* 处理事件 */

           if (e->stop)
               break;
       }

       do_exit(0);
   }

   /* 启动并运行内核线程 */
   kthread_run(my_thread_f, NULL, "%skthread%d", "my", 0);


使用上述模板，可以使用以下代码触发内核线程请求：

.. code-block:: c

   void send_event(struct event *ev)
   {
       spin_lock(&events_lock);
       list_add(&ev->lh, &events_list);
       spin_unlock(&events_lock);
       wake_up(&wq);
   }

进一步阅读
==============

* `Linux 设备驱动程序（第 3 版），第 7 章：时间、延迟和延迟工作 <http://lwn.net/images/pdf/LDD3/ch07.pdf>`_
* `调度任务 <http://tldp.org/LDP/lkmpg/2.6/html/x1211.html>`_
* `驱动程序移植：工作队列接口 <http://lwn.net/Articles/23634/>`_
* `工作队列重新工作 <http://lwn.net/Articles/211279/>`_
* `简化内核线程 <http://lwn.net/Articles/65178/>`_
* `不可靠的锁指南 <http://www.kernel.org/pub/linux/kernel/people/rusty/kernel-locking/index.html>`_

练习
========

.. include:: ../labs/exercises-summary.hrst
.. |LAB_NAME| replace:: 延迟工作

0. 简介
--------

使用 |LXR|_，找到以下符号的定义：

* :c:macro:`jiffies`
* :c:type:`struct timer_list`
* :c:func:`spin_lock_bh function`


1. 定时器
----------

我们将创建一个简单的内核模块，在模块的内核加载后的第 *TIMER_TIMEOUT* 秒显示一条消息。

生成名为 **1-2-timer** 的任务骨架，并按照标有 **TODO 1** 的部分来完成任务。

.. hint:: 使用 `pr_info(...)`。消息将显示在控制台上，并且还可以使用 dmesg 查看。在调度定时器时，我们需要使用系统的（未来）绝对时间并且以滴答数表示。系统的当前时间（以滴答数表示）由 :c:type:`jiffies` 给出。因此，我们需要将 ``jiffies + TIMER_TIMEOUT * HZ`` 作为绝对时间传递给定时器。

有关更多信息，请查阅 `定时器（Timer）`_ 部分。


2. 周期性定时器
-----------------

修改前面的模块，使消息每隔 TIMER_TIMEOUT 秒显示一次。按照骨架中标有 **TODO 2** 的部分进行修改。

3. 使用 ioctl 控制定时器
----------------------------

我们计划在从用户空间接收到 ioctl 调用后的第 N 秒显示有关当前进程的信息。N 作为 ioctl 参数传递。

生成名为 **3-4-5-deferred** 的任务骨架，并按照骨架中标有 **TODO 1** 的部分进行修改。

你需要实现以下 ioctl 操作。

* MY_IOCTL_TIMER_SET：安排定时器在接收到的秒数之后运行，该秒数作为 ioctl 的参数。该定时器并不周期运行。
  * 此命令直接接收一个值，而不是指针。

* MY_IOCTL_TIMER_CANCEL：停用定时器。

.. note:: 请查阅 :ref:`ioctl` 了解如何访问 ioctl 参数。

.. note:: 请查阅 `定时器（Timer）`_ 部分，了解如何启用/禁用定时器。在定时器处理程序中，显示当前进程标识符（PID）和进程执行镜像名称。

.. hint:: 你可以使用当前进程的 *pid* 和 *comm* 字段来查找当前进程标识符。有关详细信息，请查阅 :ref:`proc-info`。

.. hint:: 要从用户空间使用设备驱动程序，你必须使用 mknod 程序创建设备字符文件 */dev/deferred*。或者，你可以运行 *3-4-5-deferred/kernel/makenode* 脚本来执行此操作。

通过调用用户空间的 ioctl 操作来启用和禁用定时器。使用 *3-4-5-deferred/user/test* 程序来测试定时器的计划和取消。该程序在命令行上接收 ioctl 类型操作及其参数（如果有）。

.. hint:: 运行测试可执行文件时不带参数，以观察它接受的命令行选项。

	  要在 3 秒后启用定时器，请使用：

	  .. code-block:: c

	     ./test s 3

	  要停用定时器，请使用：

	  .. code-block:: c

	     ./test c


注意，定时器运行所基于的当前进程每次都是 PID 为 0 的 *swapper/0*。这个进程是空闲进程，当没有其他任务可运行时，它会一直运行。由于虚拟机非常轻量级且没有太多操作，大部分时间都会看到这个进程。

4. 阻塞操作
------------------

接下来，我们将尝试在定时器例程中执行阻塞操作，以查看会发生什么情况。为此，我们尝试在定时器处理例程中调用一个名为 alloc_io() 的模拟阻塞操作的函数。

修改模块，使得当接收到 *MY_IOCTL_TIMER_ALLOC* 命令时，定时器处理程序将调用 :c:func:`alloc_io`。按照骨架中标有 **TODO 2** 的部分进行修改。

使用相同的定时器。为了区分定时器处理程序中的功能，可以在设备结构中使用一个标志。使用代码骨架中定义的 *TIMER_TYPE_ALLOC* 和 *TIMER_TYPE_SET* 宏。对于初始化，请使用 TIMER_TYPE_NONE。

运行测试程序以验证任务 3 的功能。再次运行测试程序以调用 :c:func:`alloc_io()`。

.. note:: 该驱动程序会导致错误，因为在原子上下文（定时器处理程序运行在中断上下文中）中调用了阻塞函数。

5. 工作队列
-------------

我们将修改模块，以解决上一个任务中观察到的错误。

为此，让我们使用工作队列调用 :c:func:`alloc_io`。从定时器处理程序中安排一个工作项。在工作项处理程序中（在进程上下文中运行），调用 :c:func:`alloc_io`。按照骨架中标有 **TODO 3** 的部分进行修改，并在需要时查阅 `工作队列`_ 部分。

.. hint:: 在设备结构中添加一个类型为 :c:type:`struct work_struct` 的新字段。初始化此字段。使用 :c:func:`schedule_work` 从定时器处理程序中调度工作项。从 ioctl 后的 N 秒开始调度定时器处理程序。

6. 内核线程
----------------

实现一个简单的模块，创建一个显示当前进程标识符的内核线程。

生成名为 **6-kthread** 的任务骨架，并按照骨架中标有 **TODO** 的部分进行修改。


.. note:: 创建和运行线程有两种选择：

	  * 使用 :c:func:`kthread_run` 创建并运行线程

	  * 使用 :c:func:`kthread_create` 创建一个挂起的线程，然后使用 :c:func:`wake_up_process` 启动它。

	  如果需要，请查阅 `内核线程`_ 部分。

.. attention:: 将线程终止与模块卸载进行同步：

	       * 线程应在模块卸载时结束

	       * 在卸载之前，请等待内核线程退出


.. hint:: 为了同步，使用两个等待队列和两个标志。

	  请查阅 :ref:`waiting-queues` 了解如何使用等待队列。

	  使用原子变量作为标志。请查阅 :ref:`atomic-variables`。


7. 定时器和进程之间共享的缓冲区
------------------------------------------

该任务的目的是在延迟操作（定时器）和进程上下文之间进行同步。设置一个周期性定时器，监视进程列表。如果其中一个进程终止，将打印一条消息。可以动态添加进程到列表中。请使用 *3-4-5-deferred/kernel/* 骨架作为基础，并按照标有 **TODO 4** 的部分完成任务。

当接收到 *MY_IOCTL_TIMER_MON* 命令时，检查给定的进程是否存在，如果存在，则将其添加到监视的进程列表中，并在设置了类型后启用定时器。

.. hint:: 使用 :c:func:`get_proc` 检查 pid，找到关联的 :c:type:`struct task_struct`，并分配一个 :c:type:`struct mon_proc` 项目，可以将其添加到列表中。请注意，该函数还会增加任务的引用计数，以便在任务终止时不会释放其内存。

.. attention:: 使用自旋锁保护对列表的访问。请注意，由于我们与定时器处理程序共享数据，因此除了获取锁之外，还需要禁用底半部处理程序。请查阅 `锁定`_ 部分。

.. hint:: 每秒钟从定时器中收集信息。使用现有的定时器，并通过 TIMER_TYPE_ACCT 添加新的行为。要设置标志，请使用测试程序的 *t* 参数。


在定时器处理程序中，遍历监视的进程列表，并检查它们是否已终止。如果是，则打印进程名称和 PID，然后从列表中删除该进程，递减任务使用计数器，以便可以释放其内存，最后释放 :c:type:`struct mon_proc` 结构。

.. hint:: 使用 :c:func:`struct task_struct` 的 *state* 字段。如果任务的状态为 *TASK_DEAD*，则表示任务已终止。

.. hint:: 使用 :c:func:`put_task_struct` 递减任务使用计数器。

.. attention:: 确保使用自旋锁保护列表访问。简单的变体就足够了。

.. attention:: 确保使用安全迭代器遍历列表，因为我们可能需要从列表中删除项目。

在检查完列表后，重新启用定时器。
