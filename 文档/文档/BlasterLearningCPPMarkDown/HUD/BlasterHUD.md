# BlasterHUD

一个用于管理和显示游戏 UI 的类,绘制准星，显示角色状态，显示公告，管理UI生命周期。就是对所有显示的小组件的统一管理。

## BlasterHUD.h

```cpp
USTRUCT(BlueprintType)
struct FHUDPackage
{
 GENERATED_BODY()
public:
 class UTexture2D* CrosshairsCenter;
 UTexture2D* CrosshairsLeft;
 UTexture2D* CrosshairsRight;
 UTexture2D* CrosshairsTop;
 UTexture2D* CrosshairsBottom;
 float CrosshairSpread;
 FLinearColor CrosshairsColor;
};

/**
 * 
 */
UCLASS()
class BLASTERLEARING_API ABlasterHUD : public AHUD
{
 GENERATED_BODY()
public:
 virtual void DrawHUD() override;

 UPROPERTY(EditAnywhere, Category = "Player States")
 TSubclassOf<class UUserWidget> CharacterOverlayClass;

 void AddCharacterOverlay();

 UPROPERTY()
 class UCharacterOverlay* CharacterOverlay;

 UPROPERTY(EditAnywhere, Category = "Announcements")
 TSubclassOf<UUserWidget> AnnouncementClass;

 UPROPERTY()
 class UAnnouncement* Announcement;

 void AddAnnouncement();
 void AddElimAnnouncement(FString Attacker, FString Victim);

protected:
 virtual void BeginPlay() override;
private:
 UPROPERTY()
 class APlayerController* OwningPlayer;

 FHUDPackage HUDPackage;

 void DrawCrosshair(UTexture2D* Texture, FVector2D ViewportCenter, FVector2D Spread, FLinearColor CorsshairColor);

 UPROPERTY(EditAnywhere)
 float CrosshairSpreadMax = 16.f;

 UPROPERTY(EditAnywhere)
 TSubclassOf<class UElimAnnouncement> ElimAnnouncementClass;

 UPROPERTY(EditAnywhere)
 float ElimAnnouncementTime = 2.5f;

 UFUNCTION()
 void ElimAnnouncementTimerFinished(UElimAnnouncement* MsgToRemove);

 UPROPERTY()
 TArray<UElimAnnouncement*> ElimMessages;

public:
 FORCEINLINE void SetHUDPackage(const FHUDPackage& Package) { HUDPackage = Package; }
};
```

## BlasterHUD.cpp

### BeginPlay

```cpp
void ABlasterHUD::BeginPlay()
{
 Super::BeginPlay();
}
```

### AddCharacterOverlay

```cpp
//添加角色状态覆盖界面。
void ABlasterHUD::AddCharacterOverlay()
{
    //获得所有者的控制器。
 APlayerController* PlayerController = GetOwningPlayerController();
 if (PlayerController && CharacterOverlayClass)
 {
    //创建角色状态小组件，添加到视窗。
  CharacterOverlay = CreateWidget<UCharacterOverlay>(PlayerController, CharacterOverlayClass);
  CharacterOverlay->AddToViewport();
 }
}
```

### AddAnnouncement

```cpp
//添加通知小组件。
void ABlasterHUD::AddAnnouncement()
{
 APlayerController* PlayerController = GetOwningPlayerController();
 if (PlayerController && AnnouncementClass)
 {
  Announcement = CreateWidget<UAnnouncement>(PlayerController, AnnouncementClass);
  Announcement->AddToViewport();
 }
}
```

### AddElimAnnouncement

```cpp
//添加死亡通知。
void ABlasterHUD::AddElimAnnouncement(FString Attacker, FString Victim)
{
    //获得所有者玩家控制器。
 OwningPlayer = OwningPlayer == nullptr ? GetOwningPlayerController() : OwningPlayer;
 if (OwningPlayer && ElimAnnouncementClass)
 {
    //创建死亡通知小组件。
  UElimAnnouncement* ElimAnouncementWidget = CreateWidget<UElimAnnouncement>(OwningPlayer, ElimAnnouncementClass);
  if (ElimAnouncementWidget)
  {
    //设置小组件的文本攻击者和被攻击者。添加到视窗。
   ElimAnouncementWidget->SetElimAnnouncementText(Attacker, Victim);
   ElimAnouncementWidget->AddToViewport();

//循环已有的死亡信息数组的通知，防止和新通知重叠。
   for (UElimAnnouncement* Msg : ElimMessages)
   {
    //如果播报死亡通知的盒子存在。
    if (Msg && Msg->AnnouncementBox)
    {
        //将Msg->AnnouncementBox转化为CanvasSlot。
     UCanvasPanelSlot* CanvasSlot = UWidgetLayoutLibrary::SlotAsCanvasSlot(Msg->AnnouncementBox);
     if (CanvasSlot)
     {
        //获得转换后的位置。并将位置按照CanvasSlot的大小上移Y个单位。
      FVector2D Position = CanvasSlot->GetPosition();
      FVector2D NewPosition(
       CanvasSlot->GetPosition().X,
       Position.Y - CanvasSlot->GetSize().Y
      );
      //设置新的位置。
      CanvasSlot->SetPosition(NewPosition);
     }
    }
   }

//添加创建的新的死亡通知。
   ElimMessages.Add(ElimAnouncementWidget);

//创建定时器句柄和委托。
   FTimerHandle ElimMsgTimer;
   FTimerDelegate ElimMsgDelegate;
   //为定时器的委托绑定回调函数。传递的参数就是ElimAnouncementWidget。
   ElimMsgDelegate.BindUFunction(this, FName("ElimAnnouncementTimerFinished"), ElimAnouncementWidget);
   GetWorldTimerManager().SetTimer(
    ElimMsgTimer,
    ElimMsgDelegate,
    ElimAnnouncementTime,
    false
   );
  }
 }
}
```

