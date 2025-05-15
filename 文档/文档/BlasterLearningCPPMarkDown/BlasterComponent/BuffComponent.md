# BuffComponent

角色和Buff拾取小部件进行交互的组件。

## BuffComponent.h

```cpp
UCLASS( ClassGroup=(Custom), meta=(BlueprintSpawnableComponent) )
class BLASTERLEARING_API UBufferComponent : public UActorComponent
{
 GENERATED_BODY()

public: 
 UBufferComponent();
 friend class ABlasterCharacter;
 void Heal(float HealAmount, float HealingTime);
 void ReplenishShield(float ShieldAmount, float ReplenishTime);
 void BuffSpeed(float BuffBaseSpeed, float BuffCrouchSpeed, float BuffTime);
 void SetInitialSpeeds(float BaseSpeed, float CrouchSpeed);
 void BuffJump(float BuffJumpVelocity, float BuffTime);
 void SetInitialJumpVelocity(float Velocity);

protected:
 virtual void BeginPlay() override;
 void HealRampUp(float DeltaTime);
 void ShieldRampUp(float DeltaTime);

private:
 UPROPERTY()
 class ABlasterCharacter* Character;

 /*
 * 生命值Buff
 */
 bool bHealing = false;
 float HealingRate = 0;
 float AmountToHeal = 0.f;

 /**
 * 护盾 buff
 */
 bool bReplenishingShield = false;
 float ShieldReplenishRate = 0.f;
 float ShieldReplenishAmount = 0.f;

 /*
 * 速度Buff
 */
 FTimerHandle SpeedBuffTimer;
 void ResetSpeeds();
 float InitialBaseSpeed;
 float InitialCrouchSpeed;

 UFUNCTION(NetMulticast, Reliable)
 void MulticastSpeedBuff(float BaseSpeed, float CrouchSpeed);

 /*
 * 跳跃Buff
 */
 FTimerHandle JumpBuffTimer;
 void ResetJump();
 float InitialJumpVelocity;
 UFUNCTION(NetMulticast, Reliable)
 void MulticastJumpBuff(float JumpVelocity);

public: 
 virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;
};
```

## BuffComponent.cpp

### 构造函数

```cpp
UBufferComponent::UBufferComponent()
{
    //开启Tick。
 PrimaryComponentTick.bCanEverTick = true;
}
```

### BeginPlay

```cpp
void UBufferComponent::BeginPlay()
{
 Super::BeginPlay();
}
```

### Heal

```cpp
//治疗。
void UBufferComponent::Heal(float HealAmount, float HealingTime)
{
    //设置为正在治疗。每秒治疗的生命值等于治疗量除以治疗时间。治疗的总量等于治疗量。
 bHealing = true;
 HealingRate = HealAmount / HealingTime;
 AmountToHeal += HealAmount;
}
```

### ReplenishShield

```cpp
//恢复护盾。
void UBufferComponent::ReplenishShield(float ShieldAmount, float ReplenishTime)
{
    //设置为正在恢复护盾，计算恢复速率，恢复的总量等于恢复量。
 bReplenishingShield = true;
 ShieldReplenishRate = ShieldAmount / ReplenishTime;
 ShieldReplenishAmount += ShieldAmount;
}
```

###

```cpp
//设置初始化速度。
void UBufferComponent::SetInitialSpeeds(float BaseSpeed, float CrouchSpeed)
{
    //初始化速度等于基础速度。初始化蹲下速度等于蹲下速度。
 InitialBaseSpeed = BaseSpeed;
 InitialCrouchSpeed = CrouchSpeed;
}
```

### BuffJump

```cpp
//跳跃Buff。
void UBufferComponent::BuffJump(float BuffJumpVelocity, float BuffTime)
{
    //如果角色为空直接返回。
 if (Character == nullptr) return;
 //创建角色的跳跃Buff持续时间计时器。绑定回调函数。
 Character->GetWorldTimerManager().SetTimer(
  JumpBuffTimer,
  this,
  &UBufferComponent::ResetJump,
  BuffTime
 );
 if (Character->GetCharacterMovement())
 {
    //设置角色的Z跳跃速度等于Buff跳跃速度。
  Character->GetCharacterMovement()->JumpZVelocity = BuffJumpVelocity;
 }
 //调用多播RPC同步跳跃Buff。
 MulticastJumpBuff(BuffJumpVelocity);
}
```

### ResetJump

```cpp
//重置跳跃。
void UBufferComponent::ResetJump()
{
 if (Character->GetCharacterMovement())
 {
    //设置角色的跳跃重置回原来的初始化跳跃速度。
  Character->GetCharacterMovement()->JumpZVelocity = InitialJumpVelocity;
 }
 //调用多播RPC同步跳跃。
 MulticastJumpBuff(InitialJumpVelocity);
}
```

### MulticastJumpBuff

