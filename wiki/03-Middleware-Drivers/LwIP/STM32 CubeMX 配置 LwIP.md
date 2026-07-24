# STM32 CubeMX 配置 LwIP

## 适用范围

本页整理 raw 教程中 STM32F407、LAN8720 和 CubeMX 6.4 的一次配置流程。原文明确提示 CubeMX 6.5 的配置存在差异，不能把本页当作跨版本的固定操作手册。

## 配置前提

- PC 配置固定 IP，测试阶段按原文建议关闭防火墙。
- 确认 STM32 的 ETH 外设、PHY 型号、原理图引脚和复位电路。
- 确认 PHY 的 [[PHY 地址]]与硬件 PHYAD 状态一致。

## CubeMX 配置顺序

1. 选择目标 STM32 芯片；无操作系统时使用 SysTick，有操作系统时按系统要求配置定时器。
2. 在 ETH 中选择与硬件一致的接口模式；原素材使用 LAN8720，因此选择 [[RMII 接口]]。
3. 按板级连接配置 ETH 引脚、PHY 地址和 PHY 驱动参数。LAN8720 可参考 CubeMX 中 LAN8742 的部分通用配置，但差异项必须查芯片手册。
4. 使能 LwIP；为便于实验，原素材关闭 DHCP，使用静态 IP、掩码和网关，并关闭未使用的 UDP。
5. 配置串口，生成 MDK 工程，并保留便于添加用户代码的工程选项。

## 下载前检查

- RMII 需要 50 MHz 参考时钟；时钟可来自 PHY 晶振/PLL 或 MCU MCO，取决于硬件设计。
- PHY 初始化 GPIO、CLOCK、NVIC 和传输模式前，先对 PHY 复位引脚执行硬件复位。
- 主循环要调用 CubeMX 生成的 LwIP 处理函数；原文截图中的具体函数名未被文字转录，本页不擅自补写固定名称。
- 先用 ping 验证 IP 链路，再加入 TCP 客户端或服务端。

## TCP 验证

- 客户端：调用 [[LwIP RAW API TCP 速查]]中的 TCP 客户端初始化入口，并配置 PC 目标 IP 与端口 5001。
- 服务端：调用 TCP 服务端初始化入口，让 PC 网络助手连接板端 IP 与端口 5001。
- TCP 客户端和服务端代码可以同时加入工程，但应结合资源占用和业务需求决定是否启用。

## 相关页面

- [[STM32 ETH 外设]]
- [[MAC 层与 PHY 层]]
- [[RMII 接口]]
- [[PHY 地址]]
- [[LwIP]]
- [[LwIP RAW API TCP 速查]]
- [[STM32-以太网知识地图]]
- [[LwIP-知识地图]]

## 来源

- raw 文件：【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md
