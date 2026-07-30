# ESP32S3_IMU 项目完整总结

> **文档版本**：v1.0  
> **整理日期**：2026-07-30  
> **项目路径**：`D:\MCU_Project\esp32\ESP32S3_IMU`  
> **目标芯片**：ESP32-S3  
> **开发框架**：ESP-IDF 5.5.5  
> **传感器**：LSM6DSR 六轴 IMU  
> **本文范围**：只总结 ESP32-S3 固件、LSM6DSR 驱动、Wi-Fi/TCP 数据链路和调试过程；**不包含 Python 服务端代码**。

---

## 1. 项目目标

本项目的目标是使用 ESP32-S3 通过 SPI 高速读取 LSM6DSR 的加速度和陀螺仪数据，利用 LSM6DSR FIFO 和 INT1 中断降低 CPU 轮询压力，再将整理后的 IMU 数据通过 Wi-Fi TCP 长连接发送到局域网中的上位机。

项目最终的数据处理链路如下：

```text
LSM6DSR
  │
  │ SPI2_HOST，Mode 3，10 MHz
  │
  ▼
LSM6DSR FIFO
  │
  │ FIFO watermark = 200 words
  │ INT1 上升沿
  ▼
ESP32-S3 GPIO15 ISR
  │
  │ FreeRTOS task notification
  ▼
imu_irq_test_task（Core 1）
  │
  ├─ 读取 FIFO_STATUS1/2
  ├─ DMA 批量读取 FIFO_DATA_OUT_TAG
  ├─ 按 FIFO TAG 区分 gyro / accel
  ├─ 组合成一个六轴 IMU sample
  └─ 每 100 个 sample 组装成一个 TCP frame
       │
       ▼
   FreeRTOS No-Split RingBuffer
       │
       ▼
tcp_send_task（Core 0）
  │
  ├─ 等待 Wi-Fi 获取 IPv4 地址
  ├─ 连接局域网 TCP server
  ├─ send_all() 保证整帧发送完成
  └─ 连接断开后重新连接
       │
       ▼
局域网上位机 TCP 接收程序
```

### 1.1 项目当前状态

当前固件已经完成以下功能：

- SPI 初始化以及 LSM6DSR `WHO_AM_I` 读取；
- 传感器基础配置；
- LSM6DSR FIFO 连续模式；
- FIFO watermark 中断；
- GPIO15 INT1 中断；
- FIFO DMA 批量读取；
- FIFO TAG 解析；
- 加速度和陀螺仪数据配对；
- 每 100 个六轴 sample 组成一个固定长度 TCP 帧；
- FreeRTOS RingBuffer 解耦采集速度和网络发送速度；
- Wi-Fi STA 模式连接；
- TCP 客户端自动重连；
- 每秒打印采集、FIFO、RingBuffer、TCP 统计信息。

当前没有把 Python 服务端纳入本项目总结。Python 文件如果仍存在于 `tools/`，本文只将其视为被排除的辅助工具，不属于固件架构的一部分。

---

## 2. 硬件连接与引脚定义

### 2.1 SPI 引脚

| 信号 | ESP32-S3 GPIO | LSM6DSR 端 | 说明 |
|---|---:|---|---|
| SCLK | GPIO4 | SCL/SPC | SPI 时钟 |
| MOSI | GPIO5 | SDA/SDI | ESP32 写入传感器 |
| MISO | GPIO6 | SDO | 传感器返回数据 |
| CS | GPIO7 | CS | 片选，低有效 |
| INT1 | GPIO15 | INT1 | FIFO watermark 中断 |
| INT2 | GPIO16 | INT2 | 当前代码已定义，但当前采集链路未使用 |

当前头文件中的定义为：

```c
#define PIN_NUM_CLK   4
#define PIN_NUM_MOSI  5
#define PIN_NUM_MISO  6
#define PIN_NUM_CS    7
#define PIN_NUM_INT1  15
#define PIN_NUM_INT2  16
#define SPI_CLOCK_HZ  (10 * 1000 * 1000)
```

### 2.2 必须确认的硬件条件

1. ESP32-S3 和 LSM6DSR 必须共地。
2. LSM6DSR 的供电电压必须满足模块规格，不能直接把不兼容的电平接到 ESP32-S3。
3. `CS` 必须连接到 GPIO7，不能悬空。
4. `INT1` 必须连接到 GPIO15；当前代码只在 GPIO15 上安装了中断处理函数。
5. 当前配置使用 SPI Mode 3，传感器和程序必须保持一致。
6. 如果模块的 `SDO/SA0` 或接口选择脚影响 SPI/I3C 模式，需要按模块原理图配置。
7. INT1 的电气极性和代码一致：当前使用 GPIO 上升沿触发。

---

## 3. 工程目录结构

```text
ESP32S3_IMU/
├── CMakeLists.txt                         # ESP-IDF 顶层 CMake
├── sdkconfig                              # ESP-IDF 当前配置快照
├── .gitignore                             # 忽略构建产物和本地 Wi-Fi 凭据
├── main/
│   ├── CMakeLists.txt                     # main 组件注册
│   ├── main.c                             # 应用主程序、FIFO 消费、Wi-Fi、TCP
│   ├── wifi_private.h                     # 本地真实 Wi-Fi 凭据，不应提交或写入文档
│   └── wifi_private.h.example             # Wi-Fi 凭据模板
├── components/
│   └── lsm6dsr/
│       ├── CMakeLists.txt                 # LSM6DSR 组件注册
│       ├── lsm6dsr.c                     # SPI、寄存器、FIFO、DMA 驱动
│       └── include/
│           └── lsm6dsr.h                 # 驱动 API、寄存器、引脚和 FIFO 常量
├── docs/
│   ├── LSM6DSR_FIFO_TCP_改造指导.md       # 早期改造指导
│   ├── LSM6DSR_FIFO_TCP_逐步实操指南.md   # 分阶段操作指南
│   └── ESP32S3_IMU_项目完整总结.md         # 本文档
├── .devcontainer/                         # ESP-IDF 容器开发配置
├── .vscode/                               # VS Code / ESP-IDF 配置
├── build/                                 # 构建产物，本文不纳入源代码
└── tools/                                 # 辅助工具目录；Python 服务端不纳入本文
```

### 3.1 源代码边界

本文完整附录包含以下固件代码和构建文件：

- `components/lsm6dsr/include/lsm6dsr.h`
- `components/lsm6dsr/lsm6dsr.c`
- `components/lsm6dsr/CMakeLists.txt`
- `main/main.c`
- `main/CMakeLists.txt`
- `main/wifi_private.h.example`
- 根目录 `CMakeLists.txt`
- 与开发环境有关的 `.gitignore`、`.devcontainer/` 和 `.vscode/` 配置

以下内容不复制到本文：

- `main/wifi_private.h` 的真实 SSID 和密码；
- `tools/` 下的 Python 服务端；
- `build/`、`.cache/`、编译生成的 ELF/BIN/MAP 等产物；
- `sdkconfig` 的完整生成内容，只在本文保留关键配置摘要。

---

## 4. 软件总体架构

### 4.1 ESP-IDF 组件关系

```text
ESP-IDF project
│
├── main component
│   └── main.c
│       ├── lsm6dsr component
│       ├── driver
│       ├── esp_timer
│       ├── esp_event
│       ├── esp_netif
│       ├── esp_wifi
│       ├── nvs_flash
│       └── lwip
│
└── lsm6dsr component
    └── lsm6dsr.c
        └── driver
```

### 4.2 任务和核心分配

当前代码显式创建两个 FreeRTOS 任务：

| 任务 | 核心 | 优先级 | 栈大小 | 主要职责 |
|---|---:|---:|---:|---|
| `imu_irq_test` | Core 1 | 10 | 4096 | 等待 INT1 通知、读取 FIFO、解析 sample、组帧 |
| `tcp_send` | Core 0 | 8 | 6144 | 等待 Wi-Fi、连接 TCP、从 RingBuffer 取帧并发送 |

这样分配的核心思路是：

- IMU 高频采集任务优先级更高，避免网络处理阻塞 FIFO 消费；
- Wi-Fi/TCP 任务放在另一个核心，网络抖动不会直接阻塞 SPI FIFO 读取；
- RingBuffer 作为生产者和消费者之间的缓冲区。

### 4.3 生产者—消费者模型

```text
IMU FIFO / INT1
      │
      ▼
imu_irq_test_task
      │ 生产固定 1216 字节 frame
      ▼
RINGBUF_TYPE_NOSPLIT RingBuffer，32 KiB
      │
      ▼
tcp_send_task
      │ 消费并发送
      ▼
TCP socket
```

RingBuffer 是 `RINGBUF_TYPE_NOSPLIT`，一个完整 frame 要么完整放入，要么发送失败，不会被拆成多个 RingBuffer item。这适合当前固定长度 TCP 帧。

当前 RingBuffer 容量是：

```c
#define FRAME_RINGBUFFER_SIZE (32 * 1024)
```

每个 TCP frame 为 1216 字节，理论上大约可以缓存 26 个完整 frame，实际可用容量还受到 RingBuffer 管理开销影响。因此，当网络长时间不可用或发送速度低于采集速度时，`rb_drop` 增长是当前设计的预期行为。

---

## 5. LSM6DSR SPI 驱动设计

驱动位于：

```text
components/lsm6dsr/lsm6dsr.c
components/lsm6dsr/include/lsm6dsr.h
```

### 5.1 SPI 初始化

`esp_lsm6dsr_spi_init()` 完成以下步骤：

1. 配置 SPI2 总线的 SCLK/MOSI/MISO；
2. 设置最大一次传输长度，确保可以覆盖 FIFO DMA 批量读取；
3. 使用 `SPI_DMA_CH_AUTO`；
4. 把 LSM6DSR 设备加入 SPI2 总线；
5. 配置 SPI 时钟 10 MHz；
6. 配置 SPI Mode 3；
7. 使用 GPIO7 作为 CS。

关键配置：

```c
spi_bus_config_t buscfg = {
    .miso_io_num = PIN_NUM_MISO,
    .mosi_io_num = PIN_NUM_MOSI,
    .sclk_io_num = PIN_NUM_CLK,
    .quadwp_io_num = -1,
    .quadhd_io_num = -1,
    .max_transfer_sz = FIFO_DMA_TRANSFER_BYTES,
};

spi_device_interface_config_t devcfg = {
    .clock_speed_hz = SPI_CLOCK_HZ,
    .mode = 3,
    .spics_io_num = PIN_NUM_CS,
    .queue_size = 7,
};
```

### 5.2 寄存器读写协议

代码使用 LSM6DSR 常见 SPI 寄存器访问格式：

- 写寄存器：寄存器地址最高位清零；
- 读寄存器：寄存器地址最高位置一；
- 读取时第一个返回字节是命令阶段数据，真正的寄存器值取 `rx_data[1]`。

```c
uint8_t tx_data[2] = {reg | 0x80, 0x00};
...
*value = rx_data[1];
```

这个字节偏移非常重要。若错误地读取 `rx_data[0]`，可能会把命令阶段的返回值误认为寄存器数据。

### 5.3 WHO_AM_I 检查