```cpp
void UBufferComponent::MulticastJumpBuff_Implementation(float JumpVelocity)
{
 if (Character && Character->GetCharacterMovement())
 {
    //设置角色的Z跳跃速度等于Buff跳跃速度。
  Character->GetCharacterMovement()->JumpZVelocity = JumpVelocity;
 }
}
```

### SetInitialJumpVelocity

```cpp
void UBufferComponent::SetInitialJumpVelocity(float Velocity)
{
    //设置初始跳跃速度等于速度。
 InitialJumpVelocity = Velocity;
}
```

### HealRampUp

```cpp
//每帧逐渐治疗。
void UBufferComponent::HealRampUp(float DeltaTime)
{
    //如果不处于治疗状态或者角色已经死亡直接返回。
 if (!bHealing || Character == nullptr || Character->IsElimed()) return;

//这个帧的治疗量等于治疗速率乘以增量时间。
 const float HealThisFrame = HealingRate * DeltaTime;
 //设置角色的生命值等于当前角色的生命值加上这一帧治疗的生命值。更新角色的生命值HUD。治疗总量等于治疗总量减去这一帧治疗量。
 Character->SetHealth(FMath::Clamp(Character->GetHealth() + HealThisFrame, 0.f, Character->GetMaxHealth()));
 Character->UpdateHUDHealth();
 AmountToHeal -= HealThisFrame;

//如果治疗总量小于等于0，或者角色的生命值大于最大生命值。
 if (AmountToHeal <= 0.f || Character->GetHealth() >= Character->GetMaxHealth())
 {
    //设置为停止治疗，治疗总量为0。
  bHealing = false;
  AmountToHeal = 0.f;
 }
}
```

### ShieldRampUp

```cpp
//每帧逐渐恢复护盾。逻辑和生命值一样。
void UBufferComponent::ShieldRampUp(float DeltaTime)
{
 if (!bReplenishingShield || Character == nullptr || Character->IsElimed()) return;
 const float ReplenishThisFrame = ShieldReplenishRate * DeltaTime;
 Character->SetShield(FMath::Clamp(Character->GetShield() + ReplenishThisFrame, 0.f, Character->GetMaxShield()));
 Character->UpdateHUDShield();
 ShieldReplenishAmount -= ReplenishThisFrame;
 if (ShieldReplenishAmount <= 0.f || Character->GetShield() >= Character->GetMaxShield())
 {
  bReplenishingShield = false;
  ShieldReplenishAmount = 0.f;
 }
}
```

### BuffSpeed

```cpp
//速度Buff。
void UBufferComponent::BuffSpeed(float BuffBaseSpeed, float BuffCrouchSpeed, float BuffTime)
{
    //如果角色为空直接返回。
 if (Character == nullptr) return;
 //创建角色的速度Buff计时器。绑定回调函数。
 Character->GetWorldTimerManager().SetTimer(
  SpeedBuffTimer,
  this,
  &UBufferComponent::ResetSpeeds,
  BuffTime
 );
 if (Character->GetCharacterMovement())
 {
    //设置角色的最大速度等于速度Buff。角色的蹲下速度等于蹲下移动Buff。
  Character->GetCharacterMovement()->MaxWalkSpeed = BuffBaseSpeed;
  Character->GetCharacterMovement()->MaxWalkSpeedCrouched = BuffCrouchSpeed;
 }
 MulticastSpeedBuff(BuffBaseSpeed, BuffCrouchSpeed);
}
```

### ResetSpeeds

```cpp
//重置速度
void UBufferComponent::ResetSpeeds()
{
    //如果角色和移动组件为空直接返回。
 if (Character == nullptr || Character->GetCharacterMovement() == nullptr) return;
 //设置角色的最大速度和最大蹲下移动速度等于初始速度。
 Character->GetCharacterMovement()->MaxWalkSpeed = InitialBaseSpeed;
 Character->GetCharacterMovement()->MaxWalkSpeedCrouched = InitialCrouchSpeed;
 //调用多播RPC同步速度。
 MulticastSpeedBuff(InitialBaseSpeed, InitialCrouchSpeed);
}
```

### MulticastSpeedBuff

```cpp
void UBufferComponent::MulticastSpeedBuff_Implementation(float BaseSpeed, float CrouchSpeed)
{
    //同步客户端设置最大速度和蹲下移动速度等于初始速度。
 Character->GetCharacterMovement()->MaxWalkSpeed = BaseSpeed;
 Character->GetCharacterMovement()->MaxWalkSpeedCrouched = CrouchSpeed;
}
```

### TickComponent

```cpp
void UBufferComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
 Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
 //每帧调用逐渐恢复生命值和护盾方法，当拾取Buff小部件后，生命值治疗总量和护盾值恢复总量发生变化就立马开始进行治疗和恢复。
 HealRampUp(DeltaTime);
 ShieldRampUp(DeltaTime);
}
```
