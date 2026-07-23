# PHY 地址

## 定义

PHY 地址是 MAC/驱动通过管理接口访问外部 PHY 芯片时使用的硬件地址，通常由 PHYAD 等引脚的上下拉状态决定。

## STM32 配置含义

- CubeMX 的 PHY Address 必须与板级硬件上的 PHYAD 引脚状态一致。
- 原素材中的 LAN8720 板卡 PHYAD 引脚悬空，因内部弱下拉，配置为地址 0。
- 若地址配置错误，常见现象是 PHY 初始化失败、链路状态异常或无法 ping 通。

## 排查顺序

- 查 PHY 芯片手册确认地址引脚定义。
- 查原理图确认 PHYAD 引脚上拉、下拉或悬空状态。
- 在 CubeMX ETH 参数中填写对应 PHY Address。
- 与 [[RMII 接口]]、PHY 时钟和复位一起排查，避免把链路问题误判为 [[LwIP]] 协议栈问题。

## 相关链接

- [[MAC 层与 PHY 层]]
- [[RMII 接口]]
- [[STM32 ETH 外设]]
- [[STM32 CubeMX 配置 LwIP]]
- [[STM32-以太网知识地图]]

## 来源

- raw/【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md
