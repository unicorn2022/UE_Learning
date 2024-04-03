[TOC]

<div STYLE="page-break-after: always;"></div>

# 一、插值

插值：给两个数之间补充一些数，过渡变得更自然。

## 1.1	包含的函数

1.   F插值(Float)：浮点插值

     <img src="AssetMarkdown/image-20220802161553771.png" alt="image-20220802161553771" style="zoom:80%;" />

2.   R插值(Rotation)：角度插值

     <img src="AssetMarkdown/image-20220802161708182.png" alt="image-20220802161708182" style="zoom:80%;" />

3.   V插值(Vector)：向量插值

     <img src="AssetMarkdown/image-20220802161649423.png" alt="image-20220802161649423" style="zoom:80%;" />

4.   T插值(Transform)：移动插值

     <img src="AssetMarkdown/image-20220802161732563.png" alt="image-20220802161732563" style="zoom:80%;" />

## 1.2	参数说明

以上函数一般在**“Tick”事件**中使用：

1.   **Current**：当前值
2.   **Target**：期望的目标值
3.   **Delta Time**：时间变化值。
4.   **Interp Speed**：插值速度
5.   返回值：从**“当前值”**过渡到**“期望的目标值”**的一个中间值

# 二、IK

1.   逆向运动学**Inverse Kinematics**，简称**IK**
2.   正向运动学：即从骨骼的上级到下级进行旋转来达到自己想要的姿势，这是一个正向的思维。
3.   逆向运动学：是已知最后想要达成的姿势，然后反求出骨骼们的旋转。