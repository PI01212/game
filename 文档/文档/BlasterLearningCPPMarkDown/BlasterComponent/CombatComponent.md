# CombatComponent

战斗组件主要负责游戏关于战斗方面的各种功能的实现，例如武器开火，换弹，手雷，角色携带的弹药等相关功能的实现。

## CombatComponent.h

```cpp
UCLASS( ClassGroup=(Custom), meta=(BlueprintSpawnableComponent) )
class BLASTERLEARING_API UCombatComponent : public UActorComponent
{
 GENERATED_BODY()

public: 
 UCombatComponent();
 friend class ABlasterCharacter;
 virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;
 virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

 void EquipWeapon(class AWeapon* WeaponToEquip);
 void SwapWeapons();
 void Reload();
 UFUNCTION(BlueprintCallable)
 void FinishReloading();

 UFUNCTION(BlueprintCallable)
 void FinishSwap();

 UFUNCTION(BlueprintCallable)
 void FinishSwapAttachWeapons();

 void FireButtonPressed(bool bPressed);

 UFUNCTION(BlueprintCallable)
 void ShotgunShellReload();

 void JumpToShotgunEnd();

 UFUNCTION(BlueprintCallable)
 void ThrowGrenadeFinished();

 UFUNCTION(BlueprintCallable)
 void LaunchGrenade();

 UFUNCTION(Server, Reliable)
 void ServerLaunchGrenade(const FVector_NetQuantize& Target);

 void PickupAmmo(EWeaponType WeaponType, int32 AmmoAmount);
 bool bLocallyReloading = false;

protected:
 virtual void BeginPlay() override;
 void SetAiming(bool bIsAiming);

 UFUNCTION(Server, Reliable)
 void ServerSetAiming(bool bIsAiming);

 UFUNCTION()
 void OnRep_EquippedWeapon();

 UFUNCTION()
 void OnRep_SecondaryWeapon();

 void Fire();
 void FireProjectileWeapon();
 void FireHitScanWeapon();
 void FireShotgun();
 void LocalFire(const FVector_NetQuantize& TraceHitTarget);
 void ShotgunLocalFire(const TArray<FVector_NetQuantize>& TraceHitTargets);

 UFUNCTION(Server, Reliable, WithValidation)
 void ServerFire(const FVector_NetQuantize& TraceHitTarget, float FireDelay);

 UFUNCTION(NetMulticast, Reliable)
 void MulticastFire(const FVector_NetQuantize& TraceHitTarget);

 UFUNCTION(Server, Reliable, WithValidation)
 void ServerShotgunFire(const TArray<FVector_NetQuantize>& TraceHitTargets, float FireDelay);

 UFUNCTION(NetMulticast, Reliable)
 void MulticastShotgunFire(const TArray<FVector_NetQuantize>& TraceHitTargets);

 void TraceUnderCrosshairs(FHitResult& TraceHitResult);

 void SetHUDCrosshairs(float DeltaTime);

 UFUNCTION(Server, Reliable)
 void ServerReload();

 void HandleReload();
 int32 AmountToReload();

 void ThrowGrenade();

 UFUNCTION(Server, Reliable)
 void ServerThrowGrenade();

 UPROPERTY(EditAnywhere)
 TSubclassOf<class AProjectile> GrenadeClass;

 void DropEquippedWeapon();
 void AttachActorToRightHand(AActor* ActorToAttach);
 void AttachActorToLeftHand(AActor* ActorToAttach);
 void AttachFlagToLeftHand(AActor* Flag);
 void AttachActorToBackpack(AActor* ActorToAttach);
 void UpdateCarriedAmmo();
 void PlayEquipWeaponSound(AWeapon* WeaponToEquip);
 void ReloadEmptyWeapon();
 void ShowAttachedGrenade(bool bShowGrenade);
 void EquipPrimaryWeapon(AWeapon* WeaponToEquip);
 void EquipSecondaryWeapon(AWeapon* WeaponToEquip);

private:
 UPROPERTY()
 class ABlasterCharacter* Character;
 UPROPERTY()
 class ABlasterPlayerController* Controller;
 UPROPERTY()
 class ABlasterHUD* HUD;

 UPROPERTY(ReplicatedUsing = OnRep_EquippedWeapon)
 class AWeapon* EquippedWeapon;

 UPROPERTY(ReplicatedUsing = OnRep_SecondaryWeapon)
 AWeapon* SecondaryWeapon;

 UPROPERTY(ReplicatedUsing = OnRep_Aiming)
 bool bAiming = false;

 bool bAimButtonPressed = false;

 UFUNCTION()
 void OnRep_Aiming();

 UPROPERTY(EditAnywhere)
 float BaseWalkSpeed;

 UPROPERTY(EditAnywhere)
 float AimWalkSpeed;

 bool bFireButtonPressed;

 /*
 * HUD and crosshairs
 */
 float CrosshairVelocityFactor;
 float CrosshairInAirFactor;
 float CrosshairAimFactor;
 float CrosshairShootingFactor;

 FVector HitTarget;

 FHUDPackage HUDPackage;

 /*
 * 瞄准和视野
 */

 //不瞄准时的默认视野,设置为游玩时相机的基础视野
 float DefaultFOV;

 UPROPERTY(EditAnywhere, Category = Combat)
 float ZoomedFOV = 30.f;

 float CurrentFOV;

 UPROPERTY(EditAnywhere, Category = Combat)
 float ZoomInterpSpeed = 20.f;

 void InterpFOV(float DeltaTime);

 /*
 * 自动开火
 */

 FTimerHandle FireTimer;
 bool bCanFire = true;

 void StartFireTimer();
 void FireTimerFinished();

 bool CanFire();

 //当前装备武器携带的弹药
 UPROPERTY(ReplicatedUsing = OnRep_CarriedAmmo)
 int32 CarriedAmmo;

 UFUNCTION()
 void OnRep_CarriedAmmo();

 TMap<EWeaponType, int32> CarriedAmmoMap;

 UPROPERTY(EditAnywhere)
 int32 MaxCarriedAmmo = 500;

 UPROPERTY(EditAnywhere)
 int32 StartingARAmmo = 30;

 UPROPERTY(EditAnywhere)
 int32 StartingRocketAmmo = 0;

 UPROPERTY(EditAnywhere)
 int32 StartingPistolAmmo = 0;

 UPROPERTY(EditAnywhere)
 int32 StartingSMGAmmo = 0;

 UPROPERTY(EditAnywhere)
 int32 StartingShotgunAmmo = 0;

 UPROPERTY(EditAnywhere)
 int32 StartingSniperAmmo = 0;

 UPROPERTY(EditAnywhere)
 int32 StartingGrenadeAmmo = 0;

 void InitializeCarriedAmmo();

 UPROPERTY(ReplicatedUsing = OnRep_CombatState)
 ECombatState CombatState = ECombatState::ECS_Unoccuiped;

 UFUNCTION()
 void OnRep_CombatState();

 void UpdateAmmoValues();

 void UpdateShotgunAmmoValues();

 UPROPERTY(ReplicatedUsing = OnRep_Grenades)
 int32 Grenades = 4;

 UFUNCTION()
 void OnRep_Grenades();

 UPROPERTY(EditAnywhere)
 int32 MaxGrenades = 4;

 void UpdateHUDGrenade();

 UPROPERTY(ReplicatedUsing = OnRep_HoldingTheFlag)
 bool bHoldingTheFlag = false;

 UFUNCTION()
 void OnRep_HoldingTheFlag();

 UPROPERTY(ReplicatedUsing = OnRep_Flag)
 AWeapon* TheFlag;

 UFUNCTION()
 void OnRep_Flag();

public: 
 FORCEINLINE int32 GetGrenades() const { return Grenades; }
 bool ShouldSwapWeapons();
};
```

