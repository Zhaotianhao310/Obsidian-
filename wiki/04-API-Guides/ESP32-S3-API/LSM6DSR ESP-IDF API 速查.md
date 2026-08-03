# LSM6DSR ESP-IDF 官方 API 速查

> **定位**：本页只收录 ESP-IDF 5.5.5、ESP-IDF 随附 FreeRTOS，以及 ESP-IDF 官方组件提供的 API。项目中自定义的 LSM6DSR 封装函数、业务函数和寄存器常量不属于本页 API。
>
> **来源**：`raw/ESP32S3_IMU_项目完整总结.md`。raw 文件只读，本页是对其中官方接口的归纳，不是对项目私有函数的复制。

## API 归属边界

| 类别 | 本页处理方式 | 示例 |
|---|---|---|
| ESP-IDF 官方 API | 收录作用、签名、参数、返回值和 Demo | `spi_bus_initialize`、`gpio_config`、`esp_wifi_start` |
| ESP-IDF 随附 FreeRTOS API | 收录；它们是固件运行时官方接口，但不是乐鑫专属设计 | `xTaskCreatePinnedToCore`、`xRingbufferSend` |
| POSIX/lwIP 标准接口 | 不在本页展开 | `socket`、`connect`、`send`、`setsockopt` |
| 项目自定义函数 | 明确排除，不提供“官方 API”说明 | `esp_lsm6dsr_spi_init`、`lsm6dsr_read_reg`、`wifi_sta_init` |
| LSM6DSR 寄存器和协议常量 | 不属于 ESP-IDF API | `WHO_AM_I`、`FIFO_CTRL4`、`FIFO_DATA_OUT_TAG` |

> 函数名出现在“排除清单”中，只是为了标明归属；它们没有被当作 API 条目，也没有为其编写官方签名或 Demo。

## 基础头文件

~~~c
#include "driver/spi_master.h"       // SPI 主机总线和事务
#include "driver/gpio.h"             // GPIO 配置和 ISR 服务
#include "esp_event.h"                // 默认事件循环和事件处理器
#include "esp_netif.h"                // 默认 Wi-Fi STA 网络接口
#include "esp_wifi.h"                 // Wi-Fi 初始化、模式和连接
#include "nvs_flash.h"                // NVS Flash 初始化和擦除
#include "freertos/FreeRTOS.h"        // pdPASS、TickType_t 等
#include "freertos/task.h"            // 任务和任务通知
#include "freertos/ringbuf.h"         // Ringbuffer API
#include "esp_err.h"                  // esp_err_t、ESP_OK
#include "esp_intr_alloc.h"          // ESP_INTR_FLAG_IRAM
#include "esp_log.h"                 // ESP_LOGW 等日志宏
#include <string.h>                   // memcpy 示例
~~~

> 本页按 ESP-IDF 5.5.5 整理。升级 ESP-IDF 后，应重新核对对应版本的头文件、结构体字段和官方 API Reference。

## 核心函数详解

以下按官方模块分组整理各 API。

## 一、SPI Master 官方 API

### `spi_bus_initialize`

- **作用**：初始化指定 SPI 主机总线，并根据 DMA 参数建立总线资源。
- **函数原型/签名**：`esp_err_t spi_bus_initialize(spi_host_device_t host_id, const spi_bus_config_t *bus_config, spi_dma_chan_t dma_chan);`
- **参数含义详解**：`host_id` 是 SPI 主机编号；`bus_config` 指向总线引脚和最大传输长度配置；`dma_chan` 选择 DMA 通道，常用 `SPI_DMA_CH_AUTO`。
- **返回值状态码**：成功返回 `ESP_OK`；参数非法返回 `ESP_ERR_INVALID_ARG`；总线已初始化或状态不允许时常见为 `ESP_ERR_INVALID_STATE`；资源不足时可能返回 `ESP_ERR_NO_MEM`。
- **使用 Demo**：

