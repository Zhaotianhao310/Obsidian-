# LwIP RAW API TCP 速查

## 适用范围

本页整理 raw 教程中使用或涉及的 LwIP RAW API TCP 调用。函数原型和错误码可能随 LwIP 版本变化，最终以工程头文件为准。

## 基础头文件

```c
#include "lwip/tcp.h"
#include "lwip/ip.h"
#include "lwip/pbuf.h"
#include "lwip/err.h"
```

## 核心 API

### `tcp_new`

- **作用**：分配新的 TCP 控制块。
- **签名**：`struct tcp_pcb *tcp_new(void)`
- **参数**：无。
- **返回值**：成功返回 PCB；内存不足返回 `NULL`。

```c
struct tcp_pcb *pcb = tcp_new();
if (pcb == NULL) return ERR_MEM;
tcp_err(pcb, on_error);
```

### `tcp_connect`

- **作用**：发起主动 TCP 连接。
- **签名**：`err_t tcp_connect(struct tcp_pcb *pcb, const ip_addr_t *ipaddr, u16_t port, tcp_connected_fn connected)`
- **参数**：`pcb` 为控制块；`ipaddr` 为服务端地址；`port` 为端口；`connected` 为连接结果回调。
- **返回值**：成功通常为 `ERR_OK`；参数、资源或状态错误返回其他 `err_t`。

```c
err_t err = tcp_connect(pcb, &server_ip, 5001, on_connected);
if (err != ERR_OK) {
    tcp_close(pcb);
    return err;
}
```

### `tcp_recv`

- **作用**：注册接收回调。
- **签名**：`void tcp_recv(struct tcp_pcb *pcb, tcp_recv_fn recv)`
- **参数**：`pcb` 为连接控制块；`recv` 的典型签名为 `err_t (*)(void *, struct tcp_pcb *, struct pbuf *, err_t)`。
- **返回值**：无返回值；接收结果由回调返回。

```c
static err_t on_recv(void *arg, struct tcp_pcb *pcb,
                     struct pbuf *p, err_t err);
tcp_recv(pcb, on_recv);
```

### `tcp_write`

- **作用**：把应用数据加入 TCP 发送队列。
- **签名**：`err_t tcp_write(struct tcp_pcb *pcb, const void *dataptr, u16_t len, u8_t apiflags)`
- **参数**：`pcb` 为连接控制块；`dataptr` 为数据；`len` 为长度；`apiflags` 常用 `TCP_WRITE_FLAG_COPY`。
- **返回值**：成功为 `ERR_OK`；发送缓存不足常见为 `ERR_MEM`。

```c
const char msg[] = "hello\n";
err_t err = tcp_write(pcb, msg, sizeof(msg) - 1, TCP_WRITE_FLAG_COPY);
if (err != ERR_OK) return err;
tcp_output(pcb);
```

### `tcp_output`

- **作用**：尝试立即输出已排队的 TCP 数据。
- **签名**：`err_t tcp_output(struct tcp_pcb *pcb)`
- **参数**：`pcb` 为连接控制块。
- **返回值**：成功为 `ERR_OK`，失败返回对应 `err_t`。

```c
err_t err = tcp_output(pcb);
if (err != ERR_OK) {
    /* 根据版本和连接状态决定重试或关闭 */
    handle_tcp_error(err);
}
```

### `tcp_recved`

- **作用**：通知协议栈应用已经处理了接收数据，使接收窗口前移。
- **签名**：`void tcp_recved(struct tcp_pcb *pcb, u16_t len)`
- **参数**：`pcb` 为连接控制块；`len` 为实际处理的字节数。
- **返回值**：无返回值。

```c
u16_t handled = p->tot_len;
process_payload(p);
tcp_recved(pcb, handled);
```

### `pbuf_free`

