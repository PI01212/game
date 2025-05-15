# RocketMovementComponent

火箭移动组件。

## RocketMovementComponent.h

```cpp
UCLASS()
class BLASTERLEARING_API URocketMovementComponent : public UProjectileMovementComponent
{
 GENERATED_BODY()
 
protected:
 virtual EHandleBlockingHitResult HandleBlockingHit(const FHitResult& Hit, float TimeTick, const FVector& MoveDelta, float& SubTickTimeRemaining) override;

 virtual void HandleImpact(const FHitResult& Hit, float TimeSlice = 0.f, const FVector& MoveDelta = FVector::ZeroVector) override;
};
```

## RocketMovementComponent.cpp

### HandleBlockingHit

```cpp
//重写此方法，就是为了当火箭弹触发碰撞之后继续飞行而不是停下，所以当触发碰撞之后，这里手动设置返回继续枚举。并且由于我们在OnHit回调函数中我们设置了如果碰撞的角色等于自己就直接返回不执行其他步骤，所以火箭弹遇到自己就会继续飞行也不会触发碰撞函数的后续逻辑。
URocketMovementComponent::EHandleBlockingHitResult URocketMovementComponent::HandleBlockingHit(const FHitResult& Hit, float TimeTick, const FVector& MoveDelta, float& SubTickTimeRemaining)
{
 Super::HandleBlockingHit(Hit, TimeTick, MoveDelta, SubTickTimeRemaining);
 return EHandleBlockingHitResult::AdvanceNextSubstep;
}
```

### HandleImpact

```cpp
//重写此方法，此方法的用于是碰撞时触发的一个函数，可以对对象做一些操作，这里设置为空就是为了火箭弹继续飞行。不做其他操作。
void URocketMovementComponent::HandleImpact(const FHitResult& Hit, float TimeSlice, const FVector& MoveDelta)
{
 //火箭不应该停止，只有在碰撞盒检测击中时才会爆炸
}
```
