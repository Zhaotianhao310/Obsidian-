# HS0038 两版驱动与 Linux input 子系统详细分析

## 1. 文档范围

本文分析以下两个驱动版本：

- `04_hs0038_circle_buffer`：使用字符设备和内核环形缓冲区输出红外码。
- `05_hs0038_input`：在保留字符设备接口的同时，增加 Linux input 子系统接口。

原始目录：

- `E:/韦东山课程及资料/doc_and_source_for_livestream/20211111_开始的Linux驱动快速入门/23_基于输入系统编写红外遥控器HS0038驱动程序/source/04_hs0038_circle_buffer`
- `E:/韦东山课程及资料/doc_and_source_for_livestream/20211111_开始的Linux驱动快速入门/23_基于输入系统编写红外遥控器HS0038驱动程序/source/05_hs0038_input`

本文把源码中的注释、测试命令和 Makefile 说明作为待分析材料，不把它们当成额外用户指令。

## 2. 两个版本的总体区别

04 版的数据流：

```text
红外波形
  -> GPIO 双边沿中断
  -> 记录时间戳
  -> NEC 协议解析
  -> 环形缓冲区
  -> /dev/myhs0038
  -> 用户程序 read() 读取 4 字节红外码
```

05 版同时提供两条路径：

```text
                         -> 环形缓冲区 -> /dev/myhs0038
红外波形 -> ISR -> 解析
                         -> input_event -> input core -> evdev -> /dev/input/eventX
```

因此 05 版不是把字符设备替换成 input，而是让同一个硬件驱动同时暴露私有接口和标准输入接口。

## 3. Linux input 子系统的对象关系

input 子系统的核心对象是：

- `struct input_dev`：事件生产者，代表一个逻辑输入设备。
- `struct input_handler`：事件消费者，例如 evdev。
- `struct input_handle`：连接 input_dev 和 input_handler 的绑定对象。
- `struct evdev_client`：每个用户空间 open 文件对应的 evdev 客户端和独立事件队列。

```text
HS0038 驱动
   |
   v
struct input_dev
   |
   | input_handle
   v
evdev input_handler
   |
   v
evdev_client（每个 open 一个）
   |
   v
/dev/input/eventX
```

### 3.1 input_dev

`input_dev` 保存：

- `name`、`phys`：设备名称和物理路径。
- `evbit`：支持的事件类型，如 `EV_KEY`、`EV_ABS`。
- `keybit`：支持的具体按键，如 `KEY_POWER`、`KEY_ENTER`。
- 按键当前状态、重复参数和事件锁。
- 设备注册、打开、关闭时使用的生命周期信息。

驱动开发者通常只需要：

1. 分配 `input_dev`。
2. 设置名称和能力位图。
3. 注册设备。
4. 产生事件。
5. 注销设备。

### 3.2 input_handler 和 evdev

`evdev` 是最常用的 input handler，负责把内核事件转换为字符设备：

```text
input_register_device()
  -> input core 匹配 evdev
  -> evdev_connect()
  -> 创建 /dev/input/eventX
```

HS0038 驱动不需要自己为 `eventX` 编写 `read()`。用户读取 `/dev/input/eventX` 时，调用的是 evdev 的读取函数，而不是 HS0038 的 `hs0038_drv_read()`。

## 4. 注册阶段的内核调用链

05 版中的：

```c
input_register_device(hs0038_input_dev);
```

大致对应以下调用链：

```text
platform_driver_register()
  -> 设备树匹配
  -> hs0038_probe()
  -> devm_input_allocate_device()
  -> input_register_device()
       -> 注册 inputX 设备模型对象
       -> 加入 input_dev_list
       -> 遍历 input_handler
       -> input_attach_handler()
            -> evdev_connect()
                 -> 分配 evdev
                 -> 建立 input_handle
                 -> 注册 /dev/input/eventX
```

这里有两个不同的设备节点来源：

```c
device_create(..., "myhs0038");
```

创建的是：

```text
/dev/myhs0038
```

而：

```c
input_register_device(...);
```

