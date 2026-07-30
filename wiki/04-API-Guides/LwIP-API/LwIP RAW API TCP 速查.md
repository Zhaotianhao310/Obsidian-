# LwIP RAW API TCP 速查

## 适用范围

本页只整理 LwIP TCP RAW API。`on_connected`、`on_recv`、`on_accept`、`on_error` 等是调用方提供的回调占位符，不是 LwIP API；`send_all`、`process_payload`、`handle_tcp_error` 等业务函数也不属于官方接口。RAW API 通常运行在 LwIP 栈所属的 TCP/IP 上下文中，不应照搬阻塞 Socket API 的调用方式。

## 基础头文件

~~~c
#include "lwip/tcp.h"
#include "lwip/pbuf.h"
#include "lwip/err.h"
#include "lwip/ip_addr.h"
~~~

## 核心 API

### `tcp_new`

- **作用**：分配并返回一个新的 TCP 协议控制块。
- **函数原型/签名**：`struct tcp_pcb *tcp_new(void);`
- **参数含义详解**：无参数；返回的 PCB 后续可用于连接、绑定或配置回调。
- **返回值状态码**：成功返回 PCB 指针；内存不足时返回 `NULL`。
- **使用 Demo**：

~~~c
struct tcp_pcb *pcb = tcp_new();
if (pcb == NULL)
    return ERR_MEM;
tcp_arg(pcb, user_arg);
~~~

### `tcp_arg`

- **作用**: 为 PCB 绑定一个由调用方管理的用户参数指针, 后续回调会收到该指针.
- **函数原型/签名**: `void tcp_arg(struct tcp_pcb *pcb, void *arg);`
- **参数含义详解**: `pcb` 是目标连接控制块, `arg` 是调用方上下文指针, 生命周期必须覆盖所有可能的回调调用.
- **返回值状态码**: 无返回值.
- **使用 Demo**:

~~~c
struct session *session = get_session();
tcp_arg(pcb, session);
tcp_recv(pcb, on_recv);
~~~

`get_session`, `session` 和 `on_recv` 是调用方对象/回调, 不是 LwIP API.
### `tcp_connect`

- **作用**：为客户端 PCB 发起到远端 IP/端口的 TCP 连接。
- **函数原型/签名**：`err_t tcp_connect(struct tcp_pcb *pcb, const ip_addr_t *ipaddr, u16_t port, tcp_connected_fn connected);`
- **参数含义详解**：`pcb` 为客户端 PCB；`ipaddr` 和 `port` 指定服务端；`connected` 是连接结果回调。
- **返回值状态码**：成功通常为 `ERR_OK`；参数、状态或资源不足时返回对应 `err_t`。
- **使用 Demo**：

~~~c
ip_addr_t server_ip;
IP4_ADDR(&server_ip, 192, 168, 1, 20);
err_t err = tcp_connect(pcb, &server_ip, 5001, on_connected);
if (err != ERR_OK)
    return err;
~~~

`on_connected` 是调用方回调占位符，不是 LwIP API。

### `tcp_recv`

- **作用**：为 PCB 注册接收回调；收到 pbuf 或连接关闭时由协议栈调用。
- **函数原型/签名**：`void tcp_recv(struct tcp_pcb *pcb, tcp_recv_fn recv);`
- **参数含义详解**：`pcb` 是连接控制块；`recv` 的典型签名为 `err_t (*)(void *, struct tcp_pcb *, struct pbuf *, err_t)`。
- **返回值状态码**：无返回值；处理结果由接收回调返回 `err_t`。
- **使用 Demo**：

~~~c
static err_t on_recv(void *arg, struct tcp_pcb *pcb,
                     struct pbuf *p, err_t err);
tcp_recv(pcb, on_recv);
tcp_arg(pcb, user_arg);
~~~

### `tcp_write`

