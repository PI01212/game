# Shotgun

霰弹枪继承自射线武器类，因为霰弹枪一次需要打出几发弹丸，所以需要对打出的所有弹丸做射线追踪检测。因此需要重写实现霰弹枪自己的方法。

## Shotgun.h

```cpp
UCLASS()
class BLASTERLEARING_API AShotgun : public AHitScanWeapon
{
 GENERATED_BODY()
 
public:
 virtual void FireShotgun(const TArray<FVector_NetQuantize>& HitTargets);
 void ShotgunTraceEndWithScatter(const FVector& HitTarget, TArray<FVector_NetQuantize>& HitTargets);

private:
 UPROPERTY(EditAnywhere, Category = "Weapon Scatter")
 uint32 NumberOfPellets = 10;
};
```

## Shotgun.cpp

### FireShotgun

```cpp
void AShotgun::FireShotgun(const TArray<FVector_NetQuantize>& HitTargets)
{
    //调用基类武器的开火函数传入空向量。
 AWeapon::Fire(FVector());
 //获得所有者类Pawn。如果不存在直接返回。
 APawn* OwnerPawn = Cast<APawn>(GetOwner());
 if (OwnerPawn == nullptr) return;
 //从Pawn中获得发起者的控制器。
 AController* InstigatorController = OwnerPawn->GetController();

//得到开火枪口插槽。
 const USkeletalMeshSocket* MuzzleFlashSocket = GetWeaponMesh()->GetSocketByName("MuzzleFlash");
 if (MuzzleFlashSocket)
 {
    //获得插槽的变换。位置设置为射线开始位置。
  const FTransform SocketTransform = MuzzleFlashSocket->GetSocketTransform(GetWeaponMesh());
  const FVector Start = SocketTransform.GetLocation();

  //map中用来存储被击中的角色和击中的弹丸数量
  TMap<ABlasterCharacter*, uint32> HitMap;
  TMap<ABlasterCharacter*, uint32> HeadShotHitMap;
  //循环传入的命中目标数组，对每一个命中目标进行射线命中检测。
  for (FVector_NetQuantize HitTarget : HitTargets)
  {
   FHitResult FireHit;
   WeaponTraceHit(Start, HitTarget, FireHit);

//从命中结果中获得命中的角色是否是玩家角色类。
   ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(FireHit.GetActor());
   if (BlasterCharacter)
   {
    //如果命中的骨骼是头部，就代表爆头。
    const float bHeadShot = FireHit.BoneName.ToString() == FString("head");

    if (bHeadShot)
    {
        //如果在爆头Map中有当前的角色，那就给当前角色的命中次数+1。
     if (HeadShotHitMap.Contains(BlasterCharacter)) HeadShotHitMap[BlasterCharacter]++;
     //没有就添加当前角色的键，值为1。
     else HeadShotHitMap.Emplace(BlasterCharacter, 1);
    }
    else
    //没有命中头部就说明击中的角色的其他部位不使用爆头伤害。
    {
        //如果在命中Map中有当前的角色，那就给当前角色的命中次数+1。
     if (HitMap.Contains(BlasterCharacter)) HitMap[BlasterCharacter]++;
     //没有就添加当前角色的键，值为1。
     else HitMap.Emplace(BlasterCharacter, 1);
    }


    if (ImpactParticles)
    {
        //生成粒子效果。
     UGameplayStatics::SpawnEmitterAtLocation(
      GetWorld(),
      ImpactParticles,
      FireHit.ImpactPoint,
      FireHit.ImpactNormal.Rotation()
     );
    }
    if (HitSound)
    {
        //播放命中声音。
     UGameplayStatics::PlaySoundAtLocation(
      this,
      HitSound,
      FireHit.ImpactPoint,
      0.5f,
      FMath::FRandRange(-0.5f, 0.5f)
     );
    }
   }
  }
  //创建命中角色的数组。
  TArray<ABlasterCharacter*> HitCharacters;

  //角色被击中的伤害计算
  TMap<ABlasterCharacter*, float> DamageMap;

  //循环计算角色身体受到的伤害
  for (auto HitPair : HitMap)
  {
    //判断Key存在，如果不判断直接使用Key可能会造成断言。程序会直接停止。
   if (HitPair.Key)
   {
    //在伤害Map中添加受击中的角色和其受到的伤害。在受击中的角色数组中添加此角色。
    DamageMap.Emplace(HitPair.Key, HitPair.Value * Damage);
    HitCharacters.AddUnique(HitPair.Key);
   }
  }

  //循环计算角色受到的爆头伤害
  for (auto HeadShotHitPair : HeadShotHitMap)
  {
   if (HeadShotHitPair.Key)
   {
    //如果伤害Map中有当前角色，就把角色受到的伤害值加上爆头次数乘以爆头伤害。
    if (DamageMap.Contains(HeadShotHitPair.Key)) DamageMap[HeadShotHitPair.Key] += HeadShotHitPair.Value * HeadShotDamage;
    //没有就直接添加当前角色Key和伤害值。
    else DamageMap.Emplace(HeadShotHitPair.Key, HeadShotHitPair.Value * HeadShotDamage);

//命中角色中添加此角色。
    HitCharacters.AddUnique(HeadShotHitPair.Key);
   }
  }

  //循环实际对受到伤害的角色造成伤害
  for (auto DamagePair : DamageMap)
  {
    //如果受到伤害的角色和发起者控制器都存在。
   if (DamagePair.Key && InstigatorController)
   {
    //原理和射线武器中的条件一致。
    bool bCauseAuthDamage = !bUseServerSideRewind || OwnerPawn->IsLocallyControlled();
    if (HasAuthority() && bCauseAuthDamage)
    {
        //施加伤害。
     UGameplayStatics::ApplyDamage(
      DamagePair.Key, //被击中的角色
      DamagePair.Value, //造成从上面循环得到的伤害之和
      InstigatorController,
      this,
      UDamageType::StaticClass()
     );
    }
   }
  }

//使用服务器倒带造成伤害。
  if (!HasAuthority() && bUseServerSideRewind)
  {
   BlasterOwnerCharacter = BlasterOwnerCharacter == nullptr ? Cast<ABlasterCharacter>(OwnerPawn) : BlasterOwnerCharacter;
   BlasterOwnerController = BlasterOwnerController == nullptr ? Cast<ABlasterPlayerController>(InstigatorController) : BlasterOwnerController;
   if (BlasterOwnerController && BlasterOwnerCharacter && BlasterOwnerCharacter->GetLagCompensation() && BlasterOwnerCharacter->IsLocallyControlled())
   {
    //使用所有者角色的延迟补偿组件调用霰弹枪服务器分数请求。
    BlasterOwnerCharacter->GetLagCompensation()->ShotgunServerScoreRequest(
     HitCharacters,
     Start,
     HitTargets,
     BlasterOwnerController->GetServerTime() - BlasterOwnerController->SingleTripTime
    );
   }
  }
 }
}
```