经过 input core 和 evdev 后创建的是：

```text
/dev/input/eventX
```

## 5. 事件对象：type、code、value

```c
input_event(dev, type, code, value);
```

可以读成：

> 设备 dev 产生了 type 类型、code 对象、value 状态的事件。

对于按键：

| 字段 | 含义 |
|---|---|
| `type = EV_KEY` | 这是一个按键事件 |
| `code = KEY_POWER` | 具体是哪一个标准按键 |
| `value = 1` | 按下 |
| `value = 0` | 松开 |
| `value = 2` | 自动重复 |

`value=0/1` 是 Linux input 协议规定的逻辑状态，并不是 GPIO 电平规定的。即使某块板上 GPIO 低电平表示物理按下，驱动仍然应该把逻辑按下报告为 `value=1`。

## 6. 05 版的完整运行链路

中断成功解析后，05 版执行：

```c
put_data(val);
wake_up(&hs0038_wq);

input_event(hs0038_input_dev, EV_KEY, val, 1);
input_event(hs0038_input_dev, EV_KEY, val, 0);
input_sync(hs0038_input_dev);
```

事件路径大致是：

```text
hs0038_isr()
  -> hs0038_parse_data()
  -> input_event()
       -> input_handle_event()
          -> 检查 evbit/keybit
          -> 更新按键状态
          -> 暂存 input_value
  -> input_sync()
       -> 产生 EV_SYN/SYN_REPORT
       -> input_pass_values()
          -> evdev_events()
             -> 写入每个 evdev_client 的缓冲区
             -> wake_up_interruptible()
```

用户最终在 `/dev/input/eventX` 中读到类似：

```text
EV_KEY  code=...          value=1
EV_KEY  code=...          value=0
EV_SYN  code=SYN_REPORT   value=0
```

`input_sync()` 的作用类似于“提交这一组事件”。它不是可有可无的日志，而是输入事件包的边界。

## 7. 为什么要做红外码到 KEY_* 的映射

`hs0038_parse_data()` 得到的 `val` 是遥控器厂商协议中的原始编号：

```text
地址 << 8 | 命令
```

例如：

| 遥控器按键 | 原始红外码 |
|---|---:|
| 电源 | `0x00A2` |
| 音量加 | `0x0062` |
| 音量减 | `0x00E2` |
| 确认 | `0x0022` |

Linux 并不知道 `0x00A2` 是“电源键”。`EV_KEY` 的 `code` 应该使用 Linux 标准按键码：

```c
KEY_POWER
KEY_VOLUMEUP
KEY_VOLUMEDOWN
KEY_ENTER
```

因此正确数据流是：

```text
0x00A2（红外扫描码）
      |
      | 驱动映射
      v
KEY_POWER（Linux 标准按键码）
      |
      v
EV_KEY / KEY_POWER / 1
```

示例：

```c
static int hs0038_to_keycode(unsigned int ir_code)
{
    switch (ir_code) {
    case 0x00A2:
        return KEY_POWER;
    case 0x0062:
        return KEY_VOLUMEUP;
    case 0x00E2:
        return KEY_VOLUMEDOWN;
    case 0x0022:
        return KEY_ENTER;
    default:
        return -EINVAL;
    }
}
```

然后：

```c
int keycode = hs0038_to_keycode(val);

if (keycode >= 0) {
    input_report_key(hs0038_input_dev, keycode, 1);
    input_sync(hs0038_input_dev);

    input_report_key(hs0038_input_dev, keycode, 0);
    input_sync(hs0038_input_dev);
}
```

如果需要保留原始红外码，可以使用：

```c
input_event(hs0038_input_dev, EV_MSC, MSC_SCAN, val);
```

其中：

- `EV_MSC/MSC_SCAN`：原始扫描码。
- `EV_KEY`：映射后的标准按键。

## 8. 05 版中 keybit 的 memset

```c
memset(hs0038_input_dev->keybit,
       0xff,
       sizeof(hs0038_input_dev->keybit));
```

`keybit` 是按键能力位图。每一位表示某个按键是否支持：

