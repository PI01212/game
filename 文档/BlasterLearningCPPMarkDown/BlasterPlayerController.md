# BlasterPlayerController

## BlasterPlayerController.h

```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FHighPingDelegate, bool, bPingTooHigh);

/**
 * 
 */
UCLASS()
class BLASTERLEARING_API ABlasterPlayerController : public APlayerController
{
 GENERATED_BODY()
public:
 void SetHUDHealth(float Health, float MaxHealth);
 void SetHUDShield(float Shield, float MaxShield);
 void SetHUDScore(float Score);
 void SetHUDDefeats(int32 Defeats);
 void SetHUDWeaponAmmo(int32 Ammo);
 void SetHUDCarriedAmmo(int32 Ammo);
 void SetHUDMatchCountdown(float CountdownTime);
 void SetHUDAnnouncementCountdown(float CountdownTime);
 void SetHUDGrenades(int32 Grenades);
 virtual void OnPossess(APawn* InPawn) override;
 virtual void Tick(float DeltaTime) override; 
 virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
 void HideTeamScores();
 void InitTeamScores();
 void SetHUDRedTeamScore(int32 RedScore);
 void SetHUDBlueTeamScore(int32 BlueScore);

 virtual float GetServerTime(); //与服务器时钟同步
 virtual void ReceivedPlayer() override; //尽快同步服务器时钟
 void OnMatchStateSet(FName State, bool bTeamMatch = false);
 void HandleMatchHasStarted(bool bTeamMatch = false);
 void HandleCooldown();

 float SingleTripTime = 0.f;

 FHighPingDelegate HighPingDelegate;

 void BroadcastElim(APlayerState* Attacker, APlayerState* Victim);
 
protected:
 virtual void BeginPlay() override;
 void SetHUDTime();
 void PollInit();
 virtual void SetupInputComponent() override;

 /*
 * 客户端和服务器的时间同步
 */

 //请求服务器时间，发送请求时传入客户端当前时间
 UFUNCTION(Server, Reliable)
 void ServerRequestServerTime(float TimeOfClientRequest);

 //报告服务器当前时间到客户端响应服务器请求服务器时间
 UFUNCTION(Client, Reliable)
 void ClientReportServerTime(float TimeOfClientRequest, float TimeServerReceivedClientRequest);

 float ClientServerDelta = 0.f; //客户端和服务器的时间差

 UPROPERTY(EditAnywhere, Category = Time)
 float TimeSyncFrequency = 5.f;

 float TimeSyncRunningTime = 0.f;
 void CheckTimeSync(float DeltaTime);

 UFUNCTION(Server, Reliable)
 void ServerCheckMatchState();

 UFUNCTION(Client, Reliable)
 void ClientJoinMidgame(FName StateOfMatch, float Warmup, float Match, float Cooldown, float StartingTime, bool bIsTeamsMatch);

 void HighPingWarning();
 void StopHighPingWarning();
 void CheckPing(float DeltaTime);

 void ShowReturnToMainMenu();

 UFUNCTION(Client, Reliable)
 void ClientElimAnnouncement(APlayerState* Attacker, APlayerState* Victim);

 UPROPERTY(Replicated)
 bool bShowTeamScores = false;

 FString GetInfoText(const TArray<class ABlasterPlayerState*>& Players);
 FString GetTeamsInfoText(class ABlasterGameState* BlasterGameState);

private:
 UPROPERTY()
 class ABlasterHUD* BlasterHUD;

 /*
 * 返回主菜单
 */
 UPROPERTY(EditAnywhere, Category = HUD)
 TSubclassOf<class UUserWidget> ReturnToMainMenuWdiget;

 UPROPERTY()
 class UReturnToMainMenu* ReturnToMainMenu;

 bool bReturnToMainMenuOpen = false;

 UPROPERTY()
 class ABlasterGameMode* BlasterGameMode;

 float LevelStartingTime = 0.f;
 float MatchTime = 0.f;
 float WarmupTime = 0.f;
 float CooldownTime = 0.f;
 uint32 CountdownInt = 0;

 UPROPERTY(ReplicatedUsing = OnRep_MatchState)
 FName MatchState;

 UFUNCTION()
 void OnRep_MatchState();

 UPROPERTY()
 class UCharacterOverlay* CharacterOverlay;
 bool bInitializeCharacterOverlay = false;

 float HUDHealth;
 bool bInitializeHealth = false;
 float HUDMaxHealth;
 float HUDScore;
 bool bInitializeScore = false;
 int32 HUDDefeats;
 bool bInitializeDefeats = false;
 int32 HUDGrenades;
 bool bInitializeGrenades = false;
 float HUDShield;
 bool bInitializeShield = false;
 float HUDMaxShield;
 float HUDCarriedAmmo;
 bool bInitializeCarriedAmmo = false;
 float HUDWeaponAmmo;
 bool bInitializeWeaponAmmo = false;

 float HighPingRunningTime = 0.f;

 UPROPERTY(EditAnywhere)
 float HighPingDuration = 5.f;

 float PingAnimationRunningTime = 0.f;

 UPROPERTY(EditAnywhere)
 float CheckPingFrequency = 20.f;

 UFUNCTION(Server, Reliable)
 void ServerReportPingStatus(bool bHighPing);

 UPROPERTY(EditAnywhere)
 float HighPingThreshold = 50.f;
};
```

