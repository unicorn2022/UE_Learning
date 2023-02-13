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

