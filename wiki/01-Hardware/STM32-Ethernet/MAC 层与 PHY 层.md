# MAC 层与 PHY 层

## 定义

在 STM32 以太网设计中，MAC 层负责以太网帧的介质访问控制，PHY 层负责物理链路上的电气/信号收发。

## STM32 场景

- 带 [[STM32 ETH 外设]]的芯片：STM32 内部提供 MAC 层能力，外部搭配 PHY 芯片。
- 常见外部 PHY：LAN8720、LAN8742、DP83848。
- MAC 与 PHY 之间常通过 [[RMII 接口]]或 MII 接口连接。
- PHY 芯片通常还涉及复位引脚、参考时钟、PHY 地址和驱动寄存器配置。

## 配置要点

- CubeMX 中 ETH 的接口模式必须匹配硬件连线与 PHY 芯片能力。
- LAN8720 原素材中使用 RMII，因为 LAN8720 只支持 RMII。
- PHY 初始化前需要关注复位：原素材建议在 PHY 层芯片初始化 GPIO、CLOCK、NVIC 和传输模式配置前执行一次硬件复位。
- PHY 驱动模板不一定完全等价：原素材中 LAN8720 可参考 LAN8742 的部分通用配置，但差异寄存器仍应查芯片手册。

## 相关链接

- [[STM32 ETH 外设]]
- [[RMII 接口]]
- [[PHY 地址]]
- [[STM32 CubeMX 配置 LwIP]]
- [[LwIP]]
- [[STM32-以太网知识地图]]

## 来源

- raw/【LWIP】初学STM32plusLWIPplus网络遇到的基础问题记录-stm3.md
- raw/【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md
