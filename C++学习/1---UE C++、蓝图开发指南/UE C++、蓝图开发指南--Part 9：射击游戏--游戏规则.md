# 目录

[TOC]

# 一、GameMode介绍

1. 是游戏关卡的核心：定义了游戏规则、游戏设置、有多少玩家/机器人参与了游戏，一轮有多久，持续几轮，比赛的结束条件是什么
2. 还可以计算各种游戏统计信息
3. 每个GameMode与一个level相关联

# 二、动态创建NPC

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

# 三、回合计时器

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

# 四、回合开始时重新生成角色

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

# 五、PlayerState：所属队伍

> 此类的存在方式与Controller类似，当玩家死亡时，不会被删除，因此可以用于存储一些与Character无关的数据，如：死亡数、杀敌数、玩家姓名、队伍数量等

1. 创建C++类`STUPlayerState`，继承于`玩家状态`

   1. 目录：`ShootThemUp/Source/ShootThemUp/Public/Player`

2. 修改`STUPlayerState`：

   ```c++
   // Shoot Them Up Game, All Rights Reserved
   
   #pragma once
   
   #include "CoreMinimal.h"
   #include "GameFramework/PlayerState.h"
   #include "STUPlayerState.generated.h"
   
   UCLASS()
   class SHOOTTHEMUP_API ASTUPlayerState : public APlayerState {
       GENERATED_BODY()
   
   public:
       void SetTeamID(int32 ID) { TeamID = ID; }
       int32 GetTeamID() const { return TeamID; }
   
       void SetTeamColor(const FLinearColor& Color) { TeamColor = Color; }
       FLinearColor GetTeamColor() const { return TeamColor; }
   
   private:
       int32 TeamID;
       FLinearColor TeamColor;
   };
   ```

