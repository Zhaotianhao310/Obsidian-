# Wi-Fi TCP 状态流程

## 概念一句话
Wi-Fi TCP 状态流程把“已获取 IPv4”作为 TCP 连接前置条件，并在断线事件中清除状态、自动重连，从而避免网络任务在无地址状态下空转；核心关联是 [[IMU 采集与网络解耦]] 与 [[IMU TCP 固定帧]]。

## 核心原理与图解
初始化顺序为 NVS、事件组、网络接口、默认事件循环、STA netif、Wi-Fi 驱动和事件处理器。只有 IP_EVENT_STA_GOT_IP 设置 WIFI_CONNECTED_BIT 后，TCP 任务才创建/重连 socket；断开时清除该位并重新发起 Wi-Fi 连接。关闭 Wi-Fi 省电模式可降低吞吐链路延迟抖动，但增加功耗。

~~~mermaid
stateDiagram-v2
    [*] --> 初始化
    初始化 --> 等待IPv4: "启动 STA 并注册事件"
    等待IPv4 --> TCP可连接: "IP_EVENT_STA_GOT_IP"
    TCP可连接 --> 发送: "connect() 成功"
    发送 --> TCP可连接: "send_all() 成功"
    发送 --> 等待IPv4: "TCP 断开/发送失败"
    等待IPv4 --> 等待IPv4: "WIFI_EVENT_STA_DISCONNECTED，清位并重连"
~~~

> 图 1：Wi-Fi 事件位控制 TCP 任务的状态转换；图示根据 raw 代码提炼，raw 未提供可直接引用的图片资源。

## 关键实现/数据结构
~~~c
case IP_EVENT_STA_GOT_IP:
    xEventGroupSetBits(s_wifi_event_group,
                       WIFI_CONNECTED_BIT); // 允许 TCP 任务工作
    break;
case WIFI_EVENT_STA_DISCONNECTED:
    xEventGroupClearBits(s_wifi_event_group,
                         WIFI_CONNECTED_BIT); // 阻止无地址连接
    esp_wifi_connect();
    break;
~~~

## 横向对比与关联
- [[IMU 采集与网络解耦]]：网络状态不应阻塞高频 FIFO 消费任务。
- [[IMU TCP 固定帧]]：TCP 只是字节流，必须由 send_all() 和接收端缓存共同保证帧边界。
- [[ESP32-S3 IMU 启动时序]]：网络在传感器采集链路准备之后初始化。
- 与直接在 app_main() 中同步连接 TCP 相比，事件位方案能把 Wi-Fi 获取 IP 和 TCP 建链解耦。

## 冲突与待核实
- raw 使用固定服务端 IP 192.168.18.70 和端口 8080，这些是项目配置示例，不是通用默认值。
- WIFI_PS_NONE 的吞吐/延迟收益与功耗代价需要在目标产品上实测，不能无条件视为最佳配置。

## 来源
- raw 文件：ESP32S3_IMU_项目完整总结.md
