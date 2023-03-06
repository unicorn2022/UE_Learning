[TOC]

# 一、步枪：长按E键瞄准

1. 新建操作映射`Zoom`：E键

2. 修改`STUBaseWeapon`：添加虚函数`Zoom()`，步枪可以瞄准，发射器不能瞄准

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTUBaseWeapon : public AActor {
       ...
   
   public:
       // 缩放
       virtual void Zoom(bool Enabled) {}
   };

3. 修改`STURifleWeapon`：实现`Zoom()`的功能

   ```c++
   UCLASS()
   class SHOOTTHEMUP_API ASTURifleWeapon : public ASTUBaseWeapon {
       ...
   
   public:
       // 缩放
       virtual void Zoom(bool Enabled) override;
   
   protected:
       // 缩放的视角
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "Weapon")
       float FOVZoomAngle = 50.0f;
   
   private:
       // 默认缩放视角
       float DefaultCameraFOV = 90.0f;
   };
   ```

   ```c++
   void ASTURifleWeapon::Zoom(bool Enabled) {
       const auto Controller = Cast<APlayerController>(GetController());
       if (!Controller || !Controller->PlayerCameraManager) return;
   
       if (Enabled) {
           DefaultCameraFOV = Controller->PlayerCameraManager->GetFOVAngle();
       }
   
       // 根据Enabled的值, 直接修改相机的视场
       Controller->PlayerCameraManager->SetFOV(Enabled ? FOVZoomAngle : DefaultCameraFOV);
   }

4. 修改`STUWeaponComponent`：添加`Zoom()`函数

   ```c++
   void USTUWeaponComponent::Zoom(bool Enabled) {
       if (CurrentWeapon) CurrentWeapon->Zoom(Enabled);
   }

5. 修改`STUPlayerCharacter`：绑定输入

   ```c++
   void ASTUPlayerCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) {
       Super::SetupPlayerInputComponent(PlayerInputComponent);
   
       check(PlayerInputComponent);
   
       // WASD控制角色移动
       PlayerInputComponent->BindAxis("MoveForward", this, &ASTUPlayerCharacter::MoveForward);
       PlayerInputComponent->BindAxis("MoveRight", this, &ASTUPlayerCharacter::MoveRight);
   
       // 鼠标控制相机移动
       PlayerInputComponent->BindAxis("LookUp", this, &ASTUPlayerCharacter::AddControllerPitchInput);
       PlayerInputComponent->BindAxis("TurnAround", this, &ASTUPlayerCharacter::AddControllerYawInput);
   
       // 空格键控制角色跳跃
       PlayerInputComponent->BindAction("Jump", IE_Pressed, this, &ASTUPlayerCharacter::Jump);
   
       // 左Shift控制角色开始跑动
       PlayerInputComponent->BindAction("Run", IE_Pressed, this, &ASTUPlayerCharacter::OnStartRunning);
       PlayerInputComponent->BindAction("Run", IE_Released, this, &ASTUPlayerCharacter::OnStopRunning);
   
       // 鼠标左键控制武器开火
       PlayerInputComponent->BindAction("Fire", IE_Pressed, WeaponComponent, &USTUWeaponComponent::StartFire);
       PlayerInputComponent->BindAction("Fire", IE_Released, WeaponComponent, &USTUWeaponComponent::StopFire);
   
       // Tab键切换武器
       PlayerInputComponent->BindAction("NextWeapon", IE_Pressed, WeaponComponent, &USTUWeaponComponent::NextWeapon);
   
       // R键切换弹夹
       PlayerInputComponent->BindAction("Reload", IE_Pressed, WeaponComponent, &USTUWeaponComponent::Reload);
   
       // 长按E键瞄准
       DECLARE_DELEGATE_OneParam(FZoomInputSignature, bool);
       PlayerInputComponent->BindAction<FZoomInputSignature>("Zoom", IE_Pressed, WeaponComponent, &USTUWeaponComponent::Zoom, true);
       PlayerInputComponent->BindAction<FZoomInputSignature>("Zoom", IE_Released, WeaponComponent, &USTUWeaponComponent::Zoom, false);
   }
   ```

