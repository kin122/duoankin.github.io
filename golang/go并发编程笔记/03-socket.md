socket 套接字，IPC方法的一种，通过网络连接让多个进程建立通信并相互传递数据，使得通信双方是否在同一台服务器变得无关紧要，使得通信端的位置透明化。  
### socket的定义
linux中存在的socket调用  
`int socket(int domain,int type,int protocol)`  
三个参数分别代表通信域、类型和所用协议：
1. 通信域分：AF_INET(IPv4),AF_INET6(IPv6),AF_UNIX(UNIX域)三个域，*AF是address family的缩写*，IPv4和IPv6是网络范围内通信，unix是单台计算机范围
2. 类型分为两大类：数据报（SOCK_DGRAM,SOCK_RAW）和字节流（SOCK_SEQPACKET,SOCK_STREAM）  
  2.1. 以数据报为数据形式意味着数据接收方的socket接口程序可以意识到数据的边界并会对它们进行切分，这样就省去了接收方的应用程序寻找数据边界和切分数据的工作量。但是数据报没有数据连接，也不能保证数据的有序性，以及不具备数据传输可靠性.  
  2.2. 以字节流为数据形式的数据传输实际上传输的是一个字节接着一个字节的串，可以把它想象成一个很长的字节数组，这样能保证数据逻辑的连接性有序性和可靠性。但是也因此，socket接口程序无法从中分离出独立的数据包，只能靠应用程序完成.  
  2.3. 但是SOCK_SEQPACKET类型的socket接口程序是个例外，可以记录数据边界。  

socket通讯有无连接：
* 有连接：有链接的socket之间一旦建立连接，发送的数据就是定向的，连接可以方便地互相传输数据
* 无连接：数据包需要包含目的地址，可以传输给不同的地址；数据流只能单向，不能使用同一个面向无连接的socket实例即发送数据又接收数据
* 数据传输的有序性和可靠性与socket是否面向连接有很大的关系
* sock_raw类型的socket提供了一个可以直接通过底层传输数据得方法。为了保证安全性，应用程序必须具有操作系统的超级用户权限才能使用，此方法成本也较高，一般很少使用。

系统调用socket的时候，一般以0作为它的第三个参数，也就是说socket的所用协议以通信域和类型默认决定，一般通信域为net的会根据类型决定调用那种网络协议，通信域为unix的则是决定是否有效（*有效则表示系统内核使用内部的socket协议*）。  
* 在网络协议中，各个类型对应的协议关系如下：SOCK_DGRAM-UDP;SOCK_RAM-IPv4/IPv6;SOCK_SEQPACKET-SCTP;SOCK_STREAM-TCP/SCTP
* unix通信域中，除了SOCK_RAW是无效外都有效

### socket的调用
socket方法的返回值是socket实例唯一标识符的文件描述符，一旦得到对应的标识符，就可以调用其他系统调用来进行各种相关操作，比如绑定和监听端口、发送或者接收数据等等。  
socket接口和TCP/IP协议栈一样是linux系统内核的一部分。  
在程序使用中，socket的调用链如下：应用层的应用程序调用socket接口，socket调用传输层的tcp/udp/sctp的协议，再调用网络互联层的IP  

### golang实现socket
此处主要实现了基于TCP的客户端和服务端通过socket接口建立tcp连接并进行通信的情形。
![基于TCP协议栈的socket通信流程](https://github.com/kin122/duoankin.github.io/blob/main/golang/go%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E7%AC%94%E8%AE%B0/%E5%9F%BA%E4%BA%8ETCP%E5%8D%8F%E8%AE%AE%E6%A0%88%E7%9A%84socket%E9%80%9A%E4%BF%A1%E6%B5%81%E7%A8%8B.webp)  

为了实现服务端和客户端程序，需要使用标准库代码包net中的API  
##### net相关函数
`func Listen(net,laddr string)(Listerner,error)`  
* 第一个参数代表的是监听地址的协议，这个参数必须是面向流的协议，因此只能是（tcp/tcp4/tcp6/unix/unixpacket中的一个，*udp则使用ListenUDP解决*）
* 第二个参数则是local address，格式为host:port

`conn,err:=Listener.Accept()`  
调用Accept方法时，流程会被阻塞，知道某个客户端程序与当前程序建立tcp连接。Accept方法返回两个结果值，第一个代表当前TCP连接的net.Conn类型，第二个依旧是error

`func Dial(network,address string)(Conn,error)` 
Dial函数用于向指定的网络地址发送链接建立申请，network参数和net.Listen函数的第一个参数net的含义类似，但是比后者拥有更多的可选值。第二个参数就是IP:PORT，只是此处是远程地址。Dial函数用于客户端连接建立。其代码类似于`conn,err:=net.Dial("tcp","127.0.0.1:8085")`    

`func DialTimeout(network,address string,timeout time.Duration)(Conn,error)`  
DialTimeout是专门针对调整网络超时时间设置的方法，time.Duration类型可以这样设置：`conn,err=net.DailTimeout("tcp","127.0.0.1:8085",2*time.Second)`  

尽管socket相关的API使用中会有阻塞式的特性，但是在底层socket接口中使用的是非阻塞式的处理方法，有着部分读部分写的特性。**具体底层逻辑待解读。**  

