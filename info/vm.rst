.. _vm_link:

=========
虚拟机设置
=========

实践工作是在基于 QEMU 的虚拟机上运行的。内核代码在主机上开发和构建，然后部署和运行在虚拟机上。

为了运行和使用虚拟机，以下软件包在 Debian/Ubuntu 系统上是必需的：

* ``flex``
* ``bison``
* ``build-essential``
* ``gcc-multilib``
* ``libncurses5-dev``
* ``qemu-system-x86``
* ``qemu-system-arm``
* ``python3``
* ``minicom``

``kvm`` 软件包不是严格必需的，但是它会通过使用 KVM 支持（使用 QEMU 的 ``-enable-kvm`` 选项）来加快虚拟机的速度。如果缺少 ``kvm``，虚拟机仍然会使用模拟运行（尽管较慢）。

虚拟机设置使用它下载的预构建的 Yocto 镜像和它自己构建的内核镜像。支持以下镜像：

* ``core-image-minimal-qemu``
* ``core-image-minimal-dev-qemu``
* ``core-image-sato-dev-qemu``
* ``core-image-sato-qemu``
* ``core-image-sato-sdk-qemu``

默认情况下，使用 ``core-image-minimal-qemu``。可以通过更新 ``tools/labs/qemu/Makefile`` 中的 ``YOCTO_IMAGE`` 变量来更改这个设置。

启动虚拟机
---------

你可以在 ``tools/labs/`` 文件夹中通过运行 ``make boot`` 来启动虚拟机：

.. code-block:: shell

   .../linux/tools/labs$ make boot

第一次运行 ``make boot`` 命令会编译内核镜像，这会花费更长的时间。后续的运行只会启动 QEMU 虚拟机，并提供详细的输出：

.. code-block:: shell

   .../linux/tools/labs$ make boot
   mkdir /tmp/tmp.7rWv63E9Wf
   sudo mount -t ext4 -o loop core-image-minimal-qemux86.ext4 /tmp/tmp.7rWv63E9Wf
   sudo make -C /home/razvan/school/so2/linux.git modules_install INSTALL_MOD_PATH=/tmp/tmp.7rWv63E9Wf
   make: Entering directory '/home/razvan/school/so2/linux.git'
     INSTALL crypto/crypto_engine.ko
     INSTALL drivers/crypto/virtio/virtio_crypto.ko
     INSTALL drivers/net/netconsole.ko
     DEPMOD  4.19.0+
   make: Leaving directory '/home/razvan/school/so2/linux.git'
   sudo umount /tmp/tmp.7rWv63E9Wf
   rmdir /tmp/tmp.7rWv63E9Wf
   sleep 1 && touch .modinst
   qemu/create_net.sh tap0

   dnsmasq: failed to create listening socket for 172.213.0.1: Address already in use
   qemu/create_net.sh tap1

   dnsmasq: failed to create listening socket for 127.0.0.1: Address already in use
   /home/razvan/school/so2/linux.git/tools/labs/templates/assignments/6-e100/nttcp -v -i &
   nttcp-l: nttcp, version 1.47
   nttcp-l: running in inetd mode on port 5037 - ignoring options beside -v and -p
   bind: Address already in use
   nttcp-l: service-socket: bind:: Address already in use, errno=98
   ARCH=x86 qemu/qemu.sh -kernel /home/razvan/school/so2/linux.git/arch/x86/boot/bzImage -device virtio-serial -chardev pty,id=virtiocon0 -device virtconsole,chardev=virtiocon0 -serial pipe:pipe1 -serial pipe:pipe2 -netdev tap,id=tap0,ifname=tap0,script=no,downscript=no -net nic,netdev=tap0,model=virtio -netdev tap,id=tap1,ifname=tap1,script=no,downscript=no -net nic,netdev=tap1,model=i82559er -drive file=core-image-minimal-qemux86.ext4,if=virtio,format=raw -drive file=disk1.img,if=virtio,format=raw -drive file=disk2.img,if=virtio,format=raw --append "root=/dev/vda loglevel=15 console=hvc0" --display none -s
   qemu-system-i386: -chardev pty,id=virtiocon0: char device redirected to /dev/pts/68 (label virtiocon0)

.. note:: 要显示 QEMU 控制台，请使用

.. code-block:: shell

   .../linux/tools/labs$ QEMU_DISPLAY=gtk make boot

          这将显示 VGA 输出，并且还可以访问标准键盘。

.. note:: 虚拟机设置脚本和配置文件位于 ``tools/labs/qemu/``。

.. _vm_interaction_link:

连接到虚拟机
---------------------------------

一旦虚拟机启动，你可以通过串口连接到它。一个名为 ``serial.pts`` 的链接到模拟的串口设备的符号链接会被创建：

.. code-block:: shell

   .../linux/tools/labs$ ls -l serial.pts
   lrwxrwxrwx 1 razvan razvan 11 Apr  1 08:03 serial.pts -> /dev/pts/68

在主机上，你可以使用 ``minicom`` 命令通过 ``serial.pts`` 链接连接到虚拟机：

.. code-block:: shell

   .../linux/tools/labs$ minicom -D serial.pts
   [...]
   Poky (Yocto Project Reference Distro) 2.3 qemux86 /dev/hvc0

   qemux86 login: root
   root@qemux86:~#

.. note:: 当你连接到虚拟机时，只需在登录提示处输入 ``root``，你就会得到一个 root 控制台，不需要密码。

.. note:: 你可以通过按 ``Ctrl+a`` 然后按 ``x`` 来退出 ``minicom``。你会得到一个确认提示，然后你就会退出 ``minicom``。