~~~c
spi_bus_config_t buscfg = {
    .mosi_io_num = 5, .miso_io_num = 6, .sclk_io_num = 4,
    .quadwp_io_num = -1, .quadhd_io_num = -1, .max_transfer_sz = 4096,
};
ESP_ERROR_CHECK(spi_bus_initialize(SPI2_HOST, &buscfg, SPI_DMA_CH_AUTO));
~~~

### `spi_bus_add_device`

- **作用**：将一个 SPI 从设备挂载到已经初始化的 SPI 主机总线。
- **函数原型/签名**：`esp_err_t spi_bus_add_device(spi_host_device_t host_id, const spi_device_interface_config_t *dev_config, spi_device_handle_t *handle);`
- **参数含义详解**：`host_id` 必须对应已初始化的主机；`dev_config` 配置时钟、SPI 模式、CS 引脚和队列深度；`handle` 用于接收设备句柄，后续事务 API 依赖它。
- **返回值状态码**：成功返回 `ESP_OK`；参数非法返回 `ESP_ERR_INVALID_ARG`；主机未初始化或状态不正确返回 `ESP_ERR_INVALID_STATE`；设备或总线资源不足时可能返回 `ESP_ERR_NO_MEM`。
- **使用 Demo**：

~~~c
spi_device_interface_config_t devcfg = {
    .clock_speed_hz = 10 * 1000 * 1000, .mode = 3,
    .spics_io_num = 7, .queue_size = 4,
};
spi_device_handle_t handle = NULL;
ESP_ERROR_CHECK(spi_bus_add_device(SPI2_HOST, &devcfg, &handle));
~~~

### `spi_device_polling_transmit`

- **作用**：以轮询方式完成一次 SPI 事务；调用线程等待事务结束后返回。
- **函数原型/签名**：`esp_err_t spi_device_polling_transmit(spi_device_handle_t handle, spi_transaction_t *trans_desc);`
- **参数含义详解**：`handle` 是 `spi_bus_add_device` 返回的设备句柄；`trans_desc` 指向事务描述符，至少配置 `length`，并按读写方向设置 `tx_buffer`、`rx_buffer`。
- **返回值状态码**：成功返回 `ESP_OK`；句柄或事务参数非法返回 `ESP_ERR_INVALID_ARG`；设备状态不允许时返回相应 `esp_err_t`，常见为 `ESP_ERR_INVALID_STATE`。
- **使用 Demo**：

~~~c
uint8_t tx[2] = { 0x12, 0x44 };
spi_transaction_t trans = { 0 };
trans.length = sizeof(tx) * 8;
trans.tx_buffer = tx;
ESP_ERROR_CHECK(spi_device_polling_transmit(handle, &trans));
~~~

### `spi_device_transmit`

- **作用**：通过 SPI 设备队列完成一次阻塞事务；驱动负责提交、等待并回收该事务。
- **函数原型/签名**：`esp_err_t spi_device_transmit(spi_device_handle_t handle, spi_transaction_t *trans_desc);`
- **参数含义详解**：`handle` 是目标设备句柄；`trans_desc` 指向事务描述符；DMA 传输时，缓冲区还必须满足对应的 DMA 内存约束。
- **返回值状态码**：成功返回 `ESP_OK`；参数非法返回 `ESP_ERR_INVALID_ARG`；设备或总线状态错误返回相应 `esp_err_t`。不要只判断“是否有返回”，应明确检查 `ESP_OK`。
- **使用 Demo**：

~~~c
uint8_t tx[13] = { 0 };
uint8_t rx[13] = { 0 };
spi_transaction_t trans = { .length = sizeof(tx) * 8,
                             .tx_buffer = tx, .rx_buffer = rx };
ESP_ERROR_CHECK(spi_device_transmit(handle, &trans));
~~~

### SPI 官方关键类型

这些是 SPI Master 官方头文件中的配置/事务类型，不是项目自定义类型：

| 类型 | 关键字段/用途 |
|---|---|
| `spi_bus_config_t` | `mosi_io_num`、`miso_io_num`、`sclk_io_num`、`max_transfer_sz` 等总线配置 |
| `spi_device_interface_config_t` | `clock_speed_hz`、`mode`、`spics_io_num`、`queue_size` 等设备配置 |
| `spi_transaction_t` | `length`、`tx_buffer`、`rx_buffer` 等单次事务描述 |
| `spi_device_handle_t` | `spi_bus_add_device` 返回的设备句柄 |