- **作用**：释放 pbuf 链，归还接收缓冲区。
- **签名**：`u8_t pbuf_free(struct pbuf *p)`
- **参数**：`p` 为需要释放的 pbuf 链。
- **返回值**：返回释放的 pbuf 数量，具体语义以版本头文件为准。

```c
if (p != NULL) {
    process_payload(p);
    pbuf_free(p);
}
```

### `tcp_close`

- **作用**：请求正常关闭 TCP 控制块。
- **签名**：`err_t tcp_close(struct tcp_pcb *pcb)`
- **参数**：`pcb` 为待关闭控制块。
- **返回值**：成功通常为 `ERR_OK`；资源不足或状态不允许时返回错误码。

```c
err_t err = tcp_close(pcb);
if (err != ERR_OK) {
    /* 不要无条件丢弃 PCB，按状态决定重试或 abort */
    retry_or_abort(pcb, err);
}
```

### `tcp_bind`

- **作用**：把 TCP 控制块绑定到本地 IP 和端口。
- **签名**：`err_t tcp_bind(struct tcp_pcb *pcb, const ip_addr_t *ipaddr, u16_t port)`
- **参数**：`pcb` 为服务端控制块；`ipaddr` 可用 `IP_ADDR_ANY`；`port` 为监听端口。
- **返回值**：成功为 `ERR_OK`；端口占用常见为 `ERR_USE`。

```c
struct tcp_pcb *pcb = tcp_new();
err_t err = tcp_bind(pcb, IP_ADDR_ANY, 5001);
if (err != ERR_OK) tcp_close(pcb);
```

### `tcp_listen`

- **作用**：把已绑定的控制块转换为监听控制块。
- **签名**：`struct tcp_pcb *tcp_listen(struct tcp_pcb *pcb)`
- **参数**：`pcb` 为已经绑定端口的控制块。
- **返回值**：成功返回监听 PCB；失败可能返回 `NULL`。

```c
struct tcp_pcb *listen_pcb = tcp_listen(pcb);
if (listen_pcb == NULL) {
    return ERR_MEM;
}
```

### `tcp_accept`

- **作用**：注册服务端新连接回调。
- **签名**：`void tcp_accept(struct tcp_pcb *pcb, tcp_accept_fn accept)`
- **参数**：`pcb` 为监听 PCB；`accept` 为连接建立回调。
- **返回值**：无返回值。

```c
static err_t on_accept(void *arg, struct tcp_pcb *newpcb, err_t err);
tcp_accept(listen_pcb, on_accept);
```

## 最小流程 Demo

### 客户端

```c
struct tcp_pcb *pcb = tcp_new();
IP4_ADDR(&server_ip, 192, 168, 1, 20);
tcp_recv(pcb, on_recv);
tcp_connect(pcb, &server_ip, 5001, on_connected);
```

### 服务端

```c
struct tcp_pcb *pcb = tcp_new();
tcp_bind(pcb, IP_ADDR_ANY, 5001);
pcb = tcp_listen(pcb);
tcp_accept(pcb, on_accept);
```

## 避坑红线

- `pbuf` 使用完成后必须释放；异常路径也不能遗漏 `pbuf_free`。
- `tcp_recved` 的长度应与实际处理字节数一致。
- `tcp_write` 必须检查返回值；不确定数据缓冲区生命周期时优先使用 `TCP_WRITE_FLAG_COPY`。
- RAW API 是回调式接口，不能把阻塞 Socket API 的调用习惯直接套用。
- 回调函数签名必须匹配目标 LwIP 版本中的函数指针定义。
- `tcp_close` 失败时不能简单丢弃 PCB；是否重试或 abort 需要结合状态和版本处理。
- API、宏和错误码以工程采用的 `lwip/tcp.h`、`lwip/err.h` 为准。

## 相关页面

- [[LwIP]]
- [[TCP IP 协议栈]]
- [[TCP 连接建立与释放]]
- [[STM32 CubeMX 配置 LwIP]]
- [[LwIP-知识地图]]

## 来源

- raw 文件：`【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md`