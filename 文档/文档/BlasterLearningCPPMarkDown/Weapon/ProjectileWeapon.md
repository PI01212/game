# ProjectileWeapon

射弹类武器，继承自基础Weapon类，重写了相应的方法实现射弹类武器专属的功能。

## ProjectileWeapon.h

```cpp
UCLASS()
class BLASTERLEARING_API AProjectileWeapon : public AWeapon
{
 GENERATED_BODY()
 
public:
 virtual void Fire(const FVector& HitTarget) override;

private:
 UPROPERTY(EditAnywhere)
 TSubclassOf<class AProjectile> ProjectileClass;

 UPROPERTY(EditAnywhere)
 TSubclassOf<AProjectile> ServerSideRewindProjectileClass;
};
```

## ProjectileWeapon.cpp

### Fire

```cpp
//重写的开火函数。
void AProjectileWeapon::Fire(const FVector& HitTarget)
{
 Super::Fire(HitTarget);

//获得发起者角色。
 APawn* InstigatorPawn = Cast<APawn>(GetOwner());
 //获得枪口开火插槽。
  const USkeletalMeshSocket* MuzzleFlashSocket = GetWeaponMesh()->GetSocketByName(FName("MuzzleFlash"));
  UWorld* World = GetWorld();
  if (MuzzleFlashSocket && World)
  {
    //通过武器网格获得插槽变换。
   FTransform SocketTransform = MuzzleFlashSocket->GetSocketTransform(GetWeaponMesh());
   //目标位置减去枪口位置得来中间向量，再得到这个向量的方向作为子弹方向。
   FVector ToTarget = HitTarget - SocketTransform.GetLocation();
   FRotator TargetRotation = ToTarget.Rotation();

//创建生成对象结构体，设置结构体的所有者为武器的所有者，设置发起者为发起者角色。
   FActorSpawnParameters SpawnParams;
   SpawnParams.Owner = GetOwner();
   SpawnParams.Instigator = InstigatorPawn;

//创建生成的子弹设置为空指针。
   AProjectile* SpawnedProjectile = nullptr;
   //如果使用服务器倒带。
   if (bUseServerSideRewind)
   {
    if (InstigatorPawn->HasAuthority())//在服务器
    {
     if (InstigatorPawn->IsLocallyControlled())//服务器控制 生成复制子弹
     {
        //生成复制子弹，传入插槽变换的位置，旋转，生成对象结构体。
      SpawnedProjectile = World->SpawnActor<AProjectile>(ProjectileClass, SocketTransform.GetLocation(), TargetRotation, SpawnParams);
      SpawnedProjectile->bUseServerSideRewind = false;
      SpawnedProjectile->Damage = Damage;
      SpawnedProjectile->HeadShotDamage = HeadShotDamage;
     }
     else//服务器但不是本地控制,生成不复制的子弹，使用倒带
     {
      SpawnedProjectile = World->SpawnActor<AProjectile>(ServerSideRewindProjectileClass, SocketTransform.GetLocation(), TargetRotation, SpawnParams);
      SpawnedProjectile->bUseServerSideRewind = true;
     }
    }
    else//客户端使用倒带
    {
     if (InstigatorPawn->IsLocallyControlled())//客户端本地控制,生成不复制的子弹，使用倒带
     {
      SpawnedProjectile = World->SpawnActor<AProjectile>(ServerSideRewindProjectileClass, SocketTransform.GetLocation(), TargetRotation, SpawnParams);
      SpawnedProjectile->bUseServerSideRewind = true;
      SpawnedProjectile->TraceStart = SocketTransform.GetLocation();
      SpawnedProjectile->InitialVelocity = SpawnedProjectile->GetActorForwardVector() * SpawnedProjectile->InitialSpeed;
      //SpawnedProjectile->Damage = Damage;//其实写不写无所谓
     }
     else//客户端不是本地控制,生成一个不复制子弹不使用倒带
     {
      SpawnedProjectile = World->SpawnActor<AProjectile>(ServerSideRewindProjectileClass, SocketTransform.GetLocation(), TargetRotation, SpawnParams);
      SpawnedProjectile->bUseServerSideRewind = false;
     }
    }
   }
   else//武器不使用倒带
   {
      //如果是在服务器上。
    if (InstigatorPawn->HasAuthority())
    {
      //生成一个复制子弹不使用服务器倒带。
     SpawnedProjectile = World->SpawnActor<AProjectile>(ProjectileClass, SocketTransform.GetLocation(), TargetRotation, SpawnParams);
     SpawnedProjectile->bUseServerSideRewind = false;
     SpawnedProjectile->Damage = Damage;
     SpawnedProjectile->HeadShotDamage = HeadShotDamage;
    }
   }
  }
}
```
