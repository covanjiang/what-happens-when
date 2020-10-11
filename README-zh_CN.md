## 当在浏览器中输入 “google.com” 时发生了什么？

非直译，增加了部分细节，原文：[what-happens-when](https://github.com/alex/what-happens-when) 。

这是一个共同协作的过程，正如原文所述，本文也有很多遗漏的细节，欢迎随时向我发送 “pull request”。

本文使用 [Creative Commons Zero](https://creativecommons.org/publicdomain/zero/1.0/) 协议发布。

------

- [当在浏览器中输入 “google.com” 时发生了什么？](#当在浏览器中输入-googlecom-时发生了什么)
  - [硬件端](#硬件端)
    - [当按下 ”g“ 键时](#当按下-g-键时)
    - [当按下 ”enter“ 键时](#当按下-enter-键时)
    - [操作系统接收到键盘按键按下的事件后](#操作系统接收到键盘按键按下的事件后)
  - [客户端发起请求](#客户端发起请求)
    - [解析地址栏里的输入](#解析地址栏里的输入)
    - [判断输入是 URL 还是待搜索的关键词](#判断输入是-url-还是待搜索的关键词)
    - [对输入中的 non-ASCII Unicode 字符进行 URL 编码](#对输入中的-non-ascii-unicode-字符进行-url-编码)
    - [检查 HSTS 列表](#检查-hsts-列表)
    - [DNS 查询](#dns-查询)
    - [ARP 过程](#arp-过程)
    - [打开一个套接字](#打开一个套接字)
    - [HTTPS 协议的 TLS 握手](#https-协议的-tls-握手)
  - [服务端回应请求](#服务端回应请求)
    - [请求处理](#请求处理)
  - [浏览器渲染](#浏览器渲染)
    - [浏览器架构](#浏览器架构)
    - [HTML 解析](#html-解析)
    - [CSS 解析](#css-解析)
    - [页面渲染](#页面渲染)
    - [GPU 渲染](#gpu-渲染)
    - [后期渲染与用户引发的处理](#后期渲染与用户引发的处理)

### 硬件端

#### 当按下 ”g“ 键时

当你按下 ”g“ 键后，浏览器会立即监听到该事件，继而触发自动完成机制，类似 onClick 事件。通过浏览器的算法，以及你是否处于无痕浏览模式，浏览器会在地址栏的下拉框内给出各种输入建议。大部分的算法会基于你的搜索历史，书签，浏览器缓存和一些互联网上的热门搜索给出最优的输入建议排序。在你输入 ”google.com“ 时浏览器会运行许多代码块，你的每一次按键都会让地址栏给出的输入建议更加的精确，甚至可能在你输入完整的内容之前就给出 ”google.com“ 的输入建议。

#### 当按下 ”enter“ 键时

以编码键盘为例，在 ”enter" 键被按下还未起来时，“enter” 键特定的电路闭合，这将允许少量的电流进入键盘的逻辑电路，该逻辑电路会扫描每个键的状态消除快速间歇性闭合的电噪，只识别处在稳定闭合或断开状态的键，再将 “enter” 转换成对应的键码值 13。键盘控制电路对该键码值进行编码，以传输给 CPU 使用。传输过程通常是通过一个通用串行总线（USB）或蓝牙连接进行，以前是通过 PS/2 或 ADB 连接传输的。CPU 接收到设备的中断请求后，将中断信息映射到中断处理器进行处理。下图说明外设传输一个中断给 CPU，图片来源：[Hämäläinen, T. (2017). NIRO ÅHMAN TESTING OF LINUX KERNEL MODULES](https://trepo.tuni.fi/bitstream/handle/123456789/24806/%c3%85hman.pdf?SEQuence=4&isAllowed=y).

<img src="https://github.com/covanjiang/what-happens-when/blob/master/image/interrupt.jpg?raw=true" alt="外设发起中断并传输给 CPU" style="zoom: 50%;" />

#### 操作系统接收到键盘按键按下的事件后

1. Windows - 一个 `WM_KEYDOWN` 消息被发往应用程序

   - HID（人机交互设备）把键盘按下的事件传输给 `KBDHID.sys` 驱动，该驱动把 HID 信号转换成扫描码 `VK_RETURN(0x0D)`。`KBDHID.sys` 驱动和 `KBDCLASS.sys` 驱动进行交互，这个驱动用安全的方式处理所有的键盘输入事件，然后去调用 `Win32K.sys`（在这之前可能会将信息消息传递给第三方键盘过滤器）。以上动作都是发生在内核模式下，大部分驱动都是运行在该模式下。下图说明用户模式组件与内核模式组件之间的通信。图片来源：[What is a Kernel in OS? What are the types of Kernel?](https://www.thewindowsclub.com/what-is-a-kernel-in-os-what-are-the-types-of-kernel)

     <img src="https://github.com/covanjiang/what-happens-when/blob/master/image/windows.png?raw=true" alt="用户模式组件与内核模式组件之间的通信图" style="zoom: 67%;" />

   - `Win32K.sys` 通过 `GetForegroundWindow()` 接口指出当前桌面哪个窗口是激活的，返回这个激活的窗口句柄 `window handle`。然后 Windows Message Pump 调用 `SendMessage(hWnd, WM_KEYDOWN, VK_RETURN, lParam)`。`SendMessage` 接口将消息添加到指定的窗口句柄 `hWnd` 的队列中，然后调用分配给该窗口句柄的主要消息处理方法 `WindowProc` 以处理队列中每个消息。`WindowProc` 有一个用于处理 `WM_KEYDOWN` 消息的处理器，该处理器会通过 `SendMessage` 第三个参数 `VK_RETURN` 判断用户按下了哪一个按键。`lParam` 是一个位掩码，用来指示有关按键的更多信息：重复计数，实际扫描码（可依赖于 OEM 厂商，不过通常不是 `VK_RETURN` ），是否有功能键（alt / shift / ctrl）被按下等等。

   - 一个窗口句柄会接收来自用户、系统内部、网络或其他线程发来的事件，这些事件可能并发同时送达该窗口。而 Windows Message Pump 机制在主线程上做一个死循环，不断从 “message pump” 中检查是否有消息到达，如果有消息抵达该窗口，则取出该消息并交付给窗口进程（Window Process）进行处理。

2. Mac OS X - 一个 `KeyDown` NSEvent 被发往应用程序

   中断信号在 I/O Kit kext 键盘驱动程序中触发一个中断事件，该驱动程序将信号转换为键码值，然后传递给 OS X `WindowServer`  进程，`WindowServer` 将这个事件通过 Mach 端口分发给合适的应用程序（激活的或正在监听的），这个事件会被放到应用程序的消息队列里。具有足够高权限的线程可以通过调用 `mach_ipc_dispatch` 函数从该队列中读取该事件。 这个过程通常是 NSEventType KeyDown 的 NSEvent 通过 NSApplication 主事件循环进行。

3. GNU / Linux - Xorg 服务端监听键码值

   当使用图形 X server 时，X server 将使用通用事件驱动程序 `evdev` 来获取按键，并使用特定的键映射规则将键编码重新映射为扫描码。当按下的键的扫描码映射完成后，X server 将字符发送到窗口管理器 `window manager`（DWM、metacity、i3），窗口管理器再把字符发送到激活的窗口，当前激活的窗口使用图形 API 把字符打印在输入框中。

### 客户端发起请求

#### 解析地址栏里的输入

浏览器对 URL 进行解析，提取 protocol/scheme、host、resource/path、port、queries 等值。

#### 判断输入是 URL 还是待搜索的关键词

当 protocol 或者 host 不合法时，浏览器将继续把地址栏的输入提供给浏览器指定的搜索引擎。大部分情况下，浏览器会附加特殊的字符串告诉搜索引擎输入来源于地址栏。

#### 对输入中的 non-ASCII Unicode 字符进行 URL 编码

浏览器先检查 URL 中的 hostname 是否含有非 ASCII Unicode 字符，如果有的话，浏览器会使用 [Punycode](https://en.wikipedia.org/wiki/Punycode)（国际化域名编码，是一种表示 Unicode 码和 ASCII 码的有限字符集）对 URL 中的 hostname 部分进行编码。

#### 检查 HSTS 列表

- 浏览器检查自带的预加载 HTST（HTTP Strict Transport Security）列表,这个列表列举了哪些网站只能通过 HTTPS 协议进行连接访问。如果地址栏的输入没有指定协议，则默认使用 HTTP 协议进行连接，如果地址栏的输入域名在 HTST 列表中，则会强制使用 HTTPS 协议进行连接。
- 即使一个网站不在 HSTS 列表里，也可以要求浏览器对自己使用 HSTS 政策进行访问。浏览器向服务端发出了第一个 HTTP 协议请求之后，服务端会返回浏览器一个响应，请求浏览器只使用 HTTPS 协议进行发送请求。然而，就是这第一个 HTTP 协议请求，也可能会使用户受到 [downgrade attack](http://en.wikipedia.org/wiki/SSL_stripping) 的威胁，所以现代浏览器都预置了 HSTS 列表。

#### DNS 查询

- 浏览器检查地址栏输入的域名是否缓存过（谷歌查看 DNS 缓存：[Chrome Cache](chrome://net-internals/#dns)），如果缓存中不存在，则调用 `gethostbyname` 函数进行查询，不同的系统所使用的函数不同。`gethostbyname` 在试图进行 DNS 解析之前会检查本地的 hosts 文件，不同的系统 hosts 的位置不同。

- 如果既没有缓存，也没在 hosts 中找到，`gethostbyname` 将会向 DNS 服务端发送一条 DNS 查询请求。DNS 服务端是由网络通信栈提供的，通常是本地路由器或者 ISP（互联网服务供应商） 的缓存 DNS 服务端。

- 如果 DNS 服务端和我们的主机在同一个子网内，系统会按照下面的 ARP 过程对 DNS 服务端进行 ARP 查询。如果 DNS 服务端和我们的主机在不同的子网，系统会按照下面的 ARP 过程对默认网关进行查询。

- 通过 DNS 服务端找到域名所解析的 IP 地址。下图说明 DNS 解析域名流程，图片来源：[大航海计划 Affiliate基础 学习笔记（一）](https://www.sunlightnotes.com/9.html)

  <img src="https://github.com/covanjiang/what-happens-when/blob/master/image/DNS.png?raw=true" alt="DNS 解析域名流程" style="zoom: 67%;" />

#### ARP 过程

ARP（ Address Resolution Protocol）是将 IP 地址解析为以太网 MAC 地址的协议。下图说明 ARP 地址解析过程，图片来源：[ARP地址解析原理_changsoon](https://blog.csdn.net/u011784495/article/details/71729492)

<img src="https://github.com/covanjiang/what-happens-when/blob/master/image/ARP.png?raw=true" alt="ARP 地址解析示例" style="zoom: 50%;" />

- 首先检查 ARP 缓存中是否存在目标 IP 的 ARP 信息

  - 如果存在，将直接返回目标 IP 对应的 MAC 地址；
  - 如果不存在，则查找路由表，判断目标 IP 是否在本地路由表中的某个子网内。
    - 如果在，则直接使用与该子网关联的接口；
    - 如果不在，则使用具有默认网关子网的接口。

- 目标主机 B 的 IP 在本地路由某个子网内，主机 A 先缓存数据报文，然后以广播方式发送一个 ARP 请求，目标 IP 为主机 B 的 IP，目标 MAC 为广播地址，因为 ARP 请求是以广播方式发送，所以该网段上的所有主机都可以接收到该请求包，但只有主机 B 才会对该请求进行处理，返回一个带有主机 B 的 MAC 地址的 ARP 响应。主机 A 在接收到主机 B 的 MAC 地址后，会重新封装数据报文发送给主机 B。

- 目标主机 B 的 IP 与本机 A 的 IP 在不同的网段下

  - 如果主机 A 不知道网关的 MAC 地址，则先在本网段发送一个 ARP 广播，目标 IP 为网关 IP，以获取网关的 MAC 地址。
  - 在获取网关的 MAC 地址后，主机 A 再发送一个 ARP 请求报文给网关，报文的目标 IP 为主机 B 的 IP，目标 MAC 地址为网关的 MAC 地址
    - 如果网关缓存了主机 B 的 MAC 信息，则网关直接修改来自主机 A 的请求报文中的目标 MAC 地址为主机 B 的 MAC 地址，然后再转发给主机 B；
    - 如果网关没有查到缓存，则网关会向其他网段发送一个新的 ARP 广播，目标 IP 为主机 B 的 IP，当网关接收到主机 B 返回的 ARP 响应后，再将来自主机 A 请求报文中的目标 MAC 地址改为主机 B 的 MAC，然后转发给主机 B。

- 发送一个两层的 ARP 请求报文（数据链路层）：

  ```shell
  # ARP Request:
  Sender MAC: interface:mac:address:here
  Sender IP: interface.ip.goes.here
  # 目标 MAC 地址为广播地址，任意网段节点都可以接受到
  Target MAC: FF:FF:FF:FF:FF:FF (Broadcast)
  Target IP: target.ip.goes.here
  ```

------

根据计算机和路由器之间的硬件类型：

- 直连路由器
  
- 如果计算机直接连接到路由器，路由器会直接以 ARP 形式回复一个响应。
  
- 集线器
  
- 如果计算机连接到集线器上，则集线器会把 ARP 请求向所有其它端口广播；如果路由器和计算机连接在同一条线上，路由器将以 ARP 形式回复一个响应。
  
- 交换机

  - 如果计算机连接到交换机上，交换机会检查本地 CAM/MAC 表，看看哪个端口有我们需要查找的 MAC 地址，如果有，交换机会向有我们想要查询的 MAC 地址的那个端口发送 ARP 请求；如果没有找到，交换机会将 ARP 请求重新光波导其他所有端口。
  - 如果路由器和计算机连接在同一条线上，路由器将以 ARP 形式回复一个响应。

  ARP 形式响应：

  ```shell
  # ARP Reply:
  Sender MAC: target:mac:address:here
  Sender IP: target.ip.goes.here
  Target MAC: interface:mac:address:here
  Target IP: interface.ip.goes.here
  ```

------

现在我们有了 DNS 服务端或者默认网关的 IP 地址，我们可以继续 DNS 请求了：

- DNS 客户端使用 1023 以上的源端口在 DNS 服务端上的 UDP 53 端口上建立套接字。如果响应太大，则将使用TCP。
- 如果本地/ISP DNS 服务端没有找到结果，则将请求递归搜索，该搜索将沿 DNS 服务端列表向上流动，直到到达 SOA（起始授权机构），如果找到会把结果返回。

#### 打开一个套接字

一旦浏览器收到目标服务端的 IP 地址，它将从 URL 中获取端口号（HTTP 协议默认端口 80，HTTPS 默认端口 443），并调用名为 `socket` 的系统库函数并请求一个 TCP 套接字流 - `AF_INET/AF_INET6` 和 `SOCK_STREAM`。

- 这个请求首先传递给传输层，在传输层被封装成 TCP segment。将目标端口加入头部，并从系统内核的动态端口范围内选择源端口（在 Linux 中为 ip_local_port_range）。
- 这个 TCP segment 会发往网络层，网络层在其头部上增加一个 IP 标头，里面包含了目标服务端的IP地址以及本机的IP地址，把它封装成一个 IP packet（数据包）。
- 这个 IP packet 进入到链路层，链路层会增加一个 frame 标头，里面包含了本机 NIC（网络接口控制器/内置网卡）的 MAC 地址以及网关（本地路由器）的 MAC 地址。如前面所说，如果内核不知道网关的 MAC 地址，它必须进行 ARP 广播来进行查询。

------

这样，TCP 数据包就已经准备好了，可以通过以下任一方式进行传输：[Ethernet（以太网）](http://en.wikipedia.org/wiki/IEEE_802.3)、[WiFi](https://en.wikipedia.org/wiki/IEEE_802.11)、[Cellular data network（蜂窝数据网络/移动网络）](https://en.wikipedia.org/wiki/Cellular_data_communication_protocol)。

- 对于大部分家庭网络和小型企业网络来说，封包会从本地计算机出发，经过本地网络，再通过调制解调器（MOdulator/DEModulator）把数字信号转换成模拟信号，使其适于在电话线路，有线电视光缆和无线电话线路上传输。在传输线路的另一端，是另外一个调制解调器，它把模拟信号转换回数字信号，交由下一个 [网络节点](https://en.wikipedia.org/wiki/Computer_network#Network_nodes) 处理。

- 大型企业和比较新的住宅通常使用光纤或直接以太网连接，这种情况下数据将保持数字状态直接传到下一个网络节点进行处理。


最终，数据包将到达管理本地子网的路由器。 从那里，它将继续前往自治系统（autonomous system's - AS）的边界路由器，其他 AS，最后到达目标服务端。 途中的每个路由器都将从 IP 标头中提取目标地址，并将其路由到适当的下一个目的地。 对于每个经过的路由器，IP 报头中的生存时间（time to live - TTL）字段减一。 如果 TTL 字段达到零或当前路由器的队列中没有空间（可能是由于网络拥塞），则该数据包将被丢弃。

------

上面的发送和接受过程在 TCP 连接后会发生很多次，在此之前需要进行 TCP 的三次握手进行连接，客户端执行 `connect()` 时触发三次握手，下图说明 TCP 的三次握手，图片来源：[两张动图-彻底明白TCP的三次握手与四次挥手](https://blog.csdn.net/qzcsu/article/details/72861891)

<img src="https://github.com/covanjiang/what-happens-when/blob/master/image/TCP_cnnect.png?raw=true" alt="TCP 的三次握手" style="zoom: 50%;" />

- 第一次握手（SYN=1，SEQ=xxx）：

  - 客户端选择一个初始序列号（ISN） xxx，保存在包头的序列号（SEQuence Number）字段里；
  - 客户端设置 SYN 标志为 1，表明客户端打算建立连接以及设置了序列号。

  发送完毕后，客户端进入 `SYN-SENT` 状态。

- 第二次握手（SYN=1，SEQ=yyy，ACK=1，ACKnum=xxx+1）：

  - 服务端选择一个初始序列号 yyy；

  - 服务端设置 SYN 标志为 1，表明自己设置了一个初始序列号；
  - 服务端将确认序号（Acknowledgement Number）设置为客户端的序列号加一，即 xxx+1。

  - 服务端设置 ACK 标志为 1，表明自己接收到客户端的第一个封包。

  发送完毕后，服务端端进入 `SYN-RCVD` 状态。

- 第三次握手（SYN=0，SEQ=xxx+1，ACK=1，ACKnum=yyy+1）

  - 客户端将自己的序列号加一，即 xxx+1；
  - 客户端设置 SYN 标志为 0；
  - 客户端将确认序号（Acknowledgement Number）设置为服务端的序列号加一，即 yyy+1；
  - 客户端设置 ACK 标志为 1，表明自己接收到服务端的确认包。

  发送完毕后，客户端进入 `ESTABLISHED` 状态，当服务端端接收到这个包时，也进入 `ESTABLISHED` 状态，TCP 握手结束。

- 数据传输：
  - 当一方发送了 N 个数据包之后，将自己的 SEQ 序列号也增加 N；
  - 另一方确认接收到这个数据包（或者一系列数据包）之后，它发送一个 ACK 包，ACK 的值设置为接收到的数据包的最后一个序列号。

------

数据传输完毕后，双方都可释放连接。最开始的时候，客户端和服务端都是处于 `ESTABLISHED` 状态，然后客户端主动关闭，服务端被动关闭，下图说明 TCP 的四次挥手，图片来源：[两张动图-彻底明白TCP的三次握手与四次挥手](https://blog.csdn.net/qzcsu/article/details/72861891)

<img src="https://github.com/covanjiang/what-happens-when/blob/master/image/TCP_close.png?raw=true" alt="TCP 的四次挥手" style="zoom:50%;" />

- 第一次挥手（FIN=1，SEQ=uuu）：

  - 客户端的设置序列号为 uuu（等于前面已经传送过来的数据的最后一个字节的序列号加一）；
  - 客户端设置 FIN 标志为 1，表明客户端打算关闭此次连接以及设置了序列号。

  发送完毕后，客户端进入 `FIN-WAIT-1` 状态。

- 第二次挥手（SEQ=vvv，ACK=1，ACKnum=uuu+1）：

  - 服务端设置序列号 vvv（等于前面已经传送过来的数据的最后一个字节的序列号加一）；

  - 服务端将确认序号（Acknowledgement Number）设置为客户端的序列号加一，即 uuu+1。

  - 服务端设置 ACK 标志为 1，表明自己接收到客户端的第一个封包。

  发送完毕后，服务端端进入 `CLOSE-WAIT` 状态，并通知上层应用程序即将关闭连接，客户端接收后进入`FIN-WAIT-2` 状态，这个状态下，服务端依然可以发送数据给客户端。

- 第三次挥手（FIN=1，SEQ=www，ACK=1，ACKnum=uuu+1）

  - 服务端的序列号为 www（假设服务端有发送了一些数据给客户端）；
  - 服务端设置 FIN 标志为 1，表明服务端已经可以释放连接；
  - 服务端将确认序号（Acknowledgement Number）设置为客户端的序列号加一，即 uuu+1；
  - 服务端设置 ACK 标志为 1。

  发送完毕后，服务端进入 `LAST-ACK` 状态，等待客户端的确认。

- 第四次挥手（SEQ=uuu+1，ACK=1，ACKnum=www+1）：

  - 客户端将自己的序列号加一，即 uuu+1；
  - 客户端将确认序号（Acknowledgement Number）设置为服务端的序列号加一，即 www+1；
  - 客户端设置 ACK 标志为 1，表明自己接收到服务端的确认释放包。

  发送完毕后，客户端进入 `TIME-ACK` 状态，等待 `2*MSL` 时间后才进入 `CLOSED` 状态。服务端在接收到客户端的确认包后，立即进入 `CLOSED` 状态。

#### HTTPS 协议的 TLS 握手

由于非对称加密相对来说是低效率的，所以一般非对称加密只用来建立安全的信道，而真实地数据加密都是使用的对称加密。下图说明 TLS 握手过程，图片来源：[TLS 握手过程](https://shunix.com/tls-handshake/)

<img src="https://github.com/covanjiang/what-happens-when/blob/master/image/TLS.jpg?raw=true" alt="TLS 握手过程" style="zoom: 50%;" />

- 第一次握手，客户端发出初始请求 ClientHello，带有以下数据
  - 客户端生成的一个随机数，用于后面生成对话密钥（Session key）；
  - 客户端支持的加密方式（Cipher Suite），供服务端选择；
  - 客户端支持的 SSL/TLS 协议版本号；
  - 客户端支持的压缩算法；
  - 缓存的 Session ID，如果存在，则会带上 Session ID 用来恢复上一次会话。

- 第二次握手，服务端响应初始请求 ServerHello，带有以下数据
  - 服务端的 SSL/TLS 协议版本号，如果客户端和服务端所支持的版本号不一样，则会关闭加密通信；
  - 服务端选择的加密方式；
  - 服务端生成的一个随机数，用于后面的对话密钥；
  - 服务端的公开证书，由数字认证机构（Certificate Authority - CA）签发，里面带有服务端的公钥；
    - 客户端在接收到服务端的证书后，会校验该证书的合法性，包括检测证书有效时间，以及证书中域名与当前会话域名是否匹配，并沿着信任链查找顶层证书颁发者是否是操作系统信任的 CA 机构。
    - 验证信任链的时候，需要用上一级证书的公钥对该证书里的签名进行解密，还原对应的摘要值；再使用该证书给出的信息去计算该证书的摘要值，最后对比两个摘要值是否相等，以此类推对整个证书链进行校验。

- 第三次握手，客户端在校验服务端的公开证书后进行回应，带有以下数据
  - 客户端使用服务端的公钥加密一个随机数（防止被窃听），生成另一个随机字符串（Premaster secret）；
    - 客户端已经获得了三个随机字符串，利用服务端选择的加密方式对该三个字符串进行加密，生成本次会话所用的对话密钥。
  - 编码改变通知，表示随后的信息都将用双方商定的加密方法和密钥发送；
  - 客户端握手结束通知，表示客户端的握手阶段已经结束。这一项同时也是前面发送的所有内容的 hash 值，用来供服务端校验。

- 第四次握手，服务端最后的回应
  - 服务端接收到客户端的第三个随机数之后，使用密钥进行解密，并结合之前两串随机字符串和加密方法，生成本次会话所用的对话密钥；
  - 编码改变通知，表示随后的信息都将用双方商定的加密方法和密钥发送；
  - 服务端握手结束通知，表示服务端的握手阶段已经结束。这一项同时也是前面发送的所有内容的 hash 值，用来供客户端校验。

- 从现在开始，接下来整个 TLS 会话都使用对称秘钥进行加密，传输应用层（HTTP）内容

### 服务端回应请求

#### 请求处理

HTTPD（HTTP Daemon）在服务端处理请求/响应。最常见的 HTTPD 有 Linux 上常用的 Apache 和 nginx，以及 Windows 上的 IIS。

- HTTPD 接收请求；
- 服务端把请求拆分为以下几个参数：
  - HTTP 请求方法（`GET`， `POST`， `HEAD`， `PUT`， `DELETE`， `CONNECT`， `OPTIONS`，`TRACE`）；
  - 域名；
  - 请求路径。
- 服务端验证是否已经配置了 google.com 的虚拟主机；
- 服务端验证 google.com 接受 GET 方法；
- 服务端验证该用户可以使用 GET 方法（根据 IP 地址，身份信息等）；
- 如果服务端安装了 URL 重写模块（例如 Apache 的 mod_rewrite 和 IIS 的 URL Rewrite），服务端会尝试匹配重写规则，如果匹配上的话，服务端会按照规则重写这个请求；
- 服务端根据请求信息获取相应的响应内容；
- 服务端会使用指定的处理程序进行分析处理捕获输出结果返回客户端。

### 浏览器渲染

当服务端提供了资源之后（HTML，CSS，JS，Image 等），浏览器会执行以下操作：

- 解析 - HTML，CSS，JS
- 渲染 - 构建 DOM 树 `->` 渲染 `->` 布局 `->` 绘制

#### 浏览器架构

- **用户界面**：包括地址栏、前进/后退按钮、书签菜单等。除了浏览器主窗口显示的您请求的页面外，其他显示的各个部分都属于用户界面；
- **浏览器引擎**：在用户界面和呈现引擎之间传送指令；
- **渲染引擎**：负责显示请求的内容。如果请求的内容是 HTML，它就负责解析 HTML 和 CSS 内容，并将解析后的内容显示在屏幕上；
- **网络组件**：用于网络调用，比如 HTTP 请求。其接口与平台无关，并为所有平台提供底层实现；
- **用户界面后端**：用于绘制基本的窗口小部件，比如组合框和窗口。其公开了与平台无关的通用接口，而在底层使用操作系统的用户界面方法；
- **Javascript 引擎**：用于解析和执行 JavaScript 代码；
- **数据存储**：这是持久层，浏览器需要在硬盘上保存各种数据，如 Cookie。新的 HTML 规范 (HTML5) 定义了“网络数据库”，这是一个完整但是轻便的浏览器内数据库。

和大多数浏览器不同，Chrome 浏览器的每个标签页都分别对应一个呈现引擎实例。每个标签页都是一个独立的进程。下图说明浏览器的主要组件，图片来源：[浏览器的工作原理：新式网络浏览器幕后揭秘](https://www.html5rocks.com/zh/tutorials/internals/howbrowserswork/)

<img src="https://github.com/covanjiang/what-happens-when/blob/master/image/layers.png?raw=true" alt="浏览器的主要组件" style="zoom:67%;" />

#### HTML 解析

渲染引擎从网络层取得请求的文档，通常文档会以 8kB 大小进行分块传输。

HTML 解析器的主要工作是对 HTML 文档进行解析，生成解析树。解析树是以 DOM 元素以及属性为节点的树。DOM 是文档对象模型（Document Object Model）的缩写，它是 HTML 文档的对象表示，同时也是 HTML 元素面向外部（如 Javascript）的接口。树的根部是 “Document” 对象。整个 DOM 和 HTML 文档几乎是一对一的关系。下图说明 HTML DOM 树的构建，图片来源：[浏览器的工作原理：新式网络浏览器幕后揭秘](https://www.html5rocks.com/zh/tutorials/internals/howbrowserswork/)

<img src="https://github.com/covanjiang/what-happens-when/blob/master/image/HTML_DOM.gif?raw=true" alt="HTML DOM 树的构建" style="zoom: 67%;" />

HTML不能使用常见的自顶向下或自底向上方法来进行分析。主要原因有以下几点:

- 语言本身的“宽容”特性；
- HTML 本身可能是残缺的，对于常见的残缺，浏览器需要有传统的容错机制来支持它们；
- 解析过程需要反复。对于其他语言来说，源码不会在解析过程中发生变化，但是对于 HTML 来说，动态代码，例如脚本元素中包含的 document.write() 方法会在源码中添加内容，也就是说，解析过程实际上会改变输入的内容。

由于不能使用常用的解析技术，浏览器创造了专门用于解析 HTML 的解析器。解析算法在 HTML5 标准规范中有详细介绍，算法主要包含了两个阶段：标记化（tokenization）和树的构建。

------

解析结束之后，浏览器开始加载网页的外部资源（CSS，Image，Javascript 等）。此时浏览器把文档标记为可交互的（interactive），浏览器开始解析处于 “推迟（deferred）” 模式的脚本，也就是那些需要在文档解析完毕之后再执行的脚本。之后文档的状态会变为 “完成（complete）”，浏览器会触发 “加载（load）” 事件。注意解析 HTML 网页时永远不会出现 “无效语法（Invalid Syntax）” 错误，浏览器会修复所有错误内容，然后继续解析。

#### CSS 解析

- 根据 [CSS词法和句法](http://www.w3.org/TR/CSS2/grammar.html) 分析CSS文件和 `<style>` 标签包含的内容以及 style 属性的值；

- 每个CSS文件都被解析成一个样式表对象（`StyleSheet object`），这个对象里包含了带有选择器的 CSS 规则，和对应 CSS 语法的对象；

- CSS 解析器可能是自顶向下的，也可能是使用解析器生成器生成的自底向上的解析器。下图说明 CSS 解析，图片来源：[浏览器的工作原理：新式网络浏览器幕后揭秘](https://www.html5rocks.com/zh/tutorials/internals/howbrowserswork)

  <img src="https://github.com/covanjiang/what-happens-when/blob/master/image/CSS.png?raw=true" alt="CSS 解析" style="zoom: 67%;" />

#### 页面渲染

- 通过遍历 DOM 节点树创建一个 “Frame 树” 或“渲染树”，并计算每个节点的各个 CSS 样式值；
- 通过累加子节点的宽度，该节点的水平内边距（padding）、边框（border）和外边距（margin），自底向上的计算 “Frame 树” 中每个节点的首选（preferred）宽度；
- 通过自顶向下的给每个节点的子节点分配可行宽度，计算每个节点的实际宽度；
- 通过应用文字折行、累加子节点的高度和此节点的内边距（padding）、边框（border）和外边距（margin），自底向上的计算每个节点的高度。

------

1. 使用上面的计算结果构建每个节点的坐标；
2. 当存在元素使用 `floated`，位置有 `absolutely` 或 `relatively` 属性的时候，会有更多复杂的计算，详见 http://dev.w3.org/csswg/css2/ 和 http://www.w3.org/Style/CSS/current-work
3. 创建 layer 来表示页面中的哪些部分可以成组的被绘制，而不用被重新栅格化处理，每个帧对象都被分配给一个层；
4. 页面上的每个层都被分配了纹理；
5. 每个层的帧对象都会被遍历，计算机执行绘图命令绘制各个层，此过程可能由 CPU 执行栅格化处理，或者直接通过 D2D/SkiaGL 在 GPU 上绘制；
6. 上面所有步骤都可能利用到最近一次页面渲染时计算出来的各个值，这样可以减少不少计算量；
7. 计算出各个层的最终位置，一组命令由 Direct3D/OpenGL 发出，GPU 命令缓冲区清空，命令传至 GPU 并异步渲染，帧被送到 Window Server。

#### GPU 渲染

- 在渲染过程中，图形处理层可能使用通用用途的 `CPU`，也可能使用图形处理器 `GPU`
- 当使用 `GPU` 用于图形渲染时，图形驱动软件会把任务分成多个部分，这样可以充分利用 `GPU` 强大的并行计算能力，用于在渲染过程中进行大量的浮点计算。

#### 后期渲染与用户引发的处理

渲染完成后，由于某些计时机制（如 Google Doodle 动画）或用户交互（将查询键入搜索框并接收建议），浏览器将执行JavaScript 代码。 类似 Flash 或 Java 之类的插件也可以执行。 脚本可以触发执行其他网络请求，以及修改页面或其布局，从而触发另一轮的页面渲染和绘画。
