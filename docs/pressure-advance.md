# 压力提前（PA）校准与保守试跑

PA 解决的是**打印移动加/减速时的挤出压力**问题（转角鼓包、线条起止端），和回抽（解决空驶拉丝）是两回事，不要混为一谈。

## 校准输入能证明什么、不能证明什么

- `rotation_distance` / steps-per-mm 加上指令-实测挤出验证，只能证明**挤出比例**正确（本机 7.7108，20 mm 指令实测 20 mm）。
- PA 还取决于 PTFE 长度/内径/柔顺性、耗材刚性、温度、喷嘴约束、流量和加速度。**不能**从挤出比例算出 PA。
- 回抽管空驶，PA 管打印移动，互相替代不了。

## 官方方塔校准法（Klipper）

用空心方塔（四角可见）。切片器按当前 Klipper 文档设置（恒定 500 mm/s² 加速度、低方角速度），命令放在任何会重置 PA 的 `PRINT_START` 宏**之后**：

```gcode
SET_VELOCITY_LIMIT SQUARE_CORNER_VELOCITY=1 ACCEL=500
# 直驱/短料路：
TUNING_TOWER COMMAND=SET_PRESSURE_ADVANCE PARAMETER=ADVANCE START=0 FACTOR=0.005
# Bowden 长料路用这个：
TUNING_TOWER COMMAND=SET_PRESSURE_ADVANCE PARAMETER=ADVANCE START=0 FACTOR=0.020
```

忽略接缝角。在最佳非接缝角的高度 H（mm）处：

```text
PA = START + H × FACTOR
```

- PA 偏低：转角鼓包
- PA 过高：转角附近变细/缺料
- 到 PA 1.0 仍无明显改善：关掉 PA，查别的原因
- 测试完 `RESTART` 清除调谐状态

> ⚠️ 直驱用 0.005、Bowden 用 0.020，别把直驱系数套到 Bowden 上。

## 命令顺序审计

在生成的 G-code 里搜（别只看切片软件界面）：

1. `PRINT_START ...`
2. 宏内部可能的基线 `SET_PRESSURE_ADVANCE ADVANCE=0`
3. 校准的 `TUNING_TOWER` 或生产的 `SET_PRESSURE_ADVANCE` 必须出现在宏**之后**

勾选了 PA 复选框 ≠ 真的生效，可执行命令顺序正确才算。

## 不想打方塔？做有边界的 A/B 试跑

1. 固定材料、温度、流量、回抽、速度和加速度——**只改 PA 一个变量**。
2. 从机器经验里选一个保守候选值。
3. 打你关心的代表零件。
4. 对比同一批非接缝角。
5. 小步增减；BMG 咔咔响/跳齿或转角缺料就立刻降。

这个结果只能叫**试跑候选值**，不能叫"标定值"。本机当前 PA 0.08 就是这条路出来的。

## 挤出机需求估算（只做数量级筛查）

1.75 mm 耗材：

```text
filament_area = π × (1.75/2)²
E_per_XY ≈ layer_height × line_width / filament_area
steady_E_speed ≈ print_speed × E_per_XY
extra_E_speed ≈ PA × feature_acceleration × E_per_XY
screened_peak_E ≈ steady_E_speed + extra_E_speed
```

用 Orca 圆角矩形珠面积更贴近真实：

```text
bead_area ≈ (line_width − layer_height × (1 − π/4)) × layer_height
E_per_XY = bead_area / filament_area
```

这只是数量级筛查，不是校准模型。本机案例：0.2×0.44 mm 线，E_per_XY≈0.0366；5250 mm/s² 下 PA 0.05 约增加 9.6 mm/s 的 E 需求，PA 0.08 约增加 15.4 mm/s（不含稳态流速）。

**不要拿 `max_extrude_only_velocity` 当 PA 的安全上限。** 那个参数只管纯挤出/回抽移动，PA 运动是耦合在 XY 打印移动上的。高 PA + 高加速度可能让 BMG 吃不消，即使 5 mm 回抽本身很安全。判断时把电机步进率、驱动电流、齿轮比和实际跳齿/咔咔声都算进去。
