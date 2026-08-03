# ESP32-S3 IMU 启动时序

## 概念一句话
ESP32-S3 IMU 启动时序的核心是先让 **SPI、任务、RingBuffer、GPIO ISR 和 DMA** 就绪，再让 LSM6DSR 进入 FIFO Continuous，从而避免第一次 watermark 事件没有消费者；核心关联是 [[LSM6DSR SPI 驱动]] 与 [[LSM6DSR FIFO 采集]]。

## 核心原理与图解

### 硬件与软件时序链

1. **SPI 总线准备**：ESP32-S3 使用 SPI2，项目配置的 SCLK/MOSI/MISO/CS 为 GPIO4/5/6/7，SPI Mode 3、10 MHz。
2. **身份闸门**：读取 LSM6DSR `WHO_AM_I`（地址 `0x0F`），素材中的验收值为 `0x6B`；SPI 交易失败或 ID 不匹配时不继续启动 FIFO。
3. **软件复位与基础配置**：向 `CTRL3_C`（`0x12`）写入 `0x01`，等待并轮询复位位；随后向 `CTRL9_XL`（`0x18`）写入 `0x02` 关闭 I3C，再向 `CTRL3_C` 写入 `0x44` 开启 BDU 与地址自动递增。
4. **消费者先行**：创建 32 KiB `RINGBUF_TYPE_NOSPLIT`、创建 FIFO 任务，并让任务句柄可被 GPIO ISR 使用。
5. **中断与 DMA 就绪**：配置 GPIO15 上升沿、安装 ISR 服务、添加 ISR handler，再预分配带 `MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL` 的 FIFO 缓冲区。
6. **最后启动传感器数据流**：写入 FIFO 配置并开启 Continuous；此后 watermark 事件才会进入已准备好的 ISR → 任务 → DMA 链路。
7. **网络异步启动**：Wi-Fi 初始化和 TCP 任务在采集路径准备完成后启动，不把建链阻塞放进 FIFO 消费临界路径。

~~~mermaid
sequenceDiagram
    participant App as "app_main"
    participant HW as "LSM6DSR"
    participant Task as "FIFO 任务"
    participant ISR as "GPIO15 ISR"
    participant Net as "TCP 任务"
    App->>HW: "SPI 初始化与 WHO_AM_I 重试"
    App->>HW: "CTRL3_C 复位与基础配置"
    App->>Task: "创建任务与 RingBuffer"
    App->>ISR: "配置 GPIO15 并注册 ISR"
    App->>App: "预分配 DMA-capable 缓冲区"
    App->>HW: "配置并开启 FIFO Continuous"
    HW-->>ISR: "watermark 上升沿"
    ISR-->>Task: "vTaskNotifyGiveFromISR"
    Task-->>HW: "读取 FIFO_STATUS1/2 与 FIFO_DATA_OUT_TAG"
    Task->>Net: "提交完整 TCP frame"
~~~

> 图 1：从 SPI 身份校验到 FIFO watermark 消费的启动先后关系；关键点是 FIFO Continuous 位于 ISR、任务和 DMA 准备之后。

### 已确认的寄存器与硬件响应

| 寄存器 | 地址 | 项目配置/检查 | 写入后硬件行为 |
|---|---:|---|---|
| `WHO_AM_I` | `0x0F` | 读取并期望 `0x6B` | 用于确认 SPI 通路和器件身份；失败时停止后续启动 |
| `CTRL3_C` | `0x12` | `0x01` → 复位；之后 `0x44` | 先触发软件复位；复位完成后开启 BDU 与地址自动递增 |
| `CTRL9_XL` | `0x18` | `0x02` | 关闭 I3C，保持 SPI 访问边界 |
| `FIFO_CTRL4` | `0x0A` | `0x00` → `0x06` | 先 Bypass 清旧 FIFO，再按项目配置进入 Continuous |
| `INT1_CTRL` | `0x0D` | `0x08` | 将 FIFO threshold 事件映射到 INT1 |

## 关键实现/数据结构

以下片段保留项目函数名，但它们是**项目自定义封装**；真正的官方底层接口见 [[LSM6DSR ESP-IDF API 速查]]。

~~~c
ESP_ERROR_CHECK(lsm6dsr_write_reg(CTRL3_C, 0x01)); // 项目封装：触发复位
vTaskDelay(pdMS_TO_TICKS(10));                    // raw 明确使用的等待
ESP_ERROR_CHECK(lsm6dsr_read_reg(CTRL3_C, &reg)); // 项目封装：轮询复位位
ESP_ERROR_CHECK(lsm6dsr_write_reg(CTRL9_XL, 0x02));
ESP_ERROR_CHECK(lsm6dsr_write_reg(CTRL3_C, 0x44));
xTaskCreatePinnedToCore(imu_irq_test_task, "imu", 4096,
                        NULL, 10, &s_imu_irq_task_handle, 1);
~~~

## 横向对比与关联

- [[LSM6DSR SPI 驱动]]：负责 SPI2、寄存器命令和 `RX[1]` 数据偏移；本页只描述它在启动链中的位置。
- [[LSM6DSR FIFO 采集]]：负责 Continuous、watermark、状态读取和 FIFO drain。
- [[IMU 采集与网络解耦]]：采集任务和 TCP 任务通过 RingBuffer 隔离。
- [[Wi-Fi TCP 状态流程]]：网络初始化虽在采集链路之后，但获得 IPv4 前不应进入 TCP 建链。

## 硬件避坑

- SPI 寄存器读命令需要置读标志位；命令阶段占用 `RX[0]`，真实寄存器值从 `RX[1]` 开始。
- FIFO DMA 传输长度必须包含 1 个命令字节；项目使用的单次读取上限为 256 words，每个 word 为 7 字节。
- FIFO 必须在 ISR、任务和 DMA 缓冲区准备完成后再切换到 Continuous，否则第一次 watermark 可能丢失。
- `CTRL3_C` 复位后不能立即假设配置已生效；raw 使用等待并轮询复位位，具体稳定时间仍以目标器件验证为准。

## 冲突与待核实

- raw 对高频 ODR/BDR 的日志标注为 6667 Hz，但同时要求通过长期统计和器件数据手册校准；本页不把该数值扩展为未经复核的通用结论。
- raw 的不同代码片段出现不同服务端 IP 示例；它们属于项目配置，不是启动时序的固定值。
- 失败回滚、任务删除、GPIO ISR 卸载和 DMA 释放尚未形成完整生命周期规范，不能把本页顺序当作完整资源管理方案。

## 来源

- raw 文件：`ESP32S3_IMU_项目完整总结.md`
