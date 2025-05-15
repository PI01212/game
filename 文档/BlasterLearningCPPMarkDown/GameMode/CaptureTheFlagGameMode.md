# CaptureTheFlagGameMode

夺旗模式继承自团队模式，因为基本逻辑一致只是得分方式不一致。

## CaptureTheFlagGameMode.h

```cpp
UCLASS()
class BLASTERLEARING_API ACaptureTheFlagGameMode : public ATeamsGameMode
{
 GENERATED_BODY()
public:
 virtual void PlayerEliminated(class ABlasterCharacter* ElimmedCharacter, class ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController) override;
 void FlagCaptured(class AFlag* Flag, class AFlagZone* Zone);
};
```

## CaptureTheFlagGameMode.cpp

###

```cpp
//玩家被淘汰。
void ACaptureTheFlagGameMode::PlayerEliminated(class ABlasterCharacter* ElimmedCharacter, class ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController)
{
    //调用基础游戏模式的玩家淘汰。因为夺旗模式的淘汰方式和基础淘汰方式一致而不是父类团队模式所以指明使用的是基础游戏模式。
 ABlasterGameMode::PlayerEliminated(ElimmedCharacter, VictimController, AttackerController);
}
```

### FlagCaptured

```cpp
//旗子被夺取。
void ACaptureTheFlagGameMode::FlagCaptured(AFlag* Flag, AFlagZone* Zone)
{
    //如果重叠的旗子和重叠区域不相等。代表抢夺成功。
 bool bValidCapture = Flag->GetTeam() != Zone->Team;
 //获得游戏状态。
 ABlasterGameState* BGameState = Cast<ABlasterGameState>(GameState);
 if (BGameState)
 {
    //如果重叠区域是蓝队。
  if (Zone->Team == ETeam::ET_BlueTeam)
  {
    //蓝队得分。
   BGameState->BlueTeamScores();
  }
  //如果是红队。
  if (Zone->Team == ETeam::ET_RedTeam)
  {
    //红队得分。
   BGameState->RedTeamScores();
  }
 }
}
```
