[TOC]

# 目录

教程地址：https://www.bilibili.com/video/BV1VK411x7bH

# 一、RHI资源

## 1.1	GPU渲染管线

### 1.1.1	一次完成的渲染流程(draw call)图示

![image-20240508161304650](AssetMarkdown/image-20240508161304650.png)

**顶点输入**

- 动画、变换：vertex shader
  - 局部坐标 => 全局坐标 => 视口坐标
- 曲面细分：hull shader、domain shader
- 几何着色器
- 透视变换
  - 3D => 2D、z-buffer
- 裁剪
- 光栅化

**像素**

- 着色：fragment shader
- 深度检测、抗锯齿
- 后处理

**Output Buffer**

### 1.1.2	程序的三个关键点

- 代码本身
- 参数：输入数据的种类、格式
- 真正的数据

在将**数据**传递给GPU之前，需要将**数据的格式**也传给GPU

## 1.2	UE5.1中的RHI资源

**封装思路：定义基类，在不同平台实现子类，上层调用时只访问基类**

### 1.2.1	RHI资源的基类：`FRHIResource`

```c++
/* 源代码: Engine/Source/Runtime/RHI/Public/RHIResources.h */
class RHI_API FRHIResource;
```

- `ERHIResourceType`：定义资源类型
  - 如：各种State、Shader、Buffer、Texture、Fence、Qurey

### 1.2.2	RHI资源：`FRHIUniformBuffer`

```c++
/* 源代码: Engine/Source/Runtime/RHI/Public/RHIResources.h */
// 这里只是接口, 代表的是一个通用的跨平台UniformBuffer, 并没有具体实现
class FRHIUniformBuffer : public FRHIResource;
```

- `Layout`：表示UniformBuffer的结构
- `LayoutConstantBuffersize`：表示UniformBuffer的大小

```c++
/* 源代码: Engine/Source/Runtime/D3D12RHI/Public/D3D12Resources.h */
// 不同的平台会继承UE定义的接口实现对应的代码
class FD3D12UniformBuffer : public FRHIUniformBuffer, public FD3D12DeviceChild, ...;
```

- `ResourceLocation`：存储该UniformBuffer在GPU中对应的D3D12Resource指针
  - 在成员变量`FD3D12Resource`中，存储了真正的D3D12Resource指针`TRefCountPtr<ID3D12Resource> Resource`

```c++
/* 源代码: Engine/Source/Runtime/OpenGLDrv/Public/OpenGLResources.h */
// 不同的平台会继承UE定义的接口实现对应的代码
class FOpenGLUniformBuffer : public FRHIUniformBuffer;
```

- `Resource`：存储该UniformBuffer在GPU中对应的OpenGL资源指针

### 1.2.3	RHI资源：`FRHITexture`

```c++
class FRHIViewableResource : public FRHIResource;
class FRHITexture : public FRHIViewableResource;
// 不同的平台会继承UE定义的接口实现对应的代码
class FD3D12Texture : public FRHITexture, public FD3D12BaseShaderResource, ...;
class FOpenGLTexture : public FRHITexture;
```

### 1.2.4	上层操作RHI资源：`FDynamicRHI`

```c++
/* 源代码: Engine/Source/Runtime/RHI/Public/DynamicRHI.h */
// 定义了使用RHI资源的操作接口方法, 如:更新贴图等
class RHI_API FDynamicRHI;
// 不同的平台会继承该接口实现对应的代码
class FOpenGLDynamicRHI : public FDynamicRHI;
```

# 二、Shader