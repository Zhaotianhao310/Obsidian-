# Linux SR501 驱动

## 驱动对象

SR501 是输出数字电平的人体红外感应模块。raw 素材围绕它演示 Linux GPIO 输入、字符设备、platform_driver、设备树匹配、中断、等待队列和用户态读取的组合方式。

## 典型数据链路

设备树中的 compatible 信息匹配 platform_driver 后，驱动在 probe 中完成以下工作：

1. 获取 SR501 对应 GPIO。
2. 把 GPIO 配置为输入。
3. 选择事件检测方式：GPIO 中断或内核线程轮询。
4. 将检测结果写入驱动状态变量。
5. 通过等待队列唤醒阻塞在 read 的用户进程。
6. 由 read 使用 copy_to_user 把状态传给用户空间。

## 驱动组成

- file_operations：把 read、poll 等用户态操作映射到驱动函数。
- platform_driver：承载 probe/remove 生命周期，并通过设备树 compatible 匹配硬件。
- GPIO consumer API：通过 gpiod_get 等接口获取 GPIO 描述符。
- IRQ 或线程：把 GPIO 电平变化转化为驱动事件。
- wait queue：没有新事件时让 read 休眠，有事件时唤醒用户进程。
- 字符设备与 class/device：为用户空间创建设备节点，例如 raw 素材中的 /dev/sr501。

## raw 中的实现变体

raw 文件不是一份单一的最终代码，而是多段递进式示例：

- 有的版本使用 GPIO 中断，在 ISR 中读取电平、设置状态并 wake_up。
- 有的版本使用内核线程轮询 GPIO，再通过等待队列通知用户空间。
- 有的版本补充 register_chrdev、class_create 和 device_create，以自动创建设备节点。
- 有的版本实现 poll，但 poll_wait 仍以注释形式出现，不能视为完整的非阻塞 poll 实现。

## 冲突与待核实

- 不同代码段的 ISR 返回 IRQ_HANDLED 或 IRQ_WAKE_THREAD，二者对应的线程化中断设计不同，不能直接混用。
- raw 中出现 kernel_thread 方案；它属于示例中的旧式写法，是否适用于目标内核版本需要单独确认。
- 多个版本的清理路径并不完整，尤其是 request_irq、GPIO 描述符、设备节点和 class 的失败回滚，需要按最终版本补齐。

## 相关页面

- [[Linux 字符设备驱动]]
- [[Linux GPIO 输入]]
- [[Linux GPIO 中断]]
- [[Linux 等待队列]]
- [[Linux Platform 驱动]]
- [[SR501 Linux 驱动 API 速查]]
- [[Linux-驱动知识地图]]

## 来源

- raw 文件：Linux驱动SR501代码.md
