# ElimAnnouncement

玩家死亡时全局播放击杀通知，告知所有玩家击杀者和被击杀者。

## ElimAnnouncement.h

```cpp
UCLASS()
class BLASTERLEARING_API UElimAnnouncement : public UUserWidget
{
 GENERATED_BODY()
 
public:
 void SetElimAnnouncementText(FString AttackerName, FString VictimName);

 UPROPERTY(meta = (BindWidget))
 class UHorizontalBox* AnnouncementBox;

 UPROPERTY(meta = (BindWidget))
 class UTextBlock* AnnouncementText;
};
```

## ElimAnnouncement.cpp

### SetElimAnnouncementText

```cpp
void UElimAnnouncement::SetElimAnnouncementText(FString AttackerName, FString VictimName)
{
    //创建死亡通知文本，修改格式为传入的玩家名字和击杀信息。
 FString ElimAnnouncementText = FString::Printf(TEXT("%s elimmed %s!"), *AttackerName, *VictimName);
 if (AnnouncementText)
 {
    //设置小部件绑定的文本。
  AnnouncementText->SetText(FText::FromString(ElimAnnouncementText));
 }
}
```
