---
title:  "网络通信、协议"
date:   2020-12-13 22:11:36 +0530
author: taoye
categories: [iOS, 2020年复现总结]
tags: [iOS, network]
---

### OSI模型
![6bbdced38f4c954435e26225b45a3991](/assets/img/iOS-network/D99EE970-926B-4144-A4BA-BA785D5B3A8D.png)
![3a57605b0546d2da0d3b5ac2efaa51f6](/assets/img/iOS-network/AACFFB22-73EA-4378-B6B8-D27DD10AAB72.png)
值得注意的是 DNS 是应用层协议，SSL 分别位于第五层会话层和第六层表示层。TLS 位于第五层会话层。

![f4ddf874f331135752738becae954d69](/assets/img/iOS-network/763CA188-7E96-4575-9306-348CFAC206C2.png)


### OSI模型通信
![59bf6ec69e7026c0b91fa0d8f58bced4](/assets/img/iOS-network/F97E2A98-A6E2-4BC9-BE5C-33D30E5F02AB.jpg)
值得说明的一点，在数据链路层的以太网帧里面，除去以太网首部 14 字节，FCS 尾部 4 字节，中间的数据长度在 46-1500 字节之间。
帧尾都有一个 4 字节的 FCS（Frame Check Sequence）。FCS 表示帧校验序列(Frame Check Sequence)，用于判断帧是否在传输过程中有损坏(比如电子噪声干扰)。FCS 保存着发送帧除以某个多项式的余数，接收到的帧也做相同计算，如果得到的值与 FCS 相同则表示没有出错。

![c2308edec2a27b947916a65c613b6c11](/assets/img/iOS-network/3E395E40-6650-490E-B02A-90EAFB95680C.png)
上图是 OSI 7 层模型中通信时候的数据图。从上图的 7 层模型中，物理层传输的是字节流，数据链路层中包的单位叫，帧。IP、TCP、UDP 网络层的包的单位叫，数据报。TCP、UDP 数据流中的信息叫，段。最后，应用层协议中数据的单位叫，消息。


![81a61d82a2f5f4d2501bf80641f680c6](/assets/img/iOS-network/4681657F-E5B4-43FC-9A6F-D12261221F98.png)


### IP协议
IP 属于面向无连接类型，原因有两点：一是为了简化，二是为了提速。为了把数据包发送到最终目标地址，尽最大努力。简单，高速。而可靠性就交给了 TCP 去完成。
![f0dda66801b4c874795d23b06ee7d4db](/assets/img/iOS-network/8BF209C6-7507-4F73-8EDD-E74D5DC35185.png)

所谓数据包是指一段二进制数据，其中包含了发送源主机和目标主机的信息。IP 网络负责源主机与目标主机之间的数据包传输。IP 协议的特点是 best effort（尽力服务，其目标是提供有效服务并尽力传输）。这意味着，在传输过程中，数据包可能会丢失，也有可能被重复传送导致目标主机收到多个同样的数据包。
IP 网络中的主机都配有自己的地址，被称为 IP 地址。每个数据包中都包含了源主机和目标主机的 IP 地址。IP 协议负责路径计算，即 IP 数据包在网络中的传输传输时，数据包所经过的每一个主机节点都会读取数据包中的目标主机地址信息，以便选择朝什么地方传送数据包。

### ICMP
（网络层协议）
ICMP 主要功能包括，确认 IP 包是否成功送达目标地址，通知在发送过程当中 IP 包被废弃的具体原因，改善网络设置等。有了这些功能以后，就可以获得网络是否正常，设置是否有误，以及设备有何异常等信息，从而便于进行网络上的问题诊断。ICMP 消息大致可以分为两类：一类是通知出错原因的错误消息，另一类是用于诊断的查询信息。


### TCP
TCP 提供一种面向连接的、可靠的字节流服务。
在一个 TCP 连接中，仅有两方进行彼此通信。广播和多播不能用于 TCP。
TCP 使用校验和，确认和重传机制来保证可靠传输。
TCP 给数据分节进行排序，并使用累积确认保证数据的顺序不变和非重复。
TCP 使用滑动窗口机制来实现流量控制，通过动态改变窗口的大小进行拥塞控制。
TCP 通过检验和、序列号、确认应答、重发控制、连接管理以及窗口控制等机制实现可靠性传输。

TCP的五大元素：目标地址，源地址，目标端口号，源端口号，协议号
不同的端口用于区分同一台主机上不同的应用程序。假设你打开了两个浏览器，浏览器 A 发出的请求不会被浏览器 B 接收，这就是因为 A 和 B 具有不同的端口。

