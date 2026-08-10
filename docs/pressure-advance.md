# 压力提前（PA）校准

压力提前（Pressure Advance）用于补偿打印移动加减速过程中的挤出压力偏差，主要影响转角鼓包与线条起止端。与回抽（空驶拉丝控制）作用阶段不同，两者不可互相替代。

## 校准前提

- `rotation_distance` 或 steps/mm 配合指令-实测挤出验证，仅能证明挤出比例正确（本机 7.7108，20 mm 指令实测 20 mm）。
- PA 与 PTFE 长度/内径/柔顺性、耗材刚性、温度、喷嘴约束、流量及加速度相关，无法由挤出比例推导。
- 方塔校准在任一重置 PA 的 `PRINT_START` 宏之后执行。

## 方塔校准法

使用空心方塔（四角可见），切片器按 Klipper 官方文档设置（恒定 500 mm/s² 加速度、低方角速度）：

```gcode
SET_VELOCITY_LIMIT SQUARE_CORNER_VELOCITY=1 ACCEL=500
# 直驱 / 短料路：
TUNING_TOWER COMMAND=SET_PRESSURE_ADVANCE PARAMETER=ADVANCE START=0 FACTOR=0.005
# Bowden 长料路：
TUNING_TOWER COMMAND=SET_PRESSURE_ADVANCE PARAMETER=ADVANCE START=0 FACTOR=0.020
```

忽略接缝角。取最佳非接缝角高度 H（mm）：

```text
PA = START + H × FACTOR
```

判定：

- PA 偏低：转角鼓包
- PA 过高：转角附近缺料/变细
- PA 至 1.0 仍无明显改善：关闭 PA，排查其他原因

测试完成后执行 `RESTART` 清除调谐状态。

> 直驱系数 0.005、Bowden 系数 0.020，不可互换。

## 命令顺序审计

以 G-code 实际内容为准，而非切片器界面：

1. `PRINT_START ...`
2. 宏内部基线 `SET_PRESSURE_ADVANCE ADVANCE=0`（如有）
3. 校准 `TUNING_TOWER` 或生产 `SET_PRESSURE_ADVANCE` 须位于宏之后

勾选 PA 复选框不代表生效，须确认可执行命令顺序正确。

## A/B 试跑法（不执行方塔时）

1. 固定材料、温度、流量、回抽、速度与加速度，仅调整 PA。
2. 依据机型经验选取保守候选值。
3. 打印代表性零件。
4. 对比同一批非接缝角。
5. 小步增减；出现 BMG 跳齿/咔嗒声或转角缺料时立即回退。

结果仅作为试跑候选值（本机当前 PA 0.08 即由此得出），不等同于塔式标定值。

## 挤出机需求估算

1.75 mm 耗材：

```text
filament_area = π × (1.75/2)²
E_per_XY ≈ layer_height × line_width / filament_area
steady_E_speed ≈ print_speed × E_per_XY
extra_E_speed ≈ PA × feature_acceleration × E_per_XY
screened_peak_E ≈ steady_E_speed + extra_E_speed
```

使用 Orca 圆角矩形珠面积更贴近实际：

```text
bead_area ≈ (line_width − layer_height × (1 − π/4)) × layer_height
E_per_XY = bead_area / filament_area
```

该计算仅作数量级筛查，不构成校准模型。本机案例：0.2×0.44 mm 线，E_per_XY≈0.0366；5250 mm/s² 下 PA 0.05 增加约 9.6 mm/s 的 E 需求，PA 0.08 增加约 15.4 mm/s（不含稳态流速）。

`max_extrude_only_velocity` 仅约束纯挤出/回抽移动，PA 运动耦合于 XY 打印移动，不可作为 PA 安全上限。判断时须综合电机步进率、驱动电流、齿轮比与实测跳齿/咔嗒声。
