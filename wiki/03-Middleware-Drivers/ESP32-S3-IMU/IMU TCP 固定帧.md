# IMU TCP 固定帧

## 概念一句话
IMU TCP 固定帧把 100 组六轴原始 sample 封装为 1216 字节的小端二进制帧，用 magic、长度语义、序号和时间戳支撑上位机重组与丢帧定位；核心关联是 [[FIFO TAG 配对]] 与 [[IMU 采集与网络解耦]]。

## 核心原理与图解
帧头为 magic + count + seq_num + timestamp_us，后面是 100 组 Ax, Ay, Az, Gx, Gy, Gz。ESP32-S3 是小端 CPU，接收端不能假设一次 recv() 就得到一帧，必须缓存字节、寻找 magic、等待 1216 字节并处理半包/粘包。

~~~mermaid
flowchart LR
    S["六轴 sample"] --> C["累计 100 组"]
    C --> H["写入 frame 头：magic/count/seq/timestamp"]
    H --> A["_Static_assert(sizeof == 1216)"]
    A --> RB["No-Split RingBuffer item"]
    RB --> SEND["send_all()"]
    SEND --> RX["接收端字节缓存"]
    RX --> SYNC["寻找 magic 并等待 1216 字节"]
    SYNC --> PARSE["按 little-endian 解码"]
~~~

> 图 1：固定帧从组装、发送到接收端重组的字节流流程；图示根据 raw 代码提炼，raw 未提供可直接引用的图片资源。

## 关键实现/数据结构
~~~c
typedef struct __attribute__((packed)) {
    uint16_t magic;                    // 0x55AA，用于帧同步
    uint16_t count;                    // 当前固定为 100
    uint32_t seq_num;                  // 应用层连续序号
    uint64_t timestamp_us;             // watermark 处理时刻
    int16_t samples[100][6];            // Ax..Gz，共 1200 字节
} tcp_frame_t;

_Static_assert(sizeof(tcp_frame_t) == 1216, "frame size mismatch");
~~~

## 横向对比与关联
- [[FIFO TAG 配对]]：决定 samples 中一行六轴数据何时成立。
- [[IMU 采集与网络解耦]]：RingBuffer item 保持整帧，不拆分成多个生产者项目。
- [[Wi-Fi TCP 状态流程]]：网络断开时 frame 可能在发送失败后丢弃，seq_num 用于定位缺口。
- 与文本/JSON 协议相比，固定二进制帧开销小、解析快，但对字节序、结构体布局和版本兼容更敏感。

## 冲突与待核实
- 当前 timestamp_us 是 watermark 通知被任务处理时的单调时间，不是每个 sample 的独立采样时刻。
- raw 采用固定 count=100 和 1216 字节布局；若增加版本、长度、CRC 或设备号，必须同步修改发送端、接收端和静态断言。

## 来源
- raw 文件：ESP32S3_IMU_项目完整总结.md