## CombatComponent.cpp

### 构造函数

```cpp
//构造函数，设置初始速度和瞄准速度。
UCombatComponent::UCombatComponent()
{
 PrimaryComponentTick.bCanEverTick = true;

 BaseWalkSpeed = 600.f;
 AimWalkSpeed = 450.f;
}
```

### BeginPlay

```cpp
void UCombatComponent::BeginPlay()
{
 Super::BeginPlay();

 if (Character)
 {
   //获得角色的移动组件设置最大移动速度等于设置的基础移动速度。
  Character->GetCharacterMovement()->MaxWalkSpeed = BaseWalkSpeed;

  if (Character->GetFollowCamera())
  {
   //从角色的相机中获得默认视野并存储起来。设置当前视野等于默认视野。
   DefaultFOV = Character->GetFollowCamera()->FieldOfView;
   CurrentFOV = DefaultFOV;
  }
  if (Character->HasAuthority())
  {
   //如果角色是权威的调用初始化备用弹药。
   InitializeCarriedAmmo();
  }
 }
}
```

### InitializeCarriedAmmo

```cpp
//对角色的携带弹药Map添加对应武器类型的键值对弹药，实际添加的值在蓝图中设置覆盖C++类。
void UCombatComponent::InitializeCarriedAmmo()
{
 CarriedAmmoMap.Emplace(EWeaponType::EWT_AssaultRifle, StartingARAmmo);
 CarriedAmmoMap.Emplace(EWeaponType::EWT_RocketLauncher, StartingRocketAmmo);
 CarriedAmmoMap.Emplace(EWeaponType::EWT_Pistol, StartingPistolAmmo);
 CarriedAmmoMap.Emplace(EWeaponType::EWT_SMG, StartingSMGAmmo);
 CarriedAmmoMap.Emplace(EWeaponType::EWT_Shotgun, StartingShotgunAmmo);
 CarriedAmmoMap.Emplace(EWeaponType::EWT_SniperRifle, StartingSniperAmmo);
 CarriedAmmoMap.Emplace(EWeaponType::EWT_GrenadeLauncher, StartingGrenadeAmmo);
}
```

### TickComponent

```cpp
//帧组件，用于没帧更新。
void UCombatComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
 Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

//如果角色是本地控制。
 if (Character && Character->IsLocallyControlled())
 {
   //创建碰撞检测结果。
  FHitResult HitResult;
  //调用准星射线检测把结果保存在刚才创建的检测结果当中。击中目标设置为检测结果当中的碰撞点。
  TraceUnderCrosshairs(HitResult);
  HitTarget = HitResult.ImpactPoint;

//调用更新准星HUD方法和视野插值方法。
  SetHUDCrosshairs(DeltaTime);
  InterpFOV(DeltaTime);
 }
}
```

### TraceUnderCrosshairs

```cpp
void UCombatComponent::TraceUnderCrosshairs(FHitResult& TraceHitResult)
{
   //创建2D向量窗口大小。
 FVector2D ViewportSize;
 if (GEngine && GEngine->GameViewport)
 {
  //使用引擎获取窗口的大小。
  GEngine->GameViewport->GetViewportSize(ViewportSize);
 }

//设置准星的中位置为屏幕的中心。创建准星世界位置和方向向量。
 FVector2D CrosshairLocation(ViewportSize.X / 2.f, ViewportSize.Y / 2.f);
 FVector CrosshairWorldPosition;
 FVector CrosshairWorldDirection;
 //获取当前玩家的控制器，通过游戏静态类的DeprojectScreenToWorld的方法把屏幕准星位置转换为世界中的位置和方向返回是否成功。
 bool bScreenToWorld = UGameplayStatics::DeprojectScreenToWorld(
  UGameplayStatics::GetPlayerController(this, 0),
  CrosshairLocation,
  CrosshairWorldPosition,
  CrosshairWorldDirection
 );

//如果转换成功
 if (bScreenToWorld)
 {
  //射线开始位置等于准星世界位置。
  FVector Start = CrosshairWorldPosition;

  if (Character)
  {
    //求出射线开始位置到角色本身的距离，并让射线开始位置沿着准星的世界方向前进原本的距离加上100.f。这样做的目的是防止射线击中角色本身。
   float DistanceToCharacter = (Character->GetActorLocation() - Start).Size();
   Start += CrosshairWorldDirection * (DistanceToCharacter + 100.f);
  }

//射线结束位置等于开始位置加上准星世界方向乘以设置的射线长度宏。
  FVector End = Start + CrosshairWorldDirection * TRACE_LENGTH;

//获得世界对象调用射线检测方法，使用开始位置和结束位置，设置检测通道为可见性通道，把结果传入检测击中结果中并返回是否击中。
  bool bHIt = GetWorld()->LineTraceSingleByChannel(
   TraceHitResult,
   Start,
   End,
   ECollisionChannel::ECC_Visibility
  );
  //如果击中返回为true代表击中目标。
  if (bHIt)
  {
    //获得击中角色并且判断角色是否继承接口。继承了接口代表击中的角色就是BlasterCharacter类，因为只有这个类设置了继承接口。
   if (TraceHitResult.GetActor() && TraceHitResult.GetActor()->Implements<UInteractWithCrosshairsInterface>())
   {
    //击中就把准星变为红色。
    HUDPackage.CrosshairsColor = FLinearColor::Red;
   }
   else
   {
    //不是角色类就变为白色。
    HUDPackage.CrosshairsColor = FLinearColor::White;
   }
  }
  else
  {
    //如果没击中就手动把击中点设置为结束位置。如果不手动修改，射线检测就会去瞄准世界原点。
   TraceHitResult.ImpactPoint = FVector_NetQuantize(End);
  }
 }
}
```

### SetHUDCrosshairs

