# ProjectileGrenade

榴弹类，武器榴弹发射器发射的炮弹，角色扔出的手雷蓝图也是继承此类实现。

## ProjectileGrenade.h

```cpp
UCLASS()
class BLASTERLEARING_API AProjectileGrenade : public AProjectile
{
 GENERATED_BODY()
 
public:
 AProjectileGrenade();
 virtual void Destroyed() override;

protected:
 virtual void BeginPlay() override;

 UFUNCTION()
 void OnBounce(const FHitResult& ImpactResult, const FVector& ImpactVelocity);

private:
 UPROPERTY(EditAnywhere)
 USoundCue* BounceSound;
};
```

## ProjectileGrenade.cpp

### AProjectileGrenade

```cpp
AProjectileGrenade::AProjectileGrenade()
{
    //创建网格组件设置为根组件。禁用碰撞。
 ProjectileMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Grenade Mesh"));
 ProjectileMesh->SetupAttachment(RootComponent);
 ProjectileMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);

//创建移动组件，设置物体的方向一致跟随速度方向，开启网络复制同步，设置物体碰撞可以反弹。
 ProjectileMovementComponent = CreateDefaultSubobject<UProjectileMovementComponent>(TEXT("ProjectileMovementComponent"));
 ProjectileMovementComponent->bRotationFollowsVelocity = true;
 ProjectileMovementComponent->SetIsReplicated(true);
 ProjectileMovementComponent->bShouldBounce = true;
}
```

### BeginPlay

```cpp
void AProjectileGrenade::BeginPlay()
{
 AActor::BeginPlay();

//生成轨迹系统和开启销毁计时器。
 SpawnTrailSystem();
 StartDestroyTimer();

//动态绑定反弹事件。
 ProjectileMovementComponent->OnProjectileBounce.AddDynamic(this, &AProjectileGrenade::OnBounce);
}
```

### OnBounce

```cpp
void AProjectileGrenade::OnBounce(const FHitResult& ImpactResult, const FVector& ImpactVelocity)
{
    //生成反弹的声音。
 if (BounceSound)
 {
  UGameplayStatics::PlaySoundAtLocation(
   this,
   BounceSound,
   GetActorLocation()
  );
 }
}
```

### Destroyed

```cpp
void AProjectileGrenade::Destroyed()
{
    //调用爆炸伤害。
 ExplodeDamage();

 Super::Destroyed();
}
```
