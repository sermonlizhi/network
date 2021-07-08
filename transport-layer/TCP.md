### TCP简介

- 点对点

  一个发送方，一个接收方

- 可靠的、有序的字节流

- 管道化(流水线)

  TCP拥塞控制和流量控制设置窗口大小

- 发送和接收缓存

  发送方和接收方都有存放报文段的缓冲区和滑动窗口

- 全双工数据

  - 在同一个连接中数据流双向流动

    一个连接的双方，既是发送方，也是接收方

  - MSS：最大报文段大小(1460字节)

    上层应用把请求报文发送到TCP传输层，TCP传输层将请求报文按照规定的报文段大小分成若干了小的请求报文，再给这些请求报文加上TCP头部消息，组装成TCP报文段；在IP层，再将报文段加上IP头部消息组成IP数据报放在一个MTU(Maximum Transmission Unit)中，MTU为包或帧的最大长度。

    TCP头和IP头都为20字节，而在以太网或者其他网络中，一个MTU的最大为1500字节，所以MSS为1460字节

- 面向连接

  在数据交换之前，通过握手(交换控制报文)初始化发送、接收方的状态变量

- 流量控制

  发送方不会淹没接收方

### TCP报文段结构

![image-20210612175424926](https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210612175424926.png)

- 序号和确认号

  序号(seq)：报文段首字节在字节流(请求报文)的编号

  每个TCP报文段的起始值在整个请求报文的索引位置(数字)

  TCP将请求报文拆分成分为1、2、3、4、5段

  序号=起始值+MSS*(段号-1)

  假设起始值为55，MSS=100字节，则第一个报文段的序号=55，第二个报文段序号=55+100*(2-1)=155，依次类推

  在客户端发起连接建立的时候，会先随机生成一个seq作为序号的起始值

  <!--起始值并不是都从1开始的原因是，防止新的TCP段对老的TCP段的影响，在TCP连接建立以及数据传输时有用-->

  

  确认号ACK：

  期望从另一方收到的下一个字节的序号

  主机A向主机B发送数据，主机B收到数据后给A一个期望的下一个字计数

  例如：

  A向的初始序号为22，向B发送了100字节数据，则B响应A的ACK确认号为123

  **TCP中如何处理乱序的报文段-没有规定**

  

- 4bit的首部长度字段，指示TCP首部长度，通常选项字段为空，所有TCP首部的典型长度是20字节

- 接收窗口

  用于流量控制，该字段指示了接收方愿意接收的字节数量

- 6bit的的标志字段

  ACK用于指定确认字段中值是否有效，即该报文段包括一个对已被成功接收报文段的确认。RST、SYN、FIN用于连接和拆除

  SYN = 1：建立连接

  FIN =1 ：断开连接

  RST = 1 ：服务端根据目标的I端口号没有找到对应的应用程序的socket

- 检验和同UDP

  对所有数据进行累加，报错首部信息和数据信息

### 往返延时(RTT)和超时

SampleRTT：测量报文段发出到收到确认的时间

<!--如果有重传，忽略此次测量-->

由于SampleRTT会变化，则对最近的测量值求平均值

计算移动平均值：

EstimateRTT = (1-a)*EstimateRTT + a * SampleRTT

a = 0.125

在以往平均值的作用下，使计算出来的当前RTT更准确



设置超时时间=EstimateRTT +安全边界时间

SampleRTT会偏离EstimateRTT多远：

DevRTT = (1-b)*DevRTT+b *|SampleRTT-EstimateRTT|（推荐值：b=0.25）

超时时间 TimeOutIntervel=EstimateRTT +4*DevRTT

### TCP的可靠数据传输

- TCP在IP不可靠服务的基础上建立了rdt

  - 管道化的报文段(GBN/SR)

  - 累计确认(像GBN)——确认号之前的全部正确接收

  - 单个重传定时器(像GBN)

    如果请求超时，只重发发送端缓冲区中最老的未确认分组，而不是每一个都发

  - 是否可以接收乱序——没有规范

- 以下事件触发重传

  - 超时（只重发那个最早的未确认的段：SR）

  - 重复确认(快速重传)

    收到了ACK50之后，在发送定时器到时间之前，又收到三个ACK50，大概率丢包了，重发

- TCP发送事件

  1. 从应用层接收数据
     - 用nextseq创建报文段(借助生成seq)
     - 序号nextseq为报文段首字节的字节流编号
     - 如果还没有运行定时器则启动定时器
       - 定时器与最早的未确认的报文段关联
       - 过期间隔 TimeOutIntervel，重发(启动新的定时器)
     - 
  2. 超时
     - 沿最老的未确认报文重传，重新启动定时器
  3. 收到确认
     - 如果是对尚未确认的报文段确认
       - 更新已被确认的报文序号
       - 如果当前还有未确认的的报文段，重新启动定时器与最早的关联

- 流量控制

  接收方控制发送方，不让发送的太多、太快以至于让接收方的缓冲区溢出。

  根据接收方响应TCP段中TCP头的流量控制字段，指定接收方能接收的最大字节数

### TCP连接管理

