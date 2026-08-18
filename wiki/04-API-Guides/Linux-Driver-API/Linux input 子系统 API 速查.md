# Linux input 子系统 API 速查

## 适用范围

本页整理 Linux 内核 input core、按键事件和 evdev 相关的官方接口与核心数据结构，服务于 [[Linux input 子系统]] 和 [[Linux HS0038 红外驱动]]。`hs0038_parse_data()`、`hs0038_report_key()` 等是工程函数，不属于 Linux 官方 API。

## 基础头文件

~~~c
#include <linux/input.h>
#include <linux/device.h>
#include <linux/errno.h>
#include <linux/err.h>
~~~

用户态读取 `/dev/input/eventX` 时使用：

~~~c
#include <linux/input.h>
#include <fcntl.h>
#include <unistd.h>
~~~

## 核心数据结构

### `struct input_dev`

- **作用**：描述一个输入设备，是驱动向 input core 注册的主要对象。
- **常用字段**：`name`、`phys`、`id`、`evbit`、`keybit`、`open`、`close`、`dev`。
- **配置原则**：优先使用 `input_set_capability()` 设置能力，不要直接把 `keybit` 全部置为 `0xff`。
- **生命周期**：分配后填写能力，再调用 `input_register_device()`；注销后不得继续从 IRQ、workqueue 或 timer 访问该对象。

### `struct input_event`

- **作用**：evdev 用户态 ABI 的事件记录格式。
- **字段**：`time` 事件时间；`type` 事件类别；`code` 事件对象；`value` 状态或数值。
- **典型组合**：`EV_KEY + KEY_POWER + 1` 按下；`EV_KEY + KEY_POWER + 0` 松开；`EV_SYN + SYN_REPORT + 0` 提交一组事件。
- **注意**：这是用户态 eventX ABI；内核驱动通常调用 `input_report_key()`、`input_event()`，而不是自行写 evdev 队列。

### `struct input_handler`

- **作用**：input core 的事件消费者，例如 evdev、键盘或触摸处理器。
- **常用字段**：`name`、`id_table`、`match`、`connect`、`disconnect`、`event`。
- **使用边界**：普通硬件驱动只需注册 `input_dev`；编写新的输入事件消费者时才实现 handler。

### `struct input_handle`

- **作用**：连接一个 `input_dev` 与一个 `input_handler`。
- **常用字段**：`dev` 输入设备；`handler` 处理器；`private` 处理器私有状态。
- **使用边界**：由 handler 的 `connect()` / `disconnect()` 管理，不应手工篡改 input core 链表。

### `struct evdev_client`

- **作用**：evdev 为每个打开的 `/dev/input/eventX` 文件维护的内部客户端队列。
- **使用边界**：属于 evdev 实现内部结构，不是普通输入驱动的稳定公共 API；驱动不能直接操作其缓冲区。

## 核心函数详解

### `devm_input_allocate_device`

- **作用**：分配由 devres 自动管理释放的 `input_dev`。
- **函数原型/签名**：`struct input_dev *devm_input_allocate_device(struct device *dev);`
- **参数含义详解**：`dev` 是当前 platform/I2C/SPI 设备的 `struct device`。
- **返回值状态码**：成功返回 `input_dev *`；失败返回 `NULL`。
- **使用 Demo**：

~~~c
priv->input = devm_input_allocate_device(&pdev->dev);
if (!priv->input)
    return -ENOMEM;
priv->input->name = "hs0038";
priv->input->phys = "gpio-ir/input0";
~~~

### `input_set_capability`

- **作用**：声明输入设备支持的事件类型和事件码。
- **函数原型/签名**：`void input_set_capability(struct input_dev *dev, unsigned int type, unsigned int code);`
- **参数含义详解**：`type` 如 `EV_KEY`、`EV_REL`；`code` 如 `KEY_POWER`、`REL_X`。
- **返回值状态码**：无返回值。
- **使用 Demo**：

~~~c
input_set_capability(priv->input, EV_KEY, KEY_POWER);
input_set_capability(priv->input, EV_KEY, KEY_VOLUMEUP);
input_set_capability(priv->input, EV_MSC, MSC_SCAN);
priv->input->id.bustype = BUS_HOST;
~~~

### `input_register_device`

- **作用**：把填写完成的 `input_dev` 注册到 input core，并触发 evdev 等 handler 匹配。
- **函数原型/签名**：`int input_register_device(struct input_dev *dev);`
- **参数含义详解**：`dev` 必须已设置名称和能力；成功后可生成 `/dev/input/eventX`。
- **返回值状态码**：成功返回 `0`；失败返回负 errno。
- **使用 Demo**：

~~~c
ret = input_register_device(priv->input);
if (ret) {
    dev_err(&pdev->dev, "input register failed: %d\n", ret);
    return ret;
}
priv->input_registered = true;
~~~

### `input_unregister_device`