3. 修改`STUCoreTypes/FGameData`：添加队伍颜色相关信息

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
   
       // 默认队伍颜色
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite)
       FLinearColor DefaultTeamColor = FLinearColor::Red;
   
       // 队伍可选颜色
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite)
       TArray<FLinearColor> TeamColors;
   };
   ```

4. 修改`STUGameModeBase`：游戏开始时，创建队伍信息

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUGameModeBase : public AGameModeBase {
       ...
   
   private:
       // 生成AI
       void SpawnBots();
       // 创建队伍信息
       void CreateTeamsInfo();
       // 根据TeamID, 决定TeamColor
       FLinearColor DetermineColorByTeamID(int32 TeamID);
       // 设置玩家颜色
       void SetPlayerColor(AController* Controller);
   
   };
   ```

   ```c++
   ASTUGameModeBase::ASTUGameModeBase() {
       DefaultPawnClass = ASTUBaseCharacter::StaticClass();
       PlayerControllerClass = ASTUPlayerController::StaticClass();
       HUDClass = ASTUGameHUD::StaticClass();
       PlayerStateClass = ASTUPlayerState::StaticClass();
   }
   
   void ASTUGameModeBase::StartPlay() {
       Super::StartPlay();
   
       SpawnBots();
       CreateTeamsInfo();
   
       // 初始化第一回合
       CurrentRound = 1;
       StartRound();
   }
   
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
           SetPlayerColor(Controller);
   
           TeamID = TeamID == 1 ? 2 : 1;
       }
   }
   
   FLinearColor ASTUGameModeBase::DetermineColorByTeamID(int32 TeamID) {
       if (TeamID <= GameData.TeamColors.Num()) {
           return GameData.TeamColors[TeamID - 1];
       } 
       else {
           UE_LOG(LogSTUGameModeBase, Warning, TEXT("No color for team id: %i, set to default: %s"), TeamID,
               *GameData.DefaultTeamColor.ToString());
           return GameData.DefaultTeamColor;
       }
   }
   
   void ASTUGameModeBase::SetPlayerColor(AController* Controller) {
       if (!Controller) return;
   
       const auto Character = Cast<ASTUBaseCharacter>(Controller->GetPawn());
       if (!Character) return;
   
       const auto PlayerState = Cast<ASTUPlayerState>(Controller->PlayerState);
       if (!PlayerState) return;
   
       Character->SetPlayerColor(PlayerState->GetTeamColor());
   }
   
   void ASTUGameModeBase::ResetOnePlayer(AController* Controller) {
       // 当Controller已经控制Character时, RestartPlayer时, SpawnRotation会直接使用当前控制的角色的Rotation
       // 因此需要将当前控制的角色Reset()一下, 实现重开的效果
       if (Controller && Controller->GetPawn()) {
           Controller->GetPawn()->Reset();
       }
   
       RestartPlayer(Controller);
       SetPlayerColor(Controller);
   }

5. 修改`STUBaseCharacter`：添加`SetPlayerColor()`

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUBaseCharacter : public ACharacter {
       ...
   
   protected:
       // 角色材质的颜色属性名
       UPROPERTY(EditDefaultsOnly, Category = "Damage")
       FName MaterialColorName = "Paint Color";
   
   public:
       // 设置角色的颜色
       void SetPlayerColor(const FLinearColor& Color);
   };
   ```

   ```c++
   void ASTUBaseCharacter::SetPlayerColor(const FLinearColor& Color) {
       const auto MaterialInst = GetMesh()->CreateAndSetMaterialInstanceDynamic(0);
       if (!MaterialInst) return;
   
       MaterialInst->SetVectorParameterValue(MaterialColorName, Color);
   }

6. 修改`BP_STUGameModeBase`：设置`TeamColor`

# 六、给角色分配团队

> 在`STUAIPerceptionComponent`中，可以判断观察到的Actor是否为敌人

1. 修改`STUUtils`：判断两个Controller是否为敌人

   ```c++
   #pragma once
   #include "Player/STUPlayerState.h"
   
   class STUUtils {
   public:
       ...
       bool static AreEnemies(AController* Controller1, AController* Controller2) {
           if (!Controller1 || !Controller2 || Controller1 == Controller2) return false;
           
           const auto PlayerState1 = Cast<ASTUPlayerState>(Controller1->PlayerState);
           const auto PlayerState2 = Cast<ASTUPlayerState>(Controller2->PlayerState);
           if (!PlayerState1 || !PlayerState2) return false;
   
           return PlayerState1->GetTeamID() != PlayerState2->GetTeamID();
       }
   };
   ```

2. 修改`STUAIPerceptionComponent/GetClosetEnemy()`：

   ```c++
   AActor* USTUAIPerceptionComponent::GetClosetEnemy() const {
       // 获取AI视野内的所有Actor
       TArray<AActor*> PerciveActors;
       GetCurrentlyPerceivedActors(UAISense_Sight::StaticClass(), PerciveActors);
       if (PerciveActors.Num() == 0) return nullptr;
   
       // 获取当前角色的Pawn
       const auto Controller = Cast<AAIController>(GetOwner());
       if (!Controller) return nullptr;
       const auto Pawn = Controller->GetPawn();
       if (!Pawn) return nullptr;
   
       // 获取距离当前角色最近的Character
       float ClosetDistance = MAX_FLT;
       AActor* ClosetActor = nullptr;
       for (const auto PerciveActor : PerciveActors) {
           // 判断character是否已死亡
           const auto HealthComponent = STUUtils::GetSTUPlayerComponent<USTUHealthComponent>(PerciveActor);
           if (!HealthComponent || HealthComponent->IsDead()) continue;
   
           // 判断两个character是否为敌人
           const auto PercivePawn = Cast<APawn>(PerciveActor);
           const auto AreEnemies = PercivePawn && STUUtils::AreEnemies(Controller, PercivePawn->Controller);
           if (!AreEnemies) continue;
           
           // 更新距离信息
           const auto CurrentDistance = (PerciveActor->GetActorLocation() - Pawn->GetActorLocation()).Size();
           if (CurrentDistance < ClosetDistance) {
               ClosetDistance = CurrentDistance;
               ClosetActor = PerciveActor;
           }
       }
   
       return ClosetActor;
   }

# 七、PlayerState：击杀数、死亡数

