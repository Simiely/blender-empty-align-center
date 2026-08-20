# CHANGELOG.md

## v1.5.5（2026-08-21）· 待优化标记(无代码变更)

**基于 3ds Max 导入场景实战(260820x03.blend)标记 3 项待优化**,文档先行,代码下轮迭代:

- **T1 multi-user 网格防护(最高优先)**:`set_origin_to_world_point` 的 `data.transform` 对 `obj.data.users > 1` 共享网格会牵连所有共享者(实战:6 对象 3 对共享) → 需自动 `obj.data = obj.data.copy()` 或提示
- **T2 动画对象检测**:带动画对象改原点会破坏动画语义(曲线记录旧原点位置) → 需检测 `animation_data` 跳过或平移动画曲线
- **T3 负缩放/非均匀缩放验证**:3ds Max 导入对象全带非单位缩放+旋转、部分负缩放,插件手动矩阵法未实测负缩放 → 需补测试,必要时先 `transform_apply` 烘焙
- 详见 `DEVELOPMENT.md` 待优化段;`AGENTS.md` 基线更新为 2026-08-21

## v1.5.4（2026-08-16）

**清理残留债务，零行为变更**（用户要求"不要产生新问题"）。

### 重构
- 新增 `preserve_children_world` 上下文管理器，替换 `align_to_children` / `center_object_itself` / `align_empty_to_children_bottom` 三处手写"记录-恢复子物体矩阵"模式（3 处 → 1 处）
- 新增 `apply_axis_steps(done_objects, mode, land)` 合并按钮2/3 的 execute 循环，两个 execute 各缩到 ~9 行
- 3 个旧单对象操作符（AlignEmptyToChildren / OriginToCenter / AlignBottomToGround）**不删除**，只加 `bl_description` 标注"单对象快捷操作"（避免破坏 F3 搜索习惯）

### 验证
- 完整回归 20 项断言全部 PASS（6 操作符 + 三层嵌套 + 不对称场景空物体贴底/落地 + 曲线），零回归
- 行数 552 → 556（标注文字所致，重复逻辑净减）

---

## v1.5.3（2026-08-16）

**父级空物体也参与贴底/落地**。

### 新增
- `align_empty_to_children_bottom(empty, mode)`：空物体的"贴底" = 移到子物体包围盒的底部中心（几何中心 xy + 最低点 z），保持子物体世界位置不变
- 按钮2：空物体 → 子物体底部中心；按钮3：空物体 → 子物体落地后的底部中心（z=0）

### 验证
- 不对称场景（cube 在 z=6）：按钮2 E→(4,2,-1)、锥体轴(3,0,-1)、立方体轴(5,4,5)、顶点不动；按钮3 E→(4,2,0)、两子物体落地；按钮1 回归不破。11 项全 PASS

---

## v1.5.2（2026-08-16）

**三按钮递进语义重构**（用户要求"三个按钮是递进关系，后两个直接引用第一个+增加步骤，不要各自独立算"）。

### 重构
- `center_tree_objects(objects, mode, ctx)` = 按钮1 逻辑，三按钮共用
- `align_axis_to_bottom(obj)` = 按钮2 追加步骤（轴 z → 几何最低点，几何不动）
- 按钮3 = 按钮2 + `land_to_ground`（整体下移贴 Z=0）

### 行为
- 按钮2 = 居中 + 轴贴底；按钮3 = 居中 + 轴贴底 + 落地。空物体在每个按钮里都先"居中"，不参与贴底/落地（v1.5.3 前语义）
- 实测 12 项全 PASS：按钮2 空物体→(4,2,1)、锥体轴→(3,0,-1)、立方体轴→(5,4,1)、顶点不动；按钮3 再落地轴 z=0/minz=0

---

## v1.5.1（2026-08-16）

**轴定位按钮改为"含子集+不限类型"**。

