---
tags:
  - UE5
  - CitySample
  - RuleProcessor
  - 程序化城市
status: 阶段四已完成
project: F:\CitySample
map: /Game/LearningLab/Maps/L_01_CityGeneration_Lab
---

# City Sample 学习笔记

> 当前里程碑：已完成建筑主体、碰撞、Roof Geo、Rooftop Biome，并用 `Override Objects Map` 将原屋顶蓝图替换为自定义 Packed Level Actor。

![[assets/CitySample/04-single-building-result-front.png]]

## 学习路线

1. Rule Processor 与 Small City 建筑生成（当前）
2. 建筑碰撞、Roof Geo 与 Rooftops（已完成）
3. Houdini 点云及其元数据来源
4. World Partition、Data Layer 与 HLOD
5. Mass Traffic、ZoneGraph 与车辆
6. Mass Crowd 与人物表现层
7. 使用 UE5 原生 PCG 复刻相同思想

---

# 01｜从点云生成一栋完整建筑

## 本节结论

City Sample 的城市生成不是把“一栋完整建筑模型”直接拖进关卡，而是：

```text
Houdini 输出的城市数据
        ↓
Point Cloud（大量模块点与元数据）
        ↓
Rule Processor 按 Metadata 过滤
        ↓
根据 Building_ID 对模块分组
        ↓
Multi Actor 创建建筑 Actor／实例组件
```

本次使用的点云：

```text
/Game/City/Small_City/PBC/CITY_buildings
```

点云共包含 `562433` 个点。每个点可以理解为一个待生成模块的“数据记录”，其中保存位置、旋转、缩放、引用资产和分类信息。

---

## 一、是什么（What）

### 1. Rule Processor 是什么

Rule Processor 是 City Sample 工程中的点云规则处理系统，不是 UE5 原生的 PCG Framework。它读取 Point Cloud 中的点，然后利用规则树进行筛选、分支、分组和生成。

规则编辑器中：

- 浅黄色 `Metadata`：读取一个元数据字段并过滤点。
- `Matches Filter`：满足条件的数据继续进入该分支。
- `Unmatched`：不满足条件的数据继续进入该分支。
- 橙色 `Multi Actor`：根据元数据对点分组，并创建 Actor／实例组件。
- 左侧复选框：控制该规则是否参与执行。
- 缩进关系：代表数据从父规则流向子规则。

### 2. 本次使用的关键 Metadata

| Metadata | 用途 |
|---|---|
| `Building_ID` | 标识点属于哪一栋建筑，是本次选择完整建筑的关键。 |
| `unreal_instance` | 指向点需要实例化的 Unreal 资产；原规则用它排除特定 arbitrary roof 模块。 |
| `type` | 区分 `building`、`building_collision` 等数据类型。 |
| `CollisionEnabled` | 区分需要 L1 Collision 的可见模块与普通可见模块。 |

### 3. 为什么一栋建筑生成了两个 Actor

报告显示 `Building_ID = 147000` 的建筑主体被原规则进一步分成：

| 分支 | 模块数 | 生成结果 |
|---|---:|---|
| `CollisionEnabled = 1` | 54 | `BLDG_L1_COL_N147000` |
| `CollisionEnabled != 1` | 1015 | `BLDG_N147000` |

这两个 Actor 不是重复建筑，而是同一栋建筑的互补模块集合。它们拥有相同的 `Building_ID`，但根据碰撞需求被放进不同的实例组件／Actor 中。

---

## 二、为什么这样做（Why）

### 1. 为什么不直接修改原始规则

原始资产属于 City Sample Demo，其他地图和生成流程可能仍然引用它。直接修改会造成长期依赖和难以排查的错误。

因此创建了独立实验目录：

```text
/Game/LearningLab/
├─ Maps/
├─ Experiments/
├─ Materials/
├─ Blueprints/
└─ Notes/
```

本次规则副本：

```text
/Game/LearningLab/Experiments/RP_01_Buildings
```

### 2. 为什么第一次用 Bounding Box 会生成残缺建筑

