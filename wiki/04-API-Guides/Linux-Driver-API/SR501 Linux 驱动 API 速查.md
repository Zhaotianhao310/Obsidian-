# SR501 Linux 驱动 API 速查

## 适用范围

本页整理 raw 中 SR501 Linux 驱动使用到的 GPIO、IRQ、字符设备、等待队列和 platform_driver API。函数签名应以目标内核版本头文件为准；raw 文件包含多段递进式示例，不能直接视为一份可编译的最终驱动。

## 基础头文件

~~~c
#include <linux/gpio/consumer.h>
#include <linux/interrupt.h>
#include <linux/fs.h>
#include <linux/platform_device.h>
#include <linux/wait.h>
~~~

常见附加依赖包括 linux/module.h、linux/of.h、linux/device.h、linux/poll.h、linux/uaccess.h 和 linux/errno.h。

## GPIO API

### gpiod_get

- 作用：从设备上下文获取 GPIO 描述符。
- 签名：struct gpio_desc *gpiod_get(struct device *dev, const char *con_id, enum gpiod_flags flags)
- 参数：dev 为设备对象；con_id 为连接标识；flags 指定初始方向或默认电平。
- 返回值：成功返回 gpio_desc；失败返回 ERR_PTR，使用 IS_ERR/PTR_ERR 判断。

~~~c
struct gpio_desc *gpio;
gpio = gpiod_get(&pdev->dev, NULL, GPIOD_IN);
if (IS_ERR(gpio)) { return PTR_ERR(gpio); }
~~~

### gpiod_direction_input

- 作用：把 GPIO 配置为输入。
- 签名：int gpiod_direction_input(struct gpio_desc *desc)
- 参数：desc 为 GPIO 描述符。
- 返回值：成功为 0；失败返回负 errno。

~~~c
int ret;
ret = gpiod_direction_input(sr501_gpio);
if (ret) { gpiod_put(sr501_gpio); }
~~~

### gpiod_get_value

- 作用：读取 GPIO 当前逻辑电平。
- 签名：int gpiod_get_value(const struct gpio_desc *desc)
- 参数：desc 为 GPIO 描述符。
- 返回值：通常返回 0 或 1；具体错误与上下文应按目标内核版本确认。

~~~c
int level;
level = gpiod_get_value(sr501_gpio);
sr501_data = level;
~~~

### gpiod_put

- 作用：释放通过 gpiod_get 获取的 GPIO 描述符。
- 签名：void gpiod_put(struct gpio_desc *desc)
- 参数：desc 为已获取的 GPIO 描述符。
- 返回值：无返回值。

~~~c
if (sr501_gpio != NULL) {
    gpiod_put(sr501_gpio);
    sr501_gpio = NULL;
}
~~~

### gpiod_to_irq

- 作用：把 GPIO 描述符转换为对应 IRQ 号。
- 签名：int gpiod_to_irq(const struct gpio_desc *desc)
- 参数：desc 为 GPIO 描述符。
- 返回值：成功返回非负 IRQ 号；失败返回负 errno。

~~~c
int irq;
irq = gpiod_to_irq(sr501_gpio);
if (irq < 0) { return irq; }
~~~

## IRQ API

### request_irq

- 作用：注册中断处理函数。
- 签名：int request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags, const char *name, void *dev)
- 参数：irq 为中断号；handler 为 ISR；flags 指定触发方式；name 为标识；dev 为设备标识。
- 返回值：成功为 0；失败返回负 errno。

~~~c
ret = request_irq(irq, sr501_isr,
                  IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING,
                  "sr501", dev);
~~~

### free_irq

- 作用：释放已注册的 IRQ。
- 签名：void free_irq(unsigned int irq, void *dev_id)
- 参数：irq 为中断号；dev_id 必须与注册时的设备标识匹配。
- 返回值：无返回值。

~~~c
if (irq >= 0) {
    free_irq(irq, dev);
    irq = -1;
}
~~~

## 用户空间数据传递

### copy_to_user

- 作用：把内核缓冲区数据复制到用户空间。
- 签名：unsigned long copy_to_user(void __user *to, const void *from, unsigned long n)
- 参数：to 为用户地址；from 为内核地址；n 为复制字节数。
- 返回值：成功返回 0；非零表示尚有字节未复制，驱动通常返回 -EFAULT。

~~~c
if (copy_to_user(buf, &sr501_data, sizeof(sr501_data))) {
    return -EFAULT;
}
return sizeof(sr501_data);
~~~

## 等待队列 API

### init_waitqueue_head

- 作用：初始化等待队列头。
- 签名：void init_waitqueue_head(wait_queue_head_t *wq_head)
- 参数：wq_head 为等待队列头地址。
- 返回值：无返回值。

~~~c
static wait_queue_head_t sr501_wq;
init_waitqueue_head(&sr501_wq);
sr501_data = 0;
~~~

### wait_event_interruptible

- 作用：条件不满足时让当前进程进入可被信号打断的睡眠。
- 签名：wait_event_interruptible(wq_head, condition)
- 参数：wq_head 为等待队列；condition 为需反复检查的条件。
- 返回值：条件满足时通常为 0；被信号打断时返回负错误码。

~~~c
int ret;
ret = wait_event_interruptible(sr501_wq, sr501_data != 0);
if (ret) { return ret; }
~~~

### wake_up

- 作用：唤醒等待队列中的任务。
- 签名：void wake_up(wait_queue_head_t *wq_head)
- 参数：wq_head 为需要唤醒的等待队列。
- 返回值：无返回值。

~~~c
sr501_data = gpiod_get_value(sr501_gpio);
wake_up(&sr501_wq);
return IRQ_HANDLED;
~~~

### poll_wait

