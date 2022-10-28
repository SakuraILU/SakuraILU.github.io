---
title: Socket网络编程-TCP编程
date: 2022-10-27 22:49:36
tags: [C, Socekt, TCP, Computer Network]
categories: Computer Network
top: 1
---
## 背景

遇到个比赛，比赛方提供了 C++代码，但是想用 Python 来弄强化学习，之前试了试 ctypes...学了之后才发现 Python 需要的是一个独立的进程而不是函数调用。想起来比设用的 Carla，似乎需要一个 Server 和 Client 的架构。目前的一个想法是 C++和 Python 之间的进程通讯，C++侧只能在算法里面做些设计，琢磨着应该是 init 和 step 里面加一些通讯的东西，这样的话应该是 1v1 的结构。
总之先看看 C++和 Py 通讯的 socket API 怎么弄吧，之前还没用过，只云过中科大的计网课 hh

<!-- more -->
## C++ Server Socket

建立基本上服务器的 Socket 的主要流程是：

1. 给进程建立一个用于网络通讯的 Socket（返回这个 Socket 的文件描述符）。既然 Unix 世界里 Everything is a file，Socekt 就是一个文件，进程读写 Socket 就是写入网络设备和读入网络设备咯。
2. 绑定 IP:Port。请求网卡设置这个 Socekt 的端口索引为 PORT，IP 是本机就不需要设置啦（bind API 里的 IP 地址是指该 socket 允许接受的连接进程地址，其他 IP 来的请求就请网卡忽视掉），由网卡读写这个 Socket，与互联网进行交流。
3. 向网卡设置 Sokect 为监听 Socket。一般服务器需要不断监听外部连接请求，所以要有一个专门做为监听口。每当 listen 到请求连接队列里有某个 Client 的连接请求后会 accept 并重新分配一个 Port 用于 Server 和这个 Client 的专门的通讯（返回这个 Socket 的文件描述符），而监听 Socekt 继续 listen 请求连接的队列。当然得到了对方的 IP：PORT，如果想拒绝连接的话 close 掉新创建的连接 socket 即可。
4. 服务器通过新开的通讯 Socket 与 Client 交流。因为 Socket 被抽象成文件，用 read 和 write API 读写就好。
5. 关闭 Socekt。同样，因为是文件，用 close 关闭 Socket 的描述符就好。

个人理解的抽象结构。准备之后做一下 CS144 的 Lab，深入理解一下。

```

    | listen socket |       |communicate socket|       (can generate other communicate sockets)
            ^                   |          ^
            |                   |          |
     |      |        |    |     v          |         |
     |  | 请求队列 |  |    | |写缓冲区|   |读缓冲区|   |
     |      ^        |    |     |          ^         |
            |                   v          |
    SRC  IP：PORT              SRC   IP:PORT
    DST  NONE: NONE            DST   IP:PORT
    | ----------------网卡 -------------------- |

```

### 创建 SOCKET

man 一下 socket，得到函数的 prototype

```c
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

domain 的话，手册上写了

```
The domain argument specifies a communication domain;  this  selects  the
protocol family which will be used for communication.  These families are
defined in <sys/socket.h>.  The formats currently understood by the Linux
kernel include:

Name         Purpose                                    Man page
AF_UNIX      Local communication                        unix(7)
AF_LOCAL     Synonym for AF_UNIX
AF_INET      IPv4 Internet protocols                    ip(7)
```

macro 定义在头文件<sys/socket.h>里面的。可以看到用 IPV4 网络层通讯协议的话需要传入宏 AF_INET。本机通讯的话 ip 地址设置成 127.0.0.1 就 ok 了，IPV4 也是可以应用的。头两个倒是看起来专门用于 local communicate 的，以后再了解吧。

type 的话，有

```
The  socket has the indicated type, which specifies the communication se‐
mantics.  Currently defined types are:

SOCK_STREAM     Provides sequenced, reliable,  two-way,  connection-based
                byte streams.  An out-of-band data transmission mechanism
                may be supported.

SOCK_DGRAM      Supports datagrams (connectionless,  unreliable  messages
                of a fixed maximum length).
