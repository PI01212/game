# Weapon

武器基类，所有的武器和旗子都继承自武器类，武器类实现各种武器一些基础的功能，具体的武器再根据武器类的方法进行重写实现自己的武器特性。

## Weapon.h

```cpp
UENUM(BlueprintType)
enum class EWeaponState : uint8
{
 EWS_Initial UMETA(DisplayName = "Initial State"),
 EWS_Equipped UMETA(DisplayName = "Equipped"),
 EWS_EquippedSecondary UMETA(DisplayName = "Equipped Secondary"),
 EWS_Dropped UMETA(DisplayName = "Dropped"),
 EWS_MAX UMETA(DisplayName = "DefaultMAX")
};

UENUM(BlueprintType)
enum class EFireType : uint8
{
 EFT_HitScan UMETA(DisplayName = "Hit Scan Weapon"),
 EFT_Projectile UMETA(DisplayName = "Projectile Weapon"),
 EFT_Shotgun UMETA(DisplayName = "Shotgun Weapon"),

 EFT_MAX UMETA(DisplayName = "DefaultMAX")
};

UCLASS()
class BLASTERLEARING_API AWeapon : public AActor
{
 GENERATED_BODY()
 
public: 
 AWeapon();
 virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
 virtual void OnRep_Owner() override;
 void SetHUDAmmo();
 void ShowPickupWidget(bool bShowWidget);
 virtual void Fire(const FVector& HitTarget);
 virtual void Dropped();
 void AddAmmo(int32 AmmoToAdd);
 FVector TraceEndWithScatter(const FVector& HitTarget);
 UFUNCTION(NetMulticast, Reliable)
 void MulticastAmmo(int32 UpdateAmmo);

 /*
 * 武器准星的纹理
 */
 UPROPERTY(EditAnywhere, Category = Crosshairs)
 class UTexture2D* CrosshairsCenter;

 UPROPERTY(EditAnywhere, Category = Crosshairs)
 UTexture2D* CrosshairsLeft;

 UPROPERTY(EditAnywhere, Category = Crosshairs)
 UTexture2D* CrosshairsRight;

 UPROPERTY(EditAnywhere, Category = Crosshairs)
 UTexture2D* CrosshairsTop;

 UPROPERTY(EditAnywhere, Category = Crosshairs)
 UTexture2D* CrosshairsBottom;

 /*
 * 瞄准视野缩放
 */
 UPROPERTY(EditAnywhere)
 float ZoomedFOV = 30.f;

 UPROPERTY(EditAnywhere)
 float ZoomInterpSpeed = 20.f;

 /*
 * 自动开火
 */
 UPROPERTY(EditAnywhere, Category = Combat)
 float FireDelay = 0.15f;;

 UPROPERTY(EditAnywhere, Category = Combat)
 bool bAutomatic = true;

 UPROPERTY(EditAnywhere)
 class USoundCue* EquipSound;

 /*
 * 启用或禁用自定义深度
 */
 void EnableCustomDepth(bool bEnable);

 bool bDestroyWeapon = false;

 UPROPERTY(EditAnywhere)
 EFireType FireType;

 UPROPERTY(EditAnywhere, Category = "Weapon Scatter")
 bool bUseScatter = false;

protected:
 virtual void BeginPlay() override;
 virtual void OnWeaponStateSet();
 virtual void OnEquipped();
 virtual void OnDropped();
 virtual void OnEquippedSecondary();

 UFUNCTION()
 virtual void OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);
 UFUNCTION()
 void OnSphereEndOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex);

 /*
 * 带散射效果的追踪终点
 */

 UPROPERTY(EditAnywhere, Category = "Weapon Scatter")
 float DistanceToSphere = 800.f;

 UPROPERTY(EditAnywhere, Category = "Weapon Scatter")
 float SphereRadius = 75.f;

 UPROPERTY(EditAnywhere)
 float Damage = 20.f;

 UPROPERTY(EditAnywhere)
 float HeadShotDamage = 40.f;

 UPROPERTY(Replicated, EditAnywhere)
 bool bUseServerSideRewind = false;

 UPROPERTY()
 class ABlasterCharacter* BlasterOwnerCharacter;
 UPROPERTY()
 class ABlasterPlayerController* BlasterOwnerController;

 UFUNCTION()
 void OnPingTooHigh(bool bPingTooHigh);

private:
 UPROPERTY(VisibleAnywhere, Category = "Weapon Properties")
 USkeletalMeshComponent* WeaponMesh;

 //拾取武器用球体
 UPROPERTY(VisibleAnywhere, Category = "Weapon Properties")
 class USphereComponent* AreaSphere;

 UPROPERTY(ReplicatedUsing = OnRep_WeaponState, VisibleAnywhere, Category = "Weapon Properties")
 EWeaponState WeaponState;

 UFUNCTION()
 void OnRep_WeaponState();

 UPROPERTY(VisibleAnywhere, Category = "Weapon Properties")
 class UWidgetComponent* PickUpWidget;

 UPROPERTY(EditAnywhere, Category = "Weapon Properties")
 class UAnimationAsset* FireAnimation;

 UPROPERTY(EditAnywhere)
 TSubclassOf<class ACasing> CasingClass;

 UPROPERTY(EditAnywhere)
 int32 Ammo;

 UFUNCTION(Client, Reliable)
 void ClientUpdateAmmo(int32 ServerAmmo);

 UFUNCTION(Client, Reliable)
 void ClientAddAmmo(int32 AmmoToAdd);

 void SpendRound();

 UPROPERTY(EditAnywhere)
 int32 MagCapacity;

 //未匹配的服务器对于弹药的请求响应
 //在 SpendRound 中增加，在 ClientUpdateAmmo 中减少
 int32 Sequence = 0;

 UPROPERTY(EditAnywhere)
 EWeaponType WeaponType;

 UPROPERTY(EditAnywhere)
 ETeam Team;

public: 
 void SetWeaponState(EWeaponState State);
 FORCEINLINE USphereComponent* GetAreaSphere() const { return AreaSphere; }
 FORCEINLINE USkeletalMeshComponent* GetWeaponMesh() const { return WeaponMesh; }
 FORCEINLINE UWidgetComponent* GetPickupWidget() const { return PickUpWidget; }
  FORCEINLINE float GetZoomedFOV() const { return ZoomedFOV; }
 FORCEINLINE float GetZoomedInterpSpeed() const { return ZoomInterpSpeed; }
 FORCEINLINE EWeaponType GetWeaponType() const { return WeaponType; }
 FORCEINLINE int32 GetAmmo() const { return Ammo; }
 FORCEINLINE int32 GetMagCapacity() const { return MagCapacity; }
 FORCEINLINE float GetDamage() const { return Damage; }
 FORCEINLINE float GetHeadShotDamage() const { return HeadShotDamage; }
 FORCEINLINE ETeam GetTeam() const { return Team; }
 FORCEINLINE void SetSquence() { Sequence = 0; }
 bool IsEmpty();
 bool IsFull();
};
```

