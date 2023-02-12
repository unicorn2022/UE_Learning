# 目录

[TOC]

# 一、角色健康栏--蓝图版

1. 修改`STUHealthComponent`：添加函数`GetHealthPercent`

   ```c++
   UCLASS( ClassGroup=(Custom), meta=(BlueprintSpawnableComponent) )
   class SHOOTTHEMUP_API USTUHealthComponent : public UActorComponent{
       ...
   public:	
       // 获取角色当前生命值
       float GetHealth() const { return Health; }
   
       // 获取角色当前生命值百分比
       UFUNCTION(BlueprintCallable, Category = "Health")
       float GetHealthPercent() const { return Health / MaxHealth; }
   
   	...
   };
   ```

2. 创建文件夹`UI`，然后在该目录下创建`用户界面/控件蓝图`，重命名为`WBP_PlayerHUD`

3. 基于`STUGameHUD`，创建蓝图类`BP_STUGameHUD`

   1. 路径：`UI`

4. 修改`BP_STUGameHUD`：

   <img src="AssetMarkdown/image-20230212124240282.png" alt="image-20230212124240282" style="zoom:80%;" />

5. 将HUD类设置为`BP_STUGameHUD`，在游戏中可以通过`Shift+F1`解放鼠标

6. 修改`WBP_PlayerHUD`

   1. 添加`进度条`控件，将填充颜色设为`绿色`
   2. 对`进度/百分比`属性创建绑定，函数重命名为`Get_HealthPercent`

   <img src="AssetMarkdown/image-20230212125221710.png" alt="image-20230212125221710" style="zoom:80%;" />

7. 修改`BP_BaseCharacter/HealthComponent`中，有关自动回复的参数

   <img src="AssetMarkdown/image-20230212125403134.png" alt="image-20230212125403134" style="zoom:80%;" />

# 二、角色健康栏--C++版

1. 创建C++类`STUPlayerHUDWidget`，继承于`UserWidget`

   1. 目录：`ShootThemUp/Source/ShootThemUp/Public/UI`

2. 修改`STUPlayerHUDWidget`

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "Blueprint/UserWidget.h"
   #include "STUPlayerHUDWidget.generated.h"
   
   UCLASS()
   class SHOOTTHEMUP_API USTUPlayerHUDWidget : public UUserWidget {
       GENERATED_BODY()
   public:
       UFUNCTION(BlueprintCallable, Category = "UI")
       float GetHealthPercent() const;
   };
   ```

   ```c++
   #include "UI/STUPlayerHUDWidget.h"
   #include "Components/STUHealthComponent.h"
   
   float USTUPlayerHUDWidget::GetHealthPercent() const {
       const auto Player = GetOwningPlayerPawn();
       if (!Player) return 0.0f;
   
       const auto Component = Player->GetComponentByClass(USTUHealthComponent::StaticClass());
       const auto HealthComponent = Cast<USTUHealthComponent>(Component);
       if (!HealthComponent) return 0.0f;
   
       return HealthComponent->GetHealthPercent();
   }

3. 修改`STUGameHUD`

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "GameFramework/HUD.h"
   #include "STUGameHUD.generated.h"
   
   UCLASS()
   class SHOOTTHEMUP_API ASTUGameHUD : public AHUD {
       GENERATED_BODY()
   
   protected:
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "UI")
       TSubclassOf<UUserWidget> PlayerHUDWidgetClass;
   
       virtual void BeginPlay() override;
   };
   ```

   ```c++
   #include "Blueprint/UserWidget.h"
   
   void ASTUGameHUD::BeginPlay() {
       Super::BeginPlay();
       auto PlayerHUDWidget = CreateWidget<UUserWidget>(GetWorld(), PlayerHUDWidgetClass);
       if (PlayerHUDWidget) {
           PlayerHUDWidget->AddToViewport();
       }
   }

4. 修改`WBP_PlayerHUD`：

   1. 修改父类：在`文件/重设蓝图父项`中，修改父类为`STUPlayerHUDWidget`
   2. 修改`图表/Get_HealthPercent`函数：将计算过程隐藏与C++类中

   <img src="AssetMarkdown/image-20230212131517786.png" alt="image-20230212131517786" style="zoom:80%;" />

