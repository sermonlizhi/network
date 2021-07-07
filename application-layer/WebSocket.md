## 一、WebSocket协议

WebSocket可以实现客户端与服务器间双向、基于消息的文本或二进制数据传输。它是浏览器中最靠近套接字的API，但WebSocket连接远远不是一个网络套接字，因为浏览器在这个简单的API之后隐藏了所有的复杂性，而且还提供了更多服务：

- 连接协商和同源策略
- 与既有HTTP基础设施的互操作
- 基于消息的通信和高效消息分帧
- 子协议协商及可扩展能力

WebSocket是浏览器中最通用最灵活的一个传输机制，其极简的API可以让我们在客户端和服务器之间以数据流的形式实现各种应用数据交换(包括JSON及自定义的二进制消息格式)，而且两端都可以向另一端发送数据。

WebSocket的优点：

- 较少的控制开销，连接建立后，客户端和服务器之间交换数据时，用于协议控制的数据包头部相对较小，相对于HTTP每次请求都要携带完整的头部，此项开销显著减少了
- 更强的实时性，由于协议是全双工的，服务器可以随时主动给客户端下发数据。相对于HTTP请求需要客户端发送请求服务端才能响应，延迟明显减少
- 长连接保持连接状态，与HTTP不同的是，WebSocket需要先创建连接，这使得它成为一种有状态的协议，之后通信就可以忽略状态信息。而HTTP请求可能需要在每个请求都携带状态信息(如身份认证等)
- 双向通信、更好的二进制支持。与HTTP协议有着良好的兼容性。默认端口也是80和443，并且握手阶段采用HTTP协议，因此握手时不容易被屏蔽，能通过各种HTTP代理服务器

WebSocket通信协议(RFC 6455)包含两个高级组件:

- 开放性HTTP握手用于协商连接参数

- 二进制消息分帧用于支持低开销的基于消息的文本和二进制数据传输

  <!--WebSocket协议尝试在既有HTTP基础设施中实现双向HTTP通信，因此也使用HTTP的80和443端口……不过，这个设计不局限于通过HTTP实现WebSocket通信，未来的实现可以在某个专用端口上使用更简单的握手，而不必重新定义这么一个协议-->

  目前用的最多的还是依赖HTTP进行握手，因为HTTP的基础设施已经相对完善。

## 二、WebSocket握手

#### 标准的握手流程

看一个具体的WebSocket握手的例子，以www.zhihu.com为例，打开这个首页，网页一渲染就会开启一个wss的握手请求，握手请求如下

##### 请求报文

```java
//请求的方法必须是GET,HTTP版本必须至少是1.1
GET wss://mqtt-web.zhihu.com/mqtt?client_info=OS%3DWeb&user_group=zhihu_web HTTP/1.1

Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,und;q=0.8
Cache-Control: no-cache
Connection: Upgrade
Host: mqtt-web.zhihu.com
Origin: https://www.zhihu.com
Pragma: no-cache
    
//可选的客户端支持的协议扩展列表，指示了客户端希望使用的协议级别的扩展
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
    
//自动生成的key,已验证服务器对协议的支持，其值必须是
Sec-WebSocket-Key: RfbP3GpmZpMiUaAfex5JRQ==
    
//可选的应用指定的子协议列表(可以是多个,用“,”分割)
Sec-WebSocket-Protocol: mqtt
    
//客户端使用的websocket协议版本
Sec-WebSocket-Version: 13
    
//请求升级到websocket协议
Upgrade: websocket
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.106 Safari/537.36
```

WebSocekt协议和普通的HTTP协议相比，不同的地方有以下几处：

请求的URL是ws://或者wss://开头，而不是HTTP://或HTTPS://。

类比HTTP，ws协议：普通请求，占用HTTP相同的80端口；wss协议：基于SSL的安全传输，占用TLS相同的443端口

```java
Connection: Upgrade
Upgrade: websocket
```

请求头里面的这两处是一般HTTP报文没有的，这里利用Upgrade进行协议升级，指明升级到websocket协议

```java
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
Sec-WebSocket-Key: RfbP3GpmZpMiUaAfex5JRQ==
Sec-WebSocket-Protocol: mqtt
Sec-WebSocket-Version: 13
```

Sec-WebSocket-Version表示websocket的版本号，如果服务器不支持该版本，需要返回一个Sec-WebSocket-Version，里面包含服务端支持的版本号。

<!--最新版本号是13-->

Sec-WebSocket-Key由浏览器生成的，提供基本的防护，防止恶意或者无意的连接

