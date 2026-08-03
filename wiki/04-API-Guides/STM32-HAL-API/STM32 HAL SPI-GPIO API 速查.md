# STM32 HAL SPI-GPIO API 速查

## 适用范围

本页只整理新增源资料中使用到的 **STM32F4 HAL 官方 API 与官方弱回调钩子**。`ST7789_Init`、`ST7789_SetWindow`、`lcd_write_cmd`、`lcd_write_data_dma`、`XT2046_ReadAdc`、`XT2046_GetPoint`、`XT2046_Update`、`MX_SPI1_Init` 和 `MX_SPI2_Init` 都是工程封装或业务函数，不属于本页 API。

## 基础头文件

```c
#include "stm32f4xx_hal.h"
#include "stm32f4xx_hal_spi.h"
#include "stm32f4xx_hal_gpio.h"
```

> 具体工程还需要由 CubeMX 生成并提供 `SPI_HandleTypeDef`、GPIO 端口、引脚宏和中断向量配置。本文不把这些工程对象伪装成 HAL API。

## 核心函数详解

### `HAL_Init`

- **作用**：初始化 HAL 库、Flash 接口和 SysTick 基础设施，为后续 HAL 外设初始化提供运行环境。
- **函数原型/签名**：`HAL_StatusTypeDef HAL_Init(void);`
- **参数含义详解**：无参数；应在系统时钟配置和外设初始化之前调用。
- **返回值状态码**：成功返回 `HAL_OK`；失败返回 `HAL_ERROR`。
- **使用 Demo**：

```c
if (HAL_Init() != HAL_OK)
    Error_Handler();
SystemClock_Config();
MX_GPIO_Init();
MX_SPI1_Init();
```

### `HAL_Delay`

- **作用**：基于 HAL 时间基准阻塞指定毫秒数，常用于 ST7789 复位和 `SLPOUT` 后等待。
- **函数原型/签名**：`void HAL_Delay(uint32_t Delay);`
- **参数含义详解**：`Delay` 为毫秒数；时间基准通常由 SysTick 提供，具体精度受系统时钟和 tick 配置影响。
- **返回值状态码**：无返回值；调用过程中可能阻塞当前上下文。
- **使用 Demo**：

```c
HAL_GPIO_WritePin(LCD_RST_GPIO_Port, LCD_RST_Pin, GPIO_PIN_RESET);
HAL_Delay(10);
HAL_GPIO_WritePin(LCD_RST_GPIO_Port, LCD_RST_Pin, GPIO_PIN_SET);
HAL_Delay(120);
```

### `HAL_GPIO_WritePin`

- **作用**：设置指定 GPIO 端口一个或多个引脚的输出电平。
- **函数原型/签名**：`void HAL_GPIO_WritePin(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin, GPIO_PinState PinState);`
- **参数含义详解**：`GPIOx` 是 GPIO 端口；`GPIO_Pin` 是一个或多个引脚掩码；`PinState` 使用 `GPIO_PIN_SET` 或 `GPIO_PIN_RESET`。
- **返回值状态码**：无返回值；端口和引脚必须已完成 GPIO 输出配置。
- **使用 Demo**：

```c
HAL_GPIO_WritePin(LCD_CS_GPIO_Port, LCD_CS_Pin, GPIO_PIN_RESET);
HAL_GPIO_WritePin(LCD_DC_GPIO_Port, LCD_DC_Pin, GPIO_PIN_SET);
HAL_SPI_Transmit(&hspi1, &cmd, 1, 100);
HAL_GPIO_WritePin(LCD_CS_GPIO_Port, LCD_CS_Pin, GPIO_PIN_SET);
```

### `HAL_SPI_Transmit`