```cpp
//设置准星HUD。
void UCombatComponent::SetHUDCrosshairs(float DeltaTime)
{
  //如果角色和角色的控制器为空直接返回。
 if (Character == nullptr || Character->Controller == nullptr) return;

//从角色获得控制器。
 Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
 if (Controller)
 {
  //从控制器获得角色HUD。
  HUD = HUD == nullptr ? Cast<ABlasterHUD>(Controller->GetHUD()) : HUD;
  if (HUD)
  {
    //如果装备了武器。
   if (EquippedWeapon)
   {
    //设置HUD包，把准星设置为装备武器的准星。
    HUDPackage.CrosshairsCenter = EquippedWeapon->CrosshairsCenter;
    HUDPackage.CrosshairsLeft = EquippedWeapon->CrosshairsLeft;
    HUDPackage.CrosshairsRight = EquippedWeapon->CrosshairsRight;
    HUDPackage.CrosshairsBottom = EquippedWeapon->CrosshairsBottom;
    HUDPackage.CrosshairsTop = EquippedWeapon->CrosshairsTop;
   }
   else
   {
    //如果没装备就全部设置为空指针。
    HUDPackage.CrosshairsCenter = nullptr;
    HUDPackage.CrosshairsLeft = nullptr;
    HUDPackage.CrosshairsRight = nullptr;
    HUDPackage.CrosshairsBottom = nullptr;
    HUDPackage.CrosshairsTop = nullptr;
   }
   // 计算准心散布

   // [0, 600] -> [0, 1]
   //创建速度的范围0到600，准星速度系数范围0到1。获得当前速度。
   FVector2D WalkSpeedRange(0.f, Character->GetCharacterMovement()->MaxWalkSpeed);
   FVector2D VelocityMultiplierRange(0.f, 1.f);
   FVector Velocity = Character->GetVelocity();
   Velocity.Z = 0.f;

  //把速度映射到准星缩放范围的，并通过当前速度获得当前映射的值成为准星速度系数。
   CrosshairVelocityFactor = FMath::GetMappedRangeValueClamped(WalkSpeedRange, VelocityMultiplierRange, Velocity.Size());

  //如果角色处于空中。
   if (Character->GetCharacterMovement()->IsFalling())
   {
    //把准星空中系数插值到2.25f。
    CrosshairInAirFactor = FMath::FInterpTo(CrosshairInAirFactor, 2.25f, DeltaTime, 2.25f);
   }
   else
   {
    //落地后就把准星空中系数迅速查知道0。
    CrosshairInAirFactor = FMath::FInterpTo(CrosshairInAirFactor, 0.f, DeltaTime, 30.f);
   }

   if (bAiming)
   {
    //如果角色在瞄准就把准星瞄准系数插值到0.58.f。
    CrosshairAimFactor = FMath::FInterpTo(CrosshairAimFactor, 0.58f, DeltaTime, 30.f);
   }
   else
   {
    //不再瞄准就把准星瞄准系数插值到0。
    CrosshairAimFactor = FMath::FInterpTo(CrosshairAimFactor, 0.f, DeltaTime, 30.f);
   }
  //准星开火系数插值为0。准星开火系数的设置在开火函数中设置，这样就变成了只要开火就会变大，不开火就会每帧都尝试去插值回0。
   CrosshairShootingFactor = FMath::FInterpTo(CrosshairShootingFactor, 0.f, DeltaTime, 40.f);
  
  //准星的扩散程度等于0.5f的基础上加上这些系数之和，减去准星瞄准系数是因为瞄准时准星应该变小才符合。
   HUDPackage.CrosshairSpread =
    0.5f +
    CrosshairVelocityFactor +
    CrosshairInAirFactor -
    CrosshairAimFactor +
    CrosshairShootingFactor;
  
  //传入设置好的HUDPackage调用SetHUDPackage。
   HUD->SetHUDPackage(HUDPackage);
  }
 }
}
```

### InterpFOV

```cpp
//插值视野，灵活改变视野。
void UCombatComponent::InterpFOV(float DeltaTime)
{
  //如果没有装备武器直接返回。
 if (EquippedWeapon == nullptr) return;

 if (bAiming)
 {
  //如果角色在瞄准，就把当前视野插值为装备武器的视野，插值速度为装备武器的插值速度。
  CurrentFOV = FMath::FInterpTo(CurrentFOV, EquippedWeapon->GetZoomedFOV(), DeltaTime, EquippedWeapon->GetZoomedInterpSpeed());
 }
 else
 {
  //不再瞄准就把当前视野插值回默认视野，插值速度为默认插值速度。
  CurrentFOV = FMath::FInterpTo(CurrentFOV, DefaultFOV, DeltaTime, ZoomInterpSpeed);
 }
 if (Character && Character->GetFollowCamera())
 {
  //修改当前角色的视野为当前视野。
  Character->GetFollowCamera()->SetFieldOfView(CurrentFOV);
 }
}
```

### GetLifetimeReplicatedProps

```cpp
//对复制参数进行注册。
void UCombatComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
 Super::GetLifetimeReplicatedProps(OutLifetimeProps);

 DOREPLIFETIME(UCombatComponent, EquippedWeapon);
 DOREPLIFETIME(UCombatComponent, SecondaryWeapon);
 DOREPLIFETIME(UCombatComponent, bAiming);
 DOREPLIFETIME_CONDITION(UCombatComponent, CarriedAmmo, COND_OwnerOnly);
 DOREPLIFETIME(UCombatComponent, CombatState);
 DOREPLIFETIME(UCombatComponent, Grenades);
 DOREPLIFETIME(UCombatComponent, bHoldingTheFlag);
 DOREPLIFETIME(UCombatComponent, TheFlag);
}
```

### ShotgunShellReload

```cpp
//霰弹枪换弹。在换弹蒙太奇中，霰弹枪换弹的动画中，有四次上弹的动作，所以在每个上弹的帧添加了一个Shell通知，其会调用此函数给霰弹枪上一发子弹。一次动画总共触发四次。所以霰弹枪的容弹量也设置为了四枚。
void UCombatComponent::ShotgunShellReload()
{
    //如果角色是权威的，代表在服务器上，那就调用更新霰弹枪的换弹函数，因为实际更改武器子弹的操作应该只在服务器上进行，来保证权威和客户端之间的同步。
 if (Character && Character->HasAuthority())
 {
  UpdateShotgunAmmoValues();

 }
}
```

### UpdateShotgunAmmoValues

```cpp
//更新霰弹枪弹药
void UCombatComponent::UpdateShotgunAmmoValues()
{
    //如果没有携带武器直接返回或者携带的类型不是霰弹枪，直接返回。
 if (Character == nullptr || EquippedWeapon == nullptr) return;
 if (CarriedAmmoMap.Contains(EquippedWeapon->GetWeaponType()))
 {
    //角色对应携带的霰弹枪弹药Map-1，因为霰弹枪是一发一发上子弹。角色携带的备用弹药等于变更后的Map。
  CarriedAmmoMap[EquippedWeapon->GetWeaponType()] -= 1;
  CarriedAmmo = CarriedAmmoMap[EquippedWeapon->GetWeaponType()];
 }
 //获取角色的控制器。
 Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
 if (Controller)
 {
    //调用更新备用弹药HUD方法，传入新的备弹数。
  Controller->SetHUDCarriedAmmo(CarriedAmmo);
 }
 //调用武器的增加子弹方法增加一枚子弹。
 EquippedWeapon->AddAmmo(1);
 //因为霰弹枪只要有一枚子弹就可以开火，所以当上了一枚子弹之后代表霰弹枪绝对可以开火了。所以设置能够开火为true。
 bCanFire = true;
 if (EquippedWeapon->IsFull() || CarriedAmmo == 0)
 {
    //如果霰弹枪装满或者备用弹药为空，代表霰弹枪不需换弹了，但霰弹枪有四次上弹，所以每次都有可能出现这种情况，那么当出现这种情况之后就要立马跳转动画到设置的动画结束末尾。
  JumpToShotgunEnd();
 }
}
```

### JumpToShotgunEnd

```cpp
//跳转动画到霰弹枪结束。
void UCombatComponent::JumpToShotgunEnd()
{
 // 跳转到动画上弹结束
 UAnimInstance* AnimInstance = Character->GetMesh()->GetAnimInstance();
 if (AnimInstance && Character->GetReloadMontage())
 {
    //跳转到动画的ShotgunEnd片段。
  AnimInstance->Montage_JumpToSection(FName("ShotgunEnd"));
 }
}
```

