# 基本TCP套接字编程
## 概述

![](http://oczira72b.bkt.clouddn.com/18-3-25/94733097.jpg)

如图，服务器首先启动，首先通过`socket()`创建套接字，然后通过`bind()`函数把一个本地协议赋予一个套接字，之后通过`listen()`函数将套接字转化为被动套接字，也就是`listen()`函数仅仅由服务端调用。完成三次握手之后，调用`accept()`函数开始数据传输。使用`read()`和`write()`完成客户和服务器之间的数据传输。在客户端调用`close()`之后结束传输，再调用`clise()`关闭连接套接字。

稍后客户进程启动，客户端首先通过`socket()`创建套接字，然后通过`connect()`函数试图连接服务器，这个阶段完成三次握手，然后`read()`和`write()`完成客户和服务器之间的数据传输，之后客户进程调用`close()`来请求断开连接，服务器收到后读取`EOF`，接着关闭连接，这时完成四次挥手的过程。下面就图中的每个函数，细细剖析他们的用途。

## socket函数
为了执行网络`I / O`，进程做的第一件事情就是调用`socket()`函数，指定期望的通信协议类型。

```c
#include <sys/socket.h>
int socket(int family ,int type, int protocol);
```

* `family`：指定协议族 
* `type`：指定套接字类型 
* `protocol`：指定某个协议，设为`0`，以选择所给定`family`和`type`组合的系统默认值。

`family`参数特定的常值定义如下，对于`TCP/IP`协议而言，一般设定为`AF_INET`或者`AF_INET6`：

![](http://oczira72b.bkt.clouddn.com/18-3-25/38960555.jpg)

`type`参数特定的常值定义如下，对于`TCP`协议，设定为`SOCK_STREAM`，对于`UDP`协议，设定为`SOCK_DGRAM`：

![](http://oczira72b.bkt.clouddn.com/18-3-25/64859877.jpg)

`protocol`参数特定的常值定义如下，如图所见，对于`TCP`协议，设定为`IPPROTO_TCP`，对于`UDP`协议，设定为`IPPROTO_UDP`：

![](http://oczira72b.bkt.clouddn.com/18-3-25/27958945.jpg)

`socket`函数调用成功的时候将返回一个小的非负整数值，成为套接字描述符，简称`sockfd`。为了得到这个描述符，我们制定了协议族和套接字类型，并未指定本地与远程协议地址。

## connect函数
客户进程调用此函数来建立与服务器间的连接。

```c
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr *servaddr,socklen_t addrlen);
```

* `sockfd`：由`socket`返回的套接字描述符。 
* `servaddr`：套接字地址结构，包含了服务器`IP`和端口号。 
* `addrlen`：套接字地址结构大小，防止读越界。

客户端调用`connect`时，将向服务器主动发起三路握手连接，直到连接建立和连接出错时才会返回，这里出错返回的可能有一下几种情况：

* `TCP`客户没有收到`SYN`分节的响应。（内核发一个`SYN`若无响应则等待`6s`再发一个，若仍无响应则等待`24s`后再发送一个。总共等待`75s`仍未收到则返回错误`ETIMEDOUT`） 
* 若对客户的`SYN`的响应是`RST`，表明服务器主机在我们指定的端口上没有进程在等待与之连接，客户端收到`RST`就会返回`ECONNREFUSED`错误。产生`RST`的三个条件是：目的地`SYN`到达却没有监听的服务器；`TCP`想取消一个已有连接；`TCP`接收到一个根本不存在连接上的分节。 
* 若客户发出的`SYN`在中间的某个路由器上引发了一个`destination unreachable`（目的地不可达）`ICMP`错误，则认为是一种软错误，在某个规定时间（比如上述`75s`）没有收到回应，内核则会把保存的信息作为`EHOSTUNREACH`或`ENETUNREACH`错误返回给进程。

书上对此举例如下：

![](http://oczira72b.bkt.clouddn.com/18-3-25/22740615.jpg)

## bind函数
`bind()`函数把一个本地协议地址赋予了一个套接字。协议地址时`32`位`IPv4`地址或`128`位的`IPv6`地址与`16`位的`TCP/UDP`端口号的组合。

在调用`bind()`函数可以制定一个特定的端口号，或者制定一个`IP`地址，或者两个都指定，或者两者都不指定。

函数声明如下：

```c
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr * myaddr,socklen_t addrlen);
```

* `sockfd`：套接字描述符。
* `myaddr`：套接字地址结构的指针。
* `addrlen`：上述结构的长度，防止内核越界。

服务器在启动时捆绑它们的众所周知的端口，例如时间获取服务的端口`13`。如果不调用`bind()`函数，当调用`connect()`或`listen()`的时候，`TCP`会创建一个临时的端口，这对于客户端来说很常见（毕竟我们从来没见过客户端程序调用过`bind()`函数），而对于`TCP`服务器来说就比较少见了，因为`TCP`服务器就是通过其众所周知的端口被大家认识。

> 服务端调用`bind()`一般都会绑定一个周知端口，但是也有例外。对于`RPC`服务器，它们通常就由内核为它们的监听套接字选择一个临时端口，而该端口随后通过`RPC端口映射器`进行注册，客户在`connet()`这些服务器之前，必须与端口映射器联系获得它们的临时端口。这种情况对于`UDP`的`RPC`也如此。

进程可以把一个特定的`IP`地址绑定到它的套接字上：对于客户端来说，这没有必要，因为内核将根据所外出网络接口来选择源`IP`地址。对于服务器来说，这将限定服务器只接收目的地为该`IP`地址的客户连接。

对于`IPv4`来说，通配地址由常值`INADDR_ANY`来指定，其值一般为`0`，它告知内核去选择`IP`地址，因此我们经常看到如下语句：

```c
struct sockaddr_in servaddr;
servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
```

同理，端口号指定为`0`时，内核就在`bind()`被调用的时候选择一个临时端口，不过`bind()`函数不返回内核选择的值，第二个参数有`const`限定。如果想要直到内核所选择的临时端口值，必须调用`getsockname()`来返回协议地址。

最后需要注意的是：`bind()`绑定保留端口号时需要超级用户权限。这就是为什么我们在`linux`下执行服务器程序的时候要加`sudo`，如果没有超级用户权限，绑定将会失败。

## listen函数
`listen()`函数由`TCP`服务器调用，其函数声明如下：

```c
#include <sys/socket.h>
int listen (int sockdfd , int backlog);
```

`listen()`函数主要有两个作用：

* 对于参数`sockfd`来说：当`socket()`函数创建一个套接字时，它被假设为一个主动套接字。`listen()`函数把该套接字转换成一个被动套接字，指示内核应接受指向该套接字的连接请求。 
* 对于参数`backlog`：规定了内核应该为相应套接字排队的最大连接数=未完成连接队列+已完成连接队列 

其中： 

* 未完成连接队列：表示服务器接收到客户端的`SYN`但还未建立三路握手连接的套接字（处于`SYN_RCVD`状态） 
* 已完成连接队列：表示已完成三路握手连接过程的套接字（`ESTABLISHED`状态）

![](http://oczira72b.bkt.clouddn.com/18-3-25/52341519.jpg)

关于这两个队列还有需要注意的地方：当客户端发送`SYN`分节到达服务器时，如果此时服务器的未完成连接队列是满的，服务器将忽略这个`SYN`分节，服务器不会立即给客户端回应一个`RST`，因为客户端有自己的重传机制，如果服务器发送`RST`，那么客户度端的`connect()`就会返回错误了。另外客户端无法区别`RST`究竟意味着“该端口没有服务器在监听”还是意味着“该端口有服务器在监听不过它的队列满了。”

## accept函数
`accept()`函数由`TCP`服务器调用，用于已完成连接队列对头返回下一个已完成连接。如果已完成连接队列为空，那么进程被投入睡眠。

```c
#include<sys/socket.h>
int accept (int sockfd, struct sockaddr *cliaddr ,socklen_t * addrlen);
```

* `sockfd`：套接字描述符 
* `cliaddr`：对端（客户）的协议地址 
* `addrlen`：大小

当`accept()`调用成功，将返回一个新的套接字描述符，例如：

```c
int connfd = Accept(listenfd, (SA*)NULL, NULL);
```

其中我们称`listenfd`为监听套接字描述符，称`connfd`为已连接套接字描述符。区分这两个套接字十分重要，一个服务器进程通常只需要一个监听套接字，但是却能够有很多已连接套接字（比如通过`fork`创建子进程），也就是说每有一个客户端建立连接时就会创建一个`connectfd`，当连接结束时，相应的已连接套接字就会被关闭。

通过指针我们可以得到客户端的套接字信息，但是如果我们对这些不感兴趣就可以设定他们为`NULL`，书中给出一个示例，服务器相应连接后，打印客户端的`IP`地址和端口号。

```c
#include  "unp.h"
#include  <time.h>

int
main(int argc, char **argv)
{
  int         listenfd, connfd;
  socklen_t     len;
  struct sockaddr_in  servaddr, cliaddr; // 定义客户端套接字地址结构
  char        buff[MAXLINE];
  time_t        ticks;

  listenfd = Socket(AF_INET, SOCK_STREAM, 0); // 创建监听套接字

  bzero(&servaddr, sizeof(servaddr));
  servaddr.sin_family      = AF_INET; // 设定地址族
  servaddr.sin_addr.s_addr = htonl(INADDR_ANY); // 设定IP地址为通配IP地址
  servaddr.sin_port        = htons(13); // 绑定端口号为13

  Bind(listenfd, (SA *) &servaddr, sizeof(servaddr)); // 调用bind，确立为服务端套接字

  Listen(listenfd, LISTENQ); // 调用listen，确定为被动套接字

  for ( ; ; ) {
    len = sizeof(cliaddr);
    connfd = Accept(listenfd, (SA *) &cliaddr, &len); // 调用accept接受请求
    printf("connection from %s, port %d\n",
         Inet_ntop(AF_INET, &cliaddr.sin_addr, buff, sizeof(buff)),
         ntohs(cliaddr.sin_port)); // 拿到客户端的IP和端口，使用inet_ntop转换为点分表达式，如127.0.0.1

        ticks = time(NULL);
        snprintf(buff, sizeof(buff), "%.24s\r\n", ctime(&ticks));
        Write(connfd, buff, strlen(buff));

    Close(connfd);
  }
}
```

运行结果：

![](http://oczira72b.bkt.clouddn.com/18-3-25/77956081.jpg)

## fork和exec函数
用`fork()`创建子进程的方法并不陌生，这里将用`fork()`编写并发服务器程序。

```c
#include <unisted.h>
pid_t fork(void);
```

在子进程中返回`0`，在父进程返回返回子进程`ID`，因为有这个父子的概念，所以`fork()`函数调用一次却要返回两次。

关于`fork()`函数的一些特性：

* 任何子进程只能有一个父进程，并且子进程可以通过`getppid`获取父进程`ID` 
* 父进程中调用`fork()`之前打开的描述符，在`fork()`之后与子进程分享，在网络通信中也正是应用到这个特性。

`fork()`的两个典型用法：

* 一个进程创建一个自身的副本，这样每个副本都可以在另一个副本执行其他任务的同时处理各自的某个操作。
* 一个进程想要执行另一个进程。fork函数创建一个副本，然后通过调用exec把其中一个副本替换成新的程序。

存放在硬盘你上的可执行程序文件能够被`Unix`执行的唯一方法：由一个现有的进程调用六个`exec`函数中的某一个。

```c
#include <unistd.h>
int execl(const char *path, const char *arg, ...);
int execlp(const char *file, const char *arg, ...);
int execle(const char *path, const char *arg,..., char * const envp[]);
int execv(const char *path, char *const argv[]);
int execvp(const char *file, char *const argv[]);
int execve(int fd,char *const argv[],char *const envp[]);
```

## 并发服务器
像上面提到的时间获取服务器，属于迭代服务器。这种服务器都被一个单一客户占用，如果客户请求时间长，则该服务器被长期占用，这显然会影响效率。所以，我们希望服务器能尽可能同时服务多个用户。这种服务器称为并发服务器。

并发服务器利用`fork()`函数创建一个子进程来服务客户。

一个典型的并发服务器模型：

```c
int
﻿main(int argc, char **argv)
{
  pid_t     pid;
  int   listenfd, connfd;
  socklen_t len;
  struct sockaddr_in  servaddr;
  time_t    ticks;
  //创建套接字
  listenfd = Socket(AF_INET, SOCK_STREAM, 0);
  //初始化套接字
  bzero(&servaddr, sizeof(servaddr));
  servaddr.sin_family      = AF_INET;//IPv4协议
  servaddr.sin_addr.s_addr = htonl(INADDR_ANY);//通配地址，一般为0
  servaddr.sin_port        = htons(13);//时间服务端口
  Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));
  Listen(listenfd, LISTENQ);
  for ( ; ; ) {
    connfd = Accept(listenfd, (SA *) &cliaddr, &len);
    if((pid = fork())==0)
    {
      close(listenfd);
      doit(connfd);
      close(connfd);
      exit();
    }
    Close(connfd);
  }
}
```

分析以上程序：

父进程：`pid`为子进程`ID`，不为`0`，则将`connfd`的引用套接字减`1`，父进程继续等待下一个客户连接

子进程：`fork()`函数之后，监听套接字和已连接套接字的引用计数都加`1`，`pid==0`，首先监听套接字`listenfd`的引用计数减1(不会关闭监听套接字)，然后执行客户所需的操作（`doit`），再关闭`connfd`(引用计数减`1`，此时为`0`)。子进程处理客户需求结束，`exit()`关闭进程。

* `fork()`并不是把父进程从头到尾执行一遍，否则这样不就无穷尽了。 
* 父进程在调用`fork()`处，整个父进程空间会原模原样地复制到子进程中，包括指令，变量值，程序调用栈，环境变量，缓冲区，等等。 
* 在并发服务器的示例中，子进程会将已连接的套接字（`connfd`）和监听套接字（`listenfd`）拷贝到自己的进程空间。 
* 对于套接字描述符来说，他们都有一个引用计数，`fork()`之后由于描述符被复制，其引用计数都变成了`2`。 
* 因此，我们在父进程中关闭`connfd`（因为我们在子进程中使用`connfd`），在子进程中关闭`listenfd`（因为我们在父进程中进行监听），此时他们的引用计数都变成了`1`。 
* 然后，我们所期望的状态就是父进程中保留一个监听套接字继续等待客户端连接，在子进程中通过已连接的套接字对客户端请求进行服务。 
* 最后在子进程中关闭`connfd`，或`exit(0)`，使得`connfd`真正完成清理和资源释放。

## close函数
`close()`函数可以用来关闭套接字并终止TCP连接。

```c
#include <unistd.h>
int close(int sockfd);
```

`close()`函数是对套接字描述符的引用计数减`1`，也就是说，如果调用`close()`后，引用计数不为0，将不会引起`TCP`的四分组连接终止序列，这正是父进程与子进程共享已连接套接字的并发服务器所期望的。不过如果我们确实想在`TCP`连接上发送一个`FIN`，那么调用`shutdown()`函数。

## getsockname和getpeername函数

```c
#include <sys/socket.h>
int getsockname(int sockfd,struct sockaddr*localaddr,socklen_t *addrlen);
int getpeername(int sockfd,struct sockaddr*peeraddr,socklen_t *addrlen); 
```

这两个函数的作用： 

* 首先我们知道在`TCP`客户端一般不使用`bind()`函数，当`connect()`返回后，`getsockname()`可以返回客户端本地`IP`地址和本地端口号。 
* 如果`bind()`绑定了端口号`0`（内核选择），由于`bind()`的参数是`const`型的，因此必须通过`getsockname()`去得到内核赋予的本地端口号。 
* 获取某个套接字的地址族 
* 以通配`IP`地址`bind()`的服务器上，`accept`成功返回之后，`getsockname()`可以用于返回内核赋予该连接的本地`IP`地址。其中套接字描述符参数必须是已连接的套接字描述符。

