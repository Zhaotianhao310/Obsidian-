# Linux 等待队列

## 概念

等待队列用于让没有新传感器事件的用户进程休眠，并在 GPIO 中断或轮询逻辑产生事件时被唤醒。raw 中的 SR501 read 函数使用 wait_event_interruptible，ISR 或检测线程使用 wake_up。

## SR501 数据路径

- read 检查 sr501_data。
- 没有新数据时，调用 wait_event_interruptible 进入可被信号打断的睡眠。
- GPIO 中断或检测线程写入 sr501_data。
- 生产者调用 wake_up 唤醒等待者。
- read 被唤醒后通过 copy_to_user 返回数据，并清除或更新事件状态。

## poll 方案

raw 中定义了 sr501_poll/sr501_drv_poll，并包含 poll_wait 的注释示例，但部分代码段没有完整注册等待队列和事件掩码。因此只能把它视为 poll 接口的演示骨架，不能直接当作完整非阻塞驱动。

## 并发注意

- sr501_data 同时由中断/线程写入、read 读取，必须设计明确的同步策略。
- 等待条件必须与状态更新配套，否则可能出现丢事件或虚假唤醒。
- wait_event_interruptible 被信号打断时，read 需要处理返回值。
- 如果传感器事件可能连续到达，单个布尔标志可能覆盖事件，应根据业务选择计数器、环形缓冲区或其他队列。

## 相关页面

- [[Linux SR501 驱动]]
- [[Linux 字符设备驱动]]
- [[Linux GPIO 中断]]
- [[SR501 Linux 驱动 API 速查]]
- [[Linux-驱动知识地图]]

## 来源

- raw 文件：Linux驱动SR501代码.md