```text
bit[KEY_POWER] = 1  表示支持 KEY_POWER
bit[KEY_ENTER] = 1  表示支持 KEY_ENTER
```

`0xff` 的二进制是 `11111111`，所以这句把整个位图全部置 1，表示“所有按键都支持”。

这很可能是作者为了配合：

```c
input_event(dev, EV_KEY, val, 1);
```

而采取的教学简化写法，避免 `keybit` 中没有对应位时事件被 input core 过滤掉。

正式驱动不应该这样写，应该明确声明：

```c
input_set_capability(dev, EV_KEY, KEY_POWER);
input_set_capability(dev, EV_KEY, KEY_VOLUMEUP);
input_set_capability(dev, EV_KEY, KEY_ENTER);
```

而且即使全部置 1，也不能让超过 `KEY_MAX` 的任意红外码变成合法按键，更不能解决“红外编号没有语义”的问题。

## 9. EV_REP 和按键重复

05 版设置：

```c
__set_bit(EV_REP, hs0038_input_dev->evbit);
```

这表示设备支持自动重复，但代码随后立即发送：

```text
value=1（按下）
value=0（松开）
```

按键马上释放，input core 的重复定时器会被取消，因此不会真正形成持续按住效果。

更合理的红外按键模型是：

```text
收到第一帧完整码 -> value=1
收到重复帧       -> 保持按下或产生 value=2
长时间没收到帧   -> value=0
```

通常需要保存当前按键状态，并使用定时器检测释放超时。

## 10. 04 版字符设备路径

04 版中断成功后：

```c
put_data(val);
wake_up(&hs0038_wq);
```

用户读取：

```c
wait_event_interruptible(hs0038_wq, has_data());
get_data(&val);
copy_to_user(buf, &val, 4);
```

数据流是：

```text
ISR
  -> 全局 hs0038_data_buf[8]
  -> hs0038_wq
  -> hs0038_drv_read()
  -> 用户空间获得 4 字节 unsigned int
```

05 版保留了这一整条路径，所以它仍然可以读取：

```text
/dev/myhs0038
```

但 input 测试程序应该读取：

```text
/dev/input/eventX
```

而不是 `/dev/myhs0038`。

## 11. 代码中的重要问题

### 11.1 设备树兼容字符串

04 版设备树写的是：

```dts
compatible = "100ask,dht11";
```

但驱动匹配表写的是：

```c
{ .compatible = "100ask,hs0038" },
```

按当前文件内容，04 版可能无法匹配并进入 `probe()`。05 版已经改正确。

### 11.2 IRQ 开启顺序

05 版先：

```c
request_irq();
```

后：

```c
devm_input_allocate_device();
input_register_device();
```

如果注册 IRQ 后立即发生中断，`hs0038_input_dev` 可能还没有准备好。更安全的顺序是先创建、配置并注册 input 设备，再开启 IRQ。

### 11.3 remove 顺序

当前代码：

```c
input_unregister_device(hs0038_input_dev);
free_irq(irq, NULL);
```

中断可能在 input 设备注销后仍然运行，而 ISR 还会访问 `hs0038_input_dev`。通常应先停止硬件并释放/禁用 IRQ，确保 ISR 完成后再注销 input 设备。

### 11.4 错误处理不完整

以下返回值都没有充分检查：

- `gpiod_get()`
- `gpiod_to_irq()`
- `request_irq()`
- `devm_input_allocate_device()`
- `input_register_device()`
- `copy_to_user()`

教学代码可以简化，但正式驱动必须完善错误回滚。

### 11.5 全局变量不支持多个设备实例

代码使用：

```c
static struct input_dev *hs0038_input_dev;
static int irq;
static int r, w;
```

如果设备树中出现多个 HS0038 节点，这些全局变量会互相覆盖。正式写法应该把状态放进每个设备自己的结构体，并使用：

```c
platform_set_drvdata(pdev, priv);
platform_get_drvdata(pdev);
```

## 12. 05 版和正式红外框架的差别

正式 Linux 红外设备通常会使用 rc-core：