5. 修改`BP_STUGameHUD`：

   1. 将事件图表中的蓝图删除，因为我们已经在C++中实现了
   2. 将`类默认值/PlayerHUDWidgetClass`设置为`WBP_PlayerHUD`
      1. 可以通过修改这个属性，选择创建哪一种UI界面

# 三、每种武器的瞄准UI

1. 将`UI/Images`迁移到本项目中

2. 修改`WBP_PlayerHUD`

   1. 添加`图像`控件，选择`RifleCrossHair`，然后勾选`大小到内容`
   2. 将锚点修改为`中央`，`位置X(Y)`均设置为0，`对齐`设置为`(0.5,0.5)`

   <img src="AssetMarkdown/image-20230212215759334.png" alt="image-20230212215759334" style="zoom:80%;" />

3. 修改`STUCoreType`：创建新类型`FWeaponUIData`

   ```c++
   USTRUCT(BlueprintType)
   struct FWeaponUIData {
       GENERATED_USTRUCT_BODY()
   
       // 武器的图标
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "UI")
       UTexture2D* MainIcon;
       
       // 武器的瞄准线图标
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "UI")
       UTexture2D* CrossHairIcon;
   };
   ```

4. 修改`STUBaseWeapon`：添加武器的显示UI数据

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUBaseWeapon : public AActor {
       ...
   
   public:
       FWeaponUIData GetUIData() const { return UIData; }
   
   protected:    // 武器的显示UI
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "UI")
       FWeaponUIData UIData;
   };

5. 修改`STUWeaponComponent`：添加`UIData`的`getter`函数

   ```c++
   UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
   class SHOOTTHEMUP_API USTUWeaponComponent : public UActorComponent {
       GENERATED_BODY()
   
   public:
       // 获取武器UI数据
       bool GetWeaponUIData(FWeaponUIData& UIData) const;
   };
   ```

   ```c++
   bool USTUWeaponComponent::GetWeaponUIData(FWeaponUIData& UIData) const {
       if (!CurrentWeapon) return false;
       UIData = CurrentWeapon->GetUIData();
       return true;
   }
   ```

6. 修改`STUPlayerHUDWidget`：添加`UIData`的`getter`函数

   ```c++
   ...
   #include "STUCoreTypes.h"
   
   UCLASS()
   class SHOOTTHEMUP_API USTUPlayerHUDWidget : public UUserWidget {
       GENERATED_BODY()
   public:
       UFUNCTION(BlueprintCallable, Category = "UI")
       float GetHealthPercent() const;
   
       UFUNCTION(BlueprintCallable, Category = "UI")
       bool GetWeaponUIData(FWeaponUIData& UIData) const;
   };
   ```

   ```c++
   bool USTUPlayerHUDWidget::GetWeaponUIData(FWeaponUIData& UIData) const {
       const auto Player = GetOwningPlayerPawn();
       if (!Player) return false;
   
       const auto Component = Player->GetComponentByClass(USTUWeaponComponent::StaticClass());
       const auto WeaponComponent = Cast<USTUWeaponComponent>(Component);
       if (!WeaponComponent) return false;
   
       return WeaponComponent->GetWeaponUIData(UIData);
   }
   ```

7. 修改`STUGameHUD/DrawHUD`

   ```c++
   void ASTUGameHUD::DrawHUD() {
       Super::DrawHUD();
       // DrawCrossHair();
   }

8. 修改`WBP_PlayerHUD`

   1. 对`图像框/外观/笔刷`属性进行绑定，函数重命名为`Get_CrossHairImage`

   <img src="AssetMarkdown/image-20230212220456411.png" alt="image-20230212220456411" style="zoom:80%;" />

9. 为每种武器设置`UIData`

   1. `BP_STURifleWeapon`：`RifleMainIcon`、`RifleCrossHair`
   2. `BP_STULauncherWeapon`：`LauncherMainIcon`、`LauncherCrossHair`

