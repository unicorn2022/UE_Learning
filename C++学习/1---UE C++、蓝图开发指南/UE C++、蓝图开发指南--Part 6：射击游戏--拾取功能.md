# 目录

[TOC]

# 一、BasePickup、AmmoPickup、HealthPickup三个类

1. 创建C++类`STUBasePickup`，继承于`Actor`
   1. 目录：`ShootThemUp/Source/ShootThemUp/Public/Pickups`

2. 在`ShootThemUp.Build.cs`中更新路径

   ```c#
   PublicIncludePaths.AddRange(new string[] { 
       "ShootThemUp/Public/Player", 
       "ShootThemUp/Public/Components", 
       "ShootThemUp/Public/Dev",
       "ShootThemUp/Public/Weapon",
       "ShootThemUp/Public/UI",
       "ShootThemUp/Public/Animations",
       "ShootThemUp/Public/Pickups"
   });
   ```

3. 创建C++类`STUAmmoPickup`，继承于`STUBasePickup`

   1. 目录：`ShootThemUp/Source/ShootThemUp/Public/Pickups`

4. 创建C++类`STUHealthPickup`，继承于`STUBasePickup`

   1. 目录：`ShootThemUp/Source/ShootThemUp/Public/Pickups`

5. 修改`STUBasePickup`：添加球形碰撞体，并设置碰撞为`overlap`，重叠后摧毁自身

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "GameFramework/Actor.h"
   #include "STUBasePickup.generated.h"
   
   class USphereComponent;
   
   UCLASS()
   class SHOOTTHEMUP_API ASTUBasePickup : public AActor {
       GENERATED_BODY()
   
   public:
       ASTUBasePickup();
   
   protected:
       // 掉落物的碰撞体
       UPROPERTY(VisibleAnywhere, Category = "Pickup")
       USphereComponent* CollisionComponent;
   
   protected:
       virtual void BeginPlay() override;
   
       // 两个Actor重叠在一起
       virtual void NotifyActorBeginOverlap(AActor* OtherActor) override;
   
   public:
       virtual void Tick(float DeltaTime) override;
   };
   ```

   ```c++
   #include "Pickups/STUBasePickup.h"
   #include "Components/SphereComponent.h"
   
   DEFINE_LOG_CATEGORY_STATIC(LogSTUBasePickup, All, All);
   
   ASTUBasePickup::ASTUBasePickup() {
       PrimaryActorTick.bCanEverTick = true;
   
       // 创建碰撞体组件
       CollisionComponent = CreateDefaultSubobject<USphereComponent>("SphereComponent");
       CollisionComponent->InitSphereRadius(50.0f);
       CollisionComponent->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
       CollisionComponent->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Overlap);
       SetRootComponent(CollisionComponent);
   }
   
   void ASTUBasePickup::BeginPlay() {
       Super::BeginPlay();
   }
   
   // 两个Actor重叠在一起
   void ASTUBasePickup::NotifyActorBeginOverlap(AActor* OtherActor) {
       Super::NotifyActorBeginOverlap(OtherActor);
   
       UE_LOG(LogSTUBasePickup, Warning, TEXT("Pickup was taken"));
       Destroy();
   }
   
   void ASTUBasePickup::Tick(float DeltaTime) {
       Super::Tick(DeltaTime);
   }

6. 将`Materials、Pichups`文件夹迁移到本项目中

7. 基于`STUAmmoPickup、STUHealthPickup`创建蓝图类`BP_STUAmmoPickup、BP_STUHealthPickup`

   1. 路径：`Pickups`
   2. 添加静态网格体组件，并设置对应的网格体、大小

8. 将创建的`BP_STUAmmoPickup、BP_STUHealthPickup`放入场景中

# 二、拾取物重新生成

