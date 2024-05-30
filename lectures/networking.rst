==================
网络管理
==================

.. meta::
   :description: 本文介绍了网络管理的相关概念和实现，包括套接字实现、路由实现、网络设备接口以及硬件和软件加速技术等内容。
   :keywords: 网络管理, 套接字实现, 路由实现, 网络设备接口, 硬件和软件加速

这个元信息描述了这个文档的内容和关键词,希望对您有帮助。如果还有其他需要补充的地方,请告诉我。

`查看幻灯片 <networking-slides.html>`_

.. slideconf::
   :autoslides: False
   :theme: single-level

课程目标：
===================

.. slide:: 网络管理
   :inline-contents: True
   :level: 2

   * 套接字实现

   * 路由实现

   * 网络设备接口

   * 硬件和软件加速技术


网络管理概述
===========================

.. slide:: 网络管理概述
   :inline-contents: True
   :level: 2

   .. ditaa::
      :height: 100%

      +---------------------------+
      | Berkeley Socket Interface |
      +---------------------------+

      +---------------------------+
      |      Transport layer      |
      +-------------+-------------+
      |      TCP    |     UDP     |
      +-------------+-------------+

      +---------------------------+
      |      Network layer        |
      +-----+---------+-----------+
      | IP  | Routing | NetFilter |
      +-----+---------+-----------+

      +---------------------------+
      |     Data link layer       |
      +-------+-------+-----------+
      |  ETH  |  ARP  | BRIDGING  |
      +-------+-------+-----------+

      +---------------------------+
      |    Queuing discipline     |
      +---------------------------+

      +---------------------------+
      | Network device drivers    |
      +---------------------------+


Sockets 实现概述
===============================

.. slide:: Sockets 实现概述
   :inline-contents: True
   :level: 2

   .. ditaa::
      :height: 100%

                                        Socket
                                        File
      +------+                          Operations
      | FILE | ----------------------> +-----------+
      +------+                         | read      |
       | |       struct socket_alloc   +-----------+
       | |        +---------------+    | write     |
       | +------->| struct socket |    +-----------+
       | f_private| +-----------+ |    | select    |
       |          | | ...       | |    +-----------+
       |          | +-----------+ |    | ...       |
       |          +---------------+    +-----------+
       +--------->| struct inode  |
        f_inode   | +-----------+ |
                  | | ...       | |
                  | +-----------+ |
                  +---------------+


Sockets 系列和协议
===============================

.. slide:: Sockets 系列和协议
   :inline-contents: True
   :level: 2

   .. ditaa::
      :height: 100%



                        struct socket                +---------> struct proto_ops
                       +--------------------+        |          +-----------------+
                       | struct socket      |        |          | release         |
                       |                    |        |          +-----------------+
                       +--------------------+        |          | bind            |
                       | struct proto_ops * |--------+          +-----------------+
                       +--------------------+                   | connect         |
                       | ...           |                        +-----------------+
                       +---------------+                        | accept          |
             +---------| struct sock * |-------+                +-----------------+
             |         +---------------+       |                | sendmsg         |
             |                                 |                +-----------------+
             |                                 |                | recvmsg         |
             |                                 |                +-----------------+
             |                                 |                | poll            |
             |                                 |                +-----------------+
             |                                 |                | ...             |
             |                                 |                +-----------------+
             |                                 |
             v                                 v            +--> struct sk_prot
        struct tcp_sock                struct tcp_sock      |   +--------------------+
      +-------------------+          +-------------------+  |   | inet_dgram_connect |
      | struct inet_sock  |          | struct inet_sock  |  |   +--------------------+
      | +---------------+ |          | +---------------+ |  |   | inet_sendmsg       |
      | | struct sock   | |          | | struct sock   | |  |   +--------------------+
      | | +-----------+ | |          | | +-----------+ | |  |   | udp_poll           |
      | | | ...       | | |          | | | ...       | | |  |   +--------------------+
      | | +-----------+ | |          | | +-----------+ | |  |   | inet_release       |
      | +---------------+ |          | +---------------+ |  |   +--------------------+
      | | sk_prot *     | |          | | sk_prot *     | |--+   | inet_bind          |
      | +---------------+ |          | +---------------+ |      +--------------------+
      +-------------------+          +-------------------+      | ...                |
      |  ...              |          |  ...              |      +--------------------+
      +-------------------+          +-------------------+