LSM6DSR 的预期 ID 为：

```c
#define WHO_AM_I  0x0F
#define LSM6DSR_ID 0x6B
```

`app_main()` 在传感器初始化后最多读取 10 次：

```text
WHO_AM_I attempt 1/10: 0x6B (expected 0x6B)
```

只有同时满足以下条件，程序才会继续：

- SPI 交易返回 `ESP_OK`；
- 读取值等于 `0x6B`。

如果一直失败，程序打印：

```text
No LSM6DSR response. Check wiring!
```

并退出 `app_main()`，不会继续启动 FIFO 和网络采集。

### 5.4 基础传感器配置

当前基础配置顺序为：

1. 向 `CTRL3_C` 写入 `0x01`，触发软件复位；
2. 等待复位完成；
3. 轮询 `CTRL3_C` 的复位位，直到硬件清零；
4. 向 `CTRL9_XL` 写入 `0x02`，关闭 I3C；
5. 向 `CTRL3_C` 写入 `0x44`，打开 BDU 和寄存器地址自动递增。

```c
ESP_ERROR_CHECK(lsm6dsr_write_reg(CTRL3_C, 0x01));
...
ESP_ERROR_CHECK(lsm6dsr_write_reg(CTRL9_XL, 0x02));
ESP_ERROR_CHECK(lsm6dsr_write_reg(CTRL3_C, 0x44));
```

### 5.5 FIFO word 格式

当前驱动按每个 FIFO word 7 字节处理：

```text
byte 0：FIFO TAG
byte 1：X 低字节
byte 2：X 高字节
byte 3：Y 低字节
byte 4：Y 高字节
byte 5：Z 低字节
byte 6：Z 高字节
```

对应宏：

```c
#define LSM6DSR_FIFO_WORD_SIZE 7u
```

FIFO TAG 的传感器部分取法：

```c
uint8_t tag_sensor = (word[0] >> 3) & 0x1F;
```

当前代码识别：

| TAG sensor 值 | 含义 |
|---:|---|
| `0x01` | Gyroscope |
| `0x02` | Accelerometer |
| 其他 | TAG 错误，增加 `s_tag_error_count` |

### 5.6 FIFO 高速配置

`lsm6dsr_fifo_config_high_rate()` 完成：

```text
FIFO_CTRL4 = 0x00       先进入 Bypass，清除旧 FIFO 状态
CTRL1_XL   = 0xA4       加速度计 6667 Hz，±16 g
CTRL2_G    = 0xAC       陀螺仪 6667 Hz，±2000 dps
FIFO_CTRL1 = watermark 低 8 位
FIFO_CTRL2 = watermark 高位
FIFO_CTRL3 = 0xAA       gyro/accel FIFO BDR 6667 Hz
INT1_CTRL  = 0x08       FIFO threshold 映射到 INT1
FIFO_CTRL4 = 0x06       Continuous 模式
```

Watermark 设置为 200 words：

```c
#define LSM6DSR_FIFO_WATERMARK_WORDS 200u
```

由于一个完整的六轴 sample 由一个 gyro word 和一个 accel word 配对得到，所以：

```text
200 FIFO words ≈ 100 个六轴 sample
6667 sample/s ÷ 100 sample/interrupt ≈ 66.7 interrupt/s
```

因此日志中出现约 62～64 次 INT1/s 是合理的，实际数值会受到 FIFO 消费时机、调度和日志开销影响。

### 5.7 FIFO DMA

驱动预先分配两块 DMA-capable、内部 RAM 缓冲区：

```c
s_fifo_tx_dma = heap_caps_malloc(
    FIFO_DMA_TRANSFER_BYTES,
    MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL);

s_fifo_rx_dma = heap_caps_malloc(
    FIFO_DMA_TRANSFER_BYTES,
    MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL);
```

最大批量读取 256 个 FIFO word：

```c
#define LSM6DSR_FIFO_MAX_WORDS_PER_READ 256u
```

因为 SPI 读 FIFO 时还要发送一个寄存器地址，所以 DMA 传输长度为：

```c
#define FIFO_DMA_TRANSFER_BYTES \
    (1u + LSM6DSR_FIFO_MAX_WORDS_PER_READ * LSM6DSR_FIFO_WORD_SIZE)
```

实际读取时，函数根据当前未读 word 数量决定本次读取多少个 word，最大不超过 256。读取完成后跳过 `RX[0]` 命令阶段数据，把 `RX[1]` 开始的 FIFO 内容复制到调用者缓冲区。

---

## 6. GPIO15 INT1 和 FIFO 消费状态机

### 6.1 ISR 的职责

GPIO15 ISR 不直接做 SPI 读取，不做复杂解析，只完成三件事：

1. 增加中断计数；
2. 使用 `vTaskNotifyGiveFromISR()` 通知 FIFO 任务；
3. 如有更高优先级任务被唤醒，执行 `portYIELD_FROM_ISR()`。

```c
static void IRAM_ATTR imu_int1_isr(void *arg)
{
    (void)arg;
    BaseType_t higher_priority_task_woken = pdFALSE;

    s_int1_irq_count++;
    vTaskNotifyGiveFromISR(s_imu_irq_task_handle,
                           &higher_priority_task_woken);

    if (higher_priority_task_woken == pdTRUE)
        portYIELD_FROM_ISR();
}
```

ISR 中不应该执行以下操作：

- SPI 事务；
- `malloc/free`；
- `ESP_LOGI()`；
- TCP 发送；
- 长循环。

这些操作都放到任务中完成，以避免阻塞中断和影响系统实时性。

### 6.2 GPIO 初始化

```c
gpio_config_t io_conf = {
    .pin_bit_mask = 1ULL << PIN_NUM_INT1,
    .mode = GPIO_MODE_INPUT,
    .pull_up_en = GPIO_PULLUP_DISABLE,
    .pull_down_en = GPIO_PULLDOWN_DISABLE,
    .intr_type = GPIO_INTR_POSEDGE,
};
```

当前代码只对 INT1 安装 ISR。INT2 GPIO16 已定义，但没有配置成中断输入，也没有安装 INT2 ISR。

### 6.3 FIFO 任务的主循环

`imu_irq_test_task()` 的处理过程：

```text
阻塞等待 task notification
        │
        ▼
记录本次 watermark 时间戳
        │
        ▼
循环读取 FIFO_STATUS1/2
        │
        ├─ status 读取失败 → 记录错误 → 请求 FIFO reset
        ├─ overrun/full → 记录溢出 → 请求 FIFO reset
        ├─ unread_words == 0 → 本次 FIFO 已排空，退出循环
        └─ unread_words > 0 → DMA 批量读取并解析
                              │
                              ▼
                   重新读取 FIFO_STATUS
```

代码不会在固定读取 200 words 后立即退出，而是重新读取 FIFO 状态，继续清理 ISR 处理期间新增的 FIFO word。这可以减少 FIFO backlog。

### 6.4 FIFO 错误恢复

以下情况会触发 `need_reset`：

- `lsm6dsr_fifo_get_status()` 失败；
- FIFO `overrun` 标志；
- FIFO `full` 标志；
- `lsm6dsr_fifo_read_words_dma()` 失败。

恢复时：

1. 清空当前 gyro/accel 配对状态；
2. 调用 `lsm6dsr_fifo_reset()`；
3. 如果 reset 失败，只记录错误并延时，不直接让整个应用崩溃。

这样做的原因是：FIFO 状态错误属于可恢复的数据通道故障，不应该因为一次 SPI/DMA 异常直接触发系统级 abort。

---

## 7. FIFO TAG 配对和 sample 组装

### 7.1 配对状态

FIFO 中 gyro 和 accel 是两个独立的 7-byte word。代码使用：

```c
typedef struct {
    imu_xyz_t accel;
    imu_xyz_t gyro;
    bool accel_valid;
    bool gyro_valid;
} imu_pair_state_t;
```

处理一个 word 时：

- TAG 为 gyro：写入 `s_pair_state.gyro`；
- TAG 为 accel：写入 `s_pair_state.accel`；
- 另一类数据还没有到达时，暂时不产生完整 sample；
- 两类数据都有效时，输出一个 `imu_sample_t`；
- 输出后清除两个 valid 标志。

### 7.2 配对异常

如果同一种数据在另一种数据到来前再次出现，代码认为前一份数据被覆盖，增加：

```c
s_pair_error_count++;
```

如果 TAG 不是 `0x01` 或 `0x02`，增加：

```c
s_tag_error_count++;
```

这两个计数器可以帮助判断 FIFO TAG 解析是否正确。

### 7.3 little-endian 数据解析

LSM6DSR 输出的 16 位轴数据按低字节在前、高字节在后，代码使用：

```c
static int16_t read_le_i16(const uint8_t *p)
{
    return (int16_t)((uint16_t)p[0] |
                     ((uint16_t)p[1] << 8));
}
```

最终 sample 字段顺序为：

```text
Ax, Ay, Az, Gx, Gy, Gz
```

---

## 8. TCP 帧协议

### 8.1 固定帧结构

当前固件发送的结构体为：

```c
typedef struct __attribute__((packed)) {
    uint16_t magic;
    uint16_t count;
    uint32_t seq_num;
    uint64_t timestamp_us;
    int16_t samples[IMU_SAMPLES_PER_FRAME][6];
} tcp_frame_t;
```

其中：

```c
#define IMU_SAMPLES_PER_FRAME 100u
#define TCP_FRAME_MAGIC       0x55AAu
```

编译期检查：

```c
_Static_assert(sizeof(tcp_frame_t) == 1216,
               "tcp_frame_t must be 1216 bytes");
```

### 8.2 字节布局

| 偏移 | 长度 | 类型 | 含义 |
|---:|---:|---|---|
| 0 | 2 | `uint16_t` | magic，值为 `0x55AA` |
| 2 | 2 | `uint16_t` | count，值为 100 |
| 4 | 4 | `uint32_t` | 序号 `seq_num` |
| 8 | 8 | `uint64_t` | 时间戳 `timestamp_us` |
| 16 | 1200 | `int16_t[100][6]` | 100 组 Ax/Ay/Az/Gx/Gy/Gz |
| 合计 | 1216 | | 一帧完整长度 |

ESP32-S3 使用小端 CPU，因此上位机必须按照 little-endian 解析：

```text
uint16：低字节在前
uint32：低字节在前
uint64：低字节在前
int16：低字节在前
```

### 8.3 序号意义

`seq_num` 在开始组装一个新 frame 时递增：

```c
s_building_frame.seq_num = s_next_sequence++;
```

序号可以帮助定位：

- RingBuffer 满导致的 frame 丢弃；
- TCP 断开前尚未成功发送的 frame；
- ESP32 重启或重新建立连接；
- 上位机错误解析或丢失完整 frame。

TCP 本身是可靠字节流，服务端不能把 TCP `recv()` 的一次返回当作一帧。正确的接收方必须：

1. 累积字节；
2. 找到 magic；
3. 等待至少 16 字节头部；
4. 根据固定长度等待 1216 字节；
5. 再解析一帧；
6. 处理粘包和半包。

### 8.4 时间戳含义