```

SOCK_STREAM 是面向连接的可靠通讯（可能是 TCP），SOCK_DGRAM 是无连接的不可靠通讯（可能是 UDP）。

看起来这个就已经设置好了通讯方式，protocal 里面的 TCP 和 UDP 这些似乎可以不设置了？后面跟着有

```
Normally only a single protocol exists to  support  a  particular  socket
type within a given protocol family, in which case protocol can be speci‐
fied as 0.
```

看起来的确如此，传入 0 就可以了。

因此需要传入的就是网络层协议 domain 和传输层协议 type。可通过如下调用创建基于 IPV4/TCP 连接的 SOCKET 并得到 socket 的文件描述符

```c
int sfd = socket(AF_INET, SOCK_STREAM, 0); //sfd short for server file descriptor
if(sfd < 0)
    panic("fail to create TCP/IPV4 socket...\n");
```

### 绑定 IP：PORT

man 一下 bind，protocol 是

```
SYNOPSIS
       #include <sys/types.h>          /* See NOTES */
       #include <sys/socket.h>

       int bind(int sockfd, const struct sockaddr *addr,
                socklen_t addrlen);
```

sockfd 就是我们得到的 sfd, addr 似乎是存地址的结构体，addrlen 就是地址结构体的长度吧。
后面跟着一段

```
When  a  socket is created with socket(2), it exists in a name space (ad‐
dress family) but has no address assigned to it.  bind() assigns the  ad‐
dress  specified by addr to the socket referred to by the file descriptor
sockfd.  addrlen specifies the size, in bytes, of the  address  structure
pointed to by addr.  Traditionally, this operation is called “assigning a
name to a socket”.

It is normally necessary to assign a local address using bind() before  a
SOCK_STREAM socket may receive connections (see accept(2)).
```

这里基本上也说完了创建 socket 的整个流程：

1. create socket by socket()
2. assign adress to the socket by bind()
3. accept connections by accept()
4. receive connections. 这时以及以后都是常规 read、write、close 了，和本机读写设备一样

<u>可以发现，尽管手册虽然比较冗长，不过毕竟是第一手资料，适当 search 一下手册还是能看到很多好东西的。</u>

然后又告诉了我们一个很关键的东西

```
The  rules  used  in name binding vary between address families.  Consult
the manual entries in Section 7 for detailed information.   For  AF_INET,
see  ip(7);  for  AF_INET6,  see  ipv6(7);  for AF_UNIX, see unix(7); for
AF_APPLETALK, see ddp(7); for AF_PACKET, see packet(7); for  AF_X25,  see
x25(7); and for AF_NETLINK, see netlink(7).
```

所以基于不同网络层的 socket 对应要 bind 的 address 结构都不大一样。对于 IPV4 的 AF_INET 而言，man 7 ip 查看到了

```
protocol  is  the  IP  protocol  in the IP header to be received or sent.
Valid values for protocol include:

• 0 and IPPROTO_TCP for tcp(7) stream sockets;

• 0 and IPPROTO_UDP for udp(7) datagram sockets;

• IPPROTO_SCTP for sctp(7) stream sockets; and

• IPPROTO_UDPLITE for udplite(7) datagram sockets.
```

ok，的确和之前猜的一样，socket()函数的 protocol 参数设置为 0 的话使用默认的常用传输层协议，TCP 和 UDP 这两个。后面两个不知道是啥玩意儿。。。

后面很快找到了两个 bind 需要的结构体

<p id="in_addr"></p>

```
struct sockaddr_in {
    sa_family_t    sin_family; /* address family: AF_INET */
    in_port_t      sin_port;   /* port in network byte order */
    struct in_addr sin_addr;   /* internet address */
};