## Weapon.cpp

### 构造函数

```cpp
AWeapon::AWeapon()
{
    //禁用武器的Tick，设置武器为网络复制达到服务器和客户端同步，设置移动组件复制保证武器掉落的位置同步。
 PrimaryActorTick.bCanEverTick = false;
 bReplicates = true;
 SetReplicateMovement(true);

//创建武器网格，并设置为根组件。
 WeaponMesh = CreateDefaultSubobject<USkeletalMeshComponent>(TEXT("WeaponMesh"));
 SetRootComponent(WeaponMesh);

//设置网格对所有通道的碰撞为阻挡。武器对于Pawn的碰撞为忽略。禁用武器的碰撞。
 WeaponMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Block);
 WeaponMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Ignore);
 WeaponMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);

//设置网格的深度为蓝色，标记为脏，需要重新渲染，打开颜色深度。
 WeaponMesh->SetCustomDepthStencilValue(CUSTOM_DEPTH_BLUE);
 WeaponMesh->MarkRenderStateDirty();
 EnableCustomDepth(true);

//创建碰撞区域球体，添加到根组件，设置所有通道的碰撞为忽略，禁用碰撞，设置对于骨骼通道的碰撞为重叠使角色交互捡起武器。
 AreaSphere = CreateDefaultSubobject<USphereComponent>(TEXT("AreaSphere"));
 AreaSphere->SetupAttachment(RootComponent);
 AreaSphere->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
 AreaSphere->SetCollisionEnabled(ECollisionEnabled::NoCollision);
 AreaSphere->SetCollisionResponseToChannel(ECC_SkeletalMesh, ECollisionResponse::ECR_Overlap);

//创建捡起小组件，添加到根组件。
 PickUpWidget = CreateDefaultSubobject<UWidgetComponent>(TEXT("PickUpWidget"));
 PickUpWidget->SetupAttachment(RootComponent);
}
```