第一次实验在规则树最外层添加了 Bounding Box，只允许一定坐标范围内的点继续执行。结果建筑出现缺面、缺模块和被切开的现象：

![[assets/CitySample/01-bounding-box-clipped.png]]

原因不是模型损坏，而是执行顺序：

```text
错误思路：先按空间范围裁切模块点 → 再按 Building_ID 分组
```

Bounding Box 过滤的是单个点／单个建筑模块，不认识“一整栋建筑”。一栋建筑跨越边界时，盒子内的模块被保留，盒子外的模块被删除，之后 `Multi Actor` 只能把剩余模块拼成一栋残缺建筑。

### 3. 为什么改用 Building_ID

`Building_ID` 是建筑的语义身份。使用它筛选时，会取得该建筑对应的全部点，再让原始规则继续处理这些点：

```text
正确思路：先选中完整 Building_ID → 再分类 → 再分组生成
```

因此无论模块位于什么空间范围，只要它属于 `147000`，就会一起进入后续规则，不会被空间边界切断。

### 4. 为什么先运行 Report 再生成

生成之前先查看 Report，可以在不创建 Actor 的情况下验证：

- 筛选是否命中数据；
- 命中了多少点；
- 点进入了哪个分支；
- 最终预计创建几个 Actor；
- 是否存在拼写错误或零匹配。

这是 Rule Processor 中成本最低、最安全的调试方式。

---

## 三、怎么做（How）

### 步骤 1：准备独立实验地图和规则副本

- 实验地图：`/Game/LearningLab/Maps/L_01_CityGeneration_Lab`
- 可见建筑规则：`/Game/LearningLab/Experiments/RP_01_Buildings`
- 碰撞规则：`/Game/LearningLab/Experiments/RP_02_BuildingCollision`
- 点云：`/Game/City/Small_City/PBC/CITY_buildings`

实验始终操作副本，原始 `Small_City_FULL_buildings` 保持不变。

### 步骤 2：将点云和规则映射到 RuleProcessor

在关卡的 Python 输入框执行：

```python
import unreal
world = unreal.get_editor_subsystem(unreal.UnrealEditorSubsystem).get_editor_world()
manager = unreal.SliceAndDiceManager.create_slice_and_dice_manager(world)
pc = unreal.load_asset("/Game/City/Small_City/PBC/CITY_buildings")
visible = unreal.load_asset("/Game/LearningLab/Experiments/RP_01_Buildings")
collision = unreal.load_asset("/Game/LearningLab/Experiments/RP_02_BuildingCollision")
manager.find_or_add_mapping(pc, visible)
manager.find_or_add_mapping(pc, collision)
unreal.log(manager.run_report())
```

本实验使用 Actor Label 为 `RuleProcessor` 的管理器；地图模板中其他 `RuleProcessor_Audio`、`RuleProcessor_City`、`RuleProcessor_Freeway` 不属于本实验。

### 步骤 3：添加单栋建筑过滤器

在 `RP_01_Buildings` 中添加新的顶层 `Metadata`：

| 参数 | 值 |
|---|---|
| Label | `Single Building 147000` |
| Key | `Building_ID` |
| Value | `147000` |
| Filter Type | `Matches the given value exactly` |

![[assets/CitySample/02-building-id-filter-settings.png]]

### 步骤 4：把原始规则放入 Matches Filter

将原来最上方的完整 `Metadata` 规则树拖入新节点的 `Matches Filter`。禁用的 `Metadata: Module occluded` 保持独立，不需要移动。

最终结构：

```text
Metadata：Module occluded（禁用）

Metadata：Single Building 147000
├─ Matches Filter
│  └─ Metadata：排除 *SM_arbitrary_roof_*
│     └─ Unmatched
│        └─ Metadata：type = building
│           └─ Matches Filter
│              └─ Metadata：CollisionEnabled = 1
│                 ├─ Matches Filter → Multi Actor
│                 └─ Unmatched → Multi Actor：Building no Col
└─ Unmatched（空）
```

