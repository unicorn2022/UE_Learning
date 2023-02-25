# 目录

[TOC]

# 一、GameMode介绍

1. 是游戏关卡的核心：定义了游戏规则、游戏设置、有多少玩家/机器人参与了游戏，一轮有多久，持续几轮，比赛的结束条件是什么
2. 还可以计算各种游戏统计信息
3. 每个GameMode与一个level相关联

# 二、GameMode：动态创建NPC

1. 修改`STUCoreTypes.h`：添加`FGameData`

   ```c++
   /* 游戏模式 */
   // 游戏基础数据
   USTRUCT(BlueprintType)
   struct FGameData {
       GENERATED_USTRUCT_BODY()
   
       // 玩家数量
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "Game", meta = (ClampMin = "1", ClampMax = "100"))
       int32 PlayersNum = 2;
   };
   ```

2. 修改`STUGameModeBase`：生成AI控制器及角色

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "GameFramework/GameModeBase.h"
   #include "STUCoreTypes.h"
   #include "STUGameModeBase.generated.h"
   
   class AAIController;
   
   UCLASS()
   class SHOOTTHEMUP_API ASTUGameModeBase : public AGameModeBase {
       GENERATED_BODY()
   public:
       ASTUGameModeBase();
   
       virtual void StartPlay() override;
   
       // 为生成的AIController配置Character
       virtual UClass* GetDefaultPawnClassForController_Implementation(AController* InController) override;
   
   protected:
       // AI控制器类
       UPROPERTY(EditDefaultsOnly, Category = "Game")
       TSubclassOf<AAIController> AIControllerClass;
   
       // AI角色类
       UPROPERTY(EditDefaultsOnly, Category = "Game")
       TSubclassOf<APawn> AIPawnClass;
   
       // 游戏的基础数据
       UPROPERTY(EditDefaultsOnly, Category = "Game")
       FGameData GameData;
   
   private:
       // 生成AI
       void SpawnBots();
   };
   ```

   ```c++
   #include "STUGameModeBase.h"
   #include "Player/STUBaseCharacter.h"
   #include "Player/STUPlayerController.h"
   #include "UI/STUGameHUD.h"
   #include "AIController.h"
   
   ASTUGameModeBase::ASTUGameModeBase() {
       DefaultPawnClass = ASTUBaseCharacter::StaticClass();
       PlayerControllerClass = ASTUPlayerController::StaticClass();
       HUDClass = ASTUGameHUD::StaticClass();
   }
   
   void ASTUGameModeBase::StartPlay() {
       Super::StartPlay();
   
       SpawnBots();
   }
   
   // 为生成的AIController配置Character
   UClass* ASTUGameModeBase::GetDefaultPawnClassForController_Implementation(AController* InController) {
       if (InController && InController->IsA<AAIController>())
           return AIPawnClass;
       else
           return Super::GetDefaultPawnClassForController_Implementation(InController);
   }
   
   
   // 生成AI
   void ASTUGameModeBase::SpawnBots() {
       if (!GetWorld()) return;
   
       for (int32 i = 0; i < GameData.PlayersNum - 1; i++) {
           FActorSpawnParameters SpawnInfo;
           SpawnInfo.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
   
           const auto STUAIController = GetWorld()->SpawnActor<AAIController>(AIControllerClass, SpawnInfo);
           RestartPlayer(STUAIController);
       }
   }

# 三、GameMode：回合计时器