### EnableCustomDepth

```cpp
//设置自定义深度。
void AWeapon::EnableCustomDepth(bool bEnable)
{
 if (WeaponMesh)
 {
    //如果网格存在设置渲染自定义深度。
  WeaponMesh->SetRenderCustomDepth(bEnable);
 }
}
```

### BeginPlay

```cpp
void AWeapon::BeginPlay()
{
 Super::BeginPlay();

//开启碰撞区域球体碰撞。设置对于pawn通道的碰撞为重叠。
 AreaSphere->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
 AreaSphere->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Overlap);
 //动态绑定区域球体的重叠事件，绑定进入和离开的回调函数。
 AreaSphere->OnComponentBeginOverlap.AddDynamic(this, &AWeapon::OnSphereOverlap);
 AreaSphere->OnComponentEndOverlap.AddDynamic(this, &AWeapon::OnSphereEndOverlap);

 if (PickUpWidget)
 {
    //不显示拾取小部件。
  PickUpWidget->SetVisibility(false);
 }
}
```

### GetLifetimeReplicatedProps

```cpp
//对需要的网络复制参数进行注册。
void AWeapon::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
 Super::GetLifetimeReplicatedProps(OutLifetimeProps);

 DOREPLIFETIME(AWeapon, WeaponState);
 DOREPLIFETIME_CONDITION(AWeapon, bUseServerSideRewind, COND_OwnerOnly);
}
```

### OnSphereOverlap

```cpp
//角色进入重叠事件。
void AWeapon::OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
   //获得重叠的玩家角色。
 ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
 if (BlasterCharacter)
 {
   //如果武器的类型等于旗子，并且旗子属于重叠角色的队伍，直接返回。因为角色不能拿起自己队伍的旗子。
  if (WeaponType == EWeaponType::EWT_Flag && BlasterCharacter->GetTeam() == Team) return;
  //如果角色正在举旗，直接返回。
  if (BlasterCharacter->IsHoldingTheFlag()) return;
  //调用角色设置重叠武器，并传入当前武器对象。
  BlasterCharacter->SetOverlappingWeapon(this);
 }
}
```

### OnSphereEndOverlap

```cpp
//角色离开重叠事件。
void AWeapon::OnSphereEndOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex)
{
   //获得玩家角色。
 ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
 if (BlasterCharacter)
 {
   //如果武器的类型等于旗子，并且旗子属于重叠角色的队伍，直接返回。因为角色不能拿起自己队伍的旗子。
  if (WeaponType == EWeaponType::EWT_Flag && BlasterCharacter->GetTeam() == Team) return;
  //如果角色正在举旗，直接返回。
  if (BlasterCharacter->IsHoldingTheFlag()) return;
  //角色离开重叠区域，调用设置重叠武器传入空指针代表无武器重叠。
  BlasterCharacter->SetOverlappingWeapon(nullptr);
 }
}
```

### SetHUDAmmo

