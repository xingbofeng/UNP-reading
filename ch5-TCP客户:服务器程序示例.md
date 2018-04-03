# TCP客户/服务器程序示例
## 概述
本章完成一个以下的程序：

* 客户从标准输入读入一行文本，并写给服务器
* 服务器从网络输入中读入这行文本，并回射给客户
* 客户从网路输入读入这行回射文本，并显示在标准输入

![](http://oczira72b.bkt.clouddn.com/18-4-2/71895530.jpg)

需要用到的函数：

* 套接字编程基本函数(`socket`，`bind`，`listen`，`accept`，`connect`，`close`等)，完成套接字编程
* 标准`I / O`库函数`fputs`和`fgets`，完成输入和输出
* `read`，`writen`，`readline`函数，完成数据的传输
* `fork`函数，完成并行服务器的编写

## TCP回射服务器程序：main函数

```c
#include  "unp.h"

int
main(int argc, char **argv)
{
  int         listenfd, connfd;
  pid_t       childpid;
  socklen_t     clilen;
  struct sockaddr_in  cliaddr, servaddr;

  listenfd = Socket(AF_INET, SOCK_STREAM, 0); // 创建TCP套接字

  bzero(&servaddr, sizeof(servaddr));
  servaddr.sin_family      = AF_INET;
  servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
  servaddr.sin_port        = htons(SERV_PORT);

  Bind(listenfd, (SA *) &servaddr, sizeof(servaddr)); // 绑定到指定的IP和端口号

  Listen(listenfd, LISTENQ); // 把套接字转换为监听套接字

  for ( ; ; ) {
    clilen = sizeof(cliaddr);
    connfd = Accept(listenfd, (SA *) &cliaddr, &clilen);

    // 创建子进程来处理客户请求
    if ( (childpid = Fork()) == 0) {  /* child process */
      Close(listenfd);  /* close listening socket */
      str_echo(connfd); /* process the request */
      exit(0);
    }
    Close(connfd);      /* parent closes connected socket */
  }
}

```

## TCP回射服务器程序：str_echo回射函数

```c
#include  "unp.h"

void
str_echo(int sockfd)
{
  ssize_t   n;
  char    buf[MAXLINE];

again:
  while ( (n = read(sockfd, buf, MAXLINE)) > 0)
    Writen(sockfd, buf, n);

  if (n < 0 && errno == EINTR) // 如果暂时没有读取到数据，内核抛出EINTR异常，再次进入循环体
    goto again;
  else if (n < 0)
    err_sys("str_echo: read error"); // 如果不是EINTR异常，没有读到数据则抛出系统异常
}

```

## TCP回射客户程序：main函数

```c
#include  "unp.h"

int
main(int argc, char **argv)
{
  int         sockfd;
  struct sockaddr_in  servaddr;

  if (argc != 2)
    err_quit("usage: tcpcli <IPaddress>");

  sockfd = Socket(AF_INET, SOCK_STREAM, 0); // 创建客户端套接字

  bzero(&servaddr, sizeof(servaddr));
  servaddr.sin_family = AF_INET;
  servaddr.sin_port = htons(SERV_PORT);
  Inet_pton(AF_INET, argv[1], &servaddr.sin_addr); // 把对应的字符串表达式IP地址转换为网络传输的二进制类型IP

  Connect(sockfd, (SA *) &servaddr, sizeof(servaddr)); // 调用connect建立TCP连接

  str_cli(stdin, sockfd); // 发送客户端消息

  exit(0);
}

```

## TCP回射客户程序：str_cli函数

```c
#include  "unp.h"

void
str_cli(FILE *fp, int sockfd)
{
  char  sendline[MAXLINE], recvline[MAXLINE];

  while (Fgets(sendline, MAXLINE, fp) != NULL) { // 通过fgets读取一行文本，如果遇到EOF或者错误，fgets反回空指针，于是客户处理循环终止。

    Writen(sockfd, sendline, strlen(sendline)); // 通过writen把该行文本发送给服务器

    if (Readline(sockfd, recvline, MAXLINE) == 0) // readline从服务器读取回射行
      err_quit("str_cli: server terminated prematurely");

    Fputs(recvline, stdout); // fputs把读取的回射行写到标准输出
  }
}

```

## 正常启动


服务器正常启动后，它调用`socket`，`bind`，`listen`和`accept`，并阻塞于`accept`调用。让我们运行`netstat`程序来检查服务器监听套接字的状态。

![](http://oczira72b.bkt.clouddn.com/18-4-2/98890785.jpg)

这个正是该服务器建立起的套接字，`*:9877`表示一个为`0`的`IP`地址(`INADDR_ANY`，通配地址)，然后内核分配的`9877`端口号。此套接字处于`LISTEN`状态，即阻塞于`accept`调用，等待客户连接。

客户端依次调用`socket`，`connect`，后者引起`TCP`的三次握手过程，当三次握手完成后，客户中的`connect`和服务器的`accept`均返回，连接于是建立。接着发生以下步骤：

* 客户调用`str_cli`函数，该函数阻塞于`fgets`调用，然后等待客户输入一行文本
* 当服务器的`accept`返回时，服务器接着调用`fork`函数，再由子进程调用`str_echo`函数。该函数调用`readline`，`readline`调用`read`，而`read`在等待客户送入一行文本期间阻塞
* 另一方面，服务器父进程再次调用`accept`并阻塞，等待下一个客户连接。


此时的网络状态如下：

如上，列出了三个套接字状态；第一个对应于服务器子进程的套接字，本地端口号为`9877`；第二个对应于客户进程的套接字，它的本地端口号为`56858`；第三个为服务器父进程则继续监听客户。

![](http://oczira72b.bkt.clouddn.com/18-4-2/48257797.jpg)

## 正常终止

连接建立后，客户端程序就可以使用服务了，如图所示：

![](http://oczira72b.bkt.clouddn.com/18-4-2/51651092.jpg)

输入`ctrl + D`后，`fgets`返回空指针，于是，`str_cli`函数返回，紧接着客户端程序进入`main`函数，`main`函数也返回，客户端进程终止，进程终止处理的部分工作是关闭所有打开的描述符，因此客户打开的套接字由内核关闭。由于客户端进程结束，这将导致客户端`TCP`发起主动终止连接的`FIN`分节，服务器则以`ACK`回应。至此完成`TCP`终止连接的四次握手中的两次握手，客户端套接字处于`FIN_WAIT_2`状态，服务器套接字进入`CLOSE_WAIT`状态，客户与服务的状态转移这部分对照那个经典的状态转移图走一遍过程就懂了，注意是哪一方主动发起关闭连接的操作。

紧接着，由于服务器收到`FIN`分节，在子进程中阻塞的`read`函数将返回，于是`str_echo`返回，整个子进程返回终止，子进程中打开的描述符随之关闭，于是服务器发送一个`FIN`分节，并等待客户端的`ACK`，这就是四次握手的后两部分了。

连接正常终止，但这里存在潜在的问题：

> 服务器子进程终止时会给父进程发送一个`SIGCHLD`信号，我们在代码中没有捕获该信号，该信号就被忽略了，于是子进程就编程了僵尸进程，在终端输入`ps`：

![](http://oczira72b.bkt.clouddn.com/18-4-2/22275922.jpg)

这里服务端子进程的`pid`为`66406`，发现它的状态为`Z+`，这里表示它为僵死进程。

![](http://oczira72b.bkt.clouddn.com/18-4-2/70443023.jpg)

## POSIX信号处理
信号是告知某个进程发生了某个事件的通知，有时也叫做软件中断。信号通常时异步发生的。

信号可以：

* 由一个进程发给另一个进程
* 由内核发给某个进程

每个信号都有一个与之关联的处置，或者说行为。我们通过调用`sigaction`函数来设定一个信号的处置，通常有如下三种选择：

* 我们可以提供一个函数，只要有特定信号发生它就会被调用。通过调用信号处理函数（`handler`），用来捕获特定的信号。有两个信号`SIGKILL`和`SIGSTOP`不能被捕获。对于大多数信号来说，调用`sigaction`函数并制定信号发生时所调用的函数就是捕获信号所需做的全部工作。不过对于`SIGIO`、`SIGPOLL`和`SIGURG`这些个别信号还要求捕获它们的进程做些额外工作。

`handler`函数原型：

```c
void handler(int signo);
```

* 通过设置某个信号为`SIG_IGN`来忽略它。`SIGKILL`和`SIGSTOP`不能被忽略。
* 通过设置某个信号为`SIG_DFL`来启用它的默认处置。默认处置通常是在收到信号后终止进程，其中某些信号还在当前工作目录产生一个进程的核心镜像。另有个别信号的默认处置是忽略，`SIGCHLD`和`SIGURG`就是默认处置为忽略。

### signal函数
因为`SIGCHLD`信号的默认处置是忽略信号，因此会产生僵死的子进程，解决方案是建立信号处置的`POSIX`方法，即调用`sigaction`函数。不过这有点复杂，因为该函数的参数之一是我们必须分配并填写的结构。简单些的方法是调用`signal`函数，其第一个参数是信号名，第二个参数是指向函数的指针。

以下是调用`POSIX`的`sigaction`函数的`signal`函数原型：

```c
/* include signal */
#include  "unp.h"

Sigfunc *
signal(int signo, Sigfunc *func)
{
  struct sigaction  act, oact;

  act.sa_handler = func;
  sigemptyset(&act.sa_mask);
  act.sa_flags = 0;
  if (signo == SIGALRM) {
#ifdef  SA_INTERRUPT
    act.sa_flags |= SA_INTERRUPT; /* SunOS 4.x */
#endif
  } else {
#ifdef  SA_RESTART
    act.sa_flags |= SA_RESTART;   /* SVR4, 44BSD */
#endif
  }
  if (sigaction(signo, &act, &oact) < 0)
    return(SIG_ERR);
  return(oact.sa_handler);
}
/* end signal */

Sigfunc *
Signal(int signo, Sigfunc *func)  /* for our signal() function */
{
  Sigfunc *sigfunc;

  if ( (sigfunc = signal(signo, func)) == SIG_ERR)
    err_sys("signal error");
  return(sigfunc);
}

```

下面就谈谈代码中的语法细节，书中用了包裹函数，在之前书中提到`signal`函数的原型为：

```c
void (*signal(int signo,void (*func)(int)))(int);
```

可以如下简化它的输入，使得代码更易读：

```c
typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);
```

从上可知：

* `signal`函数有两个参数，其一是`int`行，其二是一个函数指针型（`int`为参数，`void`为返回值）
* `signal`函数的返回值也是一个函数指针（`int`为参数，`void`为返回值）

### POSIX信号语义
符合`POSIX`的系统上的信号处理总结为一下几点：

* 一旦安装了信号处理函数，它便一直安装着。
* 在一个信号处理函数运行期间，正被递交的信号时阻塞的，而且，安装处理函数时在传递给`sigaction`函数的`sa_mask`信号集中指定的任何额外信号也被阻塞。
* 如果一个信号在被阻塞期间产生了一次或者多次，那么该信号被解阻塞之后通常只递交一次。`Unix`信号默认是不排队的。

## 处理SIGCHLD信号
设置僵死状态的目的是维护子进程的信息，以便父进程在以后某个时候获取，这些信息包括子进程的进程`ID`，终止状态以及资源利用信息(`CPU`时间，内存使用量等等)。

如果一个进程终止，而该进程有子进程处于僵死状态，那么它的所有僵死子进程的父进程`ID`都被重置为`1`(`init`进程)，继承这些子进程的`init`进程将清理他们。

### 处理僵死进程
留存僵死进程会占用内核中的空间，最终可能导致我们耗尽进程资源。无论何时，我们`fork`子进程都得`wait`它们！于是我们必须调用信号处理函数(在`LISTEN`之后添加如下函数)，并在函数体内调用`wait`。

```c
#include  "unp.h"

void
sig_chld(int signo)
{
  pid_t pid;
  int   stat;

  pid = wait(&stat);
  printf("child %d terminated\n", pid);
  return;
}

```

修改服务端程序为：

```c
#include  "unp.h"

int
main(int argc, char **argv)
{
  int         listenfd, connfd;
  pid_t       childpid;
  socklen_t     clilen;
  struct sockaddr_in  cliaddr, servaddr;
  void        sig_chld(int);

  listenfd = Socket(AF_INET, SOCK_STREAM, 0);

  bzero(&servaddr, sizeof(servaddr));
  servaddr.sin_family      = AF_INET;
  servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
  servaddr.sin_port        = htons(SERV_PORT);

  Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));

  Listen(listenfd, LISTENQ);

  Signal(SIGCHLD, sig_chld); // 增加signal，对于SIGCHLD信号捕获则调用sig_chld信号处理函数

  for ( ; ; ) {
    clilen = sizeof(cliaddr);
    connfd = Accept(listenfd, (SA *) &cliaddr, &clilen);

    if ( (childpid = Fork()) == 0) {  /* child process */
      Close(listenfd);  /* close listening socket */
      str_echo(connfd); /* process the request */
      exit(0);
    }
    Close(connfd);      /* parent closes connected socket */
  }
}

```

运行以上程序得到输出：

![](http://oczira72b.bkt.clouddn.com/18-4-2/89628438.jpg)

### 处理被中断的系统调用

具体步骤如下：

1. 当键入`EOF`字符来终止客户，客户`TCP`发送一个`FIN`给服务器，服务器响应以一个`ACK`
2. 收到客户的`FIN`导致服务器`TCP`递送一个`EOF`给子进程阻塞中的`readline`，从而子进程终止
3. 当`SIGCHLD`信号递交时，父进程阻塞于`accept`函数，`sig_chld`函数执行，其`wait`调用取到子进程的`PID`和终止状态，随后`printf`调用，最后返回
4. 该信号在父进程阻塞于慢系统调用`accept`时由父进程捕获的，内核就会使`accept`返回一个`EINTR`错误(被中断的系统调用)，而父进程不出理该错误(源代码中未处理该错误)，于是中止。

慢系统调用适用于那些可能永远阻塞的系统调用。永远阻塞的系统调用是指调用永远无法返回，多数网络支持函数都属于这一类。如`accept`函数，如果没有客户连接到服务器，那么服务器的`accept`调用就没有返回；再比如`read`函数，如果客户从未发送过一行要求服务器回射的文本，那么服务器的`read`调用将永远不会返回，等等。

慢系统调用可以被永久阻塞，包括一下几个类别：

1. 读写慢设备（包括`pipe`，终端设备，网络连接等）。读时，数据不存在，需要等待；写时，缓冲区满或其他原因，需要等待。读写磁盘文件一般不会阻塞。
2. 当打开某些特殊文件时，需要等待某些条件，才能打开。例如：打开中断设备时，需要等到连接设备的`modem`响应才能完成。
3. `pause`和`wait`函数。`pause`函数使调用进程睡眠，直到捕获到一个信号。`wait`等待子进程终止。
4. 某些`ioctl`操作。
5. 某些`IPC`操作。

适用于慢系统调用的基本规则：当阻塞于某个慢系统调用的一个进程捕获某个信号且相应信号处理函数返回时，该系统调用可能返回一个`EINTR`错误。例如：

```c
for(;;){
    clilen  = sizeof(cliaddr);
    if((connfd = accept(listenfd,(SA*)&addcli,&clilen))<0){
        if(errno == EINTR){//返回一个EINTR错误，重启被中断的系统调用
            continue;//继续for循环
        }
        else
            err_sys("accepted error");
    }
}
```

对于`accept`以及诸如`read`，`write`，`select`和`open`之类的函数来说，可以重启被中断的系统调用。

不过，`connect`函数不能重启，如果该函数返回`EINTR`，我们不能再次调用它，否则将立即返回一个错误。当`connect`被一个捕获的信号中断而且不自动重启时，我们必须调用`select`来等待连接完成。

## wait和waitpid函数
函数声明如下：

```c
#include <sys/wait.h>
pid_t wait(int *statloc);
pid_t waitpid(pid_t pid,int *statloc,int options);
```

两者均返回子进程的进程号，通过`statloc`指针得到终止状态，既然说到两个函数，就必须得说说两者的区别了。 

书中就举了一个很好的例子： 

1. 假设客户端同时向服务器建立了`5`个连接 
2. 然后客户进程关闭，引发其每个连接套接字主动发出`5`个`FIN`。 
3. 服务器子进程收到`FIN`返回终止，内核又向父进程发送`5`份`SIGCHLD`信号 
4. 由于`UNIX`信号是不排队同时有好几个子进程结束只留下一个信号，并且是异步的，我们不知道信号何时到来，假设现在子进程`A`引发的`SIGCHLD`被捕获并进入信号处理函数的时候，子进程`B`和`C`引发的`SIGCHLD`刚好来到，那么这个`SIGCHLD`在下次被捕获的时候，只处理其中一个进程（假设是`C`）。这样子进程`B`的僵尸进程就没办法处理了。

因此由`wait`到`waitpid`的改良的最大需求就是解决上述`4`提到的问题：

书中给出一个示例：


![](http://oczira72b.bkt.clouddn.com/18-4-2/6683408.jpg)

![](http://oczira72b.bkt.clouddn.com/18-4-2/66462395.jpg)

所以，在书中给的例子中，`5`个连接对应`5`个服务器程序的子进程，只有`1`个僵死进程被杀死，其他`4`个没办法处理。

### wait与waitpid的区别

要解决上述问题，就要谈谈这两个函数的区别了。

从参数上来看，`waitpid`除了可以指定监视某一特定子进程（根据`pid`号，也可以设置为`-1`表示等待第一个终止的子进程），还可以设置`options`选项，最常用的是`WNOHANG`（也可以设置`0`，表示不使用任何选项。） 

有一种说法是，`wait`是`waitpid`的某一特定包裹函数：

```c
pid_t wait(int *statloc)
{
    return waitpid(-1, statloc, 0);
}
```

对于书上的问题，使用`waitpid`代替`wait`即可解决，使用`waitpid`并制定第三个参数为`WNOHANG`

基于上述特性，我们可以在信号处理函数中写一个`while`循环：

```c
while ( (pid = waitpid(-1,&stat,WNOHANG)) > 0)
    printf("child terminated\r\n");
```

这样设想`2`种情况：

1. 当正在信号处理函数中时，并没有子进程终止，这样`waitpid`在清理完该子进程后立刻返回。 
2. 当正在信号处理函数中时，有若干个子进程终止，内核将发出一个`SIGCHLD`信号（不排队，只有一个），此时由于主进程进入信号处理函数（中断），通过`while`循环一次性收集所有僵尸子进程，就算在这个过程中，又不断有子进程终止，内核依然产生一个`SIGCHLD`信号，并在下次进入中断时一次性收集，如此循环。

这样我们就没办法用`wait`了，如果有`5`个进程，第一次终止两个，内核发送一个`SIGCHLD`，捕获到了之后，进入处理函数，`wait`清理掉其中一个；第二次终止三个，内核发送一个`SIGCHLD`，`wait`又清理掉其中一个。这样会留下三个僵尸进程。

> 那么为什么不能在`wait`外面也加一层`while`呢？ 

因为`wait`是阻塞的，在没有子进程终止时他是阻塞的，那么当第一次处理完两个之后，当另外三个子进程还没结束时，主进程就会阻塞在信号处理函数中了。

总结起来就是： 

`wait`只能是阻塞调用，`waitpid`可以选择是非阻塞的，当出现很多子进程时，为了保证可以收集处理掉子进程的僵尸进程，应该使用`waitpid`。

> 并发服务器可能遇到的三种情况：

* 当`fork`子进程时，必须捕获`SIGCHLD`信号。
* 当捕获信号时，必须处理中断的系统调用。
* `SIGCHLD`的信号处理函数必须正确编写，应使用`waitpid`函数以免留下僵死进程。

完整的客户端程序如下：

```c
#include  "unp.h"

int
main(int argc, char **argv)
{
  int         listenfd, connfd;
  pid_t       childpid;
  socklen_t     clilen;
  struct sockaddr_in  cliaddr, servaddr;
  void        sig_chld(int);

  listenfd = Socket(AF_INET, SOCK_STREAM, 0);

  bzero(&servaddr, sizeof(servaddr));
  servaddr.sin_family      = AF_INET;
  servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
  servaddr.sin_port        = htons(SERV_PORT);

  Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));

  Listen(listenfd, LISTENQ);

  Signal(SIGCHLD, sig_chld);  /* must call waitpid() */

  for ( ; ; ) {
    clilen = sizeof(cliaddr);
    if ( (connfd = accept(listenfd, (SA *) &cliaddr, &clilen)) < 0) {
      if (errno == EINTR)
        continue;   /* back to for() */
      else
        err_sys("accept error");
    }

    if ( (childpid = Fork()) == 0) {  /* child process */
      Close(listenfd);  /* close listening socket */
      str_echo(connfd); /* process the request */
      exit(0);
    }
    Close(connfd);      /* parent closes connected socket */
  }
}

```

## accept返回前连接中止

除了被中断的系统调用的例子，另有一种情形能够导致`accept`返回一个非致命的错误，在这种情况下，只需要再次调用`accept`。

这里，三路握手完成从而建立连接之后，客户`TCP`却发送了一个`RST`（复位）。在服务器端看来，就在该连接已由`TCP`排队，等着服务器进程调用`accept`的时候`RST`到达。稍后，服务器进程调用`accept`。模拟这种情况也非常简单：启动服务器，调用`socket`，`bind`，`listen`，然后在调用`accept`之前睡眠一段时间。在服务器进程睡眠时，启动客户，让它调用`socket`和`connect`，一旦`connect`返回，就设置`SO_LINGER`套接字选项以产生这个`RST`，然后终止。

![](http://oczira72b.bkt.clouddn.com/18-4-2/18081626.jpg)

这种情况一般被忽略，直接再次调用`accept`即可，之后我们将再次回到这些中止的连接，查看在与`select`函数和正常阻塞模式下的监听套接字组合时他们是如何成为问题的。

## 服务器进程终止
在服务器启动以后，客户也正常启动，然后在客户端输入一行，在这个时候杀死服务器子进程，看有什么效果。

1. 在同一主机上启动服务器和客户，并在客户上键入一行文本，以验证一切正常。正常情况下文本由服务器子进程回射给客户。
2. 找到服务器子进程的进程`ID`，并执行`kill`命令杀死它，作为进程终止处理的部分工作，子进程中所有打开着的描述符都被关闭。这就导致向客户发送一个`FIN`，而客户`TCP`则响应以一个`ACK`。这就是`TCP`连接终止工作的前半部分。
3. `SIGCHLD`信号被发送给服务器父进程，并得到正确处理。
4. 客户上没有发送任何特殊之事。客户`TCP`接受来自服务器`TCP`的`FIN`并响应以一个`ACK`，然而问题是客户进程阻塞在`fgets`上，等到从终端接受一行文本。
5. 此时，在另外一个窗口上运行`netstat`命令，以观察套接字的状态。可以发现有两个相关的进程（子进程被kill）,其中一个进程处于`LISTEN`状态，一个处于FIN_WAIT2状态，一个处于`CLOSE_WAIT`状态，可以看到TCP连接终止序列的前半部分已经完成。
6. 我们可以在客户上再键入一行文本”another line“字符串时，`str_cli`调用`write`，客户`TCP`接着把数据发送给服务器、`TCP`允许这么做，因为客户`TCP`接受到`FIN`只是表示服务器进程已关闭了链接的服务器端，从而不再往其中发送任何数据而已、`FIN`的接受并没有告知客户`TCP`服务器进程已经终止（在这里确实终止了）在之后讨论`TCP`的半关闭时我们再讨论这一点。
7. 当服务器`TCP`接受到来自客户的数据时，既然先前打开那个套接字的进程已经终止，于是响应一个`RST`。通过使用`tcpdump`来观察分组，RST确实发送了。然而客户进程看不到这个`RST`，因为它在调用`write`后立即调用`read`，并且由于又接受到了客户的`FIN`，所调用的`read`立即返回`0`，我们的客户此时并为期望收到`EOF`。
8. 当客户终止时，它所打开的描述符都被关闭。

本例子的问题在于，当`FIN`到达客户端，而是应该阻塞在其中任何一个套接字时，客户正阻塞在`fgets`调用上。客户实际上再应对两个描述符------套接字和用户输入，它不能单纯阻塞在这两个源中某个特定源的输入上（正如目前编写的`str_cli`函数所为），而是应该阻塞在其中任何一个源的输入上，事实上这正是`select`和`poll`这两个函数的目的之一，在重写的`stl_cli`函数之后，一旦杀死服务器子进程，客户就会立即被告知已收到`FIN`。（非阻塞`I / O`）

![](http://oczira72b.bkt.clouddn.com/18-4-3/77431812.jpg)

## SIGPIPE信号
当客户不理会`readline`函数返回的错误，反而写入更多的数据到服务器上，如客户可能读回任何数据之前执行两次针对服务器的写操作，而`RST`是由其中一个写操作引发的。此时会发生什么？

当一个进程向某个已收到`RST`的套接字执行写操作时，内核向该进程发送一个`SIGPIPE`信号。该信号的默认行为是终止进程。

不管该进程时捕获了该信号并从信号处理函数返回，还是简单的忽略，第二次写操作都将返回`EPIPE`错误。

测试代码：

```c
void str_cli(FILE *fp,int sockfd)
{
    char sendline[MAXLINE], recvline[MAXLINE];
    //获取客户端标准输入的字节
    while(Fgets(sendline,MAXLINE,fp)!=NULL)
    {
        Write(sockfd,sendline,1);
        //将标准输入读入的字节传给服务器
        sleep(1);
        Write(sockfd,sendline+1,strlen(sendline)-1);//连续写两次来测试是否产生SIGPIPE信号
        //读取服务器回射的字节
        if(Readline(sockfd,recvline,MAXLINE)==0)
        {
            err_quit("strcli:server terminated prematurely");
        }
        //将获取的服务器回射字节打印到标准输出
        Fputs(recvline,stdout);
    }
}
```

## 服务器主机崩溃
服务器主机崩溃的情况下，客户端会发生什么？

当服务器主机崩溃(注意不是此处不是服务器主机关机)，客户端并不知情，所有客户端端连续发送数据，试图从服务器端接受一个`ACK`。

这时，可以看出`TCP`的重传机制，客户端总共重传该数据`12`次，共等待大约`9`分钟才放弃重传。当客户`TCP`最后终于放弃时，服务器主机并没有重新启动，或者如果时服务器主机未崩溃但是从网络上不可达，则返回`ETIMEOUT`错误；然而如果时中间某个路由器判定服务器主机不可达，从而响应一个“destination unreachable”目的地不可达`ICMP`信息，则返回的错误时`EHOSTUNREACH`或者`ENETUNREACH`。

## 服务器主机崩溃后重启
当服务器崩溃后重启(`9`分钟内)，它的`TCP`丢失了崩溃前的所有连接信息，因此服务器`TCP`对于所收到的来自客户的数据分节响应一个`RST`。

当客户`TCP`收到该`RST`时，客户正阻塞与`readline`调用，导致该调用返回`ECONNRESET`错误。

## 服务器关机
`Unix`系统关机时，`init`进程通常先给所有进程发送`SIGTERM`信号，等待一段时间后，给所有仍在运行的进程发送`SIGKILL`信号，这么做留给所有运行的进程一小段时间来清除和终止。

当服务器子进程终止时，它所有打开着的描述符都被关闭，随后发生和服务器主机崩溃一样的效果。

## 客户和服务器传送的数据类型
在客户和服务器间传送文本串，不论客户和服务器主机的字节序如何，都能很好的工作。

在客户和服务器间传送二进制结构，如下结构体：

```c
struct args{
  long arg1;
  long arg2;
};

struct result{
  long sum;
};
```

`socket`不提倡传送这种二进制结构，因为如果客户或者服务器端某一方不支持这种数据结构，或字节序不一样，支持的`long`的字节不一样，这都会导致异常的现象。

所以在`socket`传输时，应当把所有的二进制结构转换成字符型文本串，再进行传输。

或者，还有一种解决办法：显示的定义所支持的数据类型的二进制格式(位数，大端或小端字节序)，并以这样的格式在客户与服务器之间传递所有数据。