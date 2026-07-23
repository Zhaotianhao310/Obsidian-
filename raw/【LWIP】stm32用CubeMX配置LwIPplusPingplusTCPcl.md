> 提示：文章写完后，目录可以自动生成，如何生成可参考右边的帮助文档

#### 文章目录

-   [前言](https://blog.csdn.net/lrqblack/article/details/123791613?spm=1001.2014.3001.5502#_7)
-   [零、更新（2022.08.07）](https://blog.csdn.net/lrqblack/article/details/123791613?spm=1001.2014.3001.5502#20220807_13)
-   [一、实验平台](https://blog.csdn.net/lrqblack/article/details/123791613?spm=1001.2014.3001.5502#_22)
-   [二、手把手全程配置步骤](https://blog.csdn.net/lrqblack/article/details/123791613?spm=1001.2014.3001.5502#_25)
-   -   [1.配置电脑环境](https://blog.csdn.net/lrqblack/article/details/123791613?spm=1001.2014.3001.5502#1_26)
    -   [2.配置cubeMX](https://blog.csdn.net/lrqblack/article/details/123791613?spm=1001.2014.3001.5502#2cubeMX_37)
    -   [3.配置MDK（Keil5）](https://blog.csdn.net/lrqblack/article/details/123791613?spm=1001.2014.3001.5502#3MDKKeil5_70)
    -   [4.配置TCPclient通信程序](https://blog.csdn.net/lrqblack/article/details/123791613?spm=1001.2014.3001.5502#4TCPclient_102)
    -   [5.配置TCPserver通信程序](https://blog.csdn.net/lrqblack/article/details/123791613?spm=1001.2014.3001.5502#5TCPserver_238)
-   [三、总结](https://blog.csdn.net/lrqblack/article/details/123791613?spm=1001.2014.3001.5502#_334)

___

## 前言

**这是我写的第一篇博客，欢迎大家给点鼓励和提出建议！**

本人由于理想和爱好，辞去土木工作，于不到一个月前入职某科技公司开始从事嵌入式，专业能力和刚毕业的大学生一样都是很薄弱的。然后被分配到了关于stm32网络方面的工作，经过两个星期的苦学，从一个对cubeMX、网络和LWIP都是零基础的新手学会了LWIP和网络的基础原理，能用CubeMX对LWIP进行相关的配置，还能相互通信。

**这个过程真的花了超多时间踩坑！！** 必须感叹一下网上很多教程对LWIP和网络的新手真的不友好，至少我都没成功过，我相信很多新手也很苦恼这个，于是我希望这个手把手配置教学可以尽我一点绵薄之力帮助到广大新手！（该教程我在其他的板子上也进行过测试，也是没问题的，放心好了!）

**本章是操作部分，让和我一样的新手们先把东西搞出来，毕竟实践是检验真理的唯一标准，只要能搞出效果，之后的研究都好说！理论部分等我整理出来会陆陆续续开始上传，希望和广大爱好者一同进步！**

本篇博客我也录制了操作视频，欢迎观看和提出建议！[【三】STM32+LWIP 0基础 小白90分钟快速入门 （初次实战）](https://www.bilibili.com/video/BV1a94y1Z7Gp?spm_id_from=333.999.0.0)

## 零、更新（2022.08.07）

1、已更新测试F207+DP83848也能使用该方法

2、已更新STM32配置为服务器端的办法

3、**本程序适用于CubeMX6.4！！！**

**本程序适用于CubeMX6.4！！！**

**本程序适用于CubeMX6.4！！！**

**有人反馈6.5配置不一样，确实不一样，请暂时使用6.4配合我的博客！！！**

4、补加 PHY芯片 复位步骤

## 一、实验平台

我用一块野火的F407板子进行实验，以太网卡PHY层芯片是LAN8720。（已测试DP83848芯片也能使用该方法）

## 二、手把手全程配置步骤

### 1.配置电脑环境

先得配置自己的电脑的IP为固定，并且关闭防火墙。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0a47c0a83f6329377212f1858eabb9b9.png#pic_center)![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8ffa1fb332187504fd083501a0b891b1.png#pic_center)

### 2.配置cubeMX

打开CubeMX选择F407芯片。

由于没有使用系统，所以配置为systick就行了（如果有系统，就要使用定时器）

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/10d29f35598ec6e3ded333216bab62f1.png#pic_center)

然后配置ETH，这里选择RMII，因为LAN8720只支持RMII,也只有24个引脚。然后引脚配置按实际的来就行。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/38cdcbd56465ef9b52d04ef8738c4ed6.png#pic_center)

然后是这里的PHY Address要根据芯片的PHYAD引脚的实际连接来配置，这块板子上LAN8720的PHYAD引脚是悬空的，由于这根引脚内部带一个弱下拉，所以我这儿就选择0。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/b6731c5b5e78b8e35825f6b85bd285d6.png#pic_center)

然后这里的高级设置，由于cubeMX已经有了LAN8742和DP83848主流的两种芯片的默认配置，我这芯片是LAN8720，选LAN8742也能用，前半部分是通用的，后面哪些如果用的是不同的芯片，就要看手册配置了。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/780c36f4504f4d3c31f8c12b3031f7ca.png#pic_center)

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/08a7c302c29c9663e4b30c7ab4040161.png#pic_center)

然后是配置LWIP。使能LWIP，为了方便，把DHCP关了，我自己配置IP、掩码和网关（我这里是网线直连，网关无所谓）,这里用的是TCP，就把UDP关掉。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/1de11a924d933a29d9ed584e8b53263a.png#pic_center)

为了方便调试，按实际板子上的接线，打开串口。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/f0cffc18eb0f49dbe301447d9afe9097.png#pic_center)

由于我这块板子上LAN8720的接线是用了25M晶振然后配置LAN8720内部锁相环自己倍频 出50M的时钟，所以我时钟配置直接默认就好了，无所谓。需要注意一点，RMII必须要由50M时钟，如果没用自己的晶振，就要用MCO给它提供50M时钟。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/6b0511f055c1f8c0a1d6ad3fe4a61b52.png#pic_center)

然后在这里选择输出 MDK 代码

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4e95f092f081210545485324dde256c0.png#pic_center)

为了方便添加自己的代码，我们勾上这个，尽可能的分开这些代码。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/58c75318b35dd42567f6fe22a01df13f.png#pic_center)

### 3.配置MDK（ Keil5 ）

打开代码，我们要先配置MDK（Keil5）

这个C库一定要勾选！

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/2d88856a3f53ed31ffcf79f2d8fa3103.png#pic_center)

为了代码编译得快一点，我们把这两个关掉。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/3349992b724344a9910aa9c75c7dcfa1.png)

Reset and run勾上，常规操作了，主要给新手看。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/e1713c31db3792b9a8e883ff4b53b460.png)

主循环中加入这个函数，这样就可以试试能不能ping通了！

编译！ 下载 ！

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d89e77f53e09a6952bb13f9ac8a4a64d.png#pic_center)

在PHY层芯片初始化（GPIO、CLOCK、NVIC和传输模式配置）之前加入一个复位！其实就是连接PHY芯片复位引脚的那个GPIO拉低拉高一下。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/9de74e4d0764a56e49779406e2740f9d.png)

让我们来ping一下。果然ping通了!

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/21d068e8b785b5dae19b9accc8cb46a1.png#pic_center)

为了使用串口调试程序，打印一些信息，我们把串口重定向（这里很奇怪，如果不进行重定向，就算你无意使用串口，也无法用网络发送信息）

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/a2db18cc975a2b6698a91917089458fb.png#pic_center)

在main.h中也添加stdio.h，这是为了能各个程序中使用printf打印信息。再在main函数中添加一个printf，测试一下串口，这样串口也能正常使用了！

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/b78abc6c3b1b34192d4e76fe5e9ee173.png#pic_center)

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4c891e435e410a746aa8a0b16193c5d6.png#pic_center)

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/3df0a4475717dc167621be9dfc73b1c4.png#pic_center)

### 4.配置TCPclient通信程序

接下来就要添加和TCPclinet通信的代码了！

我们新建两个文件，一个是tcpclient.c和tcpclient.h。

tcpclinet.c代码如下

```
#include "lwip/netif.h"
#include "lwip/ip.h"
#include "lwip/tcp.h"
#include "lwip/init.h"
#include "netif/etharp.h"
#include "lwip/udp.h"
#include "lwip/pbuf.h"
#include <stdio.h>	
#include <string.h>
#include "main.h"




static void client_err(void *arg, err_t err)       //出现错误时调用这个函数，打印错误信息，并尝试重新连接
{
  printf("连接错误!!\n");
	printf("尝试重连!!\n");
  
  //连接失败的时候释放TCP控制块的内存
	printf("关闭连接，释放TCP控制块内存\n");
  //tcp_close(client_pcb);
	  
  
  //重新连接
	printf("重新初始化客户端\n");
	TCP_Client_Init();
	
}


static err_t client_send(void *arg, struct tcp_pcb *tpcb)   //发送函数，调用了tcp_write函数
{
  uint8_t send_buf[]= "我是客户端，是你的好哥哥\n";
  
  //发送数据到服务器
  tcp_write(tpcb, send_buf, sizeof(send_buf), 1); 
  
  return ERR_OK;
}

static err_t client_recv(void *arg, struct tcp_pcb *tpcb, struct pbuf *p, err_t err)
{
  if (p != NULL) 
  {        
    /* 接收数据*/
    tcp_recved(tpcb, p->tot_len);
      
    /* 返回接收到的数据*/  
    tcp_write(tpcb, p->payload, p->tot_len, 1);
      
    memset(p->payload, 0 , p->tot_len);
    pbuf_free(p);
  } 
  else if (err == ERR_OK) 
  {
    //服务器断开连接
    printf("服务器断开连接!\n");
    tcp_close(tpcb);
    
    //重新连接
    TCP_Client_Init();
  }
  return ERR_OK;
}

static err_t client_connected(void *arg, struct tcp_pcb *pcb, err_t err)
{
  printf("connected ok!\n");
  
  //注册一个周期性回调函数
  tcp_poll(pcb,client_send,2);
  
  //注册一个接收函数
  tcp_recv(pcb,client_recv);
  
  return ERR_OK;
}


void TCP_Client_Init(void)
{        
	struct tcp_pcb *client_pcb = NULL;   //这一句一定要放在里面，否则会没用
  ip4_addr_t server_ip;     //因为客户端要主动去连接服务器，所以要知道服务器的IP地址
  /* 创建一个TCP控制块  */
  client_pcb = tcp_new();	  

  IP4_ADDR(&server_ip, DEST_IP_ADDR0,DEST_IP_ADDR1,DEST_IP_ADDR2,DEST_IP_ADDR3);//合并IP地址

  printf("客户端开始连接!\n");
  
  //开始连接
  tcp_connect(client_pcb, &server_ip, TCP_CLIENT_PORT, client_connected);
	ip_set_option(client_pcb, SOF_KEEPALIVE);	
	
	printf("已经调用了tcp_connect函数\n");
  
  //注册异常处理
  tcp_err(client_pcb, client_err);
	printf("已经注册异常处理函数\n");	
}
```

然后把tcpclient.c加入工程中。

然后是tcpclient.h的代码

```
#ifndef _TCPCLIENT_H_
#define _TCPCLIENT_H_

#define TCP_CLIENT_PORT 5001

void TCP_Client_Init(void);

#endif
```

为了在主函数中调用TCP\_Client\_Init();（初始化TCP客户端程序，加入这个就能通信了，不用加入其他代码到主循环里） ，所以在main.h中加入#include “tcpclient.h”

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/a3254790bc3624563f74656a5cecb7a1.png#pic_center)

再把这个加入到主循环前，这样客户端发送信息的功能就配置好了

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/adfa68ff5ff8a143105125ce58f8893a.png#pic_center)

由于我们配置的cubeMX只配置了板子的IP，但板子作为客户端，是要去主动连接PC的，所以我们要把目标地址也放入程序，我们姑且把程序放在main.h中吧，如果你要方便管理，可以把这些东西统一放在其他地方。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/6fe703e65eb8ae92b02e0a2503330275.png#pic_center)

编译！下载！然后打开串口和网络助手，端口号输入5001。

我们可以看到串口打印出信息，网络助手也不断收到消息。成功了！

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/3ad243b7b9568aa33e267b810e8bb077.png#pic_center)

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/f0266fe0d7fec56ef8c6a49a96e30911.png#pic_center)

### 5.配置TCPserver通信程序

接着步骤3，我们创建两个文件。一个是tcpserver.c，另一个是tcpserver.h。

tcpserver.c代码如下

```
#include "tcpserver.h"
#include "lwip/netif.h"
#include "lwip/ip.h"
#include "lwip/tcp.h"
#include "lwip/init.h"
#include "netif/etharp.h"
#include "lwip/udp.h"
#include "lwip/pbuf.h"
#include <stdio.h>	
#include <string.h>


static err_t tcpecho_recv(void *arg, struct tcp_pcb *tpcb, struct pbuf *p, err_t err)
{                                   //对应接收数据连接的控制块   接收到的数据   
  if (p != NULL) 
  {        
	//int a = 666;
	/* 更新窗口*/
	tcp_recved(tpcb, p->tot_len);     //读取数据的控制块   得到所有数据的长度   
			
    /* 返回接收到的数据*/  
  //tcp_write(tpcb, p->payload, p->tot_len, 1);

	uint8_t send_buf1[]= "我收到了你的信息！是";
	uint8_t send_buf2[]= "吗？\n";	
	tcp_write(tpcb, send_buf1, sizeof(send_buf1), 1);
	tcp_write(tpcb, p->payload, p->tot_len, 1);	
	tcp_write(tpcb, send_buf2, sizeof(send_buf2), 1);	
    
  memset(p->payload, 0 , p->tot_len);
  pbuf_free(p);
    
  } 
  else if (err == ERR_OK)    //检测到对方主动关闭连接时，也会调用recv函数，此时p为空
  {
    return tcp_close(tpcb);
  }
  return ERR_OK;
}

static err_t tcpecho_accept(void *arg, struct tcp_pcb *newpcb, err_t err) //由于这个函数是*tcp_accept_fn类型的
																																					//形参的数量和类型必须一致
{     

  tcp_recv(newpcb, tcpecho_recv);    //当收到数据时，回调用户自己写的tcpecho_recv
  return ERR_OK;
}

void TCP_Echo_Init(void)
{
  struct tcp_pcb *server_pcb = NULL;	            		
  
  /* 创建一个TCP控制块  */
  server_pcb = tcp_new();	
	printf("创建了一个控制块\n");
  
  /* 绑定TCP控制块 */
  tcp_bind(server_pcb, IP_ADDR_ANY, TCP_ECHO_PORT);       
	printf("已经绑定一个控制块\n");

  /* 进入监听状态 */
  server_pcb = tcp_listen(server_pcb);
	printf("进入监听状态\n");	

  /* 处理连接 注册函数，侦听到连接时被注册的函数被回调 */	
  tcp_accept(server_pcb, tcpecho_accept);  //侦听到连接后，回调用户编写的tcpecho_accept 
																	  //这个函数是*tcp_accept_fn类型的
}
```

然后把文件添加入工程

tcpserverc.h代码如下

```
#ifndef _TCPECHO_H_
#define _TCPECHO_H_

#define TCP_ECHO_PORT 5001

void TCP_Echo_Init(void);

#endif
```

接下来只要用类似步骤4一样添加头文件，再在主循环上方加一条 TCP\_Echo\_Init();

编译下载后，电脑上的网络助手配置为客户端，填入板子IP和端口号，连接成功，发送信息后会发现板子会返回相同的数据给网络助手。

当我们把客户端代码和服务器端代码都配置好时，STM32可以同时接收数据，也可以发送数据。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/22e90835eef506f77d83bdb5cf97959c.png#pic_center)

___

## 三、总结

学习LWIP这个过程实在是艰难，希望我的努力能帮到各位！第一篇博客多有不熟练，希望各位多留言互动。

本人新手，在座各位的水平一定比我高，希望能给我指出错误，提出建议，谢谢！

如果不能配置，或者丢图丢步骤，可以评论留言。我尽力而为。