![[assets/CitySample/03-filter-wrapped-rule.png]]

### 步骤 5：运行详细报告

```python
import unreal
world = unreal.get_editor_subsystem(unreal.UnrealEditorSubsystem).get_editor_world()
manager = [m for m in unreal.SliceAndDiceManager.get_slice_and_dice_managers(world)
           if m.get_actor_label() == "RuleProcessor"][0]
unreal.log(manager.run_report(unreal.PointCloudReportLevel.VALUES))
```

本次验证结果：

```text
Building_ID = 147000                 → 1138 points
unreal_instance 非 arbitrary roof    → 1138 points
type = building                     → 1069 points
CollisionEnabled = 1                → 54 modules  → 1 Actor
CollisionEnabled != 1               → 1015 modules → 1 Actor
```

关键判断：匹配数不是 0，并且最终预计生成两个 Actor，说明筛选和分组规则有效。

### 步骤 6：执行可见建筑生成

```python
import unreal
world = unreal.get_editor_subsystem(unreal.UnrealEditorSubsystem).get_editor_world()
manager = [m for m in unreal.SliceAndDiceManager.get_slice_and_dice_managers(world)
           if m.get_actor_label() == "RuleProcessor"][0]
pc = unreal.load_asset("/Game/City/Small_City/PBC/CITY_buildings")
rules = unreal.load_asset("/Game/LearningLab/Experiments/RP_01_Buildings")
mapping = manager.find_mapping(pc, rules)
unreal.log("Generation result: " + str(manager.run_rules_on_mappings([mapping])))
```

成功标志：

```text
Generation result: True
```

Outliner 中出现：

```text
BLDG_L1_COL_N147000
BLDG_N147000
```

### 步骤 7：验证最终结果

在 Outliner 搜索 `147000`，选中任意生成 Actor 后按 `F` 聚焦。两个 Actor 共同组成完整建筑主体：

![[assets/CitySample/04-single-building-result-front.png]]

![[assets/CitySample/05-single-building-result-overview.png]]

---

## 安全回滚：删除规则生成的 Actor

不要手动逐个删除生成 Actor。通过对应 Mapping 清理，后续可以随时重新生成：

```python
import unreal
world = unreal.get_editor_subsystem(unreal.UnrealEditorSubsystem).get_editor_world()
manager = [m for m in unreal.SliceAndDiceManager.get_slice_and_dice_managers(world)
           if m.get_actor_label() == "RuleProcessor"][0]
pc = unreal.load_asset("/Game/City/Small_City/PBC/CITY_buildings")
rules = unreal.load_asset("/Game/LearningLab/Experiments/RP_01_Buildings")
mapping = manager.find_mapping(pc, rules)
unreal.log("Cleanup result: " + str(manager.delete_managed_actors_from_mapping(mapping, True)))
```

---

## 容易混淆的地方

1. `BLDG_L1_COL_N147000` 与 `BLDG_N147000` 不是重复生成，而是同栋建筑的两类互补模块。
2. Bounding Box 适合裁出空间区域，但不保证保留区域边缘的完整语义对象。
3. 想研究完整对象时优先用稳定 ID；想研究某个空间切片时才使用 Bounding Box。
4. `Label` 主要用于编辑器阅读，真正决定过滤的是 `Key`、`Value` 和 `Filter Type`。
5. 生成建筑主体成功，不代表碰撞、屋顶附属物、街道、地面和 HLOD 已全部生成；它们属于后续阶段。

---

## 复习卡片

**Q：为什么 Bounding Box 会把建筑切坏？**  
A：它在 `Building_ID` 分组之前按点过滤，边界外的模块点会被丢弃。

**Q：为什么 `Building_ID` 可以保留完整建筑？**  
A：它表达点属于哪栋建筑，筛选同一个 ID 会取得该建筑的全部相关点。

**Q：为什么一个 Building_ID 最后出现两个 Actor？**  
A：原规则按照 `CollisionEnabled` 把模块分成两个互补集合，并分别生成 Actor。