当前 frame 的 `timestamp_us` 来自收到 INT1 通知后调用的：

```c
uint64_t watermark_timestamp_us = esp_timer_get_time();
```

这表示本次 FIFO watermark 被任务处理时的 ESP32 单调时间戳，不是每个样本的独立采样时刻。一个 frame 内的 100 个 sample 使用同一个 watermark 时间戳。

---

## 9. RingBuffer 和 TCP 发送

### 9.1 发送流程

`tcp_send_task()` 的逻辑为：

```text
等待 WIFI_CONNECTED_BIT
        │
        ▼
创建 TCP socket
        │
        ├─ 失败：累计 tcp_error，延时 1 秒重试
        └─ 成功
             │
             ▼
       循环取 RingBuffer item
             │
             ├─ item_size != 1216 → 发送失败流程
             ├─ send_all 成功 → frame_sent_count++
             └─ send_all 失败 → tcp_error/tcp_drop++，断开重连
```

### 9.2 socket 配置

连接函数执行：

- `socket(AF_INET, SOCK_STREAM, IPPROTO_IP)`；
- `inet_pton()` 检查服务端 IP；
- `connect()` 建立 TCP 连接；
- `TCP_NODELAY` 禁用 Nagle 延迟；
- `SO_SNDTIMEO` 设置 2 秒发送超时。

当前 TCP 目标配置为：

```c
#define TCP_SERVER_IP    "192.168.18.70"
#define TCP_SERVER_PORT  8080
```

实际使用时必须确保：

- 电脑的局域网 IP 与 `TCP_SERVER_IP` 一致；
- 上位机程序监听的端口与 `TCP_SERVER_PORT` 一致；
- Windows 防火墙允许该端口的专用网络通信；
- 8080 没有被其他 HTTP/Web 服务占用。

### 9.3 `send_all()` 的必要性

`send()` 不保证一次就发送完全部缓冲区。因此代码使用循环：

```c
while (sent < length) {
    int ret = send(sock, buffer + sent, length - sent, 0);
    if (ret <= 0) {
        return false;
    }
    sent += (size_t)ret;
}
```

只有完整 1216 字节都发送成功，才增加 `s_frame_sent_count`。

### 9.4 RingBuffer item 的归还

无论发送成功还是失败，都必须执行：

```c
vRingbufferReturnItem(s_frame_ringbuffer, item);
```

如果忘记归还，RingBuffer 的可用空间会逐渐耗尽，最终导致持续 `rb_drop`。

---

## 10. Wi-Fi 初始化和事件流程

### 10.1 初始化顺序

`wifi_sta_init()` 的主要顺序为：

1. 初始化 NVS；
2. 如果发现 NVS 空间不足或版本不兼容，擦除后重新初始化；
3. 创建 Wi-Fi EventGroup；
4. 初始化网络接口；
5. 创建默认事件循环；
6. 创建默认 STA netif；
7. 初始化 Wi-Fi 驱动；
8. 注册 Wi-Fi 事件和获取 IP 事件；
9. 设置 SSID 和密码；
10. 设置 STA 模式；
11. 启动 Wi-Fi；
12. 关闭 Wi-Fi 省电模式。

```c
ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_NONE));
```

关闭省电模式有利于降低高吞吐 TCP 数据链路的延迟和抖动，但会增加功耗。

### 10.2 事件状态

| 事件 | 处理 |
|---|---|
| `WIFI_EVENT_STA_START` | 调用 `esp_wifi_connect()` |
| `WIFI_EVENT_STA_DISCONNECTED` | 清除连接位并重新连接 |
| `IP_EVENT_STA_GOT_IP` | 设置 `WIFI_CONNECTED_BIT`，允许 TCP 任务工作 |

TCP 任务不会在没有 IPv4 地址时反复尝试连接，而是先阻塞等待：

```c
xEventGroupWaitBits(s_wifi_event_group,
                    WIFI_CONNECTED_BIT,
                    pdFALSE,
                    pdTRUE,
                    portMAX_DELAY);
```

### 10.3 Wi-Fi 凭据安全

真实凭据位于：

```text
main/wifi_private.h
```

该文件已经加入 `.gitignore`：

```text
/main/wifi_private.h
```

模板为：

```c
#define WIFI_SSID     "YOUR_2_4GHZ_SSID"
#define WIFI_PASSWORD "YOUR_WIFI_PASSWORD"
```

不要把真实密码写入：

- 项目总结文档；
- 代码截图；
- Git 仓库；
- 串口日志；
- 公开压缩包。

---

## 11. app_main 启动顺序

当前 `app_main()` 的启动顺序是项目稳定性的关键：

```text
1. 初始化 SPI
2. 延时等待传感器稳定
3. 读取 WHO_AM_I，最多重试 10 次
4. 传感器软件复位和基础配置
5. 创建 32 KiB RingBuffer
6. 创建 imu_irq_test 任务
7. 安装 GPIO15 INT1 ISR
8. 预分配 FIFO DMA 缓冲区
9. 开启 LSM6DSR 高速 FIFO Continuous 模式
10. 初始化 Wi-Fi
11. 创建 tcp_send 任务
```

特别注意第 6～9 步：

```text
任务和 ISR 必须先准备好
然后才开启 FIFO Continuous 模式
```

如果先开启 FIFO，再创建任务或安装 GPIO ISR，第一次 watermark 上升沿可能在软件还未准备好时到达，造成 FIFO 启动竞态和数据滞留。

---

## 12. 统计日志说明

`imu_irq_test_task()` 每秒打印两行统计。

### 12.1 采集统计

```text
INT1 GPIO15: irq=63/s fifo_peak=200 sample=7179/s frame_new=72/s frame_sent=0/s
```

| 字段 | 含义 |
|---|---|
| `irq` | 最近一秒 GPIO15 中断次数 |
| `fifo_peak` | 最近一次 FIFO drain 中观察到的最大 unread words |
| `sample` | 最近一秒完成的六轴 sample 数 |
| `frame_new` | 最近一秒组装完成的 frame 数 |
| `frame_sent` | 最近一秒完整发送成功的 frame 数 |

### 12.2 FIFO 和网络统计

```text
fifo: status_err=0 ovr=0 dma_err=0 tag_err=0 pair_err=0; net: rb_drop=116 tcp_err=0 tcp_drop=0; A=[66,-69,-2055] G=[-19,31,-7]
```

| 字段 | 含义 |
|---|---|
| `status_err` | FIFO 状态寄存器读取失败或 reset 失败累计次数 |
| `ovr` | FIFO overrun/full 累计次数 |
| `dma_err` | FIFO DMA 读取失败次数 |
| `tag_err` | 发现未知 FIFO TAG 的次数 |
| `pair_err` | gyro/accel 配对覆盖异常次数 |
| `rb_drop` | RingBuffer 满，导致新 frame 无法入队的累计次数 |
| `tcp_err` | TCP 连接或发送失败累计次数 |
| `tcp_drop` | TCP 发送失败时丢弃的当前 frame 累计次数 |
| `A` | 最近一个完整 sample 的加速度原始值 |
| `G` | 最近一个完整 sample 的陀螺仪原始值 |

这些计数器从本次上电开始累计，不会每秒清零。因此排查问题时应记录重启后的第一组统计。

---

## 13. 已遇到的问题和解决过程

### 13.1 WHO_AM_I 读到 `0x00`

#### 现象

最初读取 LSM6DSR ID 时得到：

```text
WHO_AM_I = 0x00
```

#### 可能原因

- SPI Mode 不匹配；
- CS、MISO、MOSI、SCLK 接线错误；
- 传感器没有正确供电或没有共地；
- SPI 读取偏移错误；
- I3C/SPI 接口状态不符合预期；
- 传感器尚未稳定就立即访问。

#### 当前代码采取的解决措施

1. SPI 设置为 Mode 3；
2. 明确使用 GPIO4/5/6/7；
3. 读取寄存器值使用 `rx_data[1]`；
4. 初始化后延时 100 ms；
5. WHO_AM_I 最多重试 10 次；
6. 基础配置中关闭 I3C；
7. 通过日志打印每次尝试值和预期值。

成功标准：

```text
读取值 = 0x6B
ESP_OK
```

### 13.2 FIFO 高速模式下第一次中断丢失

#### 原因

如果应用先启动 FIFO Continuous 模式，再安装 GPIO ISR 或创建 FIFO 消费任务，FIFO watermark 可能在软件未准备好之前触发。

#### 解决

当前顺序固定为：

```text
RingBuffer
→ FIFO 任务
→ GPIO15 ISR
→ FIFO DMA 缓冲区
→ FIFO Continuous
```

### 13.3 FIFO 读取速度和 DMA 分配

#### 问题

如果第一次进入中断任务时才申请 DMA 内存，会把内存分配和实时采集路径混在一起，增加第一次处理抖动和失败风险。

#### 解决

在开启高速 FIFO 前执行：

```c
ESP_ERROR_CHECK(lsm6dsr_fifo_dma_init());
```

DMA 缓冲区一次分配、重复使用。

### 13.4 FIFO 是否只读固定 200 words

#### 问题

watermark 是 200 words，但从中断触发到任务真正运行之间，FIFO 可能已经积累了更多 word。如果读取一次 200 words 就退出，剩余数据会继续堆积。

#### 解决

每次 DMA 读取后重新读取 FIFO 状态，直到：

```text
unread_words == 0
```

并且单次 DMA 读取不超过 256 words。

### 13.5 FIFO full / overrun

#### 问题

只处理 `overrun` 而忽略 `full` 会漏掉一种 FIFO 已达到危险状态的情况。

#### 当前处理

```c
if (status.overrun || status.full) {
    s_fifo_overrun_count++;
    need_reset = true;
}
```

恢复时清空配对状态并 reset FIFO。

### 13.6 RingBuffer drop 增长

#### 现象

当 Wi-Fi 尚未获取 IP、TCP 服务端未启动或 TCP 不断重连时，采集任务仍然继续运行，因此日志中的 `rb_drop` 增长：

```text
rb_drop=116
rb_drop=187
rb_drop=259
```

#### 原因

当前架构选择了：

```text
采集任务不等待网络
```

这样可以避免网络阻塞导致 FIFO 溢出，但代价是网络长期不可用时 RingBuffer 最终装满，新 frame 被丢弃。

#### 取舍

| 方案 | 优点 | 缺点 |
|---|---|---|
| 采集等待网络 | 几乎不主动丢 frame | 可能造成 FIFO 堵塞和传感器 overrun |
| 采集和网络解耦，RingBuffer 缓冲 | 采集实时性好 | 网络持续变慢时 RingBuffer 会满 |
| 降低采样率 | 网络压力低 | 改变采样需求 |
| 扩大 RingBuffer | 可吸收更长网络抖动 | 占用更多 RAM，不能解决长期带宽不足 |

当前实现选择第二种方案。

### 13.7 `errno=104` / TCP send failed

#### 现象

日志类似：

```text
TCP connected: 192.168.18.70:8080
TCP send failed, errno=104
```

#### 含义

在 ESP-IDF/LwIP 中，`errno=104` 对应 `ECONNRESET`，表示 TCP 连接被对端重置。也就是说：

