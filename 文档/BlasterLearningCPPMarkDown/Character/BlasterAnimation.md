# BlasterAnimInstance

## BlasterAnimInstance.h

动画实例就是玩家控制的角色的动画，用于当前角色，通过蓝图接收到这些值，实现相应的动画逻辑的判断。

```cpp
UCLASS()
class BLASTERLEARING_API UBlasterAnimInstance : public UAnimInstance
{
 GENERATED_BODY()
public:
 virtual void NativeInitializeAnimation() override;
 virtual void NativeUpdateAnimation(float DeltaTime) override;

private:
 UPROPERTY(BlueprintReadOnly, Category = Character, meta = (AllowPrivateAccess = "true"))
 class ABlasterCharacter* BlasterCharacter;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 float Speed;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 bool bIsInAir;

 //是否在加速
 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 bool bIsAccelerating;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 bool bWeaponEquipped;

 class AWeapon* EquippedWeapon;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 bool bIsCrouched;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 bool bAiming;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 float YawOffset;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 float Lean;

 FRotator CharacterRotationLastFrame;
 FRotator CharacterRotation;
 FRotator DeltaRotation;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 float AO_Yaw;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 float AO_Pitch;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 FTransform LeftHandTransform;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 ETurningInPlace TurningInPlace;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 FRotator RightHandRotation;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 bool bLocallyController;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 bool bRotateRootBone;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 bool bElimmed;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 bool bUseFABRIK;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 bool bUseAimOffsets;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 bool bTransformRightHand;

 UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
 bool bHoldingTheFlag;
};
```

## BlasterAnimInstance.cpp的主要方法实现

### NativeInitializeAnimation

```cpp
//初始化动画实例类，获得当前类的所有者也就是角色实例对象。
void UBlasterAnimInstance::NativeInitializeAnimation()
{
 Super::NativeInitializeAnimation();

 BlasterCharacter = Cast<ABlasterCharacter>(TryGetPawnOwner());
}
```

### NativeUpdateAnimation

