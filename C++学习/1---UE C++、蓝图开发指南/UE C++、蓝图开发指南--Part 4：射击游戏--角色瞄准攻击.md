# 目录

[TOC]

# 一、ABaseWeapon类

1.   创建一个C++类`STUBaseWeapon`，继承于`Actor`

     1.   目录：`ShootThemUp/Source/ShootThemUp/Public/Weapon`

2.   在`ShootThemUp.Build.cs`中更新路径

     ```c#
     PublicIncludePaths.AddRange(new string[] { 
         "ShootThemUp/Public/Player", 
         "ShootThemUp/Public/Components", 
         "ShootThemUp/Public/Dev",
         "ShootThemUp/Public/Weapon"
     });
     ```

3.   修改`STUBaseWeapon`：创建属性--骨骼网格体组件

     ```c++
     #pragma once
     
     #include "CoreMinimal.h"
     #include "GameFramework/Actor.h"
     #include "STUBaseWeapon.generated.h"
     
     class USkeletalMeshComponent;
     
     UCLASS()
     class SHOOTTHEMUP_API ASTUBaseWeapon : public AActor {
         GENERATED_BODY()
     
     public:
         ASTUBaseWeapon();
     
     protected:
         // 武器的骨骼网格体
         UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Components")
         USkeletalMeshComponent* WeaponMesh;
         virtual void BeginPlay() override;
     };
     ```

     ```c++
     #include "Weapon/STUBaseWeapon.h"
     #include "Components/SkeletalMeshComponent.h"
     
     ASTUBaseWeapon::ASTUBaseWeapon() {
         PrimaryActorTick.bCanEverTick = false;
     
         WeaponMesh = CreateDefaultSubobject<USkeletalMeshComponent>("WeaponMesh");
         SetRootComponent(WeaponMesh);
     }
     
     void ASTUBaseWeapon::BeginPlay() {
         Super::BeginPlay();
     }
     

4.   创建一个文件夹`Weapon`，创建蓝图类`BP_STUBaseWeapon`

     1.   修改蓝图类，将默认骨骼网格体设置为`Rifle`

5.   修改角色骨骼体`HeroTPP`：创建插槽，用于后续设置生成武器的位置

     1.   在`b_RightWeapon`骨骼上，创建一个插槽，重命名为`WeaponSocket`
     2.   添加预览资产`Rifle`
     3.   在骨骼树中，修改步枪的位置：
     4.   在动画中，修改步枪的位置：位置`(0,0,0)`，旋转`(-90,0,0)`

6.   修改`STUBaseCharacter`：进入场景时生成武器

     1.   `AttachToComponent`：将actor附加到组件上，参数如下
          1.   `Parent`：指向要将此actor附加到的组件的指针
          2.   `AttachmentRules`：附加规则
          3.   `SocketName`：插槽的名称
     2.   `FAttachmentTransformRules`：附加组件时的变化规则
          1.   `EAttachmentRule`：规则的枚举类
               1.   `KeepRelative`：最终转换时，乘actor的相对转换矩阵
               2.   ``KeepWorld`：首先计算actor在世界中的位置，与该actor所连接的组件之间的相对位置，并将在最终转换矩阵中将其考虑在内
               3.   `SnapToTarget`：actor会继承附加到组件上的变换
          2.   创建`FAttachmentTransformRules`时，既可以分别指定`Location`/`Rotation`/`Scale`的规则，也可以同时指定为一种规则
          3.   `bInWeldSimulatedBodies`：附着时是否将模拟实体焊接在一起

     ```c++
     class ASTUBaseWeapon;
     
     UCLASS()
     class SHOOTTHEMUP_API ASTUBaseCharacter : public ACharacter {
     	...
     protected:
         // 武器的类别
         UPROPERTY(EditDefaultsOnly, Category = "Weapon")
         TSubclassOf<ASTUBaseWeapon> WeaponClass;
     
     private:
         // 生成武器
         void SpawnWeapon();
     };
     ```

     ```c++
     void ASTUBaseCharacter::BeginPlay() {
         Super::BeginPlay();
     
         // 检查组件是否成功创建(仅开发阶段可用)
         check(HealthComponent);
         check(HealthTextComponent);
         check(GetCharacterMovement());
     
         // 订阅OnDeath委托
         HealthComponent->OnDeath.AddUObject(this, &ASTUBaseCharacter::OnDeath);
         // 订阅OnHealthChanged委托
         HealthComponent->OnHealthChanged.AddUObject(this, &ASTUBaseCharacter::OnHealthChanged);
         // 先调用一次OnHealthChanged, 获取角色的初始血量
         OnHealthChanged(HealthComponent->GetHealth());
     
         // 订阅LandedDelegate委托
         LandedDelegate.AddDynamic(this, &ASTUBaseCharacter::OnGroundLanded);
     
         // 生成武器
         SpawnWeapon();
     }
     
     // 生成武器
     void ASTUBaseCharacter::SpawnWeapon() {
         if (!GetWorld()) return;
         // 生成actor
         const auto Weapon = GetWorld()->SpawnActor<ASTUBaseWeapon>(WeaponClass);
         // 将actor绑定到角色身上
         if (Weapon) {
             FAttachmentTransformRules AttachmentRules(EAttachmentRule::SnapToTarget, false);
             Weapon->AttachToComponent(GetMesh(), AttachmentRules, "WeaponSocket");
         }
     }

7.   修改`BP_STUBaseCharacter`：设置默认生成的武器类为`BP_STUBaseWeapon`

# 二、AHUD类：十字瞄准线

>   始终处于屏幕中间的十字瞄准线

1.   `HUD`：Head Up Display，此类负责渲染界面，该界面将叠加在游戏顶部

     1.   我们可以继承该类，并使用此类绘制所需的所有元素

2.   创建C++类`STUGameHUD`，继承于`HUD`

     1.   目录：`ShootThemUp/Source/ShootThemUp/Public/UI`

3.   在`ShootThemUp.Build.cs`中更新路径

     ```c#
     PublicIncludePaths.AddRange(new string[] { 
         "ShootThemUp/Public/Player", 
         "ShootThemUp/Public/Components", 
         "ShootThemUp/Public/Dev",
         "ShootThemUp/Public/Weapon",
         "ShootThemUp/Public/UI"
     });
     ```

4.   修改`STUGameHUD`：绘制中心瞄准线

     ```c++
     #pragma once
     
     #include "CoreMinimal.h"
     #include "GameFramework/HUD.h"
     #include "STUGameHUD.generated.h"
     
     UCLASS()
     class SHOOTTHEMUP_API ASTUGameHUD : public AHUD
     {
     	GENERATED_BODY()
     public:
         virtual void DrawHUD() override;
     
     private:
         // 绘制屏幕中心的十字准线
         void DrawCrossHair();
     };
     ```

     ```c++
     #include "UI/STUGameHUD.h"
     #include "Engine/Canvas.h"
     
     void ASTUGameHUD::DrawHUD() {
         Super::DrawHUD();
         DrawCrossHair();
     }
     
     // 绘制屏幕中心的十字准线
     void ASTUGameHUD::DrawCrossHair() {
         const float CenterX = Canvas->SizeX * 0.5f, CenterY = Canvas->SizeY * 0.5f;
     
         
         const float HalfLineSize = 10.0f;
         const float LineThickness = 2.0f;
         const FLinearColor LineColor = FLinearColor::Green;
     
         // 水平线
         DrawLine(CenterX - HalfLineSize, CenterY, CenterX + HalfLineSize, CenterY, LineColor, LineThickness);
         // 垂直线
         DrawLine(CenterX, CenterY - HalfLineSize, CenterX, CenterY + HalfLineSize, LineColor, LineThickness);
     }