## BlasterPlayerController.cpp

### BroadcastElim

```cpp
//广播死亡。
void ABlasterPlayerController::BroadcastElim(APlayerState* Attacker, APlayerState* Victim)
{
    //调用客户端多播RPC。
 ClientElimAnnouncement(Attacker, Victim);
}
```

### ClientElimAnnouncement

```cpp
//客户端多播RPC。
void ABlasterPlayerController::ClientElimAnnouncement_Implementation(APlayerState* Attacker, APlayerState* Victim)
{
    //获得客户端的玩家状态。
 APlayerState* Self = GetPlayerState<APlayerState>();
 if (Attacker && Victim && Self)
 {
    //获得客户端HUD。
  BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
  if (BlasterHUD)
  {
    //如果攻击者等于此客户端并且被攻击者不等于此客户端。
   if (Attacker == Self && Victim != Self)
   {
    //调用HUD显示你击杀了谁。
    BlasterHUD->AddElimAnnouncement("You", Victim->GetPlayerName());
    return;
   }
   //如果被攻击者等于此客户端并且攻击者等于此客户端。
   if (Victim == Self && Attacker != Self)
   {
    //调用HUD显示谁击杀了你。
    BlasterHUD->AddElimAnnouncement(Attacker->GetPlayerName(), "you");
    return;
   }
   //如果全部等于此客户端。
   if (Attacker == Victim && Attacker == Self)
   {
    //调用HUD显示误伤。
    BlasterHUD->AddElimAnnouncement("You", "yourself");
    return;
   }
   //如果攻击者和被攻击者同一人，不等于此客户端。
   if (Attacker == Victim && Attacker != Self)
   {
    //调用HUD显示它们自己误伤。
    BlasterHUD->AddElimAnnouncement(Attacker->GetPlayerName(), "themselves");
    return;
   }
   //最后一次情况被攻击者和攻击者不是本人也不是此客户端。直接调用谁攻击了谁。
   BlasterHUD->AddElimAnnouncement(Attacker->GetPlayerName(), Victim->GetPlayerName());
  }
 }
}
```

### BeginPlay

```cpp
void ABlasterPlayerController::BeginPlay()
{
 Super::BeginPlay();

//获得BlasterHUD。调用服务器RPC检查比赛状态。
 BlasterHUD = Cast<ABlasterHUD>(GetHUD());
 ServerCheckMatchState();
}
```

### ServerCheckMatchState

```cpp
//服务器检查比赛状态RPC。
void ABlasterPlayerController::ServerCheckMatchState_Implementation()
{
    //获得游戏模式。
 ABlasterGameMode* GameMode = Cast<ABlasterGameMode>(UGameplayStatics::GetGameMode(this));
 if (GameMode)
 {
    //获取热身时间，比赛时间，冷却时间，关卡开始时间，比赛状态，是否是团队比赛。调用客户端中途加入RPC。
  WarmupTime = GameMode->WarmupTime;
  MatchTime = GameMode->MatchTime;
  CooldownTime = GameMode->CooldownTime;
  LevelStartingTime = GameMode->LevelStartingTime;
  MatchState = GameMode->GetMatchState();
  bShowTeamScores = GameMode->IsTeamsMatch();
  ClientJoinMidgame(MatchState, WarmupTime, MatchTime, CooldownTime, LevelStartingTime, bShowTeamScores);
 }
}
```

### ClientJoinMidgame

```cpp
//客户端中途加入游戏RPC。
void ABlasterPlayerController::ClientJoinMidgame_Implementation(FName StateOfMatch, float Warmup, float Match, float Cooldown, float StartingTime, bool bIsTeamsMatch)
{
    //同步从服务器发送来的游戏信息。
 WarmupTime = Warmup;
 MatchTime = Match;
 CooldownTime = Cooldown;
 LevelStartingTime = StartingTime;
 MatchState = StateOfMatch;
 bShowTeamScores = bIsTeamsMatch;
 //调用处理比赛状态变化方法。
 OnMatchStateSet(MatchState, bIsTeamsMatch);
 //如果当前的比赛状态是等待去开始。
 if (BlasterHUD && MatchState == MatchState::WaitingToStart)
 {
    //调用显示通知HUD。
  BlasterHUD->AddAnnouncement();
 }
}
```

### OnMatchStateSet

```cpp
//处理吧比赛状态变化。
void ABlasterPlayerController::OnMatchStateSet(FName State, bool bTeamMatch)
{
 MatchState = State;

//如果比赛状态等于进行中。
 if (MatchState == MatchState::InProgress)
 {
    //调用处理比赛开始。
  HandleMatchHasStarted(bTeamMatch);
 }
 //如果比赛状态等于冷却中。
 else if (MatchState == MatchState::Cooldown)
 {
    //调用处理比赛结束冷却中。
  HandleCooldown();
 }
}
```

### HandleMatchHasStarted

