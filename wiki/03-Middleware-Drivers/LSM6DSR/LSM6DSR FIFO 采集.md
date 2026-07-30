# LSM6DSR FIFO 采集

## 概念一句话
LSM6DSR FIFO 采集通过 watermark、INT1 和 DMA 批量读取，把高频传感器数据从轮询模式转换为“事件触发 + 连续 drain”模式；核心关联是 [[LSM6DSR SPI 驱动]] 与 [[FIFO TAG 配对]]。

## 核心原理与图解
当前配置把加速度和陀螺仪都写入 FIFO，watermark 为 200 words，INT1 映射为 FIFO threshold。一个完整六轴 sample 需要一个 gyro word 和一个 accel word，因此 watermark 大约对应 100 个 sample。任务被唤醒后必须反复读取 FIFO_STATUS1/2，直到 unread words 为 0，不能只读固定的 200 words。

~~~mermaid
flowchart TD
    CFG["Bypass 清旧状态"] --> ODR["配置 XL/G ODR 与 FIFO BDR"]
    ODR --> WM["watermark=200 words"]
    WM --> RUN["Continuous FIFO"]
    RUN --> IRQ["INT1 threshold 上升沿"]
    IRQ --> STATUS["读取 FIFO_STATUS1/2"]
    STATUS -->|"overrun/full/读取失败"| RESET["清配对状态并 reset FIFO"]
    STATUS -->|"unread_words=0"| DONE["本次 drain 完成"]
    STATUS -->|"unread_words>0"| DMA["DMA 读取，单次最多 256 words"]
    DMA --> STATUS
~~~

> 图 1：FIFO 从启动、触发到持续排空及异常恢复的闭环；图示根据 raw 代码提炼，raw 未提供可直接引用的图片资源。

## 关键实现/数据结构
~~~c
typedef struct { // FIFO 状态快照
    uint16_t unread_words;
    bool watermark, overrun, full; // 水位与异常标志
} lsm6dsr_fifo_status_t;
while (status.unread_words > 0) {
    size_t n = MIN(status.unread_words, 256u); // 单次 DMA 上限
    lsm6dsr_fifo_read_words_dma(raw, n);
    lsm6dsr_fifo_get_status(&status);          // 继续 drain
    if (status.overrun || status.full) need_reset = true;
}
~~~

## 横向对比与关联
- [[LSM6DSR SPI 驱动]]：FIFO 读取仍遵循 SPI 命令字节和 RX[1] 起始偏移。
- [[FIFO TAG 配对]]：FIFO word 只是原始载荷，必须按 TAG 配对后才是完整 sample。
- [[IMU 采集与网络解耦]]：采集任务优先于网络任务，避免发送阻塞 FIFO 消费。
- 固定读取 watermark 适合简单演示；连续读取 status 更适合中断到任务之间可能继续积累数据的场景。

## 冲突与待核实
- raw 把寄存器值标注为 6667 Hz，但又要求通过长时间统计与数据手册校准；实际 ODR、FIFO BDR 和中断频率应人工复核。
- overrun 和 full 都会触发 reset，reset 会丢弃当前 FIFO 中未处理数据；这是可恢复性与数据完整性的明确取舍。

## 来源
- raw 文件：ESP32S3_IMU_项目完整总结.md
