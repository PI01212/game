# Announcement

用于在游戏开始和结束时在屏幕播放通知。

## Announcement.h

```cpp
UCLASS()
class BLASTERLEARING_API UAnnouncement : public UUserWidget
{
 GENERATED_BODY()
 
public:
 UPROPERTY(meta = (BindWidget))
 class UTextBlock* WarmupTime;

 UPROPERTY(meta = (BindWidget))
 UTextBlock* AnnouncementText;

 UPROPERTY(meta = (BindWidget))
 UTextBlock* InfoText;
};
```