## 二、GPIO 官方 API

### `gpio_config`

- **作用**：一次性配置 GPIO 的方向、上下拉、输入输出能力和中断类型。
- **函数原型/签名**：`esp_err_t gpio_config(const gpio_config_t *pGPIOConfig);`
- **参数含义详解**：`pGPIOConfig` 指向 `gpio_config_t`；`pin_bit_mask` 指定 GPIO 集合；`mode` 设置输入/输出模式；`pull_up_en`、`pull_down_en` 配置上下拉；`intr_type` 设置中断边沿或电平。
- **返回值状态码**：成功返回 `ESP_OK`；配置指针、GPIO 编号或字段组合非法时返回 `ESP_ERR_INVALID_ARG`。
- **使用 Demo**：

~~~c
gpio_config_t io = {
    .pin_bit_mask = 1ULL << 15, .mode = GPIO_MODE_INPUT,
    .pull_up_en = GPIO_PULLUP_DISABLE, .pull_down_en = GPIO_PULLDOWN_DISABLE,
    .intr_type = GPIO_INTR_POSEDGE,
};
ESP_ERROR_CHECK(gpio_config(&io));
~~~

### `gpio_install_isr_service`

- **作用**：安装 GPIO 中断服务，并为后续 GPIO ISR handler 注册提供运行时基础。
- **函数原型/签名**：`esp_err_t gpio_install_isr_service(int intr_alloc_flags);`
- **参数含义详解**：`intr_alloc_flags` 是中断分配标志，例如 `ESP_INTR_FLAG_IRAM`；标志必须与 ISR 代码及其调用链的内存驻留要求匹配。
- **返回值状态码**：成功返回 `ESP_OK`；服务已经安装通常返回 `ESP_ERR_INVALID_STATE`；内存不足可能返回 `ESP_ERR_NO_MEM`。
- **使用 Demo**：

~~~c
esp_err_t err = gpio_install_isr_service(ESP_INTR_FLAG_IRAM);
if (err != ESP_OK && err != ESP_ERR_INVALID_STATE) {
    ESP_ERROR_CHECK(err);
}
~~~

### `gpio_isr_handler_add`

- **作用**：为指定 GPIO 添加 ISR 回调。
- **函数原型/签名**：`esp_err_t gpio_isr_handler_add(gpio_num_t gpio_num, gpio_isr_t isr_handler, void *args);`
- **参数含义详解**：`gpio_num` 是目标引脚；`isr_handler` 是调用方提供的中断回调；`args` 原样传给回调，可为 `NULL`。回调必须遵守 ISR 上下文限制。
- **返回值状态码**：成功返回 `ESP_OK`；GPIO 编号或回调非法返回 `ESP_ERR_INVALID_ARG`；ISR 服务未安装时返回 `ESP_ERR_INVALID_STATE`；回调表资源不足时可能返回 `ESP_ERR_NO_MEM`。
- **使用 Demo**：

~~~c
static void IRAM_ATTR gpio_callback(void *arg) { (void)arg; }
ESP_ERROR_CHECK(gpio_install_isr_service(ESP_INTR_FLAG_IRAM));
ESP_ERROR_CHECK(gpio_isr_handler_add(GPIO_NUM_15, gpio_callback, NULL));
~~~

## 三、FreeRTOS 与 ESP-IDF 扩展官方 API

### `xTaskCreatePinnedToCore`

