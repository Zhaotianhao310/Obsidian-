# LwIP

## 概念一句话精炼

LwIP 是面向资源受限设备的轻量级 TCP/IP 协议栈，负责把应用数据连接到 [[STM32 ETH 外设]]和网络协议流程。

## 核心原理与图解

```mermaid
flowchart LR
    A[应用任务] --> B[RAW API/Netconn/Socket]
    B --> C[TCP/UDP]
    C --> D[IP/ARP/ICMP]
    D --> E[netif]
    E --> F[ETH 驱动与 PHY]
```

> 图 1：LwIP 通过 netif 与底层网卡驱动衔接，并向上提供 TCP/IP 协议服务。


> 图 1：LwIP 通过 netif 与底层网卡驱动衔接，并向上提供 TCP/IP 协议服务。
LwIP 可按工程需要裁剪功能，常见配置包括 DHCP、DNS、HTTP、TCP 和 UDP。是否启用某项能力取决于 `lwipopts.h`、操作系统模式和网卡适配层，不能仅凭协议栈名称判断。

## 关键实现与数据结构

```c
struct netif netif;
ip4_addr_t ip, mask, gw;
netif_add(&netif, &ip, &mask, &gw, NULL, ethernetif_init, tcpip_input);
netif_set_default(&netif);
netif_set_up(&netif);
```

无操作系统和带 RTOS 的 LwIP 在输入处理、定时器和 API 线程约束上不同；RAW API 通过回调工作，不应直接套用阻塞式 Socket 的编程习惯。

## 横向对比与关联

- **RAW API**：低开销、回调式，适合资源受限场景。
- **Netconn API**：以消息和线程安全封装为主，使用成本高于 RAW API。
- **Socket API**：更接近传统网络程序，但通常需要更完整的系统支持。
- [[TCP IP 协议栈]]解释协议分层；[[LwIP RAW API TCP 速查]]解释具体调用。

## 来源

- raw 文件：`【LWIP】初学STM32plusLWIPplus网络遇到的基础问题记录-stm3.md`
- raw 文件：`【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md`
