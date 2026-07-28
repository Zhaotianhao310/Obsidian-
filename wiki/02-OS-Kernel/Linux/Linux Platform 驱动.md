# Linux Platform 驱动

## 概念一句话精炼

Platform 驱动用于把设备树或板级注册的非枚举设备与驱动代码绑定起来，核心生命周期是匹配、`probe` 初始化和 `remove` 释放，并关联 [[Linux GPIO 输入]]。

## 核心原理与图解

```mermaid
flowchart LR
    A[设备树 compatible] --> B[of_match_table]
    B --> C[platform_driver_register]
    C --> D[匹配成功]
    D --> E[probe 获取 GPIO/IRQ/设备节点]
    E --> F[remove 释放资源]
```

匹配表决定驱动是否接管设备；`probe` 失败时必须回滚已获取资源，`remove` 则应释放所有非 devm 资源并注销用户态接口。

## 关键实现与数据结构

```c
static const struct of_device_id sr501_of_match[] = {
    { .compatible = "100ask,sr501" },
    { }
};

static struct platform_driver sr501_driver = {
    .probe = sr501_probe,
    .remove = sr501_remove,
    .driver = { .name = "sr501", .of_match_table = sr501_of_match },
};
module_platform_driver(sr501_driver);
```

## 横向对比与关联

- Platform 驱动依赖设备树/板级描述；USB、PCI 等总线驱动则由各自总线枚举设备。
- `probe` 是资源获取和初始化入口；`remove` 是资源释放和状态撤销出口。
- 原始代码存在驱动名、`compatible` 字符串和设备节点创建路径的版本/实现差异，不能直接视为最终可编译驱动。

- [[Linux SR501 驱动]]
- [[Linux 字符设备驱动]]
- [[SR501 Linux 驱动 API 速查]]

## 来源

- raw 文件：`Linux驱动SR501代码.md`