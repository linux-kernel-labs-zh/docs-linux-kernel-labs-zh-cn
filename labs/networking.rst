============================
网络
============================

.. meta::
   :description: 理解 Linux 内核网络架构，掌握使用数据包（packet）过滤器或防火墙进行 IP 数据包管理，熟悉在 Linux 内核级别使用套接字的方法

实验目标
==============

  * 理解 Linux 内核网络架构
  * 掌握使用数据包（packet）过滤器或防火墙进行 IP 数据包管理
  * 熟悉在 Linux 内核级别使用套接字的方法

概述
========

互联网的发展导致网络应用程序呈指数增长，因此操作系统的网络子系统对速度和生产力的要求也在不断增加。网络子系统并非操作系统内核的必需组件（Linux 内核可以选择在编译时不包含网络支持）。然而，由于对连接的需要，计算机系统（甚至嵌入式设备）很少会使用不支持网络的操作系统。现代操作系统使用 `TCP/IP 协议栈 <https://zh.wikipedia.org/zh-cn/TCP/IP协议族>`_。它们的内核实现了传输层以下的协议，而应用层协议通常在用户空间实现（如 HTTP、FTP 以及 SSH 等）。

用户空间中的网络编程
------------------

套接字（socket）是在用户空间中，对于网络通信的抽象。套接字抽象了通信通道，也是基于内核的 TCP/IP 栈交互接口。IP 套接字与 IP 地址、所使用的传输层协议（如 TCP、UDP 等）和端口相关联。常用的使用套接字的函数调用有：创建 (``socket``)、初始化 (``bind``)、连接 (``connect``)、等待连接 (``listen``, ``accept``) 以及关闭套接字 (``close``)。

我们通过 ``read``/``write`` 或 ``recv``/``send`` 调用实现 TCP 套接字的网络通信，通过 ``recvfrom``/``sendto`` 调用实现 UDP 套接字的网络通信。传输和接收操作对应用程序来说是透明的，封装和网络传输由内核自行决定。然而，也可以使用原始套接字（创建套接字时使用 ``PF_PACKET`` 选项）在用户空间中实现 TCP/IP 栈，或者在内核中实现应用层协议（例如 `TUX web 服务器 <https://zh.wikipedia.org/zh-cn/TUX_Web服务器>`_）。

有关使用套接字进行用户空间编程的更多详细信息，请参阅 `Beej 的网络套接字编程指南 <https://www.beej.us/guide/bgnet/>`_。

Linux 网络编程
================

Linux 内核提供了三种基本的用于处理网络数据包的结构：:c:type:`struct socket`、:c:type:`struct sock` 和 :c:type:`struct sk_buff`。

前两者是对套接字的抽象：

  * :c:type:`struct socket` 是非常接近用户空间的抽象，即用于编写网络应用程序的 `BSD 套接字 <http://zh.wikipedia.org/zh-cn/Berkeley套接字>`_；
  * :c:type:`struct sock` 或 Linux 术语中的 *INET 套接字* 是套接字的网络表示。

这两个结构有关联: :c:type:`struct socket` 包含 INET 套接字字段，而每个 :c:type:`struct sock` 都有一个 BSD 套接字持有它。

:c:type:`struct sk_buff` 结构是网络数据包及其状态的表示。当从用户空间或网络接口接收到内核数据包时，该结构被创建。

:c:type:`struct socket` 结构
-------------------------------------

:c:type:`struct socket` 结构是 BSD 套接字的内核表示，可以执行在它上面执行的操作类似于内核提供的操作（通过系统调用）。使用套接字的常见操作（创建、初始化/绑定、关闭等）会导致特定的系统调用；它们与 :c:type:`struct socket` 结构一起工作。

:c:type:`struct socket` 的操作在 :file:`net/socket.c` 中进行描述，且与协议类型无关。因此, :c:type:`struct socket` 结构是特定网络操作实现的通用接口。通常，这些操作的名称以 ``sock_`` 前缀开头。

.. _SocketStructOps:

对 socket 结构的操作
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

socket 相关操作包括：

创建
""""""""

创建类似于在用户空间调用 :c:func:`socket` 函数，但创建的 :c:type:`struct socket` 将存储在 ``res`` 参数中：

  * ``int sock_create(int family, int type, int protocol, struct socket **res)``：在 :c:func:`socket` 系统调用之后创建 socket；
  * ``int sock_create_kern(struct net *net, int family, int type, int protocol, struct socket **res)``：创建内核 socket；
  * ``int sock_create_lite(int family, int type, int protocol, struct socket **res)``：创建内核 socket, 不经过参数完整性检查。

这些调用的参数如下：

  * ``net`` (如果存在)用作对所使用的网络命名空间的引用；通常我们会使用 ``init_net`` 进行初始化；
  * ``family`` 表示在信息传输中使用的协议族；它们通常以 ``PF_``(协议族)字符串开头；表示所使用的协议族的常量可以在 :file:`linux/socket.h` 中找到，其中最常用的是 ``PF_INET``，用于 TCP/IP 协议；
  * ``type`` 是 socket 的类型；用于此参数的常量可以在 :file:`linux/net.h` 中找到，其中最常用的是 ``SOCK_STREAM`` (用于基于连接的源到目的地通信)以及 ``SOCK_DGRAM`` (用于无连接通信)；
  * ``protocol`` 表示使用的协议，与 ``type`` 参数密切相关；用于此参数的常量可以在 :file:`linux/in.h` 中找到，其中最常用的是 ``IPPROTO_TCP`` (用于 TCP)， ``IPPROTO_UDP`` (用于 UDP)。

要在内核空间中创建 TCP socket，你需要调用：

.. code-block:: c

  	struct socket *sock;
  	int err;

  	err = sock_create_kern(&init_net, PF_INET, SOCK_STREAM, IPPROTO_TCP, &sock);
  	if (err < 0) {
  		/* 处理错误 */
  	}

