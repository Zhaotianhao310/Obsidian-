# Linux GPIO 输入

## 概念一句话精炼

Linux GPIO 输入接口把设备树或板级描述中的 GPIO 映射为内核可读的输入信号，是 [[Linux SR501 驱动]]获取人体感应状态的硬件入口。

## 核心原理与图解

```mermaid
flowchart LR
    A[设备树 GPIO 描述] --> B[gpiod_get]
    B --> C[gpiod_direction_input]
    C --> D[gpiod_get_value]
    D --> E[驱动状态或中断判断]
```

GPIO 输入只解决“如何读取电平”，不自动解决去抖、边沿触发、并发保护和用户态通知。需要事件通知时，应结合 [[Linux GPIO 中断]]与 [[Linux 等待队列]]。

## 关键实现与数据结构

```c
struct gpio_desc *gpio;

gpio = devm_gpiod_get(&pdev->dev, NULL, GPIOD_IN);
if (IS_ERR(gpio)) return PTR_ERR(gpio);
if (gpiod_direction_input(gpio)) return -EINVAL;
int level = gpiod_get_value_cansleep(gpio); // 允许可睡眠 GPIO
```

## 横向对比与关联

- `gpiod_get` / `devm_gpiod_get`：获取 GPIO 描述符；后者便于设备解绑时自动释放。
- `gpiod_get_value`：适用于不可睡眠路径；`gpiod_get_value_cansleep`：适用于可能睡眠的 GPIO 控制器。
- GPIO 输入轮询简单但浪费 CPU；中断 + 等待队列更适合 SR501 这类状态变化通知。

- [[Linux GPIO 中断]]
- [[Linux 等待队列]]
- [[Linux Platform 驱动]]

## 来源

- raw 文件：`Linux驱动SR501代码.md`