### ElimAnnouncementTimerFinished

```cpp
//定时器的委托的回调函数。
void ABlasterHUD::ElimAnnouncementTimerFinished(UElimAnnouncement* MsgToRemove)
{
 if (MsgToRemove)
 {
    //把通知移除。
  MsgToRemove->RemoveFromParent();
 }
}
```

### DrawHUD

```cpp
//绘制HUD。
void ABlasterHUD::DrawHUD()
{
 Super::DrawHUD();

//创建屏幕尺寸。
 FVector2D ViewportSize;
 if (GEngine)
 {
    //从引擎的游戏视窗中获得视窗的尺寸。
  GEngine->GameViewport->GetViewportSize(ViewportSize);
  //准星的中心位置就等于窗口视窗的中心。
  const FVector2D ViewportCenter(ViewportSize.X / 2.f, ViewportSize.Y / 2.f);

//系数等于最大系数乘以包中的系数。
  float SpreadScaled = CrosshairSpreadMax * HUDPackage.CrosshairSpread;

  if (HUDPackage.CrosshairsCenter)
  {
    //绘制准星的中心。不使用扩散。
   FVector2D Spread(0.f, 0.f);
   DrawCrosshair(HUDPackage.CrosshairsCenter, ViewportCenter, Spread, HUDPackage.CrosshairsColor);
  }
  if (HUDPackage.CrosshairsLeft)
  {
    //绘制准星的左边，向x轴的负向扩散。
   FVector2D Spread(-SpreadScaled, 0.f);
   DrawCrosshair(HUDPackage.CrosshairsLeft, ViewportCenter, Spread, HUDPackage.CrosshairsColor);
  }
  if (HUDPackage.CrosshairsRight)
  {
    //绘制准星的右边，向x轴的正向扩散。
   FVector2D Spread(SpreadScaled, 0.f);
   DrawCrosshair(HUDPackage.CrosshairsRight, ViewportCenter, Spread, HUDPackage.CrosshairsColor);
  }
  if (HUDPackage.CrosshairsTop)
  {
    //绘制准星的上边，向y轴的负向扩散。
   FVector2D Spread(0.f, -SpreadScaled);
   DrawCrosshair(HUDPackage.CrosshairsTop, ViewportCenter, Spread, HUDPackage.CrosshairsColor);
  }
  if (HUDPackage.CrosshairsBottom)
  {
    //绘制准星的下边，向y轴的正向扩散。
   FVector2D Spread(0.f, SpreadScaled);
   DrawCrosshair(HUDPackage.CrosshairsBottom, ViewportCenter, Spread, HUDPackage.CrosshairsColor);
  }
 }
}
```

### DrawCrosshair

```cpp
//绘制准星。
void ABlasterHUD::DrawCrosshair(UTexture2D* Texture, FVector2D ViewportCenter, FVector2D Spread, FLinearColor CrosshairColor)
{
    //获得纹理的尺寸。
 const float TextureWidth = Texture->GetSizeX();
 const float TextureHeight = Texture->GetSizeY();
 //通过纹理的尺寸和缩放系数设置纹理绘制在屏幕的位置中心。
 const FVector2D TextureDrawPoint(ViewportCenter.X - (TextureWidth / 2.f) + Spread.X, ViewportCenter.Y - (TextureHeight / 2.f) + Spread.Y);

//在视窗绘制纹理。
 DrawTexture(Texture, TextureDrawPoint.X, TextureDrawPoint.Y, TextureWidth, TextureHeight, 0.f, 0.f, 1.f, 1.f, CrosshairColor);
}
```
