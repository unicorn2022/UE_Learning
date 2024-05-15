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

## 2.1	Shader相关类

### 2.1.1	RHI层次的基类：`FRHIShader`

```c++
/* 源代码: Engine/Source/Runtime/RHI/Public/RHIREsources.h */
class FRHIShader : public FRHIResource{
private:
	FSHAHash Hash;				// 哈希值
	EShaderFrequency Frequency;	// 定义是哪种类型的Shader，如Vertex、Compute、Pixel、RayGen…
};
```

### 2.1.2	RHI层次的不同Shader对应的类

```c++
/** 这里的继承只是用于初始化 RHIShader 里面的两个成员变量 **/

/* 实时渲染 */
// 先使用 GraphicsShader 继承一次
class FRHIGraphicsShader : public FRHIShader;
// 然后常用的 Shader 类型继承自 GraphicsShader
class FRHIVertexShader : public FRHIGraphicsShader;
class FRHIMeshShader : public FRHIGraphicsShader;
class FRHIPixelShader : public FRHIGraphicsShader;
class FRHIGeometryShader : public FRHIGraphicsShader;

/* 光追 */
// 先使用 RayTracingShader 继承一次
class FRHIRayTracingShader : public FRHIShader;
// 然后继承 RayTracingShader
class FRHIRayGenShader : public FRHIRayTracingShader;
class FRHIRayMissShader : public FRHIRayTracingShader;
class FRHIRayCallableShader : public FRHIRayTracingShader;
class FRHIRayHitGroupShader : public FRHIRayTracingShader;
```

### 2.1.3	不同硬件对应的实现

```c++
/* 源代码: Engine/Source/Runtime/D3D12RHI/Private/D3D12Shader.h */
struct FD3D12ShaderData {
    // 着色器代码
	TArray<uint8> Code; 
    ...
    // 当前着色器用到的资源数量
    // 即sampler, SRV, CB, UAV各自的数量
    // 可以通过FShaderCodeReader读取Code, 从而得到这些信息
    FShaderCodePackedResourceCounts ResourceCounts{}; 
    
}
class FD3D12VertexShader : public FRHIVertexShader, public FD3D12ShaderData;
```

- `SRV`：shader resource view
- `CB`：constant buffer
- `UAV`：unordered access view

## 2.2	创建Shader

### 2.2.1	RHI层面调用

```c++
/* 源代码: Engine/Source/Runtime/RHI/Public/RHICommandList.h */
FVertexShaderRHIRef RHICreateVertexShader(TArrayView<const uint8> Code, const FSHAHash& Hash);
```

### 2.2.2	不同硬件对应的实现

```c++
/* 源代码: Engine\Source\Runtime\OpenGLDrv\Private\OpenGLShaders.cpp */
FVertexShaderRHIRef FOpenGLDynamicRHI::RHICreateVertexShader(TArrayView<const uint8> Code, const FSHAHash& Hash);
```

- 在UE中，`TArrayView`并不是只存数据本身，还有一定的参数
- 因此需要用一个Reader解析数据，如`FShaderReader`
  - Array中不仅存储了代码`Code`，还存储了数据`OptionalData`，先代码后数据
  - Array的最后一个对象表示存储的数据大小
    - 可以通过`FShaderReader.GetOptionalDataSize()`获取到
  - 通过`Key`获取数据本身：
    - 可以通过`FShaderReader.FindOptionalData(InKey)`获取到
- 获取到代码后，会先通过根据平台转化GLSL代码，然后调用OpenGL的函数编译代码

```c++
/* 源代码: Engine/Source/Runtime/D3D12RHI/Private/D3D12Shader.cpp */
FVertexShaderRHIRef FD3D12DynamicRHI::RHICreateVertexShader(TArrayView<const uint8> Code, const FSHAHash& Hash) {
	return CreateStandardShader<FD3D12VertexShader>(Code);
}

template <typename TShaderType>
TShaderType* CreateStandardShader(TArrayView<const uint8> InCode) {
    // 解析 Code
	FShaderCodeReader ShaderCode(InCode);
	TShaderType* Shader = new TShaderType();

	FMemoryReaderView Ar(InCode, true);
	Ar << Shader->ShaderResourceTable;

	const int32 Offset = Ar.Tell();

   	// 初始化 Shader
	if (!InitShaderCommon(ShaderCode, Offset, Shader)) {
		delete Shader;
		return nullptr;
	}

	UE::RHICore::InitStaticUniformBufferSlots(Shader->StaticSlots, Shader->ShaderResourceTable);
	return Shader;
}
```

## 2.3	上层使用Shader：`FShader`

### 2.3.1	`FShader`中的宏代码讲解

```c++
/* 源代码: Engine\Source\Runtime\RenderCore\Public\Shader.h */
class RENDERCORE_API FShader {
    friend class FShaderType;
    // 由于未继承UObject, 因此需要实现新的反射机制
	DECLARE_TYPE_LAYOUT(FShader, NonVirtual);
public:
    ...
    // 编译环境发生改变时的操作
    static void ModifyCompilationEnvironment(const FShaderPermutationParameters&, FShaderCompilerEnvironment&) {}

	// 是否需要编译Permutation
	static bool ShouldCompilePermutation(const FShaderPermutationParameters&) { return true; }
    ...
protected:
    ...
    // 定义反射变量
    LAYOUT_FIELD_EDITORONLY(FSHAHash, OutputHash);
private:
    ...
   	// 定义反射变量
    LAYOUT_FIELD(FShaderTarget, Target);
};
```

