# STM32 ETH 外设

## 概念

STM32 ETH 是部分 STM32 芯片集成的以太网外设，主要承担以太网 MAC 层的数据帧收发，并通过 DMA 搬运数据。

## 选型判断

- 原素材以 STM32F407 为例，说明该系列具备 ETH 外设。
- 原素材将 STM32F103 作为没有 ETH 外设的对比例子；实际选型仍应以具体型号数据手册为准。
- 带 ETH 的 MCU 通常需要外接 PHY 芯片，例如 LAN8720、LAN8742 或 DP83848。
- 不带 ETH 的 MCU 可以考虑 W5500 等集成 MAC 与 PHY 的外部芯片，由 STM32 通过 SPI 访问。

## 与协议栈的关系

STM32 ETH 只解决底层硬件接口的一部分问题，不等于完整 TCP/IP 协议栈。典型链路是：

[[STM32 ETH 外设]] → [[MAC 层与 PHY 层]] → [[LwIP]] → TCP 应用代码

## 相关页面

- [[MAC 层与 PHY 层]]
- [[RMII 接口]]
- [[PHY 地址]]
- [[STM32 CubeMX 配置 LwIP]]
- [[STM32-以太网知识地图]]

## 来源

- raw 文件：【LWIP】初学STM32plusLWIPplus网络遇到的基础问题记录-stm3.md