协议号用于区分使用的是 TCP 还是 UDP。因此相同两台主机上，相同的两个进程之间的通信，在分别使用 TCP 协议和 UDP 协议时也可以被正确的区分开来。


TCP 首部：不计算选项字段，TCP 首部的长度为 20 byte


TCP 的报文段 header 信息中还有报文序列号、确认号等其他一些用于管理连接的信息。
所谓序列号信息，其实就是为每个报文段分配的唯一编号。第一个报文段的序列号是随机的，比如：1721092979，其后的每一个报文段的序列号都以此号为基础依次加 1，1721092980，1721092981 等等。至于确认号，是目标端反馈给源的确认信息，通知源目前已经接到哪些报文段了。由于 TCP 是双向的，所以数据和确认信息发送也都是双向的。

![571f64c617d9b308a916c1586bcd4f4b](/assets/img/iOS-network/05B96C80-47B7-4FD8-9775-913C4F4FB730.png)

### TCP 安全传输、控制
TCP 使用超时重传来实现可靠传输：如果一个已经发送的报文段在超时时间内没有收到确认，那么就重传这个报文段。

流量控制是为了控制发送方发送速率，保证接收方来得及接收。
接收方发送的确认报文中的窗口字段可以用来控制发送方窗口大小，从而影响发送方的发送速率。将窗口字段设置为 0，则发送方不能发送数据。
流量控制的原则是发送方发送数据的速度不能比接收方处理数据的速度快。接收方，也就是所谓的 接收窗口 (receive window) 会告知发送方自身接收窗口数据缓冲区的大小

TCP 主要通过四种算法来进行拥塞控制：慢开始、拥塞避免、快重传、快恢复。发送方需要维护一个叫做拥塞窗口（cwnd）的状态变量。

1. 慢开始与拥塞避免
发送的最初执行慢开始，令 cwnd=1，发送方只能发送 1 个报文段；当收到确认后，将 cwnd 加倍，因此之后发送方能够发送的报文段数量为：2、4、8 ...

注意到慢开始每个轮次都将 cwnd 加倍，这样会让 cwnd 增长速度非常快，从而使得发送方发送的速度增长速度过快，网络拥塞的可能也就更高。设置一个慢开始门限 ssthresh，当 cwnd >= ssthresh 时，进入拥塞避免，每个轮次只将 cwnd 加 1。

如果出现了超时，则令 ssthresh = cwnd/2，然后重新执行慢开始。

2. 快重传与快恢复
在接收方，要求每次接收到报文段都应该发送对已收到有序报文段的确认，例如已经接收到 M1 和 M2，此时收到 M4，应当发送对 M2 的确认。

在发送方，如果收到三个重复确认，那么可以确认下一个报文段丢失，例如收到三个 M2 ，则 M3 丢失。此时执行快重传，立即重传下一个报文段。

在这种情况下，只是丢失个别报文段，而不是网络拥塞，因此执行快恢复，令 ssthresh = cwnd/2 ，cwnd = ssthresh，注意到此时直接进入拥塞避免。

> TCP 并不能保证数据一定会被对方接收到，因为这是不可能的。TCP 能够做到的是，如果有可能，就把数据递送到接收方，否则就（通过放弃重传并且中断连接这一手段）通知用户。因此准确说 TCP 也不是 100% 可靠的协议，它所能提供的是数据的可靠递送或故障的可靠通知。


### TCP 三次握手

SYN 是 synchronize sequence numbers (同步序列号) 的缩写
ACK 是 acknowledgment (确认)的缩写。

Flags 表示 TCP 报文段 header 信息中的一些缩写标识：S 代表 SYN，. 代表ACK，P 代表PUSH，

输出三次握手命令行
sudo tcpdump -c 3 -i en3 -nS host 23.63.125.15

数据段的序号、确认序号以及滑动窗口大小都在这个过程中完成。socket 编程中，客户端执行 connect() 时，将触发三次握手。
三次握手的目的是连接服务器指定端口，建立 TCP 连接，并同步连接双方的序列号和确认号，交换 TCP 窗口大小信息。
![c27141caf5004fa9927c41506139f16c](/assets/img/iOS-network/CABDFB0C-73B6-4B3C-8074-88ECB8826513.png)

