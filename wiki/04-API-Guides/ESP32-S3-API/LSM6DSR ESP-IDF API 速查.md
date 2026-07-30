# LSM6DSR ESP-IDF API 速查

## 基础头文件
~~~c
#include "lsm6dsr.h"
#include "driver/spi_master.h"
#include "esp_heap_caps.h"
#include "esp_err.h"
~~~

> 本页按 raw/ESP32S3_IMU_项目完整总结.md 附录中的公共头文件整理。寄存器编码、ODR 和 TAG 值仍需结合目标器件数据手册复核。

## esp_lsm6dsr_spi_init

- **作用**：初始化 SPI2 总线并把 LSM6DSR 加入总线。
- **函数原型/签名**：void esp_lsm6dsr_spi_init(void);
- **参数含义详解**：无参数；引脚、Mode 3、10 MHz 和 DMA 通道由驱动宏/实现决定。
- **返回值状态码**：无返回值；raw 实现内部使用 ESP_ERROR_CHECK，失败可能触发系统级错误处理。
- **使用 Demo**：
~~~c
esp_lsm6dsr_spi_init();
vTaskDelay(pdMS_TO_TICKS(100));
uint8_t id = 0;
ESP_ERROR_CHECK(lsm6dsr_read_reg(WHO_AM_I, &id));
~~~

## lsm6dsr_write_reg

- **作用**：写入单个 LSM6DSR 寄存器。
- **函数原型/签名**：esp_err_t lsm6dsr_write_reg(uint8_t reg, uint8_t data);
- **参数含义详解**：reg 为寄存器地址；data 为待写入的 8 位值，驱动会清除地址最高位。
- **返回值状态码**：成功返回 ESP_OK；SPI 事务失败返回对应 esp_err_t。
- **使用 Demo**：
~~~c
ESP_ERROR_CHECK(lsm6dsr_write_reg(CTRL3_C, 0x01)); // 软件复位
vTaskDelay(pdMS_TO_TICKS(10));
ESP_ERROR_CHECK(lsm6dsr_write_reg(CTRL9_XL, 0x02)); // 关闭 I3C
ESP_ERROR_CHECK(lsm6dsr_write_reg(CTRL3_C, 0x44)); // BDU+自动递增
~~~

## lsm6dsr_read_reg

- **作用**：读取单个寄存器的 8 位值。
- **函数原型/签名**：esp_err_t lsm6dsr_read_reg(uint8_t reg, uint8_t *value);
- **参数含义详解**：reg 为寄存器地址；value 必须指向可写的输出字节，不能为 NULL。
- **返回值状态码**：成功返回 ESP_OK；参数非法或 SPI 失败返回对应错误码。
- **使用 Demo**：
~~~c
uint8_t who = 0;
esp_err_t err = lsm6dsr_read_reg(WHO_AM_I, &who);
if (err != ESP_OK || who != LSM6DSR_ID) return err;
ESP_LOGI("IMU", "WHO_AM_I=0x%02X", who);
~~~

## lsm6dsr_fifo_get_status

- **作用**：读取 FIFO unread words、watermark、overrun 和 full 状态。
- **函数原型/签名**：esp_err_t lsm6dsr_fifo_get_status(lsm6dsr_fifo_status_t *status);
- **参数含义详解**：status 指向输出结构；字段分别表示未读 word 数量、水位、溢出和满标志。
- **返回值状态码**：status == NULL 返回 ESP_ERR_INVALID_ARG；SPI 成功返回 ESP_OK。
- **使用 Demo**：
~~~c
lsm6dsr_fifo_status_t st = {0};
ESP_ERROR_CHECK(lsm6dsr_fifo_get_status(&st));
if (st.overrun || st.full) ESP_ERROR_CHECK(lsm6dsr_fifo_reset());
if (st.unread_words) ESP_LOGD("IMU", "fifo=%u", st.unread_words);
~~~

## lsm6dsr_fifo_dma_init

