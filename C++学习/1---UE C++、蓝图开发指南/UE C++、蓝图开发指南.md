# 目录

[TOC]

# 零、UE与VS的交互

1.   用UE编译VS的C++代码：
     1.   在VS中，点击`调试`按钮，自动唤出`UE4Editor`
     2.   如果已经打开了`UE4Editor`，则可以`调试 => 附加到进程`，选择`UE4Editor`进程
2.   修改VS中的C++代码之后
     1.   在`UE4Editor`中，点击`编译`即可，不需要重新启动

# 一、创建第一个AActor类型的C++类

1.   在UE中：`文件 => 新建C++类 => Actor`，创建一个AActor类型的C++类，重命名为`BasicGeometryActor`，为`公共`类

     <img src="AssetMarkdown/image-20230117175425764.png" alt="image-20230117175425764" style="zoom:67%;" />

2.   创建后，UE会自动进行编译，并将其添加进VS工程

3.   自动生成的代码如下：

     1.   `BasicGeometryActor.h`

          ```c++
          // Fill out your copyright notice in the Description page of Project Settings.
          
          #pragma once
          
          #include "CoreMinimal.h"
          #include "GameFramework/Actor.h"
          // 这个库必须最后引用，用于生成一些宏
          #include "BasicGeometryActor.generated.h" 
          
          UCLASS()
          class UE_CPP_API ABasicGeometryActor : public AActor
          {
          	GENERATED_BODY()
          	
          public:	
          	// Sets default values for this actor's properties
          	ABasicGeometryActor();
          
          protected:
          	// 在游戏开始或生成时调用
          	virtual void BeginPlay() override;
          
          public:	
          	// 每一帧调用一次
          	virtual void Tick(float DeltaTime) override;
          
          };
          ```

     2.   `BasicGeometryActor.cpp`

          ```c++
          // Fill out your copyright notice in the Description page of Project Settings.
          
          
          #include "BasicGeometryActor.h"
          
          // Sets default values
          ABasicGeometryActor::ABasicGeometryActor()
          {
           	// true: 当前actor在每一帧都可以调用Tick()
          	// false: 当前actor在每一帧不调用Tick()
          	PrimaryActorTick.bCanEverTick = true;
          
          }
          
          // 在游戏开始或生成时调用
          void ABasicGeometryActor::BeginPlay()
          {
          	Super::BeginPlay();
          	
          }
          
          // 每一帧调用一次
          void ABasicGeometryActor::Tick(float DeltaTime)
          {
          	Super::Tick(DeltaTime);
          
          }
          ```

# 二、宏 `UE_LOG`：输出日志信息

## 2.1	输出字符串

