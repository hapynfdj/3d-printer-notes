# OrcaSlicer 预设选择与检查

适用于：确认当前激活工艺、切换预设、排查异常长裙边/边圈。

## 只读探查

1. `list_presets(type="print", include_system=false)`：仅用户预设。
2. `get_status()`：记录当前工艺、机器、耗材、对象列表及切片状态。
3. `get_preset_config(type="printer", name=<当前机器>)`：检查 `default_print_profile`、`machine_max_acceleration_*`、`machine_max_speed_*`、`extruder_type`、`extruder_variant_list`。
4. `get_preset_config(type="print", name=<候选>)`：对比 `compatible_printers`、`default_acceleration`、内外壁/空驶加速度、桥接速度/加速度/流量、`brim_type`/`brim_width`/`skirt_loops`、`enable_support`、`layer_height`。
5. 当前耗材预设：`filament_type`、喷嘴/热床温度、`pressure_advance`、`filament_extruder_variant`、`filament_max_volumetric_speed`。

界面显示 `Default Setting` 不代表用户预设丢失，可能为项目/临时工艺状态。机器预设的 `default_print_profile` 是默认意图的强证据，仍须验证实际激活状态。

## 预设切换

确认目标后执行：

```json
{"type": "print", "name": "<精确的保存工艺名>"}
```

切换后验证：

- `get_status().presets.print` 等于目标
- `get_config` 暴露目标关键值
- `get_slice_status` 达到 `state=done` 后再报告完成
- 切换过程未启动打印或改变 Klipper 运行时状态

预设切换不等同于编辑工艺权限。参数清理（温度、加速度、裙边、支撑）须作为独立变更处理。

## 边圈（skirt）与裙边（brim）的区分

- `skirt_loops`：边圈数
- `brim_type` 与 `brim_width`：裙边生成
- 分解结果中的 `skirt` 与 `brim` 角色及其时间/耗材

5 mm `auto_brim` 在 `skirt_loops=0` 时仍会占据大量外围材料与时间，不应归类为长边圈。需要移除外围粘附几何时：将 brim 模式改为无裙边枚举、裙边圈数置零、重新切片、确认两个角色均消失。仅调整 `brim_width` 无法改变 auto_brim 行为，模式为控制项。

### 本机实测：auto_brim 为"鼠耳"结构

Orca 默认 `brim_type=auto_brim` 在模型尖角/窄端生成"鼠耳"补丁，而非整圈环。94×31×31 长薄条实测：auto_brim 产生 **2.40g / 737s** 裙边；改为 `brim_type=outer_only` + `brim_width=2` 后为 **0.19g / 57s**。双件板节约约 11 分钟与 2.2g。本机偏好：PETG 使用 1–2 mm 外环裙边。

`compare_settings` 注意：对比 `brim_width` 5/3/0 返回相同切片结果（裙边恒为 2.40g），因 auto_brim 宽度非主导因素，且 Orca 复用有效切片缓存（`slice_result_valid=true`）。须直接修改 `brim_type` 并确认配置读回与新切片统计变化。

## 本机预设实证（Apollo3D 会话）

- `0.20mm PETG Stable @Saturn10Plus`：Saturn10 Plus 专属 PETG 工艺；0.20 层、默认/内/稀疏加速度 5250、桥接 40/1000/0.95、支撑关、auto-brim 5mm、skirt 0
- `0.20mm PETG Stable Apollo Bowden (standalone)`：PETG/Bowden 工艺；0.20 层、加速度 5250、桥接 40/1000/0.95、外环 2mm 裙边、支撑关。实时配置中工艺/耗材变体字段仍报 `Direct Drive Standard`，预设名称不能作为 Bowden 链状态的证明
- `0.20mm Standard @MyKlipper - 拷贝`：通用复制工艺；0.20 层、默认加速度 5000、桥接 50、auto-brim 5mm，非 Saturn10 Plus 专属

机器预设 `Apollo3D Saturn10 Plus Bowden (standalone)` 的 `default_print_profile` 指向 `0.20mm PETG Stable @Saturn10Plus`。识别专属工艺以此为准；迁移至其他机型时须重新执行实时检查。

## 新模型加载与多 STL 重叠

新加载的 STL 可能显示为 `Default Setting`，属项目/临时状态。每次加载后：重新选择已批准工艺 → 验证 `get_status().presets.print` → 再切片。`slicing=true` 或 `get_slice_status.state` 未达 done 时不报告完成。

多 STL 同时加载时可能共享默认 XY 中心。先执行 Arrange，再以 `list_objects` 包围盒与编辑器/板渲染验证重叠。多组件 STL（单文件含多块板）在 Orca 中仍为一个对象；需独立摆放或独立工艺时使用单独 STL 文件。
