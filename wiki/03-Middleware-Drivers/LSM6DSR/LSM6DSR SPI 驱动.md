# LSM6DSR SPI 驱动

## 概念一句话
LSM6DSR SPI 驱动把 ESP-IDF SPI Master 的总线/设备/事务接口与 LSM6DSR 的寄存器协议连接起来，完成身份校验、寄存器配置和 FIFO DMA 访问；核心关联是 [[ESP32-S3 IMU 启动时序]] 与 [[LSM6DSR FIFO 采集]]。

## 核心原理与图解

### 初始化与访问时序

1. ESP32-S3 初始化 SPI2 总线：SCLK/MOSI/MISO/CS 为 GPIO4/5/6/7，项目配置 SPI Mode 3、10 MHz。
2. 通过 `spi_bus_add_device()` 挂载 LSM6DSR 片选设备，保存设备句柄。
3. 寄存器读命令把地址最高位置 1；SPI 事务的命令阶段占用 `RX[0]`，实际返回值从 `RX[1]` 读取。
4. 读取 `WHO_AM_I`（`0x0F`），素材中的验收值为 `0x6B`；失败时不启动 FIFO。
5. 写 `CTRL3_C`（`0x12`）执行软件复位，等待并轮询复位位；再写 `CTRL9_XL`（`0x18`）和 `CTRL3_C` 完成接口基础配置。
6. FIFO 批量读取额外包含 1 个命令字节，DMA 缓冲区必须满足项目要求的 DMA 能力和内部 RAM 条件。

~~~mermaid
sequenceDiagram
    participant App as "应用初始化"
    participant Drv as "项目 LSM6DSR 封装"
    participant SPI as "ESP-IDF SPI Master"
    participant IMU as "LSM6DSR"
    App->>Drv: "项目函数：初始化 SPI"
    Drv->>SPI: "spi_bus_initialize / spi_bus_add_device"
    App->>Drv: "项目函数：读取 WHO_AM_I"
    Drv->>SPI: "spi_device_polling_transmit"
    SPI->>IMU: "reg | 0x80 + dummy byte"
    IMU-->>SPI: "RX[0]=命令阶段，RX[1]=数据"
    SPI-->>Drv: "ESP_OK 或错误码"
    Drv-->>App: "ID 匹配后允许继续配置"
~~~

> 图 1：官方 SPI Master API 位于项目 LSM6DSR 封装之下；寄存器读标志和 `RX[1]` 偏移属于 LSM6DSR 协议实现，不是 ESP-IDF 自动完成的行为。

### 核心寄存器映射

| 寄存器 | 地址 | 关键位域/功能 | 项目配置 | 写入后的硬件行为 |
|---|---:|---|---|---|
| `WHO_AM_I` | `0x0F` | 器件身份只读值 | 期望 `0x6B` | 确认 SPI 通路和目标器件；不匹配时停止启动 |
| `CTRL3_C` | `0x12` | `SW_RESET` bit0；`IF_INC` bit2；`BDU` bit6 | `0x01` → `0x44` | 先软件复位；随后开启多字节地址递增与块数据更新 |
| `CTRL9_XL` | `0x18` | `I3C_DISABLE` bit1 | `0x02` | 关闭 I3C，保持 SPI 访问配置 |
| `FIFO_DATA_OUT_TAG` | `0x78` | TAG sensor 位于 bit[7:3] | 连续读取 | 每个 FIFO word 的首字节标识数据来源 |

## 关键实现/数据结构

以下函数名来自工程源码，均为**项目自定义函数**，不是官方 API；官方接口是 `spi_bus_initialize`、`spi_bus_add_device` 和 `spi_device_*`。

~~~c
spi_device_interface_config_t devcfg = {
    .clock_speed_hz = 10 * 1000 * 1000, // 项目配置
    .mode = 3, .spics_io_num = 7, .queue_size = 7,
};
uint8_t tx[2] = { reg | 0x80, 0x00 };     // 项目协议：读标志 + dummy
spi_transaction_t t = { .length = 16, .tx_buffer = tx, .rx_buffer = rx };
ESP_ERROR_CHECK(spi_device_polling_transmit(s_imu_spi, &t));
*value = rx[1];                           // 跳过 RX[0] 命令阶段
~~~

## 横向对比与关联

- [[LSM6DSR FIFO 采集]]：在寄存器访问边界之上增加 FIFO 状态读取、DMA drain 和异常恢复。
- [[FIFO TAG 配对]]：使用驱动读出的 7 字节 FIFO word 识别 gyro/accel。
- [[LSM6DSR ESP-IDF API 速查]]：只记录官方 ESP-IDF/FreeRTOS API，不收录本页项目封装函数。
- 与单次寄存器访问相比，FIFO DMA 更关注传输长度、DMA-capable 缓冲区和命令字节偏移。

## 硬件避坑

- SPI 读地址必须带 read bit；错误读取 `RX[0]` 会把命令阶段数据当成寄存器值。
- `CTRL3_C` 复位后必须等待并轮询 `SW_RESET` 自动清零；raw 记录的等待为 10 ms，最终时序仍需按目标器件验证。
- 开启 `IF_INC` 后才能安全进行连续多字节寄存器访问；该位是传感器寄存器行为，不是 SPI Master API 的参数。
- DMA 传输缓冲区需要 `MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL`；失败路径必须释放已经分配的缓冲区。

## 冲突与待核实

- raw 的高频 ODR/BDR 日志使用 6667 Hz 标注，但素材同时要求通过统计和数据手册校准；本页保留该冲突，不把它当成无条件正确值。
- `esp_lsm6dsr_spi_init`、`lsm6dsr_read_reg`、`lsm6dsr_write_reg` 等函数属于项目封装，是否改造成可恢复错误返回取决于工程故障策略。

## 来源

- raw 文件：`ESP32S3_IMU_项目完整总结.md`