5.   修改`STUGameModeBase`：修改默认HUD类

     ```c++
     #include "STUGameModeBase.h"
     #include "Player/STUBaseCharacter.h"
     #include "Player/STUPlayerController.h"
     #include "UI/STUGameHUD.h"
     
     ASTUGameModeBase::ASTUGameModeBase() {
         DefaultPawnClass = ASTUBaseCharacter::StaticClass();
         PlayerControllerClass = ASTUPlayerController::StaticClass();
         HUDClass = ASTUGameHUD::StaticClass();
     }
     ```

6.   修改`BP_STUBaseCharacter/SpringArmComponent`：修改相机的位置

     <img src="AssetMarkdown/image-20230203135405181.png" alt="image-20230203135405181" style="zoom:80%;" />

7.   修改`BP_STUBaseCharacter/HealthTextComponent`：隐藏自己的血量，但

     <img src="AssetMarkdown/image-20230203135604101.png" alt="image-20230203135604101" style="zoom:80%;" />

8.   也可以通过C++修改上述设置：`STUBaseCharacter`

     ```c++
     // 由于CharacterMovementComponent组件是默认组件, 因此我们需要通过参数显式指定
     // 在调用父类的构造函数时, 显式指定CharacterMovementComponentName使用自定义的USTUCharacterMovementComponent
     ASTUBaseCharacter::ASTUBaseCharacter(const FObjectInitializer& ObjInit) 
         : Super(ObjInit.SetDefaultSubobjectClass<USTUCharacterMovementComponent>(ACharacter::CharacterMovementComponentName)) {
         // 允许该character每一帧调用Tick()
         PrimaryActorTick.bCanEverTick = true;
     
         // 创建弹簧臂组件, 并设置其父组件为根组件, 允许pawn控制旋转
         SpringArmComponent = CreateDefaultSubobject<USpringArmComponent>("SpringArmComponent");
         SpringArmComponent->SetupAttachment(GetRootComponent());
         SpringArmComponent->bUsePawnControlRotation = true;
         SpringArmComponent->SocketOffset = FVector(0.0f, 100.0f, 80.0f);
     
         // 创建相机组件, 并设置其父组件为弹簧臂组件
         CameraComponent = CreateDefaultSubobject<UCameraComponent>("CameraComponent");
         CameraComponent->SetupAttachment(SpringArmComponent);
     
         // 创建血量组件, 由于其是纯逻辑的, 不需要设置父组件
         HealthComponent = CreateDefaultSubobject<USTUHealthComponent>("STUHealthComponent");
     
         // 创建血量显示组件, 并设置其父组件为根组件
         HealthTextComponent = CreateDefaultSubobject<UTextRenderComponent>("TextRenderComponent");
         HealthTextComponent->SetupAttachment(GetRootComponent());
         HealthTextComponent->SetOwnerNoSee(true);
     }
     ```

# 三、武器组件

> 创建一个组件，用于管理武器相关的内容

1. 添加操作映射：`Fire` => `鼠标左键`

   <img src="AssetMarkdown/image-20230207213851843.png" alt="image-20230207213851843" style="zoom:80%;" />

2. 创建C++类`STUWeaponComponent`，继承于`Actor组件`

   1. 目录：`ShootThemUp/Source/ShootThemUp/Public/Components`

3. 将`SpawnWeapon`函数移到`STUWeaponComponent`中