```cpp
//设置弹药HUD。
void AWeapon::SetHUDAmmo()
{
   //获得武器类所属的角色类。
 BlasterOwnerCharacter = BlasterOwnerCharacter == nullptr ? Cast<ABlasterCharacter>(GetOwner()) : BlasterOwnerCharacter;
 if (BlasterOwnerCharacter)
 {
   //获得所属玩家的控制器。
  BlasterOwnerController = BlasterOwnerController == nullptr ? Cast<ABlasterPlayerController>(BlasterOwnerCharacter->Controller) : BlasterOwnerController;
  if (BlasterOwnerController)
  {
   //调用控制器的设置武器弹药传入武器的弹药。
   BlasterOwnerController->SetHUDWeaponAmmo(Ammo);
  }
 }
}
```

###

```cpp
//消耗弹药。
void AWeapon::SpendRound()
{
   //减少一发弹药。
 Ammo = FMath::Clamp(Ammo - 1, 0, MagCapacity);
 //设置弹药HUD。
 SetHUDAmmo();
 //如果是权威。
 if (HasAuthority())
 {
   //调用客户端RPC。
  ClientUpdateAmmo(Ammo);
 }
 else
 {
  //增加sequence的值，用于跟踪弹药的变化来进行客户端预测。
  ++Sequence;
 }
}
```

### ClientUpdateAmmo

```cpp
//客户端更新弹药RPC。
void AWeapon::ClientUpdateAmmo_Implementation(int32 ServerAmmo)
{
  //如果是权威就直接返回。
 if (HasAuthority()) return;
 //弹药等于服务器更新的弹药，减去这次服务器确认，弹药等于弹药减去剩余的序列。
 //原理是客户端预测，客户端进行更改之后会请求服务器，服务器确认并回传给客户端，客户端接收到确认。所以这里的实现逻辑就是客户端减少弹药的时候会记录序列+1，序列代表着客户端需要收到的服务器确认次数，并且每次服务器发送确认都会带着属于这次确认的弹药数，通过这个服务器确认客户端会更改弹药变回这次确认的弹药和抵消掉这次服务器确认也就是序列-1，再根据这次的弹药减去当前剩余需要收到的确认序列就变回了当前正确的弹药。所以不管服务器端确认是否全部发送完毕传回客户端，客户端都可以保持正确的弹药。这就实现了客户端预测。
 Ammo = ServerAmmo;
 --Sequence;
 Ammo -= Sequence;
 //设置弹药HUD。
 SetHUDAmmo();
}
```

### AddAmmo

```cpp
//增加弹药。
void AWeapon::AddAmmo(int32 AmmoToAdd)
{
  //增加弹药，设置弹药HUD，调用客户端弹药RPC。
 Ammo = FMath::Clamp(Ammo + AmmoToAdd, 0, MagCapacity);
 SetHUDAmmo();
 ClientAddAmmo(AmmoToAdd);
}
```

### ClientAddAmmo

```cpp
//客户端增加弹药。
void AWeapon::ClientAddAmmo_Implementation(int32 AmmoToAdd)
{
  //如果是权威直接返回。
 if (HasAuthority()) return;
 //增加弹药。
 Ammo = FMath::Clamp(Ammo + AmmoToAdd, 0, MagCapacity);
 //获得武器类所属的所有者角色。
 BlasterOwnerCharacter = BlasterOwnerCharacter == nullptr ? Cast<ABlasterCharacter>(GetOwner()) : BlasterOwnerCharacter;
 //判断霰弹枪是否填满。
 bool IsShotgunFull = BlasterOwnerCharacter 
  && BlasterOwnerCharacter->GetCombat() 
  && BlasterOwnerCharacter->GetEquippedWeapon() 
  && WeaponType == EWeaponType::EWT_Shotgun 
  && IsFull();
  //如果填满。
 if (IsShotgunFull)
 {
  //调用跳转到换弹动画结束。
  BlasterOwnerCharacter->GetCombat()->JumpToShotgunEnd();
 }
 //设置弹药HUD。
 SetHUDAmmo();
}
```

### OnRep_Owner

