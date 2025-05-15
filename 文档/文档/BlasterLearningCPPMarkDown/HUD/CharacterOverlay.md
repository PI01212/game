# CharacterOverlay

玩家游戏战斗中的主要HUD，显示关于玩家血量，护盾，弹药，击杀和死亡，网络,游戏时间,手雷等主要战斗HUD。

## CharacterOverlay.h

```cpp
UCLASS()
class BLASTERLEARING_API UCharacterOverlay : public UUserWidget
{
 GENERATED_BODY()
 
public:
 UPROPERTY(meta = (BindWidget))
 class UProgressBar* HealthBar;

 UPROPERTY(meta = (BindWidget))
 class UTextBlock* HealthText;

 UPROPERTY(meta = (BindWidget))
 UProgressBar* ShieldBar;

 UPROPERTY(meta = (BindWidget))
 UTextBlock* ShieldText;

 UPROPERTY(meta = (BindWidget))
 UTextBlock* ScoreAmount;

 UPROPERTY(meta = (BindWidget))
 UTextBlock* RedTeamScore;

 UPROPERTY(meta = (BindWidget))
 UTextBlock* BlueTeamScore;

 UPROPERTY(meta = (BindWidget))
 UTextBlock* ScoreSpacerText;

 UPROPERTY(meta = (BindWidget))
 UTextBlock* DefeatsAmount;

 UPROPERTY(meta = (BindWidget))
 UTextBlock* WeaponAmmoAmount;

 UPROPERTY(meta = (BindWidget))
 UTextBlock* CarriedAmmoAmount;

 UPROPERTY(meta = (BindWidget))
 UTextBlock* MatchCountdownText;

 UPROPERTY(meta = (BindWidget))
 UTextBlock* GrenadesText;

 UPROPERTY(meta = (BindWidget))
 class UImage* HighPingImage;

 UPROPERTY(meta = (BindWidgetAnim), Transient)
 UWidgetAnimation* HighPingAnimation;
};
```
