# STM32 ETH 外设

## 定义

STM32 ETH 外设是部分 STM32 芯片内置的以太网控制器能力，负责介质访问控制，即 [[MAC 层与 PHY 层|MAC 层]]相关的数据收发。

## 关键判断

- 选型时先确认 MCU 是否带 ETH 外设：例如原素材提到 STM32F407 带 ETH，而 STM32F103 不带 ETH。
- 带 ETH 外设的 STM32 通常只需要外接 [[MAC 层与 PHY 层|PHY 芯片]]，如 LAN8720、LAN8742、DP83848。
- 不带 ETH 外设时，可考虑 W5500 这类同时集成 MAC 与 PHY 的以太网芯片，通过 SPI 与 STM32 通信。

## 工程影响

- 带 ETH 外设：STM32 内部负责 MAC 层，外部 PHY 通过 [[RMII 接口]]或 MII 接口连接。
- 不带 ETH 外设：网络芯片承担更多协议/链路功能，STM32 侧接口通常转为 SPI，软件栈和驱动结构会不同。
- 使用 [[LwIP]] 前，硬件链路必须先保证 MAC、PHY、时钟、复位、引脚和地址配置正确。

## 相关链接

- [[MAC 层与 PHY 层]]
- [[RMII 接口]]
- [[PHY 地址]]
- [[STM32 CubeMX 配置 LwIP]]
- [[LwIP]]
- [[STM32-以太网知识地图]]

## 来源

- raw/【LWIP】初学STM32plusLWIPplus网络遇到的基础问题记录-stm3.md
- raw/【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md
