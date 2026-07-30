# ESP32-S3 IMU 硬件连接

## 概念一句话
ESP32-S3 通过 SPI2 与 LSM6DSR 交换数据，并用独立的 INT1 GPIO 接收 FIFO 水位事件；核心关联是 [[LSM6DSR SPI 驱动]] 与 [[LSM6DSR FIFO 采集]]。

## 核心原理与图解
硬件链路分成两条：SPI 负责配置和批量读取，INT1 只负责把 FIFO 水位事件送入 ESP32-S3。当前素材中的固定连接为 SCLK=GPIO4、MOSI=GPIO5、MISO=GPIO6、CS=GPIO7、INT1=GPIO15；GPIO16 虽定义为 INT2，但当前采集链路没有使用。

~~~mermaid
flowchart LR
    MCU["ESP32-S3"] -->|"SPI2 / Mode 3 / 10 MHz"| SENSOR["LSM6DSR"]
    MCU -->|"GPIO15 上升沿"| IRQ["INT1 FIFO 水位事件"]
    MCU -.->|"GPIO16：当前未配置"| INT2["INT2"]
    GND["共地与兼容供电"] --- MCU
    GND --- SENSOR
~~~

> 图 1：SPI 数据通道与 INT1 事件通道彼此分工；图示根据 raw/ESP32S3_IMU_项目完整总结.md 的文字和代码提炼，raw 未提供可直接引用的图片资源。

## 硬件初始化与驱动时序

1. 完成供电、共地和 SPI2 引脚准备；具体复位脚及复位保持时间未在 raw 中给出，标记为待核实。
2. 配置 SPI2 为 Mode 3、10 MHz，加入 LSM6DSR 片选设备。
3. 读取 WHO_AM_I，raw 中预期值为 0x6B；读取失败时不继续启动采集链。
4. 向 CTRL3_C 写入 0x01 执行软件复位，等待复位位清零；随后向 CTRL9_XL 写入 0x02 关闭 I3C，再向 CTRL3_C 写入 0x44 开启 BDU 和地址自动递增。
5. 先配置加速度计、陀螺仪和 FIFO 为 Bypass，准备 DMA 缓冲区、FIFO 任务和 GPIO15 ISR。
6. 最后写入 FIFO Continuous 配置；这样可避免 FIFO watermark 在任务或 ISR 尚未就绪时触发而丢失第一次中断。

~~~mermaid
flowchart TD
    PWR["供电/共地<br/>复位条件待核实"] --> SPI["SPI2 Mode 3 / 10 MHz"]
    SPI --> ID["读取 WHO_AM_I = 0x6B"]
    ID --> RESET["CTRL3_C = 0x01<br/>等待复位完成"]
    RESET --> CFG["CTRL9_XL = 0x02<br/>CTRL3_C = 0x44"]
    CFG --> READY["创建任务、ISR、DMA 缓冲区"]
    READY --> FIFO["配置 FIFO Continuous"]
~~~

> 图 2：启动顺序必须先让软件消费链就绪，再开启 FIFO Continuous；复位保持时间和具体电源时序在 raw 中未给出，需结合板卡资料核实。

## 关键实现/数据结构
~~~c
#define PIN_NUM_CLK   4       // SCLK
#define PIN_NUM_MOSI  5       // ESP32 写入传感器
#define PIN_NUM_MISO  6       // 传感器返回数据
#define PIN_NUM_CS    7       // 片选，低有效
#define PIN_NUM_INT1  15      // FIFO watermark 中断
#define PIN_NUM_INT2  16      // 当前未接入采集状态机
#define SPI_CLOCK_HZ  (10 * 1000 * 1000)
~~~

## 关键寄存器映射

下表只列出 raw 明确出现的寄存器地址和配置值；raw 未逐位展开的字段不自行补全。

