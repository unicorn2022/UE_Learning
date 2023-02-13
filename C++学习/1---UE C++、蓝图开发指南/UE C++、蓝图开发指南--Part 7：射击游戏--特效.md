# 目录

[TOC]

# 一、两个特效系统概述：Cascade、Niagara

1. 创建文件夹`VFX`，用于存储特效
2. **Cascade系统**：
   1. 创建粒子系统`PS_Test`，双击打开粒子系统，该子系统被称为`Cascade(级联系统)`
   2. 在`发射器`窗口中，`右键/位置/圆柱体`，可以使粒子沿圆柱体生成
3. **Niagara发射器**：
   1. 基于模板`Upward Mesh Burst`创建FX/Niagara发射器`NE_BaseImpact`
   2. 在`Spawn Burst Instantaneous`中，可以修改粒子数
   3. 在`Initialize Particle `中，可以修改粒子的颜色
   4. `Niagara发射器`不能直接添加进场景中，因为它不是actor
4. **Niagara系统**：
   1. 创建空白Niagara系统`NS_BaseImpact`
   2. 在蓝图界面添加刚刚创建的发射器
   3. `Niagara系统`可以添加进场景中

