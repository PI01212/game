# TeamsGameMode

团队游戏模式。

## TeamsGameMode.h

```cpp
UCLASS()
class BLASTERLEARING_API ATeamsGameMode : public ABlasterGameMode
{
 GENERATED_BODY()
 
public:
 ATeamsGameMode();
 virtual void PostLogin(APlayerController* NewPlayer) override;
 virtual void Logout(AController* Exiting) override;
 virtual float CalculateDamage(AController* Attacker, AController* Victim, float BaseDamage) override;
 virtual void PlayerEliminated(class ABlasterCharacter* ElimmedCharacter, class ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController);

protected:
 virtual void HandleMatchHasStarted() override;
};
```

## TeamsGameMode.cpp

### 构造函数

```cpp
ATeamsGameMode::ATeamsGameMode()
{
    //团队模式设置为true。
 bTeamsMatch = true;
}
```

### PostLogin

```cpp
void ATeamsGameMode::PostLogin(APlayerController* NewPlayer)
{
 Super::PostLogin(NewPlayer);

//获得游戏状态。
 ABlasterGameState* BGameState = Cast<ABlasterGameState>(UGameplayStatics::GetGameState(this));
 if (BGameState)
 {
    //获得新加入的玩家的游戏状态。
  ABlasterPlayerState* BPState = NewPlayer->GetPlayerState<ABlasterPlayerState>();
  //如果玩家状态的团队等于没有团队。
  if (BPState && BPState->GetTeam() == ETeam::ET_NoTeam)
  {
    //如果游戏状态中的蓝队个数大于红队。
   if (BGameState->BlueTeam.Num() >= BGameState->RedTeam.Num())
   {
    //游戏状态的红队中加入这个新的玩家。设置这个玩家的团队为红队。
    BGameState->RedTeam.AddUnique(BPState);
    BPState->SetTeam(ETeam::ET_RedTeam);
   }
   else
   {
    //相反则设置为蓝队。
    BGameState->BlueTeam.AddUnique(BPState);
    BPState->SetTeam(ETeam::ET_BlueTeam);
   }
  }
 }
}
```

### Logout

```cpp
//登出。
void ATeamsGameMode::Logout(AController* Exiting)
{
 Super::Logout(Exiting);

 ABlasterGameState* BGameState = Cast<ABlasterGameState>(UGameplayStatics::GetGameState(this));
 ABlasterPlayerState* BPState = Exiting->GetPlayerState<ABlasterPlayerState>();
 if (BGameState && BPState)
 {
    //如果游戏状态中的红队中有离开的玩家。
  if (BGameState->RedTeam.Contains(BPState))
  {
    //从红队中移除。
   BGameState->RedTeam.Remove(BPState);
  }
  //蓝队也是同理。
  if (BGameState->BlueTeam.Contains(BPState))
  {
   BGameState->BlueTeam.Remove(BPState);
  }
 }
}
```

### CalculateDamage

```cpp
float ATeamsGameMode::CalculateDamage(AController* Attacker, AController* Victim, float BaseDamage)
{
 ABlasterPlayerState* AttackerPState = Attacker->GetPlayerState<ABlasterPlayerState>();
 ABlasterPlayerState* VictimPState = Victim->GetPlayerState<ABlasterPlayerState>();
 //如果攻击者玩家状态为空或者被攻击者玩家状态为空直接返回基础伤害。
 if (AttackerPState == nullptr || VictimPState == nullptr) return BaseDamage;
 //如果攻击者等于被攻击者。
 if (VictimPState == AttackerPState)
 {
    //直接返回基础伤害。
  return BaseDamage;
 }
 //如果攻击者和被攻击者是一个团队。
 if (AttackerPState->GetTeam() == VictimPState->GetTeam())
 {
    //返回伤害为0。
  return 0.f;
 }
 //其他情况直接返回基础伤害。
 return BaseDamage;
}
```

###

```cpp
//比赛开始之后。
void ATeamsGameMode::HandleMatchHasStarted()
{
 Super::HandleMatchHasStarted();

//获得游戏状态。
 ABlasterGameState* BGameState = Cast<ABlasterGameState>(UGameplayStatics::GetGameState(this));
 if (BGameState)
 {
    //循环游戏状态中的玩家状态数组。
  for (auto PState : BGameState->PlayerArray)
  {
    //获得玩家状态。
   ABlasterPlayerState* BPState = Cast< ABlasterPlayerState>(PState.Get());
   //如果玩家状态等于没有团队。
   if (BPState && BPState->GetTeam() == ETeam::ET_NoTeam)
   {
    //如果蓝队大于红队。
    if (BGameState->BlueTeam.Num() >= BGameState->RedTeam.Num())
    {
        //就设置玩家到红队。
     BGameState->RedTeam.AddUnique(BPState);
     BPState->SetTeam(ETeam::ET_RedTeam);
    }
    else
    {
        //相反则设置为蓝队。
     BGameState->BlueTeam.AddUnique(BPState);
     BPState->SetTeam(ETeam::ET_BlueTeam);
    }
   }
  }
 }
}
```

### PlayerEliminated

```cpp
//玩家被淘汰。
void ATeamsGameMode::PlayerEliminated(ABlasterCharacter* ElimmedCharacter, ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController)
{
 Super::PlayerEliminated(ElimmedCharacter, VictimController, AttackerController);

 ABlasterGameState* BGameState = Cast<ABlasterGameState>(UGameplayStatics::GetGameState(this));
 //获得攻击者的玩家状态。
 ABlasterPlayerState* AttackerPlayerState = AttackerController ? Cast<ABlasterPlayerState>(AttackerController->PlayerState) : nullptr;
 if (BGameState && AttackerPlayerState)
 {
    //如果攻击者是蓝队。
  if (AttackerPlayerState->GetTeam() == ETeam::ET_BlueTeam)
  {
    //蓝队得分。
   BGameState->BlueTeamScores();
  }
  //如果是红队。
  if (AttackerPlayerState->GetTeam() == ETeam::ET_RedTeam)
  {
    //红队得分。
   BGameState->RedTeamScores();
  }
 }
}
```
