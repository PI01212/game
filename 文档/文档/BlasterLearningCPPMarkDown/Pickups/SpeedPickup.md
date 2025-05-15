# SpeedPickup

## SpeedPickup.h

```cpp
UCLASS()
class BLASTERLEARING_API ASpeedPickup : public APickup
{
 GENERATED_BODY()
 
protected:
 virtual void OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);

private:
 UPROPERTY(EditAnywhere)
 float BaseSpeedBuff = 1600.f;
 UPROPERTY(EditAnywhere)
 float CrouchSpeedBuff = 850.f;
 UPROPERTY(EditAnywhere)
 float SpeedBuffTime = 30.f;
};
```

## SpeedPickup.cpp

### OnSphereOverlap

```cpp
void ASpeedPickup::OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
 Super::OnSphereOverlap(OverlappedComponent, OtherActor, OtherComp, OtherBodyIndex, bFromSweep, SweepResult);

 ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
 if (BlasterCharacter)
 {
  UBufferComponent* Buff = BlasterCharacter->GetBuff();
  if (Buff)
  {
    //调用增加角色的速度。
   Buff->BuffSpeed(BaseSpeedBuff, CrouchSpeedBuff, SpeedBuffTime);
  }
 }
 //销毁小部件。
 Destroy();
}
```