1. 修改`STUBasePickup`：添加每隔一段时间后重新生成的功能

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUBasePickup : public AActor {
       ...
   protected:
       // 重新生成的间隔时间
       UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Pickup")
       float RespawnTime = 5.0f;
   
   private:
       // 将拾取物给到角色, 用于修改角色属性
       virtual bool GivePickupTo(APawn* PlayerPawn);
       
       // 当前Actor被拾起
       void PickupWasTaken();
       // 重新生成Actor
       void Respawn();
   };
   ```

   ```c++
   void ASTUBasePickup::NotifyActorBeginOverlap(AActor* OtherActor) {
       Super::NotifyActorBeginOverlap(OtherActor);
   
       // 将拾取物给到角色, 才能将拾取物拾起
       const auto Pawn = Cast<APawn>(OtherActor);
       if (GivePickupTo(Pawn)) {
           PickupWasTaken();
       }
   }
   
   bool ASTUBasePickup::GivePickupTo(APawn* PlayerPawn) {
       return false;
   }
   
   void ASTUBasePickup::PickupWasTaken() {
       // 取消Actor的碰撞响应
       CollisionComponent->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
       
       // 将Actor设为不可见
       if (GetRootComponent()) GetRootComponent()->SetVisibility(false, true);
   
       // 开启重新生成Actor的定时器
       FTimerHandle RespawnTimerHandle;
       GetWorldTimerManager().SetTimer(RespawnTimerHandle, this, &ASTUBasePickup::Respawn, RespawnTime);
   }
   
   void ASTUBasePickup::Respawn() {
       // 开启Actor的碰撞响应
       CollisionComponent->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Overlap);
   
       // 将Actor设为可见
       if (GetRootComponent()) GetRootComponent()->SetVisibility(true, true);
   }
   ```

2. 修改`STUAmmoPickup、STUHealthPickup`：重写`GivePickupTo()`

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "Pickups/STUBasePickup.h"
   #include "STUAmmoPickup.generated.h"
   
   UCLASS()
   class SHOOTTHEMUP_API ASTUAmmoPickup : public ASTUBasePickup {
       GENERATED_BODY()
   
   private:
       // 将拾取物给到角色, 用于修改角色属性
       virtual bool GivePickupTo(APawn* PlayerPawn) override;
   };
   ```

   ```c++
   #include "Pickups/STUAmmoPickup.h"
   
   DEFINE_LOG_CATEGORY_STATIC(LogSTUAmmoPickup, All, All);
   
   bool ASTUAmmoPickup::GivePickupTo(APawn* PlayerPawn) {
       UE_LOG(LogSTUAmmoPickup, Warning, TEXT("Ammo was taken"));
       return true;
   }
   ```

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "Pickups/STUBasePickup.h"
   #include "STUHealthPickup.generated.h"
   
   UCLASS()
   class SHOOTTHEMUP_API ASTUHealthPickup : public ASTUBasePickup {
       GENERATED_BODY()
   
   private:
       // 将拾取物给到角色, 用于修改角色属性
       virtual bool GivePickupTo(APawn* PlayerPawn) override;
   };
   ```

   ```c++
   #include "Pickups/STUHealthPickup.h"
   
   DEFINE_LOG_CATEGORY_STATIC(LogSTUHealthPickup, All, All);
   
   bool ASTUHealthPickup::GivePickupTo(APawn* PlayerPawn) {
       UE_LOG(LogSTUHealthPickup, Warning, TEXT("Health was taken"));
       return true;
   }
   ```

# 三、拾取物功能--弹药

1. 修改`STUAmmoPickup/GivePickupTo()`：

   ```c++
   class ASTUBaseWeapon;
   
   UCLASS()
   class SHOOTTHEMUP_API ASTUAmmoPickup : public ASTUBasePickup {
       GENERATED_BODY()
   
   protected:
       // 拾取物包含的弹夹数
       UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Pickup", meta = (ClampMin = "1.0", ClampMax = "10.0"))
       int32 ClipsAmount = 10;
   
       // 武器类型
       UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Pickup")
       TSubclassOf<ASTUBaseWeapon> WeaponType;
   
   private:
       // 将拾取物给到角色, 用于修改角色属性
       virtual bool GivePickupTo(APawn* PlayerPawn) override;
   };
   ```

   ```c++
   #include "Components/STUWeaponComponent.h"
   #include "STUUtils.h"
   
   DEFINE_LOG_CATEGORY_STATIC(LogSTUAmmoPickup, All, All);
   
   bool ASTUAmmoPickup::GivePickupTo(APawn* PlayerPawn) {    
       const auto WeaponComponent = STUUtils::GetSTUPlayerComponent<USTUWeaponComponent>(PlayerPawn);
       if (!WeaponComponent) return false;
       
       return WeaponComponent->TryToAddAmmo(WeaponType, ClipsAmount);
   }
   ```
   
2. 修改`STUWeaponComponent`：添加`TryToAddAmmo()`函数

   ```c++
   UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
   class SHOOTTHEMUP_API USTUWeaponComponent : public UActorComponent {
       ...
   
   public:
       // 尝试添加弹药
       bool TryToAddAmmo(TSubclassOf<ASTUBaseWeapon> WeaponType, int32 ClipsAmount);
   };
   ```

   ```c++
   bool USTUWeaponComponent::TryToAddAmmo(TSubclassOf<ASTUBaseWeapon> WeaponType, int32 ClipsAmount) {
       for (const auto Weapon : Weapons) {
           if (Weapon && Weapon->IsA(WeaponType)) {
               return Weapon->TryToAddAmmo(ClipsAmount);
           }
       }
       return false;
   }