- 作用：把当前进程加入等待队列，使 poll/select/epoll 能在事件到达时被唤醒。
- 签名：void poll_wait(struct file *filp, wait_queue_head_t *wait_address, poll_table *p)
- 参数：filp 为文件对象；wait_address 为等待队列；p 为 poll_table。
- 返回值：无返回值。

~~~c
static __poll_t sr501_poll(struct file *file, poll_table *wait) {
    poll_wait(file, &sr501_wq, wait);
    return sr501_data ? POLLIN | POLLRDNORM : 0;
}
~~~

## 字符设备 API

### register_chrdev

- 作用：注册传统字符设备并获得主设备号。
- 签名：int register_chrdev(unsigned int major, const char *name, const struct file_operations *fops)
- 参数：major 为主设备号，传 0 可请求动态主设备号；name 为设备名；fops 为操作表。
- 返回值：成功返回主设备号；失败返回负 errno。

~~~c
major = register_chrdev(0, "sr501", &sr501_fops);
if (major < 0) { return major; }
return 0;
~~~

### unregister_chrdev

- 作用：注销由 register_chrdev 注册的字符设备。
- 签名：void unregister_chrdev(unsigned int major, const char *name)
- 参数：major 和 name 必须与注册时对应。
- 返回值：无返回值。

~~~c
if (major >= 0) {
    unregister_chrdev(major, "sr501");
    major = -1;
}
~~~

### class_create

- 作用：创建设备类，供设备模型和 udev 使用。
- 签名：struct class *class_create(struct module *owner, const char *name)
- 参数：owner 通常为 THIS_MODULE；name 为类名。不同内核版本的签名可能变化。
- 返回值：成功返回 class；失败返回 ERR_PTR。

~~~c
sr501_class = class_create(THIS_MODULE, "sr501_class");
if (IS_ERR(sr501_class)) {
    return PTR_ERR(sr501_class);
}
~~~

### class_destroy

- 作用：销毁设备类。
- 签名：void class_destroy(struct class *cls)
- 参数：cls 为 class_create 返回的对象。
- 返回值：无返回值。

~~~c
if (!IS_ERR_OR_NULL(sr501_class)) {
    class_destroy(sr501_class);
    sr501_class = NULL;
}
~~~

### device_create

- 作用：在设备类下创建设备对象，通常触发用户空间生成设备节点。
- 签名：struct device *device_create(struct class *class, struct device *parent, dev_t devt, void *drvdata, const char *fmt, ...)
- 参数：class 为设备类；parent 为父设备；devt 为设备号；drvdata 为私有数据；fmt 为设备名格式。
- 返回值：成功返回 device；失败返回 ERR_PTR。

~~~c
dev = device_create(sr501_class, NULL, devno, NULL, "sr501");
if (IS_ERR(dev)) { class_destroy(sr501_class); }
return 0;
~~~

### device_destroy

- 作用：删除设备类下指定设备号对应的设备对象。
- 签名：void device_destroy(struct class *class, dev_t devt)
- 参数：class 为设备类；devt 为设备号。
- 返回值：无返回值。

~~~c
if (sr501_class != NULL) {
    device_destroy(sr501_class, devno);
    sr501_class = NULL;
}
~~~

## Platform 驱动 API

### platform_driver_register

- 作用：向 platform 总线注册 platform_driver，使设备树匹配和 probe 机制生效。
- 签名：int platform_driver_register(struct platform_driver *drv)
- 参数：drv 为 platform_driver 结构体，至少包含 probe、remove 和 driver 信息。
- 返回值：成功为 0；失败返回负 errno。

~~~c
static int __init sr501_init(void) {
    return platform_driver_register(&sr501_driver);
}
module_init(sr501_init);
~~~

### platform_driver_unregister

- 作用：注销 platform_driver，并触发已绑定设备的 remove。
- 签名：void platform_driver_unregister(struct platform_driver *drv)
- 参数：drv 为已注册的 platform_driver。
- 返回值：无返回值。

~~~c
static void __exit sr501_exit(void) {
    platform_driver_unregister(&sr501_driver);
}
module_exit(sr501_exit);
~~~

## file_operations

### 结构体作用

file_operations 不是普通函数，而是用户空间文件操作到驱动回调的映射表。raw 示例至少使用 .read 和 .poll；完整实现还需按目标内核版本填写 owner、open、release 等字段。

~~~c
static const struct file_operations sr501_fops = {
    .owner = THIS_MODULE,
    .read  = sr501_read,
    .poll  = sr501_poll,
};
~~~

## 避坑红线

- raw 包含多个递进版本，函数签名和清理路径不能机械拼接。
- request_irq 成功后必须在 remove 和失败回滚路径调用 free_irq。
- gpiod_get 成功后必须调用 gpiod_put；device_create 成功后要对应 device_destroy。
- copy_to_user 返回非零时应处理 -EFAULT，不能当作成功。
- wait_event_interruptible 的返回值可能表示信号打断，不能无条件继续读数据。
- 中断处理函数不能执行会睡眠的操作；复杂逻辑应转移到线程化中断、工作队列或内核线程。
- ISR 与 read 共享的状态变量需要明确的同步策略，单个标志位可能覆盖连续事件。
- raw 中的 kernel_thread、IRQ_WAKE_THREAD 等写法依赖具体实现上下文；使用前必须核对目标内核版本和中断注册方式。
- class_create 等 API 在不同 Linux 内核版本中可能有签名差异，以目标内核头文件为准。

## 相关页面

- [[Linux SR501 驱动]]
- [[Linux 字符设备驱动]]
- [[Linux GPIO 输入]]
- [[Linux GPIO 中断]]
- [[Linux 等待队列]]
- [[Linux Platform 驱动]]
- [[Linux-驱动知识地图]]

## 来源

- raw 文件：Linux驱动SR501代码.md
