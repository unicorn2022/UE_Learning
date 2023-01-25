# 目录

[TOC]

# 一、Health组件

1.   创建C++类`STUHealthComponent`，继承于`Actor组件`

     1.   目录：`ShootThemUp/Source/ShootThemUp/Public/Components`

2.   `STUHealthComponent`组件负责角色的血量控制

     ```c++
     #pragma once
     
     #include "CoreMinimal.h"
     #include "Components/ActorComponent.h"
     #include "STUHealthComponent.generated.h"
     
     
     UCLASS( ClassGroup=(Custom), meta=(BlueprintSpawnableComponent) )
     class SHOOTTHEMUP_API USTUHealthComponent : public UActorComponent
     {
     	GENERATED_BODY()
     
     public:	
     	USTUHealthComponent();
         float GetHealth() const { return Health; }
     
     protected:
         // 角色最大血量
         UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "Health", meta = (ClampMin = "0.0", ClampMax = "1000.0"))
         float MaxHealth = 100.0f;
     
         virtual void BeginPlay() override;
     
     private:
         // 角色当前剩余血量
         float Health = 0.0f;
     };
     ```

     ```c++
     #include "Components/STUHealthComponent.h"
     
     USTUHealthComponent::USTUHealthComponent()
     {
     	PrimaryComponentTick.bCanEverTick = false;
     }
     
     
     void USTUHealthComponent::BeginPlay()
     {
         Super::BeginPlay();
         
         Health = MaxHealth;
     }
     ```

3.   修改`STUBaseCharacter`：添加`STUHealthComponent`组件

     ```c++
     class USTUHealthComponent;
     class UTextRenderComponent;
     
     UCLASS()
     class SHOOTTHEMUP_API ASTUBaseCharacter : public ACharacter {
          ...
     
     protected:
         // 角色的血量管理
         UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Components")
         USTUHealthComponent* HealthComponent;
     
         // 显示角色的血量
         UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Components")
         UTextRenderComponent* HealthTextComponent;
         
         ...
     }
     ```

     ```c++
     #include "Components/STUHealthComponent.h"
     #include "Components/TextRenderComponent.h"
     
     ASTUBaseCharacter::ASTUBaseCharacter(const FObjectInitializer& ObjInit) 
         : Super(ObjInit.SetDefaultSubobjectClass<USTUCharacterMovementComponent>(ACharacter::CharacterMovementComponentName)) 
     {
         // 允许该character每一帧调用Tick()
         PrimaryActorTick.bCanEverTick = true;
     
         // 创建弹簧臂组件, 并设置其父组件为根组件, 允许pawn控制旋转
         SpringArmComponent = CreateDefaultSubobject<USpringArmComponent>("SpringArmComponent");
         SpringArmComponent->SetupAttachment(GetRootComponent());
         SpringArmComponent->bUsePawnControlRotation = true;
     
         // 创建相机组件, 并设置其父组件为弹簧臂组件
         CameraComponent = CreateDefaultSubobject<UCameraComponent>("CameraComponent");
         CameraComponent->SetupAttachment(SpringArmComponent);
     
         // 创建血量组件, 由于其是纯逻辑的, 不需要设置父组件
         HealthComponent = CreateDefaultSubobject<USTUHealthComponent>("STUHealthComponent");
     
         // 创建血量显示组件, 并设置其父组件为根组件
         HealthTextComponent = CreateDefaultSubobject<UTextRenderComponent>("TextRenderComponent");
         HealthTextComponent->SetupAttachment(GetRootComponent());
     }
     
     void ASTUBaseCharacter::BeginPlay() {
         Super::BeginPlay();
     
         // 检查组件是否成功创建(仅开发阶段可用)
         check(HealthComponent);
         check(HealthTextComponent);
     }
     
     void ASTUBaseCharacter::Tick(float DeltaTime) {
         Super::Tick(DeltaTime);
     
         // 获取角色当前血量并显示
         const float Health = HealthComponent->GetHealth();
         const FString HealthString = FString::Printf(TEXT("%.0f"), Health);
         HealthTextComponent->SetText(FText::FromString(HealthString));
     }
     ```

4.   修改角色蓝图`BP_STUBaseCharacter`：更改`HealthTextComponent`组件的属性

     1.   旋转：`(0,0,180)`
     2.   水平对齐：`居中`
     3.   垂直对齐：`文本中心`
     4.   颜色：`#006558`