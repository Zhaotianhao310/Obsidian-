# IMU 采集与网络解耦

## 概念一句话
IMU 采集与网络解耦用高优先级 FIFO 任务生产完整 frame，再通过 32 KiB 的 `RINGBUF_TYPE_NOSPLIT` RingBuffer 交给网络任务消费，把 SPI/FIFO 的实时性与 Wi-Fi/TCP 抖动隔离开；核心关联是 [[LSM6DSR FIFO 采集]] 与 [[IMU TCP 固定帧]]。

## 核心原理与图解

### 生产者到消费者的数据流

1. GPIO15 INT1 上升沿到达 ISR。
2. ISR 只计数并通过 `vTaskNotifyGiveFromISR()` 唤醒 Core 1 的采集任务，不在中断上下文做 SPI 或 TCP。
3. 采集任务读取 `FIFO_STATUS1/2`，按未读 words 循环 DMA drain；异常时按项目策略 reset FIFO 并清配对状态。
4. FIFO TAG 配对输出六轴 sample，累计 100 组后组成 1216 字节 `tcp_frame_t`。
5. `xRingbufferSend(..., 0)` 不等待网络；失败时递增丢帧计数，避免采集任务被网络阻塞。
6. Core 0 的 TCP 任务等待 Wi-Fi IPv4 状态，取出完整 item，调用项目 `send_all()`，成功或失败后都归还 RingBuffer item。

~~~mermaid
flowchart LR
    IRQ["GPIO15 INT1 上升沿"] --> ISR["IRAM ISR\n只通知任务"]
    ISR --> P["Core 1 / imu_irq_test\n读取状态、DMA、TAG 配对"]
    P --> F["100 组 sample\n1216 字节 frame"]
    F --> Q["32 KiB No-Split RingBuffer"]
    Q --> C["Core 0 / tcp_send\n等待 IPv4、取 item"]
    C --> NET["Wi-Fi + TCP 字节流"]
    Q -.-> DROP["RingBuffer 满\n丢新 frame 并计数"]
~~~

> 图 1：硬件中断、FIFO 消费、固定帧生产和网络发送之间的生产者—消费者边界；RingBuffer 满时采用“丢新帧而不阻塞采集”的策略。

### 任务与硬件边界

- 采集任务优先级为 10、固定 Core 1；TCP 任务优先级为 8、固定 Core 0，这是 raw 中的项目配置，不是 FreeRTOS 固定规则。
- 32 KiB 是 RingBuffer 总容量，理论上 `32768 / 1216` 约为 26，但管理开销和 item 对齐会降低实际可用数量。
- 采集链路必须优先于网络链路，否则 FIFO 可能继续累积并最终 overrun。

## 关键实现/数据结构

~~~c
s_frame_ringbuffer = xRingbufferCreate(32 * 1024, RINGBUF_TYPE_NOSPLIT);
BaseType_t ok = xRingbufferSend(s_frame_ringbuffer, &frame,
                                sizeof(frame), 0); // 不等待网络
if (ok != pdTRUE) {
    s_ringbuffer_drop_count++;                 // 丢新帧
}
void *item = xRingbufferReceive(s_frame_ringbuffer,
                                &item_size, pdMS_TO_TICKS(1000));
if (item != NULL) vRingbufferReturnItem(s_frame_ringbuffer, item);
~~~

代码中的 `frame`、`s_ringbuffer_drop_count` 和 `send_all()` 属于项目业务对象/函数；`xRingbuffer*` 才是官方 FreeRTOS RingBuffer API。

## 横向对比与关联

- [[LSM6DSR FIFO 采集]]：采集端关注水位、overrun 和 DMA drain。
- [[FIFO TAG 配对]]：把 FIFO 原始 word 转成可进入固定帧的六轴 sample。
- [[IMU TCP 固定帧]]：定义 RingBuffer item 的完整字节布局。
- [[Wi-Fi TCP 状态流程]]：Wi-Fi 未拿到 IPv4 或 TCP 重连期间，生产者仍可继续运行。
- 与阻塞式背压相比，当前方案保护采样实时性，但网络长期不可用时会牺牲数据完整性。

## 硬件与并发避坑

- ISR 中不要调用 SPI、RingBuffer 发送或阻塞 API；只做最小通知和必要的 ISR-safe 操作。
- DMA 缓冲区必须满足项目要求的 `MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL`，并在初始化失败路径释放。
- `xRingbufferReceive()` 返回的 item 无论发送成功还是失败都必须 `vRingbufferReturnItem()`。
- RingBuffer 满时的丢帧策略必须与序号、监控计数和上位机容错策略一致。

## 冲突与待核实

- RingBuffer 的理论容量不能承诺实际可缓存 26 帧。
- raw 没有给出丢帧补偿协议；若上位机需要重传或补偿，应基于 `seq_num` 和额外丢帧计数另行设计。

## 来源

- raw 文件：`ESP32S3_IMU_项目完整总结.md`
