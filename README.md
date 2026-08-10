# 3D 打印机笔记

<p align="center">
  <img src="docs/screenshots/avatar.png" width="96" height="96" alt="avatar" style="border-radius:50%" />
</p>

<p align="center">Apollo3D Saturn 10 Plus 的 Klipper 调参笔记。没有探针，没有自动调平，全靠手工。</p>

---

## 这是什么

这台 Apollo3D Saturn 10 Plus 被我改成 Bowden 方案之后，前前后后折腾了一个多月：首层粘不住、桥接下垂、转角起疙瘩、打印到一半被撞掉……每一步都是边查资料边实测试出来的。这里记的是这台机器上的真实参数、验证过的设置和踩过之后才明白的坑。

如果你用的是同款机器（或者类似的床升降 Z、无探针、Bowden 结构），可以直接照着用；如果是别的机器，请当参考，别照抄——**参数是这台机器实测出来的，不是普适的**。

## 机器硬件基线

| 项目 | 值 |
|---|---|
| 机型 | Apollo3D Saturn 10 Plus |
| 主板 | Apollo 定制 ATmega2560（MKS Gen_L 引脚映射） |
| 串口速率 | 250000 baud |
| 运动行程 | 笛卡尔 190×190×151.25 mm |
| 探针 | **无**（无 BLTouch / 无 bed mesh / 无 screws_tilt） |
| Z 轴结构 | 床升降式：**Z=0 是打印位**（喷嘴离床最近），归零时床落到 Z=151.25 |
| 挤出机 | BMG 远端（Bowden），PTFE 管约 500 mm |
| rotation_distance | 7.7108（20 mm 指令实测 20 mm） |
| 固件 | Klipper |

> ⚠️ 这台机器的 Z 轴和常见教程相反：普通教程假设"Z=0 是归零位"，这台是"Z=151.25 是归零位"。照普通教程发 `G1 Z0` 会把喷嘴怼进床里。

## 软件环境

- Klipper + Moonraker：`http://192.168.5.32:7125`（Moonraker 根路径显示 "Welcome to Moonraker" 是正常的，不是故障）
- Mainsail Web 界面：`http://192.168.5.32:8320`
- 切片：OrcaSlicer（MCP fork 版，带 Remote API）

## 当前生产参数（2026-08 实测验证）

### Klipper 端

```ini
max_velocity = 190
max_accel = 7350
max_extrude_only_velocity = 45
max_extrude_only_accel = 3000
```

### Orca 预设（三层各自独立）

- 机器：`MyKlipper 0.4 nozzle - 拷贝`（190×190，Bowden 500mm）
- 耗材：`Generic PETG @System - 拷贝`
- 工艺：`0.20mm PETG Stable @Saturn10Plus`

| 参数 | 值 |
|---|---|
| 首层温度 / 后续 | 240 °C / 235 °C |
| 热床温度 / 后续 | 78 °C / 75 °C |
| 压力提前（PA） | 0.08（启发式试跑值，非标定值） |
| 回抽 | 5 mm @ 40 mm/s，回填 35 mm/s，重启余量 0 |
| 最短回抽行程 | 2 mm |
| 擦嘴 | 2 mm，擦嘴前回抽 70% |
| Z 抬升 | 0.4 mm |
| 桥接速度 | 40 mm/s（内桥同速 100%），桥接加速度 1000 |
| 桥接流量 | 0.95 |
| 严重悬垂墙 | 20 mm/s |
| 接缝 | nearest |
| 最大体积流量（MVS） | 10.5 mm³/s |
| 首层速度 | 25 / 30 mm/s，加速度 300 |
| 上层速度 | 外壁 126，内壁/稀疏填充 132，实心 120，顶/间隙 105，空驶 190 |
| 上层加速度 | 外壁/顶 3150，内壁/默认 5250，空驶 7350 |

### 这套参数的实际效果

- 同一零件从 27m17s 优化到 26m03s，回抽次数从 811 降到 547
- 桥接从下垂+圆角锚点，变成 40/20 mm/s 均匀过渡、无停顿感
- 首层从偶尔掉件，变成稳定粘贴

## 文档导航

| 文档 | 内容 |
|---|---|
| [手动调平（无探针·床升降 Z）](docs/manual-bed-leveling.md) | 这台机器专属的调平安全流程，跟普通教程完全不同 |
| [压力提前（PA）校准与保守试跑](docs/pressure-advance.md) | Bowden 机型的 PA 校准方法，以及不想打塔时的 A/B 试跑 |
| [PETG Bowden 桥接与换岛调参](docs/petg-bowden-bridging.md) | 桥接下垂、转角圆角、看起来像"停顿"的岛间跳转 |
| [流量上限（MVS）与提速档位](docs/mvs-speed-tier.md) | 提速前先算流量账，避免改了速度实际被静默限速 |
| [Orca 预设选择与检查清单](docs/orca-presets.md) | 怎么确认当前用的到底是哪个预设 |
| [3MF 预设污染与结尾流涎](docs/3mf-preset-pollution.md) | 打开网上下载的 3MF 后设置全被覆盖的问题 |

## 使用建议

1. **改参数一次只改一类**。PA、速度、流量、回抽不要同时动，否则出了问题不知道是哪个引起的。
2. **以 G-code 为准，不要看切片软件界面**。切片器显示的值和实际生成的 G-code 经常不一致，`SET_PRESSURE_ADVANCE`、`SET_VELOCITY_LIMIT` 要直接在 G-code 里搜。
3. **先备份再改**。`printer.cfg`、Orca 预设 JSON、当前 G-code 都留一份。
4. **首层慢没问题，上层别锁死**。诊断期间把加速度锁 800 用了一天，打印时间直接翻倍，千万别把诊断参数当成生产参数存进预设。

## License

MIT
