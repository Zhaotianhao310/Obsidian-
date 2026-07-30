# LSM6DSR FIFO 采集

## 概念一句话
LSM6DSR FIFO 采集通过 watermark、INT1 和 DMA 批量读取，把高频传感器数据从轮询模式转换为“事件触发 + 连续 drain”模式；核心关联是 [[LSM6DSR SPI 驱动]] 与 [[FIFO TAG 配对]]。

## 核心原理与图解

### 初始化与驱动时序

1. `FIFO_CTRL4=0x00` 进入 Bypass，清除旧 FIFO 内容。
2. 配置 `CTRL1_XL`、`CTRL2_G` 的项目 ODR/量程值，并用 `FIFO_CTRL3` 设置 gyro/accel 的 FIFO BDR。
3. 将 watermark 200 words 写入 `FIFO_CTRL1/2`，并在 `INT1_CTRL` 打开 FIFO threshold 到 INT1 的映射。
4. 创建采集任务、配置 GPIO15 上升沿 ISR、安装任务通知路径并完成 DMA 缓冲区分配。
5. 写 `FIFO_CTRL4=0x06` 进入项目记录的 Continuous 模式；watermark 到达后由 INT1 唤醒任务。
6. 任务读取 `FIFO_STATUS1/2` 的未读 words，按最多 256 words 一批读取 `FIFO_DATA_OUT_TAG`，直到 unread words 为 0。
7. 如果状态报告 overrun/full 或 SPI 读取失败，项目策略是 reset FIFO、清空 TAG 配对状态并丢弃当前未处理 FIFO 数据。

~~~mermaid
flowchart TD
    B["FIFO_CTRL4=0x00\nBypass 清旧状态"] --> O["CTRL1_XL / CTRL2_G\n配置传感器 ODR/量程"]
    O --> R["FIFO_CTRL1/2\nwatermark=200 words"]
    R --> I["INT1_CTRL=0x08\n映射 threshold"]
    I --> D["任务、GPIO ISR、DMA 就绪"]
    D --> C["FIFO_CTRL4=0x06\n项目配置的 Continuous"]
    C --> W["INT1 watermark"]
    W --> S["读取 FIFO_STATUS1/2"]
    S -->|"异常：overrun/full/读失败"| X["项目函数 reset FIFO\n清 TAG 配对状态"]
    S -->|"unread_words > 0"| Q["项目函数 DMA 读取\n单次最多 256 words"]
    Q --> S
    S -->|"unread_words = 0"| E["本次 drain 完成"]
~~~

> 图 1：LSM6DSR FIFO 从 Bypass 清空、watermark 配置、INT1 触发到 DMA 连续排空的硬件—软件闭环；图中“项目函数”不是官方 API。

### 核心寄存器映射

| 寄存器 | 地址 | 关键位域/功能 | 项目配置/检查 | 写入或读取后的硬件行为 |
|---|---:|---|---|---|
| `CTRL1_XL` | `0x10` | `ODR_XL[7:4]`、`FS_XL` | raw 配置记录 `0xA4` | 启用加速度计并设置项目记录的 ODR/±16 g；ODR 数值需按冲突段落复核 |
| `CTRL2_G` | `0x11` | `ODR_G[7:4]`、`FS_G` | raw 配置记录 `0xAC` | 启用陀螺仪并设置项目记录的 ODR/±2000 dps；ODR 数值需复核 |
| `FIFO_CTRL1` | `0x07` | `WTM[7:0]` | watermark 低 8 位 | 设置 FIFO watermark 的低 8 位；1 LSB 对应一个 7 字节 FIFO word |
| `FIFO_CTRL2` | `0x08` | `WTM[8]` 等高位控制 | watermark 高位 | 与 `FIFO_CTRL1` 合成 watermark；其他位按目标配置复核 |
| `FIFO_CTRL3` | `0x09` | `BDR_G[7:4]`、`BDR_XL[3:0]` | raw 配置记录 `0xAA` | 选择 gyro/accel 写入 FIFO 的批处理数据率 |
| `FIFO_CTRL4` | `0x0A` | `FIFO_MODE[2:0]` | `0x00` → `0x06` | Bypass 用于清 FIFO；`0x06` 对应项目记录的 Continuous 模式 |
| `INT1_CTRL` | `0x0D` | `INT1_FIFO_TH` bit3 | `0x08` | 将 FIFO watermark threshold 事件送到 INT1 |
| `FIFO_STATUS1/2` | `0x3A/0x3B` | `DIFF_FIFO[9:0]`、watermark/overrun/full 状态 | 每次 drain 前读取 | 得到未读 FIFO word 数和异常状态 |
| `FIFO_DATA_OUT_TAG` | `0x78` | `TAG_SENSOR[4:0]` 在 bit[7:3] | 连续读取 | 每个 word 的首字节识别 gyro/accel 等数据来源 |

## 关键实现/数据结构

以下是数据流核心；`lsm6dsr_fifo_get_status`、`lsm6dsr_fifo_read_words_dma` 和 `lsm6dsr_fifo_reset` 是项目自定义封装，不是官方 API。

~~~c
do {
    lsm6dsr_fifo_status_t st;              // 项目状态结构
    if (lsm6dsr_fifo_get_status(&st) != ESP_OK) break;
    if (st.overrun || st.full) { lsm6dsr_fifo_reset(); break; }
    size_t n = MIN(st.unread_words, 256u); // 项目单次读取上限
    if (n) lsm6dsr_fifo_read_words_dma(raw, n);
} while (st.unread_words > 0);
~~~

FIFO 原始 word 交给 [[FIFO TAG 配对]]，不能在这里直接假设一个 word 就是完整六轴样本。

## 横向对比与关联

- [[LSM6DSR SPI 驱动]]：提供 SPI 命令、DMA 事务和 `RX[1]` 偏移规则。
- [[FIFO TAG 配对]]：把 7 字节 raw word 解析成有序的 gyro/accel sample。
- [[IMU 采集与网络解耦]]：采集任务先排空 FIFO，再把完整 frame 交给网络任务。
- [[ESP32-S3 IMU 启动时序]]：要求任务、ISR 和 DMA 先准备，再开启 FIFO Continuous。

## 硬件避坑

- watermark 单位是 FIFO word，不是六轴 sample；当前项目 200 words 约对应 100 组 gyro/accel 配对，但实际顺序仍由 TAG 决定。
- Continuous 模式下，新数据到达时可能覆盖最旧数据；检测到 overrun 后继续使用当前 FIFO 会破坏样本完整性，项目选择 reset 并丢弃未处理数据。
- FIFO 读取必须按 7 字节 word 和命令字节对齐；批量 DMA 不能只传输 6 字节轴数据。
- 状态读取、DMA 读取和 TAG 配对必须在同一消费循环中连续推进，不能只读取固定的 200 words。

## 冲突与待核实

- raw 把 `CTRL1_XL/CTRL2_G/FIFO_CTRL3` 的配置描述为 6667 Hz，但同时要求用长期统计和数据手册校准；本页保留这一冲突。
- `FIFO_CTRL4=0x06` 的 Continuous 解释来自项目当前配置；升级器件或切换模式时必须重新核对模式编码和状态位。

## 来源

- raw 文件：`ESP32S3_IMU_项目完整总结.md`
