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

8. 