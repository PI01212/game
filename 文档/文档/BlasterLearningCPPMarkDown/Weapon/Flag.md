# Flag

旗子也是继承自武器类，但不需要射击或者其他方法，只需要在捡起时拿起和被击杀时掉落即可。并且和夺旗模式中的玩家基地区域触发重叠事件即可。

## Flag.h

```cpp
UCLASS()
class BLASTERLEARING_API AFlag : public AWeapon
{
 GENERATED_BODY()
 
public:
 AFlag();
 virtual void Dropped() override;
 void ResetFlag();

protected:
 virtual void OnEquipped() override;
 virtual void OnDropped() override;
 virtual void BeginPlay() override;

private:
 UPROPERTY(VisibleAnywhere)
 UStaticMeshComponent* FlagMesh;

 FTransform InitialTransform;

public:
 FORCEINLINE FTransform GetInitialTransform() const { return InitialTransform; }
};
```

## Flag.cpp

### 构造函数

```cpp
AFlag::AFlag()
{
    //创建旗帜网格，设置为根组件。把区域球体组件和拾取小部件添加到旗帜网格上。
 FlagMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("FlagMesh"));
 SetRootComponent(FlagMesh);
 GetAreaSphere()->SetupAttachment(FlagMesh);
 GetPickupWidget()->SetupAttachment(FlagMesh);
 //设置旗帜网格对于所有通道的碰撞为忽略，禁用碰撞相应。
 FlagMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
 FlagMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
}
```

### BeginPlay

```cpp
void AFlag::BeginPlay()
{
 Super::BeginPlay();
 //存储初始变换用于恢复旗帜。设置开启复制和移动组件同步保持服务器客户端同步。
 InitialTransform = GetActorTransform();
 bReplicates = true;
 SetReplicateMovement(true);
}
```

### Dropped

```cpp
void AFlag::Dropped()
{
    //设置旗帜状态为掉落。
 SetWeaponState(EWeaponState::EWS_Dropped);
 //把武器从角色网格分开保持世界中的变换。
 FDetachmentTransformRules DetachRules(EDetachmentRule::KeepWorld, true);
 FlagMesh->DetachFromComponent(DetachRules);
 //设置所有者为空指针。所有者角色和控制器设置为空。
 SetOwner(nullptr);
 BlasterOwnerCharacter = nullptr;
 BlasterOwnerController = nullptr;
}
```

### OnEquipped

```cpp
void AFlag::OnEquipped()
{
    //隐藏拾取小组件。禁用区域球体碰撞。
 ShowPickupWidget(false);
 GetAreaSphere()->SetCollisionEnabled(ECollisionEnabled::NoCollision);

//禁用旗帜网格物理模拟和重力，只开启网格的重叠相应。对于动态物体通道碰撞为重叠。禁用自定义深度。
 FlagMesh->SetSimulatePhysics(false);
 FlagMesh->SetEnableGravity(false);
 FlagMesh->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
 FlagMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_WorldDynamic, ECollisionResponse::ECR_Overlap);
 EnableCustomDepth(false);
}
```

### OnDropped

```cpp
void AFlag::OnDropped()
{
    //如果是权威。
 if (HasAuthority())
 {
    //开启区域球体的重叠。
  GetAreaSphere()->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
 }
 //开启模拟物理和重力。开启旗子的碰撞。对于所有通道碰撞设置为阻挡。对于Pawn和相机通道设置为忽略。
 FlagMesh->SetSimulatePhysics(true);
 FlagMesh->SetEnableGravity(true);
 FlagMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
 FlagMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Block);
 FlagMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Ignore);
 FlagMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Camera, ECollisionResponse::ECR_Ignore);
 //设置自定义深度的颜色为蓝色，标记为脏重新渲染，开启自定义深度。
 FlagMesh->SetCustomDepthStencilValue(CUSTOM_DEPTH_BLUE);
 FlagMesh->MarkRenderStateDirty();
 EnableCustomDepth(true);
}
```

### ResetFlag

```cpp
//重置旗子。
void AFlag::ResetFlag()
{
    //获得旗子的所有者。
 ABlasterCharacter* FlagBearer = Cast<ABlasterCharacter>(GetOwner());
 if (FlagBearer)
 {
    //设置所有者的正在举旗为false，重叠的武器为空，站起。
  FlagBearer->SetHoldingTheFlag(false);
  FlagBearer->SetOverlappingWeapon(nullptr);
  FlagBearer->UnCrouch();
 }

//如果不是权威直接返回。
 if (!HasAuthority()) return;
 //设置旗子的变换等于初始变换。把旗子从所有者的网格分开保持世界中的变换。
 SetActorTransform(InitialTransform);
 FDetachmentTransformRules DetachRules(EDetachmentRule::KeepWorld, true);
 FlagMesh->DetachFromComponent(DetachRules);
 //设置旗子的状态为初始化。开启区域球体碰撞，对Pawn通道的碰撞设置为重叠。
 SetWeaponState(EWeaponState::EWS_Initial);
 GetAreaSphere()->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
 GetAreaSphere()->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Overlap);
 //设置所有者为空指针，所有者角色和控制器设置为空指针。
 SetOwner(nullptr);
 BlasterOwnerCharacter = nullptr;
 BlasterOwnerController = nullptr;
}
```