**Q：为什么生成前要运行 Report？**  
A：它能在创建 Actor 前验证匹配数量、分支流向和预计 Actor 数量，避免昂贵的错误生成。

**Q：Rule Processor 与 UE5 原生 PCG 是同一个系统吗？**  
A：不是。Rule Processor 是 City Sample 使用的点云规则处理系统；原生 PCG 使用 PCG Graph。

---

# 02｜为单栋建筑补充碰撞

## 是什么（What）

City Sample 将部分建筑碰撞保存为独立的点云记录。它们仍在 `CITY_buildings` 点云中，但其分类是：

```text
type = building_collision
```

本次使用规则：

```text
/Game/LearningLab/Experiments/RP_02_BuildingCollision
```

最终生成的独立碰撞 Actor：

```text
BLDG_COLL_N147000
```

## 为什么独立生成碰撞（Why）

可见建筑由大量门窗、墙面和装饰模块组成。如果直接使用所有渲染几何参与复杂碰撞，计算和维护成本都会很高。独立碰撞规则可以使用数量更少、结构更简单的几何体近似建筑主体。

本次 Report 只找到两个 `building_collision` 模块，但它们能够组成覆盖整栋建筑的简化碰撞外壳。这说明碰撞模块数量不需要与可见模块数量相同。

碰撞和可见建筑分开生成也便于：

- 单独启用、隐藏和调试碰撞；
- 使用不同的碰撞复杂度和响应设置；
- 避免渲染模块的细碎几何增加物理查询成本；
- 在 World Partition 中独立管理生成资产。

## 怎么做（How）

### 1. 移除空间裁切

先将 `Metadata: type collision` 及其 `Multi Actor: Collision` 从 Bounding Box 中移出，再删除空的 Bounding Box。原因与可见建筑相同：空间盒会在分组之前裁切点，不能保证语义对象完整。

### 2. 使用同一个 Building_ID

规则结构：

```text
Metadata：Single Building 147000
├─ Key = Building_ID
├─ Value = 147000
└─ Matches Filter
   └─ Metadata：type collision
      ├─ Key = type
      ├─ Value = building_collision
      └─ Matches Filter
         └─ Multi Actor：Collision
```

这里两个过滤条件不能填反：

| 层级 | Key | Value | 含义 |
|---|---|---|---|
| 第一层 | `Building_ID` | `147000` | 选择目标建筑。 |
| 第二层 | `type` | `building_collision` | 从该建筑的数据中选择碰撞模块。 |

一次错误配置把 `147000` 填到了第二层，导致第一层报告显示 `Filter=()`、匹配点数为 0。修正 Key/Value 后恢复正常。

### 3. 用 Report 验证

最终报告：

```text
Building_ID = 147000          → 1138 points
type = building_collision     → 2 modules
Tentative actor count         → 1
生成名称                       → BLDG_COLL_N147000
```

这说明 1138 个属于该建筑的点中，有两个点代表独立碰撞模块，并会组合成一个碰撞 Actor。

### 4. 单独生成碰撞 Mapping

```python
import unreal
world = unreal.get_editor_subsystem(unreal.UnrealEditorSubsystem).get_editor_world()
manager = [m for m in unreal.SliceAndDiceManager.get_slice_and_dice_managers(world)
           if m.get_actor_label() == "RuleProcessor"][0]
pc = unreal.load_asset("/Game/City/Small_City/PBC/CITY_buildings")
rules = unreal.load_asset("/Game/LearningLab/Experiments/RP_02_BuildingCollision")
mapping = manager.find_mapping(pc, rules)
unreal.log("Collision generation result: " + str(manager.run_rules_on_mappings([mapping])))
```

成功输出：

```text
Collision generation result: True
```

此时同一栋建筑由三个 Actor 共同组成：

```text
BLDG_N147000             普通可见模块
BLDG_L1_COL_N147000      带 L1 Collision 标记的可见模块
BLDG_COLL_N147000        独立简化碰撞模块
```

### 5. 可视化验证