示例：UDP 发送
-----------------

.. slide:: 示例：UDP 发送
   :inline-contents: True
   :level: 2


   .. code-block:: c

      char c;
      struct sockaddr_in addr;
      int s;

      s = socket(AF_INET, SOCK_DGRAM, 0);
      connect(s, (struct sockaddr*)&addr, sizeof(addr));
      write(s, &c, 1);
      close(s);


.. slide:: 示例：UDP 发送
   :inline-contents: True
   :level: 2

   .. ditaa::

      -:------------------------------------------------------------------------------------

      VFS layer                 sys_write → vfs_write → do_sync_write → filp->f_op->aio_write

      -:------------------------------------------------------------------------------------

      Generic socket layer      sock_aio_write → sock->ops->sendmsg

      -:------------------------------------------------------------------------------------

      IP socket layer           sk->sk_prot->sendmsg

      -:------------------------------------------------------------------------------------

      UDP socket layer          ip_append_data                   udp_flush_pending_frames
                                      |                              |
      -:------------------------------+------------------------------+-----------------------
                                      V                              V
      IP socket layer           skb = sock_alloc_send_skb();     ip_local_out
                                skb_queue_tail(sk, skb)

      -:------------------------------------------------------------------------------------

                                         routing


网络处理阶段
=========================

.. slide:: 网络处理阶段
   :inline-contents: True
   :level: 2

   * 中断处理程序——设备驱动程序从 RX 环获取数据，创建网络数据包，并将其放到网络栈队列等待处理

   * NET_SOFTIRQ——数据包通过网络栈层，并进行处理：解封装以太网帧，检查 IP 数据包并进行路由，如果是本地数据包，则解封装协议数据包（例如 TCP）并将其放到套接字里的队列

   * 进程上下文——应用程序从套接字里的队列获取数据或将数据推送到套接字里的队列


数据包路由
==============

.. slide:: 数据包路由
   :inline-contents: True
   :level: 2

   .. ditaa::

      +----------------------+           +----------------------+
      |     Application      |           |     Application      |
      +----------------------+           +----------------------+
         |            ^                     |            ^
         | send()     | recv()              | send()     | recv()
         V            |                     V            |
      +----------------------+           +----------------------+
      |       Socket         |           |       Socket         |
      +----------------------+           +----------------------+
         |            ^                     |            ^
         |            |                     |            |
         v            |                     v            |
      +---------------------------------------------------------+
      |                    Transport layer                      |
      +---------------------------------------------------------+
         |            ^                    |             ^
         |            |                    |             |
         v            |                    v             |
      +---------------------------------------------------------+
      |                    Network layer                        |
      +---------------------------------------------------------+
          |                                         ^
          |                                         |
          v                                         |
      /---------------------------------------------------------\
      |                     Routing                             |  ----> Drop packet
      \---------------------------------------------------------/
          ^             |             ^             |
          | RX          | TX          | RX          | TX
          |             v             |             v
      +-----------------------+   +-----------------------+
      | Network Device Driver |   | Network Device Driver |
      +-----------------------+   +-----------------------+


路由表
----------------