**为什么需要三次握手**
这是因为在网络请求中，我们应该时刻记住：“网络是不可靠的，数据包是可能丢失的”。假设没有第三次确认，客户端向服务端发送了 SYN，请求建立连接。由于延迟，服务端没有及时收到这个包。于是客户端重发送一个 SYN 包，这两个包的序列号显然是相同的。
假设服务端接收到了第二个 SYN 包，建立了通信，一段时间后通信结束，连接被关闭。这时候最初被发送的 SYN 包刚刚抵达服务端，服务端又会发送一次 ACK 确认。由于两次握手就建立了连接，此时的服务端就会建立一个新的连接，然而客户端觉得自己并没有请求建立连接，所以就不会向服务端发送数据。从而导致服务端建立了一个空的连接，白白浪费资源。
在三次握手的情况下，服务端直到收到客户端的应答后才会建立连接。因此在上述情况下，客户端会接受到一个相同的 ACK 包，这时候它会抛弃这个数据包，不会和服务端进行第三次握手，因此避免了服务端建立空的连接。

数据传输：
一旦建立了连接，双方就可以互发数据了。发送端所发出的每个报文段都有一个序列号，这个序列号与当下已传送的字节总数有关。接收端会针对已接收的数据包向源端发送确认报文，确认信息同样是由报文 header 所携带的 ACK。整个机制是双向运转的。

### TCP 四次挥手

TCP连接的拆除需要发送四个包，因此称为四次挥手(Four-way handshake)，也叫做改进的三次握手。客户端或服务器均可主动发起挥手动作，在 socket 编程中，任何一方执行 close() 操作即可产生挥手操作。
![3ef161f29beecd2b8ddad41816a846a7](/assets/img/iOS-network/280BEC7F-350A-4E6D-825A-13361EE3C52F.png)

**为什么 TCP 建立连接需要三次，而释放连接则需要四次？**
因为 TCP 是全双工模式，客户端请求关闭连接后，客户端向服务端的连接关闭（一二次挥手），服务端继续传输之前没传完的数据给客户端（数据传输），服务端向客户端的连接关闭（三四次挥手）。所以 TCP 释放连接时服务器的 ACK 和 FIN 是分开发送的（中间隔着数据传输），而 TCP 建立连接时服务器的 ACK 和 SYN 是一起发送的（第二次握手），所以 TCP 建立连接需要三次，而释放连接则需要四次。


### 下图中展示了 TCP 协议的 socket 交互流程
![591b1318e6832dd6c417f5090bd93718](/assets/img/iOS-network/97BB72DD-63F8-4132-96B5-B1D617056FD8.png)


### 端口号
标准的端口号由 Internet 号码分配机构(IANA)分配。这组数字被划分为特定范围，包括
熟知端口号(0 - 1023)、注册端口号(1024 - 49151)和动态/私有端口号(49152 - 65535)。


### UDP （User Datagram Protocol）
UDP 不提供复杂的控制机制，利用 IP 提供面向无连接的通信服务。传输途中即使出现丢包，UDP 也不负责重发，甚至当出现包的到达顺序乱掉时也没有纠正的功能。UDP 面向无连接，它可以随时发送数据。

主要用于以下几个方面：

包总量较少的通信 (DNS、SNMP等)
视频、音频等多媒体通信 (即时通信)
限定于 LAN 等特定网络中的应用通信
广播通信 (广播、多播)

UDP 首部:
![cfc569a7c79d35b9cbb22e86593bce23](/assets/img/iOS-network/49777F81-536C-462C-8DC7-933D32B1D7A9.png)

    
### TCP && UDP区别
TCP 提供可靠的通信传输，而 UDP 则常被用于让广播和细节控制交给应用的通信传输。

TCP 用于传输层有必要实现可靠传输的情况，由于它是面向有连接并具备顺序控制，重发控制等机制的，所以它可以为应用提供可靠传输。
UDP 主要用于那些对高速传输和实时性有较高要求的通信或广播通信。RIP、DHCP 等基于广播的协议也要依赖于 UDP。

### TLS（Transport Layer Security） 传输层协议

### HTTP (HyperText Transfer Protocol，超文本传输协议)

### HTTP 状态码
服务器返回的 响应报文 中第一行为状态行，包含了状态码以及原因短语，用来告知客户端请求的结果。
![8b86a635ac2e7ffd574097fb71a62b6d](/assets/img/iOS-network/1E55BE47-E600-49FF-BAA4-1DA6EF513EC9.png)

304 缓存
400 Bad Request ：请求报文中存在语法错误。
401 Unauthorized ：该状态码表示发送的请求需要有认证信息（BASIC 认证、DIGEST 认证）。如果之前已进行过一次请求，则表示用户认证失败。
403 Forbidden ：请求被拒绝，服务器端没有必要给出拒绝的详细理由。
404 Not Found
414 url的长度超出的服务器能处理的长度

