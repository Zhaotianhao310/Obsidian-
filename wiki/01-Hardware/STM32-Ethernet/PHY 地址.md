# PHY 地址

## 概念

PHY 地址是 MCU 通过 PHY 管理接口访问外部 PHY 芯片时使用的地址。它通常由 PHYAD 等硬件引脚的上拉、下拉或悬空状态决定。

## CubeMX 配置规则

CubeMX 中填写的 PHY Address 必须与硬件实际形成的地址一致。地址错误时，驱动可能无法读写 PHY 寄存器，进而表现为 PHY 初始化失败、链路状态异常或 ping 不通。

## 原素材案例

原素材中的 LAN8720 开发板将 PHYAD 引脚悬空；由于该引脚带内部弱下拉，作者将 PHY Address 配置为 0。

> 该数值只适用于原素材所描述的硬件连接，不应直接套用到其他开发板。

## 排查方法

- 先查看 PHY 芯片数据手册中的 PHYAD 编码表。
- 再查看开发板原理图，确认引脚是上拉、下拉还是悬空。
- 最后把得到的地址填写到 CubeMX，并结合 PHY 复位、时钟和 [[RMII 接口]]一起验证。

## 相关页面

- [[STM32 ETH 外设]]
- [[MAC 层与 PHY 层]]
- [[RMII 接口]]
- [[STM32 CubeMX 配置 LwIP]]
- [[STM32-以太网知识地图]]

## 来源

- raw 文件：【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md