1. 修改`STUPlayerState`：添加击杀数、死亡数的数据记录

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUPlayerState : public APlayerState {
       ...
   
   public:
       void AddKill() { KillsNum++; }
       int32 GetKillsNum() const { return KillsNum; }
       void AddDeath() { DeathsNum++; }
       int32 GetDeathsNum() const { return DeathsNum; }
   
       void LogInfo();
   
   private:
       int32 KillsNum = 0;
       int32 DeathsNum = 0;
   };
   ```

   ```c++
   #include "Player/STUPlayerState.h"
   
   DEFINE_LOG_CATEGORY_STATIC(LogSTUPlayerState, All, All);
   
   void ASTUPlayerState::LogInfo() {
       UE_LOG(LogSTUPlayerState, Display, TEXT("TeamID: %i, Kills: %i, Deaths: %i"), TeamID, KillsNum, DeathsNum);
   }

2. 修改`STUGameModeBase`：添加A杀死B的逻辑

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUGameModeBase : public AGameModeBase {
       ...
   public:
       // 击杀事件
       void Killed(AController* KillerController, AController* VictimController);
   
   private:
       // 输出角色信息到日志
       void LogPlayerInfo();
   };
   ```

   ```c++
   void ASTUGameModeBase::Killed(AController* KillerController, AController* VictimController) {
       const auto KillerPlayerState = KillerController ? Cast<ASTUPlayerState>(KillerController->PlayerState) : nullptr;
       const auto VictimPlayerState = VictimController ? Cast<ASTUPlayerState>(VictimController->PlayerState) : nullptr;
   
       if (KillerPlayerState) KillerPlayerState->AddKill();
       if (VictimPlayerState) VictimPlayerState->AddDeath();
   }
   
   void ASTUGameModeBase::LogPlayerInfo() {
       if (!GetWorld()) return;
   
       for (auto It = GetWorld()->GetControllerIterator(); It; ++It) {
           const auto Controller = It->Get();
           if (!Controller) continue;
   
           const auto PlayerState = Cast<ASTUPlayerState>(Controller->PlayerState);
           if (!PlayerState) continue;
   
           PlayerState->LogInfo();
       }
   }

