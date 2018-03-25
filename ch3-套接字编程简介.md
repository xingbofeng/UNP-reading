# 套接字编程简介
## 套接字地址结构
大多说套接字函数都需要一个指向套接字地址结构的指针作为参数，每个协议族都定义它自己的套接字地址结构，这些结构都以`sockadd_`开头。

### IPv4套接字地址结构

![](http://oczira72b.bkt.clouddn.com/18-3-25/71079610.jpg)

```c
struct in_addr
{
  in_addr_t  s_addr;   // 32位的IPv4地址
};

struct sockaddr_in {

  uint8_t  sin_len; // 无符号的8位整数，占1个字节
  sa_family_t  sin_family; // 地址族，8位整数，占1个字节
  in_port_t  sin_port; // TCP或UDP端口号至少16位无符号整数，占2个字节
  struct  in_addr sin_addr; // 占32位，4字节，IPv4的地址
  char  sin_zero[8]; // 未使用的拓展字段，占8个字节，并且该字段必须为0
}; //套接字的大小至少是16个字节
```

通常我们在使用套接字的时候，只会用到三个字段：`sin_family`，`sin_addr`和`sin_port`，如下：

```c
struct  sockaddr_in  servaddr; // 声明一个套接字地址
bzero(&servaddr, sizeof(servaddr)); // 套接字结构体清0
servaddr.sin_family = AF_INET; // 使用ipv4的协议族，对于TCP/IP协议，必须设置为AF_INET
servaddr.sin_port   = htons(13); // 端口号，13为获取日期和时间的端口
servaddr.sin_addr.s_addr = htonl(INADDR_ANY); // IPv4地址，INADDR_ANY表示的是通配IP地址
```

* `sin_len`是无符号短整数，这个字段是对长度可变套接字地址结构的处理，由于IPv4套接字定长，所以不需要处理。
* `sin_zero`是拓展字段，占8个字节，暂未使用。

### 通用套接字地址结构
当作为一个参数传递进任何套接字函数时，套接字地址总是以引用形式来传递。但是在套接字地址结构设计的时候，并没有`void *`类型，类似于`void *`代表通用指针类型，通用套接字地址结构可以便于参数传递，使得套接字函数能够处理来自所支持的任何协议簇的套接字地址结构。

```c
struct sockaddr {
  uint8_t sa_len;
  sa_family_t sa_family;
  char sa_data[14]; // 协议特定地址字段，14个字节
}
```

于是套接字函数被定义为指向某个通用套接字地址结构的一个指针作为其参数之一，这正如`bind`函数的`ANSIC`函数原型所示：

```c
int bind(int, struct sockaddr *, socklen_t);
```

这就要求对这些函数的任何调用都必须要将指向特定于协议的套接字地址结构的指针进行强制类型转换，变成指向某个通用套接字地址结构的指针。例如：

```c
struct sockaddr_in serv;

bind(sockfd, (struct sockaddr *) &serv, sizeof(serv));
```

### IPv6的套接字地址结构

```c
struct in6_addr
{
  uint8_t s6_addr[16]; // 128位IPv6地址，网络字节序
}

struct sockaddr_in6
{
  uint8_t sin6_len; // 套接字地址结构的长度,28字节
  sa_family_t sin6_family; // AF_INET6
  in_port_t sin6_port;  // 传输端口
  uint32_t sin6_flowinfo; // IPv6 流标字段
  struct in6_addr sin6_addr;  // IPv6 地址
  uint32_t sin6_scope_id; // 对于具备范围的地址，此字段标志其范围
};
```

### 新的通用套接字地址结构
新的套接字地址结构`sockaddr_storage`是用户透明的：

```c
struct sockaddr_storage
{
  uint8_t ss_len;
  sa_family_t ss_family;
  // ...
};
```

从套接字层面上来说`IPv6`和`IPv4`的一些小对比：`IPv6`的地址族是`AF_INET6`，`IPv4`地址族是`AF_INET`。`IPv6`套接字结构最小`28`个字节，`IPv4`套接字结构最小`16`个字节，`IPv6`套接字API的一部分而定义的新的通用套接字地址结构克服了现有`struct sockaddr`的一些缺点，新的`struct sockaddr_storage`足以容纳系统所支持的任何套接字地址结构。

### 套接字地址结构的比较
![](http://oczira72b.bkt.clouddn.com/18-3-25/96137915.jpg)

## 值-结果参数
我们往套接字函数中传递套接字结构时传递套接字结构的指针，该结构的长度用`sizeof`来计算，也作为一个参数来传递，不过其传递方式取决于该结构的传递方向：是从进程到内核，还是从内核到进程。

### 进程到内核
这个方向传递套接字地址结构的套接字函数有3个：`bind`，`connect`和`sendto`，例如：

因为`(struct sockaddr*)`指向的内容是变长的，所以要把指针指向的内容也作为参数来传递。

```c
struct sockaddr_in serv;
connect(sockfd, (struct sockaddr*) &serv, sizeof(serv));
```

### 内核到进程
这类函数有4个，`accept`，`recvfrom`，`getsockname`和`getpeername`，其传递的是指向某个套接字结构的指针和指向表示该结构大小的整数变量和指针：

```c
struct sockaddr_un cli;
socklen_t len;
len = sizeof(cli); // len这里是一个整数 
getpeername(unixfd, (SA*)&cli, &len);// len的地址被传递进去，又作为结果返回出来
```

为什么在进程到内核传递的是长度，而在内核到进程传递的却是长度的地址：前者是为了告诉内核态结构的大小使得内核在该结构地址上操作时不至于越界，后者是作为结果来返回，告诉进程内核在结构中存储了多少信息。这样做的目的是便于内核更改这个地址指向的内容，以便作为结果返回。

> 对于`IPv4`的`sockaddr_in`传递与返回的大小都是16。

## 字节排序函数
关注如何在主机字节序和网络字节序之间相互转换。

对于一个`16`位整数，其`16`进制表示假设是`0x1234`，那么我们说：


* 这个数的高字节是`0x12`
* 这个数的低字节是`0x34`

而对于地址来说，我们常用`1`个字节的地址偏移来表示。这样小端和大端的定义如下：

* 低序字节存储在起始地址（偏移小的位置）称为小端字节序
* 高序字节存储在起始地址（偏移小的位置）称为大端字节序

那么按照上面的定义，`0x1234`在大/小端模式下应该是这样的：

| 地址偏移 | 大端模式 | 小端模式 |
|----------|:-------------:|------:|
| 0x00 | 0x12(高序字节) | 0x34(低序字节) |
| 0x01 | 0x34(低序字节) | 0x12(高序字节) |

测试机器究竟是大端字节序还是小端字节序：

```c
#include  "unp.h"

int main(int argc, char **argv)
{
  union {
    short  s;
    char   c[sizeof(short)];
  } un;

  un.s = 0x0102;
  printf("%s: ", CPU_VENDOR_OS);
  if (sizeof(short) == 2) {
    if (un.c[0] == 1 && un.c[1] == 2)
      printf("big-endian\n");
    else if (un.c[0] == 2 && un.c[1] == 1)
      printf("little-endian\n");
    else
      printf("unknown\n");
  } else
    printf("sizeof(short) = %d\n", sizeof(short));

  exit(0);
}

```

运行结果得知，在`MAC-OSX`系统上是小端字节序。

![](http://oczira72b.bkt.clouddn.com/18-3-25/73286964.jpg)

我们把大端和小端的字节存储顺序统称为“主机字节序”。对应的，当然就有网络字节序了。网际协议中使用大端字节序来传送这些多字节整数。两种字节序之间的转换有以下四个函数：

```c
#include <netinet/in.h>

// 主机字节序转为网络字节序
uint16_t htons(uint16_t host16bitvalue); // host to network short，常用于转16位的数据，例如TCP/UDP端口
uint32_t htonl(uint32_t host16bitvalue); // host to network long，常用于转32位的数据，例如IP地址

// 网络字节序转为主机字节序
uint16_t ntohs(uint16_t host16bitvalue); // network to host short，常用于转16位的数据，例如TCP/UDP端口
uint32_t htohl(uint32_t host16bitvalue); // network to host long，常用于转32位的数据，例如IP地址
```

## 字节操纵函数

```c
#include <string.h>
// b开头（表示字节）的一组函数
void bzero(void* dest, size_t nbytes); //将指针dest以后的n bytes位置0
void bcopy(const void* src, void *dest , size_t nbytes); //将指针src后的n bytes位复制到指针dest
int bcmp(const void *ptr1, const void* ptr2, size_t nbytes);//比较ptr1和ptr2后的n bytes位的大小
// mem开头（表示内存）的一组函数
void *memset(void *dest, int c, size_t len);//将dest开始的一段长度为len的内存的值设为c
void *memcpy(void *dest, const void* src, size_t nbytes);//同bcopy
int memcmp(const void *ptr1, const void *ptr2, size_t nbytes);//同bcmp
```

可以使用`man`命令查看响应的函数用法：

![](http://oczira72b.bkt.clouddn.com/18-3-25/39463065.jpg)

## 地址转换函数（现在主要使用`inet_pton`和`inet_ntop`）
这些函数有： `inet_aton`，`inet_addr`，`inet_ntoa`，`inet_pton`和`inet_ntop`。 
这些函数的功能是在`ASCII`字符串网络与字节序二进制之间转换网际地址，因为人们更熟悉使用字符串来标记，而存放在套接字地址结构里面的值往往是二进制。

```c
#include <arpa/inet.h>
// 将字符串形式的点分十进制字符串转换成为IPv4地址
int inet_aton(const char * strptr,struct in_addr *addrptr);
in_addr_t inet_addr(const char * strptr);

// 返回一个指向点分十进制字符串的指针
char* inet_ntoa(struct in_addr inaddr);
```

上述函数或被废弃，或有更好的函数替换，`inet_pton`和`inet_ntop`就是随着`IPv6`出现的新函数，对于`IPv4`和`IPv6`地址都适用。

```c
#include <arpa/inet.h>
int inet_pton(int family,const char* strptr,void *addrptr); // 成功返回1，失败返回0
const char* inet_ntop(int family, const void* addrptr,char *strptr,size_t len);

/*
p：代表 Presentation 表达
n：代表 numeric 数值
inet_pton做从表达格式到数值格式的转化
inet_ntop做从数值格式到表达格式的转化（strptr必须事先分配好空间，len防止缓冲区溢出）
*/
```

`family`参数既可以是`AF_INET`也可以是`AF_INET6`。

例如：

```c
inet_pton(AF_INET, cp, &foo.sin_addr);
//上述语句等价于：
foo.sin_addr.s_addr = inet_addr(cp);

char str[len];
ptr = inet_ntop(AF_INET, &foo.sin_addr, str, sizeof(str));
// 上述语句等价于：
ptr = inet_ntoa(foo.sin_addr);
```

书中，对`inet_ntop`做了一层封装，调用者可以忽略其协议族：

> 对于书中的`sock_ntop`函数，传递的参数是套接字结构体，而`inet_pton`和`inet_ntop`则传递的是套接字IP地址结构体。

```c
#include  "unp.h"

#ifdef  HAVE_SOCKADDR_DL_STRUCT
#include  <net/if_dl.h>
#endif

/* include sock_ntop */
char *
sock_ntop(const struct sockaddr *sa, socklen_t salen)
{
    char    portstr[8];
    static char str[128];   /* Unix domain is largest */

  switch (sa->sa_family) {
  case AF_INET: {
    struct sockaddr_in  *sin = (struct sockaddr_in *) sa;

    if (inet_ntop(AF_INET, &sin->sin_addr, str, sizeof(str)) == NULL)
      return(NULL);
    if (ntohs(sin->sin_port) != 0) {
      snprintf(portstr, sizeof(portstr), ":%d", ntohs(sin->sin_port));
      strcat(str, portstr);
    }
    return(str);
  }
/* end sock_ntop */

#ifdef  IPV6
  case AF_INET6: {
    struct sockaddr_in6 *sin6 = (struct sockaddr_in6 *) sa;

    str[0] = '[';
    if (inet_ntop(AF_INET6, &sin6->sin6_addr, str + 1, sizeof(str) - 1) == NULL)
      return(NULL);
    if (ntohs(sin6->sin6_port) != 0) {
      snprintf(portstr, sizeof(portstr), "]:%d", ntohs(sin6->sin6_port));
      strcat(str, portstr);
      return(str);
    }
    return (str + 1);
  }
#endif

#ifdef  AF_UNIX
  case AF_UNIX: {
    struct sockaddr_un  *unp = (struct sockaddr_un *) sa;

      /* OK to have no pathname bound to the socket: happens on
         every connect() unless client calls bind() first. */
    if (unp->sun_path[0] == 0)
      strcpy(str, "(no pathname bound)");
    else
      snprintf(str, sizeof(str), "%s", unp->sun_path);
    return(str);
  }
#endif

#ifdef  HAVE_SOCKADDR_DL_STRUCT
  case AF_LINK: {
    struct sockaddr_dl  *sdl = (struct sockaddr_dl *) sa;

    if (sdl->sdl_nlen > 0)
      snprintf(str, sizeof(str), "%*s (index %d)",
           sdl->sdl_nlen, &sdl->sdl_data[0], sdl->sdl_index);
    else
      snprintf(str, sizeof(str), "AF_LINK, index=%d", sdl->sdl_index);
    return(str);
  }
#endif
  default:
    snprintf(str, sizeof(str), "sock_ntop: unknown AF_xxx: %d, len %d",
         sa->sa_family, salen);
    return(str);
  }
    return (NULL);
}

char *
Sock_ntop(const struct sockaddr *sa, socklen_t salen)
{
  char  *ptr;

  if ( (ptr = sock_ntop(sa, salen)) == NULL)
    err_sys("sock_ntop error"); /* inet_ntop() sets errno */
  return(ptr);
}

```

## I/O函数

我们经常使用是`read`和`write`，书中提到，我们请求的字节数往往比输入输出的字节数要多，原因在于缓冲区大小的限制，这样我们不得不多次调用`read`和`write`，于是作者为了方便期间又再一次的封装。 

* `readn`：从描述符中读n个字节。
* `wirten`：往描述符中写n个字节。
* `readline`：从描述符中读文本行一次一个字节。 

书中给出了这些函数的具体实现，在内部调用`read`和`write`。

### readn函数

```c
/* include readn */
#include  "unp.h"

ssize_t           /* Read "n" bytes from a descriptor. */
readn(int fd, void *vptr, size_t n)
{
  size_t  nleft;
  ssize_t nread;
  char  *ptr;

  ptr = vptr;
  nleft = n;
  while (nleft > 0) {
    if ( (nread = read(fd, ptr, nleft)) < 0) {
      if (errno == EINTR)
        nread = 0;    /* and call read() again */
      else
        return(-1);
    } else if (nread == 0)
      break;        /* EOF */

    nleft -= nread;
    ptr   += nread;
  }
  return(n - nleft);    /* return >= 0 */
}
/* end readn */

ssize_t
Readn(int fd, void *ptr, size_t nbytes)
{
  ssize_t   n;

  if ( (n = readn(fd, ptr, nbytes)) < 0)
    err_sys("readn error");
  return(n);
}

```

### wirten函数

```c
/* include writen */
#include  "unp.h"

ssize_t           /* Write "n" bytes to a descriptor. */
writen(int fd, const void *vptr, size_t n)
{
  size_t    nleft;
  ssize_t   nwritten;
  const char  *ptr;

  ptr = vptr;
  nleft = n;
  while (nleft > 0) {
    if ( (nwritten = write(fd, ptr, nleft)) <= 0) {
      if (nwritten < 0 && errno == EINTR)
        nwritten = 0;   /* and call write() again */
      else
        return(-1);     /* error */
    }

    nleft -= nwritten;
    ptr   += nwritten;
  }
  return(n);
}
/* end writen */

void
Writen(int fd, void *ptr, size_t nbytes)
{
  if (writen(fd, ptr, nbytes) != nbytes)
    err_sys("writen error");
}

```

### readline函数

```c
/* include readline */
#include  "unp.h"

static int  read_cnt;
static char *read_ptr;
static char read_buf[MAXLINE];

static ssize_t
my_read(int fd, char *ptr)
{

  if (read_cnt <= 0) {
again:
    if ( (read_cnt = read(fd, read_buf, sizeof(read_buf))) < 0) {
      if (errno == EINTR)
        goto again;
      return(-1);
    } else if (read_cnt == 0)
      return(0);
    read_ptr = read_buf;
  }

  read_cnt--;
  *ptr = *read_ptr++;
  return(1);
}

ssize_t
readline(int fd, void *vptr, size_t maxlen)
{
  ssize_t n, rc;
  char  c, *ptr;

  ptr = vptr;
  for (n = 1; n < maxlen; n++) {
    if ( (rc = my_read(fd, &c)) == 1) {
      *ptr++ = c;
      if (c == '\n')
        break;  /* newline is stored, like fgets() */
    } else if (rc == 0) {
      *ptr = 0;
      return(n - 1);  /* EOF, n - 1 bytes were read */
    } else
      return(-1);   /* error, errno set by read() */
  }

  *ptr = 0; /* null terminate like fgets() */
  return(n);
}

ssize_t
readlinebuf(void **vptrptr)
{
  if (read_cnt)
    *vptrptr = read_ptr;
  return(read_cnt);
}
/* end readline */

ssize_t
Readline(int fd, void *ptr, size_t maxlen)
{
  ssize_t   n;

  if ( (n = readline(fd, ptr, maxlen)) < 0)
    err_sys("readline error");
  return(n);
}

```