3. 修改`STUBaseWeapon`：添加`TryToAddAmmo()`函数

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUBaseWeapon : public AActor {
       ...
   
   public:
       // 尝试添加弹药
       bool TryToAddAmmo(int32 ClipsAmount);
       
   protected:
       // 判断弹药是否已满
       bool IsAmmoFull() const;
   };
   
   ```

   ```c++
   bool ASTUBaseWeapon::IsAmmoFull() const {
       return CurrentAmmo.Clips == DefaultAmmo.Clips && CurrentAmmo.Bullets == DefaultAmmo.Bullets;
   }
   
   bool ASTUBaseWeapon::TryToAddAmmo(int32 ClipsAmount) {
       if (CurrentAmmo.Infinite || IsAmmoFull() || ClipsAmount <= 0) return false;
   
       const auto NextClipsAmount = CurrentAmmo.Clips + ClipsAmount;
       // 拾取的弹药可以补全弹药库, 则全部补全
       if(NextClipsAmount > DefaultAmmo.Clips){
           UE_LOG(LogSTUBaseWeapon, Display, TEXT("补全弹夹&弹药"));
           CurrentAmmo.Bullets = DefaultAmmo.Bullets;
           CurrentAmmo.Clips = DefaultAmmo.Clips;
       } 
       // 拾取的弹药不能补全弹药库, 则优先补充弹夹
       else {
           UE_LOG(LogSTUBaseWeapon, Display, TEXT("补全弹夹"));
           CurrentAmmo.Clips = NextClipsAmount;
       }
       return true;
   }
   
4. 修改`BP_STUAmmoPickup`：将`WeaponType`设置为`BP_STULauncherWeapon`

# 四、当前子弹数为空时禁止发射，按R换弹后补充弹药库

