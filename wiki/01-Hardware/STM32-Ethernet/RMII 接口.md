# RMII 接口

## 定义

RMII 是 STM32 MAC 与外部 [[MAC 层与 PHY 层|PHY 层]]芯片之间的一种简化以太网接口，用更少引脚完成 MAC-PHY 数据连接。

## 原素材中的应用

- 实验平台为 STM32F407 + LAN8720。
- LAN8720 只支持 RMII，因此 CubeMX 中 ETH 选择 RMII 模式。
- 原素材提到 LAN8720 使用 25 MHz 晶振并由内部 PLL 倍频得到 50 MHz 时钟。
- 若 PHY 没有自带/外部晶振方案，则需要通过 MCO 等方式为 RMII 提供 50 MHz 时钟。

## 检查清单

- ETH 模式选择 RMII。
- GPIO 引脚映射与板级原理图一致。
- 50 MHz 参考时钟来源明确。
- PHY 复位引脚时序可靠。
- [[PHY 地址]]与硬件 PHYAD 引脚状态一致。

## 相关链接

- [[STM32 ETH 外设]]
- [[MAC 层与 PHY 层]]
- [[PHY 地址]]
- [[STM32 CubeMX 配置 LwIP]]
- [[STM32-以太网知识地图]]

## 来源

- raw/【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md