- **作用**：预分配 FIFO DMA 读写缓冲区，并重复使用它们。
- **函数原型/签名**：esp_err_t lsm6dsr_fifo_dma_init(void);
- **参数含义详解**：无参数；raw 实现分配 MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL 内存。
- **返回值状态码**：成功返回 ESP_OK；内存不足返回 ESP_ERR_NO_MEM。
- **使用 Demo**：
~~~c
ESP_ERROR_CHECK(lsm6dsr_fifo_dma_init());
ESP_ERROR_CHECK(lsm6dsr_fifo_config_high_rate());
// 后续中断任务只复用 DMA 缓冲区，不在实时路径 malloc
~~~

## lsm6dsr_fifo_read_words_dma

- **作用**：从 FIFO_DATA_OUT_TAG DMA 读取指定数量的 FIFO words。
- **函数原型/签名**：esp_err_t lsm6dsr_fifo_read_words_dma(uint8_t *raw_out, size_t word_count);
- **参数含义详解**：raw_out 指向至少 word_count * 7 字节的输出缓冲区；word_count 必须大于 0 且不超过 256。
- **返回值状态码**：空指针、0 或超上限返回 ESP_ERR_INVALID_ARG；DMA 内存不足返回 ESP_ERR_NO_MEM；成功返回 ESP_OK。
- **使用 Demo**：
~~~c
uint8_t raw[256 * LSM6DSR_FIFO_WORD_SIZE];
size_t n = MIN(status.unread_words, 256u);
ESP_ERROR_CHECK(lsm6dsr_fifo_read_words_dma(raw, n));
for (size_t i = 0; i < n; ++i) consume_one_fifo_word(&raw[i * 7]);
~~~

## lsm6dsr_fifo_reset

- **作用**：先把 FIFO 置为 Bypass，再恢复 Continuous 模式，清理异常状态。
- **函数原型/签名**：esp_err_t lsm6dsr_fifo_reset(void);
- **参数含义详解**：无参数；该操作会丢弃当前 FIFO 中尚未解析的数据，并应同步清空配对状态。
- **返回值状态码**：任一寄存器写失败返回对应错误码；成功返回 ESP_OK。
- **使用 Demo**：
~~~c
if (status.overrun || status.full) {
    pair_state_clear();
    ESP_ERROR_CHECK(lsm6dsr_fifo_reset());
}
~~~

## lsm6dsr_fifo_config_high_rate

- **作用**：配置加速度计/陀螺仪 FIFO BDR、watermark、INT1 路由和 Continuous 模式。
- **函数原型/签名**：esp_err_t lsm6dsr_fifo_config_high_rate(void);
- **参数含义详解**：无参数；当前实现使用 watermark 200 words 和 raw 中给出的寄存器值。
- **返回值状态码**：任一寄存器写失败返回对应错误码；全部成功返回 ESP_OK。
- **使用 Demo**：
~~~c
ESP_ERROR_CHECK(lsm6dsr_fifo_dma_init());
imu_fifo_task_ready = true;
imu_int1_gpio_init();
ESP_ERROR_CHECK(lsm6dsr_fifo_config_high_rate()); // 最后开启 FIFO
~~~

## 避坑红线
- **SPI 偏移**：读寄存器时真实数据从 RX[1] 取；连续读 FIFO 时要跳过 RX[0] 命令阶段。
- **DMA 内存**：FIFO DMA 缓冲区必须满足 ESP-IDF DMA 能力要求，避免在中断或高频任务中反复分配。
- **FIFO 状态**：overrun 与 full 都要处理；reset 后必须清空 gyro/accel 配对状态。
- **边界检查**：raw_out 容量至少为 word_count * LSM6DSR_FIFO_WORD_SIZE，且 word_count <= 256。
- **时序**：先创建任务、安装 ISR、初始化 DMA，再开启 Continuous FIFO，避免第一次 watermark 事件丢失。
- **数据手册复核**：raw 对 ODR、FIFO TAG 和寄存器值的描述应在目标硬件上复核，不可把项目日志直接当成通用器件规格。

## 关联笔记
- [[LSM6DSR SPI 驱动]]
- [[LSM6DSR FIFO 采集]]
- [[FIFO TAG 配对]]
- [[ESP32-S3 IMU 启动时序]]

## 来源
- raw 文件：ESP32S3_IMU_项目完整总结.md