- **作用**：把应用数据加入 PCB 的 TCP 发送队列。
- **函数原型/签名**：`err_t tcp_write(struct tcp_pcb *pcb, const void *dataptr, u16_t len, u8_t apiflags);`
- **参数含义详解**：`pcb` 为连接 PCB；`dataptr` 指向待发送数据；`len` 为字节数；`apiflags` 常用 `TCP_WRITE_FLAG_COPY`。
- **返回值状态码**：成功返回 `ERR_OK`；发送缓存不足常见为 `ERR_MEM`；其他连接错误返回对应 `err_t`。
- **使用 Demo**：

~~~c
const char msg[] = "hello\n";
err_t err = tcp_write(pcb, msg, sizeof(msg) - 1,
                      TCP_WRITE_FLAG_COPY);
if (err != ERR_OK)
    return err;
~~~

### `tcp_output`

- **作用**：请求立即把已经排队的 TCP 数据发送到网络。
- **函数原型/签名**：`err_t tcp_output(struct tcp_pcb *pcb);`
- **参数含义详解**：`pcb` 为已经建立或处于可发送状态的连接控制块；通常在 `tcp_write()` 后调用。
- **返回值状态码**：成功返回 `ERR_OK`；失败返回对应 `err_t`。
- **使用 Demo**：

~~~c
err_t err = tcp_output(pcb);
if (err != ERR_OK) {
    tcp_abort(pcb);
    return err;
}
~~~

### `tcp_recved`

- **作用**：通知 TCP 栈应用已经处理了指定长度的接收数据，使接收窗口前移。
- **函数原型/签名**：`void tcp_recved(struct tcp_pcb *pcb, u16_t len);`
- **参数含义详解**：`pcb` 为接收连接；`len` 必须是应用实际处理并释放的字节数。
- **返回值状态码**：无返回值。
- **使用 Demo**：

~~~c
u16_t handled = p->tot_len;
consume_payload(p);       /* 调用方处理函数，占位符 */
tcp_recved(pcb, handled);
pbuf_free(p);
~~~

`consume_payload` 是调用方函数，不是 LwIP API。

### `pbuf_free`

- **作用**：释放 pbuf 或 pbuf 链，归还其引用计数。
- **函数原型/签名**：`u8_t pbuf_free(struct pbuf *p);`
- **参数含义详解**：`p` 可以是单个 pbuf 或链表头；释放前应完成数据处理和必要的 `tcp_recved()`。
- **返回值状态码**：返回实际释放的 pbuf 数量；具体引用计数语义以工程 LwIP 版本为准。
- **使用 Demo**：

~~~c
if (p != NULL) {
    consume_payload(p);     /* 调用方函数，占位符 */
    pbuf_free(p);
    p = NULL;
}
~~~

### `tcp_close`

- **作用**：请求正常关闭 TCP PCB。
- **函数原型/签名**：`err_t tcp_close(struct tcp_pcb *pcb);`
- **参数含义详解**：`pcb` 为待关闭控制块；关闭后不能继续使用该 PCB。
- **返回值状态码**：成功通常为 `ERR_OK`；资源不足或状态不允许时返回对应 `err_t`。
- **使用 Demo**：

~~~c
err_t err = tcp_close(pcb);
if (err != ERR_OK)
    return err;             /* 由调用方决定稍后重试或 abort */
pcb = NULL;
~~~

### `tcp_abort`

- **作用**: 立即异常终止 PCB, 不执行正常 TCP 关闭握手.
- **函数原型/签名**: `void tcp_abort(struct tcp_pcb *pcb);`
- **参数含义详解**: `pcb` 是待终止的连接控制块, 调用后该 PCB 立即失效, 调用方不得继续访问它.
- **返回值状态码**: 无返回值. 失败处理通过调用方在调用前的状态判断完成.
- **使用 Demo**:

~~~c
if (err != ERR_OK) {
    tcp_abort(pcb);
    pcb = NULL;
}
~~~

### `tcp_bind`

