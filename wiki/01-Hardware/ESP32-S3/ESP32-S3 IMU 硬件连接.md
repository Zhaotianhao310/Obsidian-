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

## 横向对比与关联
- [[LSM6DSR SPI 驱动]]：定义 SPI Mode、寄存器访问和 DMA 传输。
- [[LSM6DSR FIFO 采集]]：消费由 INT1 触发的 FIFO 数据，而不是在 GPIO ISR 中读取 SPI。
- [[ESP32-S3 IMU 启动时序]]：要求任务、ISR、DMA 缓冲区准备完成后再开启 FIFO。
- 硬件排查优先级：共地/供电 → CS 与 SPI 四线 → Mode 3 → INT1 极性与 GPIO15。

## 冲突与待核实
- raw 将 CTRL1_XL=0xA4、CTRL2_G=0xAC 描述为 6667 Hz 配置，但同时建议通过长时间统计和数据手册核对实际 ODR；本页不把该频率视为已最终确认。
- 模块的 SPI/I3C 选择脚和电平兼容性依赖具体板卡原理图，不能只依据 GPIO 定义推断。

## 来源
- raw 文件：ESP32S3_IMU_项目完整总结.md