- **作用**：从 input core 注销输入设备，停止后续 handler 分发。
- **函数原型/签名**：`void input_unregister_device(struct input_dev *dev);`
- **参数含义详解**：`dev` 必须已成功注册；调用前应停止 IRQ、timer 和 workqueue 等异步访问者。
- **返回值状态码**：无返回值。
- **使用 Demo**：

~~~c
if (priv->input_registered) {
    disable_irq(priv->irq);
    synchronize_irq(priv->irq);
    input_unregister_device(priv->input);
    priv->input_registered = false;
}
~~~

### `input_event`

- **作用**：向 input core 提交一个原始输入事件。
- **函数原型/签名**：`void input_event(struct input_dev *dev, unsigned int type, unsigned int code, int value);`
- **参数含义详解**：`type` 是事件类别；`code` 是事件码；`value` 是状态或数值；事件应符合已声明能力。
- **返回值状态码**：无返回值；未声明能力可能被 input core 过滤。
- **使用 Demo**：

~~~c
input_event(dev, EV_MSC, MSC_SCAN, ir_code);
input_event(dev, EV_KEY, KEY_POWER, 1);
input_event(dev, EV_SYN, SYN_REPORT, 0);
input_event(dev, EV_KEY, KEY_POWER, 0);
input_event(dev, EV_SYN, SYN_REPORT, 0);
~~~

### `input_report_key`

- **作用**：提交按键状态，是 `input_event(dev, EV_KEY, code, value)` 的便捷封装。
- **函数原型/签名**：`void input_report_key(struct input_dev *dev, unsigned int code, int value);`
- **参数含义详解**：`code` 必须是已声明的 `KEY_*`；`value` 通常为 `0` 松开、`1` 按下、`2` 重复。
- **返回值状态码**：无返回值。
- **使用 Demo**：

~~~c
input_report_key(dev, KEY_POWER, 1);
input_sync(dev);
/* 收到释放或超时后再报告松开 */
input_report_key(dev, KEY_POWER, 0);
input_sync(dev);
~~~

### `input_sync`

- **作用**：提交当前一组输入事件，产生 `EV_SYN/SYN_REPORT` 边界。
- **函数原型/签名**：`void input_sync(struct input_dev *dev);`
- **参数含义详解**：`dev` 是正在报告事件的输入设备；应在一组相关事件全部提交后调用。
- **返回值状态码**：无返回值。
- **使用 Demo**：

~~~c
input_report_key(dev, KEY_VOLUMEUP, 1);
input_event(dev, EV_MSC, MSC_SCAN, raw_code);
input_sync(dev);
~~~

## 事件常量速查

| 常量 | 含义 | 典型用途 |
| :--- | :--- | :--- |
| `EV_KEY` | 按键/按钮事件 | `KEY_POWER`、`KEY_ENTER` |
| `EV_MSC` | 其他扫描信息 | `MSC_SCAN` 携带原始红外码 |
| `EV_SYN` | 同步事件 | `SYN_REPORT` 标记事件包结束 |
| `EV_REP` | 自动重复能力 | 长按重复；不能替代重复帧处理 |
| `KEY_*` | 标准按键码 | 将厂商红外码映射为 Linux 语义 |
| `MSC_SCAN` | 原始扫描码 | 保留未映射的协议编号 |

## 避坑红线

- 必须先用 `input_set_capability()` 声明能力，再报告对应事件；不要用 `memset(keybit, 0xff, ...)` 伪造“支持所有按键”。
- 红外厂商码不是 `KEY_*`；应通过 keymap 映射，未知码返回错误或只作为 `MSC_SCAN` 上报。
- `input_sync()` 是事件包边界，不是可省略的日志调用；缺失会导致用户态无法可靠判断一组事件何时完成。
- `input_report_key(..., 1)` 后立即 `input_report_key(..., 0)` 不会产生长按；重复帧应使用 `value=2` 或保持按下状态，并用定时器/超时释放。
- 硬中断中只做快速状态更新和事件提交；不能调用 `copy_to_user()`、阻塞等待或可能睡眠的操作。
- 注册 IRQ 的时序应晚于 `input_dev` 完成配置和注册；卸载时先停止/同步 IRQ，再注销 input 设备。
- `devm_input_allocate_device()`、`input_register_device()` 和 `input_unregister_device()` 的所有权必须与目标内核版本和 devres 使用方式保持一致，避免重复释放。
- `input_handler`、`input_handle` 可以用于编写自定义事件消费者，但普通硬件输入驱动通常只需维护 `input_dev`。

## 相关页面

- [[Linux input 子系统]]
- [[Linux HS0038 红外驱动]]
- [[Linux GPIO 中断]]
- [[Linux 字符设备驱动]]
- [[Linux Platform 驱动]]
- [[Linux-驱动知识地图]]

## 来源

- raw 文件：`hs0038_input_subsystem_analysis.md`