/* Internet address. */
struct in_addr {
    uint32_t       s_addr;     /* address in network byte order */
};
```

需要设置的有协议类型、服务器端口和服务器接受的端口地址:

1. sin_family：注释里已经说明了必须是 AF_INET...我们正好也是在找 AF_INET/SOCKET_STREAM 需要 bind 的地址类型嘛。man page 跟着有描述

   ```
   sin_family is always set to AF_INET.
   ```

2. sin_port：手册里有

   ```
   sin_port contains the port in network byte order. The port numbers below
   1024 are called privileged ports (or sometimes: reserved ports). Only a
   privileged process (on Linux: a process that has the CAP_NET_BIND_SERVICE
   capability in the user namespace governing its network namespace) may
   bind(2) to these sockets.

   ```

   大概介绍了 sin_port 中需要设置的是 network byte order 的端口号。以及 1024 以内的是系统级的端口，后面的才是用户级端口，可以被用户进程使用并 bind（）IP：PORT。现在的问题是 network byte order 是啥？手册上面没提到，<a href="://www.tutorialspoint.com/unix_sockets/network_byte_orders.htm">STFW 一下</a>。大概是说有些机器是大端字节表示，而有些是小端的。为了统一所有机器的网络传输，规定全部采用 network byte order（就是大端表示法，不知道是不是什么历史包袱，感觉小端感觉更合适吧，用的更多而且阅读二进制序列都更好读一些）。
   所谓小端的话就是数据的低字节放在低地址处，而大端就是数据的高字节放在地地址处，刚好按字节颠倒一下。
   个人电脑应该基本上都是小端序吧，readelf -h somefile.out 就可以看到

   ```
   Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
   Class:                             ELF64
   Data:                              2's complement, little endian
   ******
   ```

   网站教程提到了可以采用如下 API 进行本机（host）和网络（network）二进制序列之间的转换

   ```
   Function        	Description
   htons()	        Host to Network Short
   htonl()	        Host to Network Long
   ntohl()	        Network to Host Long
   ntohs()	        Network to Host Short
   ```

   又知道 IP 是 32 位(long)、端口是 16 位（short），设置端口是 sockaddr_in.sin_port = htons(12000)。

3. sin_addr：sin_addr 是个结构体，是指 server 所接受的 IP 地址！！！

   ```
   sin_addr is the IP host address.  The s_addr  member  of  struct  in_addr
   contains  the  host  interface  address  in  network byte order.  in_addr
   should be assigned one of the INADDR_* values (e.g., INADDR_LOOPBACK) us‐
   ing    htonl(3)    or   set   using   the   inet_aton(3),   inet_addr(3),
   inet_makeaddr(3) library functions or directly  with  the  name  resolver
   (see gethostbyname(3)).
   ```

   sin_addr 是个结构体，<a href="#in_addr">上文有提到</a>，但是其实里面只有一个 member，s_addr...有点奇怪的设计。IP 地址存在 sin_addr.s_addr 里面，同样是 network byte order。里面提到了转换用的几个 API。
   大概也能推测出转换过程是十点表示法->二进制小端序列->二进制大端序
   127.0.0.1 -> 0x7F|00|00|01 (0b01111111|00000000|00000000|00000001) -> 0x01|00|00|7F
   man 了一下，里面 inet_addr 是最方便的，能够帮我们从十点表示法字符串（如"127.0.0.1"）直接转成 network ordered byte，也比较符合我的使用习惯。

   ```
    SYNOPSIS
        #include <sys/socket.h>
        #include <netinet/in.h>
        #include <arpa/inet.h>

        in_addr_t inet_addr(const char *cp);

   ```

   手册上后面还提到了支持 a、a.b、a.b.c、a.b.c.d 四种格式
   也就是可以接受

   ```
   A类网络地址集合：a.any.any.any
   B类网络地址集合：a.b.any.any
   C类网络地址集合: a.b.c.any
   单个网络地址： a.b.c.d
   ```

   因此设置 IP 地址的方式可以是 sockaddr_in.sin_addr.s_addr = inet_addr("127.0.0.1")，这样设置的话就是说 server 只接受本机进程连接。有个 macro INADDR_LOOPBACK 是"127.0.0.1"的二进制表示

   另外还有个特殊的 IP 0.0.0.0, 表示 server 接受整个互联网的 IP 地址。设置 sockaddr_in.sin_addr.s_addr = inet_addr("0.0.0.0")即可接受全部 IP 地址的连接请求。

   这两个地址比较常用，所以有它们对应的 macro 定义。

   ```c
    /* Address to accept any incoming messages.  */
    #define	INADDR_ANY		((in_addr_t) 0x00000000)
    /* Address to loopback in software to local host.  */
    # define INADDR_LOOPBACK	((in_addr_t) 0x7f000001) /* Inet 127.0.0.1.  */
   ```

   宏定义是本机（host）的二进制格式，需要转 network byte order，又由于 IPv4 是 32 位的，转换 API 即为 htonl()。于是 inet_addr("127.0.0.1"/"0.0.0.0")也等价于 htonl(INADDR_LOOPBACK/INADDR_ANY)。

总结起来，bind IP：PORT 为 127.0.0.1:12000 的方式是

```c
#define PORT (12000)
#define IP "127.0.0.1"
//alloc socket adress strcture for AF_INET
struct sockaddr_in s_addr;
//clear all rubbish data in s_addr
memset(&s_addr, 0, sizeof(s_addr));
//set IP and PORT
s_addr.sin_family = AF_INET;
s_addr.sin_port = htons(PORT);
s_addr.sin_addr.s_addr = inet_addr(IP);
//bind adress to the socket
if(bind(sfd, (struct sockaddr *)&s_addr, sizeof(s_addr)) < 0)
    panic("fail to bind socket to (%s, %d)...\n", IP, PORT);
