# BlasterGameMode

基础的游戏模式，团队继承自Blaster游戏模式。

## BlasterGameMode.h

```cpp
namespace MatchState
{
 extern BLASTERLEARING_API const FName Cooldown; //比赛结束，显示赢家并开始冷却倒计时
}

UCLASS()
class BLASTERLEARING_API ABlasterGameMode : public AGameMode
{
 GENERATED_BODY()
 
public:
 ABlasterGameMode();
 virtual void Tick(float DeltaTime) override;
 virtual void PlayerEliminated(class ABlasterCharacter* ElimmedCharacter, class ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController);
 virtual void RequestRespawn(ACharacter* ElimmedCharacter, AController* ElimmedController);
 void PlayerLeftGame(class ABlasterPlayerState* PlayerLeaving);
 virtual float CalculateDamage(AController* Attacker, AController* Victim, float BaseDamage);

 UPROPERTY(EditDefaultsOnly)
 float WarmupTime = 10.f;

 UPROPERTY(EditDefaultsOnly)
 float MatchTime = 120.f;

 UPROPERTY(EditDefaultsOnly)
 float CooldownTime = 10.f;

 float LevelStartingTime = 0.f;

 bool bTeamsMatch = false;

protected:
 virtual void BeginPlay() override;
 virtual void OnMatchStateSet() override;

private:
 float CountdownTime = 0.f;

public:
 FORCEINLINE float GetCountdownTime() const { return CountdownTime; }
 FORCEINLINE bool IsTeamsMatch() const { return bTeamsMatch; }
};
```

## BlasterGameMode.cpp

### MatchState

```cpp
namespace MatchState
{
    //设置自定义的比赛状态冷却。
 const FName Cooldown = FName("Cooldown");
}
```

### 构造函数

```cpp
ABlasterGameMode::ABlasterGameMode()
{
 bDelayedStart = true;
}
```

### BeginPlay

```cpp
void ABlasterGameMode::BeginPlay()
{
 Super::BeginPlay();

//关卡开始时间等于开始游戏方法调用时的时间。
 LevelStartingTime = GetWorld()->GetTimeSeconds();
}
```

### Tick

```cpp
void ABlasterGameMode::Tick(float DeltaTime)
{
 Super::Tick(DeltaTime);

//如果比赛状态等于等待开始。
 if (MatchState == MatchState::WaitingToStart)
 {
    //倒计时时间等于热身时间减去世界运行的时间加上关卡开始的时间。
  CountdownTime = WarmupTime - GetWorld()->GetTimeSeconds() + LevelStartingTime;
  //如果倒计时时间小于等于0。
  if (CountdownTime <= 0.f)
  {
    //开始比赛。
   StartMatch();
  }
 }
 //如果比赛状态等于进行中。
 else if (MatchState == MatchState::InProgress)
 {
    //倒计时等于热身时间加上比赛时间减去世界运行的时间加上关卡开始的时间。
  CountdownTime = WarmupTime + MatchTime - GetWorld()->GetTimeSeconds() + LevelStartingTime;
  //如果倒计时时间小于等于0。
  if (CountdownTime <= 0.f)
  {
    //设置比赛状态为结束冷却。
   SetMatchState(MatchState::Cooldown);
  }
 }
 //如果比赛状态等于结束冷却。
 else if (MatchState == MatchState::Cooldown)
 {
    //倒计时时间等于冷却时间加上热身时间加上比赛时间减去世界运行的时间加上关卡开始的时间。
  CountdownTime = CooldownTime + WarmupTime + MatchTime - GetWorld()->GetTimeSeconds() + LevelStartingTime;
  //如果倒计时时间小于等于0。
  if (CountdownTime <= 0.f)
  {
    //调用重新开始游戏。
   RestartGame();
  }
 }
}
```

### OnMatchStateSet

```cpp
//在比赛状态变化后设置
void ABlasterGameMode::OnMatchStateSet()
{
 Super::OnMatchStateSet();

//迭代器迭代世界中的玩家控制器。
 for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
 {
  ABlasterPlayerController* BlasterPlayer = Cast<ABlasterPlayerController>(*It);
  if (BlasterPlayer)
  {
    //调用控制器的设置比赛状态。
   BlasterPlayer->OnMatchStateSet(MatchState, bTeamsMatch);
  }
 }
}
```

### CalculateDamage

```cpp
//计算游戏中的伤害。现阶段只是简单返回伤害，但可以对其进行扩展，比如可以根据游戏模式来改变实际造成的伤害。
float ABlasterGameMode::CalculateDamage(AController* Attacker, AController* Victim, float BaseDamage)
{
    //返回基础伤害。
 return BaseDamage;
}
```

### PlayerEliminated