- **作用**：创建任务并将其固定到指定 CPU 核心；ESP32-S3 为双核目标时可用于隔离采集和网络任务。
- **函数原型/签名**：`BaseType_t xTaskCreatePinnedToCore(TaskFunction_t pvTaskCode, const char * const pcName, const uint32_t usStackDepth, void * const pvParameters, UBaseType_t uxPriority, TaskHandle_t * const pxCreatedTask, const BaseType_t xCoreID);`
- **参数含义详解**：`pvTaskCode` 是任务入口；`pcName` 是调试名称；`usStackDepth` 是栈深度；`pvParameters` 是入口参数；`uxPriority` 是优先级；`pxCreatedTask` 接收句柄；`xCoreID` 指定核心或使用 `tskNO_AFFINITY`。
- **返回值状态码**：成功返回 `pdPASS`；任务控制块或栈分配失败返回 `errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY`（通常按 `pdFAIL` 处理）。
- **使用 Demo**：

~~~c
TaskHandle_t task = NULL;
BaseType_t ok = xTaskCreatePinnedToCore(task_entry, "imu", 4096,
                                        NULL, 10, &task, 1);
configASSERT(ok == pdPASS);
~~~

> `task_entry` 是调用方提供的任务回调，仅作为函数指针占位，不是本页收录的 API。

### `vTaskNotifyGiveFromISR`

- **作用**：在 ISR 中向指定任务递增一个直接任务通知值，适合用作轻量事件唤醒。
- **函数原型/签名**：`void vTaskNotifyGiveFromISR(TaskHandle_t xTaskToNotify, BaseType_t *pxHigherPriorityTaskWoken);`
- **参数含义详解**：`xTaskToNotify` 是目标任务句柄；`pxHigherPriorityTaskWoken` 接收是否需要触发高优先级任务切换的结果，通常随后传给 `portYIELD_FROM_ISR`。
- **返回值状态码**：无返回值；错误主要表现为句柄生命周期错误或 ISR 上下文中调用了不允许阻塞的 API。
- **使用 Demo**：

~~~c
BaseType_t higher = pdFALSE;
vTaskNotifyGiveFromISR(task, &higher);
if (higher == pdTRUE) {
    portYIELD_FROM_ISR();
}
~~~

### `ulTaskNotifyTake`

- **作用**：阻塞等待任务通知，并按参数决定退出时清零或递减通知计数。
- **函数原型/签名**：`uint32_t ulTaskNotifyTake(BaseType_t xClearCountOnExit, TickType_t xTicksToWait);`
- **参数含义详解**：`xClearCountOnExit` 为 `pdTRUE` 时退出后清零，为 `pdFALSE` 时递减；`xTicksToWait` 是最大阻塞 tick 数，可用 `portMAX_DELAY` 表示长期等待。
- **返回值状态码**：返回取出的通知计数；超时且没有通知时返回 `0`，它不是 `esp_err_t`。
- **使用 Demo**：

~~~c
uint32_t count = ulTaskNotifyTake(pdTRUE, pdMS_TO_TICKS(1000));
if (count == 0) {
    // 超时：本轮没有收到 ISR 通知
}
~~~

### `xRingbufferCreate`

- **作用**：创建 FreeRTOS Ringbuffer，返回供发送和接收 API 使用的句柄。
- **函数原型/签名**：`RingbufHandle_t xRingbufferCreate(size_t xBufferSize, RingbufferType_t xBufferType);`
- **参数含义详解**：`xBufferSize` 是环形缓冲区容量；`xBufferType` 选择 `RINGBUF_TYPE_NOSPLIT`、`RINGBUF_TYPE_ALLOWSPLIT` 或字节缓冲类型。
- **返回值状态码**：成功返回非 `NULL` 句柄；创建失败返回 `NULL`，常见原因是内存不足或容量参数不合理。
- **使用 Demo**：

~~~c
RingbufHandle_t rb = xRingbufferCreate(8192, RINGBUF_TYPE_NOSPLIT);
configASSERT(rb != NULL);
uint8_t frame[64] = { 0 };
(void)xRingbufferSend(rb, frame, sizeof(frame), pdMS_TO_TICKS(10));
~~~

### `xRingbufferSend`