在 Outliner 选择 `BLDG_COLL_N147000`，按 `F` 聚焦，再使用 `Alt + C` 或 `Show → Collision` 显示碰撞。碰撞外壳与建筑主体正确对齐：

![[assets/CitySample/06-building-collision-visualization.png]]

图中的大型外轮廓覆盖建筑主体，底部轮廓来自第二个碰撞模块。跨越表面的对角线属于简化网格的三角面边线，不表示模型破损。

## 本阶段结论

- 可见建筑主体生成成功。
- 独立 `building_collision` 数据生成成功。
- 三个 Actor 使用相同 `Building_ID` 并在空间中正确对齐。
- 碰撞可视化结果与建筑主体吻合。

---

# 03｜独立 Roof Geo 与可选屋顶分支

## 是什么（What）

City Sample 的建筑屋顶并不只有一种组成方式。建筑主体规则会先排除名称匹配下面模式的特殊屋顶模块：

```text
unreal_instance = *SM_arbitrary_roof_*
```

这些模块由独立的 Roof Geo 规则重新生成，便于单独管理：

```text
/Game/LearningLab/Experiments/RP_03_BuildingRoofGeo
```

此外，项目还存在另一条屋顶附属物分支：

```text
type = building_rooftop_biom
```

它由 `RP_04_BuildingRooftops` 使用 `Spawn Blueprint` 生成。Roof Geo 和 Rooftop Biome 是两类不同数据，某栋建筑可以有其中一种、两种或都没有。

## 为什么要分开（Why）

建筑墙体、特殊屋顶几何和屋顶蓝图附属物具有不同的生成方式与用途：

- 建筑主体由大量静态网格模块组成；
- Roof Geo 是特殊屋顶静态网格，可以独立显示、隐藏和替换；
- Rooftop Biome 通过蓝图生成，适合承载更复杂的屋顶设施或程序化逻辑。

主体规则排除 `SM_arbitrary_roof_*`，Roof Geo 规则再将其补回，可以避免同一特殊屋顶被主体规则和屋顶规则重复生成。

本次先测试 `Building_ID = 147000`：

- Roof Geo 报告中没有该 ID；
- Rooftop Biome 在该 ID 内匹配 0 个点；
- 因此 147000 的屋顶已经包含在普通建筑模块中，不需要额外屋顶 Actor。

为了观察独立 Roof Geo，改用报告中确实存在的 `Building_ID = 147001`。

## 怎么做（How）

### 1. 创建 147001 的建筑主体实验

复制 `RP_01_Buildings` 为：

```text
/Game/LearningLab/Experiments/RP_05_Buildings_147001
```

只把最外层过滤器改成：

```text
Key   = Building_ID
Value = 147001
```

报告结果：

```text
Building_ID = 147001          → 1896 points
排除 arbitrary roof 后         → 1895 points
type = building               → 1381 points
CollisionEnabled = 1          → 60 modules  → BLDG_L1_COL_N147001
其余可见模块                    → 1321 modules → BLDG_N147001
```

1896 与 1895 相差的 1 个点，正是交给 Roof Geo 规则处理的特殊屋顶记录。

### 2. 限定 Roof Geo 到同一栋建筑

在 `RP_03_BuildingRoofGeo` 最外层增加：

```text
Metadata: Single Building 147001
└─ Key = Building_ID
└─ Value = 147001
   └─ Matches Filter
      └─ Metadata: Select rooftops
         └─ unreal_instance 匹配 *SM_arbitrary_roof_*
            └─ Metadata: building
               └─ type = building
```

报告确认只有 1 个 Roof Geo 点，并生成：

```text
BLDG_ROOFGEO_N147001
```

### 3. 同时生成主体与 Roof Geo