.. slide:: 路由表
   :inline-contents: True
   :level: 2


   .. code-block:: shell

      tavi@desktop-tavi:~/src/linux$ ip route list table main
      default via 172.30.240.1 dev eth0
      172.30.240.0/20 dev eth0 proto kernel scope link src 172.30.249.241

      tavi@desktop-tavi:~/src/linux$ ip route list table local
      broadcast 127.0.0.0 dev lo proto kernel scope link src 127.0.0.1
      local 127.0.0.0/8 dev lo proto kernel scope host src 127.0.0.1
      local 127.0.0.1 dev lo proto kernel scope host src 127.0.0.1
      broadcast 127.255.255.255 dev lo proto kernel scope link src 127.0.0.1
      broadcast 172.30.240.0 dev eth0 proto kernel scope link src 172.30.249.241
      local 172.30.249.241 dev eth0 proto kernel scope host src 172.30.249.241
      broadcast 172.30.255.255 dev eth0 proto kernel scope link src 172.30.249.241

      tavi@desktop-tavi:~/src/linux$ ip rule list
      0:      from all lookup local
      32766:  from all lookup main
      32767:  from all lookup default


路由策略数据库
-----------------------

.. slide:: 路由策略数据库
   :inline-contents: True
   :level: 2

   * “常规”路由仅使用目标地址进行路由

   * 为了增加灵活性，系统使用了“路由策略数据库”，其可以根据其他字段（如源地址、协议类型、传输端口等）进行不同的路由选择

   * 这被编码为一系列规则，根据优先级进行评估（优先级 0 最高）

   * 每个规则都有一个选择器（用于匹配数据包）和一个动作（如果数据包匹配，则采取什么动作）

   * 选择器：源地址、目标地址、服务类型（TOS）、输入接口、输出接口等

   * 动作：查找（lookup）/单播（unicast）——使用给定的路由表，黑洞（blackhole）——丢弃数据包，不可达（unreachable）——发送 ICMP 不可达消息并丢弃数据包等。



路由表处理
------------------------

.. slide:: 路由表处理
   :inline-contents: True
   :level: 2

   * 本地地址的特殊表 -> 根据地址族、类型、端口将数据包路由到套接字

   * 从最具体的路由开始检查每个路由条目（例如，192.168.0.0/24 在 192.168.0.0/16 之前）

   * 如果数据包目标地址与子网掩码进行逻辑 OR 运算后等于子网地址，则表示路由匹配

   * 一旦路由匹配，将检索以下信息：接口、链路层下一跳地址、网络下一跳地址


转发信息数据库
-------------------------------

.. slide:: 转发信息数据库（在 3.6 版本移除）
   :inline-contents: True
   :level: 2

   |_|

   .. image::  ../res/fidb-overview.png


.. slide:: 转发信息数据库（在 3.6 版本移除）
   :inline-contents: True
   :level: 2

   .. image::  ../res/fidb-details.png

.. slide:: 路由缓存（在 3.6 版本移除）
   :inline-contents: True
   :level: 2

   |_|

   .. image::  ../res/routing-cache.png

.. slide:: FIB TRIE
   :inline-contents: True
   :level: 2

   |_|

   .. image::  ../res/fib-trie.png

.. slide:: Compressed Trie
   :inline-contents: True
   :level: 2

   |_|

   .. image::  ../res/fib-trie-compressed.png


Netfilter
=========

.. slide:: Netfilter
   :inline-contents: True
   :level: 2


   * 实现数据包过滤和 NAT 的框架

   * 它在数据包流中的关键位置插入钩子（hook）：

     * NF_IP_PRE_ROUTING

     * NF_IP_LOCAL_IN

     * NF_IP_FORWARD

     * NF_IP_LOCAL_OUT

     * NF_IP_POST_ROUTING

     * NF_IP_NUMHOOKS



网络数据包/skb（结构 sk_buff）
=======================================

.. slide:: 网络数据包/skb
   :inline-contents: True
   :level: 2

   .. image:: ../res/skb.png


