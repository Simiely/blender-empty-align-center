# Blender 对象轴与居中工具（Empty Align Center）

Blender 插件：把所有「轴体操作」统一在一个面板里 —— **物体（网格）永远不动，只改原点（轴）位置**。三个按钮递进，全部含子集、不限对象类型。

![Blender](https://img.shields.io/badge/Blender-2.93%2B-blue) ![Python](https://img.shields.io/badge/Python-3.7%2B-green) ![License](https://img.shields.io/badge/License-GPL--3.0-orange)

## 功能

- **居中（自身+子集）**：选中对象及其全部后代逐个处理
  - 网格/曲线等：轴（原点）居中到自身几何中心，几何不动
  - 空物体：移动到它所有子物体的中心（子物体世界位置不变）
- **轴居中贴底（一键）**：每个对象轴移动到自身底部中心（几何中心 xy + 最低点 z），几何不动
- **轴居中贴底并落地**：先轴移到底部中心，再整体下移贴到 Z=0（几何移动）
- **中心模式**（仅影响空物体定位）：包围盒中心 / 原点平均 / 顶点平均
- 另保留 3 个单对象快捷操作符（`object.align_empty_to_children` / `object.origin_to_geometry_center` / `object.align_bottom_to_ground`），可在 F3 搜索调用

> **核心语义**：除「并落地」会整体下移网格外，所有按钮都是轴体操作（原点定位），网格世界位置不变。

## 安装

**方式一（偏好设置）**：编辑（Edit）→ 偏好设置（Preferences）→ 插件（Add-ons）→ 安装（Install）→ 选择 `empty_align_center.py` → 勾选启用。

**方式二（脚本目录）**：把文件放进 `scripts/addons/`（或自定义脚本目录的 `addons/` 子目录），重启 Blender 后在偏好设置里启用。

> 兼容 Blender 2.93 ~ 5.x，仅用基础 Python API + Blender 内置 `mathutils`，无第三方依赖。

## 快速开始

1. 打开插件面板：3D 视口 → 侧边栏（`N`）→ **Tool** 标签 →「对象轴与居中工具」
2. 选中任意对象（可多选，空物体/模型/带子集的父级混合均可）
3. 点一个按钮：
   - **居中（自身+子集）**：各自原点归位，网格不动
   - **轴居中贴底（一键）**：各自原点移到自身底部中心，网格不动
   - **轴居中贴底并落地**：原点到底部中心 + 物体整体下移贴 Z=0

## 实现要点

- 直接移动父级会连累子物体（父子联动）：先记录子物体世界矩阵，移动后再逐个恢复
- 手动 Set Origin：`mw_new.translation = target`，顶点局部坐标用 `mw_new.inv() @ mw_old` 变换，网格世界位置精确不变（对带父级/旋转缩放也可靠）
- MESH 的 `bound_box` 是缓存属性，改 `v.co` 后不刷新 → 包围盒一律用真实顶点计算
- Bezier 控制柄用官方 `data.transform()` 处理，手动赋值 handle 会被对齐约束重算导致形状变化

## 兼容性说明

- Blender 5.x 图标枚举移除了旧图标：`TRANSFORM` 不存在（只有 `TRANSFORM_ORIGINS`），本插件使用 `SNAP_ON` / `OBJECT_DATA` / `SNAP_FACE` 等 5.x 仍存在的图标
- 旧式 `bl_info` 插件在 5.x 仍受支持（偏好设置里有「安装旧式插件」入口）
