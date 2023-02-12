# 目录

[TOC]

# 一、角色健康栏--蓝图版

1. 修改`STUHealthComponent`：添加函数`GetHealthPercent`

   ```c++
   UCLASS( ClassGroup=(Custom), meta=(BlueprintSpawnableComponent) )
   class SHOOTTHEMUP_API USTUHealthComponent : public UActorComponent{
       ...
   public:	
       // 获取角色当前生命值
       float GetHealth() const { return Health; }
   
       // 获取角色当前生命值百分比
       UFUNCTION(BlueprintCallable, Category = "Health")
       float GetHealthPercent() const { return Health / MaxHealth; }
   
   	...
   };
   ```

2. 创建文件夹`UI`，然后在该目录下创建`用户界面/控件蓝图`，重命名为`WBP_PlayerHUD`

3. 基于`STUGameHUD`，创建蓝图类`BP_STUGameHUD`

   1. 路径：`UI`

4. 修改`BP_STUGameHUD`：

   <img src="AssetMarkdown/image-20230212124240282.png" alt="image-20230212124240282" style="zoom:80%;" />

5. 将HUD类设置为`BP_STUGameHUD`，在游戏中可以通过`Shift+F1`解放鼠标

6. 修改`WBP_PlayerHUD`

   1. 添加`进度条`控件，将填充颜色设为`绿色`
   2. 对`进度/百分比`属性创建绑定，函数重命名为`Get_HealthPercent`

   <img src="AssetMarkdown/image-20230212125221710.png" alt="image-20230212125221710" style="zoom:80%;" />

7. 修改`BP_BaseCharacter/HealthComponent`中，有关自动回复的参数

   <img src="AssetMarkdown/image-20230212125403134.png" alt="image-20230212125403134" style="zoom:80%;" />

# 二、