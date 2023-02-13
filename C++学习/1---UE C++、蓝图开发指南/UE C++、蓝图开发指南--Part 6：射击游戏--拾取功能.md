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