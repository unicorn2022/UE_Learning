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

# 二、VFX组件，子弹击中特效

1. 创建C++类`STUWeaponFXComponent`，继承于`Actor组件`

   1. 目录：`ShootThemUp/Source/ShootThemUp/Public/Weapon/Components`

2. 在`ShootThemUp.Build.cs`中更新路径

   ```c#
   PublicDependencyModuleNames.AddRange(new string[] { 
       "Core", 
       "CoreUObject", 
       "Engine", 
       "InputCore",
       "Niagara"
   });
   PublicIncludePaths.AddRange(new string[] { 
       "ShootThemUp/Public/Player", 
       "ShootThemUp/Public/Components", 
       "ShootThemUp/Public/Dev",
       "ShootThemUp/Public/Weapon",
       "ShootThemUp/Public/UI",
       "ShootThemUp/Public/Animations",
       "ShootThemUp/Public/Pickups",
       "ShootThemUp/Public/Weapon/Components"
   });
   ```

3. 修改`STUWeaponFXComponent`：

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "Components/ActorComponent.h"
   #include "STUWeaponFXComponent.generated.h"
   
   class UNiagaraSystem;
   
   UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
   class SHOOTTHEMUP_API USTUWeaponFXComponent : public UActorComponent {
       GENERATED_BODY()
   
   public:
       USTUWeaponFXComponent();
   
       void PlayImpactFX(const FHitResult& Hit);
   
   protected:
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "VFX")
       UNiagaraSystem* Effect;
   };
   ```

   ```c++
   #include "Weapon/Components/STUWeaponFXComponent.h"
   #include "NiagaraFunctionLibrary.h"
   
   USTUWeaponFXComponent::USTUWeaponFXComponent() {
       PrimaryComponentTick.bCanEverTick = false;
   }
   
   void USTUWeaponFXComponent::PlayImpactFX(const FHitResult& Hit) {
       UNiagaraFunctionLibrary::SpawnSystemAtLocation(GetWorld(), Effect, Hit.ImpactPoint, Hit.ImpactNormal.Rotation());
   }

4. 修改`STURifleWeapon`：击中时播放特效

   ```c++
   class USTUWeaponFXComponent;
   
   UCLASS()
   class SHOOTTHEMUP_API ASTURifleWeapon : public ASTUBaseWeapon {
   	...
   
   public:
       ASTURifleWeapon();
       
   protected:
       // 武器的特效
       UPROPERTY(VisibleAnywhere, Category = "VFX")
       USTUWeaponFXComponent* WeaponFXComponent;
          
   protected:
       virtual void BeginPlay() override;
   };
   
   ```

   ```c++
   #include "Weapon/Components/STUWeaponFXComponent.h"
   
   ASTURifleWeapon::ASTURifleWeapon() {
       // 创建特效组件
       WeaponFXComponent = CreateDefaultSubobject<USTUWeaponFXComponent>("WeaponFXComponent");
   }
   
   void ASTURifleWeapon::BeginPlay() {
       Super::BeginPlay();
   
       check(WeaponFXComponent);
   }
   
   // 发射子弹
   void ASTURifleWeapon::MakeShot() {
       // 判断当前弹夹是否为空
       if (!GetWorld() || IsClipEmpty()) {
           StopFire();
           return;
       }
   
       // 获取子弹的逻辑路径
       FVector TraceStart, TraceEnd;
       if (!GetTraceData(TraceStart, TraceEnd)) {
           StopFire();
           return;
       }
   
       // 计算子弹的碰撞结果
       FHitResult HitResult;
       MakeHit(HitResult, TraceStart, TraceEnd);
   
       if (HitResult.bBlockingHit) {
           // 对子弹击中的玩家进行伤害
           MakeDamage(HitResult);
           
           // 绘制子弹的路径: 枪口位置 -> 碰撞点
           DrawDebugLine(GetWorld(), GetMuzzleWorldLocation(), HitResult.ImpactPoint, FColor::Red, false, 3.0f, 0, 3.0f);
           DrawDebugSphere(GetWorld(), HitResult.ImpactPoint, 10.0f, 24, FColor::Red, false, 5.0f);
           
           // 播放击中特效
           WeaponFXComponent->PlayImpactFX(HitResult);
   
           // UE_LOG(LogSTURifleWeapon, Display, TEXT("Fire hit bone: %s"), *HitResult.BoneName.ToString());
       } else {
           // 绘制子弹的路径: 枪口位置 -> 子弹路径的终点
           DrawDebugLine(GetWorld(), GetMuzzleWorldLocation(), TraceEnd, FColor::Red, false, 3.0f, 0, 3.0f);
       }
   
       // 减少弹药数
       DecreaseAmmo();
   }
   ```

5. 修改`STUProjectile`：击中时播放特效

   ```c++
   class USTUWeaponFXComponent;
   
   UCLASS()
   class SHOOTTHEMUP_API ASTUProjectile : public AActor {
       ...
   
   protected:
       // 武器的特效
       UPROPERTY(VisibleAnywhere, Category = "VFX")
       USTUWeaponFXComponent* WeaponFXComponent;
   };
   ```

   ```c++
   #include "Weapon/Components/STUWeaponFXComponent.h"
   
   ASTUProjectile::ASTUProjectile() {
       ...
       // 创建特效组件
       WeaponFXComponent = CreateDefaultSubobject<USTUWeaponFXComponent>("WeaponFXComponent");
   }
   
   void ASTUProjectile::OnProjectileHit(
       UPrimitiveComponent* HitComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit) {
       if (!GetWorld()) return;
   
       // 停止榴弹的运动
       MovementComponent->StopMovementImmediately();
   
       // 造成球形伤害
       UGameplayStatics::ApplyRadialDamage(GetWorld(),  // 当前世界的指针
           DamageAmount,                                // 基础伤害
           GetActorLocation(),                          // 球形伤害的中心
           DamageRadius,                                // 球形伤害的半径
           UDamageType::StaticClass(),                  // 球形伤害的类型
           {GetOwner()},                                // 球形伤害忽略的actor
           this,                                        // 造成伤害的actor
           GetController(),                             // 造成伤害的actor的controller
           DoFullDamage);                               // 是否对整个爆炸范围造成相同伤害
   
       // 绘制榴弹的爆炸范围
       DrawDebugSphere(GetWorld(), GetActorLocation(), DamageRadius, 24, FColor::Red, false, 5.0f);
       
       // 播放击中特效
       WeaponFXComponent->PlayImpactFX(Hit);
   
       // 销毁Actor
       Destroy();
   }
   ```

6. 修改`BP_STURifleWeapon、BP_STUProjectile`：将`VFX/Effect`设置为`NS_BaseImpact`

7. 修改`NE_BaseImpact`：

   1. 将`Add Velocity in Cone`的`Cone Axis`修改为`(1,0,0)`
   2. 从而让粒子沿碰撞点的法线方向的X轴生成