![04641c6f0769f7e9c255f84ab3fcc688](/assets/img/iOS-network/22A0108F-B974-4CF1-B826-D0CE3878A791.png)

### HTTP 缓存控制
![3b178108f49ee2c20c341bbe004e354a](/assets/img/iOS-network/B45656D5-E93F-4D8E-BD32-C3414F954716.png)
需要兼容 HTTP1.0 的时候需要使用 Expires，不然可以考虑直接使用 Cache-Control。
需要处理一秒内多次修改的情况，或者其他 Last-Modified 处理不了的情况，才使用 ETag，否则使用 Last-Modified。
对于所有可缓存资源，需要指定一个 Expires 或 Cache-Control，同时指定 Last-Modified 或者 Etag。
可以通过标识文件版本名、加长缓存时间的方式来减少 304 响应。


### Cookie
安全请求首部
![f3d70fefb2a8c6dc6baa2cb79e658d6e](/assets/img/iOS-network/A9458466-E57D-412C-BC01-6E580C3DBF1B.png)
安全响应首部
![d58f4cc9360e935dd5fe99fb8fa6e3a0](/assets/img/iOS-network/05C8F39C-D88E-45D4-857C-C4CD53E8B7ED.png)


### 幂等性
幂等的 HTTP 方法，同样的请求被执行一次与连续执行多次的效果是一样的，服务器的状态也是一样的。换句话说就是，幂等方法不应该具有副作用（统计用途除外）。在正确实现的条件下，GET，HEAD，PUT 和 DELETE 等方法都是幂等的，而 POST 方法不是。所有的安全方法也都是幂等的。



### 提高 HTTP 性能
1. 并行连接
通过多条 TCP 连接发起并发的 HTTP 请求。

2. 持久连接
重用 TCP 连接，以消除连接及关闭的时延。 持久连接（HTTP Persistent Connections），也称为 HTTP keep-alive 或者 HTTP connection reuse 。

在 HTTP/1.1 中，所有的连接默认都是持久连接。但是服务器端不一定都能够支持持久连接，所以除了服务端，客户端也需要支持持久连接。

3. 管道化连接
通过共享的 TCP 连接发起并发的 HTTP 请求。

持久连接使得多数请求以管线化（pipelining）方式发送成为可能。以前发送请求后需要等待并收到响应，才能发送下一个请求。管线化技术出现后，不用等待响应，直接发送下一个请求。

比如当请求一个包含 10 张图片的 HTML Web 页面，与挨个连接相比，用持久连接可以让请求更快结束。而管线化技术则比持久连接还要快。请求数越多，时间差就越明显。

4. 复用的连接
交替传送请求和响应报文（实验阶段）。


### HTTP/1.0 与 HTTP/1.1 的区别

HTTP/1.1 默认是持久连接
HTTP/1.1 支持管道化处理
HTTP/1.1 支持虚拟主机
HTTP/1.1 新增状态码 100
HTTP/1.1 支持分块传输编码
HTTP/1.1 新增缓存处理指令 max-age


### HTTPS
HTTPS 请求首先需要进行 SSL/TLS 握手，其流程如下:

客户端发送 Client Hello，携带随机数、支持的加密算法等信息;
服务端收到请求后，选择合适的加密算法，连同公钥证书、随机数等信息返回给客户端;
客户端检验服务端证书的合法性，计算产生随机数并用证书公钥加密发送给服务端;
服务端通过私钥获取随机数信息，基于之前的交互信息计算得到协商密钥并通知给客户端;
客户端验证服务端发送的数据和密钥，通过后双方握手完成，开始进行加密通信。


