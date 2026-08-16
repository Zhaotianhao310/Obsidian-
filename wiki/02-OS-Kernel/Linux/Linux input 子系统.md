# Linux input 子系统

## 概念一句话精炼

Linux input 子系统把驱动产生的按键、相对位移或绝对坐标事件统一为 `input_dev`，再由 `evdev` 转换为 `/dev/input/eventX`，连接 [[Linux GPIO 中断]] 与用户态输入程序。

## 核心原理与图解

```mermaid
flowchart LR
    A[硬件边沿或采样] --> B[驱动 ISR/线程]
    B --> C[input_dev]
    C --> D[input core]
    D --> E[evdev input_handler]
    E --> F[evdev_client]
    F --> G[/dev/input/eventX]
```

> 图 1：驱动只负责注册输入设备和提交事件；input core 负责分发，evdev 为每个打开的 eventX 文件维护独立事件队列。

核心对象关系：

- `struct input_dev`：事件生产者，描述设备名称、物理路径和能力位图。
- `struct input_handler`：事件消费者，`evdev` 是常用实现。
- `struct input_handle`：把一个 `input_dev` 与一个 handler 连接起来。
- `struct evdev_client`：每个用户态 `open()` 对应的客户端队列。

注册 `input_dev` 后，input core 会遍历 handler 并匹配 evdev，驱动不需要自行实现 `eventX` 的 `read()`。

## 事件模型与提交边界

`input_event(dev, type, code, value)` 的三个字段含义如下：

| 字段 | 作用 | 按键示例 |
| :--- | :--- | :--- |
| `type` | 事件类别 | `EV_KEY` |
| `code` | 事件对象 | `KEY_POWER` |
| `value` | 对象状态 | `1` 按下，`0` 松开，`2` 重复 |

`value` 表示 Linux 输入协议的逻辑状态，不等同于 GPIO 高低电平。多个事件组成一组报告时，必须调用 `input_sync()` 产生 `EV_SYN/SYN_REPORT`，通知用户态这一组事件已经提交。

红外驱动得到的厂商扫描码不能直接当作 `EV_KEY` 的 `code`，应先映射为 `KEY_POWER`、`KEY_ENTER` 等标准按键码；原始码可额外通过 `EV_MSC/MSC_SCAN` 保留。

## 关键实现与数据结构

```c
static int hs0038_report_key(struct input_dev *dev, unsigned int ir)
{
    int code = hs0038_to_keycode(ir);       // 厂商码 -> Linux KEY_*
    if (code < 0) return -EINVAL;           // 未知扫描码不伪装成按键
    input_report_key(dev, code, 1);         // 按下
    input_sync(dev);                        // 提交按下事件包
    input_report_key(dev, code, 0);         // 松开
    input_sync(dev);                        // 提交松开事件包
    return 0;
}
```

正式驱动应使用 `input_set_capability(dev, EV_KEY, KEY_*)` 精确声明能力，不应以 `memset(keybit, 0xff, ...)` 宣称支持所有按键。设置 `EV_REP` 只表示允许自动重复；若代码立即上报 `1` 后 `0`，重复定时器会被取消，不能形成长按效果。长按应保存当前按键状态、处理重复帧，并在超时后上报释放。

## 与其他概念的横向对比/关联

- [[Linux 字符设备驱动]]：自定义 `/dev/myhs0038` 接口，适合读取原始 4 字节码；input/evdev 是标准输入接口。
- [[Linux Platform 驱动]]：负责设备树匹配、`probe/remove` 生命周期，不负责定义输入事件语义。
- [[Linux GPIO 中断]]：负责把硬件边沿交给驱动；ISR 不应直接执行用户空间拷贝。
- [[Linux HS0038 红外驱动]]：本页输入对象和事件协议的具体生产者。
- [[Linux-驱动知识地图]]：Linux 驱动主题的集中索引。
- 正式红外链路通常是 `GPIO -> rc-core -> NEC 解码器 -> keymap -> input core -> evdev`，HS0038 教学驱动将多层逻辑合并在一个 platform driver 中。

## 来源

- raw 文件：`hs0038_input_subsystem_analysis.md`
