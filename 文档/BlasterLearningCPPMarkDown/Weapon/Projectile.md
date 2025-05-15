# Projectile

射弹的基类，所有射弹类武器发射的射弹都是继承自射弹基类。

## Projectile.h

```cpp
UCLASS()
class BLASTERLEARING_API AProjectile : public AActor
{
 GENERATED_BODY()
 
public: 
 AProjectile();
 virtual void Tick(float DeltaTime) override;
 virtual void Destroyed() override;

 /*
 * 与服务器端倒带一起使用
 */
 bool bUseServerSideRewind = false;
 FVector_NetQuantize TraceStart;
 FVector_NetQuantize100 InitialVelocity;

 UPROPERTY(EditAnywhere)
 float InitialSpeed = 15000;

 //用于蓝图覆盖范围伤害
 UPROPERTY(EditAnywhere)
 float Damage = 20.f;

 //只对于射弹武器的爆头伤害
 UPROPERTY(EditAnywhere)
 float HeadShotDamage = 40.f;

protected:
 virtual void BeginPlay() override;
 void StartDestroyTimer();
 void DestroyTimerFinished();
 void SpawnTrailSystem();
 void ExplodeDamage();

 UFUNCTION()
 virtual void OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit);

 UPROPERTY(EditAnywhere)
 class UParticleSystem* ImpactParticles;

 UPROPERTY(EditAnywhere)
 class USoundCue* ImpactSound;

 UPROPERTY(EditAnywhere)
 class UBoxComponent* CollisionBox;

 UPROPERTY(EditAnywhere)
 class UNiagaraSystem* TrailSystem;

 UPROPERTY()
 class UNiagaraComponent* TrailSystemComponent;

 UPROPERTY(VisibleAnywhere)
 class UProjectileMovementComponent* ProjectileMovementComponent;

 UPROPERTY(VisibleAnywhere)
 UStaticMeshComponent* ProjectileMesh;

 UPROPERTY(EditAnywhere)
 float DamageInnerRadius = 200.f;

 UPROPERTY(EditAnywhere)
 float DamageOutRadius = 500.f;

private: 
 UPROPERTY(EditAnywhere)
 UParticleSystem* Tracer;

 UPROPERTY()
 class UParticleSystemComponent* TracerComponent;

 FTimerHandle DestroyTimer;

 UPROPERTY(EditAnywhere)
 float DestroyTime = 3.f;

public:
};
```

## Projectile.cpp

###

```cpp
AProjectile::AProjectile()
{
//开启tick，开启网络复制。
 PrimaryActorTick.bCanEverTick = true;
 bReplicates = true;

//创建碰撞盒子组件。设置为根组件。设置子弹的碰撞类型为动态物体。开启碰撞检测。对所有通道的碰撞为忽略，对可见性，静态物体，骨骼通道设置为阻挡。
 CollisionBox = CreateDefaultSubobject<UBoxComponent>(TEXT("CollisionBox"));
 SetRootComponent(CollisionBox);
 CollisionBox->SetCollisionObjectType(ECollisionChannel::ECC_WorldDynamic);
 CollisionBox->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
 CollisionBox->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
 CollisionBox->SetCollisionResponseToChannel(ECollisionChannel::ECC_Visibility, ECollisionResponse::ECR_Block);
 CollisionBox->SetCollisionResponseToChannel(ECollisionChannel::ECC_WorldStatic, ECollisionResponse::ECR_Block);
 CollisionBox->SetCollisionResponseToChannel(ECC_SkeletalMesh, ECollisionResponse::ECR_Block);
}
```

### BeginPlay

```cpp
void AProjectile::BeginPlay()
{
    //调用父类开始函数。
 Super::BeginPlay();

//生成粒子特效。
 if (Tracer)
 {
  TracerComponent = UGameplayStatics::SpawnEmitterAttached(Tracer, CollisionBox, FName(), GetActorLocation(), GetActorRotation(), EAttachLocation::KeepWorldPosition);
 }

//如果是权威，动态绑定碰撞组件的回调函数OnHit。
 if (HasAuthority())
 {
  CollisionBox->OnComponentHit.AddDynamic(this, &AProjectile::OnHit);
 }
}
```

### AddDynamic

所有继承自Projectile类的射弹类都重写了OnHit函数，因为多态性和动态绑定，AddDynamic(this, &AProjectile::OnHit)这里实际绑定的回调函数是根据当前的实际类型，并且是否重写OnHit函数，这也是为什么在后续的多种射弹武器的子弹调用Super::BeginPlay却绑定的是自身重写OnHit函数，因为多态性覆盖了父类的OnHit函数，动态绑定导致实际的绑定是在运行时根据当前对象的实际类型进行绑定的。

### OnHit

```cpp
void AProjectile::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
{
    //调用销毁对象。
 Destroy();
}
```

### SpawnTrailSystem

```cpp
//生成特效。
void AProjectile::SpawnTrailSystem()
{
 if (TrailSystem)
 {
    //生成尾焰烟雾特效。
  TrailSystemComponent = UNiagaraFunctionLibrary::SpawnSystemAttached(TrailSystem,
   GetRootComponent(),
   FName(),
   GetActorLocation(),
   GetActorRotation(),
   EAttachLocation::KeepWorldPosition,
   false);
 }
}
```

### ExplodeDamage

```cpp
//爆炸伤害。
void AProjectile::ExplodeDamage()
{
    //获得发起者。
 APawn* FireingPawn = GetInstigator();
 //如果发起者存在并且时权威。
 if (FireingPawn && HasAuthority())
 {
    //获得发起者的控制器。
  AController* FireingController = FireingPawn->GetController();
  if (FireingController)
  {
    //施加一个爆炸伤害。
   UGameplayStatics::ApplyRadialDamageWithFalloff(
    this, //世界上下文对象
    Damage, //基础伤害
    10.f, //最小伤害
    GetActorLocation(), //爆炸伤害中心
    DamageInnerRadius, //爆炸内半径
    DamageOutRadius, //爆炸外半径
    1.f, //伤害衰减
    UDamageType::StaticClass(), //伤害类型
    TArray<AActor*>(), //无伤的对象数组
    this, //伤害造成者
    FireingController //发起者控制器
   );
  }
 }
}
```

### Tick

```cpp
//每帧更新。
void AProjectile::Tick(float DeltaTime)
{
 Super::Tick(DeltaTime);
}
```

### StartDestroyTimer

```cpp
//开始销毁计时器。
void AProjectile::StartDestroyTimer()
{
    //创建一个计时器，并绑定回调函数DestroyTimerFinished。
 GetWorldTimerManager().SetTimer(
  DestroyTimer,
  this,
  &AProjectile::DestroyTimerFinished,
  DestroyTime);
}
```

### DestroyTimerFinished

```cpp
//销毁计时器的回调函数。
void AProjectile::DestroyTimerFinished()
{
    //销毁对象。
 Destroy();
}
```

### Destroyed

```cpp
//在销毁对象时执行我们自定义的逻辑。
void AProjectile::Destroyed()
{
 Super::Destroyed();

 if (ImpactParticles)
 {
    //生成击中爆炸粒子特效。
  UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), ImpactParticles, GetActorLocation());
 }

 if (ImpactSound)
 {
    //播放爆炸声音。
  UGameplayStatics::PlaySoundAtLocation(this, ImpactSound, GetActorLocation());
 }
}
```