.. slide:: 结构 sk_buff
   :inline-contents: True
   :level: 2

   .. code-block:: c

      struct sk_buff {
          struct sk_buff *next;
          struct sk_buff *prev;

          struct sock *sk;
          ktime_t tstamp;
          struct net_device *dev;
          char cb[48];

          unsigned int len,
          data_len;
          __u16 mac_len,
          hdr_len;

          void (*destructor)(struct sk_buff *skb);

          sk_buff_data_t transport_header;
          sk_buff_data_t network_header;
          sk_buff_data_t mac_header;
          sk_buff_data_t tail;
          sk_buff_data_t end;

          unsigned char *head,
          *data;
          unsigned int truesize;
          atomic_t users;


.. slide:: skb APIs
   :inline-contents: True
   :level: 2

   .. code-block:: c

      /* 预留头部空间 */
      void skb_reserve(struct sk_buff *skb, int len);

      /* 在尾部添加数据 */
      unsigned char *skb_put(struct sk_buff *skb, unsigned int len);

      /* 在顶部添加数据 */
      unsigned char *skb_push(struct sk_buff *skb, unsigned int len);

      /* 丢弃顶部的数据 */
      unsigned char *skb_pull(struct sk_buff *skb, unsigned int len);

      /* 丢弃尾部的数据 */
      unsigned char *skb_trim(struct sk_buff *skb, unsigned int len);

      unsigned char *skb_transport_header(const struct sk_buff *skb);

      void skb_reset_transport_header(struct sk_buff *skb);

      void skb_set_transport_header(struct sk_buff *skb, const int offset);

      unsigned char *skb_network_header(const struct sk_buff *skb);

      void skb_reset_network_header(struct sk_buff *skb);

      void skb_set_network_header(struct sk_buff *skb, const int offset);

      unsigned char *skb_mac_header(const struct sk_buff *skb);

      int skb_mac_header_was_set(const struct sk_buff *skb);

      void skb_reset_mac_header(struct sk_buff *skb);

      void skb_set_mac_header(struct sk_buff *skb, const int offset);


.. slide:: skb 数据管理
   :inline-contents: True
   :level: 2

   |_|

   .. ditaa::
      :height: 50%

                    Head
                ^ +---------------+
      skb_push  | |               | | skb_reserve
                  +---------------+ v
                  | Data          | | skb_pull
                ^ |               | v
      skb_trim  | |          Tail |
                  +---------------+
                  |               | | skb_put
                  +---------------+ v
                              End


网络设备
==============

.. slide:: 网络设备接口
   :inline-contents: True
   :level: 2

   .. image::  ../res/net-dev-hw.png


.. slide:: 高级特性
   :inline-contents: True
   :level: 2

   * Scatter-Gather（散列-聚集）

   * 校验和外包：以太网、IP、UDP、TCP

   * 自适应中断处理（聚合、自适应）



硬件和软件加速技术
=============================================

.. slide:: TCP 卸载
   :inline-contents: True
   :level: 2

   * 完全外包——在硬件中实现 TCP/IP 协议栈

   * 问题：

     * 连接数量的扩展

     * 安全性

     * 一致性

.. slide:: 性能观察
   :inline-contents: True
   :level: 2

   * 性能与要处理的数据包数量成正比

   * 例如：如果一个端点可以每秒处理 60K 个数据包

     * 1538 MSS -> 738Mbps
     * 2038 MSS -> 978Mbps
     * 9038 MSS -> 4.3Gbps
     * 20738 MSS -> 9.9Gbps

.. slide:: 无状态外包
   :inline-contents: True
   :level: 2

   * 网络堆栈处理大数据包

   * 发送路径：硬件将大数据包分割为较小的数据包（TCP 分段外包）

   * 接收路径：硬件将小数据包聚合成较大的数据包（大体量接收外包——简称 LRO）


.. slide:: TCP 分段外包
   :inline-contents: True
   :level: 2

   .. image::  ../res/tso.png

.. slide:: 大体量接收外包
   :inline-contents: True
   :level: 2

   .. image::  ../res/lro.png