### PickupAmmo

```cpp
//游戏中设置了Buff和弹药两种拾取道具，捡起子弹方法是和弹药拾取道具重叠触发的函数，会增加对应武器的携弹量。传入参数为武器类型和子弹数。
void UCombatComponent::PickupAmmo(EWeaponType WeaponType, int32 AmmoAmount)
{
    //如果在携弹Map中有对应的武器类型。
 if (CarriedAmmoMap.Contains(WeaponType))
 {
    //对应类型的携弹Map的值等于本身加上传入的子弹数，并且限制在0到最大携弹量之间。调用更新携弹量方法。
  CarriedAmmoMap[WeaponType] = FMath::Clamp(CarriedAmmoMap[WeaponType] + AmmoAmount, 0, MaxCarriedAmmo);
  UpdateCarriedAmmo();
 }
 //如果携带的武器子弹为空并且携带的类型等于增加弹药的武器类型。
 if (EquippedWeapon && EquippedWeapon->IsEmpty() && EquippedWeapon->GetWeaponType() == WeaponType)
 {
    //调用装弹方法。
  Reload();
 }
}
```

### UpdateCarriedAmmo

```cpp
void UCombatComponent::UpdateCarriedAmmo()
{
    //如果武器为空就提前返回。
 if (EquippedWeapon == nullptr) return;
 //如果携弹Map的类型和装备的武器类型一致，使角色备用弹药等于对应装备武器的携弹Map。
 if (CarriedAmmoMap.Contains(EquippedWeapon->GetWeaponType()))
 {
  CarriedAmmo = CarriedAmmoMap[EquippedWeapon->GetWeaponType()];
 }

//获得角色控制器。
 Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
 //如果控制器存在，调用更新备用弹药HUD方法。
 if (Controller)
 {
  Controller->SetHUDCarriedAmmo(CarriedAmmo);
 }
}
```

### Reload

```cpp
//换弹。
void UCombatComponent::Reload()
{
    //如果备用弹药大于0，战斗状态等于空闲，装备的武器没装满，本地装弹中等于false。
 if (CarriedAmmo > 0 && CombatState == ECombatState::ECS_Unoccuiped && !EquippedWeapon->IsFull() && !bLocallyReloading)
 {
    //调用服务器换弹和本地换弹方法，设置本地换弹中等于true。
  ServerReload();
  HandleReload();
  bLocallyReloading = true;
 }
}
```

### ServerReload

```cpp
//服务器换弹RPC，当客户端调用换弹会发送服务器RPC换弹，去申请更新武器弹药。
void UCombatComponent::ServerReload_Implementation()
{
   //如果角色没有装备武器就返回。
 if (Character == nullptr || EquippedWeapon == nullptr) return;

//改变战斗状态等于换弹。
 CombatState = ECombatState::ECS_Reloading;
 //如果角色不是本地控制就调用本地换弹方法。
 if (!Character->IsLocallyControlled()) HandleReload();
}
```

### HandleReload

```cpp
//本地换弹。为什么角色要有一个本地换弹和服务器换弹，因为如果在延迟很高的情况，角色按下换弹按键需要很长时间才能到达服务器，服务器处理完还得传回数据，那玩家需要等待才能响应，那么就会很割裂，所以就会直接播放换弹动画但是实际更新武器弹药则还是服务器在处理。
void UCombatComponent::HandleReload()
{
 if (Character)
 {
   //播放角色的换弹动画。
  Character->PlayReloadMontage();
 }
}
```

### FireButtonPressed

```cpp
//开火按键按下。
void UCombatComponent::FireButtonPressed(bool bPressed)
{
  //开火按键是否按下等于传入的值。
 bFireButtonPressed = bPressed;
 if (bFireButtonPressed)
 {
  //如果开火为true，调用开火函数。
  Fire();
 }
}
```

### Fire

```cpp
void UCombatComponent::Fire()
{
  //如果可以开火就执行开火逻辑。
 if (CanFire())
 {
  //设置能够开火等于false。
  bCanFire = false;
  if (EquippedWeapon)
  {
    //设置准星开火系数等于0.75f扩散准星。
   CrosshairShootingFactor = 0.75f;
  
  //判断武器的类型，根据武器的类型调用不同的开火。
   switch (EquippedWeapon->FireType)
   {
   case EFireType::EFT_Projectile:
    FireProjectileWeapon();
    break;
   case EFireType::EFT_HitScan:
    FireHitScanWeapon();
    break;
   case EFireType::EFT_Shotgun:
    FireShotgun();
    break;
   }
  }
  //调用开火计时器，武器都有开火速度限制也就是射速不同。
  StartFireTimer();
 }
}
```

### CanFire

```cpp
bool UCombatComponent::CanFire()
{
  //如果没有装备武器直接返回false。
 if (EquippedWeapon == nullptr) return false;
 //如果武器弹药不为空，并且开火为true，战斗状态等于换弹，武器类型等于霰弹枪。就返回true。这个判断的是对于霰弹枪，因为霰弹枪只要有子弹即使是换弹中也可以开火。
 if (!EquippedWeapon->IsEmpty() && bCanFire && CombatState == ECombatState::ECS_Reloading && EquippedWeapon->GetWeaponType() == EWeaponType::EWT_Shotgun) return true;
 //如果本地换弹中等于true返回false。
 if (bLocallyReloading) return false;
 //对于其他武器，武器弹药不为空，能开火为true，战斗状态等于空闲。那就返回true。
 return !EquippedWeapon->IsEmpty() && bCanFire && CombatState == ECombatState::ECS_Unoccuiped;
}
```

### FireProjectileWeapon

```cpp
//射弹武器开火。
void UCombatComponent::FireProjectileWeapon()
{
  //如果角色装备武器。
 if (EquippedWeapon && Character)
 {
  //如果装备的武器使用散射，那就传入命中目标调用武器的散射射线检测计算新的命中目标，如果不使用就维持原有的击中目标。
  HitTarget = EquippedWeapon->bUseScatter ? EquippedWeapon->TraceEndWithScatter(HitTarget) : HitTarget;
  //如果角色不具有权威调用本地开火传入击中目标。再调用服务器开火，传入命中目标和开火延迟。这样执行是因为，如果延迟过大，那玩家的开火响应就太慢了，所以在本地直接调用本地开火，执行开火动画。然后再去调用服务器开火执行一些同步操作。及时响应玩家的操作。
  if (!Character->HasAuthority()) LocalFire(HitTarget);
  ServerFire(HitTarget, EquippedWeapon->FireDelay);
 }
}
```

### FireHitScanWeapon

```cpp
//执行逻辑和射弹武器一致。
void UCombatComponent::FireHitScanWeapon()
{
 if (EquippedWeapon && Character)
 {
  HitTarget = EquippedWeapon->bUseScatter ? EquippedWeapon->TraceEndWithScatter(HitTarget) : HitTarget;
  if (!Character->HasAuthority()) LocalFire(HitTarget);
  ServerFire(HitTarget, EquippedWeapon->FireDelay);
 }
}
```

### FireShotgun