```cpp
//每帧调用，更新动画实例。
void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
{
 Super::NativeUpdateAnimation(DeltaTime);
//如果角色实例为空，再次获得角色实例类。
 if (BlasterCharacter == nullptr)
 {
  BlasterCharacter = Cast<ABlasterCharacter>(TryGetPawnOwner());
 }
 //获得失败直接返回。
 if (BlasterCharacter == nullptr) return;

//创建速度向量，把z方向的速度归零，只考虑x，y方向的速度，计算速度的值Speed用于动画实例蓝图根据这个值过渡动画。
 FVector Velocity = BlasterCharacter->GetVelocity();
 Velocity.Z = 0.f;
 Speed = Velocity.Size();

//获得角色是否在空中，角色是否在加速，是否装备武器，获得装备的武器，是否蹲下，是否瞄准，转向的值，是否旋转根骨头。是否死亡，是否举旗。
 bIsInAir = BlasterCharacter->GetCharacterMovement()->IsFalling();
 bIsAccelerating = BlasterCharacter->GetCharacterMovement()->GetCurrentAcceleration().Size() > 0.f ? true : false;
 bWeaponEquipped = BlasterCharacter->IsWeaponEquipped();
 EquippedWeapon = BlasterCharacter->GetEquippedWeapon();
 bIsCrouched = BlasterCharacter->bIsCrouched;
 bAiming = BlasterCharacter->IsAiming();
 TurningInPlace = BlasterCharacter->GetTurningInPlace();
 bRotateRootBone = BlasterCharacter->ShouldRotateRootBone();
 bElimmed = BlasterCharacter->IsElimed();
 bHoldingTheFlag = BlasterCharacter->IsHoldingTheFlag();

 //启动偏航角偏移
 //获得瞄准的方向。
 FRotator AimRotation = BlasterCharacter->GetBaseAimRotation();

//获得移动的方向，对瞄准和移动的方向进行增量计算，对计算出的这个增量DeltaRot进行插值。让YawOffset等于插值DeltaRotation.Yaw。这样做是为了让过渡动画更平滑而不是直接转换，虚幻5编辑器中也有可以去设置Smoothing time，但副作用就导致-180到180的插值是从左到右这样插值过去的，那就会经过所有的动画，会造成抽动，而使用插值函数则是直接最短路径插值，也就是从-180直接到180，不是从-180到0再到180。不会出现编辑器中的副作用。
 FRotator MovementRotation = UKismetMathLibrary::MakeRotFromX(BlasterCharacter->GetVelocity());
 FRotator DeltaRot = UKismetMathLibrary::NormalizedDeltaRotator(MovementRotation, AimRotation);
 DeltaRotation = FMath::RInterpTo(DeltaRotation, DeltaRot, DeltaTime, 6.f);
 YawOffset = DeltaRotation.Yaw;

//计算最后一帧的旋转，当前帧的旋转等于角色旋转。计算它俩之间的增量。增量Delta的Yaw就是我们需要的沿Yaw的倾斜值，通过这个判断Lean的值来过渡混合空间中的动画。除以增量时间是为了放大这个值，再对值进行插值让过渡动画更平滑，再把Lean的值限定在-90到90之间。
 CharacterRotationLastFrame = CharacterRotation;
 CharacterRotation = BlasterCharacter->GetActorRotation();
 const FRotator Delta = UKismetMathLibrary::NormalizedDeltaRotator(CharacterRotation, CharacterRotationLastFrame);
 const float Target = Delta.Yaw / DeltaTime;
 const float Interp = FMath::FInterpTo(Lean, Target, DeltaTime, 6.f);
 Lean = FMath::Clamp(Interp, -90.f, 90.f);

//从角色类获得瞄准偏移。
 AO_Yaw = BlasterCharacter->GetAO_Yaw();
 AO_Pitch = BlasterCharacter->GetAO_Pitch();

//如果装备武器，能得到武器的网格和角色的网格。
 if (bWeaponEquipped && EquippedWeapon && EquippedWeapon->GetWeaponMesh() && BlasterCharacter->GetMesh())
 {
    //获得左手插槽的变换。创建输出位置和旋转，把左手插槽的世界变换转变换位基于右手的骨骼空间变换，并存储在输出位置和旋转中，设置左手插槽变换为输出的位置和旋转。
  LeftHandTransform = EquippedWeapon->GetWeaponMesh()->GetSocketTransform(FName("LeftHandSocket"), ERelativeTransformSpace::RTS_World);
  FVector OutPosition;
  FRotator OutRotation;
  BlasterCharacter->GetMesh()->TransformToBoneSpace(FName("hand_r"), LeftHandTransform.GetLocation(), FRotator::ZeroRotator, OutPosition, OutRotation);
  LeftHandTransform.SetLocation(OutPosition);
  LeftHandTransform.SetRotation(FQuat(OutRotation));

//如果角色是本地控制，就设置是否是本地控制器为true，再获得右手的变换。通过函数计算右手变换到击中目标变换的旋转值，再对右手旋转和看向的目标旋转进行插值。再在蓝图中使用这个插值去替换原本的右手旋转值，这样就可以保证角色武器指向的角度和瞄准的角度基本一致。在开火时不会出现明显的不同。
//为何是RightHandTransform.GetLocation() + (RightHandTransform.GetLocation() - BlasterCharacter->GetHitTarget())的原因，是因为在全身骨骼的预览中，右手的x轴方向是指向肩膀，也就是和武器的指向是相反的，如果是直接在函数中使用击中目标，就会导致角色的手臂旋转是反转的，而这样计算，就是先计算击中目标到右手变换的方向向量，这样得到的结果就是原本的右手到目标的反向，再让右手向量加上这个反向向量就成了原本的目标的反向延伸，如此得到的旋转就可以符合x轴因为指向手臂而造成的反向扭曲。
//通过武器网格获得右手骨骼是因为AttackActor已经把武器添加到角色网格了，所以已经形成关联。
  if (BlasterCharacter->IsLocallyControlled())
  {
  bLocallyController = true;
  FTransform RightHandTransform = EquippedWeapon->GetWeaponMesh()->GetSocketTransform(FName("Hand_R"), ERelativeTransformSpace::RTS_World);
  FRotator LookAtRotation = UKismetMathLibrary::FindLookAtRotation(RightHandTransform.GetLocation(), RightHandTransform.GetLocation() + (RightHandTransform.GetLocation() - BlasterCharacter->GetHitTarget()));
  RightHandRotation = FMath::RInterpTo(RightHandRotation, LookAtRotation, DeltaTime, 30.f);
  }

 }
//判断是否该使用FABRIK。
 bUseFABRIK = BlasterCharacter->GetCombatState() == ECombatState::ECS_Unoccuiped;
 //判断FABRIK覆盖的值，因为如果是网络延迟较大的情况，FABRIK并不能被正确的传递，所以需对这样的情况进行判断。这时会对是否使用FABRIK重新判断。
 bool bFABRIKOverride = BlasterCharacter->IsLocallyControlled() && 
  BlasterCharacter->GetCombatState() != ECombatState::ECS_ThrowingGrenade && 
  BlasterCharacter->bFinishedSwapping;
  //如果FABRIKOverride为真，是否使用FABRIK等于角色是否在进行本地换弹中。
 if (bFABRIKOverride)
 {
  bUseFABRIK = !BlasterCharacter->IsLocallyReloading();
 }
 //判断是否使用瞄准偏移。
 bUseAimOffsets = BlasterCharacter->GetCombatState() == ECombatState::ECS_Unoccuiped && !BlasterCharacter->GetDisableGameplay();
 //判断是否使用右手变换。
 bTransformRightHand = BlasterCharacter->GetCombatState() == ECombatState::ECS_Unoccuiped && !BlasterCharacter->GetDisableGameplay();
}
```

### FABRIK

![alt text](/img/image-3.png)
![alt text](/img/image-4.png)
上面两张图是使用和不使用的区别，我们默认的动画中，角色的左手是无法做到正确的握持在枪管的正确位置上的，所以就通过使用FABRIK来对骨骼的位置进行修改让左手在我们设置的武器插槽这里，而从左手的上臂到左手，引擎会自动帮我计算相对于右手该怎么变换自然。上面的代码部分已经求出左手插槽的位置相对于右手空间的变换。
![alt text](/img/image-5.png)
Effector Target设置为右手，Transfrom Space设置为骨骼空间，Rotation Scoure设置为no change,代表目标为右手，变换为右手的骨骼空间，并且不改变旋转源只是改变位置。
Tip Bone 为左手，Root Bone为上臂，代表要改变的骨骼链为左手上臂到左手。LeftHandTransform连接Effector Tranform表示Tip Bone左手的变换改变为左手插槽的值，而FABRIK的逻辑就是其会通过LeftHandTransform这个值自动帮我调整骨骼链的位置和旋转，最终让骨骼链末端Tip Bone也就是左手到达这个LeftHandTransform目标位置。
