# AGENTS.md · 项目规则

> 📌 **文档基线**:2026-08-21(commit 待回填)标记 3 项待优化(multi-user 网格 / 动画对象 / 负缩放验证)
> 更新文档/代码后,请更新此行(日期 + 新 commit hash),并在 CHANGELOG 追加版本

## 技术栈

- Blender **2.93 ~ 5.x**（单文件 legacy addon，`bl_info` 完整），本机验证环境 Blender 5.2 LTS
- 纯 Python + Blender 内置 `bpy` / `mathutils`；**无任何第三方依赖**
- 单文件插件（`.py` 直接安装），交付路径：工作区 `empty_align_center.py` 是唯一权威版本，安装到 Blender 时复制到 `scripts/addons/`（或自定义脚本目录）

## 关键坑（务必先读）

- **直接移动父级会连累子物体**（父子联动）：移动父级前必须记录所有子物体 `matrix_world` 副本，移动后再逐个恢复（`preserve_children_world`）
- **原生 `origin_set` 对带父级对象行为与预期不符**（实测顶点被平移而 location 未变）：网格/曲线等一律用手动矩阵法 `set_origin_to_world_point`（`mw_new.translation = target` + 顶点用 `mw_new.inv() @ mw_old` 变换），META/FONT 才回退原生 `origin_set`
- **`mesh.bound_box` 是缓存属性**，手动改 `v.co` 后不立即刷新 → 包围盒/几何点一律用真实顶点 `data.vertices` 计算（`_world_geom_points`），不能依赖 bound_box
- **Bezier 控制柄约束**：手动赋值 handle 坐标会被 Blender 按 ALIGNED/AUTO 约束重算导致形状变化 → 用官方 `obj.data.transform(matrix)` 处理几何数据
- **Blender 5.x 图标枚举移除了旧图标**：`TRANSFORM` 已不存在（只有 `TRANSFORM_ORIGINS`）。icon 只用：`SNAP_ON`/`OBJECT_DATA`/`SNAP_FACE`/`PIVOT_BOUNDBOX`
- **产品语义（用户明确）**：除「轴居中贴底并落地」外，所有按钮都是**轴体操作（原点定位），网格永远不动**。不要改成"整体移动物体"——v1.4.1 曾做错过方向，v1.4.2 已回改
- **⚠️ 待优化(2026-08-21)**:① `obj.data.users > 1`(multi-user 共享网格)时 `data.transform` 会牵连所有共享者 → 先 `obj.data = obj.data.copy()`;② 带动画对象改原点会破坏动画(曲线记录旧原点位置)→ 检测 `obj.animation_data` 跳过或平移动画曲线;③ 负缩放/非均匀缩放对象未实测,3ds Max 导入场景必现(详见 DEVELOPMENT.md 待优化段)

## 约定

- 注释用中文；UI 标签/按钮/bl_info 用中文
- bl_idname 前缀 `object.`（历史遗留，未绑快捷键，勿改 idname 以免破坏 F3 搜索习惯）
- 三个面板按钮是**严格递进**关系：按钮2 = 按钮1 + 轴贴底；按钮3 = 按钮2 + 落地。新增步骤请直接引用前一步逻辑，不要各自独立计算（v1.5.2 用户明确要求）
- 空物体在每个按钮里都先执行「居中」（移到子物体中心），且参与贴底/落地（移到子物体包围盒底部中心）
- 改完必须实测验证：后台 `blender -b --python-expr` 建场景 → 调用 operator → 断言 `matrix_world.translation` 与顶点世界位置

## 常用命令

- 语法检查：`python -m py_compile empty_align_center.py`
- 后台启用验证：`blender.exe -b --python-expr "import addon_utils; addon_utils.enable('empty_align_center', default_set=True); print('OK')"`