- **作用**：向 Ringbuffer 发送一段数据；在空间不足时可等待指定 tick 数。
- **函数原型/签名**：`BaseType_t xRingbufferSend(RingbufHandle_t xRingbuffer, const void *pvItem, size_t xItemSize, TickType_t xTicksToWait);`
- **参数含义详解**：`xRingbuffer` 是句柄；`pvItem` 指向待复制的数据；`xItemSize` 是字节数；`xTicksToWait` 是空间不足时的最大等待时间。
- **返回值状态码**：成功返回 `pdTRUE`；超时、句柄无效或空间不足时返回 `pdFALSE`。
- **使用 Demo**：

~~~c
const uint8_t payload[] = { 0x01, 0x02, 0x03 };
BaseType_t ok = xRingbufferSend(rb, payload, sizeof(payload),
                                pdMS_TO_TICKS(20));
if (ok != pdTRUE) {
    // 发送失败或等待超时
}
~~~

### `xRingbufferReceive`

- **作用**：从 Ringbuffer 取出一项数据，并返回该项的地址和长度。
- **函数原型/签名**：`void *xRingbufferReceive(RingbufHandle_t xRingbuffer, size_t *pxItemSize, TickType_t xTicksToWait);`
- **参数含义详解**：`xRingbuffer` 是句柄；`pxItemSize` 接收数据长度；`xTicksToWait` 是没有数据时的最大等待时间。返回的内存由 Ringbuffer 管理。
- **返回值状态码**：成功返回数据指针；超时或没有数据时返回 `NULL`。
- **使用 Demo**：

~~~c
size_t item_size = 0;
void *item = xRingbufferReceive(rb, &item_size, pdMS_TO_TICKS(100));
if (item != NULL) {
    // 读取 item[0..item_size-1]
    vRingbufferReturnItem(rb, item);
}
~~~

### `vRingbufferReturnItem`

- **作用**：归还 `xRingbufferReceive` 取出的数据项，使 Ringbuffer 可以回收并复用该空间。
- **函数原型/签名**：`void vRingbufferReturnItem(RingbufHandle_t xRingbuffer, void *pvItem);`
- **参数含义详解**：`xRingbuffer` 必须是取出该项的同一个句柄；`pvItem` 必须是 `xRingbufferReceive` 返回的指针，不能传入普通 `malloc` 指针。
- **返回值状态码**：无返回值；错误使用属于调用方生命周期错误，可能导致 Ringbuffer 状态损坏。
- **使用 Demo**：

~~~c
size_t size = 0;
void *item = xRingbufferReceive(rb, &size, portMAX_DELAY);
if (item != NULL) {
    process_bytes((const uint8_t *)item, size);
    vRingbufferReturnItem(rb, item);
}
~~~

> `process_bytes` 是示例中的调用方处理函数，不是本页 API。

## 四、ESP-NETIF 与事件官方 API

### `esp_netif_init`

- **作用**：初始化 ESP-NETIF 网络接口组件，为 Wi-Fi、以太网等接口创建网络栈基础环境。
- **函数原型/签名**：`esp_err_t esp_netif_init(void);`
- **参数含义详解**：无参数；通常在创建默认事件循环和默认 Wi-Fi STA 网络接口之前调用。
- **返回值状态码**：成功返回 `ESP_OK`；重复初始化或状态不允许时可能返回 `ESP_ERR_INVALID_STATE`。
- **使用 Demo**：

~~~c
esp_err_t err = esp_netif_init();
if (err != ESP_OK && err != ESP_ERR_INVALID_STATE) {
    ESP_ERROR_CHECK(err);
}
~~~

### `esp_event_loop_create_default`

- **作用**：创建 ESP-IDF 默认系统事件循环。
- **函数原型/签名**：`esp_err_t esp_event_loop_create_default(void);`
- **参数含义详解**：无参数；默认事件循环由 Wi-Fi/IP 等组件发布和接收系统事件。
- **返回值状态码**：成功返回 `ESP_OK`；默认循环已经存在时通常返回 `ESP_ERR_INVALID_STATE`；资源不足时可能返回 `ESP_ERR_NO_MEM`。
- **使用 Demo**：

~~~c
esp_err_t err = esp_event_loop_create_default();
if (err != ESP_OK && err != ESP_ERR_INVALID_STATE) {
    ESP_ERROR_CHECK(err);
}
~~~