<!--Sec-WebSocket-Key字段用于握手阶段。它从客户端发送到服务器以提供部分内容，服务器用来证明它收到的信息，并且能够有效的完成WebSocket握手。有助于确保服务器不会接收来自非WebSocket客户端的连接(例如HTTP客户端)被滥用发送数据到毫无防备的WebSocket服务器-->

##### **响应报文**

```java
//101响应码确认升级到websocket协议
HTTP/1.1 101 Switching Protocols
Connection: upgrade
Date: Tue, 06 Jul 2021 02:05:15 GMT
//签名的键值验证协议支持
Sec-WebSocket-Accept: gp56u7lzuqL5TSEd2dbk4OmZ8UI=
//服务器选择的应用子协议
Sec-WebSocket-Protocol: mqtt
Server: openresty/1.13.6.2
Upgrade: websocket
X-Cache-Lookup: Cache Miss
X-Cache-Lookup: Cache Miss
X-Cache-Lookup: Cache Miss
X-Cache-Lookup: Cache Miss
x-cdn-provider: tencent
x-edge-timing: 0.191
X-NWS-LOG-UUID: 10821471124405714693
```

在Response Header中，用HTTP 101 响应码回应，确认升级到WebSocket协议。

同样也有两个WebSocket的header:

```java
//签名的键值验证协议支持
Sec-WebSocket-Accept: gp56u7lzuqL5TSEd2dbk4OmZ8UI=
//服务器选择的应用子协议
Sec-WebSocket-Protocol: mqtt
```

Sec-WebSocket-Accept是经过服务器确认后，并且加密之后的Sec-WebSocket-Key。

<!--所有兼容RFC 6455的WebSocket服务器都使用相同的算法计算客户端挑战的答案：将Sec-WebSocket-Key的内容与标准定义的唯一GUID字符串(258EAFA5-E914-47DA-95CA-C5AB0DC85B11)拼接起来，计算出SHA1散列值，结果是一个base-64编码的字符串，把这个字符串发送给客户端即可。-->

WebSocket握手成功的最低限度是客户端发送协议版本和自动生成的Sec-WebSocket-Key，服务器返回101 HTTP响应码(Switching Protocols)和散列形式的挑战答案，选择确认的协议版本：

- 客户端必须发送Sec-WebSocket-Version和Sec-WebSocket-Key
- 服务器必须返回Sec-WebSocket-Accept确认协议
- 客户端可以通过Sec-WebSocket-Protocol发送应用子协议列表(不是必须)
- 如果客户端发送了子协议列表，服务器必须选择一个子协议并通过Sec-WebSocket-Protocol返回协议名；如果服务器不支持任何一个协议，连接断开
- 客户端可以通过Sec-WebSocket-Extensions发送一个或多个扩展；如果服务器没有返回扩展，则连接不支持扩展

最后，前述握手完成后，如果握手成功，该连接就可以用作双向通信信道交换WebSocket消息。从此以后，客户端与服务器之间不会再发生HTTP通信，一切由WebSocket协议接管(只有握手过程借助HTTP完成)。

Sec-WebSocket-Key/Sec-WebSocket-Accept只是握手的时候保证握手成功，但对数据并不保证，用wss://会稍微安全一点

#### 子协议协商

WebSocket握手可能会涉及到子协议的问题。

WebSocket协议对每条消息的具体格式不做规定：仅用一位标记消息是文本还是二进制，方便客户端和服务器有效地解码数据，而除此之外地消息内容是未知的。

HTTP可以通过每次请求和响应的首部来沟通元数据，而WebSocket没有同样的机制。因此，如果需要沟通关于消息的元数据，客户端和服务器必须达成沟通这一数据的子协议。

- 客户端和服务端可以提前确定一种固定的消息格式，比如所有通信都通过JSON编码的消息或某种自定义的二进制格式进行，而必要的元数据作为这种数据结构的一个部分
- 如果客户端和服务端要发送不同的数据类型，那它们可以确定一个双方都知道的消息首部，利用它来说明信息或有关净荷的其他编码信息
- 混合使用文本和二进制消息可以沟通净荷和元数据，比如用文本消息实现HTTP首部的功能，后跟包含应用净荷的二进制消息

WebSocket为此提供了一个简单便捷的子协议协商API。客户端可以在初次连接握手时，告诉服务器支持哪种协议：

```javas
//在WebSocket握手期间发送子协议数组
var ws = new WebSocket('wss://mqtt-web.zhihu.com/mqtt',['mqtt','xxx']);
//检查服务器使用了哪个子协议
ws.onopen = function(){
	if(ws.protocol == 'mqtt'){
		……
	}else{
		……		
	}
}
```

