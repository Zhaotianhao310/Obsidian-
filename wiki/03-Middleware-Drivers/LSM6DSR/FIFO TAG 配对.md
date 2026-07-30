# FIFO TAG 配对

## 概念一句话
FIFO TAG 配对把 LSM6DSR FIFO 中独立出现的 gyro/accel word 识别、解码并组合为一个六轴 sample；核心关联是 [[LSM6DSR FIFO 采集]] 与 [[IMU TCP 固定帧]]。

## 核心原理与图解
每个 FIFO word 为 7 字节：第 0 字节包含 TAG，后 6 字节按 little-endian 存放 X/Y/Z 三轴。状态机分别缓存最近一个 gyro 和 accel；两类数据都有效时输出 Ax, Ay, Az, Gx, Gy, Gz，然后清除 valid 标志。若同类 TAG 在另一类到达前再次出现，说明前一份数据被覆盖，应累计配对错误。

~~~mermaid
stateDiagram-v2
    [*] --> 空
    空 --> 仅陀螺: "TAG=0x01"
    空 --> 仅加速度: "TAG=0x02"
    仅陀螺 --> 完整Sample: "收到 TAG=0x02"
    仅加速度 --> 完整Sample: "收到 TAG=0x01"
    仅陀螺 --> 仅陀螺: "再次收到 0x01，pair_error++"
    仅加速度 --> 仅加速度: "再次收到 0x02，pair_error++"
    完整Sample --> 空: "输出六轴数据并清 valid"
~~~

> 图 1：gyro/accel 的最小配对状态机；图示根据 raw 代码提炼，raw 未提供可直接引用的图片资源。

## 关键实现/数据结构
~~~c
typedef struct {
    imu_xyz_t accel;       // 最近一个加速度 word
    imu_xyz_t gyro;        // 最近一个陀螺仪 word
    bool accel_valid;      // 加速度缓存是否有效
    bool gyro_valid;       // 陀螺仪缓存是否有效
} imu_pair_state_t;

uint8_t tag_sensor = (word[0] >> 3) & 0x1F;
if (tag_sensor == 0x01) pair.gyro = decode_xyz(word);
else if (tag_sensor == 0x02) pair.accel = decode_xyz(word);
~~~

## 横向对比与关联
- [[LSM6DSR FIFO 采集]]：负责提供有序 FIFO word 和 overrun/full 状态。
- [[IMU TCP 固定帧]]：把配对后的六轴 sample 按 100 组装入固定长度 frame。
- [[ESP32-S3 IMU 启动时序]]：FIFO reset 或异常恢复时必须清空配对 valid 标志，避免跨 FIFO 世代配对。
- 与“按固定位置交替读取”相比，TAG 状态机能发现未知 TAG 和同类覆盖，但不能替代时间戳对齐校验。

## 冲突与待核实
- raw 只明确识别 TAG sensor 0x01 和 0x02；具体 TAG 编码、FIFO 中 gyro/accel 的交错顺序仍应与 LSM6DSR 数据手册核对。
- 发生配对覆盖时，当前策略是计数并继续工作，不会恢复到传感器级别；是否丢弃当前缓存需结合数据质量要求确认。

## 来源
- raw 文件：ESP32S3_IMU_项目完整总结.md
