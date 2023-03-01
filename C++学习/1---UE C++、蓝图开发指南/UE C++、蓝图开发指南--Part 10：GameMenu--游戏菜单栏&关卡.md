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

# 二、