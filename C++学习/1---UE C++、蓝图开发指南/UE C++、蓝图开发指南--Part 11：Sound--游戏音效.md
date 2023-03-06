[TOC]

> 在命令行界面，输入`Audio3dVisualize`，可以看到声音相关数据

# 一、蓝图播放音效：点击按钮

1. 新建音效/SoundCue`SCue_ButtonHover、SCue_ButtonPressed`

   1. 路径：`Content/Sounds/UI`

   |                       SCue_ButtonHover                       |                      SCue_ButtonPressed                      |
   | :----------------------------------------------------------: | :----------------------------------------------------------: |
   | <img src="AssetMarkdown/image-20230306203812133.png" alt="image-20230306203812133" style="zoom:80%;" /> | <img src="AssetMarkdown/image-20230306203907183.png" alt="image-20230306203907183" style="zoom:80%;" /> |

2. 修改所有包含按钮的UI控件：

   1. 在按钮的`外观/样式/按压音效、悬停音效`中，可以设置刚刚创建的音效

   <img src="AssetMarkdown/image-20230306204425544.png" alt="image-20230306204425544" style="zoom:80%;" />

# 二、C++播放音效：开始游戏、显示UI界面

1. 基于`sw_UI_GameStart`创建音效`SCue_GameStart`

   1. 路径：`Content/Sounds/UI`

2. 基于`sw_UI_Open`创建音效`SCue_WidgetOpen`

   1. 路径：`Content/Sounds/UI`

3. 修改`STUMenuUserWidget`：添加开始游戏音效

   ```c++
   class USoundCue;
   
   UCLASS()
   class SHOOTTHEMUP_API USTUMenuUserWidget : public USTUBaseWidget {
       ...
   
   protected:
       // 音效：开始游戏
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "Sound")
       USoundCue* StartGameSound;
   };
   ```

   ```c++
   #include "Sound/SoundCue.h"
   void USTUMenuUserWidget::OnStartGame() {
       PlayAnimation(HideAnimation);
       UGameplayStatics::PlaySound2D(GetWorld(), StartGameSound);
   }

4. 修改`STUBaseWidget`：添加UI出现音效

   ```c++
   class USoundCue;
   
   UCLASS()
   class SHOOTTHEMUP_API USTUBaseWidget : public UUserWidget {
       GENERATED_BODY()
   
   public:
       void Show();
   
   protected:
       // 动画：显示UI
       UPROPERTY(meta = (BindWidgetAnim), Transient)
       UWidgetAnimation* ShowAnimation;
   
       // 音效：显示UI
       UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category = "Sound")
       USoundCue* OpenSound;
   };
   ```

   ```c++
   #include "UI/STUBaseWidget.h"
   #include "Kismet/GameplayStatics.h"
   #include "Sound/SoundCue.h"
   
   void USTUBaseWidget::Show() {
       PlayAnimation(ShowAnimation);
       UGameplayStatics::PlaySound2D(GetWorld(), OpenSound);
   }

5. 修改`WBP_Menu`：

   1. 设置`StartGameSound`为`SCue_GameStart`

6. 修改`WBP_GamePause、WBP_GameOver`：

   1. 设置`OpenSound`为`SCue_WidgetOpen`

7. 在小的UI控件上不设置`OpenSound`，防止声音过大

# 三、动画音效：走路、奔跑、跳跃

1. 创建音效`SCue_FootstepsWalk`

   1. 路径：`Content/Sounds/Character`

   <img src="AssetMarkdown/image-20230306211850357.png" alt="image-20230306211850357" style="zoom:80%;" />

2. 修改动画`Run_Bwd、Run_Fwd、Run_Lt、Run_Rt`

   1. 在角色脚步触及地面的那两帧，播放音效`SCue_FootstepsWalk`

   <img src="AssetMarkdown/image-20230306212455149.png" alt="image-20230306212455149" style="zoom:80%;" />

3. 创建音效`SCue_FootstepsRun`

   1. 路径：`Content/Sounds/Character`

   <img src="AssetMarkdown/image-20230306213002791.png" alt="image-20230306213002791" style="zoom:80%;" />

4. 修改动画`RoadieRun_Fwd`

   1. 在角色脚步触及地面的那两帧，播放音效`SCue_FootstepsRun`

5. 创建音效`SCue_JumpStart`

   1. 路径：`Content/Sounds/Character`

   <img src="AssetMarkdown/image-20230306213155011.png" alt="image-20230306213155011" style="zoom:80%;" />

6. 修改动画`JumpStart`

   1. 在角色脚抬离地面的那一帧，播放音效`SCue_JumpStart`

7. 创建音效`SCue_JumpEnd`

   1. 路径：`Content/Sounds/Character`

   <img src="AssetMarkdown/image-20230306213314934.png" alt="image-20230306213314934" style="zoom:80%;" />

8. 修改动画`JumpEnd`

   1. 在角色脚抬离地面的那一帧，播放音效`SCue_JumpEnd`

# 四、动画音效：切换武器、切换弹夹

1. 创建音效`SCue_WeaponEquip`

   1. 路径：`Content/Sounds/Weapon`

   <img src="AssetMarkdown/image-20230306214621360.png" alt="image-20230306214621360" style="zoom:80%;" />

2. 创建音效`SCue_RifleReload`

   1. 路径：`Content/Sounds/Weapon/Rifle`

   <img src="AssetMarkdown/image-20230306214708054.png" alt="image-20230306214708054" style="zoom:80%;" />

3. 创建音效`SCue_LauncherReloadStart`

   1. 路径：`Content/Sounds/Weapon/Launcher`

   <img src="AssetMarkdown/image-20230306214740622.png" alt="image-20230306214740622" style="zoom:80%;" />

4. 创建音效`SCue_LauncherReloadEnd`

   1. 路径：`Content/Sounds/Weapon/Launcher`

   <img src="AssetMarkdown/image-20230306214801982.png" alt="image-20230306214801982" style="zoom:80%;" />

5. 在对应动画处插入音效

# 五、音效类：管理一组声音的设置

1. 创建音效/类/音效类`SC_Character`

   1. 路径：`Content/Sounds/Settings`

2. 将所有与角色相关的音效的类设置为`SoundClassCharacter`

   <img src="AssetMarkdown/image-20230306215917510.png" alt="image-20230306215917510" style="zoom:80%;" />

3. 创建音效/类/音效类`SC_UI、SC_Weapon`，并设置对应的音效

   1. 路径：`Content/Sounds/Settings`

# 六、音量衰减

1. 创建音效/类/音效类`SC_ProjectileFly`

   1. 路径：`Content/Sounds/Weapon/Projectile`

   <img src="AssetMarkdown/image-20230306220941740.png" alt="image-20230306220941740" style="zoom:80%;" />

2. 创建音效衰减`SA_ProjectileFly`

   1. 路径：`Content/Sounds/Weapon/Projectile`

3. 将`SC_ProjectileFly/输出`的`衰减设置`，设置为`SA_ProjectileFly`

4. 将之前所有的音效的衰减设置，设置为`ExternalContent/Sounds/Audio_Settings/Attenuation`中的设置

   1. 武器音效：`Foley`
   2. 走路：`Footsteps`
   3. 跑步：`Foley`
   4. 榴弹：`Projectile`

5. 修改`BP_STUProjectile`

   1. 添加音频组件，音效设置为`SC_ProjectileFly`

# 七、