1.   `UE_LOG`的三个参数：

     1.   `CategoryName`：日志类的名字
     2.   `Verbosity`：日志的提醒等级
     3.   `Format、__VA_ARGS__`：日志的格式和内容，通常直接使用`TEXT()`宏将要输出字符串转化过去即可

     ```c++
     // 在游戏开始或生成时调用
     void ABasicGeometryActor::BeginPlay(){
     	Super::BeginPlay();
     	
     	UE_LOG(LogTemp, Display, TEXT("Hello Unreal!"));
     	UE_LOG(LogTemp, Warning, TEXT("Hello Unreal!"));
     	UE_LOG(LogTemp, Error, TEXT("Hello Unreal!"));
     }

2.   在`窗口 => 开发者工具 => 输出日志`，将`输出日志`窗口唤出

3.   将`BasicGeometryActor`放入场景，点击运行，即可看到如下输出

     <img src="AssetMarkdown/image-20230117180838450.png" alt="image-20230117180838450" style="zoom:80%;" />

4.   可以通过`过滤器`，选择一部分日志输出

     <img src="AssetMarkdown/image-20230117181457957.png" alt="image-20230117181457957" style="zoom: 80%;" />

## 2.2	带参数输出

1.   与调用`printf()`类似

```c++
int WeaponNum = 4;
int KillNum = 7;
float Health = 34.435235f;
bool IsDead = false;
bool HasWeapon = true;
UE_LOG(LogTemp, Display, TEXT("WeaponsNum: %d, KillsNum: %i"), WeaponNum, KillNum);
UE_LOG(LogTemp, Display, TEXT("Health: %f"), Health);
UE_LOG(LogTemp, Display, TEXT("Health: %.2f"), Health);
UE_LOG(LogTemp, Display, TEXT("IsDead: %d"), IsDead);
UE_LOG(LogTemp, Display, TEXT("HasWeapon: %d"), static_cast<int>(HasWeapon));
```

<img src="AssetMarkdown/image-20230117182558048.png" alt="image-20230117182558048" style="zoom:80%;" />

## 2.3	新建private函数

1.   在`.h`文件中，创建函数的声明
2.   `右键 => 快速操作和重构 => 创建声明/定义`，即可在`.cpp`文件中创建函数的定义

## 2.4	磁盘中的日志文件

1.   存储在`工程目录/Saved/Logs`中

## 2.5	总结

```c++
void ABasicGeometryActor::printTypes(){
	int WeaponNum = 4;
	int KillNum = 7;
	float Health = 34.435235f;
	bool IsDead = false;
	bool HasWeapon = true;
	UE_LOG(LogTemp, Display, TEXT("WeaponsNum: %d, KillsNum: %i"), WeaponNum, KillNum);
	UE_LOG(LogTemp, Display, TEXT("Health: %f"), Health);
	UE_LOG(LogTemp, Display, TEXT("Health: %.2f"), Health);
	UE_LOG(LogTemp, Display, TEXT("IsDead: %d"), IsDead);
	UE_LOG(LogTemp, Display, TEXT("HasWeapon: %d"), static_cast<int>(HasWeapon));
}
```

# 三、FString类型 & 日志记录类别

## 3.1	自定义日志类别

1.   仅在单个`.cpp`中可用：`DEFINE_LOG_CATEGORY_STATIC`

     1.   `CategoryName`：日志类的名字
     2.   `DefaultVerbosity`：默认日志等级
     3.   `CompileTimeVerbosity`：编译时最高日志等级

     ```c++
     DEFINE_LOG_CATEGORY_STATIC(LogBasicGeometry, All, All);
     ```

## 3.2	日志的级别

1.   在`ELogVerbosity`中定义的枚举类：

     ```c++
     namespace ELogVerbosity {
     	enum Type : uint8 {
     		NoLogging	= 0,
     		Fatal,
     		Error,
     		Warning,
     		Display,
     		Log,
     		Verbose,
     		VeryVerbose,
     		All				= VeryVerbose,
     		NumVerbosity,
     		VerbosityMask	= 0xf,
     		SetColor		= 0x40, 
     		BreakOnLog		= 0x80
     	};
     }
     ```

2.   因此，如果自定义日志类的最高日志等级为`Error`，则只能输出类别为`Error/Fatal/NoLogging`的日志信息

## 3.3	字符串

1.   `TChar`：UE的字符类型

2.   `FString`：与`std::string`类似

     1.   用`UE_LOG`输出`FString`时，需要用`*`获得其字符串数组的首地址

          ```c++
          FString Name = "John Connor";
          UE_LOG(LogBasicGeometry, Display, TEXT("Name: %s"), *Name);
          ```

     2.   `FString`还支持一系列功能函数，从其它类型转化为`FString`、格式化构建`FString`

          ```c++
          int WeaponNum = 4;
          float Health = 34.435235f;
          bool IsDead = false;
          FString WeaponNumStr = "Weapons num = " + FString::FromInt(WeaponNum);
          FString HealthStr = "Health = " + FString::SanitizeFloat(Health);
          FString IsDeadStr = "IsDead = " + FString(IsDead ? "true" : "false");
          
          FString State = FString::Printf(TEXT("\n == All State == \n %s \n %s \n %s"), *WeaponNumStr, *HealthStr, *IsDeadStr);
          UE_LOG(LogBasicGeometry, Warning, TEXT("%s"), *State);
          ```

3.   `FName`

4.   `FText`

## 3.4	在屏幕上打印一条信息

1.   `GEngine->AddOnScreenDebugMessage()`

     1.   `int32 Key`：消息的键值，键值相同的消息不会重复显示，-1表示不使用该功能
     2.   `float TimeToDisplay`：消息在屏幕上停留的时间
     3.   `FColor DisplayColor`：消息的颜色
     4.   `const FString &DebugMessage`：消息的内容
     5.   `bool bNewerOnTop`：输出的顺序，在顶部新行还是底部新行，通常使用默认值
     6.   `const FVector2D &TextScale`：更改消息的大小，通常使用默认值

     ```c++
     #include "Engine/Engine.h"
     
     GEngine->AddOnScreenDebugMessage(-1, 3.0f, FColor::Red, Name);
     GEngine->AddOnScreenDebugMessage(-1, 5.0f, FColor::Green, State, true, FVector2D(1.5f, 1.5f));
     ```

## 3.5	总结

```c++
void ABasicGeometryActor::printStringTypes(){
	FString Name = "John Connor";
	int WeaponNum = 4;
	float Health = 34.435235f;
	bool IsDead = false;
	FString WeaponNumStr = "Weapons num = " + FString::FromInt(WeaponNum);
	FString HealthStr = "Health = " + FString::SanitizeFloat(Health);
	FString IsDeadStr = "IsDead = " + FString(IsDead ? "true" : "false");
	FString State = FString::Printf(TEXT("\n == All State == \n %s \n %s \n %s"), *WeaponNumStr, *HealthStr, *IsDeadStr);

	GEngine->AddOnScreenDebugMessage(-1, 3.0f, FColor::Red, Name);
	GEngine->AddOnScreenDebugMessage(-1, 5.0f, FColor::Green, State, true, FVector2D(1.5f, 1.5f));
}
```

# 四、宏`UPROPERTY`：定义Actor属性

作用：

1.   在编辑器中提供变量，并查看属性的访问说明符和元信息

## 4.1	定义变量

1.   定义变量

     ```c++
     UCLASS()
     class UE_CPP_API ABasicGeometryActor : public AActor{
     	GENERATED_BODY()
     	
     public:	
     	// Sets default values for this actor's properties
     	ABasicGeometryActor();
     
     protected:
     	// 在游戏开始或生成时调用
     	virtual void BeginPlay() override;
     	
     protected:
     	// 当前actor的属性
     	UPROPERTY(EditAnywhere, Category = "Weapon")
     	int32 WeaponNum = 4;
     
     	UPROPERTY(EditDefaultsOnly, Category = "State")
     	int32 KillNum = 7;
     
     	UPROPERTY(EditInstanceOnly, Category = "Health")
     	float Health = 34.435235f;
     
     	UPROPERTY(EditAnywhere, Category = "Health")
     	bool IsDead = false;
     
     	UPROPERTY(VisibleAnywhere, Category = "Weapon")
     	bool HasWeapon = true;
     
     public:	
     	// 每一帧调用一次
     	virtual void Tick(float DeltaTime) override;
     };
     ```

2.   不同说明符的含义：

     1.   `EditAnywhere`：在原型(父类/子类)、实例中均可编辑
     2.   `EditDefaultsOnly`：只能在原型(父类/子类)中编辑
     3.   `EditInstanceOnly`：只能在实例中编辑
     4.   `VisibleAnywhere`：在原型(父类/子类)、实例中均可见，但不可编辑
     5.   `VisibleDefaultsOnly`：只能在原型(父类/子类)中可见
     6.   `VisibleInstanceOnly`：只能在实例中可见

3.   `Category`：当前变量所属类别

## 4.2	获取Actor名称

```c++
UE_LOG(LogBasicGeometry, Warning, TEXT("Actor name %s"), *GetName());
```

# 五、组件--F变换类型`Transform`

## 5.1	为Actor创建网格体

```c++
#include "Components/StaticMeshComponent.h"
UCLASS()
class UE_CPP_API ABasicGeometryActor : public AActor{
	GENERATED_BODY()

public:
	// 组件的网格体
	UPROPERTY(VisibleAnywhere)
	UStaticMeshComponent* BaseMesh;
    ...
}
```

```c++
ABasicGeometryActor::ABasicGeometryActor(){
    ...
	// CreateDefaultSubobject: 创建一个默认的对象, 参数如下
	// (1)SubobjectName: 对象的名称
	// (2)bTransient = false: 
	BaseMesh = CreateDefaultSubobject<UStaticMeshComponent>("BaseMesh");
	// 指定根组件
	SetRootComponent(BaseMesh);
}
```

## 5.2	获取Actor的`Transform`信息

```c++
void ABasicGeometryActor::printTransform(){
	FTransform Transform = GetActorTransform();
	FVector Location = Transform.GetLocation();
	FRotator Rotator = Transform.Rotator();
	FVector Scale = Transform.GetScale3D();

	UE_LOG(LogBasicGeometry, Warning, TEXT("Actor name %s"), *GetName());
	UE_LOG(LogBasicGeometry, Warning, TEXT("Transform %s"), *Transform.ToString());
	UE_LOG(LogBasicGeometry, Warning, TEXT("Location %s"), *Location.ToString());
	UE_LOG(LogBasicGeometry, Warning, TEXT("Rotator %s"), *Rotator.ToString());
	UE_LOG(LogBasicGeometry, Warning, TEXT("Scale %s"), *Scale.ToString());
	UE_LOG(LogBasicGeometry, Error, TEXT("Human transform %s"), *Transform.ToHumanReadableString());
}
```

## 5.3	Actor随时间移动

```c++
UCLASS()
class UE_CPP_API ABasicGeometryActor : public AActor{
    ...
protected:
	// 当前actor的属性
	UPROPERTY(EditAnywhere, Category = "Movement")
	float Amplitude = 50.0f;
	UPROPERTY(EditAnywhere, Category = "Movement")
	float Frequency = 2.0f;
    ...
private:
	FVector InitialLocation;
};
```

```c++
// 在游戏开始或生成时调用
void ABasicGeometryActor::BeginPlay(){
	Super::BeginPlay();
	InitialLocation = GetActorLocation();
}

