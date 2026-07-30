# SR501 Linux 驱动 API 速查

## 适用范围

本页只整理 Linux 内核官方 GPIO、IRQ、等待队列、用户空间拷贝和 Platform 驱动 API。`sr501_probe`、`sr501_isr`、`sr501_read`、`sr501_poll` 等是工程函数或回调占位符，不是 Linux API；示例中的 `my_*` 名称也只表示调用方需要提供的函数。

## 基础头文件

~~~c
#include <linux/gpio/consumer.h>
#include <linux/interrupt.h>
#include <linux/platform_device.h>
#include <linux/uaccess.h>
#include <linux/wait.h>
#include <linux/err.h>
~~~

## 核心 API

### `devm_gpiod_get`

- **作用**：按设备资源描述获取 GPIO 描述符，并由 devres 在设备解绑时自动管理释放。
- **函数原型/签名**：`struct gpio_desc *devm_gpiod_get(struct device *dev, const char *con_id, enum gpiod_flags flags);`
- **参数含义详解**：`dev` 是设备对象；`con_id` 是连接名，可为 `NULL`；`flags` 可用 `GPIOD_IN` 等初始方向标志。
- **返回值状态码**：成功返回 `struct gpio_desc *`；失败返回错误指针，必须使用 `IS_ERR()` 和 `PTR_ERR()` 判断。
- **使用 Demo**：

~~~c
struct gpio_desc *gpio;
gpio = devm_gpiod_get(&pdev->dev, NULL, GPIOD_IN);
if (IS_ERR(gpio))
    return PTR_ERR(gpio);
~~~

### `gpiod_direction_input`

- **作用**：将 GPIO 描述符配置为输入方向。
- **函数原型/签名**：`int gpiod_direction_input(struct gpio_desc *desc);`
- **参数含义详解**：`desc` 是由 GPIO consumer API 获取的描述符；调用上下文和底层 GPIO 控制器能力需符合内核要求。
- **返回值状态码**：成功返回 `0`；失败返回负 errno。
- **使用 Demo**：

~~~c
int ret = gpiod_direction_input(gpio);
if (ret < 0) {
    dev_err(&pdev->dev, "input config failed\n");
    return ret;
}
~~~

### `gpiod_to_irq`

- **作用**：把 GPIO 描述符映射为对应的 Linux IRQ 号。
- **函数原型/签名**：`int gpiod_to_irq(const struct gpio_desc *desc);`
- **参数含义详解**：`desc` 必须来自支持中断映射的 GPIO；返回的 IRQ 号供 `request_irq()` 或线程化 IRQ 注册使用。
- **返回值状态码**：成功返回非负 IRQ 号；失败返回负 errno。
- **使用 Demo**：

~~~c
int irq = gpiod_to_irq(gpio);
if (irq < 0)
    return irq;
dev_info(&pdev->dev, "mapped irq=%d\n", irq);
~~~

### `request_irq`

- **作用**：为指定 IRQ 注册硬中断处理函数。
- **函数原型/签名**：`int request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags, const char *name, void *dev_id);`
- **参数含义详解**：`irq` 是 IRQ 号；`handler` 是调用方 ISR；`flags` 指定触发方式；`name` 是诊断名称；`dev_id` 必须在释放时保持一致。
- **返回值状态码**：成功返回 `0`；失败返回负 errno。
- **使用 Demo**：

~~~c
ret = request_irq(irq, my_irq_handler,
                  IRQF_TRIGGER_RISING, "sr501", dev);
if (ret < 0)
    return ret;
~~~

`my_irq_handler` 是调用方提供的 ISR 占位符，不是 Linux API；真实 SR501 项目函数不得在本页伪装成官方函数。

### `wait_event_interruptible`

- **作用**：当条件不满足时，让当前进程进入可被信号打断的睡眠状态。
- **函数原型/签名**：`int wait_event_interruptible(wait_queue_head_t wq, condition);`
- **参数含义详解**：`wq` 是等待队列头；`condition` 必须可重复检查，并由生产者在唤醒前更新。
- **返回值状态码**：条件满足返回 `0`；被信号打断时返回负值，常见为 `-ERESTARTSYS`。
- **使用 Demo**：

