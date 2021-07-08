## 一、TCP套接字编程

1. 服务器进程必须先处于运行状态

   - 创建一个监听socket(或欢迎socket)
   - 和本地端口绑定(指定监听socket的监听端口)
   - 监听socket阻塞等待接收客户端的连接

   监听socket只有服务器IP地址和程序端口号,格式如下：

   socket | ip:port|

2. 创建客户端本地套接字(隐式绑定到本地Port)

   客户端绑定的Port由传输层自行分配

   - 指定服务器进程的IP地址和port端口号

3. 当有客户端连接请求到来

   - 服务器的监听socket接收到来自客户端的请求，返回一个连接socket(与监听socket不同)，与客户端端通信

     Connection Socket信息(维护服务器和客户端IP和Port四元组信息)：

     socket | server ip:port | client ip:port|

   - 允许服务器与多个客户端通信

   - 使用客户端源IP和源端口来区分不同客户端的连接Socket

4. 客户端调用传输层的连接API调用生效后，客户端建立了与服务器的TCP连接，可以进行通信

- 数据结构

  - sockaddr_in

    IP地址和Port绑定关系的数据结构

    struct sockaddr_in {

    short sin_family;  ——协议族IPV4/IPV6

    u_short  sin_port;  ——端口

    struct in_addr sin_addr;——IP地址

    char sin_zero[8]; ——对齐

    }

  - hostent

    域名和ip地址的数据结构

    struct hostent{

    char *h_name; ——正式主机名

    char **h_aliases; ——主机别名

    int h_addrtype; ——IP地址类型 IPV4-AF_INET

    int h_length; ——IP地址字节长度，对于IPV4是4字节，即32位

    char **h_addr_list; ——IP地址列表

    }

## 二、UDP套接字编程

在客户端与服务器之间没有连接，传送的数据可能乱序，也可能丢失

- 没有握手
- 发送端在每一个报文中明确指定目标IP地址和端口
- 服务器必须从收到的分组中提取发送端的IP地址和端口号

通信过程与TCP基本相同，只是没有连接过程，服务端的监听socket来处理具体的请求