### `esp_netif_create_default_wifi_sta`

- **作用**：创建并配置默认 Wi-Fi STA 网络接口，同时把它接入默认事件循环。
- **函数原型/签名**：`esp_netif_t *esp_netif_create_default_wifi_sta(void);`
- **参数含义详解**：无参数；返回值是默认 STA 网络接口句柄，通常交由系统组件使用，不应随意释放或重复创建。
- **返回值状态码**：成功返回非 `NULL` 的 `esp_netif_t *`；创建失败返回 `NULL`。它不使用 `esp_err_t` 返回错误码。
- **使用 Demo**：

~~~c
esp_netif_t *sta_netif = esp_netif_create_default_wifi_sta();
configASSERT(sta_netif != NULL);
// 后续将 WIFI_IF_STA 配置交给 esp_wifi_set_config
~~~

### `esp_event_handler_instance_register`

- **作用**：向指定事件基座和事件 ID 注册一个事件处理器实例。
- **函数原型/签名**：`esp_err_t esp_event_handler_instance_register(esp_event_base_t event_base, int32_t event_id, esp_event_handler_t event_handler, void *event_handler_arg, esp_event_handler_instance_t *instance);`
- **参数含义详解**：`event_base` 指定事件基座；`event_id` 指定事件或使用 `ESP_EVENT_ANY_ID`；`event_handler` 是回调；`event_handler_arg` 是回调参数；`instance` 接收实例句柄，可供注销使用。
- **返回值状态码**：成功返回 `ESP_OK`；参数非法返回 `ESP_ERR_INVALID_ARG`；事件循环或资源状态错误返回相应 `esp_err_t`，常见为 `ESP_ERR_INVALID_STATE` 或 `ESP_ERR_NO_MEM`。
- **使用 Demo**：

~~~c
esp_event_handler_instance_t instance = NULL;
ESP_ERROR_CHECK(esp_event_handler_instance_register(
    WIFI_EVENT, ESP_EVENT_ANY_ID, event_handler, NULL, &instance));
~~~

> `event_handler` 是调用方提供的回调占位符，不是本页收录的 API。

## 五、Wi-Fi 官方 API

### `esp_wifi_init`

- **作用**：使用 `wifi_init_config_t` 初始化 Wi-Fi 驱动。
- **函数原型/签名**：`esp_err_t esp_wifi_init(const wifi_init_config_t *config);`
- **参数含义详解**：`config` 指向 Wi-Fi 初始化配置，通常由 `WIFI_INIT_CONFIG_DEFAULT()` 产生；配置对象在调用时必须有效。
- **返回值状态码**：成功返回 `ESP_OK`；参数非法返回 `ESP_ERR_INVALID_ARG`；驱动已初始化或状态不允许时返回 `ESP_ERR_WIFI_INIT_STATE`/相应 `esp_err_t`，以当前 IDF 头文件为准。
- **使用 Demo**：

~~~c
wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
esp_err_t err = esp_wifi_init(&cfg);
ESP_ERROR_CHECK(err);
~~~

### `esp_wifi_set_mode`

- **作用**：设置 Wi-Fi 工作模式，例如 STA、AP 或 STA+AP。
- **函数原型/签名**：`esp_err_t esp_wifi_set_mode(wifi_mode_t mode);`
- **参数含义详解**：`mode` 使用 `WIFI_MODE_STA`、`WIFI_MODE_AP` 或组合模式；一般在 `esp_wifi_start` 前设置。
- **返回值状态码**：成功返回 `ESP_OK`；驱动未初始化、模式非法或当前状态不允许时返回相应 `esp_err_t`，常见为 `ESP_ERR_WIFI_NOT_INIT`、`ESP_ERR_INVALID_ARG` 或 `ESP_ERR_WIFI_MODE`。
- **使用 Demo**：

~~~c
esp_err_t err = esp_wifi_set_mode(WIFI_MODE_STA);
if (err != ESP_OK)
    return err;
~~~

### `esp_wifi_set_config`