如例子所示，WebSocket构造函数可以接受一个可选的子协议名字的数组，通过这个数组，客户端向服务器发送自己理解的协议，服务器从中间选择一个自己也支持的协议，如果对于客户端发送的子协议数组，服务器都不支持，则WebSocket握手是不完整的，此时会触发onerror回调，连接断开。

#### 多版本的WebSocket握手

使用WebSocket版本通知能力(Sec-WebSocket-Version头字段)，客户端可以初始请求它选择的WebSocket协议的版本(不一定必须是客户端最新的)。如果服务器支持请求的版本且握手消息本来是有效的，服务器将接受该版本。如果服务器不支持请求的版本，它必须以一个包含所有它使用的版本的Sec-WebSocket-Version头字段来响应(或多个Sec-WebSocket-Version头字段)来响应，如果此时客户端支持其中的一个通知版本，它可以使用新的版本值重做WebSocket握手

举个例子：

```java
GET /server HTTP/1.1
Host:www.lizhi.com
Upgrade:websocket
Connection:Upgrade
……
Sec-WebSocket-Version:14
```

服务器不支持14的版本，则会返回：

```java
HTTP/1.1 400 Bad Request
……
Sec-WebSocket-Version:13,8,7
```

客户端支持13版本的，再次握手：

```java
GET /server HTTP/1.1
Host:www.lizhi.com
Upgrade:websocket
Connection:Upgrade
……
Sec-WebSocket-Version:13
```

## 三、升级协商

WebSocket协议提供了很多强大的功能：基于消息的通信、自定义的二进制分帧层、子协议协商、可选的协议扩展等，在交换数据前，客户端必须与服务器协商适当的参数以建立连接。

在WebSocket握手阶段，会带有5个WebSocket的header，这5个header都是和升级协议相关的。

- Sec-WebSocket-Version

  客户端发送，表示它想使用的WebSocket协议版本("13"表示RFC 6455)。如果服务器不支持这个版本，必须回应自己支持的协议版本

- Sec-WebSocket-Key

  客户端发送，自动生成的一个键，作为对服务器的“挑战”，已验证服务器支持请求的协议版本

- Sec-WebSocket-Accept

  服务器响应，包含Sec-WebSocket-Key的签名值，证明它支持请求的协议版本

- Sec-WebSocket-Protocol

  用于协商应用子协议：客户端发送支持的协议列表，服务器必须只回应一个协议名，否则握手断开

- Sec-WebSocket-Extensions

  用于协商本次连接要使用的WebSocket扩展：客户端发送支持的扩展，服务器通过返回相同的首部确认自己支持一个或多个扩展

  与浏览器中客户端发起的任何连接一样，WebSocket请求也必须遵守同源策略：浏览器会自动在升级握手请求中追加Origin首部，远程服务器可能使用CORS判断接受或拒绝跨域请求。

## 四、协议扩展

WebSocket规范允许对协议进行扩展：数据格式和WebSocket协议的语义可以通过新的操作码和数据字段扩展。

负责制定WebSocket规范的HyBi Working Group就进行了两项扩展：

- 多路复用扩展(A Multiplexing Extension for WebSocket)

  这个扩展可以将WebSocket的逻辑连接独立出来，实现共享底层的TCP连接。

  共享TCP连接可以让服务器维护更多的TCP套接字，但多个WebSocket共享同一个TCP连接会导致TCP头部阻塞的概率增加

- 压缩扩展(Compression Extensions for WebSocket)

  给WebSocket协议增加了压缩功能(例如x-webkit-deflate-frame)

WebSocket很容易发生队首阻塞的情况：消息可能会被分成一个或多个帧，但不同消息的帧不能交错发送，因为没有与HTTP2.0分帧机制中“流ID”对等的字段。显然，如果一个大消息被分成多个WebSocket帧，就会阻塞其他消息的帧。如果应用不容许有交付延迟，那可以小心控制每条消息帧的大小，甚至可以考虑将大消息分成多个小消息，这个一个大消息就可以乱序发送了。

WebSocket不支持多路复用，还意味着每个WebSocket连接都需要一个专门的TCP连接，消耗客户端和服务器资源

<!--这个扩展通过封装帧并加上信道ID，可以让一个TCP连接支持多个虚拟WebSocket连接……这个多路复用扩展维护独立的逻辑信道，每个逻辑信道与独立的WebSocket连接没有差别，包括独立的握手首部-->

