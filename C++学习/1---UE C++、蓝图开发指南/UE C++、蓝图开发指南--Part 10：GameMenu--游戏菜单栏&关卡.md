# 目录

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

# 三、游戏结束及对应UI

1. 修改`STUGameHUD`：添加游戏结束时的widget引用

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUGameHUD : public AHUD {
       ...
   
   protected:
       // 游戏结束时的UI
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "UI")
       TSubclassOf<UUserWidget> GameOverWidgetClass;
   };
   ```

   ```c++
   void ASTUGameHUD::BeginPlay() {
       Super::BeginPlay();
   
       // 将UserWidget与对应游戏状态建立映射
       GameWidgets.Add(ESTUMatchState::InProgress, CreateWidget<UUserWidget>(GetWorld(), PlayerHUDWidgetClass));
       GameWidgets.Add(ESTUMatchState::Pause, CreateWidget<UUserWidget>(GetWorld(), PauseWidgetClass));
       GameWidgets.Add(ESTUMatchState::GameOver, CreateWidget<UUserWidget>(GetWorld(), GameOverWidgetClass));
   
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

2. 复制`WBP_GamePause`，创建控件蓝图`WBP_GameOver`

   1. 将父类临时设置为`UserWidget`
   2. 删除按钮及对应的尺寸框
   3. 在`BP_STUGameHUD`中设置`GameOverWidgetClass`对应的类为`WBP_GameOver`

3. 创建控件蓝图`WBP_PlayerStateRow`：每个角色信息的UI行

   <img src="AssetMarkdown/image-20230304130737322.png" alt="image-20230304130737322" style="zoom:80%;" />

   <img src="AssetMarkdown/image-20230304130346991.png" alt="image-20230304130346991" style="zoom:80%;" />

4. 修改`WBP_GameOver`

   <img src="AssetMarkdown/image-20230304130754554.png" alt="image-20230304130754554" style="zoom:80%;" />

   <img src="AssetMarkdown/image-20230304130654818.png" alt="image-20230304130654818" style="zoom:80%;" />

5. 创建C++类`STUGameOverWidget`，继承于`UserWidget`

   1. 目录：`ShootThemUp/Source/ShootThemUp/Public/UI`

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "Blueprint/UserWidget.h"
   #include "STUCoreTypes.h"
   #include "STUGameOverWidget.generated.h"
   
   class UVerticalBox;
   
   UCLASS()
   class SHOOTTHEMUP_API USTUGameOverWidget : public UUserWidget {
       GENERATED_BODY()
   
   public:
       virtual bool Initialize() override;
   
   protected:
       // 显示角色信息的垂直框
       UPROPERTY(meta = (BindWidget))
       UVerticalBox* PlayerStateBox;
   
       // 角色信息单行显示UI
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "UI")
       TSubclassOf<UUserWidget> PlayerStateRowWidgetClass;
   
   private:
       void OnMatchStateChanged(ESTUMatchState State);
       // 更新角色信息
       void UpdatePlayerState();
   };
   ```

   ```c++
   #include "UI/STUGameOverWidget.h"
   #include "STUGameModeBase.h"
   #include "Player/STUPlayerState.h"
   #include "UI/STUPlayerStateRowWidget.h"
   #include "Components/VerticalBox.h"
   #include "STUUtils.h"
   
   bool USTUGameOverWidget::Initialize() {
       if (GetWorld()) {
           const auto GameMode = Cast<ASTUGameModeBase>(GetWorld()->GetAuthGameMode());
           if (GameMode) {
               GameMode->OnMatchStateChanged.AddUObject(this, &USTUGameOverWidget::OnMatchStateChanged);
           }
       }
       return Super::Initialize();
   }
   
   void USTUGameOverWidget::OnMatchStateChanged(ESTUMatchState State) {
       if (State == ESTUMatchState::GameOver) {
           UpdatePlayerState();
       }
   }
   
   void USTUGameOverWidget::UpdatePlayerState() {
       if (!GetWorld() || !PlayerStateBox) return;
   
       // 清空子节点, 保证垂直框中的均为本函数创建的
       PlayerStateBox->ClearChildren();
   
       for (auto It = GetWorld()->GetControllerIterator(); It; ++It) {
           // 获取角色状态
           const auto Controller = It->Get();
           if (!Controller) continue;
           const auto PlayerState = Cast<ASTUPlayerState>(Controller->PlayerState);
           if (!PlayerState) continue;
           
           // 创建UI控件
           const auto PlayerStateRowWidget = CreateWidget<USTUPlayerStateRowWidget>(GetWorld(), PlayerStateRowWidgetClass);
           if (!PlayerStateRowWidget) continue;
   
           // 修改UI控件的显示信息
           PlayerStateRowWidget->SetPlayerName(FText::FromString(PlayerState->GetPlayerName()));
           PlayerStateRowWidget->SetKills(STUUtils::TextFromInt(PlayerState->GetKillsNum()));
           PlayerStateRowWidget->SetDeaths(STUUtils::TextFromInt(PlayerState->GetDeathsNum()));
           PlayerStateRowWidget->SetTeam(STUUtils::TextFromInt(PlayerState->GetTeamID()));
           PlayerStateRowWidget->SetPlayerIndicatorVisibility(Controller->IsPlayerController());
   
           // 将UI控件添加到垂直框中
           PlayerStateBox->AddChild(PlayerStateRowWidget);
       }
   }