```

### 设置 Socket 为监听 Socket

服务器一般需要一个端口不断监听请求队列，所以把创建的这个 socket 设置为监听 socket，网卡收到的对这个端口的连接请求都往这个 socket 里面写。man 一下 listen 的手册。

```
SYNOPSIS
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int listen(int sockfd, int backlog);
```

依然是这两个头文件 o(_^▽^_)o 。第一个肯定是需要设置为监听 socket 的文件描述符，第二个的参数的话，手册写的

```
The backlog argument defines the maximum length to  which  the  queue  of
pending connections for sockfd may grow. If a connection request arrives
when the queue is full, the client may receive an error with  an  indica‐
tion  of ECONNREFUSED ******
```

就是连接请求队列的最大长度，队列溢出的话回网卡会发回 Client 一个 ECONNREFUSED 的错误表示。

例如我们只需要一个 1v1 的服务器-客户端，可以设置

```c
#define QUEST_LEN (1)
if(listen(sfd, QUEST_LEN) < 0)
    panic("fail to set socket to listen socket...\n");
```

### Socket 监听连接请求

采用 accept API 监听并接受 Client 的连接

```
SYNOPSIS
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

第一个肯定是监听 socket 的文件描述符，第二个第三个分别是连接对象的 adress 结构体以及它的长度。
<u>需要注意第三个参数 addr_len 是指针传递，而且必须传它，否则会 accept 失败（试了试，直接传 0 的话遇到 Client 请求然后建立连接时，居然返回-1）。。。</u>
所以基本使用方式为

```c
struct sockaddr_in c_addr;
socklen_t c_addr_size = sizeof(struct sockaddr_in);
cfd = accept(sfd, (struct sockaddr *)&c_addr, &c_addr_size);
if(cfd < 0)
    panic("fail to accept...\n");
```

返回的 cfd 正是新创建的 server-client 链接 socket。
cfd 的端口当然是随机分配的咯，毕竟是 1v1 连接的 socket，server 和 client 的用户代码里都不需要知道彼此的 Port，只要有 Socket 文件描述符就行。

### 关于 sockaddr 参数

在 bind 手册后面紧跟着有这样一段话，之前没有截取出来

```
The  actual structure passed for the addr argument will depend on the ad‐
dress family.  The sockaddr structure is defined as something like:

struct sockaddr {
    sa_family_t sa_family;
    char        sa_data[14];
}

The only purpose of this structure  is  to  cast  the  structure  pointer
passed in addr in order to avoid compiler warnings.  See EXAMPLES below.
```

提醒了我们传参时需要将我们协议对应的的 addr 结构体（例如我们 AF_INET 对应的 sockaddr_in）强制类型转换成 struct sockaddr 以为了避免编译警告。

因为不同协议对于的地址参数结构不同，需要统一一下函数参数，最简单的当然是 void\*。由于每种地址结构体第一个 member 都是 sa_family_t，只要知道这个的值就知道结构体。所以用 sockaddr 强转一下，然后就可以读 sa_family_t 的值了（实际上直接强转成 sa_family_t 都行感觉。。。毕竟 C 语言，过于灵活，不知道他们库函数设计的真正考量），再根据其值强转成对应的结构体。很容易想到一个朴素的处理方式

