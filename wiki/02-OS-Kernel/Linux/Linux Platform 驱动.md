# Linux Platform 驱动

## 概念

platform_driver 用于把平台设备与驱动的 probe/remove 生命周期连接起来。raw 中的 SR501 驱动通过 of_device_id 的 compatible 字段与设备树节点匹配。

## raw 示例结构

- of_device_id 表中包含 compatible = 100ask,sr501。
- platform_driver 设置 .probe 和 .remove。
- .driver.name 使用 100ask_sr501。
- .driver.of_match_table 指向设备树匹配表。
- 模块入口调用 platform_driver_register。
- 模块出口调用 platform_driver_unregister。

## probe 中的职责

probe 是资源获取和硬件初始化入口，raw 示例在这里完成 GPIO 获取、输入配置、GPIO 到 IRQ 的转换、IRQ 注册和设备节点创建。

## remove 中的职责

remove 必须按资源依赖关系撤销设备节点、释放 IRQ、释放 GPIO、销毁 class，并注销字符设备。raw 文件中的前后版本清理完整程度不同，应以最终实现为准。

## 设备树匹配

compatible 字符串必须与设备树节点一致。raw 中的 100ask,sr501 是示例值，不应直接套用到其他板卡或项目。

## 相关页面

- [[Linux SR501 驱动]]
- [[Linux 字符设备驱动]]
- [[Linux GPIO 输入]]
- [[SR501 Linux 驱动 API 速查]]
- [[Linux-驱动知识地图]]

## 来源

- raw 文件：Linux驱动SR501代码.md