6. 创建C++类`STUPlayerStateRowWidget`，继承于`UserWidget`

   1. 目录：`ShootThemUp/Source/ShootThemUp/Public/UI`

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "Blueprint/UserWidget.h"
   #include "STUPlayerStateRowWidget.generated.h"
   
   class UImage;
   class UTextBlock;
   
   UCLASS()
   class SHOOTTHEMUP_API USTUPlayerStateRowWidget : public UUserWidget {
       GENERATED_BODY()
   
   public:
       void SetPlayerName(const FText& Text);
       void SetKills(const FText& Text);
       void SetDeaths(const FText& Text);
       void SetTeam(const FText& Text);
       void SetPlayerIndicatorVisibility(bool Visible);
   
   protected:
       // 文本框: 玩家姓名
       UPROPERTY(meta = (BindWidget))
       UTextBlock* PlayerNameTextBlock;
       // 文本框: 击杀数
       UPROPERTY(meta = (BindWidget))
       UTextBlock* KillsTextBlock;
       // 文本框: 死亡数
       UPROPERTY(meta = (BindWidget))
       UTextBlock* DeathsTextBlock;
       // 文本框: 所属队伍
       UPROPERTY(meta = (BindWidget))
       UTextBlock* TeamTextBlock;
       // 图像框: 高亮显示
       UPROPERTY(meta = (BindWidget))
       UImage* PlayerIndicatorImage;
   };
   ```

   ```c++
   #include "UI/STUPlayerStateRowWidget.h"
   #include "Components/TextBlock.h"
   #include "Components/Image.h"
   
   void USTUPlayerStateRowWidget::SetPlayerName(const FText& Text) {
       if (!PlayerNameTextBlock) return;
       PlayerNameTextBlock->SetText(Text);
   }
   
   void USTUPlayerStateRowWidget::SetKills(const FText& Text) {
       if (!KillsTextBlock) return;
       KillsTextBlock->SetText(Text);
   }
   
   void USTUPlayerStateRowWidget::SetDeaths(const FText& Text) {
       if (!DeathsTextBlock) return;
       DeathsTextBlock->SetText(Text);
   }
   
   void USTUPlayerStateRowWidget::SetTeam(const FText& Text) {
       if (!TeamTextBlock) return;
       TeamTextBlock->SetText(Text);
   }
   
   void USTUPlayerStateRowWidget::SetPlayerIndicatorVisibility(bool Visible) {
       if (!PlayerIndicatorImage) return;
       PlayerIndicatorImage->SetVisibility(Visible ? ESlateVisibility::Visible : ESlateVisibility::Hidden);
   }

