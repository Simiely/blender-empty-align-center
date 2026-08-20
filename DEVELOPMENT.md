# DEVELOPMENT.md · 项目概览 → 架构说明 → 关键问题与方案

## 项目概览

`empty_align_center.py`（556 行，单文件）——Blender「对象轴与居中工具」。核心语义：**物体（网格）永远不动，只改原点（轴）位置**。三个递进按钮 + 中心模式：

1. **居中（自身+子集）**：选中对象及全部后代逐个处理（网格轴居中到几何中心；空物体移到子物体中心）
2. **轴居中贴底（一键）**：= 按钮1 + 每个对象轴移到底部中心（几何中心 xy + 最低点 z）
3. **轴居中贴底并落地**：= 按钮2 + 每个对象整体下移贴到 Z=0（唯一移动网格的按钮）

中心模式（仅影响空物体定位）：包围盒中心 / 原点平均 / 顶点平均。

## 架构说明

四层结构，依赖方向单向（UI/调度 → 层级工具 → 变换工具 → 几何工具），无循环依赖：

```
UI / 面板层          OBJECT_PT_EmptyAlignPanel + 6 个 Operator
    ↓ 调用
层级工具层           center_tree_objects / collect_self_and_descendants / apply_axis_steps
                    / center_object_itself / align_to_children / align_empty_to_children_bottom
    ↓ 调用
变换工具层           set_origin_to_world_point / origin_bottom_center / land_to_ground
                    / _transform_geom_data / preserve_children_world / preserve_selection
    ↓ 调用
几何工具层           _world_geom_points / world_aabb / world_aabb_multi
                    / get_world_origin_avg / get_world_verts_avg
```

**数据流约定**（改代码前先理解这条链）：
```
选中对象 → collect_self_and_descendants（收集+去重+深先排序）
  → 按钮1: center_tree_objects → 逐个 center_object_itself（EMPTY→align_to_children / 几何→set_origin / META/FONT→origin_set 兜底）
  → 按钮2: + apply_axis_steps(land=False) → 空物体→align_empty_to_children_bottom / 几何→align_axis_to_bottom
  → 按钮3: + apply_axis_steps(land=True) → 再 land_to_ground
```

**三按钮递进设计**（v1.5.2 用户要求，严禁各自独立计算）：
- `center_tree_objects` = 按钮1 逻辑，三按钮共用
- `apply_axis_steps(done_objects, mode, land)` = 按钮2/3 共用（land flag 区分）

## 关键问题与方案

### 1. 父子联动（最核心难点）
直接移动父级 empty/对象会连带移动所有子物体。方案：`preserve_children_world` 上下文管理器——进入前记录子物体世界矩阵，操作后逐个恢复 `matrix_world`。三处旧手写模式已在 v1.5.4 收敛到这一个工具。

### 2. 手动 Set Origin（替代原生 origin_set）
原生 `origin_set` 对带父级对象的包围盒/位置行为与预期不符。方案：`set_origin_to_world_point(obj, target)`——
```python
mw_old = obj.matrix_world.copy()
mw_new = mw_old.copy(); mw_new.translation = target
obj.matrix_world = mw_new
m = mw_new.inverted() @ mw_old
obj.data.transform(m)   # 顶点局部坐标补偿，网格世界位置精确不变
```

### 3. bound_box 缓存陷阱
`mesh.bound_box` 是缓存属性，手动改 `v.co`（Set Origin 补偿）后不刷新，继续用它算包围盒会得到过期结果。方案：`_world_geom_points` 对 MESH 用真实 `data.vertices`，CURVE/SURFACE/LATTICE 用真实数据点，仅 META/FONT 回退 bound_box。

### 4. Bezier 控制柄约束
手动给 `bezier_points[i].handle_*` 赋值会被 Blender 按 ALIGNED/AUTO 约束重算，导致曲线形状变化。方案：统一走官方 `obj.data.transform(matrix)`（try/except 返回 bool），不支持的几何类型（META/FONT）返回 False 回退 origin_set。

### 5. 空物体的"贴底"
空物体无几何，其"贴底"= 移到**子物体包围盒的底部中心**（`align_empty_to_children_bottom`），同样用 preserve_children_world 保持子物体世界位置。v1.5.3 补全（此前空物体只参与居中不参与贴底/落地）。

### 6. 选择状态保护
`preserve_selection` 上下文管理器：临时切换选择执行 origin_set 兜底后，自动恢复原选择与 active 对象。

## 待优化(TODO) · 2026-08-21 实战发现(3ds Max 导入场景 260820x03.blend)

> 来源:同日 blender-tips 实战。以下缺口在 3ds Max 导入场景(带缩放/旋转/负缩放/共享网格/动画残留)下真实触发过,下次迭代优先处理。

### T1. multi-user 网格防护(最高优先)
`set_origin_to_world_point` 调 `obj.data.transform(m)` 直接改顶点——当 `obj.data.users > 1`(多个对象共享同一 mesh 数据,3ds Max 导入常见)时,**一个对象改原点,所有共享者一起被改**。
- 实测:260820x03 主装置集合 6 个对象 3 对共享网格(`对象245x_GeomAdjust` 共用 Mesh.462x)
- 处理:入口处检测 `obj.data.users > 1` → 自动 `obj.data = obj.data.copy()`(独立副本,视觉不变)或弹提示让用户选择
- 原生 origin_set 的 `transform_apply` 同样对 multi-user 报 `Cannot apply to a multi user`

### T2. 动画对象检测
带动画对象改原点会破坏动画语义:动画曲线记录的是"原点位置",原点移动后动画播放时对象整体偏移。
- 现状:插件未检测 `obj.animation_data`,直接改
- 处理:检测到动画 → 跳过并报告,或平移动画曲线(location fcurve 每关键帧 + delta)
- 注意 5.2 slotted action:曲线遍历用 `action.fcurve_ensure_for_datablock(obj, path, index=i)`(slot 可能是空引用,导入残留)

### T3. 负缩放/非均匀缩放验证
3ds Max 导入对象常带**非单位缩放(0.006~0.594)+ 旋转,部分负缩放(镜像)**。原生 `origin_set` 对这类对象直接翻车(位置跳变),插件的手动矩阵法(`mw_new.inverted() @ mw_old`)理论上精确,但**未实测负缩放**。
- 处理:补测试用例(负缩放矩形/非均匀缩放)验证 bbox 中心位移=0;不过可先用 blender-tips 安全流程:`transform_apply(rotation, scale)` 烘焙 → 再设原点
