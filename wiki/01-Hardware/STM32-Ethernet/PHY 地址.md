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

地址错误时，MCU 可能读不到正确的 PHY ID 或链路状态，表现为初始化失败、链路异常或 ping 不通。原素材中的 LAN8720 案例把悬空 PHYAD 配置为 0，但该数值只适用于该硬件连接。

## 关键实现与配置

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