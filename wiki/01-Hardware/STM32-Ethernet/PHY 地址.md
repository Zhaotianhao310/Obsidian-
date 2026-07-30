# PHY 地址

## 概念一句话精炼

PHY 地址是 MCU 通过 MDIO/MDC 管理接口访问外部 PHY 寄存器时使用的硬件地址，必须与 PHYAD 引脚形成的实际地址一致，并关联 [[MAC 层与 PHY 层]]。

## 核心原理与图解

```mermaid
flowchart LR
    A[PHYAD 上拉/下拉/悬空] --> B[PHY 内部地址锁存]
    B --> C[MDIO/MDC 管理访问]
    C --> D[读取 PHY ID 与链路状态]
    D --> E[ETH/LwIP 初始化]
```

> 图 1：PHYAD 硬件电平形成 PHY 管理地址，随后通过 MDIO/MDC 读取身份和链路状态；地址错误会在管理访问阶段暴露。

地址错误时，MCU 可能读不到正确的 PHY ID 或链路状态，表现为初始化失败、链路异常或 ping 不通。原素材中的 LAN8720 案例把悬空 PHYAD 配置为 0，但该数值只适用于该硬件连接。

## 硬件初始化与状态读取时序

1. 根据原理图确认 PHYAD 上拉、下拉或悬空方案；不要把另一块板的地址直接复制过来。
2. 上电/硬件复位期间由 PHY 锁存地址的具体时刻和复位保持时间待核实。
3. 释放复位后等待 PHY 可响应 MDIO/MDC，再读取 PHY ID；若读值异常，优先检查地址、复位和 MDC/MDIO 连线。
4. 读取链路状态、自动协商或速率/双工状态，再让 MAC 按协商结果启动收发。

### PHY 管理寄存器映射

| 寄存器/状态对象 | 16 进制地址 | 关键位域/功能说明 | 典型读取/配置 | 硬件行为 |
|---|---:|---|---|---|
| PHY ID1 | 待核实 | 厂商/器件身份字段，具体位域待核实 | 读取并与数据手册比对 | 判断 MDIO 是否访问到目标 PHY |
| PHY ID2 | 待核实 | 型号/版本字段，具体位域待核实 | 读取并与数据手册比对 | 进一步确认 PHY 型号 |
| Basic Control | 待核实 | 复位、自动协商等字段待核实 | 配置项依 PHY 型号而定 | 控制 PHY 的复位/协商状态 |
| Basic Status | 待核实 | 链路、自动协商完成等字段待核实 | 运行时读取 | 向 MAC 反馈链路是否可用 |

raw 没有给出上述寄存器的具体地址和位值，因此这里只保留状态语义，不补写芯片专属地址。
## 关键实现与配置

> 以下为流程伪代码；board_phy_address、mdio_read 等是示意函数，不是 STM32 或 PHY 厂商官方 API。

```c
/* 伪代码：地址必须来自原理图和 PHY 数据手册 */
uint8_t phy_addr = board_phy_address();
assert(phy_addr == PHY_ADDR_FROM_SCHEMATIC);
phy_reset();
phy_id = mdio_read(phy_addr, PHY_ID1_REG);
link = mdio_read(phy_addr, PHY_BSR_REG);
```

## 横向对比与关联

- **PHY 地址**：解决“访问哪一颗 PHY”。
- **PHY 复位**：解决“PHY 是否进入确定状态”。
- **RMII 时钟**：解决“MAC 与 PHY 是否按同一时钟接口交换数据”。

排查顺序建议为：数据手册地址编码 → 开发板原理图 → 复位电平 → MDIO 读 ID → [[RMII 接口]]时钟与引脚。

- [[STM32 ETH 外设]]
- [[MAC 层与 PHY 层]]
- [[RMII 接口]]
- [[STM32 CubeMX 配置 LwIP]]

## 来源

- raw 文件：`【LWIP】stm32用CubeMX配置LwIPplusPingplusTCPcl.md`