#### 2.3.1.1	`Permutation`

- 在UE的shader中`*.usf`，同一份代码可能会有不同的编译选项，从而产生出编译选项的排列组合
- 与`Permutation`相关的类就是在处理这个排列组合

#### 2.3.1.2	`FShader`的反射机制实现思路：

- 在`DECLARE_TYPE_LAYOUT()`处定义`Initialize()`的默认函数

  ```c++
  template<int Counter>
  struct InternalLinkType{
      static void Initialize(FTypeLayoutDesc& TypeDesc){}
  }
  ```

- 在`LAYOUT_FIELD()`处定义`Initialize()`的偏特化，构成链式初始化结构，然后收集当前变量的信息

  - 宏展开后，`InternalLinkType<>`处的模板参数会自动`+1`
  - `+1`是通过`__COUNTER__`这个编译器预定义宏实现的

  ```c++
  FShaderParameterBindings Bindings;
  template<>
  struct InternalLinkType<601 - CounterBase> {
      static void Initialize(FTypeLayoutDesc& TypeDesc) {
          // 调用下一个InternalLinkType的初始化函数
          InternalLinkType<601 - CounterBase + 1>::Initialize(TypeDesc);
         	
          // 收集当前成员变量的信息
          alignas(FFieldLayoutDesc)
          static uint8 FieldBuffer[sizeof(FFieldLayoutDesc)] = {0};
          FFieldLayoutDesc& FieldDesc = *(FFieldLayoutDesc*)FieldBuffer;
          
          FieldDesc.Name = L"Bindings";
          ...
          FieldDesc.Offset = ((::size_t) & reinterpret_cast<char const volatile&>(((DerivedType*)0)->Bindings));
          ...
             
      }
  }
  ```

- 在`IMPLEMENT_GLOBAL_SHADER()`中调用`InternalLinkType<1>::Initialize(TypeDesc)`，从而链式初始化所有成员变量对应的`InternalLinkType<>`

### 2.3.2	使用示例：自己编写Shader，在UE中使用

通常是自己写一个类，继承`FGlobalShader`

- `FGlobalShader`继承自`FShader`


```c++
/* 源代码: Engine\Source\Runtime\Renderer\Private\LightRendering.h */
class FDeferredLightVS : public FGlobalShader {
    // 两个固定的宏
	DECLARE_SHADER_TYPE(FDeferredLightVS,Global);
	SHADER_USE_PARAMETER_STRUCT(FDeferredLightVS, FGlobalShader);

	class FRadialLight : SHADER_PERMUTATION_BOOL("SHADER_RADIAL_LIGHT");
	using FPermutationDomain = TShaderPermutationDomain<FRadialLight>;
	
    // 定义Shader的参数格式
	BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
		SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
		SHADER_PARAMETER_STRUCT_INCLUDE(FDrawFullScreenRectangleParameters, FullScreenRect)
		SHADER_PARAMETER_STRUCT_INCLUDE(FStencilingGeometryShaderParameters::FParameters, Geometry)
	END_SHADER_PARAMETER_STRUCT()
	
public:
    ...
};
```

然后在`.cpp`中通过`IMPLEMENT_GLOBAL_SHADER()`注册对应的着色器文件、入口函数、着色器类型

- 在这里调用了`InternalLinkType<1>::Initialize(TypeDesc)`

```c++
/* 源代码: Engine\Source\Runtime\Renderer\Private\LightRendering.cpp */
IMPLEMENT_GLOBAL_SHADER(FDeferredLightVS, "/Engine/Private/DeferredLightVertexShaders.usf", "VertexMain", SF_Vertex);
```

# 三、Shader的编译、创建流程

`GGlobalShaderMap`：

- 管理所有Shader的信息

  ```c++
  /* 源代码: Engine\Source\Runtime\RenderCore\Private\GlobalShader.cpp */
  FGlobalShaderMap* GGlobalShaderMap[SP_NumPlatforms] = {};
  ```

在引擎初始化时：

- 调用`CompileGlobalShaderMap`创建全局Shader信息

  ```c++
  /* 源代码: Engine\Source\Runtime\Engine\Public\ShaderCompiler.h */
  extern ENGINE_API void CompileGlobalShaderMap(bool bRefreshShaderMap=false);
  ```

- 在创建Shader时，会通过FShaderCodeReader解析编译后的Shader代码，获取需要的统计信息

  - 在D3D12编译Shader代码时，编译出的结果不仅包含可执行程序，还包含了对Shader的描述信息
  - UE通过D3D12提供的接口，记录Shader的信息

- 在创建PipelineState时，UE会通过`FD3D12BoundShaderState`获取所有Shader的`ShaderData`，从而创建`FD3D12QuantizedBoundShaderState`

  - `FD3D12QuantizedBoundShaderState`：是一个渲染管线的所有Shader绑定起来的信息
  - 然后根据这个QBSS创建根签名

# 四、渲染命令的封装