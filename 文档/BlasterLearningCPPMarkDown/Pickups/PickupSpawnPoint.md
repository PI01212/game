# PickupSpawnPoint

## PickupSpawnPoint.h

```cpp
UCLASS()
class BLASTERLEARING_API APickupSpawnPoint : public AActor
{
 GENERATED_BODY()
 
public: 
 APickupSpawnPoint();
 virtual void Tick(float DeltaTime) override;

protected:
 virtual void BeginPlay() override;

 UPROPERTY(EditAnywhere)
 TArray<TSubclassOf<class APickup>> PickupClasses;

 UPROPERTY()
 APickup* SpawnedPickup;

 void SpawnPickup();
 void SpawnPickupTimerFinished();

 UFUNCTION()
 void StartSpawnPickupTimer(AActor* DestroyedActor);

private:
 FTimerHandle SpawnPickupTimer;

 UPROPERTY(EditAnywhere)
 float SpawnPickupTimeMin;

 UPROPERTY(EditAnywhere)
 float SpawnPickupTimeMax;

public: 

};
```

## PickupSpawnPoint.cpp

### 构造函数

```cpp
APickupSpawnPoint::APickupSpawnPoint()
{
    //开启Tick，开启网络复制。
 PrimaryActorTick.bCanEverTick = true;
 bReplicates = true;
}
```

### BeginPlay

```cpp
void APickupSpawnPoint::BeginPlay()
{
 Super::BeginPlay();

//调用开始生成拾取小部件计时器。
 StartSpawnPickupTimer((AActor*)nullptr);
}
```

### StartSpawnPickupTimer

```cpp
void APickupSpawnPoint::StartSpawnPickupTimer(AActor* DestroyedActor)
{
    //随机小部件生成时间。
 const float SpawnTime = FMath::FRandRange(SpawnPickupTimeMin, SpawnPickupTimeMax);
 //创建生成小部件计时器，绑定回调函数。
 GetWorldTimerManager().SetTimer(SpawnPickupTimer, this, &APickupSpawnPoint::SpawnPickupTimerFinished, SpawnTime);
}
```

### SpawnPickupTimerFinished

```cpp
void APickupSpawnPoint::SpawnPickupTimerFinished()
{
    //如果是权威。
 if (HasAuthority())
 {
    //生成拾取小部件。
  SpawnPickup();
 }
}
```

### SpawnPickup

```cpp
void APickupSpawnPoint::SpawnPickup()
{
    //定义生成的拾取小部件类型的数量。
 int32 NumPickupClasses = PickupClasses.Num();
 //如果大于0，代表有可以生成的小部件类型。
 if (NumPickupClasses > 0)
 {
    //随机选取生成小部件类型。
  int32 Selection = FMath::RandRange(0, NumPickupClasses - 1);
  //生成拾取小部件。
  SpawnedPickup = GetWorld()->SpawnActor<APickup>(PickupClasses[Selection], GetActorTransform());

//如果是权威，生成的小组件存在。
  if (HasAuthority() && SpawnedPickup)
  {
    //动态绑定回调函数StartSpawnPickupTimer。
   SpawnedPickup->OnDestroyed.AddDynamic(this, &APickupSpawnPoint::StartSpawnPickupTimer);
  }
 }
}
```

### Tick

```cpp
void APickupSpawnPoint::Tick(float DeltaTime)
{
 Super::Tick(DeltaTime);
}
```
