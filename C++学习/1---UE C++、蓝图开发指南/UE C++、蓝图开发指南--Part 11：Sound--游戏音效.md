[TOC]

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