```cpp
void UCombatComponent::FireShotgun()
{
  //获得霰弹枪对象。
 AShotgun* Shotgun = Cast<AShotgun>(EquippedWeapon);
 if (Shotgun && Character)
 {
  //霰弹枪一次射出多发弹丸，所以需要创建一个数组类型的击中目标保存所有的击中目标，使用FVector_NetQuantiz是为了网络同步，这个类型专门为网络同步使用，后续需要做到霰弹枪的散射逻辑要在服务器和客户端全部保证一致。
  TArray<FVector_NetQuantize> HitTargets;
  //调用霰弹枪的散射射线检测传入击中目标和击中目标数组。
  Shotgun->ShotgunTraceEndWithScatter(HitTarget, HitTargets);
  //如果不是权威角色调用霰弹枪本地开火。再调用服务器霰弹枪开火。
  if (!Character->HasAuthority()) ShotgunLocalFire(HitTargets);
  ServerShotgunFire(HitTargets, EquippedWeapon->FireDelay);
 }
}
```

### LocalFire

```cpp
//本地开火函数。
void UCombatComponent::LocalFire(const FVector_NetQuantize& TraceHitTarget)
{
  //如果没装备武器直接返回。
 if (EquippedWeapon == nullptr) return;
 //如果战斗状态等于空闲。
 if (Character && CombatState == ECombatState::ECS_Unoccuiped)
 {
  //根据传入的是否瞄准播放开火动画。调用武器的开火函数。
  Character->PlayFireMontage(bAiming);
  EquippedWeapon->Fire(TraceHitTarget);
 }
}
```

### ShotgunLocalFire

```cpp
//本地霰弹枪开火。
void UCombatComponent::ShotgunLocalFire(const TArray<FVector_NetQuantize>& TraceHitTargets)
{
  //获得霰弹枪对象。如果为空直接返回。
 AShotgun* Shotgun = Cast<AShotgun>(EquippedWeapon);
 if (Shotgun == nullptr || Character == nullptr) return;
 //如果战斗状态等于换弹或者空闲。就代表可以开火。
 if (CombatState == ECombatState::ECS_Reloading || CombatState == ECombatState::ECS_Unoccuiped)
 {
  //是否本地换弹中等于false。传入是否瞄准调用开火蒙太奇动画。调用霰弹枪的霰弹枪开火，战斗状态设置为空闲。
  bLocallyReloading = false;
  Character->PlayFireMontage(bAiming);
  Shotgun->FireShotgun(TraceHitTargets);
  CombatState = ECombatState::ECS_Unoccuiped;
 }
}
```

### ServerFire

```cpp
//服务器开火RPC。
void UCombatComponent::ServerFire_Implementation(const FVector_NetQuantize& TraceHitTarget, float FireDelay)
{
  //传入射线击中目标调用多播开火RPC。
 MulticastFire(TraceHitTarget);
}
```

### MulticastFire

```cpp
//多播开火。服务器调用去发送请求让客户端执行开火。保证所有客户端都开火。
void UCombatComponent::MulticastFire_Implementation(const FVector_NetQuantize& TraceHitTarget)
{
  //如果角色是本地控制并且不权威代表是当前客户端的玩家，已经执行了一次本地开火所以直接返回。否则调用本地开火。
 if (Character && Character->IsLocallyControlled() && !Character->HasAuthority()) return;
 LocalFire(TraceHitTarget);
}
```

### ServerShotgunFire

```cpp
//服务器霰弹枪开火RPC，调用多播RPC开火。
void UCombatComponent::ServerShotgunFire_Implementation(const TArray<FVector_NetQuantize>& TraceHitTargets, float FireDelay)
{
 MulticastShotgunFire(TraceHitTargets);
}
```

### MulticastShotgunFire

```cpp
//多播霰弹枪开火RPC。
void UCombatComponent::MulticastShotgunFire_Implementation(const TArray<FVector_NetQuantize>& TraceHitTargets)
{
  //如果角色是本地控制并且不权威代表是当前客户端的玩家，已经执行了一次本地霰弹枪开火所以直接返回。否则调用霰弹枪本地开火。
 if (Character && Character->IsLocallyControlled() && !Character->HasAuthority()) return;
 ShotgunLocalFire(TraceHitTargets);
}
```

### StartFireTimer

```cpp
//开始开火计时器。
void UCombatComponent::StartFireTimer()
{
  //使用TimerManager创建一个开火计时器，绑定回调函数FireTimerFinished，计时时间为武器的开火间隔。
 if (EquippedWeapon == nullptr || Character == nullptr) return;
 Character->GetWorldTimerManager().SetTimer(
  FireTimer,
  this,
  &UCombatComponent::FireTimerFinished,
  EquippedWeapon->FireDelay
 );
}
```

### FireTimerFinished

```cpp
//计时器结束时调用的回调函数。
void UCombatComponent::FireTimerFinished()
{
  //如果武器为空就返回。
 if (EquippedWeapon == nullptr) return;
 //设置能够开火为true。
 bCanFire = true;
 //如果开火按钮还在按着，武器类型是自动武器。
 if (bFireButtonPressed && EquippedWeapon->bAutomatic)
 {
  //继续调用开火函数。
  Fire();
 }
 //调用弹药射完重新装弹方法。
 ReloadEmptyWeapon();
}
```

### ReloadEmptyWeapon

```cpp
void UCombatComponent::ReloadEmptyWeapon()
{
  //如果装备武器并且装备的武器弹药为空。
 if (EquippedWeapon && EquippedWeapon->IsEmpty())
 {
  //调用换弹方法。
  Reload();
 }
}
```

### EquipWeapon

```cpp
//装备武器方法。
void UCombatComponent::EquipWeapon(AWeapon* WeaponToEquip)
{
  //角色没有装备武器就直接返回。
 if (Character == nullptr || WeaponToEquip == nullptr) return;
 //如果角色的状态不等于空闲代表正在执行其他操作，所以直接返回。
 if (CombatState != ECombatState::ECS_Unoccuiped) return;
 //调用装备武器的多播RPC,设置装备武器的弹药。
 WeaponToEquip->MulticastAmmo(WeaponToEquip->GetAmmo());

//如果装备的武器等于举旗。
 if (WeaponToEquip->GetWeaponType() == EWeaponType::EWT_Flag)
 {
  //调用角色的蹲下方法，让角色的胶囊组件变矮。设置正在举旗等于true。设置武器的状态为装备，添加装备的旗子到左手。设置旗子的拥有者。旗子等于当前装备的旗子。
  Character->Crouch();
  bHoldingTheFlag = true;
  WeaponToEquip->SetWeaponState(EWeaponState::EWS_Equipped);
  AttachFlagToLeftHand(WeaponToEquip);
  WeaponToEquip->SetOwner(Character);
  TheFlag = WeaponToEquip;
 }
 else
 //如果不是旗子那就是武器。
 {
  //如果角色装备武器不等于空并且第二把武器等于空。代表角色现在服务器为空。
  if (EquippedWeapon != nullptr && SecondaryWeapon == nullptr)
  {
    //调用装备服务器。
   EquipSecondaryWeapon(WeaponToEquip);
  }
  else
  {
    //否则要么就是角色没有装备武器，要么就是装备两把武器，直接调用装备武器。
   EquipPrimaryWeapon(WeaponToEquip);
  }

//当角色装备武器之后，就需要设定不跟随自己转向和使用控制器旋转，因为需要固定角色的视角进行射击。
  Character->GetCharacterMovement()->bOrientRotationToMovement = false;
  Character->bUseControllerRotationYaw = true;
 }
}
```