1. 修改`STUCoreTypes.h/FGameData`：

   ```c++
   USTRUCT(BlueprintType)
   struct FGameData {
       GENERATED_USTRUCT_BODY()
   
       // 玩家数量
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "Game", meta = (ClampMin = "1", ClampMax = "100"))
       int32 PlayersNum = 2;
   
       // 回合数量
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "Game", meta = (ClampMin = "1", ClampMax = "10"))
       int32 RoundsNum = 4;
   
       // 一回合的时间(s)
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "Game", meta = (ClampMin = "3", ClampMax = "300"))
       int32 RoundTime = 10;
   };

2. 修改`STUGameModeBase`：

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUGameModeBase : public AGameModeBase {
       ...
   
   private:
       // 当前回合
       int32 CurrentRound = 1;
       // 当前回合剩余时间
       int32 RoundCountDown = 0;
       // 回合计时器
       FTimerHandle GameRoundTimerHandle;
   
       // 开始回合
       void StartRound();
       // 更新计时器
       void GameTimerUpdate();
   };
   ```

   ```c++
   // 开始回合
   void ASTUGameModeBase::StartRound() {
       RoundCountDown = GameData.RoundTime;
       // 每秒一次, 减少RoundCountDown的值
       GetWorldTimerManager().SetTimer(GameRoundTimerHandle, this, &ASTUGameModeBase::GameTimerUpdate, 1.0f, true);
   }
   
   // 更新计时器
   void ASTUGameModeBase::GameTimerUpdate() {
       UE_LOG(LogSTUGameModeBase, Display, TEXT("Time: %i / Round: %i/%i"), RoundCountDown, CurrentRound, GameData.RoundsNum);
   
       RoundCountDown--;
       // 也可以使用如下方案, 但是此时RoundCountDown就是float了
       // const auto TimerRate = GetWorldTimerManager().GetTimerRate(GameRoundTimerHandle);
       // RoundCountDown -= TimerRate;
   
       // 当前回合结束
       if (RoundCountDown == 0) {
           // 停止计时器
           GetWorldTimerManager().ClearTimer(GameRoundTimerHandle);
           // 回合数+1
           CurrentRound++;
   
           // 仍有剩余回合
           if (CurrentRound <= GameData.RoundsNum) {
               StartRound();
           }
           // 回合已经全部结束
           else {
               UE_LOG(LogSTUGameModeBase, Warning, TEXT("=========== Game over =========="));
           }
       }
   }
   ```

# 四、GameMode：回合开始时重新生成角色

1. 修改`STUGameModeBase`：

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUGameModeBase : public AGameModeBase {
       ...
   
   private:
       // 回合开始时，重新生成所有角色
       void ResetPlayers();
       // 重新生成单个角色
       void ResetOnePlayer(AController* Controller);
   };
   ```

   ```c++
   // 回合开始时，重新生成所有角色
   void ASTUGameModeBase::ResetPlayers() {
       if (!GetWorld()) return;
       for (auto It = GetWorld()->GetControllerIterator(); It; ++It) {
           ResetOnePlayer(It->Get());
       }
   }
   
   // 重新生成单个角色
   void ASTUGameModeBase::ResetOnePlayer(AController* Controller) {
       // 当Controller已经控制Character时, RestartPlayer时, SpawnRotation会直接使用当前控制的角色的Rotation
       // 因此需要将当前控制的角色Reset()一下, 实现重开的效果
       if (Controller && Controller->GetPawn()) {
           Controller->GetPawn()->Reset();
       }
   
       RestartPlayer(Controller);
   }

2. 修改`BP_STUAICharacter`：将`自动控制AI`设置为`已禁用`

3. 修改`STUAICharacter`：取消自动控制AI

   ```c++
   ASTUAICharacter::ASTUAICharacter(const FObjectInitializer& ObjInit)
       : Super(ObjInit.SetDefaultSubobjectClass<USTUAIWeaponComponent>("STUWeaponComponent")) {
       // 不自动生成Controller, 而是沿用之前回合的Controller
       AutoPossessAI = EAutoPossessAI::Disabled;
       AIControllerClass = ASTUAIController::StaticClass();
   
       // 设置character的旋转
       bUseControllerRotationYaw = false;
       if (GetCharacterMovement()) {
           GetCharacterMovement()->bUseControllerDesiredRotation = true;
           GetCharacterMovement()->RotationRate = FRotator(0.0f, 200.0f, 0.0f);
       }
   }

4. 修改`STUAIController`：为AI分配PlayerState

   ```c++
   ASTUAIController::ASTUAIController() {
       // 创建AI感知组件
       STUAIPerceptionComponent = CreateDefaultSubobject<USTUAIPerceptionComponent>("STUAIPerceptionComponent");
       SetPerceptionComponent(*STUAIPerceptionComponent);
   
       // 为AI分配PlayerState
       bWantsPlayerState = true;
   }
   ```

5. 