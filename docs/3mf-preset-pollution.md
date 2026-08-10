# 3MF 预设污染与结尾流涎

## 症状

从外部下载的 `.3mf` 项目（如 Bambu MakerWorld）打开并打印后：各停点/接缝出现毛刺、停靠位流涎、G-code 页头温度与已调优预设不符。已保存的机器/耗材预设显示异常，尽管保存时配置正确。

## 根因：3MF 内嵌预设链

OrcaSlicer 打开 `.3mf` 项目时切换至文件内嵌的预设链。Bambu 生态文件典型内嵌值：

- 机器：`Bambu Lab P1S 0.4 nozzle`（256×256 床、Bambu 限制）
- 耗材：`Generic PLA` 或 Bambu PETG @ 255 °C
- 回抽：0.8 mm @ 30 mm/s（直驱值）、擦嘴 1 mm、最短回抽行程 1 mm
- 机器限制：max speed 500/200、accel 20000/20000、E 速度 25

用户预设被静默覆盖。证据链（本会话）：

1. Moonraker 实时查询运行中打印：挤出 255 °C、PA 0。
2. Moonraker 历史：同名文件 + 相同 UUID 的"成功"任务与一次误重启为同一 G-code 文件。
3. G-code 页头/页脚审计：`nozzle_temperature 255`、`retraction_length 0.8`、`retraction_speed 30`、`deretraction_speed 30`、`wipe_distance 1`、`retraction_minimum_travel 1`、`printer_extruder_variant "Direct Drive Standard"`、`machine_max_speed_x 500,200`。
4. Orca `get_config` 与 `get_preset_config` 均显示污染值：预设文件被 3MF 加载覆盖，而非仅项目级覆盖。

## 修复：独立预设（detach）

创建不继承污染基座的预设副本：

1. `set_config` 实时写入正确值（Bowden 5@40/35、最短行程 2、擦嘴 2、XY 190/7350、E 45/3000、PETG 240/235、床 78/75、PA 0.08、MVS 10.5）。
2. `save_preset(type=printer|filament|print, name="... (standalone)", detach=True)`：生成无 `inherits` 链回 Bambu 基座的预设。
3. `edit_preset` 写入完整正确值（数组字段以逗号分隔字符串提交，避免 HTTP 422）。
4. 三个预设均 `select_preset`，执行 `check_profile_physics`（裁决不可为 blocked），`get_config` 读回验证。
5. 使用规范：打开任何 `.3mf` 后、切片前重新选择独立预设；直接导入裸 STL 可完全规避。

每次持久化保存前执行 `check_profile_physics`；编辑前备份三个预设 JSON。

## 结尾流涎：主动挤出与被动残余压力

判定：挤出机电机仍推进耗材，或停靠后熔体在残余压力下滴出。

- G-code 结尾块：最后墙 → 回抽 `E-.56` → 擦嘴 → Z 抬升 → 下一岛 `E.8` 回填 → 最终回抽 → `PRINT_END`。
- 本机 Klipper `PRINT_END`：条件 `M83/G92 E0/G1 E-2.0 F1800`，随后 `M104 S0`、`M140 S0`、Z 抬升、停靠 `G1 X5 Y185 F6000`、`M107`、`G92 E0`。
- 最后墙之后无正的 E 移动：非主动挤出。停靠位流涎为被动行为——喷嘴温度（255 °C）维持熔体流动性，残余熔腔压力持续排料直至热端冷却。
- 处理：纠正温度（PETG 235 °C），而非加长结尾回抽。检查完成前不应归因于宏或 Klipper。

## 模型尺寸验证链

怀疑打印尺寸与模型不符时：

1. 真实网格尺寸：3MF 为 ZIP 容器。解析 `3D/3dmodel.model`（单位、构建项）及各 `3D/Objects/object_N.model` XML `<vertex x y z>` 计算真实包围盒（Bambu 3MF 网格存于独立 .model XML）。
2. Orca 对象列表 `size_mm`。
3. 实际打印足迹：G-code `EXCLUDE_OBJECT_DEFINE ... POLYGON=[[...]]` 提供精确 XY 轮廓。

三者一致（本机 70×70×3 圆盘案例）→ 切片比例正确。尺寸不符指向 XY steps/mm（步进电机 `rotation_distance`），与切片器无关。

注意：旋转对象会增大轴对齐包围盒。28×108 mm 零件旋转约 35° 后对象列表显示约 84.6×84.6，属方向变化而非比例变化。

## Orca MCP 工具说明

- `edit_preset` 需要 `type`+`name`；`set_config` 为原子操作（任一非法 key 全部不生效）。
- `save_preset(detach=True)` 为脱离污染继承链的出口。