```cpp
//武器所有者改变的通知。
void AWeapon::OnRep_Owner()
{
 Super::OnRep_Owner();
 //如果所有者为空。
 if (Owner == nullptr)
 {
  //设置所有者角色和控制器为空。
  BlasterOwnerCharacter = nullptr;
  BlasterOwnerController = nullptr;
 }
 else
 {
  //获得所有者角色
  BlasterOwnerCharacter = BlasterOwnerCharacter == nullptr ? Cast<ABlasterCharacter>(Owner) : BlasterOwnerCharacter;
  //如果所有者装备的武器等于此武器类，调用设置弹药HUD。
  if (BlasterOwnerCharacter && BlasterOwnerCharacter->GetEquippedWeapon() && BlasterOwnerCharacter->GetEquippedWeapon() == this)
  {
   SetHUDAmmo();
  }
 }
}
```

### SetWeaponState

```cpp
//设置武器状态。
void AWeapon::SetWeaponState(EWeaponState State)
{
  //武器状态等于传入的武器状态。调用OnWeaponStateSet。
 WeaponState = State;
 OnWeaponStateSet();
}
```

### OnWeaponStateSet

```cpp
//在武器状态设置后调用，根据不同的状态执行不同的逻辑。
void AWeapon::OnWeaponStateSet()
{
 switch (WeaponState)
 {
  //装备状态。
 case EWeaponState::EWS_Equipped:
  OnEquipped();
  break;
  //装备为第二把武器状态。
 case EWeaponState::EWS_EquippedSecondary:
  OnEquippedSecondary();
  break;
  //掉落武器状态。
 case EWeaponState::EWS_Dropped:
  OnDropped();
  break;
 }
}
```

### OnEquipped

```cpp
//装备时。
void AWeapon::OnEquipped()
{
  //隐藏拾取小部件。
 ShowPickupWidget(false);
 //禁用碰撞球形区域。禁用网格的物理模拟和重力，禁用碰撞。
 AreaSphere->SetCollisionEnabled(ECollisionEnabled::NoCollision);
 WeaponMesh->SetSimulatePhysics(false);
 WeaponMesh->SetEnableGravity(false);
 WeaponMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
 //如果武器类型等于冲锋枪。
 if (WeaponType == EWeaponType::EWT_SMG)
 {
  //开启网格碰撞，开启重力，对所有碰撞通道设置为忽略。
  WeaponMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
  WeaponMesh->SetEnableGravity(true);
  WeaponMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
 }
 //禁用自定义颜色深度。
 EnableCustomDepth(false);

//获取所有者角色。
 BlasterOwnerCharacter = BlasterOwnerCharacter == nullptr ? Cast<ABlasterCharacter>(GetOwner()) : BlasterOwnerCharacter;
 //如果使用服务器倒带。
 if (BlasterOwnerCharacter && bUseServerSideRewind)
 {
  //获得所有者的控制器。
  BlasterOwnerController = BlasterOwnerController == nullptr ? Cast<ABlasterPlayerController>(BlasterOwnerCharacter->Controller) : BlasterOwnerController;
  //如果是权威，控制器高ping委托未绑定回调函数。
  if (BlasterOwnerController && HasAuthority() && !BlasterOwnerController->HighPingDelegate.IsBound())
  {
    //动态绑定高ping委托的回调函数OnPingTooHigh。
   BlasterOwnerController->HighPingDelegate.AddDynamic(this, &AWeapon::OnPingTooHigh);
  }
 }
}
```

### OnEquippedSecondary

