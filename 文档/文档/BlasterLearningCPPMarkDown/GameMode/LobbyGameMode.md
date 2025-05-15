# LobbyGameMode

大厅游戏模式，当玩家加入先进入大厅，等人数达到设置的连接数就会调用服务器跳转地图。

## LobbyGameMode.h

```cpp
UCLASS()
class BLASTERLEARING_API ALobbyGameMode : public AGameMode
{
 GENERATED_BODY()

public:
 virtual void PostLogin(APlayerController* NewPlayer) override;
};
```

## LobbyGameMode.cpp

### PostLogin

```cpp
//登录完成后。
void ALobbyGameMode::PostLogin(APlayerController* NewPlayer)
{
 Super::PostLogin(NewPlayer);

//从游戏状态中获得玩家个数。
 int32 NumberOfPlayer = GameState.Get()->PlayerArray.Num();

//获得游戏实例。
 UGameInstance* GameInstance = GetGameInstance();
 if (GameInstance)
 {
    //从游戏实例中获得在线会话子系统。检查是否为空，为空触发断言。
  UMultiplayerSessionsSubsystem* Subsystem = GameInstance->GetSubsystem<UMultiplayerSessionsSubsystem>();
  check(Subsystem);

//如果现在的玩家数量等于子系统中设置的连接数量。代表人数足够。
  if (NumberOfPlayer == Subsystem->DesiredNumPublicConnections)
  {
   UWorld* World = GetWorld();
   if (World)
   {
    //启用无缝切换。
    bUseSeamlessTravel = true;

//从子系统中获得比赛模式。
    FString MatchType = Subsystem->DesiredMatchType;
    //如果是个人模式。
    if (MatchType == "FreeForAll")
    {
        //跳跃到BlasterMap地图。
     World->ServerTravel(FString("/Game/Maps/BlasterMap?listen"));
    }
    //如果是团队模式。
    else if (MatchType == "Teams")
    {
        //跳跃到团队地图。
     World->ServerTravel(FString("/Game/Maps/Teams?listen"));
    }
    //如果是夺旗模式。
    else if (MatchType == "CaptureTheFlag")
    {
        //跳跃到夺旗模式。
     World->ServerTravel(FString("/Game/Maps/CaptureTheFlag?listen"));
    }
   }
  }
 }
}
```
