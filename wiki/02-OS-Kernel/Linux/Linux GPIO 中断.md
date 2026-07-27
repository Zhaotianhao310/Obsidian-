# Linux GPIO 中断

## 概念

GPIO 中断把 SR501 的电平变化转化为内核事件。raw 示例在 probe 中将 GPIO 转换为 IRQ，并通过 request_irq 注册 ISR。

## 典型流程

1. gpiod_get 获取 GPIO 描述符。
2. gpiod_direction_input 设置输入方向。
3. gpiod_to_irq 将 GPIO 映射为 IRQ 号。
4. request_irq 注册上升沿、下降沿或其他触发方式。
5. ISR 读取 GPIO，更新 sr501_data，并 wake_up 等待队列。
6. remove 中 free_irq，释放 GPIO 描述符和其他资源。

## raw 中的两种中断返回形式

raw 文件不同代码段出现了：

- return IRQ_HANDLED：表示硬中断处理函数已经完成处理。
- return IRQ_WAKE_THREAD：表示希望唤醒线程化中断的下半部。

这两种返回值对应不同的 request_irq/request_threaded_irq 设计，不能只替换返回值而不调整注册方式。

## 中断上下文红线

- ISR 中不应执行可能睡眠的操作。
- 复杂处理应移到线程化中断、工作队列或内核线程。
- ISR 与 read 共享 sr501_data 时，需要考虑并发、可见性和事件覆盖问题。
- request_irq 成功后，所有失败路径都必须保证 free_irq。

## 相关页面

- [[Linux SR501 驱动]]
- [[Linux GPIO 输入]]
- [[Linux 等待队列]]
- [[Linux Platform 驱动]]
- [[SR501 Linux 驱动 API 速查]]
- [[Linux-驱动知识地图]]

## 来源

- raw 文件：Linux驱动SR501代码.md
