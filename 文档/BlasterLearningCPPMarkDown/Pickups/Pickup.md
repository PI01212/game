# Pickup

拾取小部件基类，游戏当中有两种拾取小部件，一种是弹药小部件，一种是Buff小部件。都继承自拾取小部件。只不过拾取之后的效果不同。

## Pickup.h

```cpp
UCLASS()
class BLASTERLEARING_API APickup : public AActor
{
 GENERATED_BODY()
 
public: 
 APickup();
 virtual void Tick(float DeltaTime) override;
 virtual void Destroyed() override;

protected:
 virtual void BeginPlay() override;

 UFUNCTION()
 virtual void OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);

 UPROPERTY(EditAnywhere)
 float BaseTurnRate = 45.f;

private:
 UPROPERTY(EditAnywhere)
 class USphereComponent* OverlapSphere;

 UPROPERTY(EditAnywhere)
 class USoundCue* PickupSound;

 UPROPERTY(EditAnywhere)
 UStaticMeshComponent* PickupMesh;

 UPROPERTY(VisibleAnywhere)
 class UNiagaraComponent* PickupEffectComponent;

 UPROPERTY(EditAnywhere)
 class UNiagaraSystem* PickupEffect;

 FTimerHandle BindOverlapTimer;

 float BindOverlapTime = 0.25f;

 void BindOverlapTimerFinished();

public: 
};
```

## Pickup.cpp

### 构造函数

```cpp
APickup::APickup()
{
    //开启tick.开启网络复制同步。
 PrimaryActorTick.bCanEverTick = true;
 bReplicates = true;

//创建场景类型根组件。
 RootComponent = CreateDefaultSubobject<USceneComponent>(TEXT("Root"));

//创建重叠区域球体组件。添加到根组件上，设置半径为150.f，开启碰撞的重叠检测。对于所有通道都设置为忽略，开启骨骼通道的重叠检测。
 OverlapSphere = CreateDefaultSubobject<USphereComponent>(TEXT("OverlapSphere"));
 OverlapSphere->SetupAttachment(RootComponent);
 OverlapSphere->SetSphereRadius(150.f);
 OverlapSphere->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
 OverlapSphere->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
 OverlapSphere->SetCollisionResponseToChannel(ECC_SkeletalMesh, ECollisionResponse::ECR_Overlap);

//创建拾取网格组件，添加到重叠区域球体上，禁用碰撞，设置x，y，z轴的缩放为5,开启自定义深度，设置颜色为紫色。
 PickupMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("PickupMesh"));
 PickupMesh->SetupAttachment(OverlapSphere);
 PickupMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
 PickupMesh->SetRelativeScale3D(FVector(5.f, 5.f, 5.f));
 PickupMesh->SetRenderCustomDepth(true);
 PickupMesh->SetCustomDepthStencilValue(CUSTOM_DEPTH_PURPLE);

//创建粒子系统组件添加到根组件上。
 PickupEffectComponent = CreateDefaultSubobject<UNiagaraComponent>(TEXT("PickupEffectComponent"));
 PickupEffectComponent->SetupAttachment(RootComponent);
}
```

### BeginPlay

```cpp
void APickup::BeginPlay()
{
 Super::BeginPlay();

//如果是权威。
 if (HasAuthority())
 {
    //创建绑定重叠定时器。绑定回调函数。
    //为何要延迟绑定重叠时事件。是因为存在一种情况，如果角色一直站在生成拾取位置，当下次拾取时，因为是角色已经站在重叠区域，所以重叠事件是马上触发，但销毁事件是动态绑定回调函数，需要时间。如果立即触发，销毁事件的回调函数没有绑定，就会导致不再生成拾取小部件。
  GetWorldTimerManager().SetTimer(BindOverlapTimer, this, &APickup::BindOverlapTimerFinished, BindOverlapTime);
 }
}
```

### OnSphereOverlap

```cpp
//重叠事件回调函数，不同类型的拾取小部件会对其进行重写。
void APickup::OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
 
}
```

### BindOverlapTimerFinished

```cpp
//重叠计时器回调函数。
void APickup::BindOverlapTimerFinished()
{
    //动态绑定重叠事件回调函数。
 OverlapSphere->OnComponentBeginOverlap.AddDynamic(this, &APickup::OnSphereOverlap);
}
```

### Tick

```cpp
void APickup::Tick(float DeltaTime)
{
 Super::Tick(DeltaTime);

 if (PickupMesh)
 {
    //设置拾取小组件一直旋转。
  PickupMesh->AddWorldRotation(FRotator(0.f, BaseTurnRate * DeltaTime, 0.f));
 }
}
```

### Destroyed

```cpp
void APickup::Destroyed()
{
 Super::Destroyed();

 if (PickupSound)
 {
    //播放拾取后的声音。
  UGameplayStatics::PlaySoundAtLocation(this, PickupSound, GetActorLocation());
 }
 if (PickupEffect)
 {
    //生成拾取后的粒子特效。
  UNiagaraFunctionLibrary::SpawnSystemAtLocation(this, PickupEffect, GetActorLocation(), GetActorRotation());
 }
}
```
