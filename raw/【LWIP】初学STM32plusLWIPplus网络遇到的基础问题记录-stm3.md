#### 目录

-   [一、前言](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#_2)
-   [二、问题汇总](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#_6)
-   -   [问题1：为什么要用以太网通信？](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#1_7)
    -   [问题2：既然要用以太网，那什么是以太网？](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#2_10)
    -   [问题3：IIC、SPI都有自己的协议，那以太网用什么协议通信呢？](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#3IICSPI_15)
    -   [问题4：TCP/IP又是什么？](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#4TCPIP_18)
    -   [问题5：为什么要对网络进行分层？分层有什么好处?](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#5_22)
    -   [问题6：网络分了几层？这几层该怎么理解？](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#6_28)
    -   [问题7：TCP/IP协议这么复杂，STM32就这么点空间，怎么放得下呢？](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#7TCPIPSTM32_66)
    -   [问题8：LwIP是什么？和TCP/IP有什么关系？](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#8LwIPTCPIP_69)
    -   [问题9：让你用LwIP协议栈编程，这个LwIP协议栈又是什么意思？](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#9LwIPLwIP_86)
    -   [问题10：TCP的“三次握手”和“四次挥手”是什么意思？](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#10TCP_90)
    -   [问题11：有基本的理论基础了，但实际要用STM32+LWIP该怎么操作？](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#11STM32LWIP_117)
    -   -   [11-1硬件部分](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#111_118)
        -   [11-2软件部分](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#112_126)
-   [三、没有结语](https://blog.csdn.net/lrqblack/article/details/123842063?ops_request_misc=elastic_search_misc&request_id=f1370faaa3d41ee13f533beda9e90bbe&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123842063-null-null.142^v102^pc_search_result_base2&utm_term=lwip%E5%88%9D%E5%AD%A6&spm=1018.2226.3001.4187#_135)

## 一、前言

本人作为一个网络方面的新手，由于工作需要用到STM32+LWIP实现以太网通讯，故从零开始学习网络和LWIP知识。开始学习的时候，应该和所有新手一样，是一脸懵逼的，什么是网络？什么是LWIP？硬件怎么办？程序怎么写？我该从哪里下手？阿巴阿巴阿巴学了半天发现问题不减反增。

由于网络这方面的东西实在太多也太深奥，我在学习的过程中也遇到很多问题，所以为了能学而时习之，同时也希望分享我的学习经验，故整理记录一个全程的学习笔记。

本篇博客我录制了视频讲解，欢迎观看和提出建议！[【二】STM32+LWIP 0基础 小白90分钟快速入门 （萌新扫盲）](https://www.bilibili.com/video/BV1kL4y1L7jJ?spm_id_from=333.999.0.0)

## 二、问题汇总

### 问题1：为什么要用以太网通信？

工业以太网是在以太网技术和TCP/IP技术的基础上开发出来的一种工业网络。在工业中，由于以太网的用户培训成本低、产品技术成熟、价格低廉供应稳定、信息集成容易，可以实现将信息网络和控制网络扁平化为一层形成一网打尽的局面等多种优势，以太网的使用越来越广泛。将来工业以太网由可能成为工业控制网络结构的主要形式，形成一网到底的局面。所以STM32在以太网通信方面的使用是十分重要的。

### 问题2：既然要用以太网，那什么是以太网？

**以太网≠互联网。**

以太网(Ethernet) 是指遵守 IEEE 802.3 标准组成的局域网,是互联网技术的一种，由于它是在组网技术中占的比例最高，很多人直接把以太网理解为互联网。

IEEE 还有其它局域网标准，如 IEEE 802.11 是无线局域网，俗称 Wi-Fi。IEEE802.15 是个人域网，即蓝牙技术，其中的 802.15.4 标准则是 ZigBee 技术。

### 问题3：IIC、SPI都有自己的协议，那以太网用什么协议通信呢？

主要用TCP/IP。

### 问题4：TCP/IP又是什么？

TCP/IP (Transmission Control Protocol/Internet Protocol)**是一个庞大的协议族**（TCP和IP只是其中两个协议），它**是众多网络协议的集合**，包括：ARP、IP、ICMP、UDP、TCP、DNS、DHCP、 HTTP 、FTP、MQTT等等。

TCP/IP 协议栈中不同协议所完成的功能是不一样的， 某些协议的实现要依赖于其它协议，依据这种依赖关系，可以将协议栈分层。低层协议为相邻的上层协议提供服务，是上层协议得以实现的基础。

### 问题5：为什么要对网络进行分层？分层有什么好处?

这是为了让不同的操作系统、不同的厂商的终端设备能够通信，大家遵循一样的协议标准。

分层还可以让不同的底层网卡可以独立实现自己的功能，对下(物理层)实现相同的编码方式，对上(网络层）提供标准的接口，至于内部如何实现，别人并不需要知道。

对于IP层也是，内部独立完成自己的功能，对上提供标准的接口，对下调用标准的接口，这样开发起来非常便利。

通俗的来说，就好像做面包，面包店不需要知道加工厂是怎么制造面粉的，只需要知道无论是哪个加工厂利用何种手段加工面粉，最后都会提供标准的面粉就行了。

### 问题6：网络分了几层？这几层该怎么理解？

**网上的网络分层图片各有不同，我把网上所有的图都整理到了一张图上，供大家一眼看到底。若有差错请指正！**

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/39c1970bb1d764ff67645290810f88a6.jpeg#pic_center)

我们嵌入式STM32主要学习TCP/IP五层模型就行了。各位都是实践派。

抛去目的谈原理都是耍流氓！我根据我的理解，通过一个传输过程来通俗解释一下这几层起的作用。

————————————你给朋友发QQ的时候————————————

一天，你打开QQ，看见了朋友在线，于是你传给他一句“来打游戏吗？”并附带了一个表情包。在你按下“发送”键后，计算机怎么把这些信息传输过去呢？

首先，应用层发现你传送的数据有一条文字信息和一条图片格式的信息。于是做好标记，加上应用层首部，送到传输层。

我们用QQ发送信息，对方也一定要用QQ才能收到，那就要用到“端口”，一个电脑上有很多程序，不同程序有不同端口来识别，于是我们做好端口标记，加上TCP首部，和之前的部分形成TCP段，送到了网络层。

我们所处的网络是由无数子网组成的，我们要把消息送到对方电脑，就要用到 IP地址 ，把数据送到相应的子网中找到对应IP地址的电脑（IP地址包括网络号和主机号，网络号相同则判断在同一个子网下，那怎么知道是不是在同一个子网下呢？这时候就用到了子网掩码，用子网掩码和IP地址做与运算，结果相同则判断在同一个子网下），于是乎，做好标记，加上IP首部，与之前的部分形成IP数据包。送到链路层。

连入网络的每一个计算机都会有网卡接口，每一个网卡都会有一个唯一的地址，这个地址就叫做 MAC 地址。计算机之间的数据传送，就是通过 MAC 地址来唯一寻找、传送的。MAC地址 由 48 位二进制数所构成，在网卡生产时就被唯一标识了。

所以当我们需要发送数据时，不仅要找到电脑主机，还要知道MAC地址。我们假设ARP协议已经帮我们要到了目标主机的MAC地址。但是我们发送的信息可能有点大，一次发不出去，这时链路层帮我们把切开成一组一组的，做好标记，一次一次发送出去。于是我们做好记号，加上以太网首部和尾部，形成“以太网帧”，数据一帧一帧送给物理层发送出去。

物理层收到了这些数据，把高电平当作1低电平当作0，把数据发送出去。

————————————你朋友电脑接收你的QQ消息的时候————————————

然后这些0101的电平信号送到了你朋友电脑的物理层。物理层把数据接收给链路层。

链路层一看，呦呵这不是我的MAC地址嘛，原来数据是发给我的，赶紧排排顺序看看数据有没有问题，没问题就拆掉以太网首部，把剩下的交给IP层。

IP层看见数据，拆掉IP首部，交给传输层。

传输层一看，原来是交给QQ端口的数据，于是拆掉TCP首部，交给应用层

应用层一看，一个文字格式的数据，一个图片格式的数据，于是用不同的方法解读渲染这两种数据，于是乎，一条“来打游戏吗？”和一个表情包就出现在你朋友的QQ聊天框里。

这里引用知乎大佬@360lingker的图做个补充，是不是配上图就更好理解了？

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/fd7d5b4e64a6247b0824cca39f3de3fd.png)![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ffcbdc910ff25159d76af08a8652844c.png)

### 问题7：TCP/IP协议这么复杂，STM32就这么点空间，怎么放得下呢？

用LwIP。个不完整的轻量级TCP/IP协议，占用资源极小，常用于嵌入式。

### 问题8：LwIP是什么？和TCP/IP有什么关系？

**LwIP 全名：** Light weight IP，顾名思义是一个轻量化的 TCP/IP 协议， 是瑞典计算机科学院 (SICS)的 Adam Dunkels 开发的一个小型开源的 TCP/IP 协议栈。

**LwIP 的设计目的：** 用少量的资源消耗实现一个较为完整的 TCP/IP 协议栈，

**与TCP/IP的区别：**

① LwIP并没有实现TCP/IP的全部功能

②极大的减少了对RAM 的占用。

③ LwIP既可以移植到操作系统上运行，也可以在无操作系统的情况下独立运行。

④LwIP 并没有采用很明确的分层结构，它假设各层之间的部分数据和结构体和实现原理在其他层是可见的，简单来说就传输层知道 IP 层是如何封装数据、传递数据的， IP 层知道链路层是怎么封装数据的等等，可以实现内存区域共用，极力避免数据拷贝，尽可能减少占用。

**特点：**

①资源开销低。高度可剪裁。一切不需要的功能都可以通过宏编译选项去掉。LwIP 的流畅运行需要 40KB 的代码 ROM 和几十 KB 的RAM

②支持的协议较为完整。

③实现了一些常见的应用程序。DHCP 客户端、 DNS 客户端、 HTTP 服务器等等。

④提供了三种编程接口： RAW/Callback API（适合少任务、交互数据量小、数据处理时间开销小的场合）、 NETCONN API（就是Sequential API（适合多任务、大数据量、大型应用））和 Socket API（是对NETCONN API的简单封装，更简单，但效率低、功能不完整）。

⑤高度可移植。其源代码全部用 C 实现。

⑥开源、免费，用户可以不用承担任何商业风险地使用它。

⑦发展历史长

### 问题9：让你用LwIP协议栈编程，这个LwIP协议栈又是什么意思？

**协议栈是协议的具体的实现形式，我们通俗的来讲就是用代码实现的库函数，从而方便开发人员的调用。**

我个人认为就是到LwIP官网上下载LwIP的最新版，然后把里面的东西放在你自己的工程里，然后用LwIP提供给用户的接口函数在自己的程序里实现TCP通信的功能。（和STM32的库函数一个道理，在MDK中使用库函数也是要去ST官网下载包的）

### 问题10：TCP的“三次握手”和“四次挥手”是什么意思？

TCP是一种面向连接的 传输协议 ，通俗来讲，UDP就像你妈在楼下扯一嗓子让你下楼帮忙，所有人都听得到，而且你也容易听不清；而TCP就像你妈打电话给你，信息传输一对一且稳定。

“三次握手”是确认连接的方式：

通俗来讲就是用三次发送来让双方确认自己与对方的发送与接收是正常的。

第一次 A：我要和B通信

第二次 B：好，我可以和你通信（当A收到这段话时，A已经确认了连接的可靠性，但B还不能确认A能否听到自己的回话）

第三次：A：我能听到你的回话，可以开始了。（这时候双方都确认了连接的可靠性）

如果握手尝试5次都没回应，就断开。

“四次挥手”是确认断开连接的方式：

通俗来讲就是用四次发送来让双方都确认连接的断开。

第一次 A：我要去忙别的了，不和你说了

第二次 B：我这句话还有一半没讲完呢，你听我说完再走

第三次 B：好的我说完了，你去忙吧

第四次 A：好的，我走了（A发完后会停留2毫秒，如果收到B的释放报文，说明B已经收到了释放信号，连接断开）

如果B没收到第四次挥手，就会重复发送第三次的释放信号，A发完后会停留2毫秒，如果还受到了B的第三次挥手信号，说明B没收到第四次挥手，这时候会A会再次重复第四次挥手。

具体的可以看下面两幅图，

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/34eec4bc9b2591f02f2390b367a43482.png#pic_center)![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d727f55ffb150e51955ba968b50b312b.png#pic_center)

SYN：同步序列编号

ACK：确认字符

Seq：序列号

（第三次握手阶段会有数据传输，握手信号是“捎带”的。）

### 问题11：有基本的理论基础了，但实际要用STM32+LWIP该怎么操作？

#### 11-1硬件部分

通过上面的问题，大家都能意识到硬件部分肯定和上三层关系不大，主要关心的就是两层——MAC层和PHY层。

首先看自己手上的STM32芯片，众所周知，STM32中有众多外设，而涉及到网络的一个关键的外设就是“ETH”，比如F103就没有，而F407就有，有没有这个外设决定了你需要选购怎样一个PHY层芯片。

如果你的STM32中集成了ETH外设，那么恭喜你，你的芯片可以通过 DMA 控制器进行介质访问控制(MAC) 实现 MAC 层的任务，通过 ETH 外设可以按照 IEEE 802.3-2002 标准发送和接收 MAC 数据包。简单来说，你的STM32中是有MAC层的，你只需要搭配一个PHY层的芯片就可以了，例如DP83848或者LAN8742（LAN8720）。而你可以通过RMII或者MII来连接STM32MAC层和PHY层芯片。（注意LAN8742系列芯片只支持RMII）

如果很不幸你的芯片没有集成以太网外设，那你可以购买W5500，这个芯片同时集成了MAC层和PHY层的功能，MAC层数据通过SPI传输进STM32芯片。

具体接线、芯片详情，RMII和MII的区别，我之后整理出单独的文章分享。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/159ae185f0075921cd53ea3938a07895.png#pic_center)

#### 11-2软件部分

这方面可以看我的上一篇文章，手把手用 CubeMX 配置STM32+LwIP，十分管用！

[【LWIP】stm32用CubeMX配置LwIP+Ping+TCPclient+TCPserver发送信息到PC（操作部分）](https://blog.csdn.net/lrqblack/article/details/123791613?spm=1001.2014.3001.5502)

这里谈到的硬件和软件都是浅尝辄止的地步。

具体的硬件选型、接线以及软件程序实现的原理我会单独整理一篇博客分享出来。

## 三、没有结语

这篇文章会不定期更新，因为本人作为一个新手，还并不能熟练的使用网络，之后还会遇到大量的问题。我会不断的去解决问题、消化知识、然后用尽可能通俗简练的方式分享出来。

**本文章主要是记录基础性的问题，更加深入的研究和学习成果会整理成单独的文章。**

**希望大佬们能帮忙纠错和提出建议！**

___

本文部分参考、整理、转载于（不限于）如下书籍和文章：

《嵌入式网络那些事》

《野火lwip应用开发实战指南》

简书@该用户快成仙了https://www.jianshu.com/p/e37f1f7bff25

bilibili@蛋黄酱研究院https://www.bilibili.com/video/BV1py4y1V7iA?spm\_id\_from=333.999.0.0

bilibili@敬维https://www.bilibili.com/video/BV14A411E7d5?spm\_id\_from=333.999.0.0

知乎@车小胖

知乎@帅地https://www.zhihu.com/question/49335649

知乎@360linker https://zhuanlan.zhihu.com/p/114863612

CSDN@一门清https://blog.csdn.net/weilexuexi12/article/details/73457626