```cpp
//玩家淘汰。
void ABlasterGameMode::PlayerEliminated(ABlasterCharacter* ElimmedCharacter, ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController)
{
    //获得攻击者和被攻击者的控制器。
 ABlasterPlayerState* AttackerPlayerState = AttackerController ? Cast<ABlasterPlayerState>(AttackerController->PlayerState) : nullptr;
 ABlasterPlayerState* VictimPlayerState = VictimController ? Cast<ABlasterPlayerState>(VictimController->PlayerState) : nullptr;

//获得游戏状态。
 ABlasterGameState* BlasterGameState = GetGameState<ABlasterGameState>();

//如果攻击者不等于被攻击者并且游戏状态存在。
 if (AttackerPlayerState && AttackerPlayerState != VictimPlayerState && BlasterGameState)
 {
    //创建当前领先玩家的状态数组。
  TArray<ABlasterPlayerState*> PlayersCurrentlyInTheLead;
  //循环游戏状态中的得分最高的玩家数组。
  for (auto LeadPlayer : BlasterGameState->TopScoringPlayers)
  {
    //添加得分最高的玩家到领先玩家的数组中。
   PlayersCurrentlyInTheLead.Add(LeadPlayer);
  }
  //攻击者玩家状态得1分。
  AttackerPlayerState->AddToScore(1.f);
  //调用游戏状态的更新最高得分。
  BlasterGameState->UpdateTopScore(AttackerPlayerState);
  //如果游戏状态中的最高得分数组中有攻击者玩家状态。
  if (BlasterGameState->TopScoringPlayers.Contains(AttackerPlayerState))
  {
    //从攻击者玩家状态获得玩家角色对象。
   ABlasterCharacter* Leader = Cast<ABlasterCharacter>(AttackerPlayerState->GetPawn());
   if (Leader)
   {
    //调用多播RPC给角色带上领先的皇冠。
    Leader->MulticastGainedTheLead();
   }
  }

//循环领先者数组。
  for (INT32 i = 0; i < PlayersCurrentlyInTheLead.Num(); i++)
  {
    //如果最高得分的玩家数组中不再有领先者数组中的成员。
   if (!BlasterGameState->TopScoringPlayers.Contains(PlayersCurrentlyInTheLead[i]))
   {
    //获得这个成员的玩家角色类对象。
    ABlasterCharacter* Loser = Cast<ABlasterCharacter>(PlayersCurrentlyInTheLead[i]->GetPawn());
    if (Loser)
    {
        //调用多播RPC失去皇冠。
     Loser->MulticastLostTheLead();
    }
   }
  }
 }
 if (VictimPlayerState)
 {
    //调用玩家状态的死亡次数加一次。
  VictimPlayerState->AddToDefeats(1);
 }

 if (ElimmedCharacter)
 {
    //调用死亡玩家的死亡方法传入false表示不离开游戏。
  ElimmedCharacter->Elim(false);
 }

//迭代器迭代世界中的玩家控制器。
 for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
 {
    //获得玩家角色控制器。
  ABlasterPlayerController* BlasterPlayer = Cast<ABlasterPlayerController>(*It);
  if (BlasterPlayer && AttackerPlayerState && VictimPlayerState)
  {
    //调用世界中这些玩家角色控制器的广播死亡方法广播谁击杀了谁。
   BlasterPlayer->BroadcastElim(AttackerPlayerState, VictimPlayerState);
  }
 }
}
```

### RequestRespawn

```cpp
void ABlasterGameMode::RequestRespawn(ACharacter* ElimmedCharacter, AController* ElimmedController)
{
 if (ElimmedCharacter)
 {
    //调用死亡的角色的重置和销毁。
  ElimmedCharacter->Reset();
  ElimmedCharacter->Destroy();
 }
 if (ElimmedController)
 {
    //创建玩家开始数组。
  TArray<AActor*> PlayerStarts;
  //获得所有的玩家开始并存入玩家开始数组。
  UGameplayStatics::GetAllActorsOfClass(this, APlayerStart::StaticClass(), PlayerStarts);
  //在玩家开始数组中随机选择一个。玩家重生在选择的玩家开始。
  int32 Selection = FMath::RandRange(0, PlayerStarts.Num() - 1);
  RestartPlayerAtPlayerStart(ElimmedController, PlayerStarts[Selection]);
 }
}
```

### PlayerLeftGame

```cpp
void ABlasterGameMode::PlayerLeftGame(ABlasterPlayerState* PlayerLeaving)
{
 //调用Elim函数，传递参数bLeftGame为真
 if (PlayerLeaving == nullptr) return;
 //获得游戏状态。
 ABlasterGameState* BlasterGameState = GetGameState<ABlasterGameState>();
 //如果最高得分的玩家数组中有这个玩家。
 if (BlasterGameState && BlasterGameState->TopScoringPlayers.Contains(PlayerLeaving))
 {
    //从数组中移除。
  BlasterGameState->TopScoringPlayers.Remove(PlayerLeaving);
 }
 //获得离开玩家的玩家角色。
 ABlasterCharacter* CharacterLeaving = Cast<ABlasterCharacter>(PlayerLeaving->GetPawn());
 if (CharacterLeaving)
 {
    //调用死亡方法并告知离开游戏。
  CharacterLeaving->Elim(true);
 }
}
```