### ShotgunTraceEndWithScatter

```cpp
//霰弹枪射线散射。
void AShotgun::ShotgunTraceEndWithScatter(const FVector& HitTarget, TArray<FVector_NetQuantize>& HitTargets)
{
    //获得枪口开火插槽。
 const USkeletalMeshSocket* MuzzleFlashSocket = GetWeaponMesh()->GetSocketByName("MuzzleFlash");
 if (MuzzleFlashSocket == nullptr) return;

//获得插槽的变换。变换的位置作为射线检测开始位置。
 const FTransform SocketTransform = MuzzleFlashSocket->GetSocketTransform(GetWeaponMesh());
 const FVector TraceStart = SocketTransform.GetLocation();

//命中目标减去开始位置归一化作为方向。球体中心等于射线开始位置减去方向乘以DistanceToSphere。
 const FVector ToTargetNormalized = (HitTarget - TraceStart).GetSafeNormal();
 const FVector SphereCenter = TraceStart + ToTargetNormalized * DistanceToSphere;
 
 //根据一发霰弹枪的弹丸数进行循环。原理和射线追踪武器的散射一样，最后把结果添加到命中目标数组当中。
 for (uint32 i = 0; i < NumberOfPellets; i++)
 {
  const FVector RandVec = UKismetMathLibrary::RandomUnitVector() * FMath::FRandRange(0.f, SphereRadius);
  const FVector EndLoc = SphereCenter + RandVec;
  FVector ToEndLoc = EndLoc - TraceStart;
  ToEndLoc = TraceStart + ToEndLoc * TRACE_LENGTH / ToEndLoc.Size();

  HitTargets.Add(ToEndLoc);
 }
}
```