7. 修改`STUGameModeBase/CreateTeamsInfo()`：创建角色的姓名

   ```c++
   void ASTUGameModeBase::CreateTeamsInfo() {
       if (!GetWorld()) return;
   
       int32 TeamID = 1;
       for (auto It = GetWorld()->GetControllerIterator(); It; ++It) {
           const auto Controller = It->Get();
           if (!Controller) continue;
           
           const auto PlayerState = Cast<ASTUPlayerState>(Controller->PlayerState);
           if (!PlayerState) continue;
   
           PlayerState->SetTeamID(TeamID);
           PlayerState->SetTeamColor(DetermineColorByTeamID(TeamID));
           PlayerState->SetPlayerName(Controller->IsPlayerController() ? "Player" : "Bot");
           SetPlayerColor(Controller);
   
           TeamID = TeamID == 1 ? 2 : 1;
       }
   }
   ```

8. 修改`WBP_GameOver`

   1. 将蓝图父类设置为`STUGameOverWidget`
   2. 重命名UI控件
   3. 将`PlayerStateRowWidgetClass`设置为`WBP_PlayerStateRow`

   <img src="AssetMarkdown/image-20230304153451839.png" alt="image-20230304153451839" style="zoom:80%;" />

9. 修改`WBP_PlayerStateRow`

   1. 将蓝图父类设置为`STUPlayerStateRowWidget`
   2. 重命名UI控件

   <img src="AssetMarkdown/image-20230304154609739.png" alt="image-20230304154609739" style="zoom:80%;" />

10. 修改`WBP_GameOver`：

    1. 将`PlayerStateBox`包裹为`滚动框`，并修改`滚动框/细节/样式、滚动`
    2. 将`滚动框`包裹为`尺寸框`，并设置最大高度为`300`

    <img src="AssetMarkdown/image-20230304154425455.png" alt="image-20230304154425455" style="zoom:80%;" />

# 四、游戏结束后，重新开始游戏

1. 修改`STUGameOverWidget`

   ```c++
   class UButton;
   
   UCLASS()
   class SHOOTTHEMUP_API USTUGameOverWidget : public UUserWidget {
       ...
   
   protected:
       // 按钮：重新开始游戏
       UPROPERTY(meta = (BindWidget))
       UButton* ResetLevelButton;
   
       // Initialize()时调用的函数
       virtual void NativeOnInitialized() override;
   
   private:
       // 委托：重新开始游戏
       UFUNCTION()
       void OnResetLevel();
   };
   ```

   ```c++
   #include "Components/Button.h"
   #include "Kismet/GameplayStatics.h"
   
   void USTUGameOverWidget::NativeOnInitialized() {
       Super::NativeOnInitialized();
   
       // 订阅 OnMatchStateChanged 委托
       if (GetWorld()) {
           const auto GameMode = Cast<ASTUGameModeBase>(GetWorld()->GetAuthGameMode());
           if (GameMode) {
               GameMode->OnMatchStateChanged.AddUObject(this, &USTUGameOverWidget::OnMatchStateChanged);
           }
       }
   
       // 订阅 ResetLevelButton 的点击事件
       if (ResetLevelButton) {
           ResetLevelButton->OnClicked.AddDynamic(this, &USTUGameOverWidget::OnResetLevel);
       }
   }
   
   void USTUGameOverWidget::OnResetLevel() {
       // 硬重置: 直接重新打开关卡
       const FString CurrentLevelName = UGameplayStatics::GetCurrentLevelName(this);
       UGameplayStatics::OpenLevel(this, FName(CurrentLevelName));
   }

2. 将所有`Widget`类的`Initialize()`修改为`NativeOnInitialized`

3. 修改`WBP_GameOver`：添加`ResetLevelButton`

   <img src="AssetMarkdown/image-20230304161523671.png" alt="image-20230304161523671" style="zoom:80%;" />

4. 