### AttachFlagToLeftHand

```cpp
//添加旗子到左手。
void UCombatComponent::AttachFlagToLeftHand(AActor* Flag)
{
  //如果角色和旗子都为空直接返回。
 if (Character == nullptr || Character->GetMesh() == nullptr || Flag == nullptr) return;
 //获得角色网格中旗子插槽。
 const USkeletalMeshSocket* HandSocket = Character->GetMesh()->GetSocketByName(FName("FlagSocket"));
 //如果旗子插槽存在，把武器添加到旗子插槽上。
 if (HandSocket)
 {
  HandSocket->AttachActor(Flag, Character->GetMesh());
 }
}
```

### EquipPrimaryWeapon

```cpp
void UCombatComponent::EquipPrimaryWeapon(AWeapon* WeaponToEquip)
{
  //如果去装备的武器为空直接返回。
 if (WeaponToEquip == nullptr) return;
 //调用放下武器。
 DropEquippedWeapon();
 //把装备的武器变成即将装备武器。
 EquippedWeapon = WeaponToEquip;
 //调用装备武器的设置武器状态方法设置为装备，把武器添加到角色的右手上，设置武器的拥有者和弹药HUD。
 EquippedWeapon->SetWeaponState(EWeaponState::EWS_Equipped);
 AttachActorToRightHand(EquippedWeapon);
 EquippedWeapon->SetOwner(Character);
 EquippedWeapon->SetHUDAmmo();
 //调用更新备用弹药方法，播放要装备武器的装备声音，调用武器弹药为空换弹方法，保证当装备武器弹药为空时自动换弹。
 UpdateCarriedAmmo();
 PlayEquipWeaponSound(WeaponToEquip);
 ReloadEmptyWeapon();
}
```

### DropEquippedWeapon

```cpp
void UCombatComponent::DropEquippedWeapon()
{
 if (EquippedWeapon)
 {
  //调用装备武器的掉落方法。
  EquippedWeapon->Dropped();
 }
}
```

### AttachActorToRightHand

```cpp
//添加到角色的右手。
void UCombatComponent::AttachActorToRightHand(AActor* ActorToAttach)
{
  //如果角色和添加的网格都为空直接返回。
 if (Character == nullptr || Character->GetMesh() == nullptr || ActorToAttach == nullptr) return;
 //获得右手插槽，调用添加到角色方法。
 const USkeletalMeshSocket* HandSocket = Character->GetMesh()->GetSocketByName(FName("RightHandSocket"));
 if (HandSocket)
 {
  HandSocket->AttachActor(ActorToAttach, Character->GetMesh());
 }
}
```

### PlayEquipWeaponSound

```cpp
//播放装备武器声音。
void UCombatComponent::PlayEquipWeaponSound(AWeapon* WeaponToEquip)
{
  //如果装备的武器和声音都存在调用播放声音在当前位置方法。
 if (Character && WeaponToEquip && WeaponToEquip->EquipSound)
 {
  UGameplayStatics::PlaySoundAtLocation(
   this,
   WeaponToEquip->EquipSound,
   Character->GetActorLocation()
  );
 }
}
```

### EquipSecondaryWeapon

```cpp
//装备副武器。
void UCombatComponent::EquipSecondaryWeapon(AWeapon* WeaponToEquip)
{
  //如果要装备的武器为空直接返回。
 if (WeaponToEquip == nullptr) return;
 //第二把武器等于要装备的武器。
 SecondaryWeapon = WeaponToEquip;
 //设置第二把武器的状态等于装备的第二把武器。
 SecondaryWeapon->SetWeaponState(EWeaponState::EWS_EquippedSecondary);
 //添加要装备的武器到角色。播放装备武器的声音，调用武器的设置所有者方法。
 AttachActorToBackpack(WeaponToEquip);
 PlayEquipWeaponSound(WeaponToEquip);
 SecondaryWeapon->SetOwner(Character);
}
```

### AttachActorToBackpack

```cpp
//添加武器的背包。
void UCombatComponent::AttachActorToBackpack(AActor* ActorToAttach)
{
  //如果网格都为空就返回。
 if (Character == nullptr || Character->GetMesh() == nullptr || ActorToAttach == nullptr) return;
 //获得背包的网格插槽，调用添加武器到插槽。
 const USkeletalMeshSocket* BackpackSocket = Character->GetMesh()->GetSocketByName(FName("BackpackSocket"));
 if (BackpackSocket)
 {
  BackpackSocket->AttachActor(ActorToAttach, Character->GetMesh());
 }
}
```

### SwapWeapons

```cpp
//调用切换武器。
void UCombatComponent::SwapWeapons()
{
  //如果战斗状态不等于空闲或者角色为空或者角色不是权威。直接返回。
 if (CombatState != ECombatState::ECS_Unoccuiped || Character == nullptr || !Character->HasAuthority()) return;

//调用角色切换武器蒙太奇。设置战斗状态等于切换武器，设置角色完成切换武器为false。
 Character->PlaySwapMontage();
 CombatState = ECombatState::ECS_SwappingWeapon;
 Character->bFinishedSwapping = false;
 //如果第二把武器存在，调用武器的深度为false，不再发光。
 if (SecondaryWeapon) SecondaryWeapon->EnableCustomDepth(false);
}
```

### FinishReloading

```cpp
//完成换弹。
void UCombatComponent::FinishReloading()
{
  //如果角色为空直接返回。
 if (Character == nullptr) return;
 //设置本地换弹为true。
 bLocallyReloading = false;
 //如果角色是权威。
 if (Character->HasAuthority())
 {
  //战斗组件等于空闲。调用更新子弹数量方法。
  CombatState = ECombatState::ECS_Unoccuiped;
  UpdateAmmoValues();
 }
 //如果开火按钮按下就调用开火方法。
 if (bFireButtonPressed)
 {
  Fire();
 }
}
```

### UpdateAmmoValues

```cpp
//更新弹药数量。
void UCombatComponent::UpdateAmmoValues()
{
  //如果角色和装备武器为空直接返回。
 if (Character == nullptr || EquippedWeapon == nullptr) return;
 //定义换弹数量等于重新装弹数量。
 int32 ReloadAmount = AmountToReload();
 //如果携带的备用弹药中有装备武器类型的。
 if (CarriedAmmoMap.Contains(EquippedWeapon->GetWeaponType()))
 {
  //备用弹药等于备用弹药减去换弹数量。携带弹药等于更新的备用弹药。
  CarriedAmmoMap[EquippedWeapon->GetWeaponType()] -= ReloadAmount;
  CarriedAmmo = CarriedAmmoMap[EquippedWeapon->GetWeaponType()];
 }
 //获得控制器。
 Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
 //如果控制器存在，调用更新携带弹药HUD方法，调用武器的增加弹药方法。
 if (Controller)
 {
  Controller->SetHUDCarriedAmmo(CarriedAmmo);
 }
 EquippedWeapon->AddAmmo(ReloadAmount);
}
```

### FinishSwap

