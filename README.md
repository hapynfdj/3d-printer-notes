# 3D 打印机参数与调参笔记

Apollo3D Saturn 10 Plus（Klipper / Bowden 改造）的调参记录与操作指南。

## 背景

本机为床升降式 Cartesian 结构（190×190×151.25 mm），无探针、无自动调平，挤出机为 BMG 远端（Bowden，PTFE 管约 500 mm）。固件为 Klipper，切片使用 OrcaSlicer。

以下内容记录本机在 2026-08 期间完成并验证的配置，以及调参过程中总结的操作方法。所有参数均为本机实测值，迁移到其他机型前请自行验证。

## 硬件基线

| 项目 | 规格 |
|---|---|
| 机型 | Apollo3D Saturn 10 Plus |
| 主板 | Apollo 定制 ATmega2560（MKS Gen_L 引脚映射） |
| 串口速率 | 250000 baud |
| 行程 | 190×190×151.25 mm（笛卡尔） |
| 探针 | 无（无 BLTouch / bed mesh / screws_tilt） |
| Z 轴 | 床升降式；Z=0 为打印位，归零后床位于 Z=151.25 |
| 挤出机 | BMG 远端（Bowden），PTFE 约 500 mm |
| rotation_distance | 7.7108（20 mm 指令实测 20 mm） |

> 注意：本机 Z 轴方向与常见教程相反（Z=0 为打印位，非归零位），执行 `G1 Z0` 前需分步下降，详见[手动调平](docs/manual-bed-leveling.md)。

## 软件环境

- Klipper + Moonraker：`http://192.168.5.32:7125`
- Mainsail：`http://192.168.5.32:8320`
- OrcaSlicer（MCP fork，启用 Remote API）

## 当前参数（2026-08 验证）

### Klipper

```ini
max_velocity = 190
max_accel = 7350
max_extrude_only_velocity = 45
max_extrude_only_accel = 3000
```

### OrcaSlicer 预设

- 机器：`MyKlipper 0.4 nozzle - 拷贝`
- 耗材：`Generic PETG @System - 拷贝`
- 工艺：`0.20mm PETG Stable @Saturn10Plus`

| 参数 | 值 |
|---|---|
| 首层 / 后续喷嘴温度 | 240 °C / 235 °C |
| 首层 / 后续热床温度 | 78 °C / 75 °C |
| 压力提前（PA） | 0.08（试跑候选值，非塔式标定值） |
| 回抽 | 5 mm @ 40 mm/s，回填 35 mm/s，重启余量 0 |
| 最短回抽行程 | 2 mm |
| 擦嘴 | 2 mm，擦嘴前回抽 70% |
| Z 抬升 | 0.4 mm |
| 桥接速度 / 加速度 / 流量 | 40 mm/s / 1000 mm/s² / 0.95 |
| 严重悬垂墙 | 20 mm/s |
| 接缝 | nearest |
| 最大体积流量（MVS） | 10.5 mm³/s |
| 首层速度 / 加速度 | 25 / 30 mm/s，300 mm/s² |
| 上层速度 | 外壁 126，内壁/稀疏填充 132，实心 120，顶/间隙 105，空驶 190 mm/s |
| 上层加速度 | 外壁/顶 3150，内壁/默认 5250，空驶 7350 mm/s² |

## 文档列表

| 文档 | 内容 |
|---|---|
| [手动调平（无探针·床升降 Z）](docs/manual-bed-leveling.md) | 本机调平安全流程与纸感判定标准 |
| [压力提前（PA）校准](docs/pressure-advance.md) | 方塔校准法与保守 A/B 试跑 |
| [PETG Bowden 桥接与换岛调参](docs/petg-bowden-bridging.md) | 桥接下垂、转角圆角、岛间跳转 |
| [流量上限（MVS）与提速档位](docs/mvs-speed-tier.md) | 提速前的流量核算与 MCU 步进率限制 |
| [OrcaSlicer 预设选择](docs/orca-presets.md) | 预设识别、auto-brim 行为、按对象支撑 |
| [3MF 预设污染与结尾流涎](docs/3mf-preset-pollution.md) | 3MF 内嵌预设链覆盖问题及修复 |
| [Bowden 转换检查清单](docs/bowden-conversion-checklist.md) | 直驱改 Bowden 后需重新核对的项 |

## 调参约定

1. 单次仅调整一类参数（速度、流量、PA、回抽分开验证），避免归因困难。
2. 以生成的 G-code 为准，不以切片器界面显示为准；`SET_PRESSURE_ADVANCE`、`SET_VELOCITY_LIMIT` 需在 G-code 中直接确认。
3. 修改前备份 `printer.cfg`、Orca 预设与当前 G-code。
4. 诊断用参数与生产参数分离保存，诊断完成后恢复生产预设。

## License

MIT
