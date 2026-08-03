# Wi-Fi TCP 状态流程

## 概念一句话
Wi-Fi TCP 状态流程把“已获取 IPv4”作为 TCP 建链前置条件：事件处理器设置或清除连接位，TCP 任务据此等待、连接和重连；核心关联是 [[IMU 采集与网络解耦]] 与 [[IMU TCP 固定帧]]。

## 核心原理与图解

### 初始化与状态边界

raw 中确认的官方初始化顺序为：

1. `nvs_flash_init()`；仅在明确的 NVS 页不足或版本错误场景下擦除后重试。
2. `xEventGroupCreate()` 创建连接状态同步对象。
3. `esp_netif_init()` 和 `esp_event_loop_create_default()` 准备网络接口与默认事件循环。
4. `esp_netif_create_default_wifi_sta()` 创建默认 STA netif。
5. `esp_wifi_init()`、事件处理器注册、`esp_wifi_set_mode()`、`esp_wifi_set_config()`、`esp_wifi_start()`。
6. raw 配置 `esp_wifi_set_ps(WIFI_PS_NONE)`；功耗与延迟收益需要目标板实测。
7. 收到 `IP_EVENT_STA_GOT_IP` 后设置 `WIFI_CONNECTED_BIT`，TCP 任务才尝试连接服务端。
8. `WIFI_EVENT_STA_DISCONNECTED` 时清位并调用 `esp_wifi_connect()`，让 TCP 任务退出当前连接并重新等待 IPv4。

~~~mermaid
stateDiagram-v2
    [*] --> 初始化
    初始化 --> 等待IPv4: "STA 启动并注册事件"
    等待IPv4 --> TCP可连接: "IP_EVENT_STA_GOT_IP：置 WIFI_CONNECTED_BIT"
    TCP可连接 --> 发送: "TCP connect 成功"
    发送 --> TCP可连接: "发送成功，继续取 frame"
    发送 --> 等待IPv4: "断线或发送失败"
    等待IPv4 --> 等待IPv4: "WIFI_EVENT_STA_DISCONNECTED：清位并重连"
~~~

> 图 1：Wi-Fi 事件位是采集任务与 TCP 任务之间的状态边界；只有获得 IPv4 后，网络消费者才进入建链和发送阶段。

### 官方 API 与项目业务边界

- `esp_netif_init`、`esp_event_loop_create_default`、`esp_wifi_*` 和 `xEventGroup*` 是官方 API。
- `wifi_sta_init`、`wifi_connect`、`wifi_event_handler`、`tcp_connect_server`、`send_all` 是项目自定义函数或回调，不是官方 API。
- TCP socket 的 `socket`、`connect`、`send`、`setsockopt` 属于 POSIX/lwIP 接口，不在本页冒充 ESP-IDF API。

## 关键实现/数据结构

~~~c
case IP_EVENT_STA_GOT_IP:
    xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
    break; // 允许 TCP 任务工作
case WIFI_EVENT_STA_DISCONNECTED:
    xEventGroupClearBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
    esp_wifi_connect(); // 官方 API：请求重新关联
    break;
~~~

`s_wifi_event_group` 与事件分支属于项目状态机；`xEventGroupSetBits`、`xEventGroupClearBits` 和 `esp_wifi_connect` 才是官方接口。

## 横向对比与关联

- [[IMU 采集与网络解耦]]：网络状态不应阻塞高频 FIFO 消费任务。
- [[IMU TCP 固定帧]]：TCP 仍是字节流，固定帧边界由应用协议维护。
- [[ESP32-S3 IMU 启动时序]]：传感器接收路径先准备，网络随后启动。
- 与在 `app_main()` 中同步阻塞等待 TCP 相比，事件位方案把 Wi-Fi 获取 IPv4 和 TCP 建链解耦。

## 网络与硬件避坑

- 没有 `IP_EVENT_STA_GOT_IP` 时不要反复创建 TCP socket；应阻塞等待连接位。
- 断线时先清除连接位，再结束当前 TCP 会话，否则发送任务可能在无地址状态下继续工作。
- `WIFI_PS_NONE` 可能降低省电机制带来的延迟抖动，但会增加功耗；不能无条件视为最佳配置。
- NVS 擦除会丢失持久化配置，必须匹配明确错误码，不可作为普通失败的通用重试。

## 冲突与待核实

- raw 的不同代码片段出现 `192.168.18.70` 与 `192.168.18.55` 等服务端 IP 示例；它们都是项目配置，不能当作通用默认值。
- Wi-Fi 重连次数、退避策略和 TCP 关闭失败后的资源回收，raw 未形成完整规范，需要以当前固件版本和目标产品策略核实。

## 来源

- raw 文件：`ESP32S3_IMU_项目完整总结.md`
