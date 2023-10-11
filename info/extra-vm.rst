===============
自定义虚拟机配置
===============

通过 SSH 连接到虚拟机
--------------------
QEMU 虚拟机的默认 Yocto 镜像（core-image-minimal-qemu）仅提供了运行内核和内核模块所需要的最小功能。要使用额外的功能，例如 SSH 连接，需要一个更完整的镜像，例如 core-image-sato-dev-qemu。

要使用新的镜像，需要更改 ``tools/labs/qemu/Makefile`` 中的 ``YOCTO_IMAGE`` 变量：

… code-block:: shell

   YOCTO_IMAGE = core-image-sato-qemu$(ARCH).ext4

当你第一次使用新的镜像配置通过 ``make boot`` 命令启动虚拟机时，它会下载相应的镜像文件，然后启动虚拟机。这个镜像比最小的镜像要大（大约400MB），所以下载需要一些时间。

然后你可以通过 ``minicom`` 进入虚拟机，确定 eth0 接口的 IP 地址，之后你就可以通过 SSH 连接到虚拟机了：

.. code-block:: shell

   $ minicom -D serial.pts
   Poky (Yocto Project Reference Distro) 2.3 qemux86 /dev/hvc0

   qemux86 login: root
   root@qemux86:~# ip a s
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
       inet 127.0.0.1/8 scope host lo
          valid_lft forever preferred_lft forever
       inet6 ::1/128 scope host 
          valid_lft forever preferred_lft forever
   2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
       link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
       inet 172.213.0.18/24 brd 172.213.0.255 scope global eth0
          valid_lft forever preferred_lft forever
       inet6 fe80::5054:ff:fe12:3456/64 scope link 
          valid_lft forever preferred_lft forever
   3: sit0@NONE: <NOARP> mtu 1480 qdisc noop qlen 1000
       link/sit 0.0.0.0 brd 0.0.0.0

   $ ssh -l root 172.213.0.18
   The authenticity of host '172.213.0.18 (172.213.0.18)' can't be established.
   RSA key fingerprint is SHA256:JUWUcD7LdvURNcamoPePMhqEjFFtUNLAqO+TtzUiv5k.
   Are you sure you want to continue connecting (yes/no)? yes
   Warning: Permanently added '172.213.0.18' (RSA) to the list of known hosts.
   root@qemux86:~# uname -a
   Linux qemux86 4.19.0+ #3 SMP Sat Apr 4 22:45:18 EEST 2020 i686 GNU/Linux

将调试器（debugger）连接到虚拟机内核
----------------------------------

.. code-block:: shell

   .../linux/tools/labs$ make gdb
   ln -fs /home/tavi/src/linux/vmlinux vmlinux
   gdb -ex "target remote localhost:1234" vmlinux
   GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.04) 7.11.1
   Copyright (C) 2016 Free Software Foundation, Inc.
   License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
   This is free software: you are free to change and redistribute it.
   There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
   and "show warranty" for details.
   This GDB was configured as "x86_64-linux-gnu".
   Type "show configuration" for configuration details.
   For bug reporting instructions, please see:
   <http://www.gnu.org/software/gdb/bugs/>.
   Find the GDB manual and other documentation resources online at:
   <http://www.gnu.org/software/gdb/documentation/>.
   For help, type "help".
   Type "apropos word" to search for commands related to "word"...
   Reading symbols from vmlinux...done.
   Remote debugging using localhost:1234
   0xc13cf2f2 in native_safe_halt () at ./arch/x86/include/asm/irqflags.h:53
   53asm volatile("sti; hlt": : :"memory");
   (gdb) bt
   #0  0xc13cf2f2 in native_safe_halt () at ./arch/x86/include/asm/irqflags.h:53
   #1  arch_safe_halt () at ./arch/x86/include/asm/irqflags.h:95
   #2  default_idle () at arch/x86/kernel/process.c:341
   #3  0xc101f136 in arch_cpu_idle () at arch/x86/kernel/process.c:332
   #4  0xc106a6dd in cpuidle_idle_call () at kernel/sched/idle.c:156
   #5  do_idle () at kernel/sched/idle.c:245
   #6  0xc106a8c5 in cpu_startup_entry (state=<optimized out>)
   at kernel/sched/idle.c:350
   #7  0xc13cb14a in rest_init () at init/main.c:415
   #8  0xc1507a7a in start_kernel () at init/main.c:679
   #9  0xc10001da in startup_32_smp () at arch/x86/kernel/head_32.S:368
   #10 0x00000000 in ?? ()
   (gdb)

重建内核镜像
------------

内核镜像是在虚拟机第一次启动时构建的。要重建内核，删除由在 ``tools/labs/qemu/Makefile`` 中的 ``ZIMAGE`` 变量定义的内核镜像文件：

.. code-block:: shell

   ZIMAGE = $(KDIR)/arch/$(ARCH)/boot/$(b)zImage

通常，内核的完整路径是 ``arch/x86/boot/bzImage``。

删除后，可以使用以下命令重新构建内核镜像：

.. code-block:: shell

   ~/src/linux/tools/labs$ make zImage

或者简单地启动虚拟机

.. code-block:: shell

   ~/src/linux/tools/labs$ make boot

使用 Docker 容器
----------------

如果你的设备不允许安装实验室配置所需的软件包，你可以构建和运行一个容器，它已经为虚拟机环境准备好了所有的配置。

为了运行容器化的配置，你需要安装以下软件包：

* ``docker``
* ``docker-compose``

为了运行容器基础设施，在 ``tools/labs/`` 目录下运行以下命令：

.. code-block:: shell

    sergiu@local:~/src/linux/tools/labs$ make docker-kernel
    ...
    ubuntu@so2:~$

你第一次运行上面的命令时，会花很长时间，因为你需要构建容器环境并安装所需的应用程序。

每次你运行 ``make docker-kernel`` 命令时，另一个 shell 会连接到容器。这将允许你在多个窗口中工作。

你在常规环境中使用的所有命令都可以在容器化环境中使用。

linux 仓库被挂载在 ``/linux`` 目录下。你在这里做的所有更改也会在你的本地实例上看到。

要停止容器运行，可以使用以下命令：

.. code-block:: shell

    make stop-docker-kernel