```cpp
//作为装备第二把武器。
void AWeapon::OnEquippedSecondary()
{
  //
 ShowPickupWidget(false);
 AreaSphere->SetCollisionEnabled(ECollisionEnabled::NoCollision);
 WeaponMesh->SetSimulatePhysics(false);
 WeaponMesh->SetEnableGravity(false);
 WeaponMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
 if (WeaponType == EWeaponType::EWT_SMG)
 {
  WeaponMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
  WeaponMesh->SetEnableGravity(true);
  WeaponMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
 }
 //如果网格存在。
 if (WeaponMesh)
 {
  //设置网格的自定义深度颜色。标记为脏，重新渲染。
  WeaponMesh->SetCustomDepthStencilValue(CUSTOM_DEPTH_TAN);
  WeaponMesh->MarkRenderStateDirty();
 }

 BlasterOwnerCharacter = BlasterOwnerCharacter == nullptr ? Cast<ABlasterCharacter>(GetOwner()) : BlasterOwnerCharacter;
 if (BlasterOwnerCharacter)
 {
  BlasterOwnerController = BlasterOwnerController == nullptr ? Cast<ABlasterPlayerController>(BlasterOwnerCharacter->Controller) : BlasterOwnerController;
  //如果是权威，并且所有者的控制器的高ping委托绑定了回调函数。
  if (BlasterOwnerController && HasAuthority() && BlasterOwnerController->HighPingDelegate.IsBound())
  {
    //动态移除高ping委托的回调函数OnPingTooHigh。
   BlasterOwnerController->HighPingDelegate.RemoveDynamic(this, &AWeapon::OnPingTooHigh);
  }
 }
}
```

### OnDropped

```cpp
void AWeapon::OnDropped()
{
  //如果是权威。
 if (HasAuthority())
 {
  //开启区域球体的射线检测和重叠。
  AreaSphere->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
 }
 //开启武器网格物理模拟和重力。开启碰撞检测，对所有碰撞通道设置为阻挡。对Pawn通道和相机的碰撞设置为忽略。
 WeaponMesh->SetSimulatePhysics(true);
 WeaponMesh->SetEnableGravity(true);
 WeaponMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
 WeaponMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Block);
 WeaponMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Ignore);
 WeaponMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Camera, ECollisionResponse::ECR_Ignore);

//设置武器的自定义深度为蓝色，并标记为脏重新渲染。打开自定义深度。
 WeaponMesh->SetCustomDepthStencilValue(CUSTOM_DEPTH_BLUE);
 WeaponMesh->MarkRenderStateDirty();
 EnableCustomDepth(true);

 BlasterOwnerCharacter = BlasterOwnerCharacter == nullptr ? Cast<ABlasterCharacter>(GetOwner()) : BlasterOwnerCharacter;
 if (BlasterOwnerCharacter)
 {
  BlasterOwnerController = BlasterOwnerController == nullptr ? Cast<ABlasterPlayerController>(BlasterOwnerCharacter->Controller) : BlasterOwnerController;
  if (BlasterOwnerController && HasAuthority() && BlasterOwnerController->HighPingDelegate.IsBound())
  {
   BlasterOwnerController->HighPingDelegate.RemoveDynamic(this, &AWeapon::OnPingTooHigh);
  }
 }
}
```

### OnPingTooHigh

```cpp
//ping太高不使用服务器倒带。
void AWeapon::OnPingTooHigh(bool bPingTooHigh)
{
  //是否使用服务器倒带等于ping值是否过高的相反值。
 bUseServerSideRewind = !bPingTooHigh;
}
```

### OnRep_WeaponState

```cpp
//武器状态复制触发的通知。
void AWeapon::OnRep_WeaponState()
{
  //武器状态设置之后进行状态设置。
 OnWeaponStateSet();
}
```

### ShowPickupWidget

```cpp
//显示拾取小部件。
void AWeapon::ShowPickupWidget(bool bShowWidget)
{
 if (PickUpWidget)
 {
  //调用拾取小部件的设置可见性。
  PickUpWidget->SetVisibility(bShowWidget);
 }
}
```

### Fire

```cpp
//开火函数。
void AWeapon::Fire(const FVector& HitTarget)
{
  //如果开火动画存在。
 if (FireAnimation)
 {
  //播放开火动画。
  WeaponMesh->PlayAnimation(FireAnimation, false);
 }
 //如果蛋壳类存在。
 if (CasingClass)
 {
  //获得弹药弹出插槽。
  const USkeletalMeshSocket* AmmoEjectSocket = WeaponMesh->GetSocketByName(FName("AmmoEject"));
  if (AmmoEjectSocket)
  {
    //通过武器网格获得插槽的插槽变换。
   FTransform SocketTransform = AmmoEjectSocket->GetSocketTransform(WeaponMesh);

   UWorld* World = GetWorld();
   if (World)
   {
    //生成弹出的蛋壳。
    World->SpawnActor<ACasing>(
     CasingClass,
     SocketTransform.GetLocation(),
     SocketTransform.GetRotation().Rotator()
    );
   }
  }
 }
 //调用消耗弹药。
 SpendRound();
}
```