### 新增
- `collect_self_and_descendants(objects)`：从 center_all_with_children 抽出（去重+深先排序），三个按钮共用
- `_transform_geom_data(obj, matrix)`：改用官方 `obj.data.transform(matrix)`（try/except 返回 bool）——正确处理 Bezier ALIGNED/AUTO 控制柄约束（手动赋值 handle 会被重算导致形状变化）；META/FONT 返回 False 回退 origin_set

### 验证
- 实测 12 项全 PASS：曲线 set_origin 控制点+handle 世界位置零变化、曲线落地贴 Z=0、含子集（空物体根→锥体+立方体）轴居中贴底/落地、原回归不破

---

## v1.5.0（2026-08-16）

**架构重构**（用户要求先审查再按计划优化）。

### 重构
- 新增 `world_aabb(obj)` / `world_aabb_multi(objs)` 返回 AABB namedtuple（mn, mx, center, minz），统一几何计算（MESH 一律真实顶点）
- `set_origin_to_world_point` 成唯一手动轴移动实现，`OBJECT_OT_OriginToCenter` 改手动法（不再切选择）
- 新增 `preserve_selection(context)` 上下文管理器
- `center_object_itself` 变纯分发器（EMPTY→align_to_children / MESH→set_origin / 其它→origin_set 兜底）
- 新增 `origin_bottom_center(obj)` / `land_to_ground(obj)` 供轴定位操作符共用
- docstring / bl_info / 面板名改「对象轴与居中工具」，idname 保留兼容

### 验证
- 文件 469→483 行；min/max 手算仅剩 world_aabb 内 2 处；手动矩阵仅 1 处；**回归 17 项断言全部 PASS**（6 operator + 三层嵌套 + 空物体跳过）

---

## v1.4.4（2026-08-16）

**新增「轴居中贴底并落地」**。

### 新增
- `object.center_bottom_and_land`：先轴居中贴底（轴→底部中心，网格不动），再物体整体下移贴 Z=0（网格移动）。最终轴在 (cx, cy, 0)

### 验证
- 实测：顶点统一下移 (0,0,-4)、轴到 (2,3,0)、最低点 z=0。全过

### 最终面板
中心模式 + 居中（自身+子集） + 轴居中贴底（一键） + 轴居中贴底并落地

---

## v1.4.3（2026-08-16）

**面板精简**：用户发现"轴贴底部"与"轴居中贴底"效果差不多（轴 xy 常在几何中心）→ 面板移除「轴贴底部」按钮（operator 保留，F3 可调）。

---

## v1.4.2（2026-08-16）

**核心语义纠正：全部按钮只动轴，网格永不移动**（用户澄清："插件不会移动物体，物体永远在原地，只是改变轴体的位置"）。

### 变更
- v1.4.1 把"居中并贴地"改成整体移动网格是**错误方向，已回改**
- 最终语义（全部按钮只动轴，MESH 网格世界位置不变）：
  - `center_then_align_bottom`（轴居中贴底）：轴 → 世界包围盒底部中心
  - `align_bottom_to_ground`（轴贴底部）：轴 z → 物体最低点，xy 保持
  - `center_all_with_children`（居中自身+子集）：不变

### 新增
- `set_origin_to_world_point(obj, target)`：通用"轴移到任意世界点"
- `_world_geom_points(obj)`：MESH 用 data.vertices，其它 GEOMETRY_TYPES 用 bound_box

### 验证
- 实测：两个按钮顶点世界位置零变化，轴分别到 (2,3,4)；空物体不受影响。全部通过

---

## v1.4.1（2026-08-16）

**修复"居中贴地只移轴不移动体"**（用户反馈："所有修改都是不移动物体只是轴体"）。

### 变更
- 重写 `center_then_align_bottom`：**真正整体移动**（网格跟着动）——顶层对象+全部后代几何点合并包围盒 → delta = (-cx, -cy, -minz) → 每个顶层对象 matrix_world.translation += delta
- 顺带修复 `align_bottom_to_ground` 的 bound_box 缓存 bug：新增 `_world_min_z(obj)`（MESH 用真实顶点）

> ⚠️ 注：本版本方向被 v1.4.2 纠正——用户最终语义是"只动轴"，已回改。

