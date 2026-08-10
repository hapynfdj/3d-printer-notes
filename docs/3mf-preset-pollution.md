# 3MF 预设污染与结尾流涎

## 症状

从网上下载的 `.3mf`（比如 Bambu MakerWorld 的模型）打开打印后：每个停点/接缝都有毛刺、结束停靠位流涎、页头温度不再匹配调好的预设。调好的机器/耗材预设看起来"坏了"，尽管保存是对的。

## 根因：3MF 内嵌自己的预设链

OrcaSlicer 打开 `.3mf` 项目会切到文件内嵌的预设链。Bambu 生态文件内嵌的典型值：

- 机器：`Bambu Lab P1S 0.4 nozzle`（256×256 床、Bambu 限制）
- 耗材：`Generic PLA` 或 Bambu PETG 在 **255 °C**
- 回抽：**0.8 mm @ 30 mm/s**（直驱值！）、擦嘴 1 mm、最短回抽行程 1 mm
- 机器限制：max speed 500/200、accel 20000/20000、E 速度 25

调好的用户预设被悄悄丢弃。证据链（本次会话）：

1. Moonraker 实时查询运行中打印：挤出 255 °C、PA 0。
2. Moonraker 历史：同名文件 + 相同 UUID 的"成功"任务和一次误重启 → 是同一个 G-code 文件。
3. G-code 页头/页脚审计：`nozzle_temperature 255`、`retraction_length 0.8`、`retraction_speed 30`、`deretraction_speed 30`、`wipe_distance 1`、`retraction_minimum_travel 1`、`printer_extruder_variant "Direct Drive Standard"`、`machine_max_speed_x 500,200`。
4. Orca `get_config` 和 `get_preset_config` 都显示污染值 → 预设文件本身被 3MF 加载覆盖了，不只是项目覆盖。

## 修复：分离式独立预设

创建**不继承**污染基座的调好预设副本，防止再次污染：

1. `set_config` 实时写正确值（Bowden 5@40/35、最短行程 2、擦嘴 2、XY 190/7350、E 45/3000、PETG 240/235、床 78/75、PA 0.08、MVS 10.5）。
2. `save_preset(type=printer|filament|print, name="... (standalone)", detach=True)` → 创建没有 `inherits` 链回 Bambu 基座的预设。
3. `edit_preset` 把完整正确值写进每个新预设（数组会 HTTP 422 → 用逗号分隔字符串）。
4. 三个都 `select_preset`，`check_profile_physics`（裁决不能是 blocked），`get_config` 读回验证。
5. 使用纪律：打开任何 `.3mf` 后，切片前重新选择独立预设；直接导入裸 STL 完全避开此问题。

每次持久化保存前跑 `check_profile_physics`；编辑前备份三个预设 JSON。

## 结尾流涎：主动挤料 vs 被动残余压力

要回答的问题：挤出机电机还在推料，还是停靠后熔体在残余压力下滴出？

- 读 G-code 结尾块：最后墙 → 回抽 `E-.56` → 擦嘴 → Z 抬升 → 下一岛 `E.8` 回填，然后最终回抽和 `PRINT_END`。
- 本机 Klipper `PRINT_END`：条件 `M83/G92 E0/G1 E-2.0 F1800`，然后 `M104 S0`、`M140 S0`、Z 抬升、停靠 `G1 X5 Y185 F6000`、`M107`、`G92 E0`。
- 最后墙之后**没有正的 E 移动** → 不是主动挤料。停靠位流涎是被动的：喷嘴温度高（255 °C）保持熔体流动，残余熔腔压力把料推出来直到热端冷却。
- 修法：纠正温度（PETG 235 °C），而不是把结尾回抽加长到几 mm 以上。做这个检查之前，别怪"我们的代码"或 Klipper。

## 模型尺寸验证链（切片尺寸 vs 打印尺寸）

怀疑打印尺寸和模型不符时：

1. 真实网格尺寸：3MF 是 ZIP。解析 `3D/3dmodel.model`（单位、构建项）和每个 `3D/Objects/object_N.model` XML `<vertex x y z>` 算真实包围盒（Bambu 3MF 的网格存成独立 .model XML，不是 .stl）。
2. Orca 对象列表 `size_mm`。
3. 实际打印足迹：G-code `EXCLUDE_OBJECT_DEFINE ... POLYGON=[[...]]` 给出精确 XY 轮廓，比如 X60..130 / Y60..130 = 70 mm 圆盘。

三者一致（70×70×3）→ 切片比例正确。真的尺寸不符，问题指向 XY steps/mm（步进电机 `rotation_distance`），不是切片器；去问实测值。

坑：旋转对象会膨胀轴对齐包围盒。28×108 mm 零件旋转约 35° 在 Orca 对象列表显示约 84.6×84.6——这是方向，不是比例变化。别叫它尺寸不符。

## Orca MCP 工具备注

- `edit_preset` 需要 `type`+`name`；`set_config` 是原子的（任一非法 key 全部不生效）。
- `save_preset(detach=True)` 是从污染继承链逃生的出口。
