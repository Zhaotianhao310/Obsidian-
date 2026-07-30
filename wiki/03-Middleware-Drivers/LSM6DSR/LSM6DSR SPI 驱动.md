# LSM6DSR SPI 驱动

## 概念一句话
LSM6DSR SPI 驱动把 ESP-IDF SPI 总线、寄存器读写、WHO_AM_I 校验和 DMA 缓冲区封装成传感器访问边界；核心关联是 [[ESP32-S3 IMU 硬件连接]] 与 [[LSM6DSR FIFO 采集]]。

## 核心原理与图解
驱动初始化 SPI2，总线使用 GPIO4/5/6，设备片选为 GPIO7，Mode 3、10 MHz，并把最大传输长度设置为可以覆盖 FIFO 批量读取。寄存器读取时地址最高位置 1，命令阶段占用 RX[0]，真正数据从 RX[1] 开始；WHO_AM_I 不匹配时不应继续启动 FIFO 和网络链路。

~~~mermaid
sequenceDiagram
    participant App as "app_main"
    participant Drv as "LSM6DSR 驱动"
    participant SPI as "ESP-IDF SPI2"
    participant IMU as "LSM6DSR"
    App->>Drv: "esp_lsm6dsr_spi_init()"
    Drv->>SPI: "初始化总线并添加设备"
    App->>Drv: "lsm6dsr_read_reg(WHO_AM_I)"
    Drv->>IMU: "reg | 0x80 + dummy byte"
    IMU-->>Drv: "RX[0]=命令阶段, RX[1]=寄存器值"
    Drv-->>App: "0x6B 才允许继续"
~~~

> 图 1：SPI 初始化和身份校验的边界；图示根据 raw 代码提炼，raw 未提供可直接引用的图片资源。

## 关键实现/数据结构
~~~c
spi_device_interface_config_t devcfg = {
    .clock_speed_hz = SPI_CLOCK_HZ, // 10 MHz
    .mode = 3,                      // SPI Mode 3
    .spics_io_num = PIN_NUM_CS,
    .queue_size = 7,
};
uint8_t tx_data[2] = { reg | 0x80, 0x00 }; // 读命令
// RX[0] 是命令阶段，不能误当成寄存器数据
*value = rx_data[1];
~~~

## 横向对比与关联
- [[LSM6DSR FIFO 采集]]：在寄存器驱动之上增加 FIFO 状态、DMA 和连续 drain。
- [[FIFO TAG 配对]]：使用驱动输出的 7 字节 FIFO word 解析六轴 sample。
- [[LSM6DSR ESP-IDF API 速查]]：集中记录公共头文件和函数签名。
- 与普通单次寄存器读取相比，FIFO DMA 读取要预留命令字节，并要求 DMA-capable 内部 RAM。

## 冲突与待核实
- raw 的 esp_lsm6dsr_spi_init() 使用 ESP_ERROR_CHECK，初始化失败会触发 ESP-IDF 级别的错误处理；是否需要改为可恢复返回值，取决于项目故障策略。
- WHO_AM_I=0x6B 是素材中的验收条件；若实际板卡型号不同，应以器件数据手册和丝印确认。

## 来源
- raw 文件：ESP32S3_IMU_项目完整总结.md