```cpp
//完成切换武器。
void UCombatComponent::FinishSwap()
{
  //如果角色是权威的就调用战斗状态等于空闲。
 if (Character && Character->HasAuthority())
 {
  CombatState = ECombatState::ECS_Unoccuiped;
 }
 //设置角色的完成切换为true，设置第二把武器的深度为true，开启武器的描边发光。
 if (Character) Character->bFinishedSwapping = true;
 if (SecondaryWeapon) SecondaryWeapon->EnableCustomDepth(true);
}
```

### FinishSwapAttachWeapons

```cpp
//完成切换并附加武器。
void UCombatComponent::FinishSwapAttachWeapons()
{
  //如果角色为空或者角色不是权威直接返回。
 if (Character == nullptr || !Character->HasAuthority()) return;
 //播放切换武器声音。
 PlayEquipWeaponSound(SecondaryWeapon);
 //把主武器和副武器进行对调。
 AWeapon* TempWeapon = EquippedWeapon;
 EquippedWeapon = SecondaryWeapon;
 SecondaryWeapon = TempWeapon;

//调用设置武器状态为装备。添加武器到右手，设置武器的弹药HUD，更新携带的弹药。
 EquippedWeapon->SetWeaponState(EWeaponState::EWS_Equipped);
 AttachActorToRightHand(EquippedWeapon);
 EquippedWeapon->SetHUDAmmo();
 UpdateCarriedAmmo();

//调用第二把武器的设置武器状态为装备第二把武器，把武器添加到背包插槽上。
 SecondaryWeapon->SetWeaponState(EWeaponState::EWS_EquippedSecondary);
 AttachActorToBackpack(SecondaryWeapon);
}
```

### OnRep_Grenades

```cpp
//手雷的网络通知。
void UCombatComponent::OnRep_Grenades()
{
  //如果手雷变化，通知客户端调用更新手雷HUD方法。
 UpdateHUDGrenade();
}
```

### UpdateHUDGrenade

```cpp
void UCombatComponent::UpdateHUDGrenade()
{
  //获得控制器。
 Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
 if (Controller)
 {
  //调用控制器的设置手雷HUD方法。
  Controller->SetHUDGrenades(Grenades);
 }
}
```

### ThrowGrenadeFinished

```cpp
//扔手雷完成。
void UCombatComponent::ThrowGrenadeFinished()
{
  //战斗状态设置为空闲，添加装备武器到右手。
 CombatState = ECombatState::ECS_Unoccuiped;
 AttachActorToRightHand(EquippedWeapon);
}
```

### LaunchGrenade

```cpp
//发射实际的手雷。
void UCombatComponent::LaunchGrenade()
{
  //设置手雷的网格不可见。
 ShowAttachedGrenade(false);
 //如果角色是本地控制。调用服务器RPC发射手雷并传递击中目标。
 if (Character && Character->IsLocallyControlled())
 {
  ServerLaunchGrenade(HitTarget);
 }
}
```

### ShowAttachedGrenade

```cpp
void UCombatComponent::ShowAttachedGrenade(bool bShowGrenade)
{
  //如果角色存在手雷，设置手雷网格的可见性为传入的值。
 if (Character && Character->GetAttachedGrenade())
 {
  Character->GetAttachedGrenade()->SetVisibility(bShowGrenade);
 }
}
```

### ServerLaunchGrenade

```cpp
//服务器RPC发射手雷。
void UCombatComponent::ServerLaunchGrenade_Implementation(const FVector_NetQuantize& Target)
{
  //如果手雷类存在并且能得到网格。
 if (GrenadeClass && Character->GetAttachedGrenade())
 {
  //设置开始位置等于角色的手雷网格位置。
  const FVector StartingLocation = Character->GetAttachedGrenade()->GetComponentLocation();
  //计算从目标点到开始位置的向量。
  FVector ToTarget = Target - StartingLocation;
  //创建生成对象结构体。设置结构体的所有者为角色，挑起者为角色。
  FActorSpawnParameters SpawnParams;
  SpawnParams.Owner = Character;
  SpawnParams.Instigator = Character;
  UWorld* World = GetWorld();
  //如果世界存在，调用生成角色方法，生成手雷并传入类型，开始位置，手雷的旋转，设置的生成对象结构体。
  if (World)
  {
   World->SpawnActor<AProjectile>(
    GrenadeClass,
    StartingLocation,
    ToTarget.Rotation(),
    SpawnParams
   );
  }
 }
}
```

### OnRep_CombatState

```cpp
//战斗状态复制通知。
void UCombatComponent::OnRep_CombatState()
{
 switch (CombatState)
 {
 case ECombatState::ECS_Reloading:
 //如果战斗状态是换弹。并且角色不是本地控制，调用本地换弹方法。
  if (Character && !Character->IsLocallyControlled()) HandleReload();
  break;
 case ECombatState::ECS_Unoccuiped:
 //如果战斗状态是空闲，并且开火按钮按下，调用开火方法。
  if (bFireButtonPressed)
  {
   Fire();
  }
  break;
 case ECombatState::ECS_ThrowingGrenade:
 //如果角色不是本地控制，并且战斗状态时扔手雷。调用播放扔手雷蒙太奇，添加装备武器到左手，显示手雷网格。
  if (Character && !Character->IsLocallyControlled())
  {
   Character->PlayThrowGrenadeMontage();
   AttachActorToLeftHand(EquippedWeapon);
   ShowAttachedGrenade(true);
  }
  break;
 case ECombatState::ECS_SwappingWeapon:
 //如果战斗状态是切换武器，并且角色不是本地控制，播放切换武器动画。
  if (Character && !Character->IsLocallyControlled())
  {
   Character->PlaySwapMontage();
  }
  break;
 }
}
```

### AmountToReload

```cpp
//装填武器的数量。
int32 UCombatComponent::AmountToReload()
{
  //如果装备武器为空直接返回。
 if (EquippedWeapon == nullptr) return 0;
 //计算需要装填的弹药，武器最大装弹量减去现在武器的弹药。
 int32 RoomInMag = EquippedWeapon->GetMagCapacity() - EquippedWeapon->GetAmmo();

//如果携带的弹药Map有这个类型的武器类型。
 if (CarriedAmmoMap.Contains(EquippedWeapon->GetWeaponType()))
 {
  //获得这个类型的武器的携带弹药数量。计算需要装填的弹药和携带弹药的最小值。返回需要装填的弹药。为什么要这样计算？因为给武器装填的弹药首先应该满足它现在所需多少才能装满，但是应该保证这个所需的弹药不能大于携带的备用弹药，所以取两者的最小者。并用clamp函数限制在0到Least之间。
  int32 AmountCarried = CarriedAmmoMap[EquippedWeapon->GetWeaponType()];
  int32 Least = FMath::Min(RoomInMag, AmountCarried);
  return FMath::Clamp(RoomInMag, 0, Least);
 }
 return 0;
}
```

### ThrowGrenade