```text
GPIO 红外接收
  -> 原始 pulse/space
  -> rc-core
  -> NEC 协议解码器
  -> scancode
  -> rc keymap
  -> KEY_POWER 等标准 keycode
  -> input core
  -> evdev
```

本例把多个层次合并在一个 platform driver 中：

- GPIO 采集。
- NEC 解码。
- 红外扫描码生成。
- 按键事件上报。

它适合学习：

- platform driver。
- GPIO 中断。
- wait queue。
- 字符设备。
- input_dev 分配和注册。
- input_event/input_sync。

但它还不是完整的生产级红外输入驱动。

## 13. 两份完整驱动源码

以下内容为原始文件的完整副本，便于与前面的分析逐行对照。

### 13.1 04_hs0038_circle_buffer/hs0038_drv.c

原始文件：

```text
E:/韦东山课程及资料/doc_and_source_for_livestream/20211111_开始的Linux驱动快速入门/23_基于输入系统编写红外遥控器HS0038驱动程序/source/04_hs0038_circle_buffer/hs0038_drv.c
```

```c
#include <linux/module.h>
#include <linux/poll.h>

#include <linux/fs.h>
#include <linux/errno.h>
#include <linux/miscdevice.h>
#include <linux/kernel.h>
#include <linux/major.h>
#include <linux/mutex.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/stat.h>
#include <linux/init.h>
#include <linux/device.h>
#include <linux/tty.h>
#include <linux/kmod.h>
#include <linux/gfp.h>
#include <linux/gpio/consumer.h>
#include <linux/platform_device.h>
#include <linux/of_gpio.h>
#include <linux/of_irq.h>
#include <linux/interrupt.h>
#include <linux/irq.h>
#include <linux/slab.h>
#include <linux/fcntl.h>
#include <linux/timer.h>
#include <linux/workqueue.h>
#include <asm/current.h>
#include <linux/delay.h>
#include <linux/ktime.h>
#include <linux/version.h>

static int major;
static struct class *hs0038_class;
static struct gpio_desc *hs0038_data_pin;
//static struct gpio_desc *hs0038_test_pin;
static int irq;
static unsigned int hs0038_data = 0;  
static wait_queue_head_t hs0038_wq;
static u64 hs0038_edge_time[100];
static int hs0038_edge_cnt = 0;


static unsigned int hs0038_data_buf[8];
static int r, w;

static void put_data(unsigned int val)
{
	if (((w+1) & 7) != r)
	{
		hs0038_data_buf[w] = val;
		w = (w + 1) & 7;
	}
}

static int get_data(unsigned int *val)
{
	if (r == w)
	{
		return -1;
	}
	else
	{
		*val = hs0038_data_buf[r];
		r = (r + 1) & 7;		
		return 0;
	}
}

static int has_data(void)
{
	if (r == w)
		return 0;
	else
		return 1;
}

/* 0 : 成功, *val中记录数据
 * -1: 没接收完毕
 * -2: 解析错误
 */
int hs0038_parse_data(unsigned int *val)
{
	u64 tmp;
	unsigned char data[4];
	int i, j, m;
	
	/* 判断是否重复码 */
	if (hs0038_edge_cnt == 4)
	{
		tmp = hs0038_edge_time[1] - hs0038_edge_time[0];
		if (tmp > 8000000 && tmp < 10000000)
		{
			tmp = hs0038_edge_time[2] - hs0038_edge_time[1];
			if (tmp < 3000000)
			{
				/* 获得了重复码 */
				*val = hs0038_data;
				return 0;
			}
		}
	}

	/* 接收到了66次中断 */
	m = 3;
	if (hs0038_edge_cnt >= 68)
	{
		/* 解析到了数据 */
		for (i = 0; i < 4; i++)
		{
			data[i] = 0;
			/* 先接收到bit0 */
			for (j = 0; j < 8; j++)
			{
				/* 数值: 1 */	
				if (hs0038_edge_time[m+1] - hs0038_edge_time[m] > 1000000)
					data[i] |= (1<<j);
				m += 2;
			}
		}

		/* 检验数据 */
		data[1] = ~data[1];
		if (data[0] != data[1])
		{
			printk("%s %s line %d, %x, %x, %x\n", __FILE__, __FUNCTION__, __LINE__, data[0], data[1], ~data[1]);
			return -2;
		}

		data[3] = ~data[3];
		if (data[2] != data[3])
		{
			printk("%s %s line %d, %x, %x, %x\n", __FILE__, __FUNCTION__, __LINE__, data[2], data[3], ~data[3]);
			return -2;
		}

		hs0038_data = (data[0] << 8) | (data[2]);
		*val = hs0038_data;
		return 0;
	}
	else
	{
		/* 数据没接收完毕 */
		return -1;
	}	
}
	

static irqreturn_t hs0038_isr(int irq, void *dev_id)
{
	unsigned int val;
	int ret;
	//printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);

#if (LINUX_VERSION_CODE >= KERNEL_VERSION(5, 0, 0))
		hs0038_edge_time[hs0038_edge_cnt++] = ktime_get_boottime_ns();
#else
		hs0038_edge_time[hs0038_edge_cnt++] = ktime_get_boot_ns();
#endif

	/* 判断超时 */
	
	if (hs0038_edge_cnt >= 2)
	{
		if (hs0038_edge_time[hs0038_edge_cnt-1] - hs0038_edge_time[hs0038_edge_cnt-2] > 30000000)
		{
			/* 超时 */
			hs0038_edge_time[0] = hs0038_edge_time[hs0038_edge_cnt-1];
			hs0038_edge_cnt = 1;			
			return IRQ_HANDLED; // IRQ_WAKE_THREAD;
		}
	}

	ret = hs0038_parse_data(&val);
	if (!ret)
	{
		/* 解析成功 */
		hs0038_edge_cnt = 0;
		// printk("get ir code = 0x%x\n", val);
		put_data(val);
		wake_up(&hs0038_wq);
	}
	else if (ret == -2)
	{
		/* 解析失败 */
		hs0038_edge_cnt = 0;
	}
	
	return IRQ_HANDLED; // IRQ_WAKE_THREAD;
}



/* 实现对应的open/read/write等函数，填入file_operations结构体                   */
static ssize_t hs0038_drv_read (struct file *file, char __user *buf, size_t size, loff_t *offset)
{
	unsigned int val;
	
	if (size != 4)
		return -EINVAL;

	wait_event_interruptible(hs0038_wq, has_data());

	get_data(&val);

	copy_to_user(buf, &val, 4);
		
	return 4;
}

static unsigned int hs0038_drv_poll(struct file *fp, poll_table * wait)
{
//	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
//	poll_wait(fp, &hs0038_wait, wait);
	return 0;
}



/* 定义自己的file_operations结构体                                              */
static struct file_operations hs0038_fops = {
	.owner	 = THIS_MODULE,
	.read    = hs0038_drv_read,
	.poll    = hs0038_drv_poll,
};




/* 1. 从platform_device获得GPIO
 * 2. gpio=>irq
 * 3. request_irq
 */
static int hs0038_probe(struct platform_device *pdev)
{
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
	/* 1. 获得硬件信息 */
	hs0038_data_pin = gpiod_get(&pdev->dev, NULL, 0);
	if (IS_ERR(hs0038_data_pin))
	{
		printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
	}

	irq = gpiod_to_irq(hs0038_data_pin);

	request_irq(irq, hs0038_isr, IRQF_TRIGGER_RISING|IRQF_TRIGGER_FALLING, "hs0038", NULL);

	/* 2. device_create */
	device_create(hs0038_class, NULL, MKDEV(major, 0), NULL, "myhs0038");

	return 0;
}

static int hs0038_remove(struct platform_device *pdev)
{
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
	device_destroy(hs0038_class, MKDEV(major, 0));
	free_irq(irq, NULL);
	gpiod_put(hs0038_data_pin);
//	gpiod_put(hs0038_test_pin);

	return 0;
}


static const struct of_device_id ask100_hs0038[] = {
    { .compatible = "100ask,hs0038" },
    { },
};

/* 1. 定义platform_driver */
static struct platform_driver hs0038_driver = {
    .probe      = hs0038_probe,
    .remove     = hs0038_remove,
    .driver     = {
        .name   = "100ask_hs0038",
        .of_match_table = ask100_hs0038,
    },
};

/* 2. 在入口函数注册platform_driver */
static int __init hs0038_init(void)
{
    int err;
    
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);

	/* 注册file_operations 	*/
	major = register_chrdev(0, "hs0038", &hs0038_fops);  

	hs0038_class = class_create(THIS_MODULE, "hs0038_class");
	if (IS_ERR(hs0038_class)) {
		printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
		unregister_chrdev(major, "hs0038");
		return PTR_ERR(hs0038_class);
	}

	init_waitqueue_head(&hs0038_wq);

	
    err = platform_driver_register(&hs0038_driver); 
	
	return err;
}

/* 3. 有入口函数就应该有出口函数：卸载驱动程序时，就会去调用这个出口函数
 *     卸载platform_driver
 */
static void __exit hs0038_exit(void)
{
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);

    platform_driver_unregister(&hs0038_driver);
	class_destroy(hs0038_class);
	unregister_chrdev(major, "hs0038");
}


/* 7. 其他完善：提供设备信息，自动创建设备节点                                     */

module_init(hs0038_init);
module_exit(hs0038_exit);

MODULE_LICENSE("GPL");




```