3. 修改`STUHealthComponent`：添加被A杀死的逻辑

   ```c++
   UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
   class SHOOTTHEMUP_API USTUHealthComponent : public UActorComponent {
       ...
   
   private:
       // 被杀死
       void Killed(AController* KillerController);
   };
   ```

   ```c++
   #include "STUGameModeBase.h"
   
   void USTUHealthComponent::OnTakeAnyDamageHandler(
       AActor* DamagedActor, float Damage, const UDamageType* DamageType, AController* InstigatedBy, AActor* DamageCauser) {
       if (Damage <= 0.0f || IsDead() || !GetWorld()) return;
   
       SetHealth(Health - Damage);
   
       // 角色受伤时, 停止自动恢复
       GetWorld()->GetTimerManager().ClearTimer(HealTimerHandle);
       
       // 角色死亡后, 广播OnDeath委托
       if (IsDead()) {
           Killed(InstigatedBy);
           OnDeath.Broadcast();
       }
       // 角色未死亡且可以自动恢复
       else if (AutoHeal) {
           GetWorld()->GetTimerManager().SetTimer(HealTimerHandle, this, &USTUHealthComponent::HealUpdate, HealUpdateTime, true, HealDelay);
       }
   
       // 相机抖动
       PlayCameraShake();
   }
   
   void USTUHealthComponent::Killed(AController* KillerController) {
       if (!GetWorld()) return;
   
       const auto GameMode = Cast<ASTUGameModeBase>(GetWorld()->GetAuthGameMode());
       if (!GameMode) return;
   
       const auto Player = Cast<APawn>(GetOwner());
       if (!Player) return;
   
       const auto VictimController = Player->GetController();
       GameMode->Killed(KillerController, VictimController);
   }

4. 修改`STURifleWeapon/MakeDamage()`：造成伤害时，使用AController*传递造成伤害的控制器

   ```c++
   void ASTURifleWeapon::MakeDamage(const FHitResult& HitResult) {
       const auto DamageActor = HitResult.GetActor();
       if (!DamageActor) return;
   
       DamageActor->TakeDamage(DamageAmount, FDamageEvent{}, GetController(), this);
   }
   
   AController* ASTURifleWeapon::GetController() const {
       const auto Pawn = Cast<APawn>(GetOwner());
       return Pawn ? Pawn->GetController() : nullptr;
   }
   ```

5. 修改`STUBaseWeapon/GetPlayerViewPoint`：删除`GetPlayerController()`函数

   ```c++
   bool ASTUBaseWeapon::GetPlayerViewPoint(FVector& ViewLocation, FRotator& ViewRotation) const {
       const auto STUCharacter = Cast<ACharacter>(GetOwner());
       if (!STUCharacter) return false;
   
       // 如果为玩家控制, 则返回玩家的朝向
       if (STUCharacter->IsPlayerControlled()) {
           const auto Controller = STUCharacter->GetController<APlayerController>();
           if (!Controller) return false;
           Controller->GetPlayerViewPoint(ViewLocation, ViewRotation);
       } 
       // 如果为AI控制, 则返回枪口的朝向
       else {
           ViewLocation = GetMuzzleWorldLocation();
           ViewRotation = WeaponMesh->GetSocketRotation(MuzzleSocketName);
       }
   
       return true;
   }
   ```

# 八、使用UI显示回合时间等数据

1. 创建C++类`STUGameDataWidget`，继承于`UUserWidget`

   1. 目录：`ShootThemUp/Source/ShootThemUp/Public/UI`

2. 修改`STUGameModeBase`：创建GameData、RoundNum、RoundSeconds的getter函数

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUGameModeBase : public AGameModeBase {
       ...
   public:
       FGameData GetGameData() const { return GameData; }
       int32 GetCurrentRoundNum() const { return CurrentRound; }
       int32 GetRoundSecondsRemaining() const { return RoundCountDown; }
   };

3. 修改`STUGameDataWidget`：

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "Blueprint/UserWidget.h"
   #include "STUGameDataWidget.generated.h"
   
   class ASTUGameModeBase;
   class ASTUPlayerState;
   
   UCLASS()
   class SHOOTTHEMUP_API USTUGameDataWidget : public UUserWidget {
       GENERATED_BODY()
   
   public:
       UFUNCTION(BlueprintCallable, Category = "UI")
       int32 GetKillsNum() const;
       
       UFUNCTION(BlueprintCallable, Category = "UI")
       int32 GetCurrentRoundNum() const;
   
       UFUNCTION(BlueprintCallable, Category = "UI")
       int32 GetTotalRoundsNum() const;
   
       UFUNCTION(BlueprintCallable, Category = "UI")
       int32 GetRoundSecondsRemaining() const;
   
   private:
       ASTUGameModeBase* GetSTUGameMode() const;
   
       ASTUPlayerState* GetSTUPlayerState() const;
   }
   ```

   ```c++
   #include "UI/STUGameDataWidget.h"
   #include "STUGameModeBase.h"
   #include "Player/STUPlayerState.h"
   
   int32 USTUGameDataWidget::GetKillsNum() const {
       const auto PlayerState = GetSTUPlayerState();
       if (!PlayerState) return 0;
       return PlayerState->GetKillsNum();
   }
   
   int32 USTUGameDataWidget::GetCurrentRoundNum() const {
       const auto GameMode = GetSTUGameMode();
       if (!GameMode) return 0;
       return GameMode->GetCurrentRoundNum();
   }
   
   int32 USTUGameDataWidget::GetTotalRoundsNum() const {
       const auto GameMode = GetSTUGameMode();
       if (!GameMode) return 0;
       return GameMode->GetGameData().RoundsNum;
   }
   
   int32 USTUGameDataWidget::GetRoundSecondsRemaining() const {
       const auto GameMode = GetSTUGameMode();
       if (!GameMode) return 0;
       return GameMode->GetRoundSecondsRemaining();
   }
   
   ASTUGameModeBase* USTUGameDataWidget::GetSTUGameMode() const {
       if (!GetWorld()) return nullptr;
       return Cast<ASTUGameModeBase>(GetWorld()->GetAuthGameMode());
   }
   
   ASTUPlayerState* USTUGameDataWidget::GetSTUPlayerState() const {
       if (GetOwningPlayer()) return nullptr;
       return Cast<ASTUPlayerState>(GetOwningPlayer()->PlayerState);
   }

4. 新建蓝图类`WBP_GameData`，继承于`STUGameDataWidget`

   1. 路径：`UI`

5. 修改`WBP_GameData`：

   <img src="AssetMarkdown/image-20230225212951685.png" alt="image-20230225212951685" style="zoom:80%;" />

   <img src="AssetMarkdown/image-20230225213506893.png" alt="image-20230225213506893" style="zoom:80%;" />

   <img src="AssetMarkdown/image-20230225213003425.png" alt="image-20230225213003425" style="zoom:80%;" />

   <img src="AssetMarkdown/image-20230225213016231.png" alt="image-20230225213016231" style="zoom:80%;" />

6. 修改`BP_PlayerHUD`：添加`WBP_GameData`，可视性绑定为`Is_Player_Alive`

# 九、重生组件：在回合结束前重生

1. 创建C++类`STURespawnComponent`，继承于`Actor组件`

   1. 目录：`ShootThemUp/Source/ShootThemUp/Public/Components`

2. 修改`STUCoreTypes/FGameData`：设置重生时间

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
   
       // 默认队伍颜色
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite)
       FLinearColor DefaultTeamColor = FLinearColor::Red;
   
       // 队伍可选颜色
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite)
       TArray<FLinearColor> TeamColors;
   
       // 重生的时间(s)
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "Game", meta = (ClampMin = "3", ClampMax = "20"))
       int32 RespawnTime = 5;
   };
   ```

