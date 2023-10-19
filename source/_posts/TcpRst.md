---
title: Python实现TCP旁路阻断
date: 2019-12-02 19:54:32
categories:
- Program
- Python
tags:
- By-pass Block
---
## 1 背景

TCP旁路阻断技术通过监听通信信道，抓包还原TCP协议，然后通过给通信两方发送TCP Reset包实现旁路阻断TCP连接。

Python中可通过Raw Socket进行以太网嗅探和发送Reset包。

<!-- more -->

## 2 Raw Socket

Raw Socket提供一种方法来绕过整个网络堆栈遍历和直接将以太网帧数据送到一个应用程序。

有很多种方法来创建raw sockets，例如AF_INET，PF_INET，AF_PACKET，PF_PACKET等。相互之前的区别在于AF_INET和PF_INET只能到网络7层模型的网络层，而AF_PACKET和PF_PACKET可以到MAC层，即数据链路层。

### 2.1	传输层

最简单的传输层socket就是最常用的socket，而非raw socket。建立方式为：

```Python
sockfd = socket(AF_INET,SOCK_STREAM/SOCK_DGRAM, protocol)
```

收发数据用于TCP和UDP协议，发送和接收的数据都不包含UDP头或TCP头，使用ip地址+端口号作为地址。

### 2.2	网络层

建立方式为：

```Python
sockfd = socket(AF_INET,SOCK_STREAM/SOCK_DGRAM, protocol)
```

第一个参数：PF_INET/AF_INET。二者的的区别为指定address family时一般设置为AF_INET，即使用IP；指定protocal family时一般设置PF_INET。在头文件中有如下定义：

```Code
#define AF_INET 0
#define PF_INET AF_INET
```

所以AF_INET与PF_INET完全一样；

第二个参数说明建立的是一个raw socket；

第三个参数分为3种情况：

* a.参数protocol用来指明所要接收的协议号，如果是象IPPROTO_TCP(6)这种非0，非255号的协议，则内核碰到ip头中protocol域和创建socket所使用参数protocol相同的IP包，就会交给这个rawsocket来处理。

* b.如果protocol是IPPROTO_RAW(255)，这个socket只能用来发送IP包，而不能接收任何数据.发送的数据需要自己填充IP包头，并且自己计算校验和。

* c.如果protocol是IPPROTO_IP(0)，在linux和sco unix上是不允许建立的。

网络层raw socket的特点为该套接字可以接收协议类型为(icmp，igmp等)发往本机的ip数据包；不能收到非发往本地ip的数据包(ip软过滤会丢弃这些不是发往本机ip的数据包)；不能收到从本机发送出去的数据包；发送时需要自己组织tcp，udp，icmp等传输层协议头部，可以setsockopt来自己包装ip头部；接收的UDP和TCP协议号的数据不会传给任何原始套接字接口，UDP和TCP协议的数据需要通过MAC层原始套接字来获得。

### 2.3	数据链路层

建立方式为：

```Python
sockfd = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_IP/ETH_P_ARP/ETH_P_ALL))
```

发送和接收数据使用MAC地址，可以在MAC层发送和接收任意的数据，如果有上层协议需要自行添加协议头。

第三个参数protocol协议类型一共有四个：

* ETH_P_IP 0x800 只接收发往本机mac的ip类型的数据帧；

* ETH_P_ARP 0x806 只接受发往本机mac的arp类型的数据帧；

* ETH_P_ARP 0x8035 只接受发往本机mac的rarp类型的数据帧；

* ETH_P_ALL 0x3 接收发往本机mac的所有类型ip，arp，rarp的数据帧, 接收从本机发出的所有类型的数据帧.(混杂模式打开的情况下,会接收到非发往本地mac的数据帧)。

需要注意的是这一种socket使用的收发地址都是通过指定网卡实现，接受和发送的数据都是数据帧，格式如下：

---------------------------------------------------------------
    	|eth header|             data          |
---------------------------------------------------------------   

即： 

---------------------------------------------------------------
    |MACdesAddr(6)|MACsrcAddr(6)|protocal(2)|        data      |
--------------------------------------------------------------- 

综述MAC层socket可以接收发往本地mac的数据帧；可以接收从本机发送出去的数据帧(第3个参数需要设置为ETH_P_ALL)；可以接收非发往本地mac的数据帧(网卡需要设置为promisc混杂模式)。

## 3 具体实现

### 3.1	原理

把任意的TCP连接给Reset掉是比较容易的，因为TCP在收到数据包时，对发送者仅仅做以下简单的校验：五元组校验；校验和检测；序列号校验。如此松散的校验机制很容易就可以pass。

包构造的原理图如下图所示：

![](/img/TcpRst/TcpRst1.png)

### 3.2	实现

首先创建raw socket连接：

```Python
def listen():
    listen_socket = socket.socket(
        socket.AF_PACKET,
        socket.SOCK_RAW,
        socket.ntohs(0x0003)
    )
```

然后监听网卡，接收数据，解析数据帧，寻找tcp协议的数据帧，放弃掉原本的RST包以及FIN包，然后判断ip地址是否在黑名单中:

```Python
if sender in addresses:
    if sender not in stats["seen"]:
        stats["seen"].append(sender)
elif receiver in addresses:
    if receiver not in stats["seen"]:
        stats["seen"].append(receiver)
```

如果检测到禁止的tcp连接，就会给连接双方均发送TCP Reset包断开连接，实现tcp旁路阻断的功能。