- Wi-Fi 已经连上；
- ESP32 已经成功建立 TCP 连接；
- 对端在发送过程中关闭或重置了连接。

#### 排查结果

电脑上的 8080 端口曾被以下服务占用：

```text
NIApplicationWebServer
```

该程序不是当前 IMU 原始 TCP 协议接收服务，可能在接收少量数据后关闭连接，导致 ESP32 侧持续重连和 `errno=104`。

#### 处理原则

1. 确保 8080 只由正确的原始 TCP 接收程序监听；
2. 上位机必须持续 `recv()`，不能只接收一次；
3. 上位机必须按 TCP 字节流处理半包和粘包；
4. 上位机不能把 1216 字节二进制数据当作 HTTP 请求；
5. 如果端口被 NI 或其他 Web 服务占用，需要停止该服务或更换端口并重新烧录固件。

### 13.8 Windows `WinError 10013`

#### 现象

Python 服务端尝试监听 8080 时出现：

```text
[WinError 10013] 以一种访问权限不允许的方式做了一个访问套接字的尝试
```

#### 实际原因

检查发现 8080 已被 `ApplicationWebServer.exe` 监听，PID 为 10900，所属服务为：

```text
NIApplicationWebServer
```

因此不是 Python 代码语法问题，而是端口已经被其他服务占用或该服务使用了独占监听权限。

#### 解决方法

需要以管理员权限停止该服务：

```powershell
Stop-Service -Name NIApplicationWebServer -Force
```

然后确认：

```powershell
Get-NetTCPConnection -LocalPort 8080 -State Listen
```

没有输出后，正确的 TCP 接收程序才能监听 8080。

### 13.9 采样率约为 7.1 ksample/s

配置目标为 6667 Hz，但日志曾显示：

```text
sample=7088/s
sample=7134/s
sample=7185/s
```

当前 FIFO、DMA、TAG 和配对错误都为 0，说明数据通道基本正常。这个差异需要后续单独测量确认，可能受到：

- FIFO BDR 实际寄存器含义；
- `sample` 计数窗口与任务调度；
- 日志时间基准和 Tick 采样；
- FIFO TAG 配对方式；
- 传感器 ODR 配置值解释。

在 `fifo status_err=0`、`ovr=0`、`dma_err=0`、`tag_err=0`、`pair_err=0` 的情况下，它不是当前 TCP `ECONNRESET` 的直接原因。应在网络链路稳定后，用更长时间窗和独立时间基准测量真实采样率。

---

## 14. 编译、烧录和串口监控

### 14.1 准备环境

本项目配置使用 ESP-IDF 5.5.5。Windows 环境中需要先加载 ESP-IDF 环境，例如：

```powershell
. D:\esp\v5.5.5\export.ps1
```

然后确认目标：

```powershell
idf.py set-target esp32s3
```

### 14.2 配置本地 Wi-Fi

如果 `main/wifi_private.h` 不存在：

```powershell
Copy-Item .\main\wifi_private.h.example .\main\wifi_private.h
```

然后只在本地文件中填写：

```c
#define WIFI_SSID     "你的 2.4GHz Wi-Fi 名称"
#define WIFI_PASSWORD "你的 Wi-Fi 密码"
```

不要修改模板来保存真实密码，也不要把 `wifi_private.h` 上传到仓库。

### 14.3 编译

```powershell
idf.py build
```

### 14.4 烧录

当前 VS Code 配置中的串口为 `COM7`，但实际使用时以设备管理器显示为准：

```powershell
idf.py -p COM7 flash
```

### 14.5 监控串口

```powershell
idf.py -p COM7 monitor
```

或者合并操作：

```powershell
idf.py -p COM7 flash monitor
```

### 14.6 推荐测试顺序

1. 关闭会占用 TCP 8080 的旧服务；
2. 启动正确的 TCP 接收程序；
3. 启动串口监控；
4. 重启 ESP32；
5. 先观察 `WHO_AM_I`；
6. 再观察 FIFO 统计；
7. 再观察 Wi-Fi 获取 IP；
8. 最后观察 TCP 连接和 `frame_sent/s`；
9. 若出现异常，按错误层次逐级定位，不要一开始同时修改 SPI、FIFO 和网络。

---

## 15. 分层故障排查方法

### 15.1 传感器层

检查：

```text
WHO_AM_I 是否为 0x6B
```

如果不是，优先检查：

- 电源和 GND；
- CS；
- SPI Mode 3；
- GPIO4/5/6/7；
- MISO 是否真正连接到 GPIO6；
- I3C/SPI 模式。

### 15.2 FIFO 层

正常期望：

```text
status_err=0
ovr=0
dma_err=0
tag_err=0
pair_err=0
fifo_peak 接近 200
```

如果 `ovr` 增加：

- 检查 INT1 是否连接 GPIO15；
- 检查中断极性；
- 检查 SPI DMA 读取速度；
- 检查任务是否被高优先级任务长期阻塞；
- 检查 FIFO 配置和采样率。

### 15.3 组帧层

正常期望：

```text
sample ≈ frame_new × 100
```

如果 `sample` 增加但 `frame_new` 不增加，检查：

- `IMU_SAMPLES_PER_FRAME`；
- RingBuffer 是否已经满；
- `xRingbufferSend()` 是否失败；
- `s_ringbuffer_drop_count`。

### 15.4 Wi-Fi 层

先确认日志出现：

```text
Wi-Fi connected and IPv4 address acquired
```

如果没有：

- 检查 SSID 和密码；
- 确认使用 2.4 GHz 网络；
- 检查 NVS 初始化；
- 检查 AP 是否允许新客户端；
- 检查 ESP32 与电脑是否在同一局域网。

### 15.5 TCP 层

正常期望：

```text
TCP connected: 电脑IP:8080
frame_sent/s > 0
```

如果 `TCP connected` 成功但 `frame_sent/s=0`：

- 服务端可能没有正确接收；
- 8080 可能是 HTTP 服务；
- 服务端可能连接后立即关闭；
- Windows 防火墙可能丢弃或阻止通信；
- RingBuffer 中可能没有可取的 frame。

如果出现 `errno=104`：

```text
优先检查对端是否主动关闭 TCP 连接
```

如果 `rb_drop` 继续增长：

```text
优先检查网络发送速度是否低于采集速度
```

---

## 16. 代码 API 总表

### 16.1 LSM6DSR 驱动公开 API

| API | 作用 |
|---|---|
| `esp_lsm6dsr_spi_init()` | 初始化 SPI2 总线并添加 LSM6DSR 设备 |
| `lsm6dsr_write_reg()` | 写单个寄存器 |
| `lsm6dsr_read_reg()` | 读单个寄存器 |
| `lsm6dsr_read_gyro_accel_dma()` | 读取普通输出寄存器中的 gyro+accel 数据 |
| `lsm6dsr_fifo_config_low_rate_irq_test()` | 低速 FIFO/中断测试配置，主要用于早期验证 |
| `lsm6dsr_fifo_get_status()` | 读取 FIFO unread/watermark/overrun/full 状态 |
| `lsm6dsr_fifo_reset()` | Bypass 后重新设置 FIFO 连续模式 |
| `lsm6dsr_fifo_dma_init()` | 预分配 FIFO DMA 缓冲区 |
| `lsm6dsr_fifo_read_words_dma()` | 通过 DMA 读取指定数量的 FIFO words |
| `lsm6dsr_fifo_config_high_rate()` | 配置 6667 Hz 高速采样、watermark 和 INT1 |

### 16.2 `main.c` 内部函数

| 函数 | 作用 |
|---|---|
| `tcp_connect_server()` | 创建并配置 TCP socket，连接电脑服务端 |
| `send_all()` | 循环发送，保证完整 frame 写入 TCP |
| `wifi_connect()` | 发起 Wi-Fi 连接请求 |
| `wifi_event_handler()` | 处理 Wi-Fi 状态和获取 IP 事件 |
| `wifi_sta_init()` | 初始化 Wi-Fi STA 和事件组 |
| `append_sample_to_frame()` | 把 sample 加入当前 frame，满 100 个后放入 RingBuffer |
| `read_le_i16()` | 解析 little-endian 的有符号 16 位值 |
| `consume_one_fifo_word()` | 解析 FIFO TAG 并完成 gyro/accel 配对 |
| `imu_int1_isr()` | GPIO15 中断服务函数 |
| `imu_int1_gpio_init()` | 配置 GPIO15 和 ISR |
| `imu_irq_test_task()` | FIFO 消费、解析、组帧和采集统计 |
| `tcp_send_task()` | RingBuffer 消费、TCP 发送和自动重连 |
| `app_main()` | 完成全局启动流程 |

---

## 17. 后续可改进方向

### 17.1 采样率校准

在网络稳定后，建议：

1. 用 `esp_timer_get_time()` 统计至少 30 秒；
2. 记录完成 sample 数；
3. 计算长时间平均采样率；
4. 对比传感器数据手册中实际 ODR 对应的寄存器编码；
5. 确认 `CTRL1_XL=0xA4` 和 `CTRL2_G=0xAC` 的实际含义。

### 17.2 RingBuffer 背压策略

当前策略是 RingBuffer 满时丢弃新 frame。可选改进：

- 增大 RingBuffer；
- 在网络断开时降低采样率；
- 只保留最新 frame；
- 加入丢帧策略和明确的丢帧序号；
- 将原始数据写入本地存储后再发送。

### 17.3 时间戳改进

当前每 100 个 sample 共用 watermark 时间戳。若需要精确时域分析，可以：

- 为每个 sample 保存采样序号；
- 使用固定 ODR 计算每个 sample 的理论时间；
- 让上位机根据 frame 起始时间和采样间隔重建时间轴；
- 如果芯片支持更细粒度 FIFO timestamp，再读取并发送传感器时间戳。

### 17.4 网络协议增强

当前固定帧协议已经适合高速传输，但后续可以加入：

- 协议版本号；
- payload 长度；
- CRC32；
- 设备编号；
- 采样率字段；
- 丢帧累计计数；
- 连接握手和服务端能力协商。

如果增加字段，必须同步修改：

- `tcp_frame_t`；
- `_Static_assert`；
- 上位机解析程序；
- 协议文档；
- 可能的版本兼容逻辑。

### 17.5 INT2 的使用

当前 INT2 只定义了 GPIO16：

```c
#define PIN_NUM_INT2 16
```

尚未配置 INT2。后续如果需要把另一类事件映射到 INT2，应新增：

- GPIO16 输入配置；
- INT2 ISR；
- 任务通知或事件位；
- LSM6DSR 对应中断路由寄存器配置；
- 独立统计和故障处理。

---

## 18. 关键设计结论

