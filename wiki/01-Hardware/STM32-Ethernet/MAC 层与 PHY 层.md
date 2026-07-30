# MAC 层与 PHY 层

## 概念一句话精炼

MAC 负责组织和收发以太网帧，PHY 负责把数字接口转换为网线上的物理信号；二者通过 [[RMII 接口]] 或 MII 协作完成链路通信。

## 核心原理与图解

```mermaid
flowchart LR
    A[应用数据] --> B[LwIP 协议栈]
    B --> C[STM32 ETH MAC]
    C -->|RMII/MII| D[外部 PHY]
    D --> E[网线与以太网链路]
    D -->|链路状态/速率/双工| C
> 图 1：数据从 LwIP 进入 STM32 ETH MAC，经 RMII 到外部 PHY；接收方向沿相反路径返回，链路状态由 PHY 反馈给 MAC。
```


- MAC 侧处理帧缓存、目的地址、长度和 DMA 描述符。
- PHY 侧处理自动协商、物理编码、链路检测和 MII 管理寄存器。
## 硬件初始化与数据流

1. 确认 MCU 具有 ETH MAC，并根据板级连接选择 RMII 或 MII；具体型号能力以芯片数据手册为准。
2. 完成 ETH 时钟、GPIO 复用和外部 PHY 供电/复位；复位电平与保持时间未在 raw 中给出，标记为待核实。
3. 配置 MAC 接口模式和参考时钟，再初始化 TX/RX DMA 描述符及收发缓冲区。
4. 通过 MDC/MDIO 读取 PHY 身份和链路状态，确认速率/双工后再启动收发。
5. 发送方向为 LwIP pbuf → TX 描述符 → MAC → RMII/MII → PHY；接收方向为 PHY → MAC → RX 描述符 → ETH 驱动 → LwIP netif。

### 状态与寄存器边界

| 对象 | 地址/布局 | raw 可确认信息 | 硬件含义 |
|---|---|---|---|
| MAC 配置/状态寄存器 | 具体地址和位域待核实 | raw 只确认 ETH MAC 负责帧收发 | 控制接口模式、帧收发和链路相关状态 |
| TX/RX DMA 描述符 | 具体结构布局待核实 | raw 确认 DMA 通过描述符指向收发缓冲区 | 描述缓冲区、长度和收发所有权状态 |
| PHY 管理寄存器 | 具体寄存器地址待核实 | raw 只确认通过 MDIO 读取 PHY ID、链路/速率/双工状态 | 供 MAC 驱动判断 PHY 是否可用 |

不要把这里的 netif、ethif 或 eth_dma_init 等工程变量/封装函数误认为统一的 STM32 官方 API；具体 API 归属需按所用 HAL、ETH 驱动和 LwIP 版本分别核对。
- MAC 与 PHY 都正常，只能说明底层链路可用；TCP/IP 和应用回调仍需由 [[LwIP]] 配置。

## 关键实现与数据结构

> 以下为流程伪代码；函数名只表达硬件时序，不代表 STM32 HAL、ETH 驱动或 LwIP 官方 API。
```c
/* 伪代码：先复位 PHY，再初始化 MAC 和 DMA */
phy_reset_low();
phy_reset_high();
eth_config_rmii();                 // MAC 与 PHY 的接口模式一致
eth_dma_init(&rx_desc, &tx_desc);   // 描述符指向收发缓冲区
phy_link = phy_read_status();       // 读取链路、速率和双工状态
```

## 横向对比与关联

| 项目 | MAC | PHY |
|---|---|---|
| 主要职责 | 以太网帧和 DMA 收发 | 物理信号、链路协商 |
| 常见位置 | STM32 ETH 外设 | LAN8720、LAN8742、DP83848 |
| 关键排查点 | DMA、MAC 地址、缓存 | 复位、时钟、[[PHY 地址]] |

- [[STM32 ETH 外设]]：说明 MAC 在 STM32 中的集成方式。
- [[STM32 CubeMX 配置 LwIP]]：说明硬件配置如何进入协议栈。
- [[LwIP RAW API TCP 速查]]：说明上层如何使用 TCP。

## 来源

- raw 文件：`【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md`
- raw 文件：`【LWIP】初学STM32plusLWIPplus网络遇到的基础问题记录-stm3.md`