~~~c
ret = wait_event_interruptible(dev->wq, dev->data_ready);
if (ret < 0)
    return ret;
value = dev->data;
~~~

### `wake_up_interruptible`

- **作用**：唤醒等待队列中可被信号打断的进程。
- **函数原型/签名**：`void wake_up_interruptible(wait_queue_head_t *wq);`
- **参数含义详解**：`wq` 指向等待队列头；调用前应先更新条件变量，并按并发模型使用锁或原子变量。
- **返回值状态码**：无返回值。
- **使用 Demo**：

~~~c
dev->data_ready = true;
wake_up_interruptible(&dev->wq);
return IRQ_HANDLED;
~~~

### `copy_to_user`

- **作用**：把内核地址范围内的数据复制到用户空间缓冲区。
- **函数原型/签名**：`unsigned long copy_to_user(void __user *to, const void *from, unsigned long n);`
- **参数含义详解**：`to` 是用户地址；`from` 是内核地址；`n` 是字节数。只能在可睡眠的进程上下文使用。
- **返回值状态码**：返回未成功复制的字节数；非零时通常向用户返回 `-EFAULT`。
- **使用 Demo**：

~~~c
if (copy_to_user(buf, &value, sizeof(value)))
    return -EFAULT;
return sizeof(value);
~~~

### `free_irq`

- **作用**：注销此前通过 `request_irq()` 注册的 IRQ 处理函数。
- **函数原型/签名**：`void free_irq(unsigned int irq, void *dev_id);`
- **参数含义详解**：`irq` 和 `dev_id` 必须与注册时匹配；调用后不得再使用对应的中断上下文资源。
- **返回值状态码**：无返回值。
- **使用 Demo**：

~~~c
free_irq(dev->irq, dev);
dev->irq = -1;
return 0;
~~~

### `platform_driver_register`

- **作用**：向 Platform 总线注册驱动，并根据匹配表触发设备的 `probe`。
- **函数原型/签名**：`int platform_driver_register(struct platform_driver *drv);`
- **参数含义详解**：`drv` 包含 `probe`、`remove`、驱动名称及设备树匹配表；回调由内核调用。
- **返回值状态码**：成功返回 `0`；失败返回负 errno。
- **使用 Demo**：

~~~c
static int __init sr501_init(void)
{
    return platform_driver_register(&sr501_driver);
}
module_init(sr501_init);
~~~

`sr501_init` 和 `sr501_driver` 是工程对象；本条目只说明官方注册函数。

## 避坑红线

- `devm_gpiod_get()` 失败必须用 `IS_ERR/PTR_ERR`，不能把错误指针当 GPIO 描述符。
- `request_irq()` 的返回值必须检查；如果使用 `IRQ_WAKE_THREAD`，必须同时提供线程化处理函数，普通 ISR 不应返回该值。
- 硬中断上下文不能调用 `copy_to_user()`、阻塞等待或可能睡眠的 GPIO 访问；复杂处理应转移到 threaded IRQ、workqueue 或进程上下文。
- `wait_event_interruptible()` 的返回值必须处理；共享条件需要锁、原子变量或事件计数，避免丢事件和重复消费。
- `copy_to_user()` 返回的是未拷贝字节数，不是简单的 `errno`；非零时通常返回 `-EFAULT`。
- `free_irq()` 必须与 `request_irq()` 的 `irq/dev_id` 配对；devm 资源与手动资源不能重复释放。
- Platform `probe` 失败时应按已成功获取的资源逆序回滚；本页不把 `sr501_probe`、`sr501_remove` 等工程回调写成 API。

## 相关页面

- [[Linux GPIO 输入]]
- [[Linux GPIO 中断]]
- [[Linux Platform 驱动]]
- [[Linux 字符设备驱动]]
- [[Linux 等待队列]]
- [[Linux SR501 驱动]]

## 来源

- raw 文件：`Linux驱动SR501代码.md`
