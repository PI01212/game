# BlasterGameState

用来存储游戏状态，比如当前哪个玩家分数最高，红蓝队伍的分数。

## BlasterGameState.h

```cpp
UCLASS()
class BLASTERLEARING_API ABlasterGameState : public AGameState
{
 GENERATED_BODY()
 
public:
 virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
 void UpdateTopScore(class ABlasterPlayerState* ScoringPlayer);

 UPROPERTY(Replicated)
 TArray<class ABlasterPlayerState*> TopScoringPlayers;

 /*
 * Teams
 */
 void RedTeamScores();
 void BlueTeamScores();

 TArray<ABlasterPlayerState*> RedTeam;
 TArray<ABlasterPlayerState*> BlueTeam;

 UPROPERTY(ReplicatedUsing = OnRep_RedTeamScore)
 float RedTeamScore = 0.f;

 UPROPERTY(ReplicatedUsing = OnRep_BlueTeamScore)
 float BlueTeamScore = 0.f;

 UFUNCTION()
 void OnRep_RedTeamScore();

 UFUNCTION()
 void OnRep_BlueTeamScore();

private:
 float TopScore = 0.f;
};
```

## BlasterGameState.cpp

### GetLifetimeReplicatedProps

```cpp
//网络复制参数的注册。
void ABlasterGameState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
 Super::GetLifetimeReplicatedProps(OutLifetimeProps);

 DOREPLIFETIME(ABlasterGameState, TopScoringPlayers);
 DOREPLIFETIME(ABlasterGameState, RedTeamScore);
 DOREPLIFETIME(ABlasterGameState, BlueTeamScore);
}
```

### UpdateTopScore

```cpp
//更新最高分数。
void ABlasterGameState::UpdateTopScore(ABlasterPlayerState* ScoringPlayer)
{
    //如果最高分数玩家的数组为0。
 if (TopScoringPlayers.Num() == 0)
 {
    //把传入的玩家状态添加到数组中。最高分数等于传入的玩家状态的分数。
  TopScoringPlayers.Add(ScoringPlayer);
  TopScore = ScoringPlayer->GetScore();
 }
 //如果不为0，把传入的玩家状态的分数和最高分数作比较。
 else if (ScoringPlayer->GetScore() == TopScore)
 {
    //最高分数玩家数组中添加传入的玩家。
  TopScoringPlayers.AddUnique(ScoringPlayer);
 }
 //如果传入的玩家分数比最高分数还高。
 else if (ScoringPlayer->GetScore() > TopScore)
 {
    //代表现存的最高玩家都不再是最高玩家，把最高玩家数组清空。添加传入的玩家状态。最高分数等于传入玩家的分数。
  TopScoringPlayers.Empty();
  TopScoringPlayers.AddUnique(ScoringPlayer);
  TopScore = ScoringPlayer->GetScore();
 }
}
```

### RedTeamScores

```cpp
//红队得分。
void ABlasterGameState::RedTeamScores()
{
    //红队分数+1。
 ++RedTeamScore;
 //从世界中获得第一个玩家控制器也就是当前玩家控制的角色控制器。
 ABlasterPlayerController* BPlayer = Cast<ABlasterPlayerController>(GetWorld()->GetFirstPlayerController());
 if (BPlayer)
 {
    //调用设置红队分数HUD。
  BPlayer->SetHUDRedTeamScore(RedTeamScore);
 }
}
```

### BlueTeamScores

```cpp
//蓝队得分。
void ABlasterGameState::BlueTeamScores()
{
    //蓝队分数+1。
 ++BlueTeamScore;
 //从世界中获得第一个玩家控制器也就是当前玩家控制的角色控制器。
 ABlasterPlayerController* BPlayer = Cast<ABlasterPlayerController>(GetWorld()->GetFirstPlayerController());
 if (BPlayer)
 {
    //调用设置蓝队分数HUD。
  BPlayer->SetHUDBlueTeamScore(BlueTeamScore);
 }
}
```

### OnRep_RedTeamScore

```cpp
//红队分数复制通知。
void ABlasterGameState::OnRep_RedTeamScore()
{
    //从世界中获得第一个玩家控制器也就是当前玩家控制的角色控制器。
 ABlasterPlayerController* BPlayer = Cast<ABlasterPlayerController>(GetWorld()->GetFirstPlayerController());
 if (BPlayer)
 {
    //调用设置红队分数HUD。
  BPlayer->SetHUDRedTeamScore(RedTeamScore);
 }
}
```

### OnRep_BlueTeamScore

```cpp
//蓝队分数复制通知。
void ABlasterGameState::OnRep_BlueTeamScore()
{
    //从世界中获得第一个玩家控制器也就是当前玩家控制的角色控制器。
 ABlasterPlayerController* BPlayer = Cast<ABlasterPlayerController>(GetWorld()->GetFirstPlayerController());
 if (BPlayer)
 {
    //调用设置蓝队分数HUD。
  BPlayer->SetHUDBlueTeamScore(BlueTeamScore);
 }
}
```