- **作用**：以阻塞方式通过 SPI 发送指定长度的数据。
- **函数原型/签名**：`HAL_StatusTypeDef HAL_SPI_Transmit(SPI_HandleTypeDef *hspi, const uint8_t *pData, uint16_t Size, uint32_t Timeout);`
- **参数含义详解**：`hspi` 是已初始化的 SPI 句柄；`pData` 指向待发送缓冲区；`Size` 是数据单元数量；`Timeout` 是毫秒超时值。
- **返回值状态码**：成功返回 `HAL_OK`；超时返回 `HAL_TIMEOUT`；参数或状态错误返回 `HAL_ERROR`；忙状态可能返回 `HAL_BUSY`。
- **使用 Demo**：

```c
uint8_t cmd = 0x2C;
HAL_GPIO_WritePin(LCD_DC_GPIO_Port, LCD_DC_Pin, GPIO_PIN_RESET);
HAL_GPIO_WritePin(LCD_CS_GPIO_Port, LCD_CS_Pin, GPIO_PIN_RESET);
HAL_StatusTypeDef st = HAL_SPI_Transmit(&hspi1, &cmd, 1, 100);
HAL_GPIO_WritePin(LCD_CS_GPIO_Port, LCD_CS_Pin, GPIO_PIN_SET);
```

### `HAL_SPI_TransmitReceive`

- **作用**：以阻塞方式同时发送和接收 SPI 数据，适合 XPT2046 的命令与 ADC 数据收发。
- **函数原型/签名**：`HAL_StatusTypeDef HAL_SPI_TransmitReceive(SPI_HandleTypeDef *hspi, const uint8_t *pTxData, uint8_t *pRxData, uint16_t Size, uint32_t Timeout);`
- **参数含义详解**：`hspi` 是 SPI 句柄；`pTxData` 和 `pRxData` 分别为发送与接收缓冲区；`Size` 是传输数据单元数；`Timeout` 是毫秒超时值。
- **返回值状态码**：成功返回 `HAL_OK`；超时返回 `HAL_TIMEOUT`；错误返回 `HAL_ERROR`；忙状态可能返回 `HAL_BUSY`。
- **使用 Demo**：

```c
uint8_t tx = 0xD0, rx = 0;
HAL_GPIO_WritePin(TOUCH_CS_GPIO_Port, TOUCH_CS_Pin, GPIO_PIN_RESET);
HAL_StatusTypeDef st = HAL_SPI_TransmitReceive(&hspi2, &tx, &rx, 1, 100);
HAL_GPIO_WritePin(TOUCH_CS_GPIO_Port, TOUCH_CS_Pin, GPIO_PIN_SET);
if (st != HAL_OK) Error_Handler();
```

### `HAL_SPI_Transmit_DMA`

- **作用**：启动 SPI DMA 发送并立即返回，适合 ST7789 的连续像素流。
- **函数原型/签名**：`HAL_StatusTypeDef HAL_SPI_Transmit_DMA(SPI_HandleTypeDef *hspi, const uint8_t *pData, uint16_t Size);`
- **参数含义详解**：`hspi` 是 SPI 句柄；`pData` 指向 DMA 读取的发送缓冲区；`Size` 是数据单元数量，HAL 常用 16 bit 计数；传输完成后由 HAL 调用完成回调。
- **返回值状态码**：启动成功返回 `HAL_OK`；SPI 忙返回 `HAL_BUSY`；参数/状态错误返回 `HAL_ERROR`。
- **使用 Demo**：

```c
HAL_GPIO_WritePin(LCD_DC_GPIO_Port, LCD_DC_Pin, GPIO_PIN_SET);
HAL_GPIO_WritePin(LCD_CS_GPIO_Port, LCD_CS_Pin, GPIO_PIN_RESET);
HAL_StatusTypeDef st = HAL_SPI_Transmit_DMA(&hspi1, pixel_buf, 60000);
if (st != HAL_OK)
    HAL_GPIO_WritePin(LCD_CS_GPIO_Port, LCD_CS_Pin, GPIO_PIN_SET);
```

### `HAL_SPI_TxCpltCallback`