### Dropped

```cpp
//掉落。
void AWeapon::Dropped()
{
  //设置武器状态等于掉落。
 SetWeaponState(EWeaponState::EWS_Dropped);
 //武器和角色的网格进行分离，保持世界中的位置和旋转。
 FDetachmentTransformRules DetachRules(EDetachmentRule::KeepWorld, true);
 WeaponMesh->DetachFromComponent(DetachRules);
 //设置所有者为空，所有者对象和控制器为空。
 SetOwner(nullptr);
 BlasterOwnerCharacter = nullptr;
 BlasterOwnerController = nullptr;
}
```

### IsEmpty

```cpp
//是否为空。
bool AWeapon::IsEmpty()
{
  //返回弹药是否小于等于0。
 return Ammo <= 0;
}
```

### IsFull

```cpp
//是否装满。
bool AWeapon::IsFull()
{
  //返回弹药是否等于弹夹。
 return Ammo == MagCapacity;
}
```

### TraceEndWithScatter

```cpp
//散射射线追踪。
FVector AWeapon::TraceEndWithScatter(const FVector& HitTarget)
{
  //获得枪火开火插槽。
 const USkeletalMeshSocket* MuzzleFlashSocket = GetWeaponMesh()->GetSocketByName("MuzzleFlash");
 //如果插槽为空直接返回空向量。
 if (MuzzleFlashSocket == nullptr) return FVector();

//获得插槽的变换。
 const FTransform SocketTransform = MuzzleFlashSocket->GetSocketTransform(GetWeaponMesh());
 //获得射线开始位置。
 const FVector TraceStart = SocketTransform.GetLocation();

//获得从开始到击中目标的方向。
 const FVector ToTargetNormalized = (HitTarget - TraceStart).GetSafeNormal();
 //球体中心等于射线开始位置加上方向向量乘以DistanceToSphere。
 const FVector SphereCenter = TraceStart + ToTargetNormalized * DistanceToSphere;
 //随机偏移向量等于随机单位向量乘以0到球体半径的随机值。
 const FVector RandVec = UKismetMathLibrary::RandomUnitVector() * FMath::FRandRange(0.f, SphereRadius);
 //结束位置等于球体中心加上随机偏移量。
 const FVector EndLoc = SphereCenter + RandVec;
 //计算从射线位置到球体结束位置的方向。
 const FVector ToEndLoc = EndLoc - TraceStart;

//最终的散射击中目标为把方向ToEndLoc归一化乘以射线长度加上射线开始位置。
 return FVector(TraceStart + ToEndLoc * TRACE_LENGTH / ToEndLoc.Size());
}
```

### MulticastAmmo

```cpp
//这是我发现的一个问题，如果从服务器端捡起开火再掉落，客户端捡起时弹药会不一致。但是开火一次之后又会同步。所以我的思路就是同步就是从捡起武器开始来计算，因为捡起武器是在服务器上调用，所以每当一个客户端角色捡起一把武器就对这把武器的弹药在客户端进行同步调用一次多播RPC。这样客户端捡起武器时弹药和服务器上的数量就是一致的了。
//我想到了一种更简洁的方法，其实武器弹药真正在乎是否正确的是玩家控制的哪个客户端，因为玩家需要开火和显示正确的弹药。所以不用使用多播RPC，这样进行一次多播同步太过夸张，现在只需要在设置武器的Owner之后调用一次Client RPC就可以了，这样只会去这个武器的所有者这台客户端上更新弹药就可以了，网络更高效，资源利用更好。详细的代码修改请看Github。
void AWeapon::MulticastAmmo_Implementation(int32 UpdateAmmo)
{
  //弹药等于更新的弹药。
 Ammo = UpdateAmmo;
}
```