```cpp
void ABlasterPlayerController::HandleMatchHasStarted(bool bTeamMatch)
{
    //如果是权威，是否显示团队分数等于传入的分数。
 if (HasAuthority()) bShowTeamScores = bTeamMatch;
 //获得HUD。
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 if (BlasterHUD)
 {
    //如果角色覆盖HUD为空，调用添加角色覆盖HUD。
  if (BlasterHUD->CharacterOverlay == nullptr) BlasterHUD->AddCharacterOverlay();
  //如果角色通知HUD存在。
  if (BlasterHUD->Announcement)
  {
    //设置隐藏通知HUD。
   BlasterHUD->Announcement->SetVisibility(ESlateVisibility::Hidden);
  }

//如果是团队比赛就是显示团队分数。
  if (bShowTeamScores)
  {
    //调用初始化团队分数。
   InitTeamScores();
  }
  else
  {
    //否则就是隐藏团队分数。
   HideTeamScores();
  }
 }
}
```

### InitTeamScores

```cpp
//初始化分数。
void ABlasterPlayerController::InitTeamScores()
{
    //获得BlasterHUD。
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 //判断分数显示的文本控件是否存在。
 bool bHUDValid = BlasterHUD &&
  BlasterHUD->CharacterOverlay &&
  BlasterHUD->CharacterOverlay->RedTeamScore &&
  BlasterHUD->CharacterOverlay->BlueTeamScore &&
  BlasterHUD->CharacterOverlay->ScoreSpacerText;
  //如果都存在。
 if (bHUDValid)
 {
    //创建字符串0和|表示两队分数。
  FString Zero("0");
  FString Spacer("|");
  //设置分数显示文本。
  BlasterHUD->CharacterOverlay->RedTeamScore->SetText(FText::FromString(Zero));
  BlasterHUD->CharacterOverlay->BlueTeamScore->SetText(FText::FromString(Zero));
  BlasterHUD->CharacterOverlay->ScoreSpacerText->SetText(FText::FromString(Spacer));
 }
}
```

### HideTeamScores

```cpp
//隐藏团队分数。
void ABlasterPlayerController::HideTeamScores()
{
    //获得BlasterHUD。
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 //判断分数显示的文本控件是否存在。
 bool bHUDValid = BlasterHUD &&
  BlasterHUD->CharacterOverlay &&
  BlasterHUD->CharacterOverlay->RedTeamScore &&
  BlasterHUD->CharacterOverlay->BlueTeamScore &&
  BlasterHUD->CharacterOverlay->ScoreSpacerText;
  //如果都存在。
 if (bHUDValid)
 {
    //全部文本设置为空，也就是不显示。
  BlasterHUD->CharacterOverlay->RedTeamScore->SetText(FText());
  BlasterHUD->CharacterOverlay->BlueTeamScore->SetText(FText());
  BlasterHUD->CharacterOverlay->ScoreSpacerText->SetText(FText());
 }
}
```

### HandleCooldown

```cpp
void ABlasterPlayerController::HandleCooldown()
{
    //获得HUD。
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 if (BlasterHUD)
 {
    //移除角色的覆盖层HUD。
  BlasterHUD->CharacterOverlay->RemoveFromParent();
  //判断如果通知HUD，通知HUD的文本都存在。
  bool bHUDValid = BlasterHUD->Announcement && BlasterHUD->Announcement->AnnouncementText && BlasterHUD->Announcement->InfoText;

  if (bHUDValid)
  {
    //设置通知HUD为可见。
   BlasterHUD->Announcement->SetVisibility(ESlateVisibility::Visible);
   //创建新的比赛开始字符串。
   FString AnnouncementText = Announcement::NewMatchStartsIn;
   //设置通知的文本为比赛开始字符串。
   BlasterHUD->Announcement->AnnouncementText->SetText(FText::FromString(AnnouncementText));

//获得游戏状态和玩家状态。
   ABlasterGameState* BlasterGameState = Cast<ABlasterGameState>(UGameplayStatics::GetGameState(this));
   ABlasterPlayerState* BlasterPlayerState = GetPlayerState<ABlasterPlayerState>();
   if (BlasterGameState && BlasterPlayerState)
   {
    //从游戏状态中获得最高得分玩家数组。
    TArray<ABlasterPlayerState*> TopPlayers = BlasterGameState->TopScoringPlayers;
    //如果是团队模式需要显示团队分数，调用获得获胜的团队信息。如果不是团队，调用获得最高得分的玩家。
    FString InfoTextString = bShowTeamScores ? GetTeamsInfoText(BlasterGameState) : GetInfoText(TopPlayers);
    
    //设置通知等于获得的信息字符串。
    BlasterHUD->Announcement->InfoText->SetText(FText::FromString(InfoTextString));
   }
  }
 }
 //获得玩家角色。
 ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(GetPawn());
 //如果获得到玩家战斗组。
 if (BlasterCharacter && BlasterCharacter->GetCombat())
 {
    //调用禁止操作游戏，禁止开火。
  BlasterCharacter->bDisableGameplay = true;
  BlasterCharacter->GetCombat()->FireButtonPressed(false);
 }
}
```

### GetTeamsInfoText