```python
import unreal
world = unreal.get_editor_subsystem(unreal.UnrealEditorSubsystem).get_editor_world()
manager = [m for m in unreal.SliceAndDiceManager.get_slice_and_dice_managers(world)
           if m.get_actor_label() == "RuleProcessor"][0]
pc = unreal.load_asset("/Game/City/Small_City/PBC/CITY_buildings")
body = unreal.load_asset("/Game/LearningLab/Experiments/RP_05_Buildings_147001")
roof = unreal.load_asset("/Game/LearningLab/Experiments/RP_03_BuildingRoofGeo")
body_mapping = manager.find_mapping(pc, body)
roof_mapping = manager.find_mapping(pc, roof)
unreal.log("Body + Roof generation result: " +
           str(manager.run_rules_on_mappings([body_mapping, roof_mapping])))
```

最终得到三个互补 Actor：

```text
BLDG_N147001
BLDG_L1_COL_N147001
BLDG_ROOFGEO_N147001
```

### 4. 分层显示验证

隐藏 Roof Geo 后，建筑主体顶部是开放的，说明主体规则确实没有包含这部分特殊屋顶：

![[assets/CitySample/08-building-without-roofgeo.png]]

只显示 `BLDG_ROOFGEO_N147001` 时，可以看到两个彼此分离的屋顶平面：

![[assets/CitySample/09-roofgeo-isolated.png]]

重新显示主体后，Roof Geo 与建筑正确对齐并补全屋顶结构：

![[assets/CitySample/07-roofgeo-combined-outline.png]]

报告中的“1 个 Roof Geo 点”并不等于“只能看到一块几何体”。该点通过 `unreal_instance` 引用一个静态网格资产，而这个资产内部可以包含多个互不连接的几何岛。因此画面中的两个屋顶平面属于同一个 Roof Geo 实例，不是重复生成。

### 5. 生成 Rooftop Biome 蓝图

将 `RP_04_BuildingRooftops` 的最外层过滤器改为：

```text
Building_ID = 147001
└─ type = building_rooftop_biom
   └─ Spawn Blueprint
```

Report 返回：

```text
Building_ID = 147001          → 1896 points
type = building_rooftop_biom  → 1 point
Spawn Blueprint              → Instance count = 1
```

这一个点通过 `unreal_instance` 指定要实例化的蓝图。执行生成后得到：

```text
ROOFTOP_BIOM_N558124
```

单独选中该 Actor，可以看到它不是一块简单静态网格，而是一组由蓝图组织的水箱、管线、平台和屋顶设备：

![[assets/CitySample/10-rooftop-biome-isolated.png]]

它与主体和 Roof Geo 一起显示时，正确落在 `147001` 的屋面上：

![[assets/CitySample/11-rooftop-biome-on-building.png]]

这里的 Actor 名称使用点索引 `$INDEX`，所以编号 `558124` 不等于 `Building_ID`。二者分别表达“点云记录索引”和“所属建筑 ID”，不要混淆。

## 本阶段结论

- 已证明不同建筑使用不同的屋顶组合方式；
- 147000 没有独立 Roof Geo，也没有 Rooftop Biome；
- 147001 拥有 1 个独立 Roof Geo 点；
- 147001 还拥有 1 个 `building_rooftop_biom` 点，并成功生成一个屋顶蓝图 Actor；
- 主体与 Roof Geo 可以分别生成、分别显示，并在组合后正确对齐；
- Rooftop Biome 与 Roof Geo、建筑主体在同一坐标系统下正确对齐；
- Rule Processor 的一个点可以引用包含多个几何岛的单个静态网格资产。

---

# 04｜自定义屋顶套件替换

本阶段已经把原 `BPP_Rooftop_small_I` 非破坏式替换为自制的三水箱 Packed Level Actor，并验证生成类、点云 Transform 和 `$INDEX` 命名机制。

完整步骤、截图、原理图和排错清单见：

[[04-自定义屋顶套件替换-Override-Objects-Map]]

GitHub 阅读入口：[04｜用自定义 Packed Level Actor 替换屋顶蓝图](04-自定义屋顶套件替换-Override-Objects-Map.md)

## 下一步

制作第二个自定义 Rooftop Kit，研究如何利用不同 `unreal_instance` 或新的 Metadata 分支选择不同屋顶套件，再继续分析 Pivot、碰撞和实例化性能。
