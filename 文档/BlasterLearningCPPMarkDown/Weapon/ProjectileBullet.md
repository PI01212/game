# ProjectileBullet

步枪使用的子弹，继承自Projectile类。

## ProjectileBullet.h

```cpp
UCLASS()
class BLASTERLEARING_API AProjectileBullet : public AProjectile
{
 GENERATED_BODY()
 
public:
 AProjectileBullet();

#if WITH_EDITOR
 virtual void PostEditChangeProperty(struct FPropertyChangedEvent& Event) override;
#endif

protected:
 virtual void OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit) override;
 virtual void BeginPlay() override;
};
```

## ProjectileBullet.cpp

### 构造函数

```cpp
AProjectileBullet::AProjectileBullet()
{
    //创建子弹类移动组件。开启旋转跟随速度，开启网络复制同步，设置初速度等于设置的初速度，最大速度等于初速度。旋转跟随速度方向，初速度和最大速度。
 ProjectileMovementComponent = CreateDefaultSubobject<UProjectileMovementComponent>(TEXT("ProjectileMovementComponent"));
 ProjectileMovementComponent->bRotationFollowsVelocity = true;
 ProjectileMovementComponent->SetIsReplicated(true);
 ProjectileMovementComponent->InitialSpeed =InitialSpeed;
 ProjectileMovementComponent->MaxSpeed = InitialSpeed;
}
```

###

```cpp
//在编辑器中修改初始化速度时，自动同步修改移动组件的初始化速度。
#if WITH_EDITOR
void AProjectileBullet::PostEditChangeProperty(FPropertyChangedEvent& Event)
{
 Super::PostEditChangeProperty(Event);

//创建类成员的FName，从事件中获取，如果不为空代表有成员变量被修改，获得FName。
 FName PropertyName = Event.Property != nullptr ? Event.Property->GetFName() : NAME_None;
 //如果修改的是初始化速度。
 if (PropertyName == GET_MEMBER_NAME_CHECKED(AProjectileBullet, InitialSpeed))
 {
    //同步更新移动组件中的初始化速度和最大速度。
  if (ProjectileMovementComponent)
  {
   ProjectileMovementComponent->InitialSpeed = InitialSpeed;
   ProjectileMovementComponent->MaxSpeed = InitialSpeed;
  }
 }
}
#endif
```

### OnHit

```cpp
//命中回调函数。
void AProjectileBullet::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
{
    //获得所有者角色。
 ABlasterCharacter* OwnerCharacter = Cast<ABlasterCharacter>(GetOwner());
 if (OwnerCharacter)
 {
    //获得所有者控制器。
  ABlasterPlayerController* OwnerController = Cast<ABlasterPlayerController>(OwnerCharacter->Controller);
  if (OwnerController)
  {
    //如果控制器是权威，不使用服务器倒带。
   if (OwnerCharacter->HasAuthority() && !bUseServerSideRewind)
   {
    //造成伤害等于是否是爆头，爆头就用爆头伤害不然就是基本伤害。
    const float DamageToCause = Hit.BoneName.ToString() == FString("head") ? HeadShotDamage : Damage;

//施加伤害。调用父类OnHit直接返回。
    UGameplayStatics::ApplyDamage(OtherActor, DamageToCause, OwnerController, this, UDamageType::StaticClass());
    Super::OnHit(HitComp, OtherActor, OtherComp, NormalImpulse, Hit);
    return;
   }
   //获得被击中的角色。
   ABlasterCharacter* HitCharacter = Cast<ABlasterCharacter>(OtherActor);
   //如果使用服务器倒带并获得所有者的延迟补偿组件。所有者是本地控制。代表现在是在一个玩家控制的客户端上并使用服务器倒带。
   if (bUseServerSideRewind && OwnerCharacter->GetLagCompensation() && OwnerCharacter->IsLocallyControlled() && HitCharacter)
   {
    //调用射弹服务器分数请求。
    OwnerCharacter->GetLagCompensation()->ProjectileServerScoreRequest(
     HitCharacter,
     TraceStart,
     InitialVelocity,
     OwnerController->GetServerTime() - OwnerController->SingleTripTime
    );
   }
  }
 }
 //调用父类的命中回调函数。
 Super::OnHit(HitComp, OtherActor, OtherComp, NormalImpulse, Hit);
}
```

### BeginPlay

```cpp
//调用父类的开始函数。
void AProjectileBullet::BeginPlay()
{
 Super::BeginPlay();
}
```
