# TCP、UDP、SCTP
## 用户数据报协议（UDP）
UDP是一个简单的传输层协议。UDP不保证UDP数据报会到达其最终目的地，不保证各个数据报的先后顺序跨网络后保持不变，也不保证每个数据报只到达一次。

每个UDP数据报都有一个长度。

我们也说UDP提供无连接的（connectionless）服务，因为UDP客户和服务器之间不必存在任何长期的关系。举例来说，一个UDP客户可以创建一个套接字并发送一个数据报给一个给定的服务器，然后立即用同一个套接字发送另一个数据报给另一个服务器。同样的，一个UDP服务器也可以用同一个UDP套接字从若干个不同的客户端接收数据报，每个客户一个数据报。

## 传输控制协议（TCP）
TCP提供客户和服务器之间的连接（connection）。TCP客户先与某个给定服务器建立一个连接，再跨该连接与那个服务器交换数据，然后终止这个连接。

TCP不能被描述成是100%可靠的协议，它提供的是数据的可靠递送或故障的可靠通知。

TCP含有用于动态估算客户和服务器之间的往返时间（round-trip time, RTT）的算法，以便它知道等待一个确认需要多少时间。

TCP通过给其中每个字节关联一个序列号对所发送的数据进行排序（sequencing）。

TCP提供流量控制（flow control）。TCP总是告知对端在任何时刻它一次能够从对端接收多少字节的数据，这称为通告窗口（advertised window）。在任何时刻，该窗口指出接收缓冲区中当前可用的空间量，从而确保发送端发送的数据不会使接收缓冲区溢出。该窗口时刻动态变化。

TCP连接是全双工的（full-duplex）。建立一个全双工连接后，需要的话可以把它转换为一个单工连接。

## 流传输控制协议（SCTP）
SCTP在客户和服务器之间提供关联（association）。并像TCP那样给应用提供可靠性、排序、流量控制以及全双工的数据传送。SCTP支持多宿而涉及不止两个地址。

与TCP不同的是，SCTP是面向消息的（message-oriented）。它提供各个记录的按序递送服务，与UDP一样，由发送端写入的每条记录的长度随数据一道传递给接收端应用。

SCTP能够在所连接的断点之间提供多个流，每个流各自可靠地按序递送消息。一个流上某个消息的丢失不会阻塞同一关联其他流上消息的投递。

SCTP还提供多宿特性，使得单个SCTP端点能够支持多个IP地址。该特性可以增强应对网络故障的健壮性。

## 三路握手
![](http://oczira72b.bkt.clouddn.com/18-3-18/57888140.jpg)

## TCP 选项
* MSS选项 发送SYN的TCP一端使用本选项通告对端它的最大分节大小（maximum segment size）即MSS，也就是它在本连接的每个TCP分节中愿意接受的最大数据量。
* 窗口规模选项
* 时间戳选项 作为网络编程选项，我们无需考虑这个选项。

## TCP端口号与并发服务器
并发服务器中主服务器循环通过派生一个子进程来处理每个新的连接。

![](http://oczira72b.bkt.clouddn.com/18-3-18/41942019.jpg)

对于并发服务器，TCP监听套接字fork自身进程创建子进程来处理请求。监听套接字根据源地址+源端口把对应请求转发给对应子进程。


## TCP连接终止
![](http://oczira72b.bkt.clouddn.com/18-3-18/28829119.jpg)

当一个Unix进程无论自愿地（调用exit或从main函数返回）还是非自愿地（收到一个终止本进程的信号）终止时，所有打开的描述符都被关闭，这也导致仍然打开的任何TCP连接上也发出一个FIN。

## TCP状态转换图
![](http://oczira72b.bkt.clouddn.com/18-3-18/73167570.jpg)

## 观察分组

![](http://oczira72b.bkt.clouddn.com/18-3-18/93090529.jpg)

注意，服务器对客户请求的确认是伴随其应答发送的，这种做法称为捎带（piggybacking），它通常在服务器处理请求并产生应答的时间少于200ms时发生。如果服务器好用更长时间，譬如1s，那么我们将看到先是确认后是应答。

许多网络应用仍然使用UDP构建的，因为它们需要交换的数据量较少，而UDP避免了TCP连接建立和终止所需的开销。

## 缓冲区大小及限制
*IPv4数据总长度最大为65535字节，IPv4首部为20字节的固定部分和40字节的可选项。IPv4要求最小链路MTU为68字节。IPv4的主机和路由器可对其产生以及转发的数据包分片，若首部DF位被设置则不可分片
* IPv6数据报净荷长度最大为65535字节，首部长度为40字节，所以总长度最大为65575字节。有特大净荷选项，可以把净荷大小字段扩展到32位，但是需要MTU支持。IPv6要求最小链路MTU为1280字节（可以运行在MTU小于1280字节的链路上，利用分片和重组）。IPv6主机和路由器只可对其产生的数据包分片。
* MTU（maximum tramission unit,最大传输单元）是数据链路对传输的数据报大小限制，以太网MTU为1500字节。
* 最小重组缓冲区大小，它是IPv4和IPv6都必须支持的最小数据报大小，对于IPv4是576字节， 对于IPv6是1500字节
* MSS（maximum segment size,最大分节大小），TCP用MSS向对端通告对端在每个分节中能发送的最大TCP数据量，MSS经常设置为MTU减去IP和TCP首部的固定长度，在以太网中使用IPv4的MSS为1460，使用IPv6的MSS为1440（两者TCP首部为20字节，但IPv4首部是20字节，IPv6首部是40字节）