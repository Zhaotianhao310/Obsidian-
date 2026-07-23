# LwIP RAW API TCP 速查

## 定义

LwIP RAW API 是基于回调函数的轻量级接口，适合少任务、交互数据量较小、处理逻辑较短的嵌入式 TCP 通信场景。

## 头文件

- lwip/tcp.h：TCP 控制块、连接、监听、回调、读写等 API。
- lwip/ip.h：IP 地址类型与宏，如 ip4_addr_t、IP4_ADDR。
- lwip/pbuf.h：接收缓冲区 pbuf 及释放函数。
- netif/etharp.h、lwip/netif.h：以太网接口相关能力。

## TCP 客户端流程

- tcp_new()：创建 TCP 控制块 tcp_pcb。
- IP4_ADDR()：组装目标服务器 IP 地址。
- tcp_connect()：主动连接服务器，并注册连接成功回调。
- ip_set_option(..., SOF_KEEPALIVE)：设置 keepalive 选项。
- tcp_err()：注册异常处理回调。
- tcp_poll()：在连接成功后注册周期性回调，可用于定时发送。
- tcp_recv()：注册接收回调。
- tcp_write()：发送数据。
- tcp_recved()：通知协议栈已处理接收数据，更新窗口。
- pbuf_free()：释放收到的 pbuf。
- tcp_close()：关闭连接。

## TCP 服务端流程

- tcp_new()：创建服务端 TCP 控制块。
- tcp_bind(..., IP_ADDR_ANY, port)：绑定本地端口。
- tcp_listen()：进入监听状态。
- tcp_accept()：注册连接接收回调。
- tcp_recv()：在 accept 回调中为新连接注册接收回调。
- tcp_write()：向客户端返回数据。
- tcp_recved()：确认接收窗口。
- pbuf_free()：释放接收缓冲。
- tcp_close()：对端关闭或服务端主动关闭时释放连接。

## 常量与入口

- 原素材客户端端口宏：TCP_CLIENT_PORT 5001。
- 原素材服务端端口宏：TCP_ECHO_PORT 5001。
- 客户端初始化入口：TCP_Client_Init()。
- 服务端初始化入口：TCP_Echo_Init()。

## 注意事项

- RAW API 回调函数签名必须与 LwIP 要求的函数指针类型匹配。
- pbuf 使用后应释放，避免内存泄漏。
- 收到 p == NULL 且错误码为 ERR_OK 时，通常表示对端主动关闭连接。
- 客户端需要板子自身 IP，也需要目标服务器 IP；目标 IP 可集中放在配置头文件中管理。
- 端口号需要与 PC 网络助手或对端应用保持一致。

## 相关链接

- [[LwIP]]
- [[TCP IP 协议栈]]
- [[TCP 连接建立与释放]]
- [[STM32 CubeMX 配置 LwIP]]
- [[LwIP-知识地图]]

## 来源

- raw/【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md
