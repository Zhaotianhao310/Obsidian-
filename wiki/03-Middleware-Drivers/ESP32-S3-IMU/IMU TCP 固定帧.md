# IMU TCP 固定帧

## 概念一句话
IMU TCP 固定帧把 100 组六轴原始 sample 封装为固定长度的小端二进制帧，用 magic、样本数、序号和时间戳帮助接收端重同步与定位丢帧；核心关联是 [[FIFO TAG 配对]] 与 [[IMU 采集与网络解耦]]。

## 核心原理与图解

### 帧布局与数据流

- 帧头：`magic` 2 字节、`count` 2 字节、`seq_num` 4 字节、`timestamp_us` 8 字节，共 16 字节。
- 载荷：100 组 `int16_t` 六轴数据，每组 6 × 2 字节，共 1200 字节。
- 项目静态断言要求总大小为 1216 字节；是否保持该布局取决于 `packed` 和编译器 ABI，接收端不能只凭 C 结构体猜测。
- TCP 是字节流：一次 `send()` 或接收回调不保证对应一整帧，必须在应用层缓存、寻找 magic、等待完整长度后再解析。

~~~mermaid
flowchart LR
    S["TAG 配对后的六轴 sample"] --> C["累计 100 组"]
    C --> H["写入 magic/count/seq/timestamp"]
    H --> A["静态断言 1216 字节"]
    A --> RB["No-Split RingBuffer item"]
    RB --> SEND["项目 send_all 封装"]
    SEND --> RX["接收端字节缓存"]
    RX --> SYNC["寻找 magic 并等待 1216 字节"]
    SYNC --> PARSE["按 little-endian 解码"]
~~~

> 图 1：六轴 sample 经固定帧组装、RingBuffer 暂存、TCP 字节流发送后，在接收端通过 magic 和长度重新获得帧边界。

### 字节序与边界

ESP32-S3 项目按小端布局生成字段；网络接收端应明确使用小端解码，而不是直接把收到的字节强制转换为本机结构体。`magic` 只能用于候选同步，仍应同时检查 `count`、帧长度和序号连续性。

## 关键实现/数据结构

~~~c
typedef struct __attribute__((packed)) {
    uint16_t magic;                  // 项目帧同步字段
    uint16_t count;                  // raw 配置为 100
    uint32_t seq_num;                // 应用层连续序号
    uint64_t timestamp_us;           // watermark 处理时刻
    int16_t samples[100][6];         // Ax, Ay, Az, Gx, Gy, Gz
} tcp_frame_t;

_Static_assert(sizeof(tcp_frame_t) == 1216, "frame size mismatch");
~~~

`tcp_frame_t` 是项目自定义数据结构，不是 TCP 或 ESP-IDF API。`send_all()` 同样是项目自定义的完整发送封装，只在数据流图中作为业务边界标出。

## 横向对比与关联

- [[FIFO TAG 配对]]：决定何时产生一组完整六轴 sample。
- [[IMU 采集与网络解耦]]：RingBuffer 以整帧为 item，避免网络任务看到半帧。
- [[Wi-Fi TCP 状态流程]]：网络断开时，序号可以帮助接收端发现缺口，但不能自动恢复丢失 payload。
- 与 JSON/文本协议相比，固定二进制帧开销较低、解析较快，但对字节序、结构体布局和版本兼容更敏感。

## 硬件与协议避坑

- LSM6DSR FIFO 的 7 字节 word 必须先完成 TAG 配对，再进入 `samples[100][6]`；不能把单个 gyro/accel word 当成六轴样本。
- `timestamp_us` 是项目在 watermark 处理时记录的时间，不等于每个传感器样本的独立采样时间。
- TCP 发送接口的短写入、半包和粘包都必须在应用层处理；本页不把 `send_all()` 归类为官方 API。
- 若变更字段、样本数、版本号、CRC 或设备号，发送端、接收端和静态断言必须同步修改。

## 冲突与待核实

- raw 中不同片段对服务端配置示例存在差异；IP/端口属于项目配置，不能写入协议格式的固定语义。
- 1216 字节依赖当前 `packed` 结构和字段定义；跨编译器、跨语言实现时仍需用协议文档或显式序列化确认。

## 来源

- raw 文件：`ESP32S3_IMU_项目完整总结.md`