要在内核空间中创建 UDP socket，你需要调用：

.. code-block:: c

  	struct socket *sock;
  	int err;

  	err = sock_create_kern(&init_net, PF_INET, SOCK_DGRAM, IPPROTO_UDP, &sock);
  	if (err < 0) {
  		/* 处理错误 */
  	}

一个使用示例是 :c:func:`sys_socket` 系统调用处理程序的一部分：

.. code-block:: c

  SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
  {
  	int retval;
  	struct socket *sock;
  	int flags;

  	/* 检查 SOCK_* 常量是否一致。 */
  	BUILD_BUG_ON(SOCK_CLOEXEC != O_CLOEXEC);
  	BUILD_BUG_ON((SOCK_MAX | SOCK_TYPE_MASK) != SOCK_TYPE_MASK);
  	BUILD_BUG_ON(SOCK_CLOEXEC & SOCK_TYPE_MASK);
  	BUILD_BUG_ON(SOCK_NONBLOCK & SOCK_TYPE_MASK);

  	flags = type & ~SOCK_TYPE_MASK;
  	if (flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK))
  		return -EINVAL;
  	type &= SOCK_TYPE_MASK;

  	if (SOCK_NONBLOCK != O_NONBLOCK && (flags & SOCK_NONBLOCK))
  		flags = (flags & ~SOCK_NONBLOCK) | O_NONBLOCK;

  	retval = sock_create(family, type, protocol, &sock);
  	if (retval < 0)
  		goto out;

  	return sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
  }

关闭连接
"""""""

关闭连接（对于使用连接的 socket）并释放相关资源：

  * ``void sock_release(struct socket *sock)`` 调用 socket 结构 ``ops`` 字段中的 ``release`` 函数：

.. code-block:: c

  void sock_release(struct socket *sock)
  {
  	if (sock->ops) {
  		struct module *owner = sock->ops->owner;

  		sock->ops->release(sock);
  		sock->ops = NULL;
  		module_put(owner);
  	}
  	//...
  }

发送/接收消息
""""""""""""""""""""""""""

使用以下函数来发送/接收消息：

  * ``int sock_recvmsg(struct socket *sock, struct msghdr *msg, int flags);``
  * ``int kernel_recvmsg(struct socket *sock, struct msghdr *msg, struct kvec *vec, size_t num, size_t size, int flags);``
  * ``int sock_sendmsg(struct socket *sock, struct msghdr *msg);``
  * ``int kernel_sendmsg(struct socket *sock, struct msghdr *msg, struct kvec *vec, size_t num, size_t size);``

消息的发送/接收函数将调用 socket ``ops`` 字段中的 ``sendmsg``/``recvmsg`` 函数。当 socket 在内核中使用时，应使用以 ``kernel_`` 为前缀的函数。

参数包括：

  * ``msg``, :c:type:`struct msghdr` 结构，包含要发送/接收的消息。该结构的重要组成部分包括 ``msg_name`` 和 ``msg_namelen``，对于 UDP 套接字，必须使用目标地址填充 (:c:type:`struct sockaddr_in`)；
  * ``vec``, :c:type:`struct kvec` 结构，其中有一个指针指向缓冲区，缓冲区内包含该 :c:type:`struct kvec` 结构的数据和大小；正如所见，它的结构类似于 :c:type:`struct iovec` 结构 (:c:type:`struct iovec` 结构对应用户空间数据，而 :c:type:`struct kvec` 结构对应内核空间数据)。

可以在 :c:func:`sys_sendto` 系统调用处理程序中看到用法示例：

.. code-block:: c

  SYSCALL_DEFINE6(sendto, int, fd, void __user *, buff, size_t, len,
  		unsigned int, flags, struct sockaddr __user *, addr,
  		int, addr_len)
  {
  	struct socket *sock;
  	struct sockaddr_storage address;
  	int err;
  	struct msghdr msg;
  	struct iovec iov;
  	int fput_needed;

  	err = import_single_range(WRITE, buff, len, &iov, &msg.msg_iter);
  	if (unlikely(err))
  		return err;
  	sock = sockfd_lookup_light(fd, &err, &fput_needed);
  	if (!sock)
  		goto out;

  	msg.msg_name = NULL;
  	msg.msg_control = NULL;
  	msg.msg_controllen = 0;
  	msg.msg_namelen = 0;
  	if (addr) {
  		err = move_addr_to_kernel(addr, addr_len, &address);
  		if (err < 0)
  			goto out_put;
  		msg.msg_name = (struct sockaddr *)&address;
  		msg.msg_namelen = addr_len;
  	}
  	if (sock->file->f_flags & O_NONBLOCK)
  		flags |= MSG_DONTWAIT;
  	msg.msg_flags = flags;
  	err = sock_sendmsg(sock, &msg);

  out_put:
  	fput_light(sock->file, fput_needed);
  out:
  	return err;
  }

:c:type:`struct socket` 字段
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: c

  /**
   *  struct socket——通用的 BSD socket 
   *  @state: socket 状态（%SS_CONNECTED 等）
   *  @type: socket 类型（%SOCK_STREAM 等）
   *  @flags: socket 标志（%SOCK_NOSPACE 等）
   *  @ops: 协议特定的 socket 操作
   *  @file: 反向指向 file 的指针，用于垃圾回收
   *  @sk: 内部的网络协议无关 socket 表示
   *  @wq: 有多种用途的等待队列
   */
  struct socket {
  	socket_state		state;

  	short			type;

  	unsigned long		flags;

  	struct socket_wq __rcu	*wq;

  	struct file		*file;
  	struct sock		*sk;
  	const struct proto_ops	*ops;
  };

值得注意的字段包括：

  * ``ops``——该结构内部有指针，指针指向协议特定函数；
  * ``sk``——与之关联的 ``INET socket``。