1. **ISR 只通知任务，不直接读取 SPI。**
2. **FIFO 启动前，任务、GPIO ISR 和 DMA 缓冲区必须准备完成。**
3. **FIFO 必须通过重新读取 status 持续 drain，而不是只读固定 watermark 数量。**
4. **FIFO `overrun` 和 `full` 都应触发恢复。**
5. **RingBuffer 用于隔离高速采集和不稳定网络。**
6. **TCP 是字节流，不能假设一次 send 对应一次 recv。**
7. **`send_all()` 用于保证固件一帧完整写入 socket。**
8. **`seq_num` 是判断应用层丢帧的主要依据。**
9. **`errno=104` 表示对端重置连接，优先检查服务端。**
10. **Wi-Fi 真实凭据必须与代码和文档分离。**
11. **网络服务器不是 ESP32 固件的一部分，必须确认监听的是原始 TCP 服务，而不是 HTTP/Web 服务。**

---

# 附录 A：完整固件源代码

以下代码内容来自当前工程。为了保护本地网络安全，真实 `main/wifi_private.h` 不在文档中展开，只展开脱敏模板。

## A.1 根目录 CMakeLists.txt

文件：$relativePath  

`$language
# The following five lines of boilerplate have to be in your project's
# CMakeLists in this exact order for cmake to work correctly
cmake_minimum_required(VERSION 3.16)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(ESP32S3_IMU)
` 

## A.2 main/CMakeLists.txt

文件：$relativePath  

`$language
idf_component_register(SRCS "main.c"
                    INCLUDE_DIRS "."
                    REQUIRES lsm6dsr driver esp_timer esp_event esp_netif esp_wifi nvs_flash lwip)
` 

## A.3 LSM6DSR 组件 CMakeLists.txt

文件：$relativePath  

`$language
idf_component_register(SRCS "lsm6dsr.c"
                    INCLUDE_DIRS "include"
                    REQUIRES driver)
` 

## A.4 LSM6DSR 公共头文件

文件：$relativePath  

`$language
#ifndef LSM6DSR_H
#define LSM6DSR_H

#include <stdint.h>
#include "esp_err.h"
#include <stdbool.h>
#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

// 引脚定义（如果需要切换引脚，在这里修改或作为参数传入）
#define PIN_NUM_CLK   4
#define PIN_NUM_MOSI  5
#define PIN_NUM_MISO  6
#define PIN_NUM_CS    7
#define PIN_NUM_INT1  15
#define PIN_NUM_INT2  16
#define SPI_CLOCK_HZ  (10 * 1000 * 1000)

// 寄存器定义
#define WHO_AM_I      0x0F
#define CTRL1_XL      0x10
#define CTRL2_G       0x11
#define CTRL3_C       0x12
#define CTRL9_XL      0x18
#define OUTX_L_G      0x22
#define OUTX_L_A      0x28
#define LSM6DSR_ID    0x6B
#define FIFO_CTRL1             0x07
#define FIFO_CTRL2             0x08
#define FIFO_CTRL3             0x09
#define FIFO_CTRL4             0x0A
#define INT1_CTRL              0x0D
#define FIFO_STATUS1           0x3A
#define FIFO_STATUS2           0x3B
#define FIFO_DATA_OUT_TAG      0x78

#define LSM6DSR_FIFO_WORD_SIZE 7u
#define LSM6DSR_FIFO_MAX_WORDS_PER_READ 256u

#define LSM6DSR_FIFO_WATERMARK_WORDS 200u
#define LSM6DSR_FIFO_RAW_MAX_BYTES \
    (LSM6DSR_FIFO_MAX_WORDS_PER_READ * LSM6DSR_FIFO_WORD_SIZE)

typedef struct {
    uint16_t unread_words;
    bool watermark;
    bool overrun;
    bool full;
} lsm6dsr_fifo_status_t;

// 核心 API 函数声明
void esp_lsm6dsr_spi_init(void);
esp_err_t lsm6dsr_write_reg(uint8_t reg, uint8_t data);
esp_err_t lsm6dsr_read_reg(uint8_t reg, uint8_t *value);
esp_err_t lsm6dsr_read_gyro_accel_dma(int16_t *gx, int16_t *gy, int16_t *gz,
                                       int16_t *ax, int16_t *ay, int16_t *az);
esp_err_t lsm6dsr_fifo_config_low_rate_irq_test(void);
esp_err_t lsm6dsr_fifo_get_status(lsm6dsr_fifo_status_t *status);
esp_err_t lsm6dsr_fifo_reset(void);
esp_err_t lsm6dsr_fifo_dma_init(void);
esp_err_t lsm6dsr_fifo_read_words_dma(uint8_t *raw_out, size_t word_count);
esp_err_t lsm6dsr_fifo_config_high_rate(void);

#ifdef __cplusplus
}
#endif

#endif // LSM6DSR_H
` 

## A.5 LSM6DSR 驱动实现

文件：$relativePath  

`$language
#include "lsm6dsr.h"
#include <stdio.h>
#include <string.h>
#include "esp_err.h"
#include "esp_log.h"
#include "esp_heap_caps.h"
#include "driver/spi_master.h"
#include "driver/gpio.h"

static const char *TAG = "LSM6DSR_SPI";
static spi_device_handle_t SPI_Handle;
#define FIFO_DMA_TRANSFER_BYTES \
    (1u + LSM6DSR_FIFO_MAX_WORDS_PER_READ * LSM6DSR_FIFO_WORD_SIZE)

static uint8_t *s_fifo_tx_dma = NULL;
static uint8_t *s_fifo_rx_dma = NULL;

void esp_lsm6dsr_spi_init(void)
{
    esp_err_t ret;

    spi_bus_config_t buscfg = {
        .miso_io_num = PIN_NUM_MISO,
        .mosi_io_num = PIN_NUM_MOSI,
        .sclk_io_num = PIN_NUM_CLK,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1,
        .max_transfer_sz = FIFO_DMA_TRANSFER_BYTES,
    };
    ret = spi_bus_initialize(SPI2_HOST, &buscfg, SPI_DMA_CH_AUTO);
    ESP_ERROR_CHECK(ret);

    spi_device_interface_config_t devcfg = {
        .clock_speed_hz = SPI_CLOCK_HZ,
        .mode = 3,
        .spics_io_num = PIN_NUM_CS,
        .queue_size = 7, 
    };
    ret = spi_bus_add_device(SPI2_HOST, &devcfg, &SPI_Handle);
    ESP_ERROR_CHECK(ret);

    ESP_LOGI(TAG,
             "SPI initialized: SCLK=GPIO%d MOSI/SDI=GPIO%d MISO/SDO=GPIO%d CS=GPIO%d, mode=3, %d Hz",
             PIN_NUM_CLK, PIN_NUM_MOSI, PIN_NUM_MISO, PIN_NUM_CS, SPI_CLOCK_HZ);
}

esp_err_t lsm6dsr_write_reg(uint8_t reg, uint8_t data)
{
    uint8_t tx_data[2] = {reg & 0x7F, data};
    spi_transaction_t t = {
        .length = sizeof(tx_data) * 8,
        .tx_buffer = tx_data,
    };
    return spi_device_polling_transmit(SPI_Handle, &t);
}

esp_err_t lsm6dsr_read_reg(uint8_t reg, uint8_t *value)
{
    uint8_t tx_data[2] = {reg | 0x80, 0x00};
    uint8_t rx_data[2] = {0};
    spi_transaction_t t = {
        .length = sizeof(tx_data) * 8,
        .tx_buffer = tx_data,
        .rx_buffer = rx_data,
    };

    esp_err_t ret = spi_device_polling_transmit(SPI_Handle, &t);
    if (ret == ESP_OK) {
        *value = rx_data[1];
    }
    return ret;
}

esp_err_t lsm6dsr_read_gyro_accel_dma(int16_t *gx, int16_t *gy, int16_t *gz,
                                      int16_t *ax, int16_t *ay, int16_t *az) 
{
    size_t length = 1 + 12;

    uint8_t *tx_buf = heap_caps_malloc(length, MALLOC_CAP_DMA);
    uint8_t *rx_buf = heap_caps_malloc(length, MALLOC_CAP_DMA);

    if (!tx_buf || !rx_buf) {
        if (tx_buf) free(tx_buf);
        if (rx_buf) free(rx_buf);
        return ESP_ERR_NO_MEM;
    }

    memset(tx_buf, 0x00, length);
    tx_buf[0] = OUTX_L_G | 0x80;

    spi_transaction_t t = {
        .length = length * 8,
        .tx_buffer = tx_buf,
        .rx_buffer = rx_buf,
    };

    esp_err_t ret = spi_device_transmit(SPI_Handle, &t);

    if (ret == ESP_OK) {
        *gx = (int16_t)(rx_buf[2] << 8 | rx_buf[1]);
        *gy = (int16_t)(rx_buf[4] << 8 | rx_buf[3]);
        *gz = (int16_t)(rx_buf[6] << 8 | rx_buf[5]);

        *ax = (int16_t)(rx_buf[8] << 8 | rx_buf[7]);
        *ay = (int16_t)(rx_buf[10] << 8 | rx_buf[9]);
        *az = (int16_t)(rx_buf[12] << 8 | rx_buf[11]);
    }

    free(tx_buf);
    free(rx_buf);
    return ret;
}

esp_err_t lsm6dsr_fifo_reset(void)
{
    esp_err_t ret = lsm6dsr_write_reg(FIFO_CTRL4, 0x00);
    if(ret != ESP_OK)
        return ret;
    return lsm6dsr_write_reg(FIFO_CTRL4, 0x06);
}

esp_err_t lsm6dsr_fifo_config_low_rate_irq_test(void)
{
    esp_err_t ret;

    ret = lsm6dsr_write_reg(FIFO_CTRL4, 0x00);
    if(ret != ESP_OK) return ret;

    ret = lsm6dsr_write_reg(FIFO_CTRL1, 0x02);
    if(ret != ESP_OK) return ret;

    ret = lsm6dsr_write_reg(FIFO_CTRL2, 0x00);
    if(ret != ESP_OK) return ret;

    ret = lsm6dsr_write_reg(FIFO_CTRL3, 0x44);
    if(ret != ESP_OK) return ret;

    ret = lsm6dsr_write_reg(INT1_CTRL, 0x08);
    if(ret != ESP_OK) return ret;

    return lsm6dsr_write_reg(FIFO_CTRL4, 0x06);
}

esp_err_t lsm6dsr_fifo_get_status(lsm6dsr_fifo_status_t *status)
{
    if (status == NULL) {
        return ESP_ERR_INVALID_ARG;
    }

    // 当前单寄存器读函数以 rx_data[1] 作为数据，因此连续读 2 个寄存器时，
    // rx_data[1] 是 FIFO_STATUS1，rx_data[2] 是 FIFO_STATUS2。
    uint8_t tx_data[3] = { FIFO_STATUS1 | 0x80, 0x00, 0x00 };
    uint8_t rx_data[3] = { 0 };
    spi_transaction_t t = {
        .length = sizeof(tx_data) * 8,
        .tx_buffer = tx_data,
        .rx_buffer = rx_data,
    };

    esp_err_t ret = spi_device_polling_transmit(SPI_Handle, &t);
    if (ret != ESP_OK) {
        return ret;
    }

    uint8_t s1 = rx_data[1];
    uint8_t s2 = rx_data[2];
    status->unread_words = (uint16_t)s1 | ((uint16_t)(s2 & 0x03) << 8);
    status->watermark = (s2 & BIT(7)) != 0;
    status->overrun = (s2 & BIT(6)) != 0;
    status->full = (s2 & BIT(5)) != 0;
    return ESP_OK;
}

esp_err_t lsm6dsr_fifo_dma_init(void)
{
    if(s_fifo_tx_dma != NULL && s_fifo_rx_dma != NULL)
        return ESP_OK;

    s_fifo_tx_dma = heap_caps_malloc(FIFO_DMA_TRANSFER_BYTES, MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL);
    s_fifo_rx_dma = heap_caps_malloc(FIFO_DMA_TRANSFER_BYTES, MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL);

    if(s_fifo_tx_dma == NULL || s_fifo_rx_dma == NULL)
    {
        free(s_fifo_tx_dma);
        free(s_fifo_rx_dma);
        s_fifo_tx_dma = NULL;
        s_fifo_rx_dma = NULL;
        return ESP_ERR_NO_MEM;
    }
    memset(s_fifo_tx_dma, 0, FIFO_DMA_TRANSFER_BYTES);
    memset(s_fifo_rx_dma, 0, FIFO_DMA_TRANSFER_BYTES);
    return ESP_OK;
}

esp_err_t lsm6dsr_fifo_read_words_dma(uint8_t *raw_out, size_t word_count)
{
    if (raw_out == NULL || word_count == 0 || word_count > LSM6DSR_FIFO_MAX_WORDS_PER_READ) 
        return ESP_ERR_INVALID_ARG;

    esp_err_t ret = lsm6dsr_fifo_dma_init();
    if (ret != ESP_OK) {
        return ret;
    }

    const size_t fifo_bytes = word_count * LSM6DSR_FIFO_WORD_SIZE;
    const size_t transfer_bytes = 1u + fifo_bytes;

    memset(s_fifo_tx_dma, 0, transfer_bytes);
    s_fifo_tx_dma[0] = FIFO_DATA_OUT_TAG | 0x80;

    spi_transaction_t t = {
        .length = transfer_bytes * 8, // ESP-IDF SPI length 单位是 bit
        .tx_buffer = s_fifo_tx_dma,
        .rx_buffer = s_fifo_rx_dma,
    };

    ret = spi_device_transmit(SPI_Handle, &t);
    if (ret != ESP_OK) {
        return ret;
    }

    // 与当前 read_reg() 一致：RX[0] 是命令阶段，真正 FIFO 数据从 RX[1] 开始。
    memcpy(raw_out, &s_fifo_rx_dma[1], fifo_bytes);
    return ESP_OK;
}

esp_err_t lsm6dsr_fifo_config_high_rate(void)
{
    esp_err_t ret;

    ret = lsm6dsr_write_reg(FIFO_CTRL4, 0x00); // 先 Bypass
    if (ret != ESP_OK) return ret;

    ret = lsm6dsr_write_reg(CTRL1_XL, 0xA4); // 6667 Hz, ±16 g
    if (ret != ESP_OK) return ret;

    ret = lsm6dsr_write_reg(CTRL2_G, 0xAC);  // 6667 Hz, ±2000 dps
    if (ret != ESP_OK) return ret;

    ret = lsm6dsr_write_reg(FIFO_CTRL1,
                            LSM6DSR_FIFO_WATERMARK_WORDS & 0xFF);
    if (ret != ESP_OK) return ret;

    ret = lsm6dsr_write_reg(FIFO_CTRL2,
                            (LSM6DSR_FIFO_WATERMARK_WORDS >> 8) & 0x01);
    if (ret != ESP_OK) return ret;

    ret = lsm6dsr_write_reg(FIFO_CTRL3, 0xAA); // gyro/accel FIFO BDR 均 6667 Hz
    if (ret != ESP_OK) return ret;

    ret = lsm6dsr_write_reg(INT1_CTRL, 0x08);  // FIFO threshold 到 INT1
    if (ret != ESP_OK) return ret;

    return lsm6dsr_write_reg(FIFO_CTRL4, 0x06); // Continuous
}
` 

