# Linux 字符设备驱动

## 概念

字符设备驱动通过 file_operations 将用户空间的 open、read、write、poll 等操作映射到内核函数。raw 中的 SR501 示例以 read 和 poll 为主，将传感器事件暴露给用户空间。

## raw 示例结构

- 定义 read 函数，例如 sr501_drv_read 或 sr501_read。
- 定义 poll 函数，例如 sr501_drv_poll 或 sr501_poll。
- 用 file_operations 结构体填写 .read 和 .poll。
- 通过 register_chrdev 注册字符设备。
- 通过 class_create 和 device_create 创建设备类与设备节点。
- 卸载时调用 device_destroy、class_destroy 和 unregister_chrdev 等清理函数。

## 读数据路径

1. 用户进程调用 read。
2. 驱动检查 SR501 状态；没有新事件时进入 [[Linux 等待队列]]。
3. [[Linux GPIO 中断]]或轮询逻辑更新状态并唤醒等待者。
4. 驱动使用 copy_to_user 把状态复制到用户缓冲区。
5. 驱动清除或更新事件标志，准备下一次读取。

## 设备节点

raw 示例中使用 device_create 创建 /dev/sr501。设备节点是否能成功出现，还取决于主设备号、class、内核设备模型和错误回滚是否正确。

## 注意事项

- file_operations 的函数签名必须匹配目标内核版本。
- copy_to_user 的返回值不能忽略；返回非零表示仍有数据没有复制成功。
- 设备注册的每一步都要有对应的失败回滚和卸载清理。
- 不能把 raw 中的多个渐进式代码块直接拼接为一个最终驱动。

## 相关页面

- [[Linux SR501 驱动]]
- [[Linux 等待队列]]
- [[Linux GPIO 中断]]
- [[Linux Platform 驱动]]
- [[SR501 Linux 驱动 API 速查]]
- [[Linux-驱动知识地图]]

## 来源

- raw 文件：Linux驱动SR501代码.md