:c:type:`struct proto_ops` 结构
""""""""""""""""""""""""""""""""""""""""

:c:type:`struct proto_ops` 结构体包含了特定操作（TCP、UDP 等）的实现；这些函数将通过通用函数(如 :c:func:`sock_release`, :c:func:`sock_sendmsg` 等)使用 :c:type:`struct socket` 为参数来调用。

因此, :c:type:`struct proto_ops` 结构体包含了一些特定协议实现的函数指针：

.. code-block:: c

  struct proto_ops {
  	int		family;
  	struct module	*owner;
  	int		(*release)   (struct socket *sock);
  	int		(*bind)      (struct socket *sock,
  				      struct sockaddr *myaddr,
  				      int sockaddr_len);
  	int		(*connect)   (struct socket *sock,
  				      struct sockaddr *vaddr,
  				      int sockaddr_len, int flags);
  	int		(*socketpair)(struct socket *sock1,
  				      struct socket *sock2);
  	int		(*accept)    (struct socket *sock,
  				      struct socket *newsock, int flags, bool kern);
  	int		(*getname)   (struct socket *sock,
  				      struct sockaddr *addr,
  				      int peer);
  	//...
  }

:c:type:`struct socket` 结构体的 ``ops`` 字段的初始化是在 :c:func:`__sock_create` 函数中完成的，该函数通过调用针对每个协议的 :c:func:`create` 函数来实现；等效调用是 :c:func:`__sock_create` 函数的实现：

.. code-block:: c

  //...
  	err = pf->create(net, sock, protocol, kern);
  	if (err < 0)
  		goto out_module_put;
  //...

这将使用与 socket 关联的协议类型特定的调用来实例化函数指针。 :c:func:`sock_register` 和 :c:func:`sock_unregister` 调用用于填充 ``net_families`` 向量。

对于 socket 结构的其余操作（除了在 `对 socket 结构的操作`_ 部分中描述的创建、关闭和发送/接收消息之外），将调用通过这个结构中的指针传递的函数。例如，对于 ``bind`` 操作（它将一个 socket 与一个本地机器上的 socket 关联）有以下代码：

.. code-block:: c

  #define MY_PORT 60000

  struct sockaddr_in addr = {
  	.sin_family = AF_INET,
  	.sin_port = htons (MY_PORT),
  	.sin_addr = { htonl (INADDR_LOOPBACK) }
  };

  //...
  	err = sock->ops->bind (sock, (struct sockaddr *) &addr, sizeof(addr));
  	if (err < 0) {
  		  /* 处理错误 */
  	}
  //...

在以上代码中，用于传输与 socket 关联的地址和端口信息的是 :c:type:`struct sockaddr_in` 结构体。

:c:type:`struct sock` 结构
-----------------------------

:c:type:`struct sock` 结构描述了 ``INET`` 套接字。这样的结构与用户空间的 socket 相关联，并且与 :c:type:`struct socket` 结构相关联，其中与 :c:type:`struct socket` 结构的关联是隐式的。该结构用于存储关于连接状态的信息。结构体的字段和相关操作通常以 ``sk_`` 字符串开头。以下列出了一些字段：

.. code-block:: c

  struct sock {
  	//...
  	unsigned int		sk_padding : 1,
  				sk_no_check_tx : 1,
  				sk_no_check_rx : 1,
  				sk_userlocks : 4,
  				sk_protocol  : 8,
  				sk_type      : 16;
  	//...
  	struct socket		*sk_socket;
  	//...
  	struct sk_buff		*sk_send_head;
  	//...
  	void			(*sk_state_change)(struct sock *sk);
  	void			(*sk_data_ready)(struct sock *sk);
  	void			(*sk_write_space)(struct sock *sk);
  	void			(*sk_error_report)(struct sock *sk);
  	int			(*sk_backlog_rcv)(struct sock *sk,
  						  struct sk_buff *skb);
  	void                    (*sk_destruct)(struct sock *sk);
  };

\

  * ``sk_protocol`` 是套接字使用的协议类型；
  * ``sk_type`` 是套接字类型 (``SOCK_STREAM``, ``SOCK_DGRAM`` 等)；
  * ``sk_socket`` 是持有该套接字的 BSD 套接字；
  * ``sk_send_head`` 是用于传输的 :c:type:`struct sk_buff` 结构列表；
  * 最后的函数指针是用于不同情况的回调函数。

使用从 ``net_families`` 创建的回调函数(称为 :c:func:`__sock_create`) 来初始化 :c:type:`struct sock` 并将其附加到 BSD 套接字。以下是在 :c:func:`inet_create` 函数中初始化 IP 协议的 :c:type:`struct sock` 结构体的方法：

.. code-block:: c

  /*
   *   创建 inet 套接字。
   */

  static int inet_create(struct net *net, struct socket *sock, int protocol,
  		       int kern)
  {

  	struct sock *sk;

  	//...
  	err = -ENOBUFS;
  	sk = sk_alloc(net, PF_INET, GFP_KERNEL, answer_prot, kern);
  	if (!sk)
  		goto out;

  	err = 0;
  	if (INET_PROTOSW_REUSE & answer_flags)
  		sk->sk_reuse = SK_CAN_REUSE;


  	//...
  	sock_init_data(sock, sk);

  	sk->sk_destruct	   = inet_sock_destruct;
  	sk->sk_protocol	   = protocol;
  	sk->sk_backlog_rcv = sk->sk_prot->backlog_rcv;
  	//...
  }

.. _StructSKBuff:

:c:type:`struct sk_buff` 结构体
--------------------------------

:c:type:`struct sk_buff` (套接字缓冲区)描述了一个网络数据包。该结构的字段包含有关报头和数据包内容、使用的协议、使用的网络设备以及指向其他 :c:type:`struct sk_buff` 的指针的信息。下面是该结构体的内容的概要描述：

.. code-block:: c

  struct sk_buff {
    union {
      struct {
        /* 这两个成员必须放在最前面。 */
        struct sk_buff *next;
        struct sk_buff *prev;

        union {
          struct net_device *dev;
          /* 一些协议可能会使用此空间来存储信息，此种情形下设备指针为 NULL。
           * UDP 接收路径就是其中之一。
           */
          unsigned long dev_scratch;
        };
      };

      struct rb_node rbnode; /* 在 netem 和 tcp 栈中使用 */
    };
    struct sock *sk;

    union {
      ktime_t tstamp;
      u64 skb_mstamp;
    };

    /*
     * 这是控制缓冲区。每一层都可以自由使用它。
     * 请将你的私有变量放在这里。如果要在多个层之间保留这些变量，首先必须进行 skb_clone()。
     * 此缓冲区由当前此 skb 的队列的所有者拥有。
     */
    char cb[48] __aligned(8);

    unsigned long _skb_refdst;
    void (*destructor)(struct sk_buff *skb);
    union {
      struct {
        unsigned long _skb_refdst;
        void (*destructor)(struct sk_buff *skb);
      };
      struct list_head tcp_tsorted_anchor;
    };
    /* ... */

    unsigned int len,
                 data_len;
    __u16 mac_len,
          hdr_len;

    /* ... */

    __be16 protocol;
    __u16 transport_header;
    __u16 network_header;
    __u16 mac_header;

    /* 私有：*/
    __u32 headers_end[0];
    /* 公有：*/

    /* 这些元素必须放在最后。有关详细信息，请参阅 alloc_skb()。*/
    sk_buff_data_t tail;
    sk_buff_data_t end;
    unsigned char *head,
                  *data;
    unsigned int truesize;
    refcount_t users;
  };

其中：

  * ``next`` 和 ``prev`` 是指向缓冲区列表中下一个和前一个元素的指针；
  * ``dev`` 是发送或接收缓冲区内容的设备；
  * ``sk`` 是与缓冲区相关联的套接字；
  * ``destructor`` 是负责释放缓冲区的回调函数；
  * ``transport_header``, ``network_header`` 和 ``mac_header`` 是数据包起始位置和各个头部起始位置之间的偏移量。它们由数据包经过的各个处理层内部维护。要获取指向头部的指针，请使用以下函数之一: :c:func:`tcp_hdr`, :c:func:`udp_hdr` 以及 :c:func:`ip_hdr` 等。原则上，每个协议都对应一个函数。这些函数用于在接收到的数据包中，获取对该协议的头部的引用。请注意，当数据包到达网络层时, ``network_header`` 字段才被设置；而当数据包到达传输层时, ``transport_header`` 字段才被设置。

`IP 头部 <https://zh.wikipedia.org/zh-cn/IPv4#首部>`_ 的结构 (:c:type:`struct iphdr`) 包含以下字段：

.. code-block:: c

  struct iphdr {
  #if defined(__LITTLE_ENDIAN_BITFIELD)
  	__u8	ihl:4,
  		version:4;
  #elif defined (__BIG_ENDIAN_BITFIELD)
  	__u8	version:4,
    		ihl:4;
  #else
  #error	"Please fix <asm/byteorder.h>"
  #endif
  	__u8	tos;
  	__be16	tot_len;
  	__be16	id;
  	__be16	frag_off;
  	__u8	ttl;
  	__u8	protocol;
  	__sum16	check;
  	__be32	saddr;
  	__be32	daddr;
  	/* 可选项由此开始 */
  };

其中：

  * ``protocol`` 是使用的传输层协议；
  * ``saddr`` 是源 IP 地址；
  * ``daddr`` 是目标 IP 地址。

`TCP 头部 <https://zh.wikipedia.org/zh-cn/传输控制协议#封包結構>`_ 的结构 (:c:type:`struct tcphdr`) 具有以下字段：

.. code-block:: c

  struct tcphdr {
  	__be16	source;
  	__be16	dest;
  	__be32	seq;
  	__be32	ack_seq;
  #if defined(__LITTLE_ENDIAN_BITFIELD)
  	__u16	res1:4,
  		doff:4,
  		fin:1,
  		syn:1,
  		rst:1,
  		psh:1,
  		ack:1,
  		urg:1,
  		ece:1,
  		cwr:1;
  #elif defined(__BIG_ENDIAN_BITFIELD)
  	__u16	doff:4,
  		res1:4,
  		cwr:1,
  		ece:1,
  		urg:1,
  		ack:1,
  		psh:1,
  		rst:1,
  		syn:1,
  		fin:1;
  #else
  #error	"Adjust your <asm/byteorder.h> defines"
  #endif
  	__be16	window;
  	__sum16	check;
  	__be16	urg_ptr;
  };

其中：

  * ``source`` 是源端口；
  * ``dest`` 是目标端口；
  * 常使用的 TCP 标志包括 ``syn``, ``ack`` 以及 ``fin``；要想对其有更详细地了解，请参见此 `图表 <http://www.eventhelix.com/Realtimemantra/Networking/Tcp.pdf>`_。

`UDP 头部 <https://zh.wikipedia.org/zh-cn/用户数据报协议#UDP的分组结构>`_ 的结构（:c:type:`struct udphdr`）具有以下字段：

.. code-block:: c

  struct udphdr {
  	__be16	source;
  	__be16	dest;
  	__be16	len;
  	__sum16	check;
  };

其中：

  * ``source`` 是源端口；
  * ``dest`` 是目标端口。

访问网络数据包头部中的信息的示例如下：

.. code-block:: c

  	struct sk_buff *skb;

  	struct iphdr *iph = ip_hdr(skb);                 /* IP 头部 */
  	/* iph->saddr  - 源 IP 地址 */
  	/* iph->daddr  - 目标 IP 地址 */
  	if (iph->protocol == IPPROTO_TCP) {              /* TCP 协议 */
  		struct tcphdr *tcph = tcp_hdr(skb);      /* TCP 头部 */
  		/* tcph->source —— 源 TCP 端口 */
  		/* tcph->dest   —— 目标 TCP 端口 */
  	} else if (iph->protocol == IPPROTO_UDP) {       /* UDP 协议 */
  		struct udphdr *udph = udp_hdr(skb);      /* UDP 头部 */
  		/* udph->source —— 源 UDP 端口 */
  		/* udph->dest   —— 目标 UDP 端口 */
  	}

.. _Conversions:

转换
===========

不同系统的字，有多种字节的顺序方式 (`字节序 <http://zh.wikipedia.org/zh-cn/字节序>`_)，包括: `大端序 <http://zh.wikipedia.org/zh-cn/字节序#大端序>`_ (最高位字节在前) 和 `小端序 <http://zh.wikipedia.org/zh-cn/字节序#小端序>`_ (最低位字节在前)。由于网络连接了具有不同平台的系统，因此互联网对于数值数据的存储已经强加了一种标准序列，称为 `网络字节序 <http://zh.wikipedia.org/zh-cn/字节序#网络序>`_。相反，主机计算机上，表示数值数据的字节序列称为主机字节序。从网络接收/发送的数据采用网络字节序格式，并且应该在该格式和主机字节序之间进行转换。

为了进行转换，我们使用以下宏：

  * ``u16 htons(u16 x)`` 将 16 位整数从主机字节序转换为网络字节序（主机到网络短整数）；
  * ``u32 htonl(u32 x)`` 将 32 位整数从主机字节序转换为网络字节序（主机到网络长整数）；
  * ``u16 ntohs(u16 x)`` 将 16 位整数从网络字节序转换为主机字节序（网络到主机短整数）；
  * ``u32 ntohl(u32 x)`` 将 32 位整数从网络字节序转换为主机字节序（网络到主机长整数）。

.. _netfilter:

netfilter
=========

Netfilter 是一个内核接口，用于捕获网络数据包以对其进行修改/分析（用于过滤、NAT 等）。在用户空间中，由 `iptables <http://www.frozentux.net/documents/iptables-tutorial/>`_ 使用 `netfilter <http://www.netfilter.org/>`_ 接口。

在 Linux 内核中，使用 netfilter 进行数据包捕获是通过附加钩子（hook）来实现的。钩子可以在内核网络数据包所经过的路径的不同位置指定，你可以根据需要进行配置。你可以在 `这里 <http://linux-ip.net/nf/nfk-traversal.png>`_ 找到一张组织图，组织图上显示数据包所经过的路径以及钩子可能出现的区域。

使用 netfilter 时包含的头文件是 :file:`linux/netfilter.h`。

钩子通过 :c:type:`struct nf_hook_ops` 结构体进行定义：

.. code-block:: c

  struct nf_hook_ops {
  	/* 用户从这里开始填写。*/
  	nf_hookfn               *hook;
  	struct net_device       *dev;
  	void                    *priv;
  	u_int8_t                pf;
  	unsigned int            hooknum;
  	/* 钩子按优先级升序排列。*/
  	int                     priority;
  };

其中：

  * ``pf`` 是数据包类型 (``PF_INET`` 等)；
  * ``priority`` 是优先级；优先级在 :file:`uapi/linux/netfilter_ipv4.h` 中定义如下：

.. code-block:: c

  enum nf_ip_hook_priorities {
  	NF_IP_PRI_FIRST = INT_MIN,
  	NF_IP_PRI_CONNTRACK_DEFRAG = -400,
  	NF_IP_PRI_RAW = -300,
  	NF_IP_PRI_SELINUX_FIRST = -225,
  	NF_IP_PRI_CONNTRACK = -200,
  	NF_IP_PRI_MANGLE = -150,
  	NF_IP_PRI_NAT_DST = -100,
  	NF_IP_PRI_FILTER = 0,
  	NF_IP_PRI_SECURITY = 50,
  	NF_IP_PRI_NAT_SRC = 100,
  	NF_IP_PRI_SELINUX_LAST = 225,
  	NF_IP_PRI_CONNTRACK_HELPER = 300,
  	NF_IP_PRI_CONNTRACK_CONFIRM = INT_MAX,
  	NF_IP_PRI_LAST = INT_MAX,
  };

\


  * ``dev`` 是捕获操作针对的设备（网络接口）；

  * ``hooknum`` 是我们使用的钩子类型。当捕获到数据包时，处理模式由 ``hooknum`` 和 ``hook`` 字段定义。对于 IP，钩子类型在 :file:`linux/netfilter.h` 中定义：

.. code-block:: c

  enum nf_inet_hooks {
  	NF_INET_PRE_ROUTING,
  	NF_INET_LOCAL_IN,
  	NF_INET_FORWARD,
  	NF_INET_LOCAL_OUT,
  	NF_INET_POST_ROUTING,
  	NF_INET_NUMHOOKS
  };

\

  * ``hook`` 是在捕获网络数据包时调用的处理程序（数据包以 :c:type:`struct sk_buff` 结构体形式发送）。 ``private`` 字段是传递给处理程序的私有信息。捕获处理程序的原型由 :c:type:`nf_hookfn` 类型定义：

.. code-block:: c

  struct nf_hook_state {
  	unsigned int hook;
  	u_int8_t pf;
  	struct net_device *in;
  	struct net_device *out;
  	struct sock *sk;
  	struct net *net;
  	int (*okfn)(struct net *, struct sock *, struct sk_buff *);
  };

  typedef unsigned int nf_hookfn(void *priv,
  			       struct sk_buff *skb,
  			       const struct nf_hook_state *state);

捕获函数 :c:func:`nf_hookfn` 中, ``priv`` 参数是 :c:type:`struct nf_hook_ops` 初始化时传递的私有信息。 ``skb`` 是指向捕获的网络数据包的指针。根据 ``skb`` 的信息，可以进行数据包过滤决策。函数的 ``state`` 参数是与数据包捕获相关的状态信息，包括输入接口、输出接口、优先级和钩子号。优先级和钩子号可使同一个函数被多个钩子调用。

捕获处理程序可以返回以下常量之一 (``NF_*``)：

.. code-block:: c

  /* hook 函数的响应结果 */
  #define NF_DROP 0
  #define NF_ACCEPT 1
  #define NF_STOLEN 2
  #define NF_QUEUE 3
  #define NF_REPEAT 4
  #define NF_STOP 5
  #define NF_MAX_VERDICT NF_STOP

``NF_DROP`` 用于过滤（忽略）数据包, ``NF_ACCEPT`` 用于接受数据包并将其转发。

通过使用在 :file:`linux/netfilter.h` 中定义的函数来注册/注销一个 hook：

.. code-block:: c

  /* 注册/注销 hook 点的函数 */
  int nf_register_net_hook(struct net *net, const struct nf_hook_ops *ops);
  void nf_unregister_net_hook(struct net *net, const struct nf_hook_ops *ops);
  int nf_register_net_hooks(struct net *net, const struct nf_hook_ops *reg,
  			  unsigned int n);
  void nf_unregister_net_hooks(struct net *net, const struct nf_hook_ops *reg,
  			     unsigned int n);


.. attention::

  在 Linux 内核 3.11-rc2 版本之前，作为 netfilter 钩子参数的 :c:type:`struct sk_buff` 结构，内部的头部提取函数有一些限制。虽然每次都可以使用 :c:func:`ip_hdr` 获取 IP 头部，但是用于获取 TCP 和 UDP 头部的 :c:func:`tcp_hdr` 和 :c:func:`udp_hdr` 函数，却只能对从系统内部，而非外部接收的数据包使用。对于后者的情况，需要手动计算数据包中的头部偏移量：

  .. code-block:: c

    // TCP 数据包 (iph->protocol == IPPROTO_TCP)
    tcph = (struct tcphdr*)((__u32*)iph + iph->ihl);
    // UDP 数据包 (iph->protocol == IPPROTO_UDP)
    udph = (struct udphdr*)((__u32*)iph + iph->ihl);

  这段代码适用于所有过滤场景，因此建议使用它来替代头部访问函数。

下面是一个 netfilter hook 的使用示例：

.. code-block:: c

  #include <linux/netfilter.h>
  #include <linux/netfilter_ipv4.h>
  #include <linux/net.h>
  #include <linux/in.h>
  #include <linux/skbuff.h>
  #include <linux/ip.h>
  #include <linux/tcp.h>

  static unsigned int my_nf_hookfn(void *priv,
  		struct sk_buff *skb,
  		const struct nf_hook_state *state)
  {
  	/* 处理数据包 */
  	//...

  	return NF_ACCEPT;
  }

  static struct nf_hook_ops my_nfho = {
  	.hook        = my_nf_hookfn,
  	.hooknum     = NF_INET_LOCAL_OUT,
  	.pf          = PF_INET,
  	.priority    = NF_IP_PRI_FIRST
  };

  int __init my_hook_init(void)
  {
  	return nf_register_net_hook(&init_net, &my_nfho);
  }

  void __exit my_hook_exit(void)
  {
  	nf_unregister_net_hook(&init_net, &my_nfho);
  }

  module_init(my_hook_init);
  module_exit(my_hook_exit);

netcat
======

在开发包含网络编程的应用程序时，常用的工具包括 netcat。它也被昵称为“TCP/IP 的瑞士军刀”。它可以用于以下功能：

  * 发起 TCP 连接；
  * 等待 TCP 连接；
  * 发送和接收 UDP 数据包；
  * 以十六进制转储（hexdump）格式显示流量；
  * 在建立连接后运行程序（例如，一个 shell）；
  * 在发送的数据包中设置特殊选项。

发起 TCP 连接：

.. code-block:: console

  nc 主机名 端口号

监听 TCP 端口：

.. code-block:: console

  nc -l -p 端口号

发送和接收 UDP 数据包，需要添加 ``-u`` 命令行选项。

.. note::

  命令是 :command:`nc`；通常 :command:`netcat` 是此命令的别名。还有其他实现 netcat 命令的版本，其中一些与经典实现参数略有不同。运行 :command:`man nc` 或 :command:`nc -h` 以查看如何使用它。

有关 netcat 的更多信息，请参阅以下 `教程 <https://www.win.tue.nl/~aeb/linux/hh/netcat_tutorial.pdf>`_。

进一步阅读
===========

#. 了解 Linux 网络内部
#. `Linux IP 网络`_
#. `TUX Web 服务器`_
#. `Beej 的互联网套接字网络编程指南`_
#. `内核中的网络编程——Kernel Korner`_
#. `深入 Linux 内核网络堆栈`_
#. `netfilter.org 项目`_
#. `深入了解 Iptables 和 Netfilter 架构`_
#. `Linux 基金会网络页面`_

.. _Linux IP 网络: http://www.cs.unh.edu/cnrg/gherrin/
.. _TUX Web 服务器: http://www.stllinux.org/meeting_notes/2001/0719/myTUX/
.. _Beej 的互联网套接字网络编程指南: https://www.beej.us/guide/bgnet/
.. _内核中的网络编程——Kernel Korner: http://www.linuxjournal.com/article/7660
.. _深入 Linux 内核网络堆栈: http://phrack.org/issues/61/13.html
.. _netfilter.org 项目: http://www.netfilter.org/
.. _深入了解 Iptables 和 Netfilter 架构: https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture
.. _Linux 基金会网络页面: http://www.linuxfoundation.org/en/Net:Main_Page

练习
====

.. include:: ../labs/exercises-summary.hrst
.. |LAB_NAME| replace:: networking

.. important::

  你需要确保内核支持 ``netfilter``。可以通过 ``CONFIG_NETFILTER`` 来启用它。要激活它，请在 :file:`linux` 目录中运行 :command:`make menuconfig`，并在 ``Networking support -> Networking options`` 中勾选 ``Network packet filtering framework (Netfilter)`` 选项。如果它未启用，请启用它（作为内置功能，而不是外部模块——必须带有 ``*`` 标记）。


1. 在内核空间中显示数据包
------------------------

编写一个内核模块，显示发起出站连接的 TCP 数据包的源地址和端口。从 :file:`1-2-netfilter` 中的代码开始，并填写标有 ``TODO 1`` 的区域，同时考虑下面的注释。

你需要注册一个类型为 ``NF_INET_LOCAL_OUT`` 的 netfilter 钩子，如 `netfilter`_ 部分所述。

借助 `struct sk_buff 结构`_, 你可以使用特定的函数访问数据包头部。:c:func:`ip_hdr` 函数以返回指向 :c:type:`struct iphdr` 结构的指针的形式，返回 IP 头部。:c:func:`tcp_hdr` 函数以返回指向 :c:type:`struct tcphdr` 结构的指针的形式，返回 TCP 头部。

`图表`_ 解释了如何建立 TCP 连接。连接初始化数据包在 TCP 头部中设置了 ``SYN`` 标志，并清除了 ``ACK`` 标志。

.. note::

  要显示源 IP 地址，请使用 printk 函数的 ``%pI4`` 格式。详细信息可以在 `内核文档 <https://www.kernel.org/doc/Documentation/printk-formats.txt>`_ (``IPv4 addresses`` 部分)中找到。以下是使用 ``%pI4`` 的示例代码片段：

  .. code-block:: c

    printk("IP address is %pI4\n", &iph->saddr);

  在使用 ``%pI4`` 格式时，printk 的实参是指针。因此，构造应为 ``&iph->saddr`` （带有 & 运算符）而不是 ``iph->saddr``。

在 TCP 头部中，源 TCP 端口以 `网络字节序`_ 格式表示。请阅读 :ref:`转换` 部分。使用 :c:func:`ntohs` 进行转换。

为了进行测试，请使用 :file:`1-2-netfilter/user/test-1.sh` 文件。该测试创建一个到本地主机的连接，然后由内核模块拦截和显示该连接。 :command:`make copy` 命令仅会在该脚本标记为可执行的情况下，才会将该脚本复制到虚拟机上。该脚本使用静态编译得到的 :command:`netcat` 工具，该工具的路径是 :file:`skels/networking/netcat`；此程序必须具有执行权限。

运行检查器后，输出应类似于下面的示例：

.. code-block:: c

  # ./test-1.sh
  [  229.783512] TCP connection initiated from 127.0.0.1:44716
  Should show up in filter.
  Check dmesg output.

2. 按目标地址进行过滤
---------------------

扩展练习 1 中的模块，以便你可以通过 ``MY_IOCTL_FILTER_ADDRESS`` ioctl 调用指定目标地址。注意只显示包含指定目标地址的数据包。为了解决这个任务，填写标有 ``TODO 2`` 的区域，并按照以下规范进行操作。

要实现 ioctl 例程，你必须填写 ``my_ioctl`` 函数。请查看 :ref:`ioctl` 部分的内容。从用户空间发送的地址使用 `网络字节序`_, 因此 **无需** 进行转换。

.. note::

  通过 ``ioctl`` 发送的 IP 地址是通过地址发送的，而不是通过值发送的。地址必须存储在 ``ioctl_set_addr`` 变量中。可以使用 :c:func:`copy_from_user` 进行复制。

要比较地址，请填写 ``test_daddr`` 函数。这里我们无需转换地址，即可将（使用网络字节序的）地址进行比较（如果从左到右相等，则反转后也相等）。

如果要显示按目标地址过滤出的连接初始化数据包（这些过滤出的数据包，其目标地址与我们通过 ioctl 例程发送的地址相符），那么 ``test_daddr`` 函数必须在 netfilter 钩子中调用。连接初始化数据包在 TCP 头部中设置了 ``SYN`` 标志，并清除了 ``ACK`` 标志。你需要检查两件事情：

  * TCP 标志；
  * 数据包的目标地址（使用 ``test_addr``）。

为了进行测试，请使用 :file:`1-2-netfilter/user/test-2.sh` 脚本。此脚本需要编译 :file:`1-2-netfilter/user/test.c` 文件以生成测试可执行文件。在物理系统上运行 :command:`make build` 命令时，会自动进行编译。只有标记为可执行，该测试脚本才会复制到虚拟机上。该脚本使用静态编译的 :command:`netcat` 工具（在 :file:`skels/networking/netcat` 中）；该可执行文件必须具有执行权限。

运行检查器后，输出应类似于下面的示例：

.. code-block:: console

  # ./test-2.sh
  [  797.673535] TCP connection initiated from 127.0.0.1:44721
  Should show up in filter.
  Should NOT show up in filter.
  Check dmesg output.

测试首先要求对 ``127.0.0.1`` IP 地址进行数据包过滤，然后对 ``127.0.0.2`` IP 地址进行过滤。第一个连接初始化数据包（到 ``127.0.0.1``）被过滤器拦截并显示，而第二个数据包（到 ``127.0.0.2``）则未被拦截。

3. 监听 TCP socket
-------------------

编写一个内核模块，在回环接口（loopback interface）（在 ``init_module`` 中）上创建监听连接的 TCP 套接字，监听端口为 ``60000``。从 :file:`3-4-tcp-sock` 中的代码开始，填写标有 ``TODO 1`` 的区域，同时考虑以下观察结果。

请阅读 `对 socket 结构的操作`_ 和 `struct proto_ops 结构`_ 部分。

``sock`` socket 是 ``服务器套接字``，因此必须处于监听状态。也就是说，必须对该 socket 执行 ``bind`` 和 ``listen`` 操作。在内核空间中，要执行类似于 ``bind`` 和 ``listen`` 的操作，你需要调用类似 ``sock->ops->...;`` 的函数。你可以调用的示例函数包括 ``sock->ops->bind``, ``sock->ops->listen`` 等。

.. note::

  例如，调用 ``sock->ops->bind`` 或 ``sock->ops->listen`` 函数，查看在 :c:func:`sys_bind` 和 :c:func:`sys_listen` 系统调用处理程序中调用它们的方式。

  在 Linux 内核源代码树的 ``net/socket.c`` 文件中，查找系统调用处理程序。

.. note::

  对于 ``listen`` 的第二个参数（backlog），请使用 ``LISTEN_BACKLOG``。

在模块的退出函数和标有错误标签的区域中记得释放 socket；可以使用 :c:func:`sock_release` 来释放。

要进行测试，请运行 :command:`3-4-tcp_sock/test-3.sh` 脚本。只有在标记为可执行时，该脚本才会通过 :command:`make copy` 复制到虚拟机上。

运行测试后，将显示一个 TCP 套接字，通过监听端口 ``60000`` 进行连接。

4. 在内核空间接受连接
--------------------

扩展上一个练习中的模块，以允许外部连接（无需发送任何消息，只需接受新连接）。填写标有 ``TODO 2`` 的区域。

请阅读 `对 socket 结构的操作`_ 和 `struct proto_ops 结构`_ 部分。

对于内核空间的 ``accept`` 等效操作，请参阅 :c:func:`sys_accept4` 系统调用处理程序。请查看 `lnet_sock_accept <https://elixir.bootlin.com/linux/v4.17/source/drivers/staging/lustre/lnet/lnet/lib-socket.c#L511>`_ 实现，以及如何使用 ``sock->ops->accept`` 调用。将倒数第二个参数 (``flags``) 的值设为 ``0``，将最后一个参数 (``kern``) 的值设为 ``true``。

.. note::

  在 Linux 内核源代码树的 ``net/socket.c`` 文件中查找系统调用处理程序。

.. note::

  必须使用 :c:func:`sock_create_lite` 函数创建新套接字 (``new_sock``)，然后使用以下方式配置其操作：

  .. code-block:: console

    newsock->ops = sock->ops;

打印目标 socket 的地址和端口。要查找 socket 的对等名称（即地址），请参考 :c:func:`sys_getpeername` 系统调用处理程序。

.. note::

  ``sock->ops->getname`` 函数的第一个参数是连接套接字，即通过 ``accept`` 调用初始化的 ``new_sock``。

  ``sock->ops->getname`` 函数的最后一个参数是 ``1``，表示我们想要了解有关端点或对等点 (*远程端* 或 *对等点*) 的信息。

  使用文件中定义的 ``print_sock_address`` 宏显示对等地址（由 ``raddr`` 变量指示）。

在模块的退出函数中，以及在错误标签之后，释放新创建的套接字（接受连接后）。在将 ``accept`` 代码添加到模块初始化函数之后，:command:`insmod` 操作将会阻塞，直到建立连接。你可以使用 :command:`netcat` 在该端口上解锁。因此，上一个练习中的测试脚本将无法工作。

要进行测试，请运行 :file:`3-4-tcp_sock/test-4.sh` 脚本。只有在标记为可执行时，该脚本才会通过 :command:`make copy` 复制到虚拟机上。

不会显示任何特殊内容（在内核缓冲区中）。测试的成功将由连接的建立来确定。然后使用 ``Ctrl+c`` 停止测试脚本，然后可以移除内核模块。

5. UDP 套接字发送方
-------------------

编写一个内核模块，其创建一个 UDP socket，并将来自 socket 的 ``MY_TEST_MESSAGE`` 宏的消息发送到回环地址的端口 ``60001``。

从 :file:`5-udp-sock` 中的代码开始。

请阅读 `对 socket 结构的操作`_ 和 `struct proto_ops 结构`_ 部分。

要了解如何在内核空间中发送消息，请参阅 :c:func:`sys_send` 系统调用处理程序或 `发送/接收消息`_。

.. hint::

  :c:type:`struct msghdr` 结构的 ``msg_name`` 字段必须初始化为目标地址（指向 :c:type:`struct sockaddr` 的指针）, ``msg_namelen`` 字段必须初始化为地址大小。

  将 :c:type:`struct msghdr` 结构的 ``msg_flags`` 字段初始化为 ``0``。

  将 :c:type:`struct msghdr` 结构的 ``msg_control`` 和 ``msg_controllen`` 字段分别初始化为 ``NULL`` 和 ``0``。

要发送消息，请使用 :c:func:`kernel_sendmsg`。

消息传输参数从内核空间中检索。在 :c:func:`kernel_sendmsg` 调用中，将 :c:type:`struct iovec` 结构指针转换为 :c:type:`struct kvec` 指针。

.. hint::

  :c:func:`kernel_sendmsg` 的最后两个参数分别为 ``1`` (I/O 向量的数量) 和 ``len`` (消息大小)。

要进行测试，请使用 :file:`test-5.sh` 脚本。只有在标记为可执行时，该脚本才会通过 :command:`make copy` 复制到虚拟机上。该脚本使用存储在 :file:`skels/networking/netcat` 中的静态编译的 ``netcat`` 工具；该可执行文件必须具有执行权限。

如果正确实现，运行 :file:`test-5.sh` 脚本后，将显示类似下面的输出中的 ``kernelsocket`` 消息：

.. code-block:: console

  /root # ./test-5.sh
  + pid=1059
  + sleep 1
  + nc -l -u -p 60001
  + insmod udp_sock.ko
  kernelsocket
  + rmmod udp_sock
  + kill 1059
