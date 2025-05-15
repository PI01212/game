# HitScanWeapon

射线类武器，和射弹类武器不同，射线类武器是直接进行射线追踪来进行射击，而不是发射子弹。手枪，狙击枪，霰弹枪，冲锋枪，都属于射线武器。

## HitScanWeapon.h

```cpp
UCLASS()
class BLASTERLEARING_API AHitScanWeapon : public AWeapon
{
 GENERATED_BODY()
 
public:
 virtual void Fire(const FVector& HitTarget) override;

protected:
 void WeaponTraceHit(const FVector& TraceStart, const FVector& HitTarget, FHitResult& OutHit);

 UPROPERTY(EditAnywhere)
 class UParticleSystem* ImpactParticles;

 UPROPERTY(EditAnywhere)
 USoundCue* HitSound;

private:
 UPROPERTY(EditAnywhere)
 UParticleSystem* BeamParticles;

 UPROPERTY(EditAnywhere)
 UParticleSystem* MuzzleFlash;

 UPROPERTY(EditAnywhere)
 USoundCue* FireSound;
};
```

## HitScanWeapon.cpp

### Fire

```cpp
//重写的Fire函数。
void AHitScanWeapon::Fire(const FVector& HitTarget)
{
 Super::Fire(HitTarget);

//获得所有者Pawn。如果不存在直接返回。
 APawn* OwnerPawn = Cast<APawn>(GetOwner());
 if (OwnerPawn == nullptr) return;
 //从所有者Pawn中获得控制器。
 AController* InstigatorController = OwnerPawn->GetController();

//获得开火枪口的插槽。
 const USkeletalMeshSocket* MuzzleFlashSocket = GetWeaponMesh()->GetSocketByName("MuzzleFlash");
 if (MuzzleFlashSocket)
 {
    //从武器网格中获得插槽的变换，插槽的位置设置为射线追踪的开始位置。
  FTransform SocketTransform = MuzzleFlashSocket->GetSocketTransform(GetWeaponMesh());
  FVector Start = SocketTransform.GetLocation();

//创建命中结果结构体。调用武器射线击中并传入开始和击中目标判断是否击中，结果存储在创建的FireHit中。
  FHitResult FireHit;
  WeaponTraceHit(Start, HitTarget, FireHit);

//从击中结果中获得命中角色判断是否是玩家角色类。
  ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(FireHit.GetActor());
  if (BlasterCharacter  && InstigatorController)
  {
    //是否由服务器计算伤害的判断依据是不使用服务器端倒带或者所有者角色是本地控制。
   bool bCauseAuthDamage = !bUseServerSideRewind || OwnerPawn->IsLocallyControlled();
   //在这里有两种情况，是否使用服务器计算一种是不使用服务器倒带，所以肯定是使用服务器计算伤害。而另一种是使用服务器倒带但武器所有者是本地控制。再加上判断一次是否是服务器，这就表示是服务器的本地控制，服务器的本地控制肯定也使用服务器计算因为服务器没有延迟。
   if (HasAuthority() && bCauseAuthDamage)
   {
    //判断命中的部位是否是头，是爆头就是用爆头伤害。
    const float DamageToCause = FireHit.BoneName.ToString() == FString("head") ? HeadShotDamage : Damage;

//施加伤害，传入被击中者，造成的伤害，发起者控制器，造成伤害的武器，伤害类型。
    UGameplayStatics::ApplyDamage(
     BlasterCharacter,
     DamageToCause,
     InstigatorController,
     this,
     UDamageType::StaticClass()
    );
   }
   //如果不是权威，并且使用服务器倒带，那就绝对要使用服务器倒带去施加伤害。
   if (!HasAuthority() && bUseServerSideRewind)
   {
    //获得所有者角色，所有者控制器。
    BlasterOwnerCharacter = BlasterOwnerCharacter == nullptr ? Cast<ABlasterCharacter>(OwnerPawn) : BlasterOwnerCharacter;
    BlasterOwnerController = BlasterOwnerController == nullptr ? Cast<ABlasterPlayerController>(InstigatorController) : BlasterOwnerController;
    //如果能够获取所有者角色的延迟补偿组件并且是本地控制。
    if (BlasterOwnerController && BlasterOwnerCharacter && BlasterOwnerCharacter->GetLagCompensation() && BlasterOwnerCharacter->IsLocallyControlled())
    {
        //那就调用延迟补偿组件的服务器分数请求。传入击中角色，开始位置，击中位置，击中时的服务器时间。
     BlasterOwnerCharacter->GetLagCompensation()->ServerScoreRequest(
      BlasterCharacter,
      Start,
      HitTarget,
      BlasterOwnerController->GetServerTime() - BlasterOwnerController->SingleTripTime
     );
    }
   }
  }
  //生成命中例子效果。
  if (ImpactParticles)
  {
   UGameplayStatics::SpawnEmitterAtLocation(
    GetWorld(),
    ImpactParticles,
    FireHit.ImpactPoint,
    FireHit.ImpactNormal.Rotation()
   );
  }
  //生成命中声音。
  if (HitSound)
  {
   UGameplayStatics::PlaySoundAtLocation(
    this,
    HitSound,
    FireHit.ImpactPoint
   );
  }
  //生成枪口火光。
  if (MuzzleFlash)
  {
   UGameplayStatics::SpawnEmitterAtLocation(
    GetWorld(),
    MuzzleFlash,
    SocketTransform
   );
  }
  //生成开火声音。
  if (FireSound)
  {
   UGameplayStatics::PlaySoundAtLocation(
    this,
    FireSound,
    GetActorLocation()
   );
  }
 }
}
```

###

```cpp
//武器射线击中。
void AHitScanWeapon::WeaponTraceHit(const FVector& TraceStart, const FVector& HitTarget, FHitResult& OutHit)
{
 UWorld* World = GetWorld();
 if (World)
 {
    //射线命中检测结束等于开始位置加上命中位置减去开始位置乘以1.25.f。
  FVector End = TraceStart + (HitTarget - TraceStart) * 1.25f;
  //调用单通道射线检测。传入开始位置和结束位置，通道选择可见性通道。结果传入OutHit中。
  World->LineTraceSingleByChannel(
   OutHit,
   TraceStart,
   End,
   ECollisionChannel::ECC_Visibility
  );
  //射线终点位置等于检测结束位置。
  FVector BeamEnd = End;
  //如果输出击中中有受到命中阻挡。代表击中对象。
  if (OutHit.bBlockingHit)
  {
    //设置射线终点等于击中点。
   BeamEnd = OutHit.ImpactPoint;
  }
  else
  {
    //如果没有命中就设置为射线结束位置，防止结果为世界原点，使得武器指向世界原点。
   OutHit.ImpactPoint = End;
  }

//如果射线粒子效果存在。生成射线粒子效果。
  if (BeamParticles)
  {
   UParticleSystemComponent* Beam = UGameplayStatics::SpawnEmitterAtLocation(
    World,
    BeamParticles,
    TraceStart,
    FRotator::ZeroRotator,
    true
   );
   if (Beam)
   {
    //设置粒子效果的目标参数为射线终点位置。
    Beam->SetVectorParameter(FName("Target"), BeamEnd);
   }
  }
 }
}
```