- **作用**：SPI 发送完成时由 HAL 调用的弱回调钩子，工程可在其中结束 CS 周期并释放 DMA 忙标志。
- **函数原型/签名**：`void HAL_SPI_TxCpltCallback(SPI_HandleTypeDef *hspi);`
- **参数含义详解**：`hspi` 指向完成传输的 SPI 句柄；若多个 SPI 共用回调，应通过 `hspi->Instance` 区分来源。
- **返回值状态码**：无返回值；回调中的异常处理需由工程自行设计。
- **使用 Demo**：

```c
void HAL_SPI_TxCpltCallback(SPI_HandleTypeDef *hspi)
{
    if (hspi->Instance != SPI1) return;
    HAL_GPIO_WritePin(LCD_CS_GPIO_Port, LCD_CS_Pin, GPIO_PIN_SET);
    lcd_dma_busy = 0;
}
```

### `HAL_GPIO_EXTI_IRQHandler`

- **作用**：处理指定 GPIO EXTI 线的中断标志，清除标志并调用 GPIO EXTI 回调。
- **函数原型/签名**：`void HAL_GPIO_EXTI_IRQHandler(uint16_t GPIO_Pin);`
- **参数含义详解**：`GPIO_Pin` 是触发中断的引脚掩码，应与中断向量和 CubeMX 配置一致。
- **返回值状态码**：无返回值；中断错误状态由 HAL/工程状态处理逻辑承担。
- **使用 Demo**：

```c
void EXTI15_10_IRQHandler(void)
{
    HAL_GPIO_EXTI_IRQHandler(TOUCH_IRQ_Pin);
}
```

### `HAL_GPIO_EXTI_Callback`

- **作用**：GPIO EXTI 中断处理完成后由 HAL 调用的弱回调钩子，适合把硬件事件转换为软件标志。
- **函数原型/签名**：`void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin);`
- **参数含义详解**：`GPIO_Pin` 是触发 EXTI 的引脚；回调运行在中断上下文，不应执行阻塞 SPI、排序或坐标映射。
- **返回值状态码**：无返回值；业务状态应通过 `volatile` 标志、事件或队列传递给非中断上下文。
- **使用 Demo**：

```c
volatile uint8_t touch_pending;
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == TOUCH_IRQ_Pin)
        touch_pending = 1;
}
```

## 避坑红线

- `HAL_SPI_Transmit_DMA` 返回 `HAL_OK` 只代表 DMA 启动成功；发送缓冲区在完成回调前不得释放、覆盖或离开作用域。
- 阻塞式 `HAL_SPI_Transmit` 和 `HAL_SPI_TransmitReceive` 不应在中断上下文中调用；XPT2046 的 ADC 读取应由主循环或线程上下文执行。
- `HAL_StatusTypeDef` 必须检查 `HAL_OK`、`HAL_BUSY`、`HAL_TIMEOUT` 和 `HAL_ERROR`，不能只判断非零。
- SPI 的 `Size` 是数据单元数量，不一定等于字节数；本工程配置为 8 bit，因此 60000 表示 60000 字节。
- `HAL_GPIO_EXTI_Callback` 中只做快速、不可阻塞的状态更新；复杂处理放到主循环、任务或工作队列。
- `HAL_Delay` 依赖 HAL 时间基准；系统 tick 未初始化或在不允许阻塞的上下文调用会导致时序失效。
- LCD 的 CS/DC/RST 和触摸 CS/IRQ 必须分别控制，不能混用端口、引脚或 SPI 句柄。

## 相关页面

- [[ST7789 初始化时序]]
- [[ST7789 窗口机制]]
- [[ST7789 DMA 刷屏]]
- [[XPT2046 触摸采样与坐标校准]]
- [[XPT2046 EXTI 触摸中断]]
- [[STM32 SPI 双总线显示触摸架构]]
- [[ST7789-XPT2046-知识地图]]

## 来源

- raw 文件：`st7789+XPT2046驱动说明文档.md`