```cpp
//获得团队信息文本。
FString ABlasterPlayerController::GetTeamsInfoText(ABlasterGameState* BlasterGameState)
{
    //如果游戏状态为空返回空字符串。
 if (BlasterGameState == nullptr) return FString();
 //创建信息文本字符串。
 FString InfoTextString;
 //获得红队和蓝队的分数。
 const int32 RedTeamScore = BlasterGameState->RedTeamScore;
 const int32 BlueTeamScore = BlasterGameState->BlueTeamScore;
 //如果都为0。
 if (RedTeamScore == 0 && BlueTeamScore == 0)
 {
    //设置文本为没有获胜队伍。
  InfoTextString = Announcement::ThereIsNoWinner;
 }
 //如果红队和蓝队得分一致。
 else if (RedTeamScore == BlueTeamScore)
 {
    //两支队伍都获得了胜利平局。
  InfoTextString = FString::Printf(TEXT("%s\n"), *Announcement::TeamsTiedForTheWin);
  InfoTextString.Append(Announcement::RedTeam);
  InfoTextString.Append(TEXT("\n"));
  InfoTextString.Append(Announcement::BlueTeam);
  InfoTextString.Append(TEXT("\n"));
 }
 //如果红队大于蓝队。
 else if (RedTeamScore > BlueTeamScore)
 {
    //红队获得胜利。
  InfoTextString = Announcement::RedTeamWins;
  InfoTextString.Append(TEXT("\n"));
  InfoTextString.Append(FString::Printf(TEXT("%s: %d\n"), *Announcement::RedTeam, RedTeamScore));
  InfoTextString.Append(FString::Printf(TEXT("%s: %d\n"), *Announcement::BlueTeam, BlueTeamScore));
 }
 //如果蓝队大于红队。
 else if (BlueTeamScore > RedTeamScore)
 {
    //蓝队获得胜利。
  InfoTextString = Announcement::BlueTeamWins;
  InfoTextString.Append(TEXT("\n"));
  InfoTextString.Append(FString::Printf(TEXT("%s: %d\n"), *Announcement::BlueTeam, BlueTeamScore));
  InfoTextString.Append(FString::Printf(TEXT("%s: %d\n"), *Announcement::RedTeam, RedTeamScore));
 }
 //返回信息文本字符串。
 return InfoTextString;
}
```

### GetInfoText

```cpp
FString ABlasterPlayerController::GetInfoText(const TArray<class ABlasterPlayerState*>& Players)
{
    //获得玩家状态。
 ABlasterPlayerState* BlasterPlayerState = GetPlayerState<ABlasterPlayerState>();
 //如果为空直接返回空字符串。
 if (BlasterPlayerState == nullptr) return FString();
 FString InfoTextString;
 //如果玩家数为0。
 if (Players.Num() == 0)
 {
    //没有人获胜。
  InfoTextString = Announcement::ThereIsNoWinner;
 }
 //如果最高分玩家为一个并且等于此玩家状态。
 else if (Players.Num() == 1 && Players[0] == BlasterPlayerState)
 {
    //本机玩家获胜。
  InfoTextString = Announcement::YouAreTheWinner;
 }
 //如果只有一个玩家最高分。
 else if (Players.Num() == 1)
 {
    //显示获胜的玩家并显示玩家的名字。
  InfoTextString = FString::Printf(TEXT("Winner: \n%s"), *Players[0]->GetPlayerName());
 }
 //如果最高分玩家大于一个。
 else if (Players.Num() > 1)
 {
    //显示这些玩家获胜。
  InfoTextString = Announcement::PlayersTiedForTheWin;
  InfoTextString.Append(FString("\n"));
  //循环最高分的玩家并添加到显示文本中。
  for (auto TiedPlayer : Players)
  {
   InfoTextString.Append(FString::Printf(TEXT("%s\n"), *TiedPlayer->GetPlayerName()));
  }
 }
 //返回显示文本。
 return InfoTextString;
}
```

### Tick

```cpp
void ABlasterPlayerController::Tick(float DeltaTime)
{
 Super::Tick(DeltaTime);

//设置时间HUD。检查时间同步。初始化轮询。检查Ping值。
 SetHUDTime();
 CheckTimeSync(DeltaTime);
 PollInit();

 CheckPing(DeltaTime);
}
```

### SetHUDTime

```cpp
void ABlasterPlayerController::SetHUDTime()
{
   //创建剩余时间浮点数。
 float TimeLeft = 0.f;
 //比赛状态等于等待去开始。剩余时间等于热身时间减去服务器时间加上关卡开始时间。计算原理拆分成服务器时间减去关卡开始时间就得到了关卡已经过去的时间。热身时间减去这个时间就是剩余的时间。
 if (MatchState == MatchState::WaitingToStart) TimeLeft = WarmupTime - GetServerTime() + LevelStartingTime;
 //如果比赛状态是进行中，剩余时间就是热身时间加上比赛时间减去关卡过去的时间。
 else if (MatchState == MatchState::InProgress) TimeLeft = WarmupTime + MatchTime - GetServerTime() + LevelStartingTime;
 //如果比赛状态是冷却中，那剩余时间就是冷却时间加上剩余时间加上比赛时间减去关卡过去的时间。
 else if (MatchState == MatchState::Cooldown) TimeLeft = CooldownTime + WarmupTime + MatchTime - GetServerTime() + LevelStartingTime;
 //将计算的浮点数向上取整为秒数。
 uint32 SecondsLeft = FMath::CeilToInt(TimeLeft);

//如果是权威。
 if (HasAuthority())
 {
   //如果游戏模式为空。
  if (BlasterGameMode == nullptr)
  {
   //获得游戏模式。获得关卡开始时间。
   BlasterGameMode = Cast<ABlasterGameMode>(UGameplayStatics::GetGameMode(this));
   LevelStartingTime = BlasterGameMode->LevelStartingTime;
  }
  //再次判断获得游戏模式。
  BlasterGameMode = BlasterGameMode == nullptr ? Cast<ABlasterGameMode>(UGameplayStatics::GetGameMode(this)) : BlasterGameMode;
  if (BlasterGameMode)
  {
   //剩余时间等于游戏模式获得倒计时时间加上关卡开始的时间。
   SecondsLeft = FMath::CeilToInt(BlasterGameMode->GetCountdownTime() + LevelStartingTime);
  }
 }

//如果倒计时时间不等于计算的剩余时间。
 if (CountdownInt != SecondsLeft)
 {
   //如果比赛状态是等待开始或者冷却。
  if (MatchState == MatchState::WaitingToStart || MatchState == MatchState::Cooldown)
  {
   //设置通知的倒计时为TimeLeft。
   SetHUDAnnouncementCountdown(TimeLeft);
  }
  if (MatchState == MatchState::InProgress)
  {
   //设置比赛倒计时时间等于TimeLeft。
   SetHUDMatchCountdown(TimeLeft);
  }
 }

//倒计时时间等于计算的剩余时间。
 CountdownInt = SecondsLeft;
}
```