## A.6 ESP32-S3 应用主程序

文件：$relativePath  

`$language
#include <assert.h>
#include <errno.h>
#include <stdint.h>
#include <stdio.h>
#include <sys/time.h>
#include "esp_err.h"
#include "esp_intr_alloc.h"
#include "freertos/FreeRTOS.h"
#include "freertos/projdefs.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "driver/gpio.h"
#include "hal/gpio_types.h"
#include "lsm6dsr.h"
#include "portmacro.h"
#include <stdbool.h>
#include <string.h>
#include "esp_timer.h"
#include "freertos/ringbuf.h"
#include "freertos/event_groups.h"
#include "nvs_flash.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "esp_wifi.h"
#include "lwip/sockets.h"
#include "lwip/inet.h"
#include "lwip/tcp.h"
#include "wifi_private.h"

#define IMU_SAMPLES_PER_FRAME 100u
#define TCP_FRAME_MAGIC       0x55AAu
#define FRAME_RINGBUFFER_SIZE (32 * 1024)
#define WIFI_CONNECTED_BIT BIT0
#define TCP_SERVER_IP    "192.168.18.55"  // ⚠️ 请修改为你电脑上位机的实际局域网 IP
#define TCP_SERVER_PORT   8080             // ⚠️ 请修改为你上位机监听的实际端口

static volatile uint32_t s_frame_sent_count = 0;
static volatile uint32_t s_tcp_error_count = 0;
static volatile uint32_t s_tcp_frame_drop_count = 0;
static EventGroupHandle_t s_wifi_event_group;
static RingbufHandle_t s_frame_ringbuffer = NULL;
static uint32_t s_ringbuffer_drop_count = 0;

typedef struct __attribute__((packed)) {
    uint16_t magic;
    uint16_t count;
    uint32_t seq_num;
    uint64_t timestamp_us;
    int16_t samples[IMU_SAMPLES_PER_FRAME][6];
    // 每个 sample: Ax, Ay, Az, Gx, Gy, Gz
} tcp_frame_t;

_Static_assert(sizeof(tcp_frame_t) == 1216, "tcp_frame_t must be 1216 bytes");

static tcp_frame_t s_building_frame;
static size_t s_building_sample_index = 0;
static uint32_t s_next_sequence = 0;
static uint32_t s_frame_created_count = 0;


typedef struct {
    int16_t x;
    int16_t y;
    int16_t z;
} imu_xyz_t;

typedef struct {
    imu_xyz_t accel;
    imu_xyz_t gyro;
    bool accel_valid;
    bool gyro_valid;
} imu_pair_state_t;

typedef struct {
    int16_t ax;
    int16_t ay;
    int16_t az;
    int16_t gx;
    int16_t gy;
    int16_t gz;
} imu_sample_t;

static imu_pair_state_t s_pair_state;
static imu_sample_t s_last_sample;
static uint32_t s_tag_error_count = 0;
static uint32_t s_pair_error_count = 0;
static uint32_t s_complete_sample_count = 0;

static const char *TAG = "MAIN";

static TaskHandle_t s_imu_irq_task_handle = NULL;
static volatile uint32_t s_int1_irq_count = 0;

static uint32_t s_fifo_status_error_count = 0;
static uint32_t s_fifo_overrun_count = 0;
static uint16_t s_last_fifo_words = 0;

static uint32_t s_fifo_dma_error_count = 0;
static uint8_t s_raw_fifo[LSM6DSR_FIFO_RAW_MAX_BYTES];

static int tcp_connect_server(void)
{
    struct sockaddr_in dest_addr = { 0 };
    dest_addr.sin_family = AF_INET;
    dest_addr.sin_port = htons(TCP_SERVER_PORT);

    if (inet_pton(AF_INET, TCP_SERVER_IP, &dest_addr.sin_addr) != 1) {
        ESP_LOGE(TAG, "Invalid TCP server IPv4 address: %s", TCP_SERVER_IP);
        return -1;
    }

    int sock = socket(AF_INET, SOCK_STREAM, IPPROTO_IP);
    if (sock < 0) {
        ESP_LOGW(TAG, "TCP socket creation failed, errno=%d", errno);
        return -1;
    }

    if (connect(sock, (struct sockaddr *)&dest_addr, sizeof(dest_addr)) != 0) {
        ESP_LOGW(TAG, "TCP connect failed, errno=%d", errno);
        close(sock);
        return -1;
    }

    int one = 1;
    if (setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, &one, sizeof(one)) != 0) {
        ESP_LOGW(TAG, "Failed to enable TCP_NODELAY, errno=%d", errno);
        close(sock);
        return -1;
    }

    const struct timeval send_timeout = {
        .tv_sec = 2,
        .tv_usec = 0,
    };
    if (setsockopt(sock, SOL_SOCKET, SO_SNDTIMEO,
                   &send_timeout, sizeof(send_timeout)) != 0) {
        ESP_LOGW(TAG, "Failed to set TCP send timeout, errno=%d", errno);
        close(sock);
        return -1;
    }

    return sock;
}

static bool send_all(int sock, const uint8_t *buffer, size_t length)
{
    size_t sent = 0;

    while (sent < length) {
        int ret = send(sock, buffer + sent, length - sent, 0);
        if (ret <= 0) {
            ESP_LOGW(TAG, "TCP send failed, errno=%d", errno);
            return false;
        }
        sent += (size_t)ret;
    }

    return true;
}

static void wifi_connect(void)
{
    esp_err_t ret = esp_wifi_connect();
    if (ret != ESP_OK && ret != ESP_ERR_WIFI_STATE) {
        ESP_LOGW(TAG, "Wi-Fi connect request failed: %s", esp_err_to_name(ret));
    }
}

static void wifi_event_handler(void *arg,
                               esp_event_base_t event_base,
                               int32_t event_id,
                               void *event_data)
{
    (void)arg;
    (void)event_data;

    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        wifi_connect();
    } else if (event_base == WIFI_EVENT &&
               event_id == WIFI_EVENT_STA_DISCONNECTED) {
        xEventGroupClearBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
        ESP_LOGW(TAG, "Wi-Fi disconnected; reconnecting");
        wifi_connect();
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
        ESP_LOGI(TAG, "Wi-Fi connected and IPv4 address acquired");
    }
}

static void wifi_sta_init(void)
{
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES ||
        ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    s_wifi_event_group = xEventGroupCreate();
    assert(s_wifi_event_group != NULL);

    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    ESP_ERROR_CHECK(esp_event_handler_instance_register(
        WIFI_EVENT, ESP_EVENT_ANY_ID, &wifi_event_handler, NULL, NULL));
    ESP_ERROR_CHECK(esp_event_handler_instance_register(
        IP_EVENT, IP_EVENT_STA_GOT_IP, &wifi_event_handler, NULL, NULL));

    wifi_config_t wifi_config = { 0 };
    strncpy((char *)wifi_config.sta.ssid, WIFI_SSID,
            sizeof(wifi_config.sta.ssid) - 1);
    strncpy((char *)wifi_config.sta.password, WIFI_PASSWORD,
            sizeof(wifi_config.sta.password) - 1);

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());
    ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_NONE));
}

static void append_sample_to_frame(const imu_sample_t *sample, uint64_t watermark_timestamp_us)
{
    if (s_building_sample_index == 0) {
        s_building_frame.magic = TCP_FRAME_MAGIC;
        s_building_frame.count = IMU_SAMPLES_PER_FRAME;
        s_building_frame.seq_num = s_next_sequence++;
        s_building_frame.timestamp_us = watermark_timestamp_us;
    }

    int16_t *dst = s_building_frame.samples[s_building_sample_index];
    dst[0] = sample->ax;
    dst[1] = sample->ay;
    dst[2] = sample->az;
    dst[3] = sample->gx;
    dst[4] = sample->gy;
    dst[5] = sample->gz;

    s_building_sample_index++;
    if (s_building_sample_index < IMU_SAMPLES_PER_FRAME) {
        return;
    }

    BaseType_t ok = xRingbufferSend(s_frame_ringbuffer,
                                &s_building_frame,
                                sizeof(s_building_frame),
                                0);
    if (ok != pdTRUE) {
        s_ringbuffer_drop_count++;
    }

    s_building_sample_index = 0;
    s_frame_created_count++;
}

static int16_t read_le_i16(const uint8_t *p)
{
    return (int16_t)((uint16_t)p[0] | ((uint16_t)p[1] << 8));
}	

// word 指向完整 7-byte FIFO word。只有凑齐 accel 和 gyro 后才返回 true。
static bool consume_one_fifo_word(const uint8_t *word, imu_sample_t *out)
{
    uint8_t tag_sensor = (word[0] >> 3) & 0x1F;
    imu_xyz_t value = {
        .x = read_le_i16(&word[1]),
        .y = read_le_i16(&word[3]),
        .z = read_le_i16(&word[5]),
    };

    if (tag_sensor == 0x01) {              // Gyroscope
        if (s_pair_state.gyro_valid) {
            s_pair_error_count++;          // 未配对 gyro 被新的 gyro 覆盖
        }
        s_pair_state.gyro = value;
        s_pair_state.gyro_valid = true;
    } else if (tag_sensor == 0x02) {       // Accelerometer
        if (s_pair_state.accel_valid) {
            s_pair_error_count++;
        }
        s_pair_state.accel = value;
        s_pair_state.accel_valid = true;
    } else {
        s_tag_error_count++;
        return false;
    }

    if (!s_pair_state.accel_valid || !s_pair_state.gyro_valid) {
        return false;
    }

    out->ax = s_pair_state.accel.x;
    out->ay = s_pair_state.accel.y;
    out->az = s_pair_state.accel.z;
    out->gx = s_pair_state.gyro.x;
    out->gy = s_pair_state.gyro.y;
    out->gz = s_pair_state.gyro.z;

    s_pair_state.accel_valid = false;
    s_pair_state.gyro_valid = false;
    return true;
}


static void IRAM_ATTR imu_int1_isr(void *arg)
{
    (void)arg;
    BaseType_t higher_priority_task_woken = pdFALSE;

    s_int1_irq_count++;
    vTaskNotifyGiveFromISR(s_imu_irq_task_handle, &higher_priority_task_woken);

    if(higher_priority_task_woken == pdTRUE)
        portYIELD_FROM_ISR();
}

static void imu_int1_gpio_init(void)
{
    gpio_config_t io_conf = {
        .pin_bit_mask = 1ULL << PIN_NUM_INT1,
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_POSEDGE,
    };

    ESP_ERROR_CHECK(gpio_config(&io_conf));
    ESP_ERROR_CHECK(gpio_install_isr_service(ESP_INTR_FLAG_IRAM));
    ESP_ERROR_CHECK(gpio_isr_handler_add(PIN_NUM_INT1, imu_int1_isr, NULL));
}

static void imu_irq_test_task(void *arg)
{
    (void)arg;
    TickType_t last_log_tick = xTaskGetTickCount();
    uint32_t last_irq_count = 0;
    uint32_t last_sample_count = 0;
    uint32_t last_frame_created_count = 0;
    uint32_t last_frame_sent_count = 0;

    while(1)
    {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
        uint64_t watermark_timestamp_us = esp_timer_get_time();

        bool need_reset = false;
        uint16_t fifo_words_peak = 0;

        for (;;) 
        {
            lsm6dsr_fifo_status_t status;
            esp_err_t ret = lsm6dsr_fifo_get_status(&status);
            if (ret != ESP_OK) {
                s_fifo_status_error_count++;
                need_reset = true;
                break;
            }

            if (status.overrun || status.full) {
                s_fifo_overrun_count++;
                need_reset = true;
                break;
            }

            if (status.unread_words == 0) {
                break; // 当前 FIFO 已经读空
            }

            if (status.unread_words > fifo_words_peak) {
                fifo_words_peak = status.unread_words;
            }

            size_t words_to_read = status.unread_words;
            if (words_to_read > LSM6DSR_FIFO_MAX_WORDS_PER_READ) {
                words_to_read = LSM6DSR_FIFO_MAX_WORDS_PER_READ;
            }

            ret = lsm6dsr_fifo_read_words_dma(s_raw_fifo, words_to_read);
            if (ret != ESP_OK) {
                s_fifo_dma_error_count++;
                need_reset = true;
                break;
            }

            for (size_t i = 0; i < words_to_read; ++i) {
                imu_sample_t sample;
                const uint8_t *word = &s_raw_fifo[i * LSM6DSR_FIFO_WORD_SIZE];
                if (consume_one_fifo_word(word, &sample)) 
                {
                    s_last_sample = sample;
                    s_complete_sample_count++;

                    append_sample_to_frame(&sample, watermark_timestamp_us);
                }
            }

            // 重新读 FIFO_STATUS，而不是只读固定 200 word 后直接退出。
            // ISR 延迟期间新增的 FIFO word 也要在这一轮 drain 掉。
        }
        s_last_fifo_words = fifo_words_peak;

        if (need_reset) {
            memset(&s_pair_state, 0, sizeof(s_pair_state));
            esp_err_t reset_ret = lsm6dsr_fifo_reset();
            if (reset_ret != ESP_OK) {
                s_fifo_status_error_count++;
                ESP_LOGE(TAG, "FIFO reset failed: %s", esp_err_to_name(reset_ret));
                vTaskDelay(pdMS_TO_TICKS(10));
            }
        }
        TickType_t now = xTaskGetTickCount();
        if ((now - last_log_tick) >= pdMS_TO_TICKS(1000)) {
            // 1. 抓取当前所有计数器的快照
            uint32_t total_irq = s_int1_irq_count;
            uint32_t total_samples = s_complete_sample_count;
            uint32_t total_created = s_frame_created_count;
            uint32_t total_sent = s_frame_sent_count;

            uint32_t irq_per_sec = total_irq - last_irq_count;
            uint32_t samples_per_sec = total_samples - last_sample_count;
            uint32_t frame_created_per_sec = total_created - last_frame_created_count;
            uint32_t frame_sent_per_sec = total_sent - last_frame_sent_count;

            ESP_LOGI(TAG,
                     "INT1 GPIO%d: irq=%lu/s fifo_peak=%u sample=%lu/s "
                     "frame_new=%lu/s frame_sent=%lu/s",
                     PIN_NUM_INT1,
                     (unsigned long)irq_per_sec,
                     s_last_fifo_words,
                     (unsigned long)samples_per_sec,
                     (unsigned long)frame_created_per_sec,
                     (unsigned long)frame_sent_per_sec);

            ESP_LOGI(TAG,
                     "fifo: status_err=%lu ovr=%lu dma_err=%lu tag_err=%lu pair_err=%lu; "
                     "net: rb_drop=%lu tcp_err=%lu tcp_drop=%lu; "
                     "A=[%d,%d,%d] G=[%d,%d,%d]",
                     (unsigned long)s_fifo_status_error_count,
                     (unsigned long)s_fifo_overrun_count,
                     (unsigned long)s_fifo_dma_error_count,
                     (unsigned long)s_tag_error_count,
                     (unsigned long)s_pair_error_count,
                     (unsigned long)s_ringbuffer_drop_count,
                     (unsigned long)s_tcp_error_count,
                     (unsigned long)s_tcp_frame_drop_count,
                     s_last_sample.ax, s_last_sample.ay, s_last_sample.az,
                     s_last_sample.gx, s_last_sample.gy, s_last_sample.gz);

            last_irq_count = total_irq;
            last_sample_count = total_samples;
            last_frame_created_count = total_created;
            last_frame_sent_count = total_sent;
            last_log_tick = now;
        }
    }
}


static void tcp_send_task(void *arg)
{
    (void)arg;

    while (1) {
        // 没拿到 IP 时不尝试建连。
        xEventGroupWaitBits(s_wifi_event_group,
                            WIFI_CONNECTED_BIT,
                            pdFALSE,
                            pdTRUE,
                            portMAX_DELAY);

        int sock = tcp_connect_server();
        if (sock < 0) {
            s_tcp_error_count++;
            vTaskDelay(pdMS_TO_TICKS(1000));
            continue;
        }

        ESP_LOGI(TAG, "TCP connected: %s:%d", TCP_SERVER_IP, TCP_SERVER_PORT);

        bool reconnect = false;
        while (!reconnect) {
            size_t item_size = 0;
            tcp_frame_t *item = xRingbufferReceive(s_frame_ringbuffer,
                                                   &item_size,
                                                   pdMS_TO_TICKS(1000));
            if (item == NULL) {
                EventBits_t bits = xEventGroupGetBits(s_wifi_event_group);
                if ((bits & WIFI_CONNECTED_BIT) == 0) {
                    reconnect = true;
                }
                continue;
            }

            bool ok = (item_size == sizeof(tcp_frame_t)) &&
                      send_all(sock, (const uint8_t *)item, item_size);

            if (ok) {
                s_frame_sent_count++;
            } else {
                s_tcp_error_count++;
                s_tcp_frame_drop_count++;
                reconnect = true;
            }

            // 成功或失败都必须归还 item；不归还会耗尽 RingBuffer。
            vRingbufferReturnItem(s_frame_ringbuffer, item);
        }

        shutdown(sock, SHUT_RDWR);
        close(sock);
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

void app_main(void)
{
    esp_lsm6dsr_spi_init();
    vTaskDelay(pdMS_TO_TICKS(100));

    uint8_t id = 0;
    esp_err_t ret = ESP_FAIL;
    for (int attempt = 1; attempt <= 10; ++attempt) {
        ret = lsm6dsr_read_reg(WHO_AM_I, &id);
        ESP_LOGI(TAG, "WHO_AM_I attempt %d/10: 0x%02X (expected 0x%02X)",
                 attempt, id, LSM6DSR_ID);
        if (ret == ESP_OK && id == LSM6DSR_ID) {
            break;
        }
        if (ret != ESP_OK) {
            ESP_LOGE(TAG, "WHO_AM_I transaction failed: %s", esp_err_to_name(ret));
        }
        vTaskDelay(pdMS_TO_TICKS(100));
    }

    if (ret != ESP_OK || id != LSM6DSR_ID) {
        ESP_LOGE(TAG, "No LSM6DSR response. Check wiring!");
        return;
    }

    // 复位与传感器配置
    ESP_ERROR_CHECK(lsm6dsr_write_reg(CTRL3_C, 0x01));
    vTaskDelay(pdMS_TO_TICKS(10));
    do {
        ESP_ERROR_CHECK(lsm6dsr_read_reg(CTRL3_C, &id));
    } while (id & 0x01);

    ESP_ERROR_CHECK(lsm6dsr_write_reg(CTRL9_XL, 0x02)); // Disable I3C
    ESP_ERROR_CHECK(lsm6dsr_write_reg(CTRL3_C, 0x44));  // BDU + IF_INC

    ESP_LOGI(TAG, "LSM6DSR base configuration complete");

    s_frame_ringbuffer = xRingbufferCreate(FRAME_RINGBUFFER_SIZE,
                                           RINGBUF_TYPE_NOSPLIT);
    assert(s_frame_ringbuffer != NULL);

    BaseType_t imu_task_created = xTaskCreatePinnedToCore(
        imu_irq_test_task,
        "imu_irq_test",
        4096,
        NULL,
        10,
        &s_imu_irq_task_handle,
        1);
    assert(imu_task_created == pdPASS);

    /* The task and GPIO ISR must be ready before FIFO Continuous mode starts. */
    imu_int1_gpio_init();
    ESP_ERROR_CHECK(lsm6dsr_fifo_dma_init());
    ESP_ERROR_CHECK(lsm6dsr_fifo_config_high_rate());

    ESP_LOGI(TAG, "High-rate FIFO started: 6667 Hz, watermark=%u words, INT1=GPIO%d",
             LSM6DSR_FIFO_WATERMARK_WORDS, PIN_NUM_INT1);
    wifi_sta_init();
    ESP_LOGI(TAG, "Wi-Fi start requested");

    BaseType_t tcp_task_created = xTaskCreatePinnedToCore(
        tcp_send_task,
        "tcp_send",
        6144,
        NULL,
        8,
        NULL,
        0);
    assert(tcp_task_created == pdPASS);
}
` 