4. 修改`STUWeaponComponent`：创建`Fire`调用

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "Components/ActorComponent.h"
   #include "STUWeaponComponent.generated.h"
   
   class ASTUBaseWeapon;
   
   UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
   class SHOOTTHEMUP_API USTUWeaponComponent : public UActorComponent {
       GENERATED_BODY()
   
   public:
       USTUWeaponComponent();
   
       // 开火
       void Fire();
   
   protected:
       // 武器的类别
       UPROPERTY(EditDefaultsOnly, Category = "Weapon")
       TSubclassOf<ASTUBaseWeapon> WeaponClass;
       // 武器绑定的插槽名称
       UPROPERTY(EditDefaultsOnly, Category = "Weapon")
       FName WeaponAttachPointName = "WeaponSocket";
   
       virtual void BeginPlay() override;
   
   private:
       // 当前武器
       UPROPERTY()
       ASTUBaseWeapon* CurrentWeapon = nullptr;
   
       // 生成武器
       void SpawnWeapon();
   };
   ```

   ```c++
   #include "Components/STUWeaponComponent.h"
   #include "Weapon/STUBaseWeapon.h"
   #include "GameFramework/Character.h"
   
   USTUWeaponComponent::USTUWeaponComponent() {
       PrimaryComponentTick.bCanEverTick = false;
   }
   
   void USTUWeaponComponent::BeginPlay() {
       Super::BeginPlay();
   
       // 生成武器
       SpawnWeapon();
   }
   
   // 生成武器
   void USTUWeaponComponent::SpawnWeapon() {
       if (!GetWorld()) return;
       
       // 判断角色是否存在
       ACharacter* Character = Cast<ACharacter>(GetOwner());
       if (!Character) return;
       
       // 生成actor
       CurrentWeapon = GetWorld()->SpawnActor<ASTUBaseWeapon>(WeaponClass);
       if (!CurrentWeapon) return;
   
       // 将actor绑定到角色身上
       FAttachmentTransformRules AttachmentRules(EAttachmentRule::SnapToTarget, false);
       CurrentWeapon->AttachToComponent(Character->GetMesh(), AttachmentRules, WeaponAttachPointName);
   }
   
   // 开火
   void USTUWeaponComponent::Fire() {
       if (!CurrentWeapon) return;
       CurrentWeapon->Fire();
   }

5. 修改`STUBaseWeapon`：创建`Fire`的接口

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "GameFramework/Actor.h"
   #include "STUBaseWeapon.generated.h"
   
   class USkeletalMeshComponent;
   
   UCLASS()
   class SHOOTTHEMUP_API ASTUBaseWeapon : public AActor {
       GENERATED_BODY()
   
   public:
       ASTUBaseWeapon();
   
       // 开火, 不同武器会有不同的开火方式
       virtual void Fire();
   
   protected:
       // 武器的骨骼网格体
       UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Components")
       USkeletalMeshComponent* WeaponMesh;
       virtual void BeginPlay() override;
   };
   
   ```

   ```c++
   #include "Weapon/STUBaseWeapon.h"
   #include "Components/SkeletalMeshComponent.h"
   
   DEFINE_LOG_CATEGORY_STATIC(LogSTUBaseWeapon, All, All);
   
   ASTUBaseWeapon::ASTUBaseWeapon() {
       PrimaryActorTick.bCanEverTick = false;
   
       WeaponMesh = CreateDefaultSubobject<USkeletalMeshComponent>("WeaponMesh");
       SetRootComponent(WeaponMesh);
   }
   
   void ASTUBaseWeapon::BeginPlay() {
       Super::BeginPlay();
   }
   
   // 开火
   void ASTUBaseWeapon::Fire() {
       UE_LOG(LogSTUBaseWeapon, Warning, TEXT("Fire with Basic Weapon"));
   }