- **作用**：设置指定 Wi-Fi 接口的配置，例如 STA 的 SSID 和密码。
- **函数原型/签名**：`esp_err_t esp_wifi_set_config(wifi_interface_t interface, wifi_config_t *conf);`
- **参数含义详解**：`interface` 指定 `WIFI_IF_STA` 或 `WIFI_IF_AP`；`conf` 指向对应的 `wifi_config_t`，其中 STA 配置字段由调用方填充。
- **返回值状态码**：成功返回 `ESP_OK`；配置指针或接口非法返回 `ESP_ERR_INVALID_ARG`；驱动未初始化或模式不匹配时返回相应 `esp_err_t`。
- **使用 Demo**：

~~~c
wifi_config_t config = { 0 };
memcpy(config.sta.ssid, "YOUR_SSID", 9);
memcpy(config.sta.password, "YOUR_PASSWORD", 13);
ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &config));
~~~

### `esp_wifi_start`

- **作用**：启动已经初始化并配置好的 Wi-Fi 驱动。
- **函数原型/签名**：`esp_err_t esp_wifi_start(void);`
- **参数含义详解**：无参数；调用前应完成 `esp_wifi_init`、模式设置和必要的接口配置。
- **返回值状态码**：成功返回 `ESP_OK`；驱动未初始化、重复启动或配置状态不允许时返回相应 `esp_err_t`。
- **使用 Demo**：

~~~c
ESP_ERROR_CHECK(esp_wifi_init(&cfg));
ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &config));
ESP_ERROR_CHECK(esp_wifi_start());
~~~

### `esp_wifi_connect`

- **作用**：让 STA 接口发起连接流程；连接结果通过 Wi-Fi/IP 事件通知。
- **函数原型/签名**：`esp_err_t esp_wifi_connect(void);`
- **参数含义详解**：无参数；调用前必须完成 STA 配置和 Wi-Fi 启动；不要把它当作“已获得 IP”的同步等待函数。
- **返回值状态码**：成功返回 `ESP_OK` 表示连接请求已提交；未初始化、模式不匹配或状态不允许时返回相应 `esp_err_t`。
- **使用 Demo**：

~~~c
ESP_ERROR_CHECK(esp_wifi_start());
esp_err_t err = esp_wifi_connect();
if (err != ESP_OK) {
    ESP_LOGW("wifi", "connect request failed: %s", esp_err_to_name(err));
}
~~~

### `esp_wifi_set_ps`

- **作用**：设置 Wi-Fi 省电策略。
- **函数原型/签名**：`esp_err_t esp_wifi_set_ps(wifi_ps_type_t type);`
- **参数含义详解**：`type` 可使用 `WIFI_PS_NONE`、`WIFI_PS_MIN_MODEM` 等官方枚举值；高吞吐、低延迟采集链路常需根据功耗目标选择策略。
- **返回值状态码**：成功返回 `ESP_OK`；驱动未初始化、枚举值非法或状态不允许时返回相应 `esp_err_t`。
- **使用 Demo**：

~~~c
esp_err_t err = esp_wifi_set_ps(WIFI_PS_NONE);
if (err != ESP_OK)
    ESP_LOGW("wifi", "set power save failed");
~~~

## 六、NVS Flash 官方 API

### `nvs_flash_init`

- **作用**：初始化默认 NVS 分区，使 Wi-Fi 等组件能够使用持久化配置。
- **函数原型/签名**：`esp_err_t nvs_flash_init(void);`
- **参数含义详解**：无参数；默认操作 `nvs` 分区。应用启动阶段应先处理分区版本变化和空间不足情况。
- **返回值状态码**：成功返回 `ESP_OK`；常见异常是 `ESP_ERR_NVS_NO_FREE_PAGES` 和 `ESP_ERR_NVS_NEW_VERSION_FOUND`，表示需要擦除后重新初始化；其他错误按 `esp_err_t` 处理。
- **使用 Demo**：

~~~c
esp_err_t err = nvs_flash_init();
if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());
    err = nvs_flash_init();
}
ESP_ERROR_CHECK(err);
~~~

### `nvs_flash_erase`