```c
struct sockaddr *p = addr;
switch(p->sa_family_t){
    case AF_INET:
        struct sockaddr_in *p_addr = addr;
        //code: deal with IPV4
        break;
    case AF_INET6:
        struct sockaddr_in6 *p_addr = addr;
        //code: deal with IPV6
        break;
    case AF_UNIX:
        struct sockaddr_un *p_addr = addr;
        //code: deal with UNIX/LOCAL
    ******
    default:
        //code
}
```

强转 sockaddr 解析协议类型的代码明明可以放在函数里面完成，可以让用户使用起来更方便，但是设计者却把这个工作交给了用户，不太明白具体的设计考量，不过也只能遵循这个 API 设计了，在 socket()和 accept()中需要注意这点。

总之这就是为啥明明使用的是 sockaddr_in，但是传的时候却需要强转 sockaddr，毕竟还有各种各样其他的协议地址结构，需要统一 API。

### 关于 IP：PORT

Server 和 Client 要交流肯定需要约定一个大家都知道的地址，这就是负责监听连接请求的监听 Socket 的地址（服务器所在公有 IP 地址和域名当然问运营商要的，但是 PORT 这个端口索引是自己设置）。
建立连接请求后，server 的 TCP 层直接任意新建一个 Socket 然后和 Client 的地址绑定起来，两者交流时

1. Server 的读写都通过这个 Socekt 进行，用户无需知道彼此的 IP：PORT;
2. Client 读写自己建立的这个连接 Socekt 即可，也不需要知道彼此的 IP：PORT

因此，只有 listen socket 才需要 bind IP：PORT。

### 总结：C/C++ Socket server 代码框架

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#include <assert.h>

#define IN_LOOPBACK "127.0.0.1"
#define PORT (12000)
#define REQUEST_LEN (1)


#define panic(format, ...)             \
    do                                 \
    {                                  \
        printf(format, ##__VA_ARGS__); \
        assert(0);                     \
    } while (0)


int main()
{
    int sfd, cfd;
    sfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sfd < 0)
        panic("fail to establish the server socket...\n");

    struct sockaddr_in s_addr, c_addr;
    memset(&s_addr, 0, sizeof(s_addr));
    s_addr.sin_family = AF_INET;
    s_addr.sin_port = htons(PORT);
    s_addr.sin_addr.s_addr = inet_addr(IN_LOOPBACK);

    if (bind(sfd, (struct sockaddr *)&s_addr, sizeof(s_addr)) < 0)
        panic("fail to bind socket to (%s, %d)...\n", IP, PORT);

    if(listen(sfd, REQUEST_LEN) < 0)
        panic("fail to set socket to listen socket...\n");

    socklen_t c_addr_size = sizeof(struct sockaddr_in);
    cfd = accept(sfd, (struct sockaddr *)&c_addr, &c_addr_size);
    if (cfd == -1)
        panic("fail to accept...\n");


    //....
    //write your server code
    //  use read and write API to communicate with client...
    //....

    close(sfd);
    close(cfd);

    return 0;
}
```

## C++ Client Socket

客户端基本流程是

1. 创建 socket
2. 直接 connect 想连接的服务器服务器，connect API 会随机分配一个合适的 Port 给 Client。
3. 使用文件 API read/write 和服务器交流
4. 使用文件 API close API 关闭和服务器连接用的 Socket。

比 Server 简单多了，使用到的 API 只有一个 connect 是新的

### Connect Server

有了之前的一些经验，C socket 的套路还是清楚了很多，man 手册看到

```
SYNOPSIS
       #include <sys/types.h>          /* See NOTES */
       #include <sys/socket.h>

       int connect(int sockfd, const struct sockaddr *addr,
                   socklen_t addrlen);
```

根据 API 功能（让客户端创建的 socket 文件描述符连接到某个 IP：PORT 处的服务器进程）进行猜测，sockfd 是客户端创建的连接 socket 的文件描述符，addr 是申请连接服务器的地址，addrlen 则是 addr 结构体的长度。

但是注意一下这里面的 addr 与 server 侧的 bind 函数的 addr 区别

1. bind 里的 addr 中的 s_addr 是指服务器接受的请求连接进程的 IP 地址;
2. connect 里的 addr 中的 s_addr 是指申请连接的服务器的 IP 地址。
   所谓的一个 API 复用吧，不愧是底层语言，C 的设计实在是可谓寸土寸金啊。

所以申请连接的方式是

```c
#include <unistd.h>