6. 修改`STUBaseCharacter`：创建武器组件

   ```c++
   class USTUWeaponComponent;
   
   UCLASS()
   class SHOOTTHEMUP_API ASTUBaseCharacter : public ACharacter {
       ...
   protected:
       // 组件：武器管理
       UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Components")
       USTUWeaponComponent* WeaponComponent;
       ...
   }
   ```

   ```c++
   ASTUBaseCharacter::ASTUBaseCharacter(const FObjectInitializer& ObjInit)
       : Super(ObjInit.SetDefaultSubobjectClass<USTUCharacterMovementComponent>(ACharacter::CharacterMovementComponentName)) {
       ...
           
       // 创建武器组件, 由于其是纯逻辑的, 不需要设置父组件
       WeaponComponent = CreateDefaultSubobject<USTUWeaponComponent>("STUWeaponComponent");
   }
   
   void ASTUBaseCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) {
       ...
       // 鼠标左键控制武器开火
       PlayerInputComponent->BindAction("Fire", IE_Pressed, WeaponComponent, &USTUWeaponComponent::Fire);
   }


# 四、碰撞体积概述

1. 调试信息，显示碰撞体积：`show collision`

2. 碰撞矩阵：

   1. 当`Camera`选为`忽略/重叠`时，角色可以与该物体重叠，但相机不会被干扰

   <img src="AssetMarkdown/image-20230206191720376.png" alt="image-20230206191720376" style="zoom:80%;" />

3. 生成重叠事件：在`细节|碰撞`中，勾选`生成重叠事件`，即可在角色与该物体重叠时，角色的事件图表中可以接收到`事件Actor开始重叠`

   <img src="AssetMarkdown/image-20230206192305760.png" alt="image-20230206192305760" style="zoom:80%;" />

4. 获取碰撞信息：在`细节|碰撞`中，勾选`模拟生产命中事件`，即可在角色与该物体重叠时，角色的事件图表中可以接收到`事件命中`

   <img src="AssetMarkdown/image-20230206192343836.png" alt="image-20230206192343836" style="zoom:80%;" />

5. 修改碰撞设置：在`项目设置|碰撞`中，可以添加、修改碰撞预设

   1. 添加两种角色`Enemy`、`Geometry`
   2. 此时可以发现，在物体的`细节|碰撞|碰撞预设|对象类型`中，多了这两个选项

   <img src="AssetMarkdown/image-20230206192734818.png" alt="image-20230206192734818" style="zoom:80%;" />

# 五、发射轨迹

1. 使用`插槽`获取枪口坐标：

   1. 进入武器`Rifle`的骨骼树，在根骨骼下添加一个插槽，重命名为`MuzzleSocket`
   2. 将该插槽移动至`Rifle`的枪口位置

   <img src="AssetMarkdown/image-20230206193355594.png" alt="image-20230206193355594" style="zoom:80%;" />

2. 修改`STUBaseWeapon`：添加子弹沿直线射击功能

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUBaseWeapon : public AActor {
       ...
   
   public:
       // 开火, 不同武器会有不同的开火方式
       virtual void Fire();
   
   protected:
       // 武器枪口的插槽名称
       UPROPERTY(VisibleAnywhere, BlueprintReadWrite)
       FName MuzzleSocketName = "MuzzleSocket";
   
       // 子弹的最大路程
       UPROPERTY(VisibleAnywhere, BlueprintReadWrite)
       float TraceMaxDistance = 1500;
   
       // 发射子弹
       virtual void MakeShot();
   };
   ```

   ```c++
   #include "Engine/World.h"
   #include "DrawDebugHelpers.h"
   
   void ASTUBaseWeapon::BeginPlay() {
       Super::BeginPlay();
       check(WeaponMesh);
   }
   
   // 开火
   void ASTUBaseWeapon::Fire() {
       UE_LOG(LogSTUBaseWeapon, Warning, TEXT("Fire with Basic Weapon"));
       MakeShot();
   }
   
   // 发射子弹
   void ASTUBaseWeapon::MakeShot() {
       if (!GetWorld()) return;
   
       // 从枪口插槽中, 获取枪口位置、方向
       const FTransform SocketTransform = WeaponMesh->GetSocketTransform(MuzzleSocketName);
       const FVector TraceStart = SocketTransform.GetLocation();
       const FVector ShootDirection = SocketTransform.GetRotation().GetForwardVector();
       const FVector TraceEnd = TraceStart + ShootDirection * TraceMaxDistance;
   
       // 绘制子弹的路径
       DrawDebugLine(GetWorld(), TraceStart, TraceEnd, FColor::Red, false, 3.0f, 0, 3.0f);
   
       // 获取子弹路径上，第一个碰撞到的对象
       FHitResult HitResult;
       GetWorld()->LineTraceSingleByChannel(HitResult, TraceStart, TraceEnd, ECollisionChannel::ECC_Visibility);
   
       // 碰撞到了某个物体
       if (HitResult.bBlockingHit) {
           DrawDebugSphere(GetWorld(), HitResult.ImpactPoint, 10.0f, 24, FColor::Red, false, 5.0f);
       }
   }

3. 修改`STUWeaponComponent`：设置武器的所有者

   ```c++
   // 生成武器
   void USTUWeaponComponent::SpawnWeapon() {
       if (!GetWorld()) return;
       
       // 判断角色是否存在
       ACharacter* Character = Cast<ACharacter>(GetOwner());
       if (!Character) return;
       
       // 生成actor
       CurrentWeapon = GetWorld()->SpawnActor<ASTUBaseWeapon>(WeaponClass);
       if (!CurrentWeapon) return;
   
       // 将武器绑定到角色身上
       FAttachmentTransformRules AttachmentRules(EAttachmentRule::SnapToTarget, false);
       CurrentWeapon->AttachToComponent(Character->GetMesh(), AttachmentRules, WeaponAttachPointName);
       CurrentWeapon->SetOwner(Character); // 设置武器的所有者
   }
   ```

4. 修改`STUBaseWeapon`：让子弹射向瞄准的位置

   ```c++
   // 发射子弹
   void ASTUBaseWeapon::MakeShot() {
       if (!GetWorld()) return;
   
       // 获取武器的所有者: 即玩家
       const auto Player = Cast<ACharacter>(GetOwner());
       if (!Player) return;
       const auto Controller = Player->GetController<APlayerController>();
       if (!Controller) return;
   
       // 获取玩家的位置和朝向
       FVector ViewLocation;
       FRotator ViewRotation;
       Controller->GetPlayerViewPoint(ViewLocation, ViewRotation);
   
       // 子弹路径为: 角色当前位置 -> 角色面朝方向
       const FTransform SocketTransform = WeaponMesh->GetSocketTransform(MuzzleSocketName);
       const FVector TraceStart = ViewLocation;
       const FVector ShootDirection = ViewRotation.Vector();
       const FVector TraceEnd = TraceStart + ShootDirection * TraceMaxDistance;
   
       // 获取子弹路径上，第一个碰撞到的对象
       FHitResult HitResult;
       GetWorld()->LineTraceSingleByChannel(HitResult, TraceStart, TraceEnd, ECollisionChannel::ECC_Visibility);
   
       if (HitResult.bBlockingHit) {
           // 绘制子弹的路径: 枪口位置 -> 碰撞点
           DrawDebugLine(GetWorld(), SocketTransform.GetLocation(), HitResult.ImpactPoint, FColor::Red, false, 3.0f, 0, 3.0f);
           // 在碰撞处绘制一个球
           DrawDebugSphere(GetWorld(), HitResult.ImpactPoint, 10.0f, 24, FColor::Red, false, 5.0f);
       } else {
           // 绘制子弹的路径: 枪口位置 -> 子弹路径的终点
           DrawDebugLine(GetWorld(), SocketTransform.GetLocation(), TraceEnd, FColor::Red, false, 3.0f, 0, 3.0f);
       }
   }
   ```

5. 问题1：子弹是打在胶囊体上，而不是角色的网格体上

   1. 修改角色的胶囊体组件的大小，可以看到，子弹打到角色的胶囊体上就停止移动了
   2. 将`胶囊体组件`的碰撞预设的`检测响应|Visibility`设置为`忽略`、`网格体组件`的碰撞预设的`检测响应|Visibility`设置为`阻挡`，即可打击到角色的网格体，而非胶囊体上

6. 问题2：即使我们在敌人的身后，也可以打到敌人

   1. 也就是说，子弹的路径可能与枪的夹角为锐角

7. 问题3：可能会打到自己

   1. 检测碰撞时，添加一个参数`CollisionParams`，用于忽略自己

   ```c++
   // 发射子弹
   void ASTUBaseWeapon::MakeShot() {
       if (!GetWorld()) return;
   
       // 获取武器的所有者: 即玩家
       const auto Player = Cast<ACharacter>(GetOwner());
       if (!Player) return;
       const auto Controller = Player->GetController<APlayerController>();
       if (!Controller) return;
   
       // 获取玩家的位置和朝向
       FVector ViewLocation;
       FRotator ViewRotation;
       Controller->GetPlayerViewPoint(ViewLocation, ViewRotation);
   
       // 子弹路径为: 角色当前位置 -> 角色面朝方向
       const FTransform SocketTransform = WeaponMesh->GetSocketTransform(MuzzleSocketName);
       const FVector TraceStart = ViewLocation;
       const FVector ShootDirection = ViewRotation.Vector();
       const FVector TraceEnd = TraceStart + ShootDirection * TraceMaxDistance;
   
       // 获取子弹路径上，第一个碰撞到的对象
       FCollisionQueryParams CollisionQueryParams;
       CollisionQueryParams.AddIgnoredActor(GetOwner());   // 忽略自己
       FHitResult HitResult;                               // 碰撞的结果
       GetWorld()->LineTraceSingleByChannel(HitResult, TraceStart, TraceEnd, ECollisionChannel::ECC_Visibility, CollisionQueryParams);
   
       if (HitResult.bBlockingHit) {
           // 绘制子弹的路径: 枪口位置 -> 碰撞点
           DrawDebugLine(GetWorld(), SocketTransform.GetLocation(), HitResult.ImpactPoint, FColor::Red, false, 3.0f, 0, 3.0f);
           // 在碰撞处绘制一个球
           DrawDebugSphere(GetWorld(), HitResult.ImpactPoint, 10.0f, 24, FColor::Red, false, 5.0f);
           
           // 显示碰撞到了哪个骨骼上, 可以通过这个信息对角色造成不同的伤害
           UE_LOG(LogSTUBaseWeapon, Display, TEXT("Fire hit bone: %s"), *HitResult.BoneName.ToString());
       } else {
           // 绘制子弹的路径: 枪口位置 -> 子弹路径的终点
           DrawDebugLine(GetWorld(), SocketTransform.GetLocation(), TraceEnd, FColor::Red, false, 3.0f, 0, 3.0f);
       }
   }
   ```

# 六、重构MakeShot()函数

> 将MakeShot()函数重构为多个方法，让代码更具有可读性

1. 修改`STUBaseWeapon`：重构MakeShot()

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "GameFramework/Actor.h"
   #include "STUBaseWeapon.generated.h"
   
   class USkeletalMeshComponent;
   
   UCLASS()
   class SHOOTTHEMUP_API ASTUBaseWeapon : public AActor {
       ...
   protected:
       // 发射子弹
       virtual void MakeShot();
       // 获取玩家控制器
       APlayerController* GetPlayerController() const;
       // 获取玩家的位置和朝向
       bool GetPlayerViewPoint(FVector& ViewLocation, FRotator& ViewRotation) const;
       // 获取枪口的位置
       FVector GetMuzzleWorldLocation() const;
       // 获取子弹的逻辑路径
       bool GetTraceData(FVector& TraceStart, FVector& TraceEnd) const;
       // 执行碰撞逻辑
       void MakeHit(FHitResult& HitResult, const FVector& TraceStart, const FVector& TraceEnd) const;
    };
   ```

   ```c++
   // 发射子弹
   void ASTUBaseWeapon::MakeShot() {
       if (!GetWorld()) return;
       
       // 获取子弹的逻辑路径
       FVector TraceStart, TraceEnd;
       if (!GetTraceData(TraceStart, TraceEnd)) return;
   
       // 计算子弹的碰撞结果
       FHitResult HitResult;
       MakeHit(HitResult, TraceStart, TraceEnd);
       
       if (HitResult.bBlockingHit) {
           // 绘制子弹的路径: 枪口位置 -> 碰撞点
           DrawDebugLine(GetWorld(), GetMuzzleWorldLocation(), HitResult.ImpactPoint, FColor::Red, false, 3.0f, 0, 3.0f);
           // 在碰撞处绘制一个球
           DrawDebugSphere(GetWorld(), HitResult.ImpactPoint, 10.0f, 24, FColor::Red, false, 5.0f);
           
           // 显示碰撞到了哪个骨骼上, 可以通过这个信息对角色造成不同的伤害
           UE_LOG(LogSTUBaseWeapon, Display, TEXT("Fire hit bone: %s"), *HitResult.BoneName.ToString());
       } else {
           // 绘制子弹的路径: 枪口位置 -> 子弹路径的终点
           DrawDebugLine(GetWorld(), GetMuzzleWorldLocation(), TraceEnd, FColor::Red, false, 3.0f, 0, 3.0f);
       }
   }
   
   // 获取玩家控制器
   APlayerController* ASTUBaseWeapon::GetPlayerController() const {
       const auto Player = Cast<ACharacter>(GetOwner());
       if (!Player) return nullptr;
       return Player->GetController<APlayerController>();
   }
   
   // 获取玩家的位置和朝向
   bool ASTUBaseWeapon::GetPlayerViewPoint(FVector& ViewLocation, FRotator& ViewRotation) const {
       const auto Controller = GetPlayerController();
       if (!Controller) return false;
   
       Controller->GetPlayerViewPoint(ViewLocation, ViewRotation);
       return true;
   }
   
   // 获取枪口的位置
   FVector ASTUBaseWeapon::GetMuzzleWorldLocation() const {
       return WeaponMesh->GetSocketLocation(MuzzleSocketName);
   }
   
   // 获取子弹的逻辑路径
   bool ASTUBaseWeapon::GetTraceData(FVector& TraceStart, FVector& TraceEnd) const {
       // 获取玩家的位置和朝向
       FVector ViewLocation;
       FRotator ViewRotation;
       if (!GetPlayerViewPoint(ViewLocation, ViewRotation)) return false;
   
       // 子弹路径为: 角色当前位置 -> 角色面朝方向
       TraceStart = ViewLocation;
       const FVector ShootDirection = ViewRotation.Vector();
       TraceEnd = TraceStart + ShootDirection * TraceMaxDistance;
       return true;
   }
   
   // 执行碰撞逻辑
   void ASTUBaseWeapon::MakeHit(FHitResult& HitResult, const FVector& TraceStart, const FVector& TraceEnd) const {
       if (!GetWorld()) return;
       
       // 忽略自己
       FCollisionQueryParams CollisionQueryParams;
       CollisionQueryParams.AddIgnoredActor(GetOwner());  
       
       // 获取子弹路径上，第一个碰撞到的对象，存储到HitResult中
       GetWorld()->LineTraceSingleByChannel(HitResult, TraceStart, TraceEnd, ECollisionChannel::ECC_Visibility, CollisionQueryParams);
   }

# 七、实战作业：伤害机制

> 当子弹打到player身上时，通过TakeDamage()功能，对其造成伤害

1. 修改`STUBaseWeapon`：添加伤害机制

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUBaseWeapon : public AActor {
       ...
   
   protected:
       // 武器造成的伤害
       UPROPERTY(VisibleAnywhere, BlueprintReadWrite)
       float DamageAmount = 10.0f;
   
   protected:
       // 发射子弹
       virtual void MakeShot();
       // 对子弹击中的玩家进行伤害
       void MakeDamage(const FHitResult& HitResult) const;
   
   };
   ```

   ```c++
   // 发射子弹
   void ASTUBaseWeapon::MakeShot() {
       if (!GetWorld()) return;
       
       // 获取子弹的逻辑路径
       FVector TraceStart, TraceEnd;
       if (!GetTraceData(TraceStart, TraceEnd)) return;
   
       // 计算子弹的碰撞结果
       FHitResult HitResult;
       MakeHit(HitResult, TraceStart, TraceEnd);
       
       if (HitResult.bBlockingHit) {
           // 对子弹击中的玩家进行伤害
           MakeDamage(HitResult);
           ...
       }
   }
   
   // 对子弹击中的玩家进行伤害
   void ASTUBaseWeapon::MakeDamage(const FHitResult& HitResult) const {
       const auto DamageActor = HitResult.GetActor();
       if (!DamageActor) return;
   
       DamageActor->TakeDamage(DamageAmount, FDamageEvent{}, GetPlayerController(), nullptr);
   }

2. 修改`STUBaseCharacter`：当角色死亡时，禁用胶囊体碰撞

   ```c++
   void ASTUBaseCharacter::OnDeath() {
       UE_LOG(LogSTUBaseCharacter, Warning, TEXT("Player %s is dead"), *GetName());
       // 播放死亡动画蒙太奇
       PlayAnimMontage(DeathAnimMontage);
       // 禁止角色的移动
       GetCharacterMovement()->DisableMovement();
       // 一段时间后摧毁角色
       SetLifeSpan(LifeSpanOnDeath);
       // 切换状态, 从而将pawn切换为观察者类
       if (Controller) {
           Controller->ChangeState(NAME_Spectating);
       }
   
       // 禁止胶囊体碰撞
       GetCapsuleComponent()->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
   }

