# SR501 Linux 驱动 API 速查

## 适用范围

本页整理 `Linux驱动SR501代码.md` 中涉及的 GPIO、IRQ、等待队列、用户空间拷贝、字符设备和 Platform 驱动 API。raw 包含多个递进式代码片段，以下内容是 API 速查，不代表一份可直接编译的最终驱动。

## 基础头文件

```c
#include <linux/gpio/consumer.h>
#include <linux/interrupt.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/wait.h>
#include <linux/platform_device.h>
#include <linux/device.h>
```

## 核心 API

### `devm_gpiod_get`

- **作用**：按设备资源描述获取 GPIO 描述符，并由 devres 在设备解绑时管理释放。
- **签名**：`struct gpio_desc *devm_gpiod_get(struct device *dev, const char *con_id, enum gpiod_flags flags)`
- **参数**：`dev` 为设备对象；`con_id` 为 GPIO 连接名，可能为 `NULL`；`flags` 可用 `GPIOD_IN` 等方向标志。
- **返回值**：成功返回描述符；失败返回 `ERR_PTR`，用 `IS_ERR/PTR_ERR` 判断。

```c
struct gpio_desc *gpio;
gpio = devm_gpiod_get(&pdev->dev, NULL, GPIOD_IN);
if (IS_ERR(gpio)) return PTR_ERR(gpio);
```

### `gpiod_direction_input`

- **作用**：把 GPIO 配置为输入方向。
- **签名**：`int gpiod_direction_input(struct gpio_desc *desc)`
- **参数**：`desc` 为 GPIO 描述符。
- **返回值**：成功为 0；失败返回负 errno。

```c
int ret = gpiod_direction_input(gpio);
if (ret) {
    dev_err(&pdev->dev, "gpio input failed\n");
    return ret;
}
```

### `gpiod_to_irq`

- **作用**：把 GPIO 描述符转换为对应 IRQ 号。
- **签名**：`int gpiod_to_irq(const struct gpio_desc *desc)`
- **参数**：`desc` 为支持中断映射的 GPIO。
- **返回值**：成功返回非负 IRQ；失败返回负 errno。

```c
int irq = gpiod_to_irq(gpio);
if (irq < 0) return irq;
ret = request_irq(irq, sr501_isr, IRQF_TRIGGER_RISING, "sr501", dev);
```

### `request_irq`

- **作用**：注册硬中断处理函数。
- **签名**：`int request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags, const char *name, void *dev)`
- **参数**：`irq` 为中断号；`handler` 为 ISR；`flags` 为触发方式；`name` 为标识；`dev` 为回调上下文。
- **返回值**：成功为 0；失败返回负 errno。

```c
ret = request_irq(irq, sr501_isr,
                  IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING,
                  "sr501", dev);
if (ret) return ret;
```

### `wait_event_interruptible`

- **作用**：条件不满足时让当前任务可被信号打断地睡眠。
- **签名**：`int wait_event_interruptible(wait_queue_head_t wq, condition)`
- **参数**：`wq` 为等待队列头；`condition` 为可重复检查的条件表达式。
- **返回值**：0 表示条件满足；被信号打断时返回 `-ERESTARTSYS` 等负值。

```c
ret = wait_event_interruptible(dev->wq, dev->data != 0);
if (ret) return ret;
value = dev->data;
dev->data = 0;
```

### `wake_up_interruptible`

- **作用**：唤醒等待队列中可被信号打断的任务。
- **签名**：`void wake_up_interruptible(wait_queue_head_t *wq)`
- **参数**：`wq` 为等待队列头。
- **返回值**：无返回值。

```c
dev->data = 1;                    /* 先更新条件 */
wake_up_interruptible(&dev->wq);  /* 再唤醒等待者 */
return IRQ_HANDLED;
```

### `copy_to_user`

- **作用**：把内核缓冲区数据复制到用户空间。
- **签名**：`unsigned long copy_to_user(void __user *to, const void *from, unsigned long n)`
- **参数**：`to` 为用户地址；`from` 为内核地址；`n` 为字节数。
- **返回值**：返回未成功复制的字节数；非 0 时通常返回 `-EFAULT`。

```c
if (copy_to_user(buf, &value, sizeof(value)))
    return -EFAULT;
return sizeof(value);
```

### `free_irq`

- **作用**：注销已注册的 IRQ 处理函数。
- **签名**：`void free_irq(unsigned int irq, void *dev_id)`
- **参数**：必须使用注册时对应的 `irq` 和 `dev_id`。
- **返回值**：无返回值。

```c
free_irq(dev->irq, dev);
dev->irq = -1;
```

### `platform_driver_register`

- **作用**：向 Platform 总线注册驱动并触发设备匹配。
- **签名**：`int platform_driver_register(struct platform_driver *drv)`
- **参数**：`drv` 包含 `probe/remove` 和匹配信息。
- **返回值**：成功为 0；失败返回负 errno。

```c
static int __init sr501_init(void)
{
    return platform_driver_register(&sr501_driver);
}
```

## 避坑红线

- `IRQ_WAKE_THREAD` 必须配套线程化 IRQ handler；只有普通 ISR 时应返回 `IRQ_HANDLED` 或合适结果。
- `copy_to_user` 只能在进程上下文调用，不能在硬中断上下文使用。
- `wait_event_interruptible` 的条件需要与 ISR 更新的状态保持同步，必要时使用锁或原子变量。
- `request_irq`、字符设备、class/device 和 GPIO 的释放必须与 `probe` 成功路径对应。
- `gpiod_get` 与 `devm_gpiod_get` 的生命周期不同，不能混用释放方式。
- 函数原型和可用标志以目标 Linux 内核版本头文件为准。

## 相关页面

- [[Linux SR501 驱动]]
- [[Linux GPIO 输入]]
- [[Linux GPIO 中断]]
- [[Linux 等待队列]]
- [[Linux 字符设备驱动]]
- [[Linux Platform 驱动]]

## 来源

- raw 文件：`Linux驱动SR501代码.md`