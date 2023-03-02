目录

[TOC]

# 一、枚举游戏状态, 广播状态改变事件

1. 修改`STUCoreTypes`：

   ```c++
   /* 游戏菜单 */
   // 游戏状态: 等待开始 or 正在进行 or 暂停 or 结束
   UENUM(BlueprintType)
   enum class ESTUMatchState : uint8 {
       WaitingToStart = 0,
       InProgress,
       Pause,
       GameOver
   };
   // 委托：游戏状态改变
   DECLARE_MULTICAST_DELEGATE_OneParam(FOnMatchStateChangedSignature, ESTUMatchState);

2. 修改`STUGameModeBase`：创建游戏状态改变委托

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUGameModeBase : public AGameModeBase {
       ...
   public:
       // 委托：游戏状态改变
       FOnMatchStateChangedSignature OnMatchStateChanged;
       
   private:
       // 设置游戏状态, 同时广播事件
       void SetMatchState(ESTUMatchState State);
   };
   ```

   ```c++
   void ASTUGameModeBase::StartPlay() {
       Super::StartPlay();
   
       SpawnBots();
       CreateTeamsInfo();
   
       CurrentRound = 1;
       StartRound();
   
       SetMatchState(ESTUMatchState::InProgress);
   }
   
   void ASTUGameModeBase::GameOver() {
       UE_LOG(LogSTUGameModeBase, Warning, TEXT("=========== Game over =========="));
       LogPlayerInfo();
   
       for (auto Pawn : TActorRange<APawn>(GetWorld())) {
           if (Pawn) {
               Pawn->TurnOff();
               Pawn->DisableInput(nullptr);
           }
       }
   
       SetMatchState(ESTUMatchState::GameOver);
   }
   
   void ASTUGameModeBase::SetMatchState(ESTUMatchState State) {
       if (MatchState == State) return;
   
       MatchState = State;
       OnMatchStateChanged.Broadcast(MatchState);
   }
   ```

3. 修改`STUGameHUD`：根据`MatchState`修改UI

   ```c++
   #include "STUCoreTypes.h"
   
   UCLASS()
   class SHOOTTHEMUP_API ASTUGameHUD : public AHUD {
       ...
   
   private:    
       // 委托：游戏状态改变
       void OnMatchStateChanged(ESTUMatchState State);
   };
   ```

   ```c++
   void ASTUGameHUD::BeginPlay() {
       Super::BeginPlay();
       auto PlayerHUDWidget = CreateWidget<UUserWidget>(GetWorld(), PlayerHUDWidgetClass);
       if (PlayerHUDWidget) {
           PlayerHUDWidget->AddToViewport();
       }
   
       // 订阅OnMatchStateChanged委托
       if (GetWorld()) {
           const auto GameMode = Cast<ASTUGameModeBase>(GetWorld()->GetAuthGameMode());
           if (GameMode) {
               GameMode->OnMatchStateChanged.AddUObject(this, &ASTUGameHUD::OnMatchStateChanged);
           }
       }
   }
   
   void ASTUGameHUD::OnMatchStateChanged(ESTUMatchState State) {
       UE_LOG(LogSTUGameHUD, Warning, TEXT("Match state changed: %s"), *UEnum::GetValueAsString(State));
   }
   ```

# 二、暂停游戏及对应UI

1. 添加操作映射`PauseGame`，对应按键为`Esc/P`