### GetServerTime

```cpp
float ABlasterPlayerController::GetServerTime()
{
   //如果是权威，返回现在的时间。
 if (HasAuthority()) return GetWorld()->GetTimeSeconds();
 //否则就是客户端，返回当前的时间加上客户端和服务器的时间差。
 return GetWorld()->GetTimeSeconds() + ClientServerDelta;
}
```

### SetHUDAnnouncementCountdown

```cpp
void ABlasterPlayerController::SetHUDAnnouncementCountdown(float CountdownTime)
{
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 bool bHUDValid = BlasterHUD &&
  BlasterHUD->Announcement &&
  BlasterHUD->Announcement->WarmupTime;
 if (bHUDValid)
 {
   //如果剩余时间小于0。
  if (CountdownTime < 0.f)
  {
   //设置热身时间HUD的文本为空也就是隐藏，直接返回。
   BlasterHUD->Announcement->WarmupTime->SetText(FText());
   return;
  }

//定义分钟等于剩余时间除以60秒的除数。秒数等于剩余时间减去分钟乘以60秒。
  int32 Minutes = FMath::FloorToInt(CountdownTime / 60.f);
  int32 Seconds = CountdownTime - Minutes * 60;
  //设置显示的剩余时间文本为分钟和秒数。
  FString CountdownText = FString::Printf(TEXT("%02d:%02d"), Minutes, Seconds);
  BlasterHUD->Announcement->WarmupTime->SetText(FText::FromString(CountdownText));
 }
}
```

### SetHUDMatchCountdown

```cpp
//原理和上面的方法一致。
void ABlasterPlayerController::SetHUDMatchCountdown(float CountdownTime)
{
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 bool bHUDValid = BlasterHUD &&
  BlasterHUD->CharacterOverlay &&
  BlasterHUD->CharacterOverlay->MatchCountdownText;
 if (bHUDValid)
 {
  if (CountdownTime < 0.f)
  {
   BlasterHUD->CharacterOverlay->MatchCountdownText->SetText(FText());
   return;
  }

  int32 Minutes = FMath::FloorToInt(CountdownTime / 60.f);
  int32 Seconds = CountdownTime - Minutes * 60;
  FString CountdownText = FString::Printf(TEXT("%02d:%02d"), Minutes, Seconds);
  BlasterHUD->CharacterOverlay->MatchCountdownText->SetText(FText::FromString(CountdownText));
 }
}
```

### CheckTimeSync

```cpp
void ABlasterPlayerController::CheckTimeSync(float DeltaTime)
{
   //时间同步计时时间等于加上增量时间。
 TimeSyncRunningTime += DeltaTime;
 //如果是本地控制器并且时间同步计时时间大于规定的同步时间计算间隔时间。
 if (IsLocalController() && TimeSyncRunningTime > TimeSyncFrequency)
 {
   //调用服务器RPC请求服务器时间。时间同步计时归零。
  ServerRequestServerTime(GetWorld()->GetTimeSeconds());
  TimeSyncRunningTime = 0.f;
 }
}
```

### ServerRequestServerTime

```cpp
void ABlasterPlayerController::ServerRequestServerTime_Implementation(float TimeOfClientRequest)
{
   //创建服务器接收到的时间。调用客户端回复服务器时间。传入客户端发送请求时间和服务器收到请求时间。
 float ServerTimeOfReceipt = GetWorld()->GetTimeSeconds();
 ClientReportServerTime(TimeOfClientRequest, ServerTimeOfReceipt);
}
```

### ClientReportServerTime

```cpp
void ABlasterPlayerController::ClientReportServerTime_Implementation(float TimeOfClientRequest, float TimeServerReceivedClientRequest)
{
   //来回总共的时间等于现在时间减去客户端发送请求的时间。
 float RoundTripTime = GetWorld()->GetTimeSeconds() - TimeOfClientRequest;
 //单程时间等于来回的一半。
 SingleTripTime = 0.5f * RoundTripTime;
 //当前服务器时间等于服务器收到请求发送时间加上单程时间。
 float CurrentServerTime = TimeServerReceivedClientRequest + SingleTripTime;
 //服务器和客户端的时间差是当前服务器时间减去客户端时间。
 ClientServerDelta = CurrentServerTime - GetWorld()->GetTimeSeconds();
}
```