# 八、瞄准偏移动画

> 添加目标位移动画：上、下、左、右，使射击时瞄准动作准确

1. 修改角色的9个瞄准偏移动画

   1. 这些动画都是加法动画，是在原有动画的基础上，进行修改
   2. `附加设置|Additive动画类型`：设置为`网格体空间`
   3. `附加设置|基础姿势动画`：设置为`Idle`

2. 创建瞄准偏移动画`AO_BaseCharacter`

   1. 修改水平及垂直坐标轴：

      1. 水平坐标：`-90~90`
      2. 垂直坐标：`-90~90`

   2. 将9个瞄准偏移动画拖放到坐标轴上：

      <img src="AssetMarkdown/image-20230207172719770.png" alt="image-20230207172719770" style="zoom:80%;" />

3. 修改角色的动画蓝图`ABP_BaseCharacter`：

   1. 修改事件图表：获取相机的旋转矢量

      <img src="AssetMarkdown/image-20230207173207428.png" alt="image-20230207173207428" style="zoom:80%;" />

   2. 修改AnimGraph：

      <img src="AssetMarkdown/image-20230207173352715.png" alt="image-20230207173352715" style="zoom:80%;" />

# 九、动画逆运动：固定左手位置

> 固定左手相对于右手的位置，从而决定整个左臂的位置