有了扩展之后，多个WebSocket连接(信道)就可能在同一个TCP连接上得到复用，不过每个信道依然容易产生队首阻塞的问题！可以使用不同的信道或专用TCP连接，多路并行发送消息。

## 五、WebSocket数据帧

WebSocket的另一个高级组件是：二进制消息分帧机制。客户端和服务器WebSocket应用通过基于消息的API通信：发送端提供任意UTF-8或二进制的净荷，接收端再整个消息可用时收到消息。WebSocket会把应用消息分割成一个或多个帧，接收方收到进行组装，等到接收到完整消息之后再通知接收端。

#### 数据帧结构

![image-20210706181941313](https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210706181941313.png)

![image-20210706182333055](https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210706182333055.png)

- 帧

  最小的通信单位，包含可变长度的帧首部和净荷部分，净荷部分包含完整或部分应用消息

- 消息

  一系列帧，与应用消息对等

是否把消息分帧由客户端和服务器实现决定

- FIN：0表示不是最后一帧，1表示最后一帧

- RSV1，RSV2，RSV3：一般情况下全为0。当客户端、服务端协商采用WebSocket扩展时，这三个标志位可以非0，且值的含义由扩展进行定义。如果出现非零的值且并没有采用WebSocket扩展，连接出错

- 操作码(Opcode)：

  - %x0：表示一个延续帧。当Opcode为0时，表示本次数据传输采用了数据分片，当前收到的数据帧为其中一个数据分片
  - %x1：表示一个文本帧(text frame)
  - %x2：表示一个二进制帧(binary frame)
  - %x3-7：保留的操作代码，用于后续定义的非控制帧
  - %x8：表示连接断开
  - %x9：表示这是一个心跳请求(ping)
  - %xA(%x10)：表示这是一个心跳响应(pong)
  - %xB-F：保留的操作代码，用于后续定义的控制帧

- 掩码(Mask)

  表示是否对数据载荷进行掩码异或操作。1表示需要，0表示不需要。

  <!--只适用于客户端给服务器的消息，客户端给服务器发送消息，这里一定为1-->

- 长度(Payload len)

  表示数据载荷的长度，有三种情况：

  - 如果数据长度Payload len在0-125之间，那么Payload len用7位表示足以，表示的数也就是净荷长度
  - 如果数据长度Payload len等于126，那么Payload len用7+16位(扩展位)表示，接下来2字节表示的16位无符号整数才是这一帧的长度
  - 如果数据长度Payload len等于127，那么Payload len用7+64(扩展位)位表示，接下来8字节表示的64位无符号整数才是这一帧的长度

- 掩码键(Masking-key)

  如果Mask = 0，则没有Masking-key，如果Mask = 1，则Masking-key长度为4字节，32位。

  <!--掩码是由客户端随机选择的32值，客户端发给服务器的时候这里必须要带上Masking-key，它用于标记 Payload data (包含扩展数据和应用数据两种)-->

- 净荷/载荷(Payload Data)

  载荷数据分为扩展数据和应用数据两种

  扩展数据：如果没有协商使用扩展，则扩展数据为0字节。扩展数据的的长度如果存在，必须在握手阶段就固定下来。载荷数据的长度也要算上扩展数据。

  应用数据：如果存在扩展数据，则排在扩展数据之后

#### WebSocket控制帧

控制帧和非控制帧由操作码决定，操作码一共4位，其中0x0~0x7表示非控制帧，0x8~0xF表示表示控制帧，所以非控制帧的最高位为0，控制帧的最高位为1。当前定义的用于非控制帧的操作码包括0x0(数据分片)、0x1(文本)、0x2(二进制)，0x3~0x7为非控制帧的预留操作码；用于控制帧的操作码包括0x8(close)、0x9(ping)、0xA(pong)，其中0xB-0xF为预留的尚未定义的控制帧

控制帧用于传达有关WebSocket的状态，控制帧可以插入到分帧消息的中间(关闭帧)。

所有的控制帧必须有一个小于等于125字节的有效载荷长度，控制帧必须不能被分帧。

