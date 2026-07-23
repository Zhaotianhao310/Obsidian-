# STM32 CubeMX 配置 LwIP

## 定义

STM32 CubeMX 配置 LwIP 是指通过 CubeMX 生成 STM32 以太网、PHY 驱动和 [[LwIP]] 初始化框架，并在工程中补充 TCP 通信逻辑。

## 原素材实验条件

- MCU/开发板：STM32F407 相关板卡。
- PHY：LAN8720；原素材也提到 DP83848 可用类似方法测试。
- CubeMX 版本：原素材强调适用于 CubeMX 6.4，并提示 6.5 配置存在差异。
- 网络模式：PC 固定 IP，关闭防火墙，便于 ping 和 TCP 测试。

## CubeMX 配置要点

- SYS：无操作系统时可使用 SysTick；有操作系统时需按系统要求选择定时器。
- ETH：按硬件选择 [[RMII 接口]]，并正确配置 ETH 引脚。
- PHY：[[PHY 地址]]按 PHYAD 引脚实际连接填写。
- LwIP：使能 LwIP；调试时可关闭 DHCP，使用静态 IP、掩码和网关。
- 协议裁剪：若只测试 TCP，可关闭不需要的 UDP 等功能以简化工程。
- 调试：开启串口并重定向 printf，便于观察连接与回调状态。
- 时钟：RMII 需要 50 MHz 参考时钟，来源可为 PHY 晶振/PLL 或 MCU MCO。

## MDK/代码集成要点

- 生成 MDK 工程后，按工程选项补齐 C 库和调试下载配置。
- 主循环需要调用 CubeMX 生成的 LwIP 处理逻辑，确保协议栈周期性处理收发事件。
- PHY 初始化前建议执行硬件复位，避免 PHY 状态不确定。
- ping 通后，再进入 TCP 客户端/服务端逻辑验证。
- TCP 客户端和服务端可同时配置，使 STM32 同时具备主动连接与被连接能力。

## 相关链接

- [[LwIP]]
- [[LwIP RAW API TCP 速查]]
- [[STM32 ETH 外设]]
- [[MAC 层与 PHY 层]]
- [[RMII 接口]]
- [[PHY 地址]]
- [[LwIP-知识地图]]
- [[STM32-以太网知识地图]]

## 来源

- raw/【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md