1. 通过将相机对准角色的前面，可以看到，当角色移动时，角色的手会乱晃

2. 可以通过固定手的位置来解决该问题

3. 修改角色动画蓝图`ABP_BaseCharacter`的`AnimGraph`

   1. 添加节点`FABRIK`：Forward And Backward Reaching Inverse Kinematics

      <img src="AssetMarkdown/image-20230207210707683.png" alt="image-20230207210707683" style="zoom:80%;" />

   2. 修改节点属性：

   <img src="AssetMarkdown/image-20230207210625316.png" alt="image-20230207210625316" style="zoom:80%;" />

# 十、自动发射子弹，子弹随机散射

> 实现功能：长按左键连发子弹

1. 修改`STUBaseCharacter`：修改对`Fire`的操作响应

   ```c++
   void ASTUBaseCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) {
       ...
       // 鼠标左键控制武器开火
       PlayerInputComponent->BindAction("Fire", IE_Pressed, WeaponComponent, &USTUWeaponComponent::StartFire);
       PlayerInputComponent->BindAction("Fire", IE_Released, WeaponComponent, &USTUWeaponComponent::StopFire);
   }
   ```

2. 修改`STUWeaponComponent`：实现`StartFire`和`StopFire`

   ```c++
   UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
   class SHOOTTHEMUP_API USTUWeaponComponent : public UActorComponent {
       ...
   public:
       // 开火
       void StartFire();
       // 停止开火
       void StopFire();
       ...
   }
   ```

   ```c++
   // 开火
   void USTUWeaponComponent::StartFire() {
       if (!CurrentWeapon) return;
       CurrentWeapon->StartFire();
   }
   // 停止开火
   void USTUWeaponComponent::StopFire() {
       if (!CurrentWeapon) return;
       CurrentWeapon->StopFire();
   }
   ```

