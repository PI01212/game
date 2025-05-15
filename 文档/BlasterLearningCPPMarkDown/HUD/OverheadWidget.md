# OverheadWidget

角色头部显示小组件，用来显示玩家在网络中的属性，有三种，权威角色，模拟代理角色，自主代理橘色。后续可改成显示玩家Steam的ID。

## OverheadWidget.h

```cpp
UCLASS()
class BLASTERLEARING_API UOverheadWidget : public UUserWidget
{
 GENERATED_BODY()
 
public:
 UPROPERTY(meta = (BindWidget))
 class UTextBlock* DisplayText;

 void SetDisplayText(FString TextToDisplay);

 UFUNCTION(BlueprintCallable)
 void ShowPlayerNetRole(APawn* InPawn);

protected:
 //关卡从世界移除时调用销毁
 virtual void NativeDestruct() override;
};
```

## OverheadWidget.cpp

### ShowPlayerNetRole

```cpp
//展示玩家网络规则。
void UOverheadWidget::ShowPlayerNetRole(APawn* InPawn)
{
    //从传入的角色获得角色所属的规则属性。创建规则字符串。
 ENetRole RemoteRole = InPawn->GetRemoteRole();
 FString Role;
 switch (RemoteRole)
 {
    //权威。
 case ENetRole::ROLE_Authority:
  Role = FString("Authority");
  break;
  //自主代理。
 case ENetRole::ROLE_AutonomousProxy:
  Role = FString("Autonomous Proxy");
  break;
  //模拟代理。
 case ENetRole::ROLE_SimulatedProxy:
  Role = FString("Simulated Proxy");
  break;
 case ENetRole::ROLE_None:
  Role = FString("None");
  break;
 }
 //创建字符串拼接。调用设置显示文本方法。
 FString RemoteRoleString = FString::Printf(TEXT("Remote Role: %s"), *Role);
 SetDisplayText(RemoteRoleString);
}
```

### SetDisplayText

```cpp
//设置显示文本。
void UOverheadWidget::SetDisplayText(FString TextToDisplay)
{
 if (DisplayText)
 {
    //设置小部件显示的文本。
  DisplayText->SetText(FText::FromString(TextToDisplay));
 }
}
```

### NativeDestruct

```cpp
//关卡从世界移除时调用销毁
void UOverheadWidget::NativeDestruct()
{
    //从父容器移除，也就是从屏幕移除。再调用父类销毁逻辑。
 RemoveFromParent();
 Super::NativeDestruct();
}
```
