# ReturnToMainMenu

返回主菜单按钮小部件。按下按钮就会退出游戏返回主菜单。

## ReturnToMainMenu.h

```cpp
UCLASS()
class BLASTERLEARING_API UReturnToMainMenu : public UUserWidget
{
 GENERATED_BODY()
public:
 void MenuSetup();
 void MenuTearDown();

protected:
 virtual bool Initialize() override;

 UFUNCTION()
 void OnDestroySession(bool bWasSuccessful);

 UFUNCTION()
 void OnPlayerLeftGame();

private:
 UPROPERTY(meta = (BindWidget))
 class UButton* ReturnButton;

 UFUNCTION()
 void ReturnButtonClicked();

 UPROPERTY()
 class UMultiplayerSessionsSubsystem* MultiplayerSessionsSubsystem;

 UPROPERTY()
 class APlayerController* PlayerController;
};
```

## ReturnToMainMenu.cpp

### MenuSetup

```cpp
//菜单初始化。
void UReturnToMainMenu::MenuSetup()
{
    //添加到视窗。设置可见性，聚焦。
 AddToViewport();
 SetVisibility(ESlateVisibility::Visible);
 bIsFocusable = true;
 
 //获得世界。
 UWorld* World = GetWorld();
 if (World)
 {
    //获得世界中的第一个控制器也就是玩家控制器。
  PlayerController = PlayerController == nullptr ? World->GetFirstPlayerController() : PlayerController;
  if (PlayerController)
  {
    //创建输入模式设置结构体玩家可以与游戏和UI交互。设置输入焦点为当前小部件的根。
   FInputModeGameAndUI InputModeData;
   InputModeData.SetWidgetToFocus(TakeWidget());
   //设置输入模式，设置显示鼠标。
   PlayerController->SetInputMode(InputModeData);
   PlayerController->SetShowMouseCursor(true);
  }
 }

//如果返回按钮还没有绑定。
 if (ReturnButton && !ReturnButton->OnClicked.IsBound())
 {
    //动态绑定回调函数ReturnButtonClicked。
  ReturnButton->OnClicked.AddDynamic(this, &UReturnToMainMenu::ReturnButtonClicked);
 }

//获取游戏实例。
 UGameInstance* GameInstance = GetGameInstance();
 if (GameInstance)
 {
    //从游戏实例中获得多人会话子系统。
  MultiplayerSessionsSubsystem = GameInstance->GetSubsystem<UMultiplayerSessionsSubsystem>();
  if (MultiplayerSessionsSubsystem)
  {
    //为自定义的多人会话销毁完成委托动态绑定回调函数OnDestroySession。
   MultiplayerSessionsSubsystem->MultiplayerOnDestorySessionComplete.AddDynamic(this, &UReturnToMainMenu::OnDestroySession);
   
  }
 }
}
```

### MenuTearDown

```cpp
//菜单清理。
void UReturnToMainMenu::MenuTearDown()
{
    //获得世界。
 UWorld* World = GetWorld();
 if (World)
 {
    //获得玩家控制器。
  PlayerController = PlayerController == nullptr ? World->GetFirstPlayerController() : PlayerController;
  if (PlayerController)
  {
    //创建输入模式玩家只能与游戏交互。设置输入模式，隐藏鼠标。
   FInputModeGameOnly InputModeData;
   PlayerController->SetInputMode(InputModeData);
   PlayerController->SetShowMouseCursor(false);
  }
 }
 //如果按钮已经绑定了回调函数。
 if (ReturnButton && ReturnButton->OnClicked.IsBound())
 {
    //移除动态绑定的回调函数。
  ReturnButton->OnClicked.RemoveDynamic(this, &UReturnToMainMenu::ReturnButtonClicked);
 }
 //如果子系统的自定义委托的回调函数已经绑定，移除动态绑定的回调函数。
 if (MultiplayerSessionsSubsystem && MultiplayerSessionsSubsystem->MultiplayerOnDestorySessionComplete.IsBound())
 {
  MultiplayerSessionsSubsystem->MultiplayerOnDestorySessionComplete.RemoveDynamic(this, &UReturnToMainMenu::OnDestroySession);
 }
 //从父组件移除小组件。
 RemoveFromParent();
}
```

### Initialize

```cpp
//初始化小部件。
bool UReturnToMainMenu::Initialize()
{
    //如果初始化小部件失败。
 if (!Super::Initialize())
 {
    //返回false。
  return false;
 }
 //否则返回true。
 return true;
}
```

### OnDestroySession

```cpp
//在会话销毁完成之后的回调函数。
void UReturnToMainMenu::OnDestroySession(bool bWasSuccessful)
{
    //如果没有成功。
 if (!bWasSuccessful)
 {
    //设置按钮可以按下。
  ReturnButton->SetIsEnabled(true);
  return;
 }
 //成功了就获得世界。
 UWorld* World = GetWorld();
 if (World)
 {
    //从世界中获得游戏模式。
  AGameModeBase* GameMode = World->GetAuthGameMode<AGameModeBase>();
  if (GameMode)
  {
    //获取到游戏模式代表是服务器，调用游戏模式的主机返回主菜单。
   GameMode->ReturnToMainMenuHost();
  }
  else
  {
    //否则就是客户端，从世界中获得玩家控制器。调用客户端的返回主菜单原因传入空文本。
   PlayerController = PlayerController == nullptr ? World->GetFirstPlayerController() : PlayerController;
   if (PlayerController)
   {
    PlayerController->ClientReturnToMainMenuWithTextReason(FText());
   }
  }
 }
}
```

### ReturnButtonClicked

```cpp
//返回菜单按钮的的回调函数。
void UReturnToMainMenu::ReturnButtonClicked()
{
    //设置按钮不可按下。
 ReturnButton->SetIsEnabled(false);
 
 UWorld* World = GetWorld();
 if (World)
 {
    //获得玩家控制器。
  APlayerController* FirstPlayerController = World->GetFirstPlayerController();
  if (FirstPlayerController)
  {
    //从控制器获得玩家角色。
   ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(FirstPlayerController->GetPawn());
   if (BlasterCharacter)
   {
    //调用角色的离开游戏RPC。绑定自定义动态多播委托离开游戏的回调函数OnPlayerLeftGame。
    BlasterCharacter->ServerLeaveGame();
    BlasterCharacter->OnLeftGame.AddDynamic(this, &UReturnToMainMenu::OnPlayerLeftGame);
   }
   else
   {
    //否则设置按钮可按下。
    ReturnButton->SetIsEnabled(true);
   }
  }
 }
}
```

### OnPlayerLeftGame

```cpp
//在玩家离开游戏的回到函数。
void UReturnToMainMenu::OnPlayerLeftGame()
{
 if (MultiplayerSessionsSubsystem)
 {
    //调用子系统的销毁会话。
  MultiplayerSessionsSubsystem->DestorySession();
 }
}
```
