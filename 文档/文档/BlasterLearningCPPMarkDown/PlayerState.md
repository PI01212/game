# BlasterPlayerState

玩家状态，主要用来存储角色的分数，死亡次数，所属团队。

## BlasterPlayerState.h

```cpp
UCLASS()
class BLASTERLEARING_API ABlasterPlayerState : public APlayerState
{
 GENERATED_BODY()
 
public:
 virtual void GetLifetimeReplicatedProps(TArray< FLifetimeProperty >& OutLifetimeProps) const override;

 /*
 * 复制通知
 */
 virtual void OnRep_Score() override;

 UFUNCTION()
 virtual void OnRep_Defeats();

 void AddToScore(float ScoreAmount);
 void AddToDefeats(int32 DefeatsAmount);

private:
 UPROPERTY()
 class ABlasterCharacter* Character;
 UPROPERTY()
 class ABlasterPlayerController* Controller;

 UPROPERTY(ReplicatedUsing = OnRep_Defeats)
 int32 Defeats;

 UPROPERTY(ReplicatedUsing = OnRep_Team)
 ETeam Team = ETeam::ET_NoTeam;

 UFUNCTION()
 void OnRep_Team();

public:
 FORCEINLINE ETeam GetTeam() const { return Team; }
 void SetTeam(ETeam TeamToSet);
};
```

## BlasterPlayerState.cpp

### GetLifetimeReplicatedProps

```cpp
//复制参数注册。
void ABlasterPlayerState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
 Super::GetLifetimeReplicatedProps(OutLifetimeProps);

 DOREPLIFETIME(ABlasterPlayerState, Defeats);
 DOREPLIFETIME(ABlasterPlayerState, Team);
}
```

### OnRep_Score

```cpp
//分数复制通知。分数是玩家状态的参数。并且有一个可以重写的网络复制通知。我们只需对其进行重写。
void ABlasterPlayerState::OnRep_Score()
{
 Super::OnRep_Score();

//获得所有者角色。
 Character = Character == nullptr ? Cast<ABlasterCharacter>(GetPawn()) : Character;
 if (Character)
 {
    //获得所有者控制器。
  Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
  if (Controller)
  {
    //调用控制器的设置分数HUD。
   Controller->SetHUDScore(GetScore());
  }
 }
}
```

### OnRep_Defeats

```cpp
//死亡次数复制通知。
void ABlasterPlayerState::OnRep_Defeats()
{
    //获得所有者角色。
 Character = Character == nullptr ? Cast<ABlasterCharacter>(GetPawn()) : Character;
 if (Character)
 {
    //获得所有者角色控制器。
  Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
  if (Controller)
  {
    //调用控制器死亡次数HUD。
   Controller->SetHUDDefeats(Defeats);
  }
 }
}
```

### AddToScore

```cpp
//添加分数。
void ABlasterPlayerState::AddToScore(float ScoreAmount)
{
    //设置分数等于现在的分数加上增加的分数。
 SetScore(GetScore() + ScoreAmount);
 //获得所有者角色。
 Character = Character == nullptr ? Cast<ABlasterCharacter>(GetPawn()) : Character;
 if (Character)
 {
    //获得所有者控制器。
  Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
  if (Controller)
  {
    //调用控制器的设置分数HUD。
   Controller->SetHUDScore(GetScore());
  }
 }
}
```

### AddToDefeats

```cpp
//添加死亡次数。
void ABlasterPlayerState::AddToDefeats(int32 DefeatsAmount)
{
    //死亡次数等于加上增加的死亡次数。
 Defeats += DefeatsAmount;
 //获得所有者。
 Character = Character == nullptr ? Cast<ABlasterCharacter>(GetPawn()) : Character;
 if (Character)
 {
    //获得所有者控制器。
  Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
  if (Controller)
  {
    //调用控制器的设置死亡次数HUD。
   Controller->SetHUDDefeats(Defeats);
  }
 }
}
```

### SetTeam

```cpp
//设置团队。
void ABlasterPlayerState::SetTeam(ETeam TeamToSet)
{
    //玩家状态的团队等于传入的团队。
 Team = TeamToSet;

//获得的所有者角色。
 ABlasterCharacter* BCharacter = Cast<ABlasterCharacter>(GetPawn());
 if (BCharacter)
 {
    //设置角色的团队颜色。
  BCharacter->SetTeamColor(Team);
 }
}
```

### OnRep_Team

```cpp
//团队复制通知。
void ABlasterPlayerState::OnRep_Team()
{
    //获得所有者角色。
 ABlasterCharacter* BCharacter = Cast<ABlasterCharacter>(GetPawn());
 if (BCharacter)
 {
    //调用设置角色团队颜色。
  BCharacter->SetTeamColor(Team);
 }
}
```