### PollInit

```cpp
void ABlasterPlayerController::PollInit()
{
   //如果覆盖HUD指针为空。
 if (CharacterOverlay == nullptr)
 {
   //如果角色HUD的角色覆盖层存在。
  if (BlasterHUD && BlasterHUD->CharacterOverlay)
  {
   //获得角色的HUD的角色覆盖HUD
   CharacterOverlay = BlasterHUD->CharacterOverlay;
   //如果覆盖HUD存在。
   if (CharacterOverlay)
   {
      //如果已经初始化相应参数，就设置对应角色的参数HUD。
    if (bInitializeHealth) SetHUDHealth(HUDHealth, HUDMaxHealth);
    if (bInitializeShield) SetHUDShield(HUDShield, HUDMaxShield);
    if (bInitializeScore) SetHUDScore(HUDScore);
    if (bInitializeDefeats) SetHUDDefeats(HUDDefeats);
    if (bInitializeWeaponAmmo) SetHUDWeaponAmmo(HUDWeaponAmmo);
    if (bInitializeCarriedAmmo) SetHUDCarriedAmmo(HUDCarriedAmmo);

//获得玩家角色。
    ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(GetPawn());
    //如果战斗组件存在。
    if (BlasterCharacter && BlasterCharacter->GetCombat())
    {
      //如果已经初始化手雷，设置手雷HUD。
     if (bInitializeGrenades) SetHUDGrenades(BlasterCharacter->GetCombat()->GetGrenades());
    }
    //如果应该显示团队分数。
    if (bShowTeamScores)
    {
      //初始化团队分数。
     InitTeamScores();
    }
    else
    {
      //隐藏团队分数。
     HideTeamScores();
    }
   }
  }
 }
}
```

### SetHUDHealth

```cpp
void ABlasterPlayerController::SetHUDHealth(float Health, float MaxHealth)
{
   //如果健康相关的HUD小部件全部存在。
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 bool bHUDValid = BlasterHUD &&
  BlasterHUD->CharacterOverlay &&
  BlasterHUD->CharacterOverlay->HealthBar &&
  BlasterHUD->CharacterOverlay->HealthText;
 if (bHUDValid)
 {
   //进度条百分比等于当前生命除以最大生命。
  const float HealthPercent = Health / MaxHealth;
  BlasterHUD->CharacterOverlay->HealthBar->SetPercent(HealthPercent);
  //设置生命值文本等于当前生命/最大生命。
  FString HealthText = FString::Printf(TEXT("%d/%d"), FMath::CeilToInt(Health), FMath::CeilToInt(MaxHealth));
  BlasterHUD->CharacterOverlay->HealthText->SetText(FText::FromString(HealthText));
 }
 else
 {
   //如果不能使用，代表还没有初始化成功。HUD生命和最大等于传入的生命和最大生命。
  bInitializeHealth = true;
  HUDHealth = Health;
  HUDMaxHealth = MaxHealth;
 }
}
```

### SetHUDShield

```cpp
//护盾和生命值同理。
void ABlasterPlayerController::SetHUDShield(float Shield, float MaxShield)
{
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 bool bHUDValid = BlasterHUD &&
  BlasterHUD->CharacterOverlay &&
  BlasterHUD->CharacterOverlay->ShieldBar &&
  BlasterHUD->CharacterOverlay->ShieldText;
 if (bHUDValid)
 {
  const float ShieldPercent = Shield / MaxShield;
  BlasterHUD->CharacterOverlay->ShieldBar->SetPercent(ShieldPercent);
  FString ShieldText = FString::Printf(TEXT("%d/%d"), FMath::CeilToInt(Shield), FMath::CeilToInt(MaxShield));
  BlasterHUD->CharacterOverlay->ShieldText->SetText(FText::FromString(ShieldText));
 }
 else
 {
  bInitializeShield = true;
  HUDShield = Shield;
  HUDMaxShield = MaxShield;
 }
}
```

### SetHUDScore

```cpp
//和上面方法同理。
void ABlasterPlayerController::SetHUDScore(float Score)
{
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 bool bHUDValid = BlasterHUD &&
  BlasterHUD->CharacterOverlay &&
  BlasterHUD->CharacterOverlay->ScoreAmount;
 if (bHUDValid)
 {
  FString ScoreText = FString::Printf(TEXT("%d"), FMath::FloorToInt(Score));
  BlasterHUD->CharacterOverlay->ScoreAmount->SetText(FText::FromString(ScoreText));
 }
 else
 {
  bInitializeScore = true;
  HUDScore = Score;
 }
}
```

### SetHUDDefeats

```cpp
//和上面方法同理。
void ABlasterPlayerController::SetHUDDefeats(int32 Defeats)
{
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 bool bHUDValid = BlasterHUD &&
  BlasterHUD->CharacterOverlay &&
  BlasterHUD->CharacterOverlay->DefeatsAmount;
 if (bHUDValid)
 {
  FString DefeatsText = FString::Printf(TEXT("%d"), Defeats);
  BlasterHUD->CharacterOverlay->DefeatsAmount->SetText(FText::FromString(DefeatsText));
 }
 else
 {
  bInitializeDefeats = true;
  HUDDefeats = Defeats;
 }
}
```

