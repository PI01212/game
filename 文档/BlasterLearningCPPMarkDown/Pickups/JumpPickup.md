# JumpPickup

## JumpPickup.h

```cpp
UCLASS()
class BLASTERLEARING_API AJumpPickup : public APickup
{
 GENERATED_BODY()
protected:
 virtual void OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);

private:
 UPROPERTY(EditAnywhere)
 float JumpZVelocityBuff = 4000.f;
 UPROPERTY(EditAnywhere)
 float JumpBuffTime = 30.f;
};
```

## JumpPickup.cpp

```cpp
void AJumpPickup::OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
 Super::OnSphereOverlap(OverlappedComponent, OtherActor, OtherComp, OtherBodyIndex, bFromSweep, SweepResult);

//重叠角色。
 ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
 if (BlasterCharacter)
 {
    //角色的Buff组件。
  UBufferComponent* Buff = BlasterCharacter->GetBuff();
  if (Buff)
  {
    //调用增加角色跳跃能力。
   Buff->BuffJump(JumpZVelocityBuff, JumpBuffTime);
  }
 }
 //销毁对象。
 Destroy();
}
```
