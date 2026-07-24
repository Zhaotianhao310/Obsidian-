# RMII 接口

## 概念

RMII 是 MAC 与 PHY 之间使用的简化媒体独立接口。它比 MII 使用更少的信号线，适合引脚资源有限的嵌入式硬件。

## 原素材中的配置

- 实验平台使用 STM32F407 与 LAN8720。
- 原素材指出 LAN8720 只支持 RMII，因此 CubeMX 的 ETH 接口模式选择 RMII。
- GPIO 配置必须按照实际开发板的原理图和引脚复用关系填写。

## 50 MHz 时钟要求

RMII 需要 50 MHz 参考时钟。原素材给出的两种情况是：

- LAN8720 使用 25 MHz 晶振，再由 PHY 内部 PLL 倍频得到 50 MHz。
- 如果硬件没有采用这种晶振方案，则需要由 MCU 的 MCO 等时钟输出为 PHY 提供 50 MHz。

## 排查顺序

1. 确认 PHY 是否支持 RMII。
2. 确认 ETH 引脚和 RMII 信号方向与原理图一致。
3. 确认 50 MHz 时钟来源和时序。
4. 确认 PHY 复位和 [[PHY 地址]]配置。
5. 硬件链路正常后，再排查 [[LwIP]]。

## 相关页面

- [[STM32 ETH 外设]]
- [[MAC 层与 PHY 层]]
- [[PHY 地址]]
- [[STM32 CubeMX 配置 LwIP]]
- [[STM32-以太网知识地图]]

## 来源

- raw 文件：【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md