### SetHUDWeaponAmmo

```cpp
//和上面方法同理。
void ABlasterPlayerController::SetHUDWeaponAmmo(int32 Ammo)
{
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 bool bHUDValid = BlasterHUD &&
  BlasterHUD->CharacterOverlay &&
  BlasterHUD->CharacterOverlay->WeaponAmmoAmount;
 if (bHUDValid)
 {
  FString AmmoText = FString::Printf(TEXT("%d"), Ammo);
  BlasterHUD->CharacterOverlay->WeaponAmmoAmount->SetText(FText::FromString(AmmoText));
 }
 else
 {
  bInitializeWeaponAmmo = true;
  HUDWeaponAmmo = Ammo;
 }
}
```

### SetHUDCarriedAmmo

```cpp
//和上面方法同理。
void ABlasterPlayerController::SetHUDCarriedAmmo(int32 Ammo)
{
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 bool bHUDValid = BlasterHUD &&
  BlasterHUD->CharacterOverlay &&
  BlasterHUD->CharacterOverlay->CarriedAmmoAmount;
 if (bHUDValid)
 {
  FString AmmoText = FString::Printf(TEXT("%d"), Ammo);
  BlasterHUD->CharacterOverlay->CarriedAmmoAmount->SetText(FText::FromString(AmmoText));
 }
 else
 {
  bInitializeCarriedAmmo = true;
  HUDCarriedAmmo = Ammo;
 }
}
```

### SetHUDGrenades

```cpp
//和上面方法同理。
void ABlasterPlayerController::SetHUDGrenades(int32 Grenades)
{
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 bool bHUDValid = BlasterHUD &&
  BlasterHUD->CharacterOverlay &&
  BlasterHUD->CharacterOverlay->GrenadesText;
 if (bHUDValid)
 {
  FString GrenadesText = FString::Printf(TEXT("%d"), Grenades);
  BlasterHUD->CharacterOverlay->GrenadesText->SetText(FText::FromString(GrenadesText));
 }
 else
 {
  bInitializeGrenades = true;
  HUDGrenades = Grenades;
 }
}
```

### CheckPing

```cpp
void ABlasterPlayerController::CheckPing(float DeltaTime)
{
   //如果是权威直接返回，因为服务器没有延迟。
 if (HasAuthority()) return;
 //计算Ping值的时间等于加上增量时间。
 HighPingRunningTime += DeltaTime;
 //如果时间大于设置的检查Ping的时间。
 if (HighPingRunningTime > CheckPingFrequency)
 {
   //获得玩家状态。
  PlayerState = PlayerState == nullptr ? GetPlayerState<ABlasterPlayerState>() : PlayerState;
  if (PlayerState)
  {
   //如果获得的Ping值大于设置的Ping值。
   if (PlayerState->GetCompressedPing() * 4 > HighPingThreshold) // ping值被压缩，实际上是ping值的四分之一
   {
      //调用高Ping警告。设置Ping高动画运行时间归零。调用服务器报告Ping状态RPC传入true。
    HighPingWarning();
    PingAnimationRunningTime = 0.f;
    ServerReportPingStatus(true);
   }
   else
   {
      //如果Ping值不高于设置的值，调用服务器报告Ping状态RPC传入false。
    ServerReportPingStatus(false);
   }
  }
  //设置检查Ping值间隔归零。
  HighPingRunningTime = 0.f;
 }
 //判断是否正在播放高Ping警告标志动画等于角色覆盖HUD是否存在和覆盖HUD的高Ping动画是否正在播放。
 bool bHighPingAnimationPlaying =
  BlasterHUD && BlasterHUD->CharacterOverlay &&
  BlasterHUD->CharacterOverlay->HighPingAnimation &&
  BlasterHUD->CharacterOverlay->IsAnimationPlaying(BlasterHUD->CharacterOverlay->HighPingAnimation);
  //如果正在播放。
 if (bHighPingAnimationPlaying)
 {
   //播放时间等于加上增量时间。
  PingAnimationRunningTime += DeltaTime;
  //如果播放的时间大于设置的时间。
  if (PingAnimationRunningTime > HighPingDuration)
  {
   //停止播放动画。
   StopHighPingWarning();
  }
 }
}
```

### HighPingWarning

```cpp
void ABlasterPlayerController::HighPingWarning()
{
   //如果高ping警告的HUD都存在。
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 bool bHUDValid = BlasterHUD &&
  BlasterHUD->CharacterOverlay &&
  BlasterHUD->CharacterOverlay->HighPingImage &&
  BlasterHUD->CharacterOverlay->HighPingAnimation;
 if (bHUDValid)
 {
   //设置高Ping图片的透明度为不透明。开始从动画开始播放动画，循环5次。
  BlasterHUD->CharacterOverlay->HighPingImage->SetOpacity(1.f);
  BlasterHUD->CharacterOverlay->PlayAnimation(BlasterHUD->CharacterOverlay->HighPingAnimation, 0.f, 5);
 }
}
```

### ServerReportPingStatus

```cpp
void ABlasterPlayerController::ServerReportPingStatus_Implementation(bool bHighPing)
{
   //广播高Ping委托，广播Ping是否过高。
 HighPingDelegate.Broadcast(bHighPing);
}
```