6. 修改`STUBaseCharacter/OnDeath()`：死亡时停止缩放

   ```c++
   void ASTUBaseCharacter::OnDeath() {
       UE_LOG(LogSTUBaseCharacter, Warning, TEXT("Player %s is dead"), *GetName());
       // 播放死亡动画蒙太奇
       // PlayAnimMontage(DeathAnimMontage);
   
       // 禁止角色的移动
       GetCharacterMovement()->DisableMovement();
   
       // 一段时间后摧毁角色
       SetLifeSpan(LifeSpanOnDeath);
   
       // 禁止胶囊体碰撞
       GetCapsuleComponent()->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
   
       // 停止武器组件的开火 & 缩放
       WeaponComponent->StopFire();
       WeaponComponent->Zoom(false);
   
       // 启用物理模拟, 实现角色死亡效果
       GetMesh()->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
       GetMesh()->SetSimulatePhysics(true);
   
       // 播放音效
       UGameplayStatics::PlaySoundAtLocation(GetWorld(), DeathSound, GetActorLocation());
   }

7. 修改`STUWeaponComponent/EquipWeapon()`：切换武器时停止缩放

   ```c++
   void USTUWeaponComponent::EquipWeapon(int32 WeaponIndex) {
       if (WeaponIndex < 0 || WeaponIndex >= Weapons.Num()) {
           UE_LOG(LogSTUWeaponComponent, Warning, TEXT("Invalid Weapon Index!!!"));
           return;
       }
   
       // 判断角色是否存在
       ACharacter* Character = Cast<ACharacter>(GetOwner());
       if (!GetWorld() || !Character) return;
   
       // 如果已经有武器, 将当前武器转移到背后, 停止开火 & 缩放
       if (CurrentWeapon) {
           CurrentWeapon->StopFire();
           CurrentWeapon->Zoom(false);
           AttachWeaponToSocket(CurrentWeapon, Character->GetMesh(), WeaponAmorySocketName);
       }
   
       // 更换手上的武器
       CurrentWeapon = Weapons[WeaponIndex];
       const auto CurrentWeaponData =
           WeaponData.FindByPredicate([&](const FWeaponData& Data) { return Data.WeaponClass == CurrentWeapon->GetClass(); });
       CurrentReloadAnimMontage = CurrentWeaponData ? CurrentWeaponData->ReloadAnimMontage : nullptr;
       AttachWeaponToSocket(CurrentWeapon, Character->GetMesh(), WeaponEquipSocketName);
   
       // 播放更换武器的动画
       EquipAnimInProgress = true;
       PlayAnimMontage(EquipAnimMontage);
   }

# 二、击中不同部位，造成不同伤害

1. 修改`STURifleWeapon/MakeDamage()`：传递击中的部位

   ```c++
   void ASTURifleWeapon::MakeDamage(const FHitResult& HitResult) {
       const auto DamageActor = HitResult.GetActor();
       if (!DamageActor) return;
   
       FPointDamageEvent PointDamageEvent;
       PointDamageEvent.HitInfo = HitResult;
       DamageActor->TakeDamage(DamageAmount, PointDamageEvent, GetController(), this);
   }
   ```

