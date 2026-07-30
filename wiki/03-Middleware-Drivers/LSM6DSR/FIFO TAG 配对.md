# FIFO TAG 配对

## 概念一句话
FIFO TAG 配对把 LSM6DSR FIFO 中独立出现的 gyro/accel word 识别、解码并组合为一个六轴 sample；核心关联是 [[LSM6DSR FIFO 采集]] 与 [[IMU TCP 固定帧]]。

## 核心原理与图解

### FIFO word 到六轴 sample 的流水线

1. 从 `FIFO_DATA_OUT_TAG`（`0x78`）开始读取一个 7 字节 word。
2. byte0 为 TAG；byte1~6 按 little-endian 组合为 X/Y/Z 三个 16 位有符号值。
3. 取 `TAG_SENSOR[4:0] = (tag >> 3) & 0x1F`，项目当前识别 `0x01` 为 gyro、`0x02` 为 accel。
4. 分别覆盖最近一个 gyro 或 accel 缓存，并设置对应 valid 标志。
5. 两类缓存都有效时按 Ax、Ay、Az、Gx、Gy、Gz 输出一组 sample，然后清除两个 valid 标志。
6. 同类 TAG 在另一类到达前再次出现时，说明前一份缓存可能被覆盖；项目计数 `pair_error`，但是否丢弃哪一份数据需按质量要求确认。

~~~mermaid
stateDiagram-v2
    [*] --> 空
    空 --> 仅陀螺: "TAG_SENSOR=0x01"
    空 --> 仅加速度: "TAG_SENSOR=0x02"
    仅陀螺 --> 完整Sample: "收到 0x02，组合六轴"
    仅加速度 --> 完整Sample: "收到 0x01，组合六轴"
    仅陀螺 --> 仅陀螺: "再次收到 0x01，pair_error++"
    仅加速度 --> 仅加速度: "再次收到 0x02，pair_error++"
    完整Sample --> 空: "输出并清 valid"
~~~

> 图 1：以两个 valid 标志为核心的 TAG 配对状态机；它依赖 FIFO 的 7 字节边界，但不把固定交替顺序当成协议保证。

### 寄存器与 TAG 字段

| 寄存器/字段 | 地址/位域 | 当前项目含义 | 硬件/软件行为 |
|---|---|---|---|
| `FIFO_DATA_OUT_TAG` | `0x78`，`TAG_SENSOR[4:0]` 为 bit[7:3] | 读取每个 FIFO word 的来源标记 | 首字节与后续 6 字节轴数据组成一个 word |
| `FIFO_DATA_OUT_X/Y/Z` | `0x79~0x7E` | X/Y/Z 小端原始数据 | 每个轴由低字节和高字节组合成 `int16_t` |
| `FIFO_STATUS1/2` | `0x3A/0x3B` | 由上游采集笔记读取 | 提供未读 word 数和 FIFO 异常状态，决定是否继续消费 |

项目当前只将 `0x01`、`0x02` 作为 gyro/accel 配对输入；其他 TAG 不能静默当作这两类数据。

## 关键实现/数据结构

以下函数 `decode_xyz` 和配对状态结构属于项目实现，不是官方 API。

~~~c
typedef struct {
    imu_xyz_t accel, gyro;       // 最近一份两类传感器数据
    bool accel_valid, gyro_valid;
} imu_pair_state_t;
uint8_t sensor = (word[0] >> 3) & 0x1F;  // TAG_SENSOR[4:0]
if (sensor == 0x01) { pair.gyro = decode_xyz(&word[1]); pair.gyro_valid = true; }
if (sensor == 0x02) { pair.accel = decode_xyz(&word[1]); pair.accel_valid = true; }
if (pair.accel_valid && pair.gyro_valid) emit_sample_and_clear(&pair);
~~~

`decode_xyz`、`emit_sample_and_clear` 和 `pair` 都是项目业务实现；官方接口只负责把 FIFO 字节读到缓冲区。

## 横向对比与关联

- [[LSM6DSR FIFO 采集]]：保证读取从 `FIFO_DATA_OUT_TAG` 开始并按 7 字节推进。
- [[LSM6DSR SPI 驱动]]：保证 SPI 读标志、命令字节和 DMA 缓冲区偏移正确。
- [[IMU TCP 固定帧]]：只接收已经配对完成的六轴 sample，并累计 100 组。
- 与“按固定位置交替读取”相比，TAG 状态机能发现未知 TAG 和同类覆盖，但不能替代时间戳对齐校验。

## 硬件避坑

- 不能跳过 TAG byte 直接按 6 字节轴数据读取；这样会导致所有轴字节错位。
- `TAG_SENSOR` 只占 TAG 字节的部分位域，不能把整个 byte 直接当作传感器编号。
- FIFO reset 或异常恢复后必须清空 gyro/accel valid 标志，避免跨 FIFO 世代配对。
- 未知 TAG、奇偶校验异常或同类覆盖应计数并保留诊断信息，不能静默丢弃。

## 冲突与待核实

- raw 只明确将 `0x01`、`0x02` 映射到 gyro/accel；其他 TAG 的处理和完整交错顺序应以目标 LSM6DSR 数据手册复核。
- 项目当前遇到配对覆盖时采取计数并继续工作；是否应丢弃整组 sample，需要结合上位机数据质量要求确认。

## 来源

- raw 文件：`ESP32S3_IMU_项目完整总结.md`
