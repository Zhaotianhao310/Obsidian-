# LwIP RAW API TCP 速查

## 适用范围

本页只整理 raw 教程实际使用到的 LwIP RAW API TCP 调用。函数原型以工程采用的 LwIP 版本头文件为最终依据；raw 教程本身未给出完整原型，因此下列签名采用常见 LwIP 形式，并保留版本差异提示。

## 基础头文件

~~~c
#include "lwip/tcp.h"
#include "lwip/ip.h"
#include "lwip/pbuf.h"
#include "lwip/err.h"
~~~

- lwip/tcp.h：TCP 控制块、连接、监听、收发和回调注册。
- lwip/ip.h：IP 地址类型和 IP4_ADDR 宏。
- lwip/pbuf.h：接收缓冲区 pbuf 与释放函数。
- lwip/err.h：ERR_OK、ERR_MEM 等错误码。
- raw 教程还包含 lwip/netif.h、lwip/init.h 和 netif/etharp.h；它们属于工程初始化/网卡适配依赖，不是本页 TCP 调用的核心入口。

## 错误码约定

- ERR_OK：操作成功。
- ERR_MEM：内存不足。
- ERR_VAL：参数无效。
- ERR_USE：端口或资源已被占用。
- ERR_ABRT：连接被本地终止。
- ERR_RST：连接被对端复位。
- ERR_CLSD：连接已关闭。
- 具体函数可返回的状态码取决于 LwIP 版本和调用上下文，应以 lwip/err.h 及对应函数声明为准。

## 地址与控制块

### IP4_ADDR

**作用**：把四个八位 IPv4 数值写入 ip4_addr_t。

**签名**：IP4_ADDR(ip4_addr_t *ipaddr, u8_t a, u8_t b, u8_t c, u8_t d)

**参数**：ipaddr 为输出地址；a、b、c、d 为 IPv4 四段数值。

**返回值**：宏，无函数返回值；直接修改 ipaddr。

~~~c
ip4_addr_t server_ip;
IP4_ADDR(&server_ip, 192, 168, 1, 10);
tcp_connect(pcb, &server_ip, 5001, on_connected);
~~~

### tcp_new

**作用**：创建一个 TCP 控制块 tcp_pcb，作为后续客户端或服务端操作的句柄。

**签名**：struct tcp_pcb *tcp_new(void)

**参数**：无。

**返回值**：成功返回控制块指针；资源不足时返回 NULL。

~~~c
struct tcp_pcb *pcb;
pcb = tcp_new();
if (pcb == NULL) { return ERR_MEM; }
~~~

## TCP 客户端

### tcp_connect

**作用**：使用已有控制块主动连接目标 IP 和端口。

**签名**：err_t tcp_connect(struct tcp_pcb *pcb, const ip_addr_t *ipaddr, u16_t port, tcp_connected_fn connected)

**参数**：pcb 为客户端控制块；ipaddr 为目标 IP；port 为目标端口；connected 为连接结果回调。

**返回值**：常见为 ERR_OK；失败时可能返回 ERR_MEM、ERR_VAL 或其他 err_t 状态码。

~~~c
IP4_ADDR(&server_ip, 192, 168, 1, 20);
err_t e = tcp_connect(pcb, &server_ip, 5001, on_connected);
if (e != ERR_OK) { tcp_close(pcb); }
~~~

### tcp_err

**作用**：注册异步错误回调；发生连接异常时由协议栈通知应用。

**签名**：void tcp_err(struct tcp_pcb *pcb, tcp_err_fn err)

**参数**：pcb 为目标控制块；err 为错误回调，典型签名为 void (*)(void *arg, err_t err)。

**返回值**：无返回值；错误通过回调参数传递。

~~~c
static void on_error(void *arg, err_t e) { reconnect(); }
tcp_err(pcb, on_error);
return ERR_OK;
~~~

### tcp_poll

**作用**：注册周期性回调，用于重试、定时发送或连接保活逻辑。

**签名**：err_t tcp_poll(struct tcp_pcb *pcb, tcp_poll_fn poll, u8_t interval)

**参数**：pcb 为控制块；poll 为周期回调；interval 为以 TCP coarse timer 为单位的间隔，具体周期受 LwIP 版本配置影响。

**返回值**：常见为 ERR_OK；注册失败时返回其他 err_t 状态码。

~~~c
static err_t on_poll(void *arg, struct tcp_pcb *pcb) { send_data(pcb); return ERR_OK; }
tcp_poll(pcb, on_poll, 2);
return ERR_OK;
~~~

### tcp_recv

**作用**：注册接收回调，收到 TCP 数据或对端关闭时触发。

**签名**：void tcp_recv(struct tcp_pcb *pcb, tcp_recv_fn recv)

**参数**：pcb 为连接控制块；recv 的典型签名为 err_t (*)(void *, struct tcp_pcb *, struct pbuf *, err_t)。

**返回值**：无返回值；接收结果通过回调参数处理。

~~~c
static err_t on_recv(void *arg, struct tcp_pcb *pcb, struct pbuf *p, err_t e);
tcp_recv(pcb, on_recv);
return ERR_OK;
~~~

### tcp_write

**作用**：把应用数据加入 TCP 发送队列。

**签名**：err_t tcp_write(struct tcp_pcb *pcb, const void *dataptr, u16_t len, u8_t apiflags)

**参数**：pcb 为连接控制块；dataptr 为数据地址；len 为数据长度；apiflags 常用 TCP_WRITE_FLAG_COPY，表示由协议栈复制数据。

