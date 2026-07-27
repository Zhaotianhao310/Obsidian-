# Linux GPIO 输入

## 概念

Linux GPIO consumer API 使用 gpio_desc 表示 GPIO。raw 中的 SR501 驱动通过 gpiod_get 获取 GPIO 描述符，再配置为输入并读取电平。

## raw 中的调用链

1. gpiod_get 获取 GPIO 描述符。
2. gpiod_direction_input 将 GPIO 设置为输入方向。
3. gpiod_get_value 读取当前电平。
4. gpiod_put 在 remove 或失败回滚路径释放描述符。

## 适用位置

- 中断方案：probe 中配置 GPIO，ISR 中读取电平并生成事件。
- 线程方案：内核线程周期性读取电平，再判断是否需要唤醒用户空间。

## 注意事项

- GPIO 描述符的来源和索引必须与设备树或平台设备资源一致。
- gpiod_get、gpiod_direction_input 和 gpiod_to_irq 都可能失败，必须检查错误指针或错误码。
- 读取 GPIO 电平与更新共享状态之间存在并发关系，实际驱动应按目标内核和上下文选择锁或原子变量。
- raw 中使用的是示例性调用，不能据此推断所有平台的设备树属性名称。

## 相关页面

- [[Linux SR501 驱动]]
- [[Linux GPIO 中断]]
- [[Linux Platform 驱动]]
- [[SR501 Linux 驱动 API 速查]]
- [[Linux-驱动知识地图]]

## 来源

- raw 文件：Linux驱动SR501代码.md
