# Linux SR501 驱动

## 概念一句话精炼

SR501 驱动把人体红外传感器的 GPIO 电平变化转换为 Linux 字符设备事件，典型链路为 [[Linux GPIO 中断]] + [[Linux 等待队列]] + 用户态 `read/poll`。

## 核心原理与图解

```mermaid
sequenceDiagram
    participant S as SR501
    participant I as GPIO/IRQ
    participant D as 驱动
    participant U as 用户程序
    S->>I: 输出电平边沿
    I->>D: 调用 ISR
    D->>D: 保存事件状态
    D->>U: 唤醒 read/poll
    U->>D: 读取事件
    D->>U: copy_to_user
```

原始素材包含多个递进版本：早期代码只有 `probe/remove` 框架，后续版本加入 GPIO、IRQ、等待队列、字符设备和 `device_create`。因此本页总结的是设计链路，不把 raw 中任一片段视作可直接部署的最终实现。

## 关键实现与数据结构

```c
struct sr501 {
    struct gpio_desc *gpio;
    int irq;
    int data;
    wait_queue_head_t wq;
    struct cdev cdev;
};

/* ISR 只记录事件并唤醒；read 再完成用户空间拷贝 */
```

推荐的初始化顺序是：获取输入 GPIO → 转换 IRQ → 注册中断 → 初始化字符设备/设备节点 → 对外提供读取接口。退出顺序应反向执行。

## 原始代码问题与待确认项

- `IRQ_WAKE_THREAD` 需要配套线程化中断处理函数；raw 未给出完整实现。
- `sr501_data` 作为共享状态时需要考虑并发和重复事件，不能只依靠普通整型读写推断完整同步性。
- `copy_to_user` 的返回值必须检查，不能忽略未拷贝字节数。
- `remove` 中应释放 IRQ、GPIO、等待队列关联资源、字符设备和设备节点。
- `device_create`、主设备号、class 和 `file_operations` 的组合需要与目标内核版本 API 对齐。

## 横向对比与关联

- [[Linux GPIO 输入]]：只负责取得输入电平。
- [[Linux GPIO 中断]]：负责把边沿变成事件。
- [[Linux 等待队列]]：负责让读进程睡眠和唤醒。
- [[Linux 字符设备驱动]]：负责建立用户态访问入口。
- [[Linux Platform 驱动]]：负责设备匹配和生命周期。

## 来源

- raw 文件：`Linux驱动SR501代码.md`