---

## v1.4.0（2026-08-16）

**一键居中合并**（用户要求："对齐子物体"和"自身居中"不拆开，选到什么就自身+全部子集都居中）。

### 新增
- `OBJECT_OT_CenterAllWithChildren`（`object.center_all_with_children`，面板主按钮"居中(自身+子集)"）：收集选中对象及全部后代，按深度从深到浅去重处理；EMPTY→align_to_children，MESH→手动 Set Origin，其它几何→原生 origin_set
- **手动 Set Origin 实现**（关键）：`mw_new.translation = center; obj.matrix_world = mw_new; m = mw_new.inverted() @ mw_old; for v in obj.data.vertices: v.co = m @ v.co`——对带父级/旋转缩放也精确保持顶点世界位置

### 踩坑
- 原生 `origin_set` 对带父级对象行为不符（顶点被平移而 location 未变）→ MESH 用手动矩阵法
- `mesh.bound_box` 是缓存属性，改 v.co 后不刷新 → `get_world_bbox_center` 必须用真实顶点
- **Blender 5.2 默认锥体原点在几何中心（z -1..1），不是旧版底部（z 0..2）**——多次"验证失败"实为测试假设错，插件本身正确

### 验证
- 三层嵌套（E→ConeA/ConeB→CubeC）：顶点世界位置不变、各 mesh 原点居中、E 移到后代包围盒中心 (4.5,2.5,1.0)

---

## v1.3.0（2026-08-16）

**模型整理功能组**（用户澄清需求："居中"= 物体自身几何中心，"倒三角符号的对象"= 模型本身非空物体）。

### 新增
- `object.origin_to_geometry_center` 原点居中：`origin_set(type='ORIGIN_GEOMETRY', center='BOUNDS')`（跳过 EMPTY，暂存/恢复选择）
- `object.align_bottom_to_ground` 底部贴地：按对象自身 bound_box 世界 minZ 整体平移（子物体自动跟随，跳过 EMPTY）
- `object.center_then_align_bottom` 一键：嵌套调用前两个 operator

### 踩坑
- 锥体顶点平均≠包围盒中心（底部8顶点+顶部1顶点）；验证 Set Origin 要用 bound_box 中心，且 verts_before 必须在 Set Origin 前同状态记录

---

## v1.2.0（2026-08-16）

**对象通用版**（需求从"仅空物体"扩展为"任意对象+多选"）。

### 变更
- 核心函数 `align_empty_to_children(empty)` → `align_to_children(obj)`（children_recursive/matrix_world 对任意类型有效）
- operator `poll` 从 `active_object.type=='EMPTY'` 改为 `bool(selected_objects)`
- `execute` 遍历所有选中对象各自对齐，统计 done/skipped 报告
- bl_idname 保留 `object.align_empty_to_children`（未绑快捷键，改 idname 无收益）

### 验证
- 实测（Blender 5.2 后台脚本）：网格父级+空物体父级混合多选，各自移到子物体包围盒中心，全部子物体世界位置不变

---

## v1.1.0（2026-08-16）

**汉化插件列表名**：bl_info `name` 英文 → 「空物体对齐子物体中心」（插件列表显示名不随界面语言翻译，只能改 bl_info）；同时汉化 `location` 为 "3D视图 > 侧边栏(N) > Tool"。

---

## v1.0.0（2026-08-16）

**初始发布**：空物体对齐子物体中心插件。

- 需求：用户想把 Empty 的坐标变成其所有子物体的中心
- 核心技巧：直接移动父级 empty 会连带移动子物体 → ①记录所有子物体 matrix_world 副本 ②`empty.matrix_world.translation = center` 只改平移 ③再逐个恢复子物体 matrix_world
- 三种中心模式（面板可切换）：BBOX 世界包围盒中心（默认）/ ORIGIN 原点平均 / VERTS 网格顶点平均；递归用 children_recursive；跳过 EMPTY 类型
- 已安装到本机 Blender 5.2/5.1 并后台启用验证
