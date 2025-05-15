# FlagZone

夺旗模式需要把旗子运送到指定区域得分。

## FlagZone.h

```cpp
UCLASS()
class BLASTERLEARING_API AFlagZone : public AActor
{
 GENERATED_BODY()
 
public:
 AFlagZone();

 UPROPERTY(EditAnywhere)
 ETeam Team;
protected:
 virtual void BeginPlay() override;

 UFUNCTION()
 virtual void OnSphereOverlap(
  UPrimitiveComponent* OverlappedComponent,
  AActor* OtherActor,
  UPrimitiveComponent* OtherComp,
  int32 OtherBodyIndex,
  bool bFromSweep,
  const FHitResult& SweepResult
 );

private:

 UPROPERTY(EditAnywhere)
 class USphereComponent* ZoneSphere;

public:

};
```

## FlagZone.cpp

### 构造函数

```cpp
AFlagZone::AFlagZone()
{
    //开启Tick。
 PrimaryActorTick.bCanEverTick = false;

//创建区域球体，设置为根组件。
 ZoneSphere = CreateDefaultSubobject<USphereComponent>(TEXT("ZoneSphere"));
 SetRootComponent(ZoneSphere);
}
```

### BeginPlay

```cpp
void AFlagZone::BeginPlay()
{
 Super::BeginPlay();

//动态绑定重叠的回调函数OnSphereOverlap。
 ZoneSphere->OnComponentBeginOverlap.AddDynamic(this, &AFlagZone::OnSphereOverlap);
}
```

### OnSphereOverlap

```cpp
void AFlagZone::OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
    //获得重叠的旗子。
 AFlag* OverlappingFlag = Cast<AFlag>(OtherActor);
 //如果重叠的旗子不等于区域的队伍，代表这是玩家把敌人的旗子拿回，玩家的队伍得分。
 if (OverlappingFlag && OverlappingFlag->GetTeam() != Team)
 {
    //获得游戏模式，游戏模式只能从服务器获取，客户端获取则是nullptr，这也就是为什么在绑定事件时没有进行权威判断，因为客户端不会执行后续步骤。
  ACaptureTheFlagGameMode* GameMode = GetWorld()->GetAuthGameMode<ACaptureTheFlagGameMode>();
  if (GameMode)
  {
    //调用旗子得分，传入重叠的旗子和区域。
   GameMode->FlagCaptured(OverlappingFlag, this);
  }
  //调用重置旗子，把旗子送回开始位置。
  OverlappingFlag->ResetFlag();
 }
}
```