**返回值**：常见为 ERR_OK；发送队列空间不足时可能为 ERR_MEM，连接无效时可能为 ERR_CONN 或其他错误码。

~~~c
const char msg[] = "hello\n";
err_t e = tcp_write(pcb, msg, sizeof(msg) - 1, TCP_WRITE_FLAG_COPY);
if (e != ERR_OK) { handle_error(e); }
~~~

### tcp_recved

**作用**：通知协议栈应用已经处理了收到的字节数，使 TCP 接收窗口能够前移。

**签名**：void tcp_recved(struct tcp_pcb *pcb, u16_t len)

**参数**：pcb 为连接控制块；len 为本次实际处理的字节数，通常使用 p->tot_len。

**返回值**：无返回值。

~~~c
u16_t n = p->tot_len;
tcp_recved(pcb, n);
process_payload(p);
~~~

### pbuf_free

**作用**：释放 pbuf 链，归还接收缓冲区资源。

**签名**：u8_t pbuf_free(struct pbuf *p)

**参数**：p 为需要释放的 pbuf 链；允许传入 NULL 的行为应以具体版本实现为准，应用通常先判断非 NULL。

**返回值**：返回释放的 pbuf 数量；具体类型和语义以 lwip/pbuf.h 为准。

~~~c
if (p != NULL) {
    process_payload(p);
    pbuf_free(p);
}
~~~

### tcp_close

**作用**：关闭 TCP 控制块对应的连接，并在协议栈允许时释放资源。

**签名**：err_t tcp_close(struct tcp_pcb *pcb)

**参数**：pcb 为待关闭的控制块。

**返回值**：成功通常为 ERR_OK；若当前状态无法立即关闭，需按返回值决定重试或调用 tcp_abort 等版本相关处理。

~~~c
err_t e = tcp_close(pcb);
if (e != ERR_OK) { retry_close(pcb); }
return e;
~~~

## TCP 服务端

### tcp_bind

**作用**：把 TCP 控制块绑定到本地 IP 和端口。

**签名**：err_t tcp_bind(struct tcp_pcb *pcb, const ip_addr_t *ipaddr, u16_t port)

**参数**：pcb 为服务端控制块；ipaddr 可使用 IP_ADDR_ANY 监听本机地址；port 为监听端口。

**返回值**：常见为 ERR_OK；端口占用时可能为 ERR_USE，参数或资源异常时可能为 ERR_VAL 或 ERR_MEM。

~~~c
struct tcp_pcb *pcb = tcp_new();
err_t e = tcp_bind(pcb, IP_ADDR_ANY, 5001);
if (e != ERR_OK) { tcp_close(pcb); }
~~~

### tcp_listen

**作用**：把已绑定的 TCP 控制块转换为监听控制块。

**签名**：struct tcp_pcb *tcp_listen(struct tcp_pcb *pcb)

**参数**：pcb 为已绑定的控制块。

**返回值**：成功返回监听控制块；内存不足等异常时可能返回 NULL。部分 LwIP 版本内部使用带 backlog 的变体，需以头文件为准。

~~~c
struct tcp_pcb *listen_pcb;
listen_pcb = tcp_listen(pcb);
if (listen_pcb == NULL) { return ERR_MEM; }
~~~

### tcp_accept

**作用**：为监听控制块注册新连接回调。

**签名**：void tcp_accept(struct tcp_pcb *pcb, tcp_accept_fn accept)

**参数**：pcb 为监听控制块；accept 的典型签名为 err_t (*)(void *, struct tcp_pcb *, err_t)。

**返回值**：无返回值；新连接到达时调用 accept。

~~~c
static err_t on_accept(void *arg, struct tcp_pcb *newpcb, err_t e);
tcp_accept(listen_pcb, on_accept);
return ERR_OK;
~~~

## 最小流程 Demo

### 客户端初始化

~~~c
struct tcp_pcb *pcb = tcp_new();
IP4_ADDR(&server_ip, 192, 168, 1, 20);
tcp_err(pcb, on_error);
tcp_connect(pcb, &server_ip, 5001, on_connected);
~~~

### 服务端初始化

~~~c
struct tcp_pcb *pcb = tcp_new();
tcp_bind(pcb, IP_ADDR_ANY, 5001);
pcb = tcp_listen(pcb);
tcp_accept(pcb, on_accept);
~~~

## 避坑红线

- pbuf 使用完成后必须释放；异常路径也不能遗漏 pbuf_free。
- 调用 tcp_recved 的长度应与应用实际处理的字节数一致，原素材示例使用 p->tot_len。
- RAW API 是回调式接口，不应把阻塞式 Socket/NETCONN 的调用习惯直接套入。
- 回调函数的参数数量和类型必须与 LwIP 定义的函数指针一致。
- tcp_write 使用的数据缓冲区生命周期必须满足 apiflags 要求；不确定时使用 TCP_WRITE_FLAG_COPY，并检查返回值。
- 客户端需要配置目标 IP；服务端需要保证监听端口与 PC 网络助手一致。原素材示例端口为 5001。
- 具体错误码、宏和函数原型以工程实际 LwIP 版本头文件为准。

## 相关页面

- [[LwIP]]
- [[TCP IP 协议栈]]
- [[TCP 连接建立与释放]]
- [[STM32 CubeMX 配置 LwIP]]
- [[LwIP-知识地图]]

## 来源

- raw 文件：【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md