```cpp
//投掷手雷。
void UCombatComponent::ThrowGrenade()
{
  //如果手雷为空则直接返回。
 if (Grenades == 0) return;
 //如果战斗状态不等于空闲或者装备武器为空也直接返回。
 if (CombatState != ECombatState::ECS_Unoccuiped || EquippedWeapon == nullptr) return;
 //设置战斗状态等于投掷手雷。
 CombatState = ECombatState::ECS_ThrowingGrenade;
 //如果角色是本地控制。
 if (Character && Character->IsLocallyControlled())
 {
  //播放投掷手雷动画，添加装备武器到左手，空出右手来投掷手雷。显示手雷的网格。
  Character->PlayThrowGrenadeMontage();
  AttachActorToLeftHand(EquippedWeapon);
  ShowAttachedGrenade(true);
 }
 //如果角色不是权威角色。调用服务器RPC投掷手雷。
 if (Character && !Character->HasAuthority())
 {
  ServerThrowGrenade();
 }
 //如果角色是权威角色，手雷数量等于当前手雷数减去一个。并限制在0和最大手雷之间。更新手雷HUD。
 if (Character && Character->HasAuthority())
 {
  Grenades = FMath::Clamp(Grenades - 1, 0, MaxGrenades);
  UpdateHUDGrenade();
 }
}
```

### ServerThrowGrenade

```cpp
//服务器RPC投掷手雷。
void UCombatComponent::ServerThrowGrenade_Implementation()
{
  //如果手雷为空直接返回。
 if (Grenades == 0) return;
 //战斗状态设置为投掷手雷。
 CombatState = ECombatState::ECS_ThrowingGrenade;
 if (Character)
 {
  //如果角色存在，播放投掷手雷蒙太奇，添加武器到左手，设置手雷网格可见。
  Character->PlayThrowGrenadeMontage();
  AttachActorToLeftHand(EquippedWeapon);
  ShowAttachedGrenade(true);
 }
 //手雷等于手雷数减去1，更新手雷HUD。
 Grenades = FMath::Clamp(Grenades - 1, 0, MaxGrenades);
 UpdateHUDGrenade();
}
```

### ShouldSwapWeapons

```cpp
//应该切换武器。
bool UCombatComponent::ShouldSwapWeapons()
{
  //返回装备武器不为空并且第二把武器不为空。才能切换武器。
 return (EquippedWeapon != nullptr && SecondaryWeapon != nullptr);
}
```

### OnRep_EquippedWeapon

```cpp
//装备武器复制通知。
void UCombatComponent::OnRep_EquippedWeapon()
{
  //如果角色和装备武器都存在。
 if (EquippedWeapon && Character)
 {
  //调用装备武器重置弹药的服务器预测换弹。设置武器状态为装备。添加武器到右手。
  EquippedWeapon->SetSquence();
  EquippedWeapon->SetWeaponState(EWeaponState::EWS_Equipped);
  AttachActorToRightHand(EquippedWeapon);
  //设置角色不跟随方向转向，使用控制器旋转。播放装备武器声音。
  Character->GetCharacterMovement()->bOrientRotationToMovement = false;
  Character->bUseControllerRotationYaw = true;
  PlayEquipWeaponSound(EquippedWeapon);
  //设置装备武器的描边发光为false，设置武器的弹药HUD。
  EquippedWeapon->EnableCustomDepth(false);
  EquippedWeapon->SetHUDAmmo();
 }
}
```

### OnRep_SecondaryWeapon

```cpp
//第二把武器网络复制。
void UCombatComponent::OnRep_SecondaryWeapon()
{
  //如果角色和第二把武器存在。
 if (SecondaryWeapon && Character)
 {
  //设置武器的状态为装备，添加武器到背包，播放装备武器的声音。
  SecondaryWeapon->SetWeaponState(EWeaponState::EWS_EquippedSecondary);
  AttachActorToBackpack(SecondaryWeapon);
  PlayEquipWeaponSound(EquippedWeapon);
 }
}
```

### SetAiming

```cpp
//设置瞄准。
void UCombatComponent::SetAiming(bool bIsAiming)
{
  //如果角色和装备武器为空直接返回。
 if (Character == nullptr || EquippedWeapon == nullptr) return;
 //是否瞄准等于传入的瞄准值。调用服务器RPC设置瞄准。
 bAiming = bIsAiming;
 ServerSetAiming(bIsAiming);
 if (Character)
 {
  //设置角色移动组件的最大速度，判断是否在瞄准，如果是就变为瞄准速度，如果没有瞄准就是基础速度。
  Character->GetCharacterMovement()->MaxWalkSpeed = bIsAiming ? AimWalkSpeed : BaseWalkSpeed;
 }
 //如果角色是本地控制并且武器类型是狙击枪。
 if (Character->IsLocallyControlled() && EquippedWeapon->GetWeaponType() == EWeaponType::EWT_SniperRifle)
 {
  //设置角色显示狙击枪HUD等于是否瞄准。
  Character->ShowSniperScopeWidget(bIsAiming);
 }
 //如果角色是本地控制，设置按下瞄准按钮等于传入的瞄准值。
 if (Character->IsLocallyControlled()) bAimButtonPressed = bIsAiming;
}
```

### ServerSetAiming

```cpp
//服务器RPC设置瞄准。
void UCombatComponent::ServerSetAiming_Implementation(bool bIsAiming)
{
  //瞄准等于传入的瞄准值。
 bAiming = bIsAiming;
 if (Character)
 {
  //设置角色移动组件的最大速度，判断是否在瞄准，如果是就变为瞄准速度，如果没有瞄准就是基础速度。
  Character->GetCharacterMovement()->MaxWalkSpeed = bIsAiming ? AimWalkSpeed : BaseWalkSpeed;
 }
}
```

### OnRep_Aiming

```cpp
//瞄准的网络复制通知。
void UCombatComponent::OnRep_Aiming()
{
  //如果角色是本地控制。设置是否瞄准以玩家是否按下瞄准按钮为准。
 if (Character && Character->IsLocallyControlled())
 {
  bAiming = bAimButtonPressed;
 }
}
```

### OnRep_CarriedAmmo

```cpp
//携带弹药的网络复制通知。
void UCombatComponent::OnRep_CarriedAmmo()
{
  //获得控制器。
 Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
 if (Controller)
 {
  //调用设置携带弹药HUD方法。
  Controller->SetHUDCarriedAmmo(CarriedAmmo);
 }
 //跳转霰弹枪结尾动画的判断条件是战斗组件是否在换弹，武器是霰弹枪，携带弹药为空。
 bool bJumpToShotgunEnd =
  CombatState == ECombatState::ECS_Reloading &&
  EquippedWeapon != nullptr &&
  EquippedWeapon->GetWeaponType() == EWeaponType::EWT_Shotgun &&
  CarriedAmmo == 0;
 if (bJumpToShotgunEnd)
 {
  //如果跳转为true，则调用跳转方法。
  JumpToShotgunEnd();
 }
}
```

### OnRep_HoldingTheFlag

```cpp
//是否在举旗的网络复制通知。
void UCombatComponent::OnRep_HoldingTheFlag()
{
  //如果举旗并且角色是本地控制。
 if (bHoldingTheFlag && Character && Character->IsLocallyControlled())
 {
  //调用角色蹲下方法。
  Character->Crouch();
 }
}
```

### OnRep_Flag

```cpp
//旗帜的网络复制通知。
void UCombatComponent::OnRep_Flag()
{
  //如果旗帜为真，代表角色拿起旗帜，设置旗子的状态为装备，添加旗子到角色的左手。
 if (TheFlag)
 {
  TheFlag->SetWeaponState(EWeaponState::EWS_Equipped);
  AttachFlagToLeftHand(TheFlag);
 }
}
```