- 当接收到0x8 Close操作的控制帧以后，可以关闭底层的TCP连接了。客户端可以等待服务器关闭以后，在一段时间没有响应了，在关闭自己的TCP连接
- 当接收到0x9 Ping操作码的控制帧以后，应当立即发送一个包含pong操作码的响应帧，除非接收到了关闭帧。两端都会在连接建立后、关闭前的任意时间内发送Ping帧。Ping帧可以包含“应用数据”，Ping帧可以作为keepalive心跳包。
- 当接收到0xA Pong操作码的控制帧以后，知道对方还可响应。Pong帧必须包含与被响应Ping帧的应用程序完全相同的数据。如果终端接收到Ping帧，但还没对之前的Ping帧发送Pong响应，终端可以选择发送一个最近的Pong帧给最近处理的Ping帧。一个Pong帧可以被主动发送，作为单向心跳，但尽量不要主动发送Pong。

#### 分帧规则

分帧规则由RFC 6455进行定义，应用对于如何分帧是无感知的，分帧这一步由客户端和服务器组成。

分帧也可以更好的利用多路复用的协议扩展，多路复用需要能够将消息分割成更小的段来更好的共享传输通道。

RFC 6455规定的分帧规则：

- 一个没有分片的消息由单个带有FIN位设置和一个非0操作码的帧组成

- 一个分片的消息由单个带有FIN位为0和一个非0操作码的帧，后面跟随零个或多个带有FIN为0和操作码设置为0的帧，且终止于一个带有FIN位设置且操作码为0的帧组成。

  一个分片的消息概念上等价于单个大的消息，其负载是等价于按顺序串联片段的负载；然而，在存在扩展的情况下，这个可能不适用扩展定义的“扩展数据”存在的解释，例如：“扩展数据”可能仅在首个片段开始处存在且应用到随后的片段，或“扩展数据”可以存在于仅用于特定片段的每个片段。在没有“扩展数据”的情况下，如下展示了分片如何工作。

例子：对于一个作为三个片段发送的消息，第一个片段将有一个0x1操作码和FIN位清零，第二个片段将有一个0x0操作码和FIN位清零且第三个片段将有一个0x0操作码和FIN位置1

- 控制帧可能被注入一个分片消息的中间，控制帧本身必须不被分割

- 消息分片必须按照发送者发送顺序交付给接收方

- 片段中的一个消息不能与片段中的另一个消息交替，除非已协商了一个能解释交替的扩展

- 一个端点必须能处理一个分片消息中间的控制帧

- 一个发送者可以位非控制消息创建任何大小的片段

- 客户端和服务器必须支持接收分片和非分片的消息

- 由于控制帧不能被分片，一个中间件必须不尝试改变控制帧的分片

- 如果使用了任何保留的位值且这些值得意思对于中间件是不可知的，一个中间件必须不改变一个消息的分片

- 在一个连接上下文中，已经协商了扩展但中间件不知道协商的的扩展语义，一个中间件必须不改变任何消息的分片。同样，没有看见WebSocket握手(且没被通知有关它的内容)、导致一个WebSocket连接的中间件，必须不改变这个链接的任何消息的分片。

- 由于这些规则，一个消息的所有分片都是相同类型，以第一个片段的操作码设置。因为控制帧不能被分片，用于一个消息中的所有分片的类型必须要么是文本、要么是二进制、要么是一个保留的操作码。

  <!--如果控制帧不能被插入，一个ping延迟，例如，如果跟着一个大消息将是非常长的。因此，要求在分片消息的中间处理控制帧-->

  实现注意：在没有任何扩展时，一个接收者不必按顺序缓冲整个帧来处理它。例如，如果使用了一个流式API，一个帧的一部分被交付到应用。但是，这个假设可能不适合所有未来WebSocket的扩展

#### 分帧开销

一个分帧的消息包含：单个开始帧(FIN为0，opcode非0)、0个或多个中间帧(FIN为0，opcode设置为0)、单个结束帧(FIN为1，opcode设置为0)。一个分帧了的消息在概念上等于一个未分帧的大消息，它的有效载荷等于所有帧的有效载荷长度的累加；然而，有扩展时，这可能不成立，因为扩展定义了出现的Extension Data的解释。例如，Extension Data可能只出现在第一帧，并用于后续的所有帧，或者Extension Data出现于所有帧，但只应用于特定的那个帧。

通常来说，服务器就包含这三种帧：开始帧、中间帧和结束帧。开始帧和结束帧可以待数据也可以不带数据，分帧开销主要花费在新增加的帧头信息上，在不带数据的情况下，开销大小是1+3+4+1+7=16bit=2Byte，中间帧带数据的情况下，开销大小是1+3+4+1+7+64=80bit=10Byte(假设数据长度为127，扩展长度要加上64bit)

服务器分帧开销范围在[2，10]字节，客户端要比服务器多增加Masking-key，这个字节占4字节(32位)，所以客户端开销在[6，14]字节







​	



