# Casing

武器开火之后的弹出的弹壳，更为逼真。

## Casing.h

```cpp
UCLASS()
class BLASTERLEARING_API ACasing : public AActor
{
 GENERATED_BODY()
 
public: 
 ACasing();


protected:
 virtual void BeginPlay() override;

 UFUNCTION()
 virtual void OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit);
 
private:
 UPROPERTY(VisibleAnywhere)
 UStaticMeshComponent* CasingMesh;

 UPROPERTY(EditAnywhere)
 float ShellEjectionImpulse;

 UPROPERTY(EditAnywhere)
 class USoundCue* ShellSound;
};
```

## Casing.cpp

### 构造函数

```cpp
ACasing::ACasing()
{
 PrimaryActorTick.bCanEverTick = false;

//创建弹壳网格。设置为根组件。
 CasingMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("CasingMesh"));
 SetRootComponent(CasingMesh);
 //设置弹壳对于相机的碰撞为忽略。开启弹壳的物理模拟和重力。开启刚体碰撞，设置弹出冲量为10.f。
 CasingMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Camera, ECollisionResponse::ECR_Ignore);
 CasingMesh->SetSimulatePhysics(true);
 CasingMesh->SetEnableGravity(true);
 CasingMesh->SetNotifyRigidBodyCollision(true);
 ShellEjectionImpulse = 10.f;
}
```

### BeginPlay

```cpp
void ACasing::BeginPlay()
{
 Super::BeginPlay();
 
 //动态绑定碰撞事件OnHit。施加冲量等于弹壳的前向向量乘以冲量。
 CasingMesh->OnComponentHit.AddDynamic(this, &ACasing::OnHit);
 CasingMesh->AddImpulse(GetActorForwardVector() * ShellEjectionImpulse);
}
```

### OnHit

```cpp
//碰撞事件。
void ACasing::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
{
    //如果有声音。
 if (ShellSound)
 {
    //播放声音。
  UGameplayStatics::PlaySoundAtLocation(this, ShellSound, GetActorLocation());
 }
 //调用销毁函数。
 Destroy();
}
```