## A.7 Wi-Fi 凭据模板

文件：`main/wifi_private.h.example`

```c
#ifndef WIFI_PRIVATE_H
#define WIFI_PRIVATE_H

/* Copy this file to wifi_private.h and fill in a 2.4 GHz Wi-Fi network. */
#define WIFI_SSID     "YOUR_2_4GHZ_SSID"
#define WIFI_PASSWORD "YOUR_WIFI_PASSWORD"

#endif /* WIFI_PRIVATE_H */
```

> `main/wifi_private.h` 的真实内容不写入本文档。使用时复制模板并在本地填写。

---

# 附录 B：工程辅助配置

## B.1 `.gitignore`

```gitignore
# macOS
.DS_Store
.AppleDouble
.LSOverride

# Directory metadata
.directory

# Temporary files
*~
*.swp
*.swo
*.bak
*.tmp

# Log files
*.log

# Build artifacts and directories
**/build/
build/
*.o
*.a
*.out
*.exe # For any host-side utilities compiled on Windows

# ESP-IDF specific build outputs
*.bin
*.elf
*.map
flasher_args.json # Generated in build directory
sdkconfig.old
sdkconfig

# ESP-IDF dependencies
# For older versions or manual component management
/components/.idf/
**/components/.idf/
# For modern ESP-IDF component manager
managed_components/
# If ESP-IDF tools are installed/referenced locally to the project
.espressif/

# CMake generated files
CMakeCache.txt
CMakeFiles/
cmake_install.cmake
install_manifest.txt
CTestTestfile.cmake

# Python environment files
*.pyc
*.pyo
*.pyd
__pycache__/
*.egg-info/
dist/

# Virtual environment folders
venv/
.venv/
env/

# Language Servers
.clangd/
.ccls-cache/
compile_commands.json

# Windows specific
Thumbs.db
ehthumbs.db
Desktop.ini

# User-specific configuration files
*.user
*.workspace # General workspace files, can be from various tools
*.suo       # Visual Studio Solution User Options
*.sln.docstates # Visual Studio

# Local Wi-Fi credentials
/main/wifi_private.h
```

