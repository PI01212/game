# ProjectileRocket

火箭弹发射器。

## ProjectileRocket.h

```cpp
UCLASS()
class BLASTERLEARING_API AProjectileRocket : public AProjectile
{
 GENERATED_BODY()
public:
 AProjectileRocket();
 virtual void Destroyed() override;
 
protected:
 virtual void OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit) override;
 virtual void BeginPlay() override;

 UPROPERTY(EditAnywhere)
 USoundCue* ProjectileLoop;

 UPROPERTY()
 UAudioComponent* ProjectileLoopComponent;

 UPROPERTY(EditAnywhere)
 USoundAttenuation* LoopingSoundAttenuation;

 UPROPERTY(VisibleAnywhere)
 class URocketMovementComponent* RocketMovementComponent;

private:
};
```

## ProjectileRocket.cpp

### 构造函数

```cpp
AProjectileRocket::AProjectileRocket()
{
    //创建网格。添加到根组件。禁用碰撞。
 ProjectileMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Rocket Mesh"));
 ProjectileMesh->SetupAttachment(RootComponent);
 ProjectileMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);

//创建火箭弹移动组件，开启旋转跟随速度。开启网络复制同步。
 RocketMovementComponent = CreateDefaultSubobject<URocketMovementComponent>(TEXT("RocketMovementComponent"));
 RocketMovementComponent->bRotationFollowsVelocity = true;
 RocketMovementComponent->SetIsReplicated(true);
}
```

### BeginPlay

```cpp
void AProjectileRocket::BeginPlay()
{
 Super::BeginPlay();

//如果不是权威,绑定命中事件。这里命中事件主要是关于特效和声音方面绑定在玩家各自的客户端。因为火箭弹继承自Projectile，所以真正的在服务器的伤害命中计算绑定已经在父类中动态绑定。
 if (!HasAuthority())
 {
  CollisionBox->OnComponentHit.AddDynamic(this, &AProjectileRocket::OnHit);
 }

//生成轨迹特效系统。
 SpawnTrailSystem();

//如果循环音效资源和音效衰减设置存在。
 if (ProjectileLoop && LoopingSoundAttenuation)
 {
    //生成一个音效组件附加在根组件上。
  ProjectileLoopComponent = UGameplayStatics::SpawnSoundAttached(
   ProjectileLoop,
   GetRootComponent(),
   FName(),
   GetActorLocation(),
   EAttachLocation::KeepWorldPosition,
   false,
   1.f,
   1.f,
   0.f,
   LoopingSoundAttenuation,
   (USoundConcurrency*)nullptr,
   false
  );
 }
}
```

###

```cpp
void AProjectileRocket::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
{
    //如果击中自己就直接返回。
 if (OtherActor == GetOwner())
 {
  return;
 }
 //爆炸伤害。伤害只会在服务器上才会触发，在客户端条件判断不会通过。所以在客户端绑定只会如上面说的只是对特效声音之类的进行设置。
 ExplodeDamage();

//开始销毁计时器。
 StartDestroyTimer();

//生成碰撞粒子效果。
 if (ImpactParticles)
 {
  UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), ImpactParticles, GetActorLocation());
 }

//生成碰撞声音。
 if (ImpactSound)
 {
  UGameplayStatics::PlaySoundAtLocation(this, ImpactSound, GetActorLocation());
 }
 //设置网格不可见。
 if (ProjectileMesh)
 {
  ProjectileMesh->SetVisibility(false);
 }
 //禁用碰撞盒碰撞。
 if (CollisionBox)
 {
  CollisionBox->SetCollisionEnabled(ECollisionEnabled::NoCollision);
 }
 //停止生成轨迹粒子效果。
 if (TrailSystemComponent)
 {
  TrailSystemComponent->Deactivate(); // 停止粒子生成  
 }
 //如果音效循环组件还在播放停止播放。
 if (ProjectileLoopComponent && ProjectileLoopComponent->IsPlaying())
 {
  ProjectileLoopComponent->Stop();
 }
}
```

### Destroyed

```cpp
//火箭弹不执行父类中的销毁逻辑，所以重写销毁函数，但是不添加逻辑，相当于把销毁函数设置为空白。调用后直接销毁对象。
void AProjectileRocket::Destroyed()
{
}
```