| 寄存器 | 16 进制地址 | 关键位域/功能说明 | raw 中的典型配置 | 写入后的硬件行为 |
|---|---:|---|---|---|
| WHO_AM_I | 0x0F | 器件身份读取；具体只读位域待核实 | 预期读到 0x6B | 用于确认 SPI 通信和器件身份 |
| CTRL1_XL | 0x10 | ODR、量程字段的具体位域待核实 | 0xA4 | raw 将其解释为加速度计 6667 Hz、±16 g；频率结论仍需数据手册复核 |
| CTRL2_G | 0x11 | ODR、量程字段的具体位域待核实 | 0xAC | raw 将其解释为陀螺仪 6667 Hz、±2000 dps；频率结论仍需数据手册复核 |
| CTRL3_C | 0x12 | 软件复位、BDU、地址自动递增；具体位域待核实 | 先 0x01，后 0x44 | 先触发复位，后开启 BDU 和寄存器地址自动递增 |
| CTRL9_XL | 0x18 | I3C 禁用配置；具体位域待核实 | 0x02 | 关闭 I3C，保持 SPI 访问链路 |
| FIFO_CTRL1 | 0x07 | watermark 低 8 位 | watermark 的低 8 位 | 设置 FIFO 水位低位 |
| FIFO_CTRL2 | 0x08 | watermark 高位 | watermark 的高位 | 与 CTRL1 共同形成 200 words 水位 |
| FIFO_CTRL3 | 0x09 | gyro/accel FIFO BDR；具体位域待核实 | 0xAA | raw 将其解释为 gyro/accel 6667 Hz 写入 FIFO |
| FIFO_CTRL4 | 0x0A | FIFO 模式；具体位域待核实 | 0x00 后 0x06 | 先 Bypass 清旧状态，后进入 Continuous |
| INT1_CTRL | 0x0D | INT1 映射；具体位域待核实 | 0x08 | 将 FIFO threshold 事件映射到 INT1 |
| FIFO_STATUS1/2 | 0x3A/0x3B | 未读 word、overrun/full 状态；具体位域待核实 | 运行时读取 | 判断 FIFO 是否需要继续读取或复位 |
| FIFO_DATA_OUT_TAG | 0x78 | FIFO word 入口；TAG 和六轴原始数据格式由驱动解析 | 每 word 7 字节 | 读出 TAG、X/Y/Z 高低字节组成的 FIFO word |

## 数据流与硬件避坑

- 硬件到软件流水线：INT1 watermark → GPIO15 ISR → FreeRTOS 任务通知 → 读取 FIFO_STATUS1/2 → DMA 批量读取 FIFO_DATA_OUT_TAG → 跳过 SPI 的 RX[0] 命令阶段字节 → 按 7 字节 word 解析 TAG 和 X/Y/Z → gyro/accel 配对 → 形成六轴 sample。
- FIFO 消费策略：一次最多读取 256 个 word；读取后重新检查 FIFO 状态，直到 unread words 为 0，避免中断期间新增数据形成 backlog。
- DMA 红线：DMA 缓冲区在开启高速 FIFO 前一次性分配，并要求 MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL；不要在 ISR 或实时采集路径首次申请内存。
- ISR 红线：GPIO15 ISR 只计数、调用 vTaskNotifyGiveFromISR() 并按需 portYIELD_FROM_ISR()，不得做 SPI、malloc/free、日志、TCP 或长循环。
- 协议红线：寄存器读取的真实数据从 RX[1] 开始；误读 RX[0] 会把命令阶段返回值当成寄存器数据。
- 时序红线：FIFO Continuous 必须放在任务、ISR 和 DMA 缓冲区准备完成之后；复位完成轮询和复位后的硬件延迟仍需以器件手册核实。

## 横向对比与关联
- [[ESP32-S3 IMU 启动时序]]：要求任务、ISR、DMA 缓冲区准备完成后再开启 FIFO。
- [[FIFO TAG 配对]]：负责把 gyro/accel 两类 FIFO word 组成一个六轴 sample。
- 硬件排查优先级：共地/供电 → CS 与 SPI 四线 → Mode 3 → INT1 极性与 GPIO15。

## 冲突与待核实
- raw 将 CTRL1_XL=0xA4、CTRL2_G=0xAC 描述为 6667 Hz 配置，但同时建议通过长时间统计和数据手册核对实际 ODR；本页不把该频率视为已最终确认。
- 模块的 SPI/I3C 选择脚和电平兼容性依赖具体板卡原理图，不能只依据 GPIO 定义推断。

## 来源
- raw 文件：ESP32S3_IMU_项目完整总结.md