### HTTPS
 SSL（Secure Sockets Layer 安全套接层）/ TLS (Transport Layer Security 安全层传输协议

通过使用 SSL，HTTPS 不仅能保证密文传输，重要的是还可以做到验证通信方的身份，保证报文的完整性。完美的解决了 HTTP 在安全性上的三大缺陷。

HTTPS 采用混合的加密机制，使用公开密钥加密用于传输对称密钥，之后使用对称密钥加密进行通信。（下图中的 Session Key 就是对称密钥）

HTTPS 中的 TLS / SSL 协议
![aa96fe1336eeeea937406c17faff4499](/assets/img/iOS-network/2BDCC6E7-B31B-4720-A0C0-3D70BD19A36D.png)
TLS/SSL 协议位于应用层和传输层 TCP 协议之间。TLS 粗略的划分又可以分为 2 层：
 * 靠近应用层的握手协议 TLS Handshaking Protocols
 * 靠近 TCP 的记录层协议 TLS Record Protocol
TLS 能保证信息安全和完整性的协议是记录层协议， 

TLS 握手协议还能细分为 5 个子协议：
* change_cipher_spec (在 TLS 1.3 中这个协议已经删除，为了兼容 TLS 老版本，可能还会存在)
* alert 
* handshake（握手协议）
* application_data（应用数据协议）
* heartbeat (这个是 TLS 1.3 新加的，TLS 1.3 之前的版本没有这个协议)
 TLS 记录层协议在整个 TLS 协议中的定位如下：
* 封装处理 TLS 上层(握手层)中的平行子协议(TLS 1.3 中是 5 个子协议，TLS 1.2 及更老的版本是 4 个子协议)，加上消息头，打包往下传递给 TCP 处理。
* 对上层应用数据协议进行密码保护，对其他的子协议只是简单封装(即不加密)


握手协议由多个子消息构成，服务端和客户端第一次完成一次握手需要 2-RTT。
握手协议的目的是为了双方协商出密码块，这个密码块会交给 TLS 记录层进行密钥加密。也就是说握手协议达成的“共识”(密码块)是整个 TLS 和 HTTPS 安全的基础。

先来看看一个请求从零开始，完整的需要多少个 RTT。假设访问一个 HTTPS 网站，用户从 HTTP 开始访问，到收到第一个 HTTPS 的 Response，大概需要经历一下几个步骤(以目前最主流的 TLS 1.2 为例)：
![63c8eb92d13eb47a4608ff2cee74ea06](/assets/img/iOS-network/3805DB82-9DA8-4D2E-8E18-6FC4F9DDE3A7.png)

完整的流程

![0573111d1d2fb40fb485d7e352d3e3c5](/assets/img/iOS-network/0B4C98D3-6F16-41DD-8B4D-2FAE281477EE.png)

一般有各种缓存不会经历上面每一步。如果有各种缓存，并且有 HSTS 策略，所以用户每次访问网页都必须要经历的流程如下：
![78191c49d2bb3c7e8a24685e5faae2c1](/assets/img/iOS-network/C83DA622-3E80-40BB-A62D-50DAF8C5F4A0.png)

公钥加密技术生成共享密钥——即协商出 TLS 记录层加密和完整性保护需要用到的 Security Parameters 加密参数。协商的过程中还必须要保证网络中传输的信息不能被篡改，伪造


![b753ed29a4ee2bd55f08eb7bd80f4a89](/assets/img/iOS-network/445644B3-ED53-47E8-A3F6-4068075FCCFC.png)
会话秘钥 = random S + random C + 前主秘钥


### HTTP/1.1 与 HTTP/2.0 的区别

1. 多路复用
HTTP/2.0 使用多路复用技术，同一个 TCP 连接可以处理多个请求。

2. 首部压缩
HTTP/1.1 的首部带有大量信息，而且每次都要重复发送。HTTP/2.0 要求通讯双方各自缓存一份首部字段表，从而避免了重复传输。

3. 服务端推送
HTTP/2.0 在客户端请求一个资源时，会把相关的资源一起发送给客户端，客户端就不需要再次发起请求了。例如客户端请求 index.html 页面，服务端就把 index.js 一起发给客户端。

4. 二进制格式
HTTP/1.1 的解析是基于文本的，而 HTTP/2.0 采用二进制格式。

5. 可以取消某一次请求，同时保证 TCP 连接还打开着，可以被其他请求使用。
6. 流量控制（Flow Control）


# HTTP/2 
是一个彻彻底底的二进制协议，头信息和数据包体都是二进制的，统称为“帧”。对比 HTTP/1.1 ，在 HTTP/1.1 中，头信息是文本编码(ASCII编码)，数据包体可以是二进制也可以是文本。使用二进制作为协议实现方式的好处，更加灵活。在 HTTP/2 中定义了 10 种不同类型的帧。

由于 HTTP/2 的数据包是乱序发送的，因此在同一个连接里会收到不同请求的 response。不同的数据包携带了不同的标记，用来标识它属于哪个 response。

HTTP/2 把每个 request 和 response 的数据包称为一个数据流(stream)。每个数据流都有自己全局唯一的编号。每个数据包在传输过程中都需要标记它属于哪个数据流 ID。规定，客户端发出的数据流，ID 一律为奇数，服务器发出的，ID 为偶数。

数据流在发送中的任意时刻，客户端和服务器都可以发送信号(RST_STREAM 帧)，取消这个数据流。HTTP/1.1 中想要取消数据流的唯一方法，就是关闭 TCP 连接。而 HTTP/2 可以取消某一次请求，同时保证 TCP 连接还打开着，可以被其他请求使用。


hTTP 协议不带有状态，每次请求都必须附上所有信息。所以，请求的很多字段都是重复的，比如 Cookie 和 User Agent，每次请求即使是完全一样的内容，依旧必须每次都携带，这会浪费很多带宽，也影响速度。HTTP/1.1 虽然可以压缩请求体，但是不能压缩消息头。有时候消息头部很大。
HTTP/2 对这一点做了优化，引入了头信息压缩机制(header compression)。一方面，头信息使用 gzip 或 compress 压缩后再发送；另一方面，客户端和服务器同时维护一张头信息表，所有字段都会存入这个表，生成一个索引号，以后就不发送同样字段了，只发送索引号，这样就提高速度了。
头部压缩大概可能有 95% 左右的提升，HTTP/1.1 统计的平均响应头大小有 500 个字节左右，而 HTTP/2 的平均响应头大小只有 20 多个字节，提升比较大。


### HTTP 2.0的协商机制
NPN（Next Protocol Negotiation）是一个 TLS 扩展，由 Google 在开发 SPDY 协议时提出。随着 SPDY 被 HTTP/2 取代，NPN 也被修订为 ALPN（Application Layer Protocol Negotiation，应用层协议协商）。二者目标一致，但实现细节不一样，相互不兼容。以下是它们主要差别：
 * NPN 是服务端发送所支持的 HTTP 协议列表，由客户端选择；而 ALPN 是客户端发送所支持的 HTTP 协议列表，由服务端选择；
 * NPN 的协商结果是在 Change Cipher Spec 之后加密发送给服务端；而 ALPN 的协商结果是通过 Server Hello 明文发给客户端
 
目前很多地方开始停止对NPN的支持，仅支持 ALPN， ALPN 是 TLS 的一个扩展，  ALPN 协商结果是在 Client hello 和 Server hello 中显示的，那就先来看一下Client hello（基于TLS，在say hello中进行协商）
![3211f93d9de5d22c48d2aea1490f88dc](/assets/img/iOS-network/C24F0AD9-998B-41D9-91A3-518A4F7C5D2E.png)
![6865e4ed9ca152ad587da62d25c5cb8d](/assets/img/iOS-network/75BE8A89-392C-4CF5-8F7F-B915B674C036.png)



客户端是使用 h2c 还是  h2，它们可以说是 HTTP2.0的两个版本，h2 是使用 TLS 的HTTP2.0协议，h2c是运行在明文 TCP 协议上的 HTTP2.0协议。浏览器目前只支持h2，也就是说必须基于HTTPS部署，但是客户端可以不部署HTTPS(浏览器等常用的都已强制规定http2必须使用TLS)  

什么是h2c：
h2c 利用的是HTTP Upgrade 机制，客户端会发送一个 http 1.1的请求到服务端，这个请求中包含了 http2的升级字段，
```
GET /default.htm HTTP/1.1
  Host: server.example.com
  Connection: Upgrade, HTTP2-Settings
  Upgrade: h2c
  HTTP2-Settings: <base64url encoding of HTTP/2 SETTINGS payload>
```
服务端收到这个请求后，如果支持 Upgrade 中 列举的协议，这里是 h2c，就会返回支持的响应, 不支持的话，服务器会返回一个不包含 Upgrade 的报头字段的响应


HTTP/2 会发送有着不同类型的二进制帧，但他们都有如下的公共字段：Type, Length, Flags, Stream Identifier 和 frame payload。本规范中一共定义了 10 种不同的帧，其中最基础的两种分别对应于 HTTP 1.1 的 DATA 和 HEADERS， 所有帧都以固定的 9 字节大小的头作为帧开始，后跟可变长度的有效载荷 payload。
[流量控制原则：, stream 优先级](https://halfrost.com/http2-http-frames/)



### http3.0 
HTTP/2 使用了多路复用，一般来说同一域名下只需要使用一个 TCP 连接。但当这个连接中出现了丢包的情况，那就会导致 HTTP/2 的表现情况反倒不如 HTTP/1 了。
因为在出现丢包的情况下，整个 TCP 都要开始等待重传，也就导致了后面的所有数据都被阻塞了。但是对于 HTTP/1.1 来说，可以开启多个 TCP 连接，出现这种情况反到只会影响其中一个连接，剩余的 TCP 连接还可以正常传输数据。
Google 就更起炉灶搞了一个基于 UDP 协议的 QUIC（读quick） 协议，并且使用在了 HTTP/3 上，HTTP/3 之前名为 HTTP-over-QUIC

多路复用：
加密认证的报文：同 HTTP2.0 一样，同一条 QUIC 连接上可以创建多个 stream，来发送多个 HTTP 请求，但是，QUIC 是基于 UDP 的，一个连接上的多个 stream 之间没有依赖。另外 QUIC 在移动端的表现也会比 TCP 好。因为 TCP 是基于 IP 和端口去识别连接的，这种方式在多变的移动端网络环境下是很脆弱的。但是 QUIC 是通过 ID 的方式去识别一个连接，不管你网络环境如何变化，只要 ID 不变，就能迅速重连上
向前纠错机制：我要发送三个包，那么协议会算出这三个包的异或值并单独发出一个校验包，也就是总共发出了四个包。当出现其中的非校验包丢包的情况时，可以通过另外三个包计算出丢失的数据包的内容。

### http3 vs http2
*相似之处*
这两个协议为客户端提供了几乎相同的功能集。
- 两者都提供数据流
- 两者都提供服务器推送
- 两者都有头部压缩，QPACK与HPACK的设计非常类似
- 两者都通过单一连接上的数据流提供复用
- 两者都提供数据流的优先度设置

**不同之处**
两个协议的主要不同点在于细节，不同之处主要由HTTP/3使用的QUIC带来。
- 得益于QUIC的0-RTT握手，HTTP/3可以提供更好的早期数据支持，而TCP快速打开和TLS通常只能传输更少的数据，且经常存在问题。
- 得益于QUIC，HTTP/3的握手速度比TCP+TLS快得多。
- HTTP/3不存在明文的不安全版本。尽管在互联网上很少见，HTTP/2还是可以不配合HTTPS来实现和使用。
- 通过ALPN拓展，HTTP/2可以直接在TLS握手时进行协商。HTTP/3基于QUIC，所以需要凭借响应中的 Alt-Svc: 头部来向客户端宣告。

**http3特点**
[链接](https://zhuanlan.zhihu.com/p/102561034)
* 改进的拥塞控制、可靠传输
* 快速握手
* 集成了 TLS 1.3 加密
* 多路复用
* 连接迁移:
TCP 是按照 4 要素（客户端 IP、端口, 服务器 IP、端口）确定一个连接的。而 QUIC 则是让客户端生成一个 Connection ID （64 位）来区别不同连接。只要 Connection ID 不变，连接就不需要重新建立，即便是客户端的网络发生变化。由于迁移客户端继续使用相同的会话密钥来加密和解密数据包，QUIC 还提供了迁移客户端的自动加密验证。



# DNS（Domain Name  System）解析
dns属于应用层协议

DNS解析本质上是localDNS解析，传入一个域名，返回一个iplist

**三种方式： **
一：
1：struct hostent	*gethostbyname(const char *); // ipv4
2：struct hostent	*gethostbyname2(const char *, int); // ipv6

缺点： 进行网络切换的时候小概率卡死，自测十次有一两次左右。
    在本地的LocalDns被破坏的时候会必卡死30秒，然后返回n

二：

常用到的gethostbyname(3)和gethostbyaddr(3)函数以外, Linux(以及其它UNIX/UNIX-like系统)还提供了一套用于在底层处理DNS相关问题的函数(这里所说的底层仅是相对gethostbyname和gethostbyaddr两个函数而言). 这套函数被称为地址解析函数(resolver functions)。曾经尝试过这个方式...
```
int		res_query __P((const char *, int, int, u_char *, int));
函数原型为：
int res_query(const char *dname, int class, int type, unsigned char *answer, int anslen)
```
复制代码这个方式需要在项目中添加libresolv.tbd库，因为要依赖于库中的函数去解析
优点：没有缓存， 在LocalDns被破坏掉的情况下能及时响应不会延迟
缺点：在进行网络切换时候3G/4G切wify高概率出现卡死 这一个缺点是比较致命的，所以没有再继续使用。

三：
苹果原生的DNS解析
优点：在网络切换时候不会卡顿。
缺点：在本地DNS被破坏的情况下会出现卡死的现象(卡30s)

```
Boolean CFHostStartInfoResolution (CFHostRef theHost, CFHostInfoType info, CFStreamError *error);
/// 代码
Boolean result,bResolved;
    CFHostRef hostRef;
    CFArrayRef addresses = NULL;
    NSMutableArray * ipsArr = [[NSMutableArray alloc] init];

    CFStringRef hostNameRef = CFStringCreateWithCString(kCFAllocatorDefault, "www.meitu.com", kCFStringEncodingASCII);
    
    hostRef = CFHostCreateWithName(kCFAllocatorDefault, hostNameRef);
    CFAbsoluteTime start = CFAbsoluteTimeGetCurrent();
    result = CFHostStartInfoResolution(hostRef, kCFHostAddresses, NULL);
    if (result == TRUE) {
        addresses = CFHostGetAddressing(hostRef, &result);
    }
    bResolved = result == TRUE ? true : false;
    
    if(bResolved)
    {
        struct sockaddr_in* remoteAddr;
        for(int i = 0; i < CFArrayGetCount(addresses); i++)
        {
            CFDataRef saData = (CFDataRef)CFArrayGetValueAtIndex(addresses, i);
            remoteAddr = (struct sockaddr_in*)CFDataGetBytePtr(saData);
            
            if(remoteAddr != NULL)
            {
                //获取IP地址
                char ip[16];
                strcpy(ip, inet_ntoa(remoteAddr->sin_addr));
                NSString * ipStr = [NSString stringWithCString:ip encoding:NSUTF8StringEncoding];
                [ipsArr addObject:ipStr];
            }
        }
    }
    CFAbsoluteTime end = CFAbsoluteTimeGetCurrent();
    NSLog(@"33333 === ip === %@ === time cost: %0.3fs", ipsArr,end - start);
    CFRelease(hostNameRef);
    CFRelease(hostRef);
```

HTTP DNS 的概念，它的基本原理如下:

原本用户进行 DNS 解析是向运营商的 DNS 服务器发起 UDP 报文进行查询，而在 HTTP DNS 下，我们修改为用户带上待查询的域名和本机 IP 地址直接向 HTTP WEB 服务器发起 HTTP 请求，这个 HTTP WEB 将返回域名解析后的 IP 地址。

APS (App Transport Security)



### socket
Socket 其实并不是一个协议。它工作在 OSI 模型会话层（第5层），是为了方便大家直接使用更底层协议（一般是 TCP 或 UDP ）而存在的一个抽象层。Socket是对TCP/IP协议的封装，Socket本身并不是协议，而是一个调用接口(API)。

Socket在通讯过程中，服务端监听某个端口是否有连接请求，客户端向服务端发送连接请求，服务端收到连接请求向客户端发出接收消息，这样一个连接就建立起来了。客户端和服务端也都可以相互发送消息与对方进行通讯，直到双方连接断开。

一个简易的 Server 的流程如下：

1.建立连接，接受一个客户端连接。
2.接受请求，从网络中读取一条  请求报文。
3.处理请求，访问资源。
4.构建响应，创建带有 header 的  响应报文。
5.发送响应，传给客户端。
省略流程 3，大体的程序与调用的函数逻辑如下：

socket() 创建套接字
bind() 分配套接字地址
listen() 等待连接请求
accept() 允许连接请求
read()/write() 数据交换
close() 关闭连接


### gRPC

RPC（Remote Procedure Call—远程过程调用)
gRPC 是一个高性能、开源和通用的 RPC 框架，面向移动和 HTTP/2 设计。gRPC 基于 HTTP/2 标准设计，带来诸如双向流、流控、头部压缩、单 TCP 连接上的多复用请求等特。这些特性使得其在移动设备上表现更好，更省电和节省空间占用。

gRPC的流可以分为三类， 客户端流式发送、服务器流式返回以及客户端／服务器同时流式处理, 也就是单向流和双向流。

gRPC的一些特性：
 - 简单的服务定义
 - 跨语言和平台工作
 - 快速启动并扩展
 - 双向流媒体以及身份验证集成

使用GRPC几大核心步骤
1）Defining a service 定义服务(在.proto文件中这个是和后台交互的主要协议)
2）Generating grpc code 生成grpc代码
3）Writing a server 服务端编写一定的服务提供给客户端使用(类似接口)
4）Server implementation 服务的实现
5）Writing a client 编写客户端代码(集成grpc)
6）Calling an rpc 根据RPC协议(.proto文件约定的协议)进行代码(接口)调用
优点：客户端充分利用高级流和链接功能，从而有助于节省带宽、降低的TCP链接次数、节省CPU使用、和电池寿命。