2. 修改`STUHealthComponent`：订阅`OnTakePointDamage`

   ```c++
   class UPhysicalMaterial;
   
   UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
   class SHOOTTHEMUP_API USTUHealthComponent : public UActorComponent {
       ...
   
   private:
       // 委托：角色受到点伤害
       UFUNCTION()
       void OnTakePointDamage(AActor* DamagedActor, float Damage, class AController* InstigatedBy, FVector HitLocation,
           class UPrimitiveComponent* FHitComponent, FName BoneName, FVector ShotFromDirection, const class UDamageType* DamageType,
           AActor* DamageCauser);
   
       // 委托：角色受到范围伤害
       UFUNCTION()
       void OnTakeRadialDamage(AActor* DamagedActor, float Damage, const class UDamageType* DamageType, FVector Origin, FHitResult HitInfo,
           class AController* InstigatedBy, AActor* DamageCauser);
   
       // 造成伤害
       void ApplyDamage(float Damage, AController* InstigatedBy);
   
       // 获取击中某个骨骼需要造成的伤害修正
       float GetPointDamageModifier(AActor* DamagedActor, const FName& BoneName);
   };
   ```

   ```c++
   #include "GameFramework/Character.h"
   #include "GameFramework/Controller.h"
   #include "PhysicalMaterials/PhysicalMaterial.h"
   
   void USTUHealthComponent::BeginPlay() {
       Super::BeginPlay();
   
       check(MaxHealth > 0);
   
       SetHealth(MaxHealth);
   
       // 订阅OnTakeAnyDamage事件
       AActor* ComponentOwner = GetOwner();
       if (ComponentOwner) {
           ComponentOwner->OnTakeAnyDamage.AddDynamic(this, &USTUHealthComponent::OnTakeAnyDamageHandler);
           ComponentOwner->OnTakePointDamage.AddDynamic(this, &USTUHealthComponent::OnTakePointDamage);
           ComponentOwner->OnTakeRadialDamage.AddDynamic(this, &USTUHealthComponent::OnTakeRadialDamage);
       }
   }
   
   void USTUHealthComponent::OnTakeAnyDamageHandler(
       AActor* DamagedActor, float Damage, const UDamageType* DamageType, AController* InstigatedBy, AActor* DamageCauser) {
   
       UE_LOG(LogSTUHealthComponent, Display, TEXT("On any damage: %f"), Damage);
   
   }
   
   void USTUHealthComponent::OnTakePointDamage(AActor* DamagedActor, float Damage, AController* InstigatedBy, FVector HitLocation,
       UPrimitiveComponent* FHitComponent, FName BoneName, FVector ShotFromDirection, const UDamageType* DamageType, AActor* DamageCauser) {
       
       const auto FinalDamage = Damage * GetPointDamageModifier(DamagedActor, BoneName);
       UE_LOG(LogSTUHealthComponent, Display, TEXT("On point damage: %f, final damage: %f, bone: %s"), Damage, FinalDamage, *BoneName.ToString());
       ApplyDamage(FinalDamage, InstigatedBy);
   
   }
   
   void USTUHealthComponent::OnTakeRadialDamage(AActor* DamagedActor, float Damage, const UDamageType* DamageType, FVector Origin,
       FHitResult HitInfo, AController* InstigatedBy, AActor* DamageCauser) {
       
       UE_LOG(LogSTUHealthComponent, Display, TEXT("On radial damage: %f"), Damage);
       ApplyDamage(Damage, InstigatedBy);
   
   }
   
   void USTUHealthComponent::ApplyDamage(float Damage, AController* InstigatedBy) {
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
   
   float USTUHealthComponent::GetPointDamageModifier(AActor* DamagedActor, const FName& BoneName) {
       const auto Character = Cast<ACharacter>(DamagedActor);
       if (!Character) return 1.0f;
   
       const auto Mesh = Character->GetMesh();
       if (!Mesh) return 1.0f;
   
       const auto BodyInstance = Mesh->GetBodyInstance(BoneName);
       if (!BodyInstance) return 1.0f;
   
       const auto PhysMaterial = BodyInstance->GetSimplePhysicalMaterial();
       if (!PhysMaterial || !DamageModifiers.Contains(PhysMaterial)) return 1.0f;
   
       return DamageModifiers[PhysMaterial];
   }

3. 修改角色的骨骼树：将手、腿的物理材料赋值给对应的碰撞体

4. 修改`BP_STUAICharacter、BP_STUPlayerCharacter`：设置`DamageModifiers`

   <img src="AssetMarkdown/image-20230307020301087.png" alt="image-20230307020301087" style="zoom:80%;" />

5. 