- **作用**：把 TCP PCB 绑定到本地 IP 地址和端口。
- **函数原型/签名**：`err_t tcp_bind(struct tcp_pcb *pcb, const ip_addr_t *ipaddr, u16_t port);`
- **参数含义详解**：`pcb` 是服务端 PCB；`ipaddr` 可使用 `IP_ADDR_ANY`；`port` 是本地监听端口。
- **返回值状态码**：成功返回 `ERR_OK`；端口已占用常见为 `ERR_USE`。
- **使用 Demo**：

~~~c
struct tcp_pcb *pcb = tcp_new();
if (pcb == NULL)
    return ERR_MEM;
err_t err = tcp_bind(pcb, IP_ADDR_ANY, 5001);
if (err != ERR_OK)
    tcp_close(pcb);
~~~

### `tcp_listen`

- **作用**：把已绑定的 PCB 转换为监听 PCB。
- **函数原型/签名**：`struct tcp_pcb *tcp_listen(struct tcp_pcb *pcb);`
- **参数含义详解**：`pcb` 必须已经成功 `tcp_bind()`；转换后通常使用返回的新监听 PCB。
- **返回值状态码**：成功返回监听 PCB；资源不足时可能返回 `NULL`，原 PCB 的处理需按版本约定执行。
- **使用 Demo**：

~~~c
struct tcp_pcb *listen_pcb = tcp_listen(pcb);
if (listen_pcb == NULL)
    return ERR_MEM;
pcb = listen_pcb;
tcp_accept(pcb, on_accept);
~~~

### `tcp_accept`

- **作用**：为监听 PCB 注册新连接回调。
- **函数原型/签名**：`void tcp_accept(struct tcp_pcb *pcb, tcp_accept_fn accept);`
- **参数含义详解**：`pcb` 是监听 PCB；`accept` 的典型签名为 `err_t (*)(void *, struct tcp_pcb *, err_t)`。
- **返回值状态码**：无返回值；新连接处理结果由回调返回 `err_t`。
- **使用 Demo**：

~~~c
static err_t on_accept(void *arg, struct tcp_pcb *newpcb,
                       err_t err);
tcp_arg(listen_pcb, user_arg);
tcp_accept(listen_pcb, on_accept);
~~~

`on_accept` 是调用方回调占位符，不是 LwIP API。

## 最小流程 Demo

### 客户端

~~~c
struct tcp_pcb *pcb = tcp_new();
IP4_ADDR(&server_ip, 192, 168, 1, 20);
tcp_recv(pcb, on_recv);
tcp_connect(pcb, &server_ip, 5001, on_connected);
~~~

### 服务端

~~~c
struct tcp_pcb *pcb = tcp_new();
tcp_bind(pcb, IP_ADDR_ANY, 5001);
pcb = tcp_listen(pcb);
tcp_accept(pcb, on_accept);
~~~

以上 `on_recv`、`on_connected` 和 `on_accept` 都是调用方回调占位符，不是官方 API。

## 避坑红线

- `pbuf` 使用完成后必须释放；异常路径也不能遗漏 `pbuf_free()`。
- `tcp_recved()` 的长度应与实际处理的字节数一致，不能把未处理的 pbuf 长度提前确认。
- `tcp_write()` 必须检查返回值；不确定数据缓冲区生命周期时优先使用 `TCP_WRITE_FLAG_COPY`。
- `tcp_close()` 失败时不能简单丢弃 PCB；应按连接状态和工程版本决定重试或 `tcp_abort()`。
- RAW API 回调通常运行在 LwIP TCP/IP 上下文，回调中不能阻塞，也不应从其他线程直接并发操作同一 PCB。
- 回调函数签名、错误码和宏以工程采用版本的 `lwip/tcp.h`、`lwip/err.h` 为准。

## 相关页面

- [[LwIP]]
- [[TCP IP 协议栈]]
- [[TCP 连接建立与释放]]
- [[STM32 CubeMX 配置 LwIP]]
- [[LwIP-知识地图]]

## 来源

- raw 文件：`【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md`
