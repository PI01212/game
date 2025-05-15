# HealthPickup

## HealthPickup.h

```cpp
UCLASS()
class BLASTERLEARING_API AHealthPickup : public APickup
{
 GENERATED_BODY()
 
public:
 AHealthPickup();

protected:
 virtual void OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);

private:
 UPROPERTY(EditAnywhere)
 float HealthAmount = 100.f;

 UPROPERTY(EditAnywhere)
 float HealthingTime = 5.f;
};
```

## HealthPickup.cpp

### OnSphereOverlap

```cpp
void AHealthPickup::OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
 Super::OnSphereOverlap(OverlappedComponent, OtherActor, OtherComp, OtherBodyIndex, bFromSweep, SweepResult);

//获得重叠角色。
 ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
 if (BlasterCharacter)
 {
    //获得角色的Buff组件。
  UBufferComponent* Buff = BlasterCharacter->GetBuff();
  if (Buff)
  {
    //调用治疗方法。
   Buff->Heal(HealthAmount, HealthingTime);
  }
 }
 //销毁小部件。
 Destroy();
}
```
