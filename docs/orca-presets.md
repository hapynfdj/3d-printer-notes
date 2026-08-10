# Orca 预设选择与检查清单

用户问"哪个是我的专属工艺"、"切换到我的配置"或"怎么有条长边/长圈"时用这个。

## 只读探查

1. `list_presets(type="print", include_system=false)`——只看用户预设。
2. `get_status()`——记录当前工艺、机器、耗材、对象列表、是否在切片。
3. `get_preset_config(type="printer", name=<当前机器>)`——看 `default_print_profile`、`machine_max_acceleration_*`、`machine_max_speed_*`、`extruder_type`、`extruder_variant_list`。
4. `get_preset_config(type="print", name=<候选>)`——对比 `compatible_printers`、`default_acceleration`、内外壁/空驶加速度、桥接速度/加速度/流量、`brim_type`/`brim_width`/`skirt_loops`、`enable_support`、`layer_height`。
5. 读当前耗材预设的 `filament_type`、喷嘴/热床温度、`pressure_advance`、`filament_extruder_variant`、`filament_max_volumetric_speed`。

界面显示 `Default Setting` **不代表你的用户预设丢了**，可能只是项目/临时工艺。机器预设的 `default_print_profile` 是默认意图的强证据，但仍要验证实际激活状态。

## 显式选择

用户点名或同意目标后：

```json
{"type": "print", "name": "<精确的保存工艺名>"}
```

调用 `select_preset` 后验证：

- `get_status().presets.print` 等于目标
- `get_config` 暴露目标的关键值
- `get_slice_status` 到 `state=done` 再报告完成
- 选择过程没有启动打印或改变 Klipper 运行时状态

**选择 ≠ 允许编辑工艺。** 任何清理（温度、加速度、边、支撑）都是单独的、经批准的变更集。

## 边（skirt）和裙边（brim）是两回事

切片分解里可能同时有两者。检查：

- `skirt_loops`：边圈数
- `brim_type` 和 `brim_width`：裙边生成
- 分解角色 `skirt` 和 `brim` 的时间和耗材

5 mm `auto_brim` 即使 `skirt_loops=0` 也会占据大量可见外围材料和时间——别把它报成"一条长边"。用户不要外围粘附几何时：改 brim 模式为真正无裙边的枚举、裙边圈数设 0、重新切片、确认两个角色都消失。**只改 `brim_width` 修不了 auto_brim 行为，模式才是控制项。**

### 本机实测：auto_brim 是"鼠耳"，不是简单一圈

Orca 默认 `brim_type=auto_brim` 会在模型每个尖角/窄端生成"鼠耳"补丁，不是一圈干净的环。94×31×31 的长薄条变成一片耳朵海：实测 **2.40g / 737s** 的裙边 vs 改 `brim_type=outer_only` + `brim_width=2` 后的 **0.19g / 57s**。双件板省约 11 分钟和 2.2g。本机偏好：PETG 裙边 1–2 mm 简单外环，首层稳定后不要耳朵。

`compare_settings` 坑：对比 `brim_width` 5 / 3 / 0 返回**完全相同**的切片结果（裙边卡在 2.40g），因为 auto_brim 的宽度不是主导因素，且 Orca 复用了有效切片缓存（`slice_result_valid=true`）。要看真实差异必须改 `brim_type` 本身，再确认配置读回和新切片统计都变了。

## 本机预设实证（Apollo3D 会话）

活切片器暴露的用户工艺：

- `0.20mm PETG Stable @Saturn10Plus`——Saturn10 Plus 专属 PETG 工艺；0.20 层、默认/内/稀疏加速度 5250、桥接 40/1000/0.95、支撑关、auto-brim 5mm、skirt 0
- `0.20mm PETG Stable Apollo Bowden (standalone)`——PETG/Bowden 命名工艺；0.20 层、加速度 5250、桥接 40/1000/0.95、外环 2mm 裙边、支撑关。它的工艺/耗材变体字段在实时配置里仍报 `Direct Drive Standard`——**名字不能当干净 Bowden 链的证明**
- `0.20mm Standard @MyKlipper - 拷贝`——通用 MyKlipper 复制工艺；0.20 层、默认加速度 5000、桥接 50、auto-brim 5mm、喷嘴兼容更广。不是 Saturn10 Plus 专属工艺

当前机器预设是 `Apollo3D Saturn10 Plus Bowden (standalone)`，它的 `default_print_profile` 指向 `0.20mm PETG Stable @Saturn10Plus`——这就是识别专属工艺的方式。换机器时永远重跑实时检查，别硬编码这些名字。

## 新模型加载后工艺被重置、多 STL 重叠

新加载的 STL 可能显示为 `Default Setting`，即使保存的用户工艺还在。这是项目/临时状态，不是预设被删。每次加载新模型后：重新选择已批准的工艺 → 验证 `get_status().presets.print` → 再切片。`slicing=true` 或 `get_slice_status.state` 未到 done 时不要报完成。

多个 STL 同时加载时 Orca 可能把它们放在同一个默认 XY 中心。先 Arrange 再判断几何重叠，然后用 `list_objects` 包围盒 + 编辑器/板渲染验证。多组件 STL（比如一个文件里含左右两块墙板）在 Orca 里仍是一个对象；需要独立摆放或独立工艺时用单独 STL 文件。