3. 修改`STURespawnComponent`：

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "Components/ActorComponent.h"
   #include "STURespawnComponent.generated.h"
   
   UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
   class SHOOTTHEMUP_API USTURespawnComponent : public UActorComponent {
       GENERATED_BODY()
   
   public:
       USTURespawnComponent();
   
       // 一段时间后重生
       void Respawn(int32 RespawnTime);
   
   private:
       FTimerHandle RespawnTimerHandle;
       int32 RespawnCountDown = 0;
       void RespawnTimerUpdate();
   };
   ```

   ```c++
   #include "Components/STURespawnComponent.h"
   #include "STUGameModeBase.h"
   
   USTURespawnComponent::USTURespawnComponent() {
       PrimaryComponentTick.bCanEverTick = false;
   }
   
   // 一段时间后重生
   void USTURespawnComponent::Respawn(int32 RespawnTime) {
       if (!GetWorld()) return;
   
       RespawnCountDown = RespawnTime;
       GetWorld()->GetTimerManager().SetTimer(RespawnTimerHandle, this, &USTURespawnComponent::RespawnTimerUpdate, 1.0f, true);
   }
   
   void USTURespawnComponent::RespawnTimerUpdate() {
       RespawnCountDown--;
       if (RespawnCountDown <= 0) {
           if (!GetWorld()) return;
           GetWorld()->GetTimerManager().ClearTimer(RespawnTimerHandle);
   
           const auto GameMode = Cast<ASTUGameModeBase>(GetWorld()->GetAuthGameMode());
           if (!GameMode) return;
   
           GameMode->RespawnResqust(Cast<AController>(GetOwner()));
       }
   }
   ```

4. 修改`STUGameMode`：添加角色被击杀后重生的逻辑

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUGameModeBase : public AGameModeBase {
       ...
   public:
       // 请求重新生成Character
       void RespawnResqust(AController* Controller);
   
   private:
       // 重新生成角色
       void StartRespawn(AController* Controller);
   };
   ```

   ```c++
   constexpr static int32 MinRoundTimeForRespawn = 10;
   
   void ASTUGameModeBase::Killed(AController* KillerController, AController* VictimController) {
       const auto KillerPlayerState = KillerController ? Cast<ASTUPlayerState>(KillerController->PlayerState) : nullptr;
       const auto VictimPlayerState = VictimController ? Cast<ASTUPlayerState>(VictimController->PlayerState) : nullptr;
   
       if (KillerPlayerState) KillerPlayerState->AddKill();
       if (VictimPlayerState) VictimPlayerState->AddDeath();
   
       // 让victim重生
       StartRespawn(VictimController);
   }
   
   void ASTUGameModeBase::StartRespawn(AController* Controller) {
       // 剩余时间不足以重生
       if (RoundCountDown <= GameData.RespawnTime + MinRoundTimeForRespawn) return;
   
       const auto RespawnComponent = STUUtils::GetSTUPlayerComponent<USTURespawnComponent>(Controller);
       if (!RespawnComponent) return;
   
       RespawnComponent->Respawn(GameData.RespawnTime);
   }
   
   void ASTUGameModeBase::RespawnResqust(AController* Controller) {
       ResetOnePlayer(Controller);
   }

5. 修改`STUAIController、STUPlayerController`：添加`RespawnComponent`

   ```c++
   #include "Components/STURespawnComponent.h"
   
   ASTUAIController::ASTUAIController() {
       // 创建AI感知组件
       STUAIPerceptionComponent = CreateDefaultSubobject<USTUAIPerceptionComponent>("STUAIPerceptionComponent");
       SetPerceptionComponent(*STUAIPerceptionComponent);
   
       // 创建重生组件
       RespawnComponent = CreateDefaultSubobject<USTURespawnComponent>("STURespawnComponent");
       
   
       // 为AI分配PlayerState
       bWantsPlayerState = true;
   }
   ```

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "GameFramework/PlayerController.h"
   #include "STUPlayerController.generated.h"
   
   class USTURespawnComponent;
   
   UCLASS()
   class SHOOTTHEMUP_API ASTUPlayerController : public APlayerController {
       GENERATED_BODY()
   
   public:
       ASTUPlayerController();
   
   protected:
       UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Components")
       USTURespawnComponent* RespawnComponent;
   };
   ```

   ```c++
   #include "Player/STUPlayerController.h"
   #include "Components/STURespawnComponent.h"
   
   ASTUPlayerController::ASTUPlayerController() {
       // 创建重生组件
       RespawnComponent = CreateDefaultSubobject<USTURespawnComponent>("STURespawnComponent");
   }
   ```


