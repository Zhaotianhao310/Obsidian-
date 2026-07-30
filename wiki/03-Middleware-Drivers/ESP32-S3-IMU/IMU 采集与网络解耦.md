# IMU 采集与网络解耦

## 概念一句话
IMU 采集与网络解耦用高优先级采集任务生产完整 frame，再通过 32 KiB No-Split RingBuffer 交给网络任务消费，以隔离 SPI/FIFO 实时性和 Wi-Fi/TCP 抖动；核心关联是 [[LSM6DSR FIFO 采集]] 与 [[IMU TCP 固定帧]]。

## 核心原理与图解
imu_irq_test 固定在 Core 1、优先级 10，负责等待通知、读取 FIFO、解析和组帧；tcp_send 固定在 Core 0、优先级 8，负责等待 IPv4、连接 socket 和发送。RingBuffer 容量为 32 KiB，单帧 1216 字节，理论上约容纳 26 个完整 frame；网络长期不可用时，rb_drop 增长是当前丢新帧策略的预期结果。

~~~mermaid
flowchart LR
    P["Core 1 / imu_irq_test\n高优先级生产者"] -->|"完整 1216 字节 frame"| Q["32 KiB\nRINGBUF_TYPE_NOSPLIT"]
    Q -->|"完整 item"| C["Core 0 / tcp_send\n较低优先级消费者"]
    C --> NET["Wi-Fi + TCP socket"]
    Q -.-> DROP["满时：rb_drop++，丢弃新 frame"]
~~~

> 图 1：生产者—消费者边界以及 RingBuffer 满时的背压取舍；图示根据 raw 代码提炼，raw 未提供可直接引用的图片资源。

## 关键实现/数据结构
~~~c
#define FRAME_RINGBUFFER_SIZE (32 * 1024)

s_frame_ringbuffer = xRingbufferCreate(
    FRAME_RINGBUFFER_SIZE, RINGBUF_TYPE_NOSPLIT);
if (xRingbufferSend(s_frame_ringbuffer, &frame,
                   sizeof(frame), 0) != pdTRUE) {
    s_ringbuffer_drop_count++; // 不阻塞采集任务，丢弃新 frame
}
~~~

## 横向对比与关联
- [[LSM6DSR FIFO 采集]]：采集侧选择“不等待网络”，优先避免 FIFO overrun。
- [[IMU TCP 固定帧]]：No-Split 保证消费者拿到的是完整 frame，而不是碎片。
- [[Wi-Fi TCP 状态流程]]：Wi-Fi 未拿到 IP 或 TCP 重连期间，生产者仍可继续运行。
- 与阻塞式背压相比，当前策略保住采样实时性但牺牲网络不可用期间的数据完整性；可选方案包括降采样、只保留最新帧或本地缓存。

## 冲突与待核实
- RingBuffer 的理论容量不等于可用 item 数，管理开销会减少实际容量；不能据此承诺一定缓存 26 帧。
- raw 没有给出丢帧恢复协议；若上位机需要补偿，应利用 seq_num 和额外丢帧计数设计应用层策略。

## 来源
- raw 文件：ESP32S3_IMU_项目完整总结.md
