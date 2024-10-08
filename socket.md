posix-api与网络协议栈

socket包含两部分：fd、tcb（全称:tcp control block）

其中，fd属于文件系统，可在用户态进行操控；而tcb属于内核协议栈。

三次握手

服务端API
socekt()：创建一个tcb和fd（会将socketfd转换为listenfd）
fd最小为3，因为0、1、2号fd分别被stdin、stdout、stderr占了
一个连接包括客户端socket和服务端socket，每个socket包括一个fd和一个tcb（服务端的ip、端口和信息，客户端每建立一个连接，就新建一个connfd，并根据socket结构体创建一个tcb，将客户端的ip和端口等信息存储在tcb中）
bind(): 为tcb绑定ip和端口
两个特殊ip：0.0.0.0和127.0.0.1
绑定0.0.0.0：socekt绑定多张网卡 (一张网卡一个ip，一个机器可能有多个网卡)
绑定127.0.0.1：本地ip才能连接
listen()：指定全连接队列（或全连接+半连接队列的长度），把tcb 置为 listen 的状态。当客户端发送SYN包准备建立连接，服务端会init/add new tcb作为一个新的节点push in syn队列，此时服务端的状态由LISTEN转为SYN_RCVD, 并且服务端会发送SYN + ACK给客户端
accept()：当收到客户端的ACK后，此时会把syn队列中的tcb节点pop出来，再push in accept队列中，服务端状态由SYN_RCVD转为ESTABLISHED，三次握手完成。后续数据传输会从accept队列中取出tcb节点进行数据传输。并且accept为每个节点分配一个socket(conn_fd)
recv/read(): 从读缓冲区读数据
send/write(): 往写缓冲区写数据
close(): 将FIN置为1，发送空包给对方请求关闭连接
客户端API
socket()：创建client_fd
bind()： 可选，非必须
connect()：发起三次握手，若client_fd是阻塞的，则等待连接完成后返回；若client_fd是非阻塞的，则立即返回，如果连接成功，则返回可读/可写结果
recv/read(): 从读缓冲区读数据
send/write(): 往写缓冲区写数据
close(): 将FIN置为1，发送空包给对方请求关闭连接。

小总结：

socket和bind()都是本地操作
listen()和accept()参与了三次握手，全连接和半连接队列示意图：

服务器的tcb通过bind()绑定本地(ip+port)、通过connect知道客户端(ip+port)
send相关问题
首先需注意不能通过send()>0来判断数据发送方成功了，因为send()只负责写协议栈，不负责发送数据。

无法send的情况
    本地sendbuff满了
    对方recvbuff空间不足，无法接收发送端的数据，此时内核协议栈会保留一个push的标志位，当有足够空间时，推给发送方进行发送
多次send时的粘包问题
比如我一次向sendbuff写1k的数据，共发送三次，但协议栈每次发送1.5k的数据，两次就发完了，就导致发送方发送了三次，但接收方只接收了两次，产生粘包现象

解决方法：

包的协议头加包长度
包尾加分隔符，如http就是加\r\n\r\n作为包结束标志
然而，这两种方法实现的前提都要求包是顺序到达的，就是说不能有中间包丢失，对此，tcp有一种延迟ack的超时重传机制：比如有序列号为1，2，3，4，5的5个包需要发送，接收端设置一个定时器（如设置200ms），每接收到一个包，就将定时器重置，当定时器到期后，发送ACK消息。

接收端：2包到达，重置定时器；1包到达，重置定时器；5包到达，重置定时器；3包到达，重复定时器；然后定时器过期，发送对2包的确认，也就是需要重传3，4，5包（乱序到达，接收方有重排机制恢复原有顺序）

这种延迟ack的方法有两个不好的点：

实时性差：确认事件长，所以游戏都用UDP
重传多，占用宽带：需要重传丢包以后的后续包，所以下载用UDP快
不过这种延迟ACK是可以关闭的，也有算法对次提出了改进，利用NACK只需要重传丢失的包即可
四次挥手
挥手只分主动方和被动方


五个状态：

主动方调用close()，发送一个fin为1的空包，然后进入fin_wait 1状态
被动方recv()返回0，发送对这个包的确认，进入close_wait()状态
如果服务器出现大量的fin_wait_2,说明服务器没及时close()，因为业务逻辑太多了，服务器需要很长时间去处理
解决办法：要么先close()再处理业务逻辑；要么将这些数据放到一个队列，交给线程池去处理（应用层和传输层解耦）。
主动方接收到ack，进入fin_wait_2状态
被动方调用close()，进入last_ack状态
主动方recv()返回0，发送对这个包的确认，进入time_wait状态
这个状态会维持2MSL,因为怕这个ack丢失了，被动方会超时重传fin包，但由于主动方关闭连接了，一直无法响应，那服务器就一直关闭不了
close后的socket回收：

被动方：发送fin包：fd被回收；收到ack包：tcb被回收
主动方：发送fin包：fd被回收；time_wait到期：tcb被回收
