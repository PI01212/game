# ShieldPickup

## ShieldPickup.h

```cpp
UCLASS()
class BLASTERLEARING_API AShieldPickup : public APickup
{
 GENERATED_BODY()

protected:
 virtual void OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);

private:
 UPROPERTY(EditAnywhere)
 float ShieldReplenishAmount = 100.f;

 UPROPERTY(EditAnywhere)
 float ShieldReplenishTime = 5.f;
 
};
```

## ShieldPickup.cpp

### OnSphereOverlap

```cpp
void AShieldPickup::OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
 Super::OnSphereOverlap(OverlappedComponent, OtherActor, OtherComp, OtherBodyIndex, bFromSweep, SweepResult);

 ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
 if (BlasterCharacter)
 {
  UBufferComponent* Buff = BlasterCharacter->GetBuff();
  if (Buff)
  {
    //补充护盾。
   Buff->ReplenishShield(ShieldReplenishAmount, ShieldReplenishTime);
  }
 }
 Destroy();
}
```