# 十、重生时的UI

1. 创建C++类`STUSpectatorWidget`，继承于`UserWidget`

   1. 目录：`ShootThemUp/Source/ShootThemUp/Public/UI`

2. 修改`STURespawnComponent`：添加`RespawnTime`的getter

   ```c++
   UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
   class SHOOTTHEMUP_API USTURespawnComponent : public UActorComponent {
       ...
   
   public:
       int32 GetRespawnCountDown() const { return RespawnCountDown; }
   
       // 正在重生
       bool IsRespawnInProgress() const;
   };
   ```

   ```c++
   bool USTURespawnComponent::IsRespawnInProgress() const {
       if (!GetWorld()) return false;
       return GetWorld()->GetTimerManager().IsTimerActive(RespawnTimerHandle);
   }

3. 修改`STUSpectatorWidget`：

   ```c++
   #pragma once
   
   #include "CoreMinimal.h"
   #include "Blueprint/UserWidget.h"
   #include "STUSpectatorWidget.generated.h"
   
   UCLASS()
   class SHOOTTHEMUP_API USTUSpectatorWidget : public UUserWidget {
       GENERATED_BODY()
   
   public:
       UFUNCTION(BlueprintCallable, Category = "UI")
       bool GetRespawnTime(int32& CountDownTime) const;
   };
   ```

   ```c++
   #include "UI/STUSpectatorWidget.h"
   #include "STUUtils.h"
   #include "Components/STURespawnComponent.h"
   
   bool USTUSpectatorWidget::GetRespawnTime(int32& CountDownTime) const {
       const auto RespawnComponent = STUUtils::GetSTUPlayerComponent<USTURespawnComponent>(GetOwningPlayer());
       if (!RespawnComponent || !RespawnComponent->IsRespawnInProgress()) return false;
   
       CountDownTime = RespawnComponent->GetRespawnCountDown();
       return true;
   }

4. 修改`WBP_SpectatorHUD`：

   1. 将父类修改为`STUSpectatorWidget`
   2. 为文本添加一个绑定

   <img src="AssetMarkdown/image-20230227194002411.png" alt="image-20230227194002411" style="zoom:80%;" />