#define INTVAL 3

while (connect(cfd, (struct sockaddr *)&sock_addr, sizeof(sock_addr)) < 0)
    {
        if (errno == ECONNREFUSED)
        {
            printf("fail to connect to %s, %d, wait 3s to reconnect...\n", IP, PORT);
            sleep(INTVAL);
        }
        else
        {
            panic("fail to connect to %s, %d, and reconnect is helpless...\n", IP, PORT);
        }
    }
```

如果是由于 EAFNOSUPPORT(The passed address didn't have the correct address family in its
sa_family field)导致连接失败，隔 3s 再重新申请一次连接，直到连接申请成功，其他原因则把进程直接 panic 掉。

### 总结：C/C++ Socket Client 代码框架

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#include <unistd.h>
#include <errno.h>


#include <assert.h>

#define PORT (12000)
#define IP "127.0.0.1"

#define panic(format, ...)             \
    do                                 \
    {                                  \
        printf(format, ##__VA_ARGS__); \
        assert(0);                     \
    } while (0)


int main()
{
    int cfd = 0;
    cfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sfd < 0)
        panic("fail to establish the server socket...\n");

    struct sockaddr_in sock_addr;
    sock_addr.sin_family = AF_INET;
    sock_addr.sin_port = htons(PORT);
    sock_addr.sin_addr.s_addr = inet_addr(IP);

    while (connect(cfd, (struct sockaddr *)&sock_addr, sizeof(sock_addr)) < 0)
    {
        if (errno == ECONNREFUSED)
        {
            printf("fail to connect to %s, %d, wait 3s to reconnect...\n", IP, PORT);
            sleep(3);
        }
        else
        {
            panic("fail to connect to %s, %d, and reconnect is helpless...\n", IP, PORT);
        }
    }

    //....
    //write your server code
    //  use read and write API to communicate with client...
    //....

    close(cfd);
}
```

## Python server Socket

Python 的 API 经过了高度封装，socket 模块也不例外，比起 C 而言简单太多。之前会用 C 的了，Python 的就很容易上手啦。
Python 有个 socket 模块，使用起来非常简单

### Python Socket Server 框架

```py
import socket
from struct import unpack, pack

addr = ("127.0.0.1", 11999)

sk = socket.socket(socket.AF_INET, socket.SOCK_STREAM, 0)
sk.bind(addr)
sk.listen(1)
csk, c_addr = sk.accept()

# Write your code here
# use two API to communicate:
#     1.csk.sendall(msg) to send msg
#     2.msg = csk.recv(max_bytes) to receive msg

sk.close()
csk.close()
```

基本上一看就能 get 到。只有几个需要注意的地方

1. socket()参数和 C 一模一样;
2. bind()的参数直接写一个元组(IP, PORT)即可，IP 直接是点十表示法的字符串，PORT 是个 INT，都是小端序即可！比 C/C++的 bind 简单好多，对 IP 和 PORT 的处理全被 API 封装起来啦！
3. listen()是 sk 的成员函数，自然只需要传第二个 backlog 参数了; 同理 accept()无参数
4. Python 毕竟是跨平台的，没有继承 everthing is a file 的思想，用 csk.sendall(msg)和 msg = csk.recv(max_bytes)来读写 Socekt
5. 用 csk.close()关闭 socket

### Python Socekt 数据传输：Pack&Unpack 结构体数据

Python 的一个核心思想是 Everything is an object...甚至一个 Int 都是对象，有它们的成员函数。但是网络传输时肯定只能传数据的二进制序列，所以涉及到了 pack(打包要写的数据)-sendall 和 recv-unpack(解码要写的数据)两个过程

Python 提供了两个 API--pack & unpack 做到这件事