### 13.2 05_hs0038_input/hs0038_drv.c

原始文件：

```text
E:/韦东山课程及资料/doc_and_source_for_livestream/20211111_开始的Linux驱动快速入门/23_基于输入系统编写红外遥控器HS0038驱动程序/source/05_hs0038_input/hs0038_drv.c
```

```c
#include <linux/module.h>
#include <linux/poll.h>

#include <linux/fs.h>
#include <linux/errno.h>
#include <linux/miscdevice.h>
#include <linux/kernel.h>
#include <linux/major.h>
#include <linux/mutex.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/stat.h>
#include <linux/init.h>
#include <linux/device.h>
#include <linux/tty.h>
#include <linux/kmod.h>
#include <linux/gfp.h>
#include <linux/gpio/consumer.h>
#include <linux/platform_device.h>
#include <linux/of_gpio.h>
#include <linux/of_irq.h>
#include <linux/interrupt.h>
#include <linux/irq.h>
#include <linux/slab.h>
#include <linux/fcntl.h>
#include <linux/timer.h>
#include <linux/workqueue.h>
#include <asm/current.h>
#include <linux/delay.h>
#include <linux/ktime.h>
#include <linux/version.h>
#include <linux/input.h>

static int major;
static struct class *hs0038_class;
static struct gpio_desc *hs0038_data_pin;
//static struct gpio_desc *hs0038_test_pin;
static int irq;
static unsigned int hs0038_data = 0;  
static wait_queue_head_t hs0038_wq;
static u64 hs0038_edge_time[100];
static int hs0038_edge_cnt = 0;

static unsigned int hs0038_data_buf[8];
static int r, w;

static struct input_dev *hs0038_input_dev;


static void put_data(unsigned int val)
{
	if (((w+1) & 7) != r)
	{
		hs0038_data_buf[w] = val;
		w = (w + 1) & 7;
	}
}

static int get_data(unsigned int *val)
{
	if (r == w)
	{
		return -1;
	}
	else
	{
		*val = hs0038_data_buf[r];
		r = (r + 1) & 7;		
		return 0;
	}
}

static int has_data(void)
{
	if (r == w)
		return 0;
	else
		return 1;
}

/* 0 : 成功, *val中记录数据
 * -1: 没接收完毕
 * -2: 解析错误
 */
int hs0038_parse_data(unsigned int *val)
{
	u64 tmp;
	unsigned char data[4];
	int i, j, m;
	
	/* 判断是否重复码 */
	if (hs0038_edge_cnt == 4)
	{
		tmp = hs0038_edge_time[1] - hs0038_edge_time[0];
		if (tmp > 8000000 && tmp < 10000000)
		{
			tmp = hs0038_edge_time[2] - hs0038_edge_time[1];
			if (tmp < 3000000)
			{
				/* 获得了重复码 */
				*val = hs0038_data;
				return 0;
			}
		}
	}

	/* 接收到了66次中断 */
	m = 3;
	if (hs0038_edge_cnt >= 68)
	{
		/* 解析到了数据 */
		for (i = 0; i < 4; i++)
		{
			data[i] = 0;
			/* 先接收到bit0 */
			for (j = 0; j < 8; j++)
			{
				/* 数值: 1 */	
				if (hs0038_edge_time[m+1] - hs0038_edge_time[m] > 1000000)
					data[i] |= (1<<j);
				m += 2;
			}
		}

		/* 检验数据 */
		data[1] = ~data[1];
		if (data[0] != data[1])
		{
			printk("%s %s line %d, %x, %x, %x\n", __FILE__, __FUNCTION__, __LINE__, data[0], data[1], ~data[1]);
			return -2;
		}

		data[3] = ~data[3];
		if (data[2] != data[3])
		{
			printk("%s %s line %d, %x, %x, %x\n", __FILE__, __FUNCTION__, __LINE__, data[2], data[3], ~data[3]);
			return -2;
		}

		hs0038_data = (data[0] << 8) | (data[2]);
		*val = hs0038_data;
		return 0;
	}
	else
	{
		/* 数据没接收完毕 */
		return -1;
	}	
}
	

static irqreturn_t hs0038_isr(int irq, void *dev_id)
{
	unsigned int val;
	int ret;
	//printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);

#if (LINUX_VERSION_CODE >= KERNEL_VERSION(5, 0, 0))
		hs0038_edge_time[hs0038_edge_cnt++] = ktime_get_boottime_ns();
#else
		hs0038_edge_time[hs0038_edge_cnt++] = ktime_get_boot_ns();
#endif

	/* 判断超时 */
	
	if (hs0038_edge_cnt >= 2)
	{
		if (hs0038_edge_time[hs0038_edge_cnt-1] - hs0038_edge_time[hs0038_edge_cnt-2] > 30000000)
		{
			/* 超时 */
			hs0038_edge_time[0] = hs0038_edge_time[hs0038_edge_cnt-1];
			hs0038_edge_cnt = 1;			
			return IRQ_HANDLED; // IRQ_WAKE_THREAD;
		}
	}

	ret = hs0038_parse_data(&val);
	if (!ret)
	{
		/* 解析成功 */
		hs0038_edge_cnt = 0;
		// printk("get ir code = 0x%x\n", val);
		put_data(val);
		wake_up(&hs0038_wq);

		/* D. 输入系统: 上报数据 */
		input_event(hs0038_input_dev, EV_KEY, val, 1);
		input_event(hs0038_input_dev, EV_KEY, val, 0);
		input_sync(hs0038_input_dev);		
		//input_event(hs0038_input_dev, EV_SYN, 0, 0);
		
		
	}
	else if (ret == -2)
	{
		/* 解析失败 */
		hs0038_edge_cnt = 0;
	}
	
	return IRQ_HANDLED; // IRQ_WAKE_THREAD;
}



/* 实现对应的open/read/write等函数，填入file_operations结构体                   */
static ssize_t hs0038_drv_read (struct file *file, char __user *buf, size_t size, loff_t *offset)
{
	unsigned int val;
	
	if (size != 4)
		return -EINVAL;

	wait_event_interruptible(hs0038_wq, has_data());

	get_data(&val);

	copy_to_user(buf, &val, 4);
		
	return 4;
}

static unsigned int hs0038_drv_poll(struct file *fp, poll_table * wait)
{
//	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
//	poll_wait(fp, &hs0038_wait, wait);
	return 0;
}



/* 定义自己的file_operations结构体                                              */
static struct file_operations hs0038_fops = {
	.owner	 = THIS_MODULE,
	.read    = hs0038_drv_read,
	.poll    = hs0038_drv_poll,
};




/* 1. 从platform_device获得GPIO
 * 2. gpio=>irq
 * 3. request_irq
 */
static int hs0038_probe(struct platform_device *pdev)
{
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
	/* 1. 获得硬件信息 */
	hs0038_data_pin = gpiod_get(&pdev->dev, NULL, 0);
	if (IS_ERR(hs0038_data_pin))
	{
		printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
	}

	irq = gpiod_to_irq(hs0038_data_pin);

	request_irq(irq, hs0038_isr, IRQF_TRIGGER_RISING|IRQF_TRIGGER_FALLING, "hs0038", NULL);

	/* 2. device_create */
	device_create(hs0038_class, NULL, MKDEV(major, 0), NULL, "myhs0038");



	/* 输入系统的代码 */
	/* 参考: drivers\input\keyboard\gpio_keys.c */
	/* A. 分配input_dev */
	hs0038_input_dev = devm_input_allocate_device(&pdev->dev);

	/* B. 设置input_dev */
	hs0038_input_dev->name = "hs0038";
	hs0038_input_dev->phys = "hs0038";

	/* B.1 能产生哪类事件 */
	__set_bit(EV_KEY, hs0038_input_dev->evbit);
	__set_bit(EV_REP, hs0038_input_dev->evbit);
	
	/* B.2 能产生哪些事件 */
	//__set_bit(KEY_0, hs0038_input_dev->keybit);
	memset(hs0038_input_dev->keybit, 0xff, sizeof(hs0038_input_dev->keybit));
	
	/* C. 注册input_dev */
	input_register_device(hs0038_input_dev);

	return 0;
}

static int hs0038_remove(struct platform_device *pdev)
{
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
	input_unregister_device(hs0038_input_dev);

	device_destroy(hs0038_class, MKDEV(major, 0));
	free_irq(irq, NULL);
	gpiod_put(hs0038_data_pin);
//	gpiod_put(hs0038_test_pin);

	return 0;
}


static const struct of_device_id ask100_hs0038[] = {
    { .compatible = "100ask,hs0038" },
    { },
};

/* 1. 定义platform_driver */
static struct platform_driver hs0038_driver = {
    .probe      = hs0038_probe,
    .remove     = hs0038_remove,
    .driver     = {
        .name   = "100ask_hs0038",
        .of_match_table = ask100_hs0038,
    },
};

/* 2. 在入口函数注册platform_driver */
static int __init hs0038_init(void)
{
    int err;
    
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);

	/* 注册file_operations 	*/
	major = register_chrdev(0, "hs0038", &hs0038_fops);  

	hs0038_class = class_create(THIS_MODULE, "hs0038_class");
	if (IS_ERR(hs0038_class)) {
		printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
		unregister_chrdev(major, "hs0038");
		return PTR_ERR(hs0038_class);
	}

	init_waitqueue_head(&hs0038_wq);

	
    err = platform_driver_register(&hs0038_driver); 
	
	return err;
}

/* 3. 有入口函数就应该有出口函数：卸载驱动程序时，就会去调用这个出口函数
 *     卸载platform_driver
 */
static void __exit hs0038_exit(void)
{
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);

    platform_driver_unregister(&hs0038_driver);
	class_destroy(hs0038_class);
	unregister_chrdev(major, "hs0038");
}


/* 7. 其他完善：提供设备信息，自动创建设备节点                                     */

module_init(hs0038_init);
module_exit(hs0038_exit);

MODULE_LICENSE("GPL");




```

## 14. 最终理解要点

```text
红外 val：
    是遥控器协议自己的原始编号

input code：
    是 Linux 输入协议中的“具体对象”，
    对 EV_KEY 来说通常是 KEY_POWER、KEY_ENTER 等

input value：
    是这个对象发生的状态，
    0=松开，1=按下，2=重复

input_sync：
    提交一组相关事件

/dev/myhs0038：
    HS0038 驱动自定义的 4 字节字符设备接口

/dev/input/eventX：
    evdev 提供的标准输入事件接口
```
