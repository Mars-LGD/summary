##(1) TCP 连接
1. 三次握手与四次挥手，SYN、FIN、ACK、WAIT

  ![image](http://image.pangrun.net/18-12-25/33726184.jpg)

2. tcp长连接与http长连接的区别
  tcp长连接的本质是保持连接，需要一些心跳的监控检测机制。
  http长连接的本质是连接复用，并非保持tcp的长连接，客户端和服务端都需要设置keepAliveTimeOut。

3. tcp是一种全双工通信方式  

##(2) Socket、ServerSocket、AcceptSocket的区别
```
## Socket Demo

// Server 端
ServerSocket mServerSocket = new ServerSocket(8080);
Socket acceptSocket = mServerSocket.accept();

// Client 端
Socket mSocket = new Socket("127.0.0.1", 8080);
```
如上共有三个Socket，有几个问题需要注意：
* 区分不同应用进程间的网络通信和连接，主要有3个参数：通信的目的IP地址、使用的传输层协议(TCP 或 UDP)和使用的端口号。
* ServerSocket和AcceptSocket的监听端口一致，`不会新建随机端口`。ServerSocket只负责监听，不会处理请求。
* ServerSocket.accept()默认会阻塞进程，处理流程如下：
   * 有人从很远很远的地方尝试调用 connect()来连接你的机器上的某个端口（当然是你已经在 listen()的）。
   * 他的连接将被 listen 加入等待队列等待 accept()函数的调用（加入等待队列的最多数目由调用 listen()函数的第二个参数 backlog和系统属性/proc/sys/net/core/somaxconn的相对小值来决定）。
   * 你调用 accept()函数，告诉他你准备连接。
   * accept()函数将返回一个新的套接字描述符，这个描述符就代表了这个连接！  
![image](http://image.pangrun.net/tcp-accept-queue.png)