### StopHighPingWarning

```cpp
//停止高Ping警告。
void ABlasterPlayerController::StopHighPingWarning()
{
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 bool bHUDValid = BlasterHUD &&
  BlasterHUD->CharacterOverlay &&
  BlasterHUD->CharacterOverlay->HighPingImage &&
  BlasterHUD->CharacterOverlay->HighPingAnimation;
 if (bHUDValid)
 {
   //设置高Ping图片透明度为完全透明。
  BlasterHUD->CharacterOverlay->HighPingImage->SetOpacity(0.f);
  //如果在播放高Ping警告动画。
  if (BlasterHUD->CharacterOverlay->IsAnimationPlaying(BlasterHUD->CharacterOverlay->HighPingAnimation))
  {
   //停止播放动画。
   BlasterHUD->CharacterOverlay->StopAnimation(BlasterHUD->CharacterOverlay->HighPingAnimation);
  }
 }
}
```

### GetLifetimeReplicatedProps

```cpp
//注册网络复制参数。
void ABlasterPlayerController::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
 Super::GetLifetimeReplicatedProps(OutLifetimeProps);

 DOREPLIFETIME(ABlasterPlayerController, MatchState);
 DOREPLIFETIME(ABlasterPlayerController, bShowTeamScores);
}
```

### ShowReturnToMainMenu

```cpp
void ABlasterPlayerController::ShowReturnToMainMenu()
{
 //如果返回小组件为空直接返回。
 if (ReturnToMainMenuWdiget == nullptr) return;
 if (ReturnToMainMenu == nullptr)
 {
   //创建返回菜单小部件。
  ReturnToMainMenu = CreateWidget<UReturnToMainMenu>(this, ReturnToMainMenuWdiget);
 }
 if (ReturnToMainMenu)
 {
   //是否打开小部件等于反过来的值。因为模式是false，如果调用这个方法就代表打开小部件，所以当前值应为false的反值。
  bReturnToMainMenuOpen = !bReturnToMainMenuOpen;
  //如果打开了小部件。
  if (bReturnToMainMenuOpen)
  {
   //调用小部件的菜单初始化。
   ReturnToMainMenu->MenuSetup();
  }
  else
  {
   //调用小部件的关闭。
   ReturnToMainMenu->MenuTearDown();
  }
 }
}
```

### OnPossess

```cpp
//当拥有控制的角色时。
void ABlasterPlayerController::OnPossess(APawn* InPawn)
{
 Super::OnPossess(InPawn);

//获得玩家角色。
 ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(InPawn);
 if (BlasterCharacter)
 {
   //调用设置生命值HUD。
  SetHUDHealth(BlasterCharacter->GetHealth(), BlasterCharacter->GetMaxHealth());
 }
}
```

### SetHUDRedTeamScore

```cpp
//设置显示红队分数HUD。
void ABlasterPlayerController::SetHUDRedTeamScore(int32 RedScore)
{
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 bool bHUDValid = BlasterHUD &&
  BlasterHUD->CharacterOverlay &&
  BlasterHUD->CharacterOverlay->RedTeamScore;
 if (bHUDValid)
 {
    //创建分数文本字符串。设置红队分数文本控件的显示文本。
  FString ScoreText = FString::Printf(TEXT("%d"), RedScore);
  BlasterHUD->CharacterOverlay->RedTeamScore->SetText(FText::FromString(ScoreText));
 }
}
```

### SetHUDBlueTeamScore

```cpp
//设置蓝队显示分数HUD。
void ABlasterPlayerController::SetHUDBlueTeamScore(int32 BlueScore)
{
 BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
 bool bHUDValid = BlasterHUD &&
  BlasterHUD->CharacterOverlay &&
  BlasterHUD->CharacterOverlay->BlueTeamScore;
 if (bHUDValid)
 {
  FString ScoreText = FString::Printf(TEXT("%d"), BlueScore);
  BlasterHUD->CharacterOverlay->BlueTeamScore->SetText(FText::FromString(ScoreText));
 }
}
```

### SetupInputComponent

```cpp
//设置输入组件。
void ABlasterPlayerController::SetupInputComponent()
{
 Super::SetupInputComponent();
 //如果输入组件为空直接返回。
 if (InputComponent == nullptr) return;

//绑定按下退出按钮的回调函数ShowReturnToMainMenu。
 InputComponent->BindAction("Quit", IE_Pressed, this, &ABlasterPlayerController::ShowReturnToMainMenu);
}
```

### ReceivedPlayer

```cpp
//在玩家控制器与客户端成功连接后调用。
void ABlasterPlayerController::ReceivedPlayer()
{
 Super::ReceivedPlayer();

//如果是本地控制。
 if (IsLocalController())
 {
   //调用服务器RPC请求服务器时间。
  ServerRequestServerTime(GetWorld()->GetTimeSeconds());
 }
}
```

### OnRep_MatchState

```cpp
//比赛状态的复制通知。
void ABlasterPlayerController::OnRep_MatchState()
{
   //如果比赛状态等于进行中。
 if (MatchState == MatchState::InProgress)
 {
   //调用处理比赛开始方法。
  HandleMatchHasStarted();
 }
 else if (MatchState == MatchState::Cooldown)
 {
   //调用处理比赛结束冷却方法。
  HandleCooldown();
 }
}
```
