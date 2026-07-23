# LwIP

## 定义

LwIP 是 Light weight IP 的缩写，是面向资源受限场景的小型开源 [[TCP IP 协议栈]]，常用于嵌入式网络。

## 设计目标

- 用较少 ROM/RAM 资源实现相对完整的 TCP/IP 功能。
- 通过宏配置裁剪不需要的功能。
- 可运行在有操作系统的环境，也可在无操作系统的裸机环境中运行。
- 为减少内存占用，LwIP 内部并不追求严格分层隔离，而是允许部分数据结构跨层可见，以降低数据拷贝。

## 能力范围

- 支持常见网络协议与应用能力，如 DHCP 客户端、DNS 客户端、HTTP 服务器等。
- 提供三类编程接口：RAW/Callback API、NETCONN API、Socket API。
- 在 STM32 场景中，LwIP 位于应用代码与底层 ETH/PHY 驱动之间。

## 与 STM32 的关系

- [[STM32 ETH 外设]]和 PHY 负责底层链路。
- [[STM32 CubeMX 配置 LwIP]]可生成初始化代码与工程框架。
- 业务侧可通过 [[LwIP RAW API TCP 速查|RAW API]]实现 TCP 客户端或服务端通信。

## 相关链接

- [[TCP IP 协议栈]]
- [[LwIP RAW API TCP 速查]]
- [[STM32 CubeMX 配置 LwIP]]
- [[STM32 ETH 外设]]
- [[LwIP-知识地图]]

## 来源

- raw/【LWIP】初学STM32plusLWIPplus网络遇到的基础问题记录-stm3.md
- raw/【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md
