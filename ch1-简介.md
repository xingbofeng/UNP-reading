# 简介
## 时间获取用户程序
客户端代码：

```c
#include  "unp.h"

int
main(int argc, char **argv)
{
  int         sockfd, n;
  char        recvline[MAXLINE + 1];
  struct sockaddr_in  servaddr;

  if (argc != 2)
    err_quit("usage: a.out <IPaddress>");

  if ( (sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
    err_sys("socket error");

  bzero(&servaddr, sizeof(servaddr));
  servaddr.sin_family = AF_INET;
  servaddr.sin_port   = htons(13);  /* daytime server */
  if (inet_pton(AF_INET, argv[1], &servaddr.sin_addr) <= 0)
    err_quit("inet_pton error for %s", argv[1]);

  if (connect(sockfd, (SA *) &servaddr, sizeof(servaddr)) < 0)
    err_sys("connect error");

  while ( (n = read(sockfd, recvline, MAXLINE)) > 0) {
    recvline[n] = 0;  /* null terminate */
    if (fputs(recvline, stdout) == EOF)
      err_sys("fputs error");
  }
  if (n < 0)
    err_sys("read error");

  exit(0);
}
```

客户端代码的解释：

* socket函数创建一个网际(AF_INET)字节流(SOCK_STREAM)套接字，它是TCP套接字的一个花哨名字。该函数返回一个小整数描述符，以后的所有函数调用(如connect、read)就用该描述符来标识这个套接字

* 我们把服务器的IP地址和端口号填入一个网际套接字地址结构(一个名为servaddr的sockaddr_in结构变量)，使用bzero把这个整个结构清零后，置地址族为AF_INET，端口号为13(这是时间地址获取服务器众所周知的端口，支持该服务器的任何主机都使用这个端口号)。IP地址为第一个命令行参数的值(argv[1])，网际套接字地址结构中IP地址和端口号这两个成员必须使用特定格式，为此我们调用库函数htons(主机到网络短整数)去转换二进制端口号，又调用库函数inet_pton(呈现形式到数值)去把ASCII命令行参数转换为合适的格式。

* inet_pton是一个支持IPv6的新函数，以前的代码中使用inet_addr函数来把ASCII点分十进制数串变换为正确的格式，不该它有不少局限，这些局限在inet_pton中得以纠正

* bzero不是一个ANSIC函数，它起源于早期的Berkeley网络编程代码，在本书中使用它而不是memset，因为memset带有三个参数，并且第二个参数和第三个参数的类型相同，如果不小心出错，无法检测。几乎所有支持套接字API的厂商都提供bzero

* connect函数用于一个套接字时，将与由它指定的第二个参数指向的套接字地址结构指定的服务器建立一个TCP连接，该套接字地址结构的长度作为该函数的第三个参数指定，对于网际套接字地址结构，我们总是使用sizeof操作符由编译器来计算这个长度

* read函数读取服务器的应答，TCP 是一个没有记录边界的字节流协议，所以加上’\0’，如果服务器返回的数据量很大，我们不能保证一次read调用能返回服务器的整个应答。因此从TCP套接字读取数据时，我们总是需要把read编写在某个循环中，read返回零表示对端关闭连接，read返回负值表示发生错误

* exit终止程序运行，Unix在一个进程终止时总是关闭该进程所有打开的描述符，我们的TCP套接字就此被关闭

服务端代码：

```c
#include  "unp.h"
#include  <time.h>

int
main(int argc, char **argv)
{
  int         listenfd, connfd;
  struct sockaddr_in  servaddr;
  char        buff[MAXLINE];
  time_t        ticks;

  listenfd = Socket(AF_INET, SOCK_STREAM, 0);

  bzero(&servaddr, sizeof(servaddr));
  servaddr.sin_family      = AF_INET;
  servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
  servaddr.sin_port        = htons(13); /* daytime server */

  Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));

  Listen(listenfd, LISTENQ);

  for ( ; ; ) {
    connfd = Accept(listenfd, (SA *) NULL, NULL);

        ticks = time(NULL);
        snprintf(buff, sizeof(buff), "%.24s\r\n", ctime(&ticks));
        Write(connfd, buff, strlen(buff));

    Close(connfd);
  }
}

```

运行结果：

![](http://oczira72b.bkt.clouddn.com/18-3-18/87128822.jpg)

服务端代码的解释：

* TCP套接字的创建和客户端程序相同
* 把服务器的众所周知的端口绑定到套接字
* 把套接字转换为监听套接字：调用listen函数把该套接字转换为一个监听套接字，这样来自客户的外来连接就可以在该套接字上由内核接受。LISTENQ指定系统内核允许在这个监听描述符上排队的最大客户连接数。
接收客户连接，发送应当：通常情况下，服务器进程在accept调用中被投入睡眠，等待某个客户的连接的到达并内核接收。TCP连接使用所谓的三次握手来建立连接。握手完毕后accept返回，其返回值是一个称为已连接描述符的新描述符。该描述符用于与新近连接的那个客户通信。accept为每个连接到本服务器的客户返回一个新的描述符
* 终止连接：服务器通过调用close关闭与客户的连接，该调用引发正常的TCP连接终止序列。每个方向上发送一个FIN，每个FIN又由各自的对端确认。
* 本服务器一次只能处理一个客户。如果多个客户连接差不多同时到达，系统内核在某个最大数目的限制下把它们排入队列。然后每个返回一个给accept函数。

### OSI模型
![](http://oczira72b.bkt.clouddn.com/18-3-18/18176090.jpg)

### 测试用网络及主机
几个命令：

* netstat显示网络接口
* ifconfig获得接口详细信息
* ping连接某一个地址

### 64位地址结构
64位机器上long类型和指针类型与32位机器不同。long类型和指针类型占64位。而在32位机器上，它们只占32位。可以使用size_t类型，它随CPU位数变化而变化。