# MAC 层与 PHY 层

## MAC 层

MAC 层负责以太网帧的介质访问控制，包括帧的发送与接收、MAC 地址相关处理，以及与 DMA 搬运配合完成数据收发。

## PHY 层

PHY 层负责把 MAC 侧的数据转换为网线上的物理信号，并处理链路建立、速率/双工状态等物理层状态。LAN8720、LAN8742、DP83848 都属于外部 PHY 芯片范畴。

## STM32 连接关系

- STM32 内置 ETH 时，通常由 STM32 提供 MAC，外部芯片提供 PHY。
- MAC 与 PHY 之间通过 MII 或 [[RMII 接口]]连接。
- PHY 还需要单独处理复位、参考时钟、PHY 地址和芯片差异寄存器。
- 原素材提示：PHY 初始化 GPIO、CLOCK、NVIC 和传输模式前，应先对 PHY 复位引脚执行一次拉低/拉高复位。

## 边界

MAC/PHY 链路正常，只能说明底层以太网硬件基本可用；还需要完成 [[STM32 CubeMX 配置 LwIP]]，并在应用层使用 [[LwIP RAW API TCP 速查]]进行 TCP 通信。

## 相关页面

- [[STM32 ETH 外设]]
- [[RMII 接口]]
- [[PHY 地址]]
- [[LwIP]]
- [[STM32-以太网知识地图]]

## 来源

- raw 文件：【LWIP】初学STM32plusLWIPplus网络遇到的基础问题记录-stm3.md
- raw 文件：【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md
