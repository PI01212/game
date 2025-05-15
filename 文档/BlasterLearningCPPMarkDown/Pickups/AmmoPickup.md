# AmmoPickup

弹药拾取小部件。

## AmmoPickup.h

```cpp
UCLASS()
class BLASTERLEARING_API AAmmoPickup : public APickup
{
 GENERATED_BODY()
 
protected:
 virtual void OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult) override;

private:
 UPROPERTY(EditAnywhere)
 int32 AmmoAmount = 30;

 UPROPERTY(EditAnywhere)
 EWeaponType WeaponType;
};
```

## AmmoPickup.cpp

### OnSphereOverlap

```cpp
//重写的回调函数。
void AAmmoPickup::OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
 Super::OnSphereOverlap(OverlappedComponent, OtherActor, OtherComp, OtherBodyIndex, bFromSweep, SweepResult);

//获得重叠角色。
 ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
 if (BlasterCharacter)
 {
    //获得角色的战斗组件。
  UCombatComponent* Combat = BlasterCharacter->GetCombat();
  if (Combat)
  {
    //调用拾取弹药。
   Combat->PickupAmmo(WeaponType, AmmoAmount);
  }
 }
 //调用销毁对象。
 Destroy();
}
```