3. 修改`STUBaseWeapon`：借助定时器实现连续开火功能

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUBaseWeapon : public AActor {
      	...
   public:
       // 开火, 不同武器会有不同的开火方式
       virtual void StartFire();
       // 停止开火
       virtual void StopFire();
   
   protected:
       // 自动开火的时间间隔
       UPROPERTY(VisibleAnywhere, BlueprintReadWrite)
       float TimeBetweenShots = 0.1f;
   
   private:
       // 自动开火的定时器
       FTimerHandle ShotTimerHandle;
   };
   ```

   ```c++
   // 开火, 不同武器会有不同的开火方式
   void ASTUBaseWeapon::StartFire() {
       MakeShot();
       GetWorldTimerManager().SetTimer(ShotTimerHandle, this, &ASTUBaseWeapon::MakeShot, TimeBetweenShots, true);
   }
   // 停止开火
   void ASTUBaseWeapon::StopFire() {
       GetWorldTimerManager().ClearTimer(ShotTimerHandle);
   }
   ```

4. 修改`STUBaseWeapon::GetTraceData()`：每次开火有一个随机的偏移

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUBaseWeapon : public AActor {
       ...
   protected:
       // 子弹的随机偏移角度
       UPROPERTY(VisibleAnywhere, BlueprintReadWrite)
       float BulletSpread = 1.5f;
   };
   ```

   ```c++
   // 获取子弹的逻辑路径
   bool ASTUBaseWeapon::GetTraceData(FVector& TraceStart, FVector& TraceEnd) const {
       // 获取玩家的位置和朝向
       FVector ViewLocation;
       FRotator ViewRotation;
       if (!GetPlayerViewPoint(ViewLocation, ViewRotation)) return false;
   
       // 子弹的起点为: 角色当前位置
       TraceStart = ViewLocation;
       // 子弹的方向为: 角色当前正前方 + 一个随机的偏移
       // 当角色血量很低时, 偏移角可以变大, 模拟角色打不准的情况
       const auto HalfRad = FMath::DegreesToRadians(BulletSpread);
       const FVector ShootDirection = FMath::VRandCone(ViewRotation.Vector(), HalfRad);
       // 子弹的终点为: 角色当前位置沿子弹方向运动一定的距离
       TraceEnd = TraceStart + ShootDirection * TraceMaxDistance;
       return true;
   }
   ```

# 十一、两个新武器类别建模：步枪和榴弹发射器

1. 创建两个C++类`STURifleWeapon`、`STULauncherWeapon`，继承于`STUBaseWeapon`

   1. 目录：`ShootThemUp/Source/ShootThemUp/Public/Weapon`

2. 修改`STUBaseWeapon`：

   1. 将`自动开火`的功能移植到`步枪`中，`榴弹发射器`不能自动开火
   2. 将`开火随机偏移`的功能移植到`步枪`中，`榴弹发射器`不会有随机偏移

3. 修改`STURifleWeapon`

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "Weapon/STUBaseWeapon.h"
   #include "STURifleWeapon.generated.h"
   
   UCLASS()
   class SHOOTTHEMUP_API ASTURifleWeapon : public ASTUBaseWeapon {
       GENERATED_BODY()
   public:
       virtual void StartFire() override;
       virtual void StopFire() override;
   
   protected:
       // 自动开火的时间间隔
       UPROPERTY(VisibleAnywhere, BlueprintReadWrite)
       float TimeBetweenShots = 0.1f;
       // 子弹的随机偏移角度
       UPROPERTY(VisibleAnywhere, BlueprintReadWrite)
       float BulletSpread = 1.5f;
       // 发射子弹
       virtual void MakeShot() override;
       // 获取子弹的逻辑路径
       virtual bool GetTraceData(FVector& TraceStart, FVector& TraceEnd) const override;
   
   private:
       // 自动开火的定时器
       FTimerHandle ShotTimerHandle;
   };
   ```

   ```c++
   #include "Weapon/STURifleWeapon.h"
   #include "Engine/World.h"
   #include "DrawDebugHelpers.h"
   
   DEFINE_LOG_CATEGORY_STATIC(LogSTURifleWeapon, All, All);
   
   // 开火, 不同武器会有不同的开火方式
   void ASTURifleWeapon::StartFire() {
       MakeShot();
       GetWorldTimerManager().SetTimer(ShotTimerHandle, this, &ASTURifleWeapon::MakeShot, TimeBetweenShots, true);
   }
   // 停止开火
   void ASTURifleWeapon::StopFire() {
       GetWorldTimerManager().ClearTimer(ShotTimerHandle);
   }
   
   // 发射子弹
   void ASTURifleWeapon::MakeShot() {
       if (!GetWorld()) return;
   
       // 获取子弹的逻辑路径
       FVector TraceStart, TraceEnd;
       if (!GetTraceData(TraceStart, TraceEnd)) return;
   
       // 计算子弹的碰撞结果
       FHitResult HitResult;
       MakeHit(HitResult, TraceStart, TraceEnd);
   
       if (HitResult.bBlockingHit) {
           // 对子弹击中的玩家进行伤害
           MakeDamage(HitResult);
   
           // 绘制子弹的路径: 枪口位置 -> 碰撞点
           DrawDebugLine(GetWorld(), GetMuzzleWorldLocation(), HitResult.ImpactPoint, FColor::Red, false, 3.0f, 0, 3.0f);
           // 在碰撞处绘制一个球
           DrawDebugSphere(GetWorld(), HitResult.ImpactPoint, 10.0f, 24, FColor::Red, false, 5.0f);
   
           // 显示碰撞到了哪个骨骼上, 可以通过这个信息对角色造成不同的伤害
           UE_LOG(LogSTURifleWeapon, Display, TEXT("Fire hit bone: %s"), *HitResult.BoneName.ToString());
       } else {
           // 绘制子弹的路径: 枪口位置 -> 子弹路径的终点
           DrawDebugLine(GetWorld(), GetMuzzleWorldLocation(), TraceEnd, FColor::Red, false, 3.0f, 0, 3.0f);
       }
   }
   
   // 获取子弹的逻辑路径
   bool ASTURifleWeapon::GetTraceData(FVector& TraceStart, FVector& TraceEnd) const {
       // 获取玩家的位置和朝向
       FVector ViewLocation;
       FRotator ViewRotation;
       if (!GetPlayerViewPoint(ViewLocation, ViewRotation)) return false;
   
       // 子弹的起点为: 角色当前位置
       TraceStart = ViewLocation;
       // 子弹的方向为: 角色当前正前方 + 一个随机的偏移
       // 当角色血量很低时, 偏移角可以变大, 模拟角色打不准的情况
       const auto HalfRad = FMath::DegreesToRadians(BulletSpread);
       const FVector ShootDirection = FMath::VRandCone(ViewRotation.Vector(), HalfRad);
       // 子弹的终点为: 角色当前位置沿子弹方向运动一定的距离
       TraceEnd = TraceStart + ShootDirection * TraceMaxDistance;
       return true;
   }