1. 修改`STUWeaponComponent/CanFire()`：防止弹夹为空后仍然开火

   ```c++
   bool USTUWeaponComponent::CanFire() const {
       // 有武器 && 可以开火 && 没有在更换武器 && 没有在更换弹夹
       return CurrentWeapon && CurrentWeapon->CanFire() && !EquipAnimInProgress && !ReloadAnimInProgress;
   }

2. 修改`STUWeaponComponent/OnReloadFinished()`：切换弹夹动画完成后，补充弹药

   ```c++
   void USTUWeaponComponent::OnReloadFinished(USkeletalMeshComponent* MeshComponent) {
       // 不是当前Character, 则不响应该事件
       ACharacter* Character = Cast<ACharacter>(GetOwner());
       if (!Character || Character->GetMesh() != MeshComponent) return;
   
       ReloadAnimInProgress = false;
       // 补充弹药
       if (CurrentWeapon) CurrentWeapon->ChangeClip();
   }
   ```

3. 修改`STURifleWeapon/MakeShot()`：弹夹为空后禁止开火

   ```c++
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
           // 在碰撞处绘制一个球
           DrawDebugSphere(GetWorld(), HitResult.ImpactPoint, 10.0f, 24, FColor::Red, false, 5.0f);
   
           // 显示碰撞到了哪个骨骼上, 可以通过这个信息对角色造成不同的伤害
           UE_LOG(LogSTURifleWeapon, Display, TEXT("Fire hit bone: %s"), *HitResult.BoneName.ToString());
       } else {
           // 绘制子弹的路径: 枪口位置 -> 子弹路径的终点
           DrawDebugLine(GetWorld(), GetMuzzleWorldLocation(), TraceEnd, FColor::Red, false, 3.0f, 0, 3.0f);
       }
   
       // 减少弹药数
       DecreaseAmmo();
   }
   ```

4. 修改`STULauncherWeapon/MakeShot()`：弹夹为空后禁止开火

   ```c++
   void ASTULauncherWeapon::MakeShot() {
       // 判断当前弹夹是否为空
       if (!GetWorld() || IsClipEmpty()) return;
   
       // 获取榴弹的逻辑路径
       FVector TraceStart, TraceEnd;
       if (!GetTraceData(TraceStart, TraceEnd)) return;
   
       // 计算榴弹的碰撞结果
       FHitResult HitResult;
       MakeHit(HitResult, TraceStart, TraceEnd);
   
       // 判断榴弹的落点
       const FVector EndPoint = HitResult.bBlockingHit ? HitResult.ImpactPoint : TraceEnd;
       // 计算榴弹的射击方向(单位向量)
       const FVector Direction = (EndPoint - GetMuzzleWorldLocation()).GetSafeNormal();
   
       // 榴弹的初始位置
       const FTransform SpawnTransform(FRotator::ZeroRotator, GetMuzzleWorldLocation());
       // 在场景中延迟创建一个榴弹
       ASTUProjectile* Projectile = GetWorld()->SpawnActorDeferred<ASTUProjectile>(ProjectileClass, SpawnTransform);
       if (Projectile) {
           // 设置榴弹的参数
           Projectile->SetShotDirection(Direction);
           Projectile->SetOwner(GetOwner());
           // 完成榴弹的创建
           Projectile->FinishSpawning(SpawnTransform);
       }
   
       // 减少弹药数
       DecreaseAmmo();
   }
   ```

5. 修改`STUBaseWeapon/DecreaseAmmo()、ChangeClip()`：减小弹药时不自动换弹夹

   ```c++
   void ASTUBaseWeapon::DecreaseAmmo() {
       CurrentAmmo.Bullets--;
   }
   
   void ASTUBaseWeapon::ChangeClip() {
       // 没有剩余弹药, 则直接返回
       if (IsAmmoEmpty()) return;
   
       // 更换弹夹, 并减少弹夹数
       CurrentAmmo.Bullets = DefaultAmmo.Bullets;
       if (!CurrentAmmo.Infinite) CurrentAmmo.Clips--;
       UE_LOG(LogSTUBaseWeapon, Display, TEXT("------ Change Clip ------"));
   }
   ```

# 五、实战作业：拾取物功能--回血

1. 修改`STUHealthPichup/GivePickupTo()`：

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUHealthPickup : public ASTUBasePickup {
       GENERATED_BODY()
   
   protected:
       // 拾取物恢复的血量
       UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Pickup", meta = (ClampMin = "1.0", ClampMax = "100.0"))
       int32 HealthAmount = 50.0f;
   
   private:
       // 将拾取物给到角色, 用于修改角色属性
       virtual bool GivePickupTo(APawn* PlayerPawn) override;
   };
   ```

   ```c++
   #include "Components/STUHealthComponent.h"
   #include "STUUtils.h"
   
   DEFINE_LOG_CATEGORY_STATIC(LogSTUHealthPickup, All, All);
   
   bool ASTUHealthPickup::GivePickupTo(APawn* PlayerPawn) {
       const auto HealthComponent = STUUtils::GetSTUPlayerComponent<USTUHealthComponent>(PlayerPawn);
       if (!HealthComponent || HealthComponent->IsDead()) return false;
   
       return HealthComponent->TryToAddHealth(HealthAmount);
   }

2. 修改`STUHealthComponent`：添加`TryToAddHealth()`函数

   ```c++
   UCLASS( ClassGroup=(Custom), meta=(BlueprintSpawnableComponent) )
   class SHOOTTHEMUP_API USTUHealthComponent : public UActorComponent{
   
   public:	
       // 尝试增加血量
       bool TryToAddHealth(float HealthAmount);
       // 判断角色当前血量是否已满
       bool IsHealthFull() const;
   };
   ```

   ```c++
   bool USTUHealthComponent::TryToAddHealth(float HealthAmount) {
       // 角色已死亡、血量已满，均不可回血
       if (IsDead() || IsHealthFull()) return false;
       
       SetHealth(Health + HealthAmount);
       return true;
   }
   
   bool USTUHealthComponent::IsHealthFull() const {
       return FMath::IsNearlyEqual(Health, MaxHealth);
   }

# 六、拾取物绕Z轴旋转

1. 修改`STUBasePickup`：

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUBasePickup : public AActor {
       ...
   
   private:
       // 绕Z轴旋转速度
       float RotationYaw = 0.0f;
       // 随机生成一个旋转速度
       void GenerateRotationYaw();
   };
   ```

   ```c++
   void ASTUBasePickup::BeginPlay() {
       Super::BeginPlay();
   
       check(CollisionComponent);
   
       GenerateRotationYaw();
   }
   
   void ASTUBasePickup::Tick(float DeltaTime) {
       Super::Tick(DeltaTime);
       AddActorLocalRotation(FRotator(0.0f, RotationYaw, 0.0f));
   }
   
   void ASTUBasePickup::Respawn() {
       GenerateRotationYaw();
       // 开启Actor的碰撞响应
       CollisionComponent->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Overlap);
   
       // 将Actor设为可见
       if (GetRootComponent()) GetRootComponent()->SetVisibility(true, true);
   }
   
   void ASTUBasePickup::GenerateRotationYaw() {
       const auto Direction = FMath::RandBool() ? 1.0f : -1.0f;
       RotationYaw = FMath::RandRange(1.0f, 2.0f) * Direction;
   }