## B.2 关键 `sdkconfig` 摘要

`sdkconfig` 是 ESP-IDF 生成的配置文件，本文不复制完整 2294 行内容。当前与项目最相关的配置如下：

```ini
CONFIG_IDF_TARGET="esp32s3"
CONFIG_IDF_TARGET_ESP32S3=y
CONFIG_ESPTOOLPY_FLASHMODE="dio"
CONFIG_ESPTOOLPY_FLASHSIZE="2MB"
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ=160
CONFIG_ESP_MAIN_TASK_STACK_SIZE=3584
CONFIG_ESP_WIFI_ENABLED=y
CONFIG_ESP_WIFI_TASK_PINNED_TO_CORE_0=y
CONFIG_FREERTOS_UNICORE is not set
CONFIG_LOG_DEFAULT_LEVEL=3
CONFIG_LWIP_ENABLE=y
CONFIG_LWIP_MAX_SOCKETS=10
CONFIG_LWIP_MAX_ACTIVE_TCP=16
CONFIG_LWIP_MAX_LISTENING_TCP=16
CONFIG_LWIP_TCP_MSS=1440
CONFIG_LWIP_TCP_SND_BUF_DEFAULT=5760
CONFIG_LWIP_TCP_WND_DEFAULT=5760
```

如果需要重新生成配置，应使用：

```powershell
idf.py menuconfig
```

修改后重新编译，并确认没有意外改变目标芯片、Flash 参数、FreeRTOS 双核设置和 LWIP socket 配置。

## B.3 `.devcontainer/Dockerfile`

```dockerfile
ARG DOCKER_TAG=latest
FROM espressif/idf:${DOCKER_TAG}

ENV LC_ALL=C.UTF-8
ENV LANG=C.UTF-8

RUN apt-get update -y && apt-get install udev -y

RUN echo "source /opt/esp/idf/export.sh > /dev/null 2>&1" >> ~/.bashrc

ENTRYPOINT [ "/opt/esp/entrypoint.sh" ]

CMD ["/bin/bash", "-c"]
```

## B.4 `.devcontainer/devcontainer.json`

```json
{
	"name": "ESP-IDF QEMU",
	"build": {
		"dockerfile": "Dockerfile"
	},
	"customizations": {
		"vscode": {
			"settings": {
				"terminal.integrated.defaultProfile.linux": "bash",
				"idf.gitPath": "/usr/bin/git"
			},
			"extensions": [
				"espressif.esp-idf-extension",
				"espressif.esp-idf-web"
			]
		}
	},
	"runArgs": ["--privileged"]
}
```

## B.5 `.vscode/c_cpp_properties.json`

```json
{
  "configurations": [
    {
      "name": "ESP-IDF",
      "compilerPath": "D:\\esp\\v5.5.5\\tools\\xtensa-esp-elf\\esp-14.2.0_20260121\\xtensa-esp-elf\\bin\\xtensa-esp32-elf-gcc.exe",
      "compileCommands": "${config:idf.buildPath}/compile_commands.json",
      "intelliSenseMode": "gcc-x86",
      "includePath": [
        "${workspaceFolder}/**"
      ],
      "browse": {
        "path": [
          "${workspaceFolder}"
        ],
        "limitSymbolsToIncludedHeaders": true
      }
    }
  ],
  "version": 4
}
```

## B.6 `.vscode/launch.json`

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "gdbtarget",
      "request": "attach",
      "name": "Eclipse CDT GDB Adapter"
    }
  ]
}
```

## B.7 `.vscode/settings.json`

```json
{
  "idf.currentSetup": "D:\\esp\\v5.5.5\\esp-idf",
  "idf.openOcdConfigs": [
    "board/esp32s3-builtin.cfg"
  ],
  "idf.portWin": "COM7",
  "idf.customExtraVars": {
    "IDF_TARGET": "esp32s3"
  },
  "clangd.path": "D:\\esp\\v5.5.5\\tools\\esp-clang\\esp-19.1.2_20250312\\esp-clang\\bin\\clangd.exe",
  "clangd.arguments": [
    "--background-index",
    "--query-driver=**",
    "--compile-commands-dir=e:\\esp-project\\ESP32S3_IMU\\build"
  ],
  "idf.flashType": "UART"
}
```

---

# 附录 C：最终检查清单

## 硬件检查

- [ ] SPI SCLK 已连接 GPIO4。
- [ ] SPI MOSI/SDI 已连接 GPIO5。
- [ ] SPI MISO/SDO 已连接 GPIO6。
- [ ] CS 已连接 GPIO7。
- [ ] INT1 已连接 GPIO15。
- [ ] INT2 已连接 GPIO16，但确认当前代码没有使用 INT2。
- [ ] ESP32-S3 与 LSM6DSR 共地。
- [ ] 供电和电平兼容。

## 固件检查

- [ ] `WHO_AM_I` 为 `0x6B`。
- [ ] SPI Mode 为 3。
- [ ] FIFO DMA 初始化在 FIFO Continuous 之前。
- [ ] GPIO15 ISR 在 FIFO Continuous 之前安装。
- [ ] `fifo_peak` 接近 200。
- [ ] `status_err=0`。
- [ ] `ovr=0`。
- [ ] `dma_err=0`。
- [ ] `tag_err=0`。
- [ ] `pair_err=0`。
- [ ] `sample` 和 `frame_new × 100` 大致一致。
- [ ] 真实 Wi-Fi 凭据只在 `main/wifi_private.h` 中配置。

## 网络检查

- [ ] 电脑局域网 IP 与 `TCP_SERVER_IP` 一致。
- [ ] TCP 服务端监听 `TCP_SERVER_PORT`。
- [ ] 监听的是原始 TCP，不是 HTTP/WebSocket 服务。
- [ ] 服务端持续接收，不在第一次 `recv()` 后关闭。
- [ ] 服务端能处理半包和粘包。
- [ ] Windows 防火墙允许该端口。
- [ ] 8080 没有被 `NIApplicationWebServer` 或其他程序占用。
- [ ] `TCP connected` 后 `frame_sent/s` 大于 0。
- [ ] `errno=104` 不再持续出现。

## 性能检查

- [ ] 采集任务没有长期阻塞。
- [ ] `rb_drop` 不持续快速增长。
- [ ] `tcp_err` 不持续增加。
- [ ] `frame_sent/s` 接近 `frame_new/s`。
- [ ] 在长时间运行后重新测量真实采样率。

---

# 结语

这个项目的核心不是单独把 SPI、FIFO 或 TCP 任意一部分跑通，而是建立一条完整且可观察的数据链路：

```text
传感器可识别
→ FIFO 可持续产生数据
→ 中断及时通知
→ DMA 批量读取
→ TAG 正确配对
→ 100 sample 组帧
→ RingBuffer 解耦
→ Wi-Fi 获取 IP
→ TCP 完整发送
→ 上位机持续接收
```

每一层都有独立的状态和统计信息。后续排查问题时，应根据日志先判断故障位于传感器层、FIFO 层、组帧层、RingBuffer 层、Wi-Fi 层还是 TCP 对端，而不是同时修改多个模块。这样可以保持当前固件结构清晰，并为后续采样率校准、丢帧策略和协议升级留下空间。