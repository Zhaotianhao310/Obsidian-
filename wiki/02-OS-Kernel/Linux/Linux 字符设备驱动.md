# Linux 字符设备驱动

## 概念一句话精炼

字符设备驱动通过 `file_operations` 把内核硬件事件映射为用户态 `open/read/poll` 等文件操作，是 [[Linux SR501 驱动]]提供 `/dev` 入口的基础。

## 核心原理与图解

```mermaid
flowchart LR
    A[用户 open/read/poll] --> B[VFS]
    B --> C[file_operations]
    C --> D[SR501 驱动状态]
    D --> E[GPIO/IRQ/等待队列]
    E --> D
```

驱动需要完成设备号、`file_operations`、class/device 或 cdev 注册，并保证用户操作期间底层资源仍然有效。

## 关键实现与数据结构

```c
static const struct file_operations sr501_fops = {
    .owner = THIS_MODULE,
    .read  = sr501_read,
    .poll  = sr501_poll,
};

/* read 中等待事件，poll 中调用 poll_wait 并返回可读掩码 */
```

`read` 负责数据传输和阻塞语义，`poll` 负责向 `select/poll/epoll` 注册等待队列；二者必须使用一致的就绪条件。

## 横向对比与关联

- **字符设备**：适合事件流和字节流式接口。
- **sysfs**：适合暴露少量设备属性，不适合持续事件读取。
- **miscdevice**：可减少手动主设备号管理，但仍需正确实现文件操作和生命周期。
- `copy_to_user` 只能在进程上下文调用，不能放进 GPIO ISR。

- [[Linux GPIO 中断]]
- [[Linux 等待队列]]
- [[Linux Platform 驱动]]
- [[SR501 Linux 驱动 API 速查]]

## 来源

- raw 文件：`Linux驱动SR501代码.md`