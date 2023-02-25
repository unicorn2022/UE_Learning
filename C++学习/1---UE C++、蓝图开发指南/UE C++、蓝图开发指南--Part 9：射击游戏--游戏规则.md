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