- **作用**：擦除默认 NVS 分区中的内容，通常用于处理分区格式或版本不兼容。
- **函数原型/签名**：`esp_err_t nvs_flash_erase(void);`
- **参数含义详解**：无参数；擦除会删除该默认 NVS 分区内的持久化数据，调用前必须确认数据可丢失。
- **返回值状态码**：成功返回 `ESP_OK`；NVS 未初始化、分区不存在或底层擦除失败时返回相应 `esp_err_t`。
- **使用 Demo**：

~~~c
esp_err_t err = nvs_flash_erase();
if (err == ESP_OK)
    err = nvs_flash_init();
~~~

## 官方 API 与项目函数的最终区分

以下函数来自项目源码的自定义封装或业务逻辑，**不属于乐鑫官方 API**，本页不再为它们写 API 详解：

- `esp_lsm6dsr_spi_init`
- `lsm6dsr_write_reg`
- `lsm6dsr_read_reg`
- `lsm6dsr_fifo_get_status`
- `lsm6dsr_fifo_dma_init`
- `lsm6dsr_fifo_read_words_dma`
- `lsm6dsr_fifo_reset`
- `lsm6dsr_fifo_config_high_rate`
- `wifi_sta_init`
- `send_all`
- `consume_one_fifo_word`
- `append_sample_to_frame`

以下内容是 LSM6DSR 器件寄存器或协议定义，也不是 ESP-IDF API：

- `WHO_AM_I`、`CTRL3_C`、`CTRL9_XL`
- `FIFO_CTRL1`、`FIFO_CTRL4`、`FIFO_STATUS1`、`FIFO_STATUS2`
- `FIFO_DATA_OUT_TAG`、`LSM6DSR_ID`

## 避坑红线

- **归属红线**：不要把项目封装函数的签名、返回码或 Demo 冒充 ESP-IDF 官方 API；要查官方行为时直接以 `driver/*.h`、`esp_*.h` 和对应版本 API Reference 为准。
- **SPI 事务红线**：`spi_device_polling_transmit` 与 `spi_device_transmit` 使用的是官方 `spi_transaction_t`；SPI 从设备寄存器地址、读写位和 RX 偏移属于 LSM6DSR 协议层，不是 SPI API 自动定义的规则。
- **DMA 内存红线**：使用 SPI DMA 时，事务缓冲区必须满足 ESP-IDF 对 DMA 可访问内存的约束；不要在高频 ISR 中反复进行不可控的堆分配。
- **ISR 红线**：GPIO ISR 中只能调用 ISR 安全 API；`vTaskNotifyGiveFromISR` 的唤醒结果要配合 `portYIELD_FROM_ISR`，不能在 ISR 中调用会阻塞的普通任务 API。
- **Ringbuffer 生命周期红线**：`xRingbufferReceive` 返回的项目必须用 `vRingbufferReturnItem` 归还；不能把该指针长期保存，也不能用 `free` 释放。
- **Wi-Fi 状态红线**：`esp_wifi_start` 只表示驱动启动，`esp_wifi_connect` 只表示提交连接请求；是否联网应通过事件和 IP 获取事件判断。
- **NVS 数据红线**：只有在 `ESP_ERR_NVS_NO_FREE_PAGES` 或 `ESP_ERR_NVS_NEW_VERSION_FOUND` 等明确场景下才擦除 NVS；擦除会丢失持久化配置。
- **错误码红线**：所有返回 `esp_err_t` 的 API 都应检查 `ESP_OK` 或明确处理错误分支；不要把 FreeRTOS 的 `pdTRUE/pdFALSE/pdPASS` 与 `esp_err_t` 混用。
- **版本红线**：ESP-IDF 升级后重新核对结构体字段、枚举值、错误码和弃用状态；本页不能替代当前版本的官方文档。

## 关联笔记

- [[LSM6DSR SPI 驱动]]
- [[LSM6DSR FIFO 采集]]
- [[FIFO TAG 配对]]
- [[ESP32-S3 IMU 启动时序]]

## 来源

- `raw/ESP32S3_IMU_项目完整总结.md`