4. 创建蓝图类`BP_STURifleWeapon`，继承于`STURifleWeapon`

   1. 骨骼网格体设置为`Rifle`

5. 修改角色蓝图`BP_STUBaseCharacter|WeaponComponent`

   1. `WeaponClass`设置为`BP_STURifleWeapon`

6. 创建蓝图类`BP_STULauncherWeapon`，继承于`STULauncherWeapon`

   1. 骨骼网格体设置为`Launcher`

# 十二、榴弹发射器1：火箭生成

1. 在`Launcher`的骨骼树中，添加插槽`MuzzleSocket`，位于枪口处

   <img src="AssetMarkdown/image-20230208105027834.png" alt="image-20230208105027834" style="zoom:80%;" />

2. 创建C++类`STUProjectile`，继承于`Actor`

   1. 目录：`ShootThemUp/Source/ShootThemUp/Public/Weapon`

3. 修改`STUProjectile`：添加球形碰撞体

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "GameFramework/Actor.h"
   #include "STUProjectile.generated.h"
   
   class USphereComponent;
   
   UCLASS()
   class SHOOTTHEMUP_API ASTUProjectile : public AActor {
       GENERATED_BODY()
   
   public:
       ASTUProjectile();
   
   protected:
       // 榴弹的碰撞体
       UPROPERTY(VisibleDefaultsOnly, Category = "Weapon")
       USphereComponent* CollisionComponent;
   
       virtual void BeginPlay() override;
   };
   ```

   ```c++
   #include "Weapon/STUProjectile.h"
   #include "Components/SphereComponent.h"
   
   ASTUProjectile::ASTUProjectile() {
       PrimaryActorTick.bCanEverTick = false;
   
       // 创建球形碰撞体组件
       CollisionComponent = CreateDefaultSubobject<USphereComponent>("SphereComponent");
       CollisionComponent->InitSphereRadius(5.0f);
       SetRootComponent(CollisionComponent);
   }
   
   void ASTUProjectile::BeginPlay() {
       Super::BeginPlay();
   }

4. 修改`STULauncherWeapon`：重写`StartFire()`、`MakeShot()`

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "Weapon/STUBaseWeapon.h"
   #include "STULauncherWeapon.generated.h"
   
   class ASTUProjectile;
   
   UCLASS()
   class SHOOTTHEMUP_API ASTULauncherWeapon : public ASTUBaseWeapon
   {
   	GENERATED_BODY()
   public:
       virtual void StartFire() override;
   
   protected:
       // 要发射的榴弹的类
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "Weapon")
       TSubclassOf<ASTUProjectile> ProjectileClass;
   
       virtual void MakeShot() override;
   };
   ```

   ```c++
   #include "Weapon/STULauncherWeapon.h"
   #include "Weapon/STUProjectile.h"
   #include "Kismet/GameplayStatics.h"
   
   void ASTULauncherWeapon::StartFire() {
       MakeShot();
   }
   
   void ASTULauncherWeapon::MakeShot() {
       const FTransform SpawnTransform(FRotator::ZeroRotator, GetMuzzleWorldLocation());
       // 在场景中延迟创建一个榴弹
       auto Projectile = UGameplayStatics::BeginDeferredActorSpawnFromClass(GetWorld(), ProjectileClass, SpawnTransform);
       // 设置榴弹的参数
       // ...
       // 完成榴弹的创建
       UGameplayStatics::FinishSpawningActor(Projectile, SpawnTransform);
   }

5. 创建蓝图类`BP_STUProjectile`，继承于`STUProjectile`

   1. 添加一个`球体组件`，用于可视化碰撞体
      1. 缩放设置为：`(0.095,0.095,0.095)`

6. 修改`BP_STULauncherWeapon`：设置`Projectile Class`为`BP_STUProjectile`

7. 启动游戏，点击发射后，会在枪口处生成一个球

   1. 新建一个材质`M_BaseColor`，将`基础颜色`提升为参数
   2. 基于`M_BaseColor`创建材质实例`MT_RedColor`，将颜色改为`FF0000`
   3. 将该材质赋值给`BP_STUProjectile|球体组件`