// 每一帧调用一次
void ABasicGeometryActor::Tick(float DeltaTime){
	Super::Tick(DeltaTime);

	// 运动轨迹: z = z0 + A * sin(f * t)
	FVector CurrentLocation = GetActorLocation();
	float Time = GetWorld()->GetTimeSeconds();
	CurrentLocation.Z = InitialLocation.Z + Amplitude * FMath::Sin(Frequency * Time);
	SetActorLocation(CurrentLocation);
}
```

# 六、宏`USTRUCT、UENUM`

## 6.1	扩展enum

```c++
// 在蓝图中可用
// UE的枚举类名称都是以E开头的
// uint8表示unsigend char, 表示该枚举的最大表示元素个数为255
UENUM(BlueprintType)
enum class EMovementType : uint8{
	Sin,
	Static
};
UCLASS()
class UE_CPP_API ABasicGeometryActor : public AActor
{
	...
protected:
	UPROPERTY(EditAnywhere, Category = "Movement")
	EMovementType MoveType = EMovementType::Static;
    ...
}
```

```c++
// 每一帧调用一次
void ABasicGeometryActor::Tick(float DeltaTime){
	Super::Tick(DeltaTime);

	switch (MoveType) {
	case EMovementType::Sin:
	{
		// 运动轨迹: z = z0 + A * sin(f * t)
		FVector CurrentLocation = GetActorLocation();
		float Time = GetWorld()->GetTimeSeconds();
		CurrentLocation.Z = InitialLocation.Z + Amplitude * FMath::Sin(Frequency * Time);
		SetActorLocation(CurrentLocation);
	}
		break;
	case EMovementType::Static:
		break;
	default:
		break;
	}
}
```

## 6.2	扩展struct

```c++
// USTRUCT(BlueprintType): 让该struct在蓝图中可用
// UE的结构体名称都是以F开头的
USTRUCT(BlueprintType)
struct FGeometryData {
	GENERATED_USTRUCT_BODY()
	
	UPROPERTY(EditAnywhere, Category = "Movement")
	float Amplitude = 50.0f;
	
	UPROPERTY(EditAnywhere, Category = "Movement")
	float Frequency = 2.0f;
	
	UPROPERTY(EditAnywhere, Category = "Movement")
	EMovementType MoveType = EMovementType::Static;
};
```
