# ESP32-S3 IMU 启动时序

## 概念一句话
ESP32-S3 IMU 启动时序通过“先准备消费者、ISR 和 DMA，后开启 FIFO Continuous”的顺序消除第一次 watermark 中断竞态；核心关联是 [[LSM6DSR FIFO 采集]] 与 [[IMU 采集与网络解耦]]。

## 核心原理与图解
启动流程先验证传感器身份，再准备软件接收路径，最后让传感器开始高速产数：SPI → WHO_AM_I → 复位/基础配置 → RingBuffer → FIFO 任务 → GPIO15 ISR → DMA 缓冲区 → FIFO Continuous → Wi-Fi → TCP 任务。若 FIFO 先启动，第一次水位中断可能发生在任务或 ISR 尚未安装时。

~~~mermaid
sequenceDiagram
    participant App as "app_main"
    participant HW as "LSM6DSR"
    participant Task as "FIFO 任务"
    participant ISR as "GPIO15 ISR"
    participant Net as "TCP 任务"
    App->>HW: "SPI 初始化与 WHO_AM_I 重试"
    App->>HW: "软件复位和基础配置"
    App->>Task: "创建采集任务"
    App->>ISR: "安装 INT1 ISR"
    App->>App: "预分配 DMA 缓冲区"
    App->>HW: "开启 FIFO Continuous"
    App->>Net: "初始化 Wi-Fi 并创建 TCP 任务"
    HW-->>ISR: "watermark 上升沿"
    ISR-->>Task: "任务通知"
~~~

> 图 1：软件接收链路准备完成后才打开 FIFO 的启动时序；图示根据 raw 代码提炼，raw 未提供可直接引用的图片资源。

## 关键实现/数据结构
~~~c
esp_lsm6dsr_spi_init();                  // 1. SPI 可用
wait_sensor_stable();                    // 2. 等待硬件稳定
check_who_am_i_with_retry(10);           // 3. 身份失败则停止
create_ringbuffer_and_fifo_task();       // 4. 先准备消费者
imu_int1_gpio_init();                    // 5. 再安装 ISR
ESP_ERROR_CHECK(lsm6dsr_fifo_dma_init());// 6. 预分配 DMA
ESP_ERROR_CHECK(lsm6dsr_fifo_config_high_rate()); // 7. 最后启动 FIFO
wifi_sta_init();                         // 8. 网络异步启动
~~~

## 横向对比与关联
- [[LSM6DSR SPI 驱动]]：WHO_AM_I 是启动闸门，避免无效硬件继续进入采集。
- [[LSM6DSR FIFO 采集]]：FIFO Continuous 是启动时序的最后一个硬件动作。
- [[Wi-Fi TCP 状态流程]]：网络在采集链路已准备好后初始化，避免影响第一批 FIFO 数据。
- 与“先启动传感器再补软件接收路径”相比，当前顺序优先保证中断和 DMA 的接收闭环。

## 冲突与待核实
- raw 描述 app_main() 中 tcp_send 任务在 Wi-Fi 初始化后创建；具体任务创建时机若在代码版本间变化，应以当前固件源码为准。
- 失败回滚、任务删除、GPIO ISR 卸载和 DMA 释放未在素材中形成完整生命周期规范，不能把启动顺序当作完整资源管理方案。

## 来源
- raw 文件：ESP32S3_IMU_项目完整总结.md