2. 重写`STUPlayerController`：

   1. 只有在`STUPlayerController`中设置输入，才能角色死亡时仍能停止游戏
   2. 修改`BeginPlay`

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "GameFramework/PlayerController.h"
   #include "STUCoreTypes.h"
   #include "STUPlayerController.generated.h"
   
   class USTURespawnComponent;
   
   UCLASS()
   class SHOOTTHEMUP_API ASTUPlayerController : public APlayerController {
       GENERATED_BODY()
   
   public:
       ASTUPlayerController();
   
   protected:
       // 重生组件
       UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Components")
       USTURespawnComponent* RespawnComponent;
   
       virtual void BeginPlay() override;
       virtual void OnPossess(APawn* InPawn) override;
       virtual void SetupInputComponent() override;
   
   private:
       // 暂停游戏
       void OnPauseGame();
       // 委托：游戏状态改变
       void OnMatchStateChanged(ESTUMatchState State);
   };
   ```

   ```c++
   #include "Player/STUPlayerController.h"
   #include "Components/STURespawnComponent.h"
   #include "STUGameModeBase.h"
   
   ASTUPlayerController::ASTUPlayerController() {
       // 创建重生组件
       RespawnComponent = CreateDefaultSubobject<USTURespawnComponent>("STURespawnComponent");
   }
   
   void ASTUPlayerController::BeginPlay() {
       // 订阅 OnMatchStateChanged 事件
       if (GetWorld()) {
           const auto GameMode = Cast<ASTUGameModeBase>((GetWorld()->GetAuthGameMode()));
           if (GameMode) {
               GameMode->OnMatchStateChanged.AddUObject(this, &ASTUPlayerController::OnMatchStateChanged);
           }
       }
   }
   
   void ASTUPlayerController::OnPossess(APawn* InPawn) {
       Super::OnPossess(InPawn);
   
       OnNewPawn.Broadcast(InPawn);
   }
   
   void ASTUPlayerController::SetupInputComponent() {
       Super::SetupInputComponent();
       if (!InputComponent) return;
   
       InputComponent->BindAction("PauseGame", IE_Pressed, this, &ASTUPlayerController::OnPauseGame);
   }
   
   void ASTUPlayerController::OnPauseGame() {
       if (!GetWorld() || !GetWorld()->GetAuthGameMode()) return;
   
       GetWorld()->GetAuthGameMode()->SetPause(this);
   }
   
   void ASTUPlayerController::OnMatchStateChanged(ESTUMatchState State) {
       // 游戏进行中, 停止显示光标
       if (State == ESTUMatchState::InProgress) {
           SetInputMode(FInputModeGameOnly());
           bShowMouseCursor = false;
       } else {
           SetInputMode(FInputModeUIOnly());
           bShowMouseCursor = true;
       }
   }