pack(format,var1,var2,...)：简单来说就是把 Python Object 里面的成员变量拿出来排在一起，然后转换成对应的二进制格式，也就是弄成了 C 的存储格式。
var1、var2、...即是 Python 的基本类型，例如 Int、Long、Sring 这些，format 则是指定如何 pack 这些 vars。<a href="https://docs.python.org/3/library/struct.html">RTFM of struct module 后</a>可以找到一张表
| Format | C Type | Python type | Standard size | Notes |
|--------|--------|-------------|--------------|-------|
| x | pad byte|no value|c|char|bytes of length 1|
|1|b|signed char|integer|1|(1), (2)
|B|unsigned char|integer|1|(2)
|?|\_Bool|bool|1|(1)
|h|short|integer|2|(2)
|H|unsigned short|integer|2|(2)
|i|int|integer|4|(2)|
|I|unsigned int|integer|4|(2)
|l|long|integer|4|(2)
|L|unsigned long|integer|4|(2)
|q|long long|integer|8|(2)|Q|unsigned long long|integer|8|(2)
|n|ssize_t|integer||(3)
|N|size_t|integer||(3)
|e|(6)|float|2|(4)
|f|float|float|4|(4)
|d|double|float|8|(4)
|s|char[]|bytes
|p|char[]|bytes
|P|void\*|integer||(5)

format 中 n sym 有两种情况：

1. sym 不为 s 时：n sym 等价与 sym sym sym..sym [repeate n times]，例如"4i"等价于"iiii"
2. sym 为 s 时，"ns"意味着把对应的 string 给 pack 成 char[n]字符串

有了这张表之后，把 Python type pack 成 C struct 的方式就很简单了，例如 pack 一个{int;int;float;char[36]}的结构体然后进行 socket 传输

```py
# pack struct msg to send
x = 1
y = 1
z = 3.1415926
hello_str = "Hello, World!"
msg = pack("2if36s",x,y,z,hello_str.encode())

# send msg to socekt
csk.sendall(msg)
```

unpack 也是类似的用法, 返回一个元组，Python 可以直接拆分掉返回的元组，细节读读手册就行。例如解码这个 struct

```py
# receive msg from socket
msg = csk.recv(1024)
x, y, z, hello_str = unpack(msg)
hello_str = hello_str.decode()
```

还有一个需要注意的，可能 string 比其他 python 基本格式更复杂一点，pack 和 unpack 对 string 的支持仅是二进制格式的，所以需要分别用 encode()和 decode()函数进行二进制-Py 格式的转换，就像上述例子一样。

## Python Client Socket

至此位置已经没啥需要注意的啦！就是

1. socket()
2. connect()
3. recv() & sendall()
4. close()

### Python Socket Client 代码框架

直接贴代码框架

```py
import socket
from struct import unpack, pack
import time

INTVAL = 3
ip_port = ('127.0.0.1', 11999)
sk = socket.socket()            # 创建套接字

while True:
    try:
        sk.connect(ip_port)
        break
    except ConnectionRefusedError:
        print("Connection failed, try again in 3s later...")
        time.sleep(INTVAL)
    else:
        print("Unable to handle this error, kill the process...")
        exit(1)

# Write your code here
# use two API to communicate:
#     1.csk.sendall(msg) to send msg
#     2.msg = csk.recv(max_bytes) to receive msg


sk.close()

```

### 关于 Python Socket 的 Error

目前写 Demo 代码时只遇到了这个找不到服务器时 connect() API throw 出来的 ConnectionRefusedError，就类似 C/C++里的 errno == ECONNREFUSED。其他肯能的 error 就没处理啦，没怎么读 Python 的手册，以后遇到了再看吧。

## 总结

Socekt 通讯不受限编程语言、操作系统和计算机，只要约定好 Server 的 IP：PORT 以及传输数据的格式，任意进程都可以交流, C 和 Python 之间的本机通讯更不再话下了, 确实很 Powerful。
除了平时用到的各种互联网应用，还想起来以前做过的两个东西

1. 一门涉及 ROS 的工科创的课程，在本机上写了一些 node，涉及 C++ node 又涉及 Python node，之间好像也是用网络通讯交流，还规范了一个传输格式，只不过当时抱大腿比较混 hh 也没仔细想过这些，全靠大腿部署代码了(＠＾０＾)!
2. 还有毕设用的 Carla 平台，想着应该也是用的 Socekt 通讯吧，仿真模块独立运行，Python 启动后和仿真平台进行控制信号、状态量的数据传输。