- **两次握手**

  流程：

  - 客户端发送连接建立请求(seq=22)
  - 服务端响应，同意建立连接

  问题1(半连接)：

  - 客户端发送第一次连接请求后，服务端同意，但还未来得及响应，超时时间到了
  - 客户端发送第二次连接请求，服务端同意
  - 客户端收到了第一次的连接请求响应，然后断开连接
  - 此时，客户端已经没有与服务端的连接信息，但服务端仍然维护了一个与客户端的连接句柄

  问题2：

  - 客户端发送第一次连接请求后，服务端同意，但还未来得及响应，超时时间到了
  - 客户端发送第二次连接请求，服务端同意
  - 客户端收到了第一次的连接请求响应，然后发送数据，但没有响应，此时客户端断开与服务端的连接
  - 客户端收到第二次连接建立的响应，发现发送的数据没有响应，又重发了一份，此时服务端既维护了一个与客户端的连接句柄，还缓存了一份无用的数据

- **三次握手**

  连接过程：

  - 客户端发送一个报文段给服务端，报文段头部的标志位SYN被置为1。同时客户端随机选择选择一个client_seq，作为初始序号(告诉服务端，从client_seq开始传输数据)
  - 服务端收到一个包含TCP SYN报文段的IP数据报，服务端提取TCP报文段，为该TCP连接分配缓存和变量，并向客户端发送允许连接的报文。其中包含SYN被置为1，确认号为ack=client_seq+1(告诉客户端可以从client_seq+1字节发送数据)，服务端的初始化序号server_seq
  - 客户端收到服务端的SYN ACK报文段，客户端也要给该连接分配缓冲区和变量。连接已经建立后，客户端给服务端发送确认报文，包括SYN=0，ack=server_seq+1(告诉服务端可以从ack字节发送数据)，client_seq=client_seq+1(第一个报文段的序号)

  **三次握手可以解决两次握手出现的问题：**

  三次握手中，多了一次客户端发送给服务端的确认消息，这样如果客户端发送连接请求超时后，继续发一次新的连接请求时，而第一次的连接断开后，服务器收到客户端第二次发送的连接请求时，会响应客户端连接建立，这时，在客户端是知道该连接已经关闭了的，那么此时客户端响应给服务端的消息是拒绝连接，这样就不会存在半连接的问题了

  ![image-20210615092402139](https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210615092402139.png)

  <!--第二次连接只能在第一次连接断开后才能正确被服务端处理，如果第一次连接还没断开，第二次的连接请求到服务端之后会被拒绝-->

- **连接关闭(四次挥手)**

  参与TCP连接的两个进程中的任何一个都能够终止该连接，当连接结束后，主机中的“资源”(即缓存和变量)将被释放，每个连接对应独立的发送缓冲区和滑动窗口(保证了数据的有序性)

  **一个TCP连接的双方都要维护两个缓存区：发送缓冲区和接收缓冲区**

  ![image-20210615095730173](https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210615095730173.png)

  连接关闭流程：(连接是双向的)

  - 客户端(服务端)向服务端发送连接断开的报文段(标志位FIN=1)
  - 服务器收到连接断开请求后，响应一个确认报文
  - 服务器发送中止与客户端连接的报文段(标志位FIN=1)
  - 客户端收到服务端的中止请求，响应一个确认报文
  - 客户端发送出最后一个确认报文后开始定时(30s)，一定时间内没有收到服务方发送的数据包，则连接完全断开。

  客户端/服务端连接建立及断开的状态图：

  

  ![image-20210615104843031](https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210615104843031.png)

  ![image-20210615105610370](https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210615105610370.png)

- **SYN泛洪**

  在TCP连接的三次握手中，服务器为了响应收到的SYN，分配并初始化连接变量和缓存，如果客户端不发送ACK来完成三次握手的第三步，**最终(通常在一分钟之后)服务器将终止该半连接并回收资源**。

  可以利用TCP的这种特性进行SYN泛洪攻击，通过不停的发送SYN报文段但不完成第三次确认，服务器不断为这些半连接分配资源，导致连接资源被消耗殆尽。

  SYN cookie应用而生：

  - 服务端接收到客户端发送过来的SYN时，并不会为该SYN报文段生成一个半连接。服务器会生成一个初始化序号，该序列号是根据报文段中源和目的IP地址与端口以及仅服务器知道的一个散列函数，这种精心设计的初始化序列号被称为cookie。服务器发送这张具有特殊初始化序列号的SYN ACK报文段。服务器并不存储cookie或任何对应于SYN的其他状态信息
  - 如果客户端是合法的，则它将返回一个ACK报文段。对于一个合法的ACK，在确认字段的值等于在SYN ACK(cookie)字段中的值加一。服务器则将使用客户端在SYN ACK中的源和目的IP地址和端口号以及散列函数进行计算，如果结果加一等于客户端SYN ACK的确认值，则认为该ACK对应的较早的SYN报文段是合法的，服务器将生成一个具有套接字的全开连接。
  - 如果客户端没有返回ACK响应报文段，则初始的SYN并没有对服务器产生危害，因为服务器没有为它分配任何资源