3. 重写`STUGameModeBase`的`SetPause()/ClearPause()`函数：

   1. 调用`SetPause()`时，所有`Actor`的`Tick`均会停止
   2. 我们需要在调用`SetPause()/ClearPause()`时，修改游戏状态

   ```c++
   bool ASTUGameModeBase::SetPause(APlayerController* PC, FCanUnpause CanUnpauseDelegate) {
       // 先判断能否暂停, 然后再设置游戏状态
       const auto PauseSet = Super::SetPause(PC, CanUnpauseDelegate);
       if (PauseSet) {
           SetMatchState(ESTUMatchState::Pause);
       }
       return PauseSet;
   }
   
   bool ASTUGameModeBase::ClearPause() {
       const auto PauseCleared = Super::ClearPause();
       if (PauseCleared) {
           SetMatchState(ESTUMatchState::InProgress);
       }
       return PauseCleared;
   }

4. 添加蓝图类`WBP_GamePause`，继承于`UserWidget`

   1. 路径：`UI`

   <img src="AssetMarkdown/image-20230302214551455.png" alt="image-20230302214551455" style="zoom:80%;" />

5. 修改`STUGameHUD`：当游戏切换状态时，显示对应UI

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUGameHUD : public AHUD {
       GENERATED_BODY()
   public:
       virtual void DrawHUD() override;
   
   protected:
       // 游戏过程中的UI
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "UI")
       TSubclassOf<UUserWidget> PlayerHUDWidgetClass;
   
       // 游戏暂停时的UI
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "UI")
       TSubclassOf<UUserWidget> PauseWidgetClass;
   
       virtual void BeginPlay() override;
   
   private:
       // 将游戏状态与对应UI建立映射关系
       UPROPERTY()
       TMap<ESTUMatchState, UUserWidget*> GameWidgets;
   
       // 游戏当前UI
       UPROPERTY()
       UUserWidget* CurrentWidget = nullptr;
   
   private:
       // 绘制屏幕中心的十字准线
       void DrawCrossHair();
       
       // 委托：游戏状态改变
       void OnMatchStateChanged(ESTUMatchState State);
   };
   ```

   ```c++
   void ASTUGameHUD::BeginPlay() {
       Super::BeginPlay();
   
       // 将UserWidget与对应游戏状态建立映射
       GameWidgets.Add(ESTUMatchState::InProgress, CreateWidget<UUserWidget>(GetWorld(), PlayerHUDWidgetClass));
       GameWidgets.Add(ESTUMatchState::Pause, CreateWidget<UUserWidget>(GetWorld(), PauseWidgetClass));
   
       // 将UserWidget添加到场景中, 并设置为不可见
       for (auto GameWidgetPair : GameWidgets) {
           const auto GameWidget = GameWidgetPair.Value;
           if (!GameWidget) continue;
           GameWidget->AddToViewport();
           GameWidget->SetVisibility(ESlateVisibility::Hidden);
       }
   
       // 订阅OnMatchStateChanged委托
       if (GetWorld()) {
           const auto GameMode = Cast<ASTUGameModeBase>(GetWorld()->GetAuthGameMode());
           if (GameMode) {
               GameMode->OnMatchStateChanged.AddUObject(this, &ASTUGameHUD::OnMatchStateChanged);
           }
       }
   }
   
   void ASTUGameHUD::OnMatchStateChanged(ESTUMatchState State) {
       // 将当前UI设为不可见 
       if (CurrentWidget) CurrentWidget->SetVisibility(ESlateVisibility::Hidden);
       // 然后更改UI界面
       if (GameWidgets.Contains(State)) CurrentWidget = GameWidgets[State];
       if (CurrentWidget) CurrentWidget->SetVisibility(ESlateVisibility::Visible);
   
       UE_LOG(LogSTUGameHUD, Warning, TEXT("Match state changed: %s"), *UEnum::GetValueAsString(State));
   }

6. 创建C++类`STUPauseWidget`，继承于`UserWidget`：创建按钮的点击回调函数

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "Blueprint/UserWidget.h"
   #include "STUPauseWidget.generated.h"
   
   class UButton;
   
   UCLASS()
   class SHOOTTHEMUP_API USTUPauseWidget : public UUserWidget {
       GENERATED_BODY()
   
   public:
       virtual bool Initialize() override;
   
   protected:
       UPROPERTY(meta = (BindWidget))
       UButton* ClearPauseButton;
   
   private:
       // 委托：点击按钮
       UFUNCTION()
       void OnClearPause();
   };
   ```

   ```c++
   #include "UI/STUPauseWidget.h"
   #include "Gameframework/GameModeBase.h"
   #include "Components/Button.h"
   
   bool USTUPauseWidget::Initialize() {
       const auto InitStatue = Super::Initialize();
       if (ClearPauseButton) {
           ClearPauseButton->OnClicked.AddDynamic(this, &USTUPauseWidget::OnClearPause);
       }
       return InitStatue;
   }
   
   void USTUPauseWidget::OnClearPause() {
       if (!GetWorld() || !GetWorld()->GetAuthGameMode()) return;
   
       GetWorld()->GetAuthGameMode()->ClearPause();
   }
   ```

7. 修改`WBP_GamePause`的父类为`STUPauseWidget`

   1. 将按钮重命名为`ClearPauseButton`
   2. 添加`背景模糊`处理效果，强度设置为`4`

   <img src="AssetMarkdown/image-20230302221219509.png" alt="image-20230302221219509" style="zoom:80%;" />

