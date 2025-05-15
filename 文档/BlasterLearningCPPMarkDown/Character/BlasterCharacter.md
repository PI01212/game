# BlasterCharacter

游戏的玩家控制的角色叫做BlasterCharacter，使用的是虚幻5引擎官方教程中的角色资源，再根据免费动画网站的动画进行骨骼重定向得到的相关动作动画。

![alt text](/img/component.png)
角色的主要组件为胶囊组件，静态网格，碰撞盒，相机和相机臂，手雷静态网格，头部显示小组件，溶解时间线组件，角色移动组件，延迟补偿组件，增益Buff组件，战斗组件。
胶囊组件的用途是为游戏中的角色或物体提供简单的碰撞检测和物理行为。
静态网格的用途是设置了角色使用的静态网格和添加了碰撞盒，为了后面服务器倒带使用，并且还把手雷静态网格附加在静态网格上，以便丢手雷的时候显示出来，相机臂也附加在网格上。伤害射线和射弹武器的判定都使用了SkeletalMesh而不再是胶囊，为了让判定更准确。
头部显示小组件在调试时显示的这个角色的信息，时代理还是权威，后续打算根据玩家的Steam名称来显示。
溶解时间线组件的用途是为了在角色死亡的时候播放死亡动画并且快速溶解，溶解就需要去实时改变控制材质溶解效果的参数。
角色移动组件用途负责角色移动相关的功能。
延迟补偿组件就是对服务器倒带和服务器预测的网络延迟方面功能的实现。
增益Buff组件的用途就是角色捡起加速，增加跳跃，增加生命，增加护盾等拾取道具的交互。
战斗组件负责角色射击相关功能的实现和管理。

首先介绍角色类相关的方法和代码，再去按照角色的组件来依次讲解每个组件的用途的和代码实现。

## 角色类BlasterCharacter.h主要内容

```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnLeftGame);

UCLASS()
class BLASTERLEARING_API ABlasterCharacter : public ACharacter, public IInteractWithCrosshairsInterface
{
 GENERATED_BODY()

public:
 ABlasterCharacter();

 virtual void Tick(float DeltaTime) override;
 virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;
 virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
 virtual void PostInitializeComponents() override;

 /*
 * 播放动画蒙太奇
 */
 void PlayFireMontage(bool Aiming);
 void PlayReloadMontage();
 void PlayElimMontage();
 void PlayThrowGrenadeMontage();
 void PlaySwapMontage();

 /*UFUNCTION(NetMulticast, Unreliable)
 void MulticastHit();*/

 virtual void OnRep_ReplicatedMovement() override;
 void Elim(bool bPlayerLeftGame);
 UFUNCTION(NetMulticast, Reliable)
 void MulticastElim(bool bPlayerLeftGame);
 virtual void Destroyed() override;

 UPROPERTY(Replicated)
 bool bDisableGameplay = false;

 UFUNCTION(BlueprintImplementableEvent)
 void ShowSniperScopeWidget(bool bShowScope);

 void UpdateHUDHealth();
 void UpdateHUDShield();
 void UpdateHUDAmmo();
 void UpdateHUDGrenade();

 void SpawnDefaultWeapon();

 UPROPERTY()
 TMap<FName, class UBoxComponent*> HitCollisionBoxes;

 bool bFinishedSwapping = false;

 UFUNCTION(Server, Reliable)
 void ServerLeaveGame();

 FOnLeftGame OnLeftGame;

 UFUNCTION(NetMulticast, Reliable)
 void MulticastGainedTheLead();

 UFUNCTION(NetMulticast, Reliable)
 void MulticastLostTheLead();

 void SetTeamColor(ETeam Team);

protected:
 virtual void BeginPlay() override;

 void MoveForward(float Value);
 void MoveRight(float Value);
 void LookUp(float Value);
 void Turn(float Value);
 void EquipButtonPressed();
 void CrouchButtonPressed();
 void ReloadButtonPressed();
 void AimButtonPressed();
 void AimButtonReleased();
 void AimOffset(float DeltaTime);
 void CalculateAO_Pitch();
 void SimProxiesTurn();
 virtual void Jump() override;
 void FireButtonPressed();
 void FireButtonReleased();
 void PlayHitReactMontage();
 void GrenadeButtonPressed();
 void DropOrDestroyWeapon(AWeapon* Weapon);
 void DropOrDestroyWeapons();
 void SetSpawnPoint();
 void OnPlayerStateInitialized();

 UFUNCTION()
 void ReceiveDamage(AActor* DamageActor, float Damage, const UDamageType* DamageType, class AController* InstigatorController, AActor* DamageCauser);
 //轮询获取相关类并初始化对应的HUD
 void PollInit();
 void RotateInPlace(float DeltaTime);

 /*
 * Hit Boxes(命中框用于服务器倒带时的命中判断)
 */

 UPROPERTY(EditAnywhere)
 UBoxComponent* head;

 UPROPERTY(EditAnywhere)
 UBoxComponent* pelvis;

 UPROPERTY(EditAnywhere)
 UBoxComponent* spine_02;

 UPROPERTY(EditAnywhere)
 UBoxComponent* spine_03;

 UPROPERTY(EditAnywhere)
 UBoxComponent* upperarm_l;

 UPROPERTY(EditAnywhere)
 UBoxComponent* upperarm_r;

 UPROPERTY(EditAnywhere)
 UBoxComponent* lowerarm_l;

 UPROPERTY(EditAnywhere)
 UBoxComponent* lowerarm_r;

 UPROPERTY(EditAnywhere)
 UBoxComponent* hand_l;

 UPROPERTY(EditAnywhere)
 UBoxComponent* hand_r;

 UPROPERTY(EditAnywhere)
 UBoxComponent* backpack;

 UPROPERTY(EditAnywhere)
 UBoxComponent* blanket;

 UPROPERTY(EditAnywhere)
 UBoxComponent* thigh_l;

 UPROPERTY(EditAnywhere)
 UBoxComponent* thigh_r;

 UPROPERTY(EditAnywhere)
 UBoxComponent* calf_l;

 UPROPERTY(EditAnywhere)
 UBoxComponent* calf_r;

 UPROPERTY(EditAnywhere)
 UBoxComponent* foot_l;

 UPROPERTY(EditAnywhere)
 UBoxComponent* foot_r;

private:
 UPROPERTY(VisibleAnyWhere, Category = Camera)
 class USpringArmComponent* CameraBoom;

 UPROPERTY(VisibleAnyWhere, Category = Camera)
 class UCameraComponent* FollowCamera;

 UPROPERTY(EditAnywhere, BlueprintReadOnly, meta = (AllowPrivateAccess = "true"))
 class UWidgetComponent* OverheadWidget;

 UPROPERTY(ReplicatedUsing = OnRep_OverlappingWeapon)
 class AWeapon* OverlappingWeapon;

 UFUNCTION()
 void OnRep_OverlappingWeapon(AWeapon* LastWeapon);

 UFUNCTION(Server, Reliable)
 void ServerEquipButtonPressed();

 /*
 * 角色组件
 */

 UPROPERTY(VisibleAnywhere, BlueprintReadOnly, meta = (AllowPrivateAccess = "true"))
 class UCombatComponent* Combat;

 UPROPERTY(VisibleAnywhere)
 class UBufferComponent* Buff;

 UPROPERTY(VisibleAnywhere)
 class ULagCompensationComponent* LagCompensation;

 float AO_Yaw;
 float InterpAO_Yaw;
 float AO_Pitch;
 FRotator StartingAimRotation;

 ETurningInPlace TurningInPlace;
 void TurnInPlace(float DeltaTime);

 /*
 * 动画蒙太奇
 */

 UPROPERTY(EditAnywhere, Category = Combat)
 class UAnimMontage* FireWeaponMontage;

 UPROPERTY(EditAnywhere, Category = Combat)
 UAnimMontage* ReloadMontage;

 UPROPERTY(EditAnywhere, Category = Combat)
 UAnimMontage* HitReactMontage;

 UPROPERTY(EditAnywhere, Category = Combat)
 UAnimMontage* ElimMontage;

 UPROPERTY(EditAnywhere, Category = Combat)
 UAnimMontage* ThrowGrenadeMontage;

 UPROPERTY(EditAnywhere, Category = Combat)
 UAnimMontage* SwapMontage;

 void HideCameraIfCharacterClose();

 UPROPERTY(EditAnywhere)
 float CameraThreshold = 200.f;

 bool bRotateRootBone;
 float TurnThreshold = 0.5f;
 FRotator ProxyRotationLastFrame;
 FRotator ProxyRotation;
 float ProxyYaw;
 float TimeSinceLastMovementReplication;
 float CalculateSpeed();

 /*
 * 玩家生命值
 */
 UPROPERTY(EditAnywhere, Category = "Player States")
 float MaxHealth = 100.f;

 UPROPERTY(ReplicatedUsing = OnRep_Health, VisibleAnywhere, Category = "Player States")
 float Health = 100.f;

 UFUNCTION()
 void OnRep_Health(float LastHealth);

 /*
 * 护盾
 */
 UPROPERTY(EditAnywhere, Category = "Player Stats")
 float MaxShield = 100.f;

 UPROPERTY(ReplicatedUsing = OnRep_Shield, VisibleAnywhere, Category = "Player Stats")
 float Shield = 0.f;

 UFUNCTION()
 void OnRep_Shield(float LastShield);

 UPROPERTY()
 class ABlasterPlayerController* BlasterPlayerController;

 bool bElimmed = false;

 FTimerHandle ElimTimer;

 UPROPERTY(EditDefaultsOnly)
 float ElimDelay = 3.f;

 void ElimTimerFinished();

 bool bLeftGame = false;

 /*
 * 溶解效果
 */

 UPROPERTY(VisibleAnywhere)
 UTimelineComponent* DissolveTimeline;
 FOnTimelineFloat DissolveTrack;

 UPROPERTY(EditAnywhere)
 UCurveFloat* DissolveCurve;

 UFUNCTION()
 void UpdateDissolveMaterial(float DissolveValue);
 void StartDissolve();

 //运行时更改的动态材质实例
 UPROPERTY(VisibleAnywhere, Category = Elim)
 UMaterialInstanceDynamic* DynamicDissolveMaterialInstance;

 //角色材质实例
 UPROPERTY(VisibleAnywhere, Category = Elim)
 UMaterialInstance* DissolveMaterialInstance;

 /*
 * 团队颜色
 */
 UPROPERTY(EditAnywhere, Category = Elim)
 UMaterialInstance* RedDissolveMathInst;

 UPROPERTY(EditAnywhere, Category = Elim)
 UMaterialInstance* RedMaterial;

 UPROPERTY(EditAnywhere, Category = Elim)
 UMaterialInstance* BlueDissolveMathInst;

 UPROPERTY(EditAnywhere, Category = Elim)
 UMaterialInstance* BlueMaterial;

 UPROPERTY(EditAnywhere, Category = Elim)
 UMaterialInstance* OriginalMaterial;

 /*
 * 淘汰
 */
 UPROPERTY(EditAnywhere)
 UParticleSystem* ElimBotEffect;

 UPROPERTY(VisibleAnywhere)
 UParticleSystemComponent* ElimBotComponent;

 UPROPERTY(EditAnywhere)
 class USoundCue* ElimBotSound;

 UPROPERTY()
 class ABlasterPlayerState* BlasterPlayerState;

 UPROPERTY(EditAnywhere)
 class UNiagaraSystem* CrownSystem;

 UPROPERTY()
 class UNiagaraComponent* CrownComponent;

 /*
 * 手雷
 */
 UPROPERTY(VisibleAnywhere)
 UStaticMeshComponent* AttachedGrenade;

 /*
 * 默认武器
 */
 UPROPERTY(EditAnywhere)
 TSubclassOf<AWeapon> DefaultWeaponClass;

 UPROPERTY()
 class ABlasterGameMode* BlasterGameMode;

public: 
 void SetOverlappingWeapon(AWeapon* Weapon);
 bool IsWeaponEquipped();
 bool IsAiming();
 FORCEINLINE float GetAO_Yaw() const { return AO_Yaw; }
 FORCEINLINE float GetAO_Pitch() const { return AO_Pitch; }
 AWeapon* GetEquippedWeapon();
 FORCEINLINE ETurningInPlace GetTurningInPlace() const { return TurningInPlace; }
 FVector GetHitTarget() const;
 FORCEINLINE UCameraComponent* GetFollowCamera() const { return FollowCamera; }
 FORCEINLINE bool ShouldRotateRootBone() const { return bRotateRootBone; }
 FORCEINLINE bool IsElimed() const { return bElimmed; }
 FORCEINLINE float GetHealth() const { return Health; }
 FORCEINLINE void SetHealth(float Amount) { Health = Amount; }
 FORCEINLINE float GetMaxHealth() const { return MaxHealth; }
 FORCEINLINE float GetShield() const { return Shield; }
 FORCEINLINE void SetShield(float Amount) { Shield = Amount; }
 FORCEINLINE float GetMaxShield() const { return MaxShield; }
 FORCEINLINE UCombatComponent* GetCombat() const { return Combat; }
 FORCEINLINE bool GetDisableGameplay() const { return bDisableGameplay; }
 FORCEINLINE UAnimMontage* GetReloadMontage() const { return ReloadMontage; }
 FORCEINLINE UStaticMeshComponent* GetAttachedGrenade() const { return AttachedGrenade; }
 FORCEINLINE UBufferComponent* GetBuff() const { return Buff; }
 FORCEINLINE ULagCompensationComponent* GetLagCompensation() const { return LagCompensation; }
 FORCEINLINE bool IsHoldingTheFlag() const;
 void SetHoldingTheFlag(bool bHolding);
 ECombatState GetCombatState() const;
 bool IsLocallyReloading();
 ETeam GetTeam();
};
```

## BlasterCharacter.cpp的主要方法实现

### 构造函数

```cpp
ABlasterCharacter::ABlasterCharacter()
{
 PrimaryActorTick.bCanEverTick = true;
 //作用是设置这个类可以每帧更新，角色需要玩家时刻控制，所以需要进行每帧更新，如果是不需要每帧更新的则可以false来提高性能。

 CameraBoom = CreateDefaultSubobject<USpringArmComponent>(TEXT("CameraBoom"));
 CameraBoom->SetupAttachment(GetMesh());
 CameraBoom->TargetArmLength = 600.f;
 CameraBoom->bUsePawnControlRotation = true;
 //创建相机臂组件，并添加到Mesh静态网格下，设置臂长600，使用角色控制器的旋转。这样相机将和角色控制器的旋转绑定，角度正确。

 FollowCamera = CreateDefaultSubobject<UCameraComponent>(TEXT("FollowCamera"));
 FollowCamera->SetupAttachment(CameraBoom, USpringArmComponent::SocketName);
 FollowCamera->bUsePawnControlRotation = false;
 //创建相机组件，把相机添加的相机臂的插槽上，这里设置不使用角色控制器旋转是因为相机已经添加到相机臂伤的插槽上。

 bUseControllerRotationYaw = false;
 GetCharacterMovement()->bOrientRotationToMovement = true;
 //角色不使用控制器这样角色才可以根据自己的角度旋转，否则角色将一直跟着控制器旋转角度，设置Orient为true是为了角色根据自己的旋转转向。

 OverheadWidget = CreateDefaultSubobject<UWidgetComponent>(TEXT("OverheadWidget"));
 OverheadWidget->SetupAttachment(RootComponent);
 //创建头部显示小组件并添加到根组件上。

 Combat = CreateDefaultSubobject<UCombatComponent>(TEXT("CombatComponent"));
 Combat->SetIsReplicated(true);
 //创建战斗组件，设置战斗组件为网络复制。关于战斗方面的功能主要都是在Combat组件中实现，并且需要客户端和服务器端是同步的才能保证正常的逻辑，所以设置此组件为复制。

 Buff = CreateDefaultSubobject<UBufferComponent>(TEXT("BuffComponent"));
 Buff->SetIsReplicated(true);
 //创建Buff组件，设置为网络复制。Buff同理，因为角色需要做到客户端和服务器端是同步的，所以设置为复制。

 LagCompensation = CreateDefaultSubobject<ULagCompensationComponent>(TEXT("LagCompensation"));
 //创建延迟补偿组件，负责角色后续延迟方面的功能。

 GetCharacterMovement()->NavAgentProps.bCanCrouch = true;
 //使用角色移动组件的蹲下功能。
 GetCapsuleComponent()->SetCollisionResponseToChannel(ECollisionChannel::ECC_Camera, ECollisionResponse::ECR_Ignore);
 //忽略胶囊和相机的碰撞，防止在一些情况和相机发生碰撞产生的突然镜头的缩进。
 GetMesh()->SetCollisionObjectType(ECC_SkeletalMesh);
//设置角色网格碰撞类型为自定义的碰撞ECC_SkeletalMesh，替换原本的胶囊和子弹的碰撞，这样瞄准伤害判定更精准。
 GetMesh()->SetCollisionResponseToChannel(ECollisionChannel::ECC_Camera, ECollisionResponse::ECR_Ignore);
 //忽略网格和相机的碰撞，防止在一些情况和相机发生碰撞产生的突然镜头的缩进。
 GetMesh()->SetCollisionResponseToChannel(ECollisionChannel::ECC_Visibility, ECollisionResponse::ECR_Block);
 //设置网格和可见性的对象碰撞。
 GetCharacterMovement()->RotationRate = FRotator(0.f, 0.f, 850.f);
//设置角色和Yaw的旋转速度为850。

 TurningInPlace = ETurningInPlace::ETIP_NotTurning;
 //设置转身枚举为不转身。
 NetUpdateFrequency = 66.f;
 MinNetUpdateFrequency = 33.f;
 //设置网络的更新频率为66，最小值33。这是现在的常规数值。

 DissolveTimeline = CreateDefaultSubobject<UTimelineComponent>(TEXT("DissolveTimelineComponent"));
 //创建溶解时间线组件。

 AttachedGrenade = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Attached Grenade"));
 AttachedGrenade->SetupAttachment(GetMesh(), FName("GrenadeSocket"));
 AttachedGrenade->SetCollisionEnabled(ECollisionEnabled::NoCollision);
 //创建手雷的静态网格，把网格添加到角色网格的GrenadeSocket的插槽上，并且设置无碰撞，防止和其他东西产生不必要的碰撞。

 /*
 * Hit Boxes(用于服务器倒带命中判断)
 * 为了服务器倒带时把角色完整的倒带会指定时间并进行伤害判定和计算。
 */

 head = CreateDefaultSubobject<UBoxComponent>(TEXT("head"));
 head->SetupAttachment(GetMesh(), FName("head"));
 HitCollisionBoxes.Add(FName("head"), head);

 pelvis = CreateDefaultSubobject<UBoxComponent>(TEXT("pelvis"));
 pelvis->SetupAttachment(GetMesh(), FName("pelvis"));
 HitCollisionBoxes.Add(FName("pelvis"), pelvis);

 spine_02 = CreateDefaultSubobject<UBoxComponent>(TEXT("spine_02"));
 spine_02->SetupAttachment(GetMesh(), FName("spine_02"));
 HitCollisionBoxes.Add(FName("spine_02"), spine_02);

 spine_03 = CreateDefaultSubobject<UBoxComponent>(TEXT("spine_03"));
 spine_03->SetupAttachment(GetMesh(), FName("spine_03"));
 HitCollisionBoxes.Add(FName("spine_03"), spine_03);

 upperarm_l = CreateDefaultSubobject<UBoxComponent>(TEXT("upperarm_l"));
 upperarm_l->SetupAttachment(GetMesh(), FName("upperarm_l"));
 HitCollisionBoxes.Add(FName("upperarm_l"), upperarm_l);

 upperarm_r = CreateDefaultSubobject<UBoxComponent>(TEXT("upperarm_r"));
 upperarm_r->SetupAttachment(GetMesh(), FName("upperarm_r"));
 HitCollisionBoxes.Add(FName("upperarm_r"), upperarm_r);

 lowerarm_l = CreateDefaultSubobject<UBoxComponent>(TEXT("lowerarm_l"));
 lowerarm_l->SetupAttachment(GetMesh(), FName("lowerarm_l"));
 HitCollisionBoxes.Add(FName("lowerarm_l"), lowerarm_l);

 lowerarm_r = CreateDefaultSubobject<UBoxComponent>(TEXT("lowerarm_r"));
 lowerarm_r->SetupAttachment(GetMesh(), FName("lowerarm_r"));
 HitCollisionBoxes.Add(FName("lowerarm_r"), lowerarm_r);

 hand_l = CreateDefaultSubobject<UBoxComponent>(TEXT("hand_l"));
 hand_l->SetupAttachment(GetMesh(), FName("hand_l"));
 HitCollisionBoxes.Add(FName("hand_l"), hand_l);

 hand_r = CreateDefaultSubobject<UBoxComponent>(TEXT("hand_r"));
 hand_r->SetupAttachment(GetMesh(), FName("hand_r"));
 HitCollisionBoxes.Add(FName("hand_r"), hand_r);

 blanket = CreateDefaultSubobject<UBoxComponent>(TEXT("blanket"));
 blanket->SetupAttachment(GetMesh(), FName("backpack"));
 HitCollisionBoxes.Add(FName("blanket"), blanket);

 backpack = CreateDefaultSubobject<UBoxComponent>(TEXT("backpack"));
 backpack->SetupAttachment(GetMesh(), FName("backpack"));
 HitCollisionBoxes.Add(FName("backpack"), backpack);

 thigh_l = CreateDefaultSubobject<UBoxComponent>(TEXT("thigh_l"));
 thigh_l->SetupAttachment(GetMesh(), FName("thigh_l"));
 HitCollisionBoxes.Add(FName("thigh_l"), thigh_l);

 thigh_r = CreateDefaultSubobject<UBoxComponent>(TEXT("thigh_r"));
 thigh_r->SetupAttachment(GetMesh(), FName("thigh_r"));
 HitCollisionBoxes.Add(FName("thigh_r"), thigh_r);

 calf_l = CreateDefaultSubobject<UBoxComponent>(TEXT("calf_l"));
 calf_l->SetupAttachment(GetMesh(), FName("calf_l"));
 HitCollisionBoxes.Add(FName("calf_l"), calf_l);

 calf_r = CreateDefaultSubobject<UBoxComponent>(TEXT("calf_r"));
 calf_r->SetupAttachment(GetMesh(), FName("calf_r"));
 HitCollisionBoxes.Add(FName("calf_r"), calf_r);

 foot_l = CreateDefaultSubobject<UBoxComponent>(TEXT("foot_l"));
 foot_l->SetupAttachment(GetMesh(), FName("foot_l"));
 HitCollisionBoxes.Add(FName("foot_l"), foot_l);

 foot_r = CreateDefaultSubobject<UBoxComponent>(TEXT("foot_r"));
 foot_r->SetupAttachment(GetMesh(), FName("foot_r"));
 HitCollisionBoxes.Add(FName("foot_r"), foot_r);

//循环存储在HitCollisionBoxes中的碰撞盒，HitCollisionBoxes是一个TMap,Key为FName,Value为碰撞盒。设置碰撞盒类型为ECC_HitBox，设置所有碰撞通道为忽略，再设置ECC_HitBox碰撞，之后其他使用ECC_HitBox类型碰撞的对象就可以和其发生碰撞，最后再暂时设置无碰撞，后续倒带的时候直接启用就可以了。
 for (auto& Box : HitCollisionBoxes)
 {
  if (Box.Value)
  {
   Box.Value->SetCollisionObjectType(ECC_HitBox);
   Box.Value->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
   Box.Value->SetCollisionResponseToChannel(ECC_HitBox, ECollisionResponse::ECR_Block);
   Box.Value->SetCollisionEnabled(ECollisionEnabled::NoCollision);
  }
 }
}
```

### BeginPlay

```cpp
void ABlasterCharacter::BeginPlay()
{
  //开始函数，调用生成默认武器。
 Super::BeginPlay();
 SpawnDefaultWeapon();
 
 //如果角色有权限，对击中事件动态绑定回调函数。
 if (HasAuthority())
 {
  OnTakeAnyDamage.AddDynamic(this, &ABlasterCharacter::ReceiveDamage);
 }
 //如果手雷网格存在，设置为不可见。
 if (AttachedGrenade)
 {
  AttachedGrenade->SetVisibility(false);
 }
}
```

### SetupPlayerInputComponent

```cpp
//设置角色的按输入绑定。
void ABlasterCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
 Super::SetupPlayerInputComponent(PlayerInputComponent);


 PlayerInputComponent->BindAxis("MoveForward", this, &ABlasterCharacter::MoveForward);
 PlayerInputComponent->BindAxis("MoveRight", this, &ABlasterCharacter::MoveRight);
 PlayerInputComponent->BindAxis("Turn", this, &ABlasterCharacter::Turn);
 PlayerInputComponent->BindAxis("LookUp", this, &ABlasterCharacter::LookUp);

 PlayerInputComponent->BindAction("Jump", IE_Pressed, this, &ABlasterCharacter::Jump);
 PlayerInputComponent->BindAction("Equip", IE_Pressed, this, &ABlasterCharacter::EquipButtonPressed);
 PlayerInputComponent->BindAction("Crouch", IE_Pressed, this, &ABlasterCharacter::CrouchButtonPressed);
 PlayerInputComponent->BindAction("Aim", IE_Pressed, this, &ABlasterCharacter::AimButtonPressed);
 PlayerInputComponent->BindAction("Aim", IE_Released, this, &ABlasterCharacter::AimButtonReleased);
 PlayerInputComponent->BindAction("Fire", IE_Pressed, this, &ABlasterCharacter::FireButtonPressed);
 PlayerInputComponent->BindAction("Fire", IE_Released, this, &ABlasterCharacter::FireButtonReleased);
 PlayerInputComponent->BindAction("Reload", IE_Pressed, this, &ABlasterCharacter::ReloadButtonPressed);
 PlayerInputComponent->BindAction("ThrowGrenade", IE_Pressed, this, &ABlasterCharacter::GrenadeButtonPressed);
}
```

### MoveForward

```cpp
void ABlasterCharacter::MoveForward(float Value)
{
    //如果可以游戏并且控制器不为空Value不为0，那么就开始移动。
 if (bDisableGameplay) return;
 if (Controller != nullptr && Value != 0.f)
 {
    //创建一个获得控制器Yaw旋转值的旋转器，根据这个旋转器转化为一个旋转矩阵并获得沿X轴的旋转方向向量，最后沿着这个方向向量加上Vlaue进行角色的移动。
  const FRotator YawRotation(0.f, Controller->GetControlRotation().Yaw, 0.f);
  const FVector Direction(FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X));
  AddMovementInput(Direction, Value);
 }
}
```

### MoveRight

```cpp
void ABlasterCharacter::MoveRight(float Value)
{
 if (bDisableGameplay) return;
 if (Controller != nullptr && Value != 0.f)
 {
    //创建一个获得控制器Yaw旋转值的旋转器，根据这个旋转器转化为一个旋转矩阵并获得沿Y轴的旋转方向向量，最后沿着这个方向向量加上Vlaue进行角色的移动。
  const FRotator YawRotation(0.f, Controller->GetControlRotation().Yaw, 0.f);
  const FVector Direction(FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y));
  AddMovementInput(Direction, Value);
 }
}
```

### Turn

```cpp
//沿Yaw轴进行旋转。
void ABlasterCharacter::Turn(float Value)
{
 AddControllerYawInput(Value);
}
```

### LookUp

```cpp
//沿Pitch轴进行旋转。
void ABlasterCharacter::LookUp(float Value)
{
 AddControllerPitchInput(Value);
}
```

### Jump

```cpp
void ABlasterCharacter::Jump()
{
    //判断战斗组件存在并且没有举旗，判断可以进行游戏。判断角色是否蹲伏，如果蹲下就站起，如果不是就调用跳跃。
 if (Combat && Combat->bHoldingTheFlag) return;
 if (bDisableGameplay) return;
 if (bIsCrouched)
 {
  UnCrouch();
 }
 else
 {
  Super::Jump();
 }
}
```

### Elim

```cpp
//角色死亡，掉落武器，调用多播死亡RPC，传入角色是否离开游戏。
void ABlasterCharacter::Elim(bool bPlayerLeftGame)
{
 DropOrDestroyWeapons();
 MulticastElim(bPlayerLeftGame);
}
```

### DropOrDestroyWeapons

```cpp
void ABlasterCharacter::DropOrDestroyWeapons()
{
  //如果战斗组件存在，调用丢弃武器或摧毁武器，传入战斗组件装备武器。
 if (Combat)
 {
  if (Combat->EquippedWeapon)
  {
   DropOrDestroyWeapon(Combat->EquippedWeapon);
  }
  //如果第二把武器存在，调用丢弃武器并传入第二把武器。
  if (Combat->SecondaryWeapon)
  {
   DropOrDestroyWeapon(Combat->SecondaryWeapon);
  }
  //如果旗帜存在，调用战斗组件的旗帜丢弃。
  if (Combat->TheFlag)
  {
   Combat->TheFlag->Dropped();
  }
 }
}
```

### DropOrDestroyWeapon

```cpp
void ABlasterCharacter::DropOrDestroyWeapon(AWeapon* Weapon)
{
  //如果武器为空就返回。
 if (Weapon == nullptr) return;
 //如果武器是要被摧毁的就调用摧毁。
 if (Weapon->bDestroyWeapon)
 {
  Weapon->Destroy();
 }
 //其他调用放下。
 else
 {
  Weapon->Dropped();
 }
}
```

### MulticastElim

```cpp
void ABlasterCharacter::MulticastElim_Implementation(bool bPlayerLeftGame)
{
  //是否离开游戏等于传入的值。
 bLeftGame = bPlayerLeftGame;
 //如果角色控制器存在调用设置武器Ammo等于0。
 if (BlasterPlayerController)
 {
  BlasterPlayerController->SetHUDWeaponAmmo(0);
 }
 //死亡等于true，播放死亡动画。
 bElimmed = true;
 PlayElimMontage();
 // 开始溶解效果
 if (DissolveMaterialInstance)
 {
  //使用溶解实例创建动态溶解实例，获得角色的网格设置使用动态溶解实例。设置动态实例的溶解强度和发光强度。
  DynamicDissolveMaterialInstance = UMaterialInstanceDynamic::Create(DissolveMaterialInstance, this);
  GetMesh()->SetMaterial(0, DynamicDissolveMaterialInstance);
  DynamicDissolveMaterialInstance->SetScalarParameterValue(TEXT("Dissolve"), 0.55f);
  DynamicDissolveMaterialInstance->SetScalarParameterValue(TEXT("Glow"), 200.f);
 }
 //开始溶解效果。
 StartDissolve();

 // 设置游戏不能游戏禁止玩家操作，禁用角色移动，设置角色开火按钮为false，停止开火。
 bDisableGameplay = true;
 GetCharacterMovement()->DisableMovement();
 if (Combat)
 {
  Combat->FireButtonPressed(false);
 }
 // 禁用碰撞
 GetCapsuleComponent()->SetCollisionEnabled(ECollisionEnabled::NoCollision);
 GetMesh()->SetCollisionEnabled(ECollisionEnabled::NoCollision);
 AttachedGrenade->SetCollisionEnabled(ECollisionEnabled::NoCollision);

 // 生成消除机器人，并把机器人放在玩家头顶。
 if (ElimBotEffect)
 {
  FVector ElimBotSpawnPoint(GetActorLocation().X, GetActorLocation().Y, GetActorLocation().Z + 200.f);
  ElimBotComponent = UGameplayStatics::SpawnEmitterAtLocation(
   GetWorld(),
   ElimBotEffect,
   ElimBotSpawnPoint,
   GetActorRotation()
  );
 }
 //如果机器人声音存在，生成播放机器人声音。
 if (ElimBotSound)
 {
  UGameplayStatics::SpawnSoundAtLocation(
   this,
   ElimBotSound,
   GetActorLocation()
  );
 }
 //获得是否正在使用狙击枪。
 bool bHideSniperScope = IsLocallyControlled() &&
  Combat &&
  Combat->bAiming &&
  Combat->EquippedWeapon &&
  Combat->EquippedWeapon->GetWeaponType() == EWeaponType::EWT_SniperRifle;
//关闭正在使用的狙击枪瞄准镜头。
 if (bHideSniperScope)
 {
  ShowSniperScopeWidget(false);
 }
 //如果角色戴着皇冠，摧毁皇冠。
 if (CrownComponent)
 {
  CrownComponent->DestroyComponent();
 }
 //使用TimerManager设置一个定时器，绑定回调函数为ElimTimerFinished，设置定时时间为ElimDelay，并把句柄存储在ElimTimer中。定时器到时执行回调函数。
 GetWorldTimerManager().SetTimer(
  ElimTimer,
  this,
  &ABlasterCharacter::ElimTimerFinished,
  ElimDelay
 );
}
```

### ServerLeaveGame

```cpp
//服务器离开游戏。
void ABlasterCharacter::ServerLeaveGame_Implementation()
{
  //获得游戏模式和玩家状态。
 BlasterGameMode = BlasterGameMode == nullptr ? GetWorld()->GetAuthGameMode<ABlasterGameMode>() : BlasterGameMode;
 BlasterPlayerState = BlasterPlayerState == nullptr ? GetPlayerState<ABlasterPlayerState>() : BlasterPlayerState;
 //如果都存在，调用游戏模式的玩家离开游戏方法并传入当前玩家状态。
 if (BlasterGameMode && BlasterPlayerState)
 {
  BlasterGameMode->PlayerLeftGame(BlasterPlayerState);
 }
}
```

### Destroyed

```cpp
//重写摧毁方法。
void ABlasterCharacter::Destroyed()
{
 Super::Destroyed();

//如果死亡机器人存在调用摧毁组件。
 if (ElimBotComponent)
 {
  ElimBotComponent->DestroyComponent();
 }

//获得游戏模式，设置比赛是否未开始。
 BlasterGameMode = BlasterGameMode == nullptr ? GetWorld()->GetAuthGameMode<ABlasterGameMode>() : BlasterGameMode;
 bool bMatchNotInProgress = BlasterGameMode && BlasterGameMode->GetMatchState() != MatchState::InProgress;
 //如果比赛未开始，角色装备武器，调用武器摧毁方法。
 if (Combat && Combat->EquippedWeapon && bMatchNotInProgress)
 {
  Combat->EquippedWeapon->Destroy();
 }
}
```

### ElimTimerFinished

```cpp
//角色死亡倒计时结束。
void ABlasterCharacter::ElimTimerFinished()
{
  //获取游戏模式，调用请求重生方法。
 BlasterGameMode = BlasterGameMode == nullptr ? GetWorld()->GetAuthGameMode<ABlasterGameMode>() : BlasterGameMode;
 if (BlasterGameMode && !bLeftGame)
 {
  BlasterGameMode->RequestRespawn(this, Controller);
 }
 //如果角色离开游戏并且时本地控制，广播委托OnLeftGame。
 if (bLeftGame && IsLocallyControlled())
 {
  OnLeftGame.Broadcast();
 }
}
```

### EquipButtonPressed

```cpp
void ABlasterCharacter::EquipButtonPressed()
{
 if (bDisableGameplay) return;
 if (Combat)
 {
  if (Combat->bHoldingTheFlag) return;
  //如果可以进行游戏，战斗组件存在并且没有举旗。战斗组件状态为空闲，则执行服务器端RPC装备武器。
  if (Combat->CombatState == ECombatState::ECS_Unoccuiped) ServerEquipButtonPressed();

  //能否进行切枪决定于战斗组件的能够切枪方法返回为true并不是服务器端并战斗状态为空闲并没有重叠武器。
  bool bSwap = Combat->ShouldSwapWeapons() &&
   !HasAuthority() &&
   Combat->CombatState == ECombatState::ECS_Unoccuiped &&
   OverlappingWeapon == nullptr;

//如果换弹为真，则播放切枪动画蒙太奇，战斗状态为切枪，完成切枪为false。
  if (bSwap)
  {
   PlaySwapMontage();
   Combat->CombatState = ECombatState::ECS_SwappingWeapon;
   bFinishedSwapping = false;
  }
 }
}
```

### ServerEquipButtonPressed

```cpp
void ABlasterCharacter::ServerEquipButtonPressed_Implementation()
{
 if (Combat)
 {
    //如果重叠武器就装备重叠武器。
  if (OverlappingWeapon)
  {
   Combat->EquipWeapon(OverlappingWeapon);
  }
  //如果没有，判断能否切换武器是否为true，为true则切换武器。
  else if (Combat->ShouldSwapWeapons())
  {
   Combat->SwapWeapons();
  }
 }
}
```

### CrouchButtonPressed

```cpp
void ABlasterCharacter::CrouchButtonPressed()
{
 if (Combat && Combat->bHoldingTheFlag) return;
 if (bDisableGameplay) return;
 //判断是否处于蹲伏状态，为true就站起，为false则蹲伏。
 if (bIsCrouched)
 {
  UnCrouch();
 }
 else
 {
  Crouch();
 }
}
```

### AimButtonPressed

```cpp
void ABlasterCharacter::AimButtonPressed()
{
 if (Combat && Combat->bHoldingTheFlag) return;
 if (bDisableGameplay) return;
 //设置瞄准为true。
 if (Combat)
 {
  Combat->SetAiming(true);
 }
}
```

### AimButtonReleased

```cpp
void ABlasterCharacter::AimButtonReleased()
{
 if (Combat && Combat->bHoldingTheFlag) return;
 if (bDisableGameplay) return;
 //设置瞄准为false。
 if (Combat)
 {
  Combat->SetAiming(false);
 }
}
```

### FireButtonPressed

```cpp
void ABlasterCharacter::FireButtonPressed()
{
 if (Combat && Combat->bHoldingTheFlag) return;
 if (bDisableGameplay) return;
 //按下开火按钮为true。
 if (Combat)
 {
  Combat->FireButtonPressed(true);
 }
}
```

### FireButtonReleased

```cpp
void ABlasterCharacter::FireButtonReleased()
{
 if (Combat && Combat->bHoldingTheFlag) return;
 if (bDisableGameplay) return;
 //按下开火按钮为false。
 if (Combat)
 {
  Combat->FireButtonPressed(false);
 }
}
```

### ReloadButtonPressed

```cpp
void ABlasterCharacter::ReloadButtonPressed()
{
 if (Combat && Combat->bHoldingTheFlag) return;
 if (bDisableGameplay) return;
 //执行装弹。
 if (Combat)
 {
  Combat->Reload();
 }
}
```

### GrenadeButtonPressed

```cpp
void ABlasterCharacter::GrenadeButtonPressed()
{
 if (Combat)
 {
  if (Combat->bHoldingTheFlag) return;
  //执行扔手雷。
  Combat->ThrowGrenade();
 }
}
```

### GetLifetimeReplicatedProps

```cpp
//所有声明为网络复制的参数都需要在这个方法中进行注册，否则无法正常使用。
void ABlasterCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
 Super::GetLifetimeReplicatedProps(OutLifetimeProps);

 DOREPLIFETIME_CONDITION(ABlasterCharacter, OverlappingWeapon, COND_OwnerOnly);
 DOREPLIFETIME(ABlasterCharacter, Health);
 DOREPLIFETIME(ABlasterCharacter, bDisableGameplay);
 DOREPLIFETIME(ABlasterCharacter, Shield);
}
```

### PostInitializeComponents

```cpp
//在所有组件都初始化完毕之后调用，为组件初始化成功之后执行需要的设置。
void ABlasterCharacter::PostInitializeComponents()
{
 Super::PostInitializeComponents();
 //设置战斗组件的角色为当前类。
 if (Combat)
 {
  Combat->Character = this;
 }
 //设置Buff组件的角色为当前类，设置角色的初始化移动速度和蹲下移动速度，设置角色初始跳跃速度。
 if (Buff)
 {
  Buff->Character = this;
  Buff->SetInitialSpeeds(GetCharacterMovement()->MaxWalkSpeed, GetCharacterMovement()->MaxWalkSpeedCrouched);
  Buff->SetInitialJumpVelocity(GetCharacterMovement()->JumpZVelocity);
 }
 //设置延迟补偿组件的角色类当前类，控制器为当前类控制器。
 if (LagCompensation)
 {
  LagCompensation->Character = this;
  if (Controller)
  {
   LagCompensation->Controller = Cast<ABlasterPlayerController>(Controller);
  }
 }
}
```

### PlayFireMontage

```cpp
//播放开火蒙太奇
void ABlasterCharacter::PlayFireMontage(bool bAiming)
{
    //首先判断角色是否有战斗组件并且装备武器
 if (Combat == nullptr || Combat->EquippedWeapon == nullptr) return;

//获得动画实例，如果实例和开火蒙太奇都存在那就播放开火蒙太奇动画，并且判断角色是否在瞄准，根据这个结果动画直接跳转到设置好的片段中。
 UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
 if (AnimInstance && FireWeaponMontage)
 {
  AnimInstance->Montage_Play(FireWeaponMontage);
  FName SectionName;
  SectionName = bAiming ? FName("RifleAim") : FName("RifleHip");
  AnimInstance->Montage_JumpToSection(SectionName);
 }
}
```

### PlayReloadMontage

```cpp
void ABlasterCharacter::PlayReloadMontage()
{
 if (Combat == nullptr || Combat->EquippedWeapon == nullptr) return;

 UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
 if (AnimInstance && ReloadMontage)
 {
  AnimInstance->Montage_Play(ReloadMontage);
  FName SectionName;

//通过战斗组件获得当前装备的武器是什么类型。
  switch (Combat->EquippedWeapon->GetWeaponType())
  {
  case EWeaponType::EWT_AssaultRifle:
   SectionName = FName("Rifle");
   break;
  case EWeaponType::EWT_RocketLauncher:
   SectionName = FName("RocketLauncher");
   break;
  case EWeaponType::EWT_Pistol:
   SectionName = FName("Pistol");
   break;
  case EWeaponType::EWT_SMG:
   SectionName = FName("Pistol");
   break;
  case EWeaponType::EWT_Shotgun:
   SectionName = FName("Shotgun");
   break;
  case EWeaponType::EWT_SniperRifle:
   SectionName = FName("SniperRifle");
   break;
  case EWeaponType::EWT_GrenadeLauncher:
   SectionName = FName("GrenadeLauncher");
   break;
  }

//根据不同武器类型动画跳转到对应武器类型的不同片段。
  AnimInstance->Montage_JumpToSection(SectionName);
 }
}
```

### PlayElimMontage

```cpp
void ABlasterCharacter::PlayElimMontage()
{
    //播放死亡蒙太奇动画。
 UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
 if (AnimInstance && ElimMontage)
 {
  AnimInstance->Montage_Play(ElimMontage);
 }
}
```

### PlayThrowGrenadeMontage

```cpp

void ABlasterCharacter::PlayThrowGrenadeMontage()
{
    //播放扔手雷蒙太奇动画。
 UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
 if (AnimInstance && ThrowGrenadeMontage)
 {
  AnimInstance->Montage_Play(ThrowGrenadeMontage);
 }
}
```

### PlaySwapMontage

```cpp
void ABlasterCharacter::PlaySwapMontage()
{
    //播放切枪蒙太奇。
 UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
 if (AnimInstance && SwapMontage)
 {
  AnimInstance->Montage_Play(SwapMontage);
 }
}
```

### PlayHitReactMontage

```cpp
void ABlasterCharacter::PlayHitReactMontage()
{
 if (Combat == nullptr || Combat->EquippedWeapon == nullptr) return;

//播放蒙太奇并跳转到对应的片段。
 UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
 if (AnimInstance && HitReactMontage)
 {
  AnimInstance->Montage_Play(HitReactMontage);
  FName SectionName("FromFront");
  AnimInstance->Montage_JumpToSection(SectionName);
 }
}
```

### Tick

```cpp
void ABlasterCharacter::Tick(float DeltaTime)
{
    //调用父类Tick函数，调用原地转向函数，角色靠近相机隐藏角色和武器函数，轮询初始化函数。
 Super::Tick(DeltaTime);

 RotateInPlace(DeltaTime);
 HideCameraIfCharacterClose();
 PollInit();
}
```

### RotateInPlace

```cpp
void ABlasterCharacter::RotateInPlace(float DeltaTime)
{
   //如果角色现在举着旗子，就设置不使用控制旋转，朝旋转转向开启，转向设置为不转向。角色就会类似没有携带武器一样的移动方式。因为角色不需要瞄准所以这样的移动方式更自然。
 if (Combat && Combat->bHoldingTheFlag)
 {
  bUseControllerRotationYaw = false;
  GetCharacterMovement()->bOrientRotationToMovement = true;
  TurningInPlace = ETurningInPlace::ETIP_NotTurning;
  return;
 }
 //如果就角色装备武器，则需要设置为相反的方式，固定视角使用瞄准进行射击。
 if (Combat && Combat->EquippedWeapon) GetCharacterMovement()->bOrientRotationToMovement = false;
 if (Combat && Combat->EquippedWeapon) bUseControllerRotationYaw = true;
 //如果不能游戏代表游戏结束，不再使用控制器旋转，解除视角限制，转向设置为不转向。玩家可以自由移动视角，但不能做其他事情，只能等待游戏开始。
 if (bDisableGameplay)
 {
  bUseControllerRotationYaw = false;
  TurningInPlace = ETurningInPlace::ETIP_NotTurning;
  return;
 }
 //如果当前的角色大于模拟代理，代表这个角色要么是个权威角色，要么是个本地自主代理，只有这两种才可能是个本地控制的角色，之后再判断是否是本地控制。如果是就执行瞄准偏移。
 if (GetLocalRole() > ENetRole::ROLE_SimulatedProxy && IsLocallyControlled())
 {
  AimOffset(DeltaTime);
 }
 else
 {
   //如果是模拟代理，不是本地控制。也就是每隔0.25f，显式调用一次OnRep_ReplicatedMovement，每帧执行CalculateAO_Pitch()。设置成这样的原因是因为，我们在全身姿势当中使用了旋转根骨头，而旋转根骨头的变化在我们本地控制时，旋转角度取决于我们的输入，也就是可以做到每帧都进行更新，所以旋转角度可以做到很平滑。但是，模拟代理的更新则来自及网络复制，网络复制的速度不可能做到每帧都更新，并且网络复制也不会是连续稳定的，取决于网路的实际情况，这就会造成模拟代理角色抖动，因为模拟代理角色就不能使用本地控制的瞄准偏移。最后再去执行CalculateAO_Pitch。
  TimeSinceLastMovementReplication += DeltaTime;
  if (TimeSinceLastMovementReplication > 0.25f)
  {
   OnRep_ReplicatedMovement();
  }
  //每帧都执行是因为Pitch需要时刻保证值是正确的。
  CalculateAO_Pitch();
 }
}
```

### OnRep_ReplicatedMovement

```cpp
void ABlasterCharacter::OnRep_ReplicatedMovement()
{
   //调用父类OnRep_ReplicatedMovement，调用SimProxiesTurn函数，更新时间归零。我们的BlasterCharacter类层层继承自Actor类，Actor类中的ReplicatedMovement是一个复制参数，正是因为其是网络复制参数，我们的角色的移动才在服务器端和客户端做到了同步移动，OnRep_ReplicatedMovement就是其通知函数。我们在这里实现它添加我们自己的逻辑。显式调用其的原因是，网络复制决定于服务器端参数的改变来通知客户端，因此服务器永远不会自己去执行通知，所以服务器端的非本地控制的角色只会改变旋转角度但不会执行转向，我们显示调用就是为了每隔0.25f时间去执行一次OnRep_ReplicatedMovement中的转向逻辑，这样服务器端也会在旋转的时候播放转向方面的动画逻辑。客户端显示调用是因为网络复制是由参数改变发生的，但并不是稳定且规律执行的，取决于当前网络和参数的变化，如果延迟过高那么网络复制就会很慢和不稳定，所以显示调用也是为了执行转向逻辑，让角色旋转时播放转向动画，使过渡更自然。
 Super::OnRep_ReplicatedMovement();
 SimProxiesTurn();
 TimeSinceLastMovementReplication = 0.f;
}
```

### SimProxiesTurn

```cpp
{
   //如果角色携带武器就把旋转根骨头设置位false。
 if (Combat == nullptr || Combat->EquippedWeapon == nullptr) return;
 bRotateRootBone = false;
 //获得速度，如果速度大于0说明角色在移动，就不转向。
 float Speed = CalculateSpeed();
 if (Speed > 0.f)
 {
  TurningInPlace = ETurningInPlace::ETIP_NotTurning;
  return;
 }
//计算上一帧和当前帧旋转的增量，并保存。之前SimProxiesTurn函数是在Tick函数中调用的，现在却放在在了OnRep_ReplicatedMovement通知中去调用。因为现在需要使用网络复制同步改变这个Yaw，如果还是在Tick函数中调用，那么造成的问题就是，Tick函数是每帧运行的，而网络复制频率低于Tick函数的每帧运行，根本无法达到。会出现的情况就是ProxyYaw中间会穿插很多0，这是因为网络复制还没有同步过来，所以计算出的增量肯定是0，那实际上判定的结果是角色根本没有旋转，那底下的转向逻辑也不会生效。而现在改为OnRep_ReplicatedMovement中调用，则代表，要么是已经完成了复制，要么是隔了0.25f的时间，这样保证了计算ProxYaw时保存的上一帧和当前帧肯定是有差值了，所以得出的ProxyYaw不会再穿插0，转向逻辑就可以正常计算结果执行。
 ProxyRotationLastFrame = ProxyRotation;
 ProxyRotation = GetActorRotation();
 ProxyYaw = UKismetMathLibrary::NormalizedDeltaRotator(ProxyRotation, ProxyRotationLastFrame).Yaw;

//如果ProxyYaw绝对值大于TurnThreshold代表该进行转向逻辑。
 if (FMath::Abs(ProxyYaw) > TurnThreshold)
 {
   //如果本身大于TurnThreshold代表向右转。
  if (ProxyYaw > TurnThreshold)
  {
   TurningInPlace = ETurningInPlace::ETIP_Right;
  }
  else if (ProxyYaw < -TurnThreshold)
  {
   //如果大于负，该进行左转逻辑。
   TurningInPlace = ETurningInPlace::ETIP_Left;
  }
  /*else
  {
   TurningInPlace = ETurningInPlace::ETIP_NotTurning;
  }*/
  return;
 }
 //如果不大于，代表还没有达到转向的值，就不进行转向，为默认状态。并且当ProxyYaw绝对值不再大于转向TurnThreshold时，设置为不转向，因为动画蓝图中的转向状态到站立的状态条件为判断枚举是否为不转向，不设置会导致角色一直处在转向状态。
 TurningInPlace = ETurningInPlace::ETIP_NotTurning;
}
```

### CalculateSpeed

```cpp
float ABlasterCharacter::CalculateSpeed()
{
   //获得角色的速度，并把Z的速度设置为0，也就是只考虑角色的X,Y的速度并返回值。
 FVector Velocity = GetVelocity();
 Velocity.Z = 0.f;
 return Velocity.Size();
}
```

### CalculateAO_Pitch

```cpp
void ABlasterCharacter::CalculateAO_Pitch()
{
   //获得Pitch角度，平判断是否90度和非本地控制
 AO_Pitch = GetBaseAimRotation().Pitch;
 if (AO_Pitch > 90.f && !IsLocallyControlled())
 {
  // 将map俯仰角从 [270, 360) 转换为 [-90, 0)
  FVector2D InRange(270.f, 360.f);
  FVector2D OutRange(-90.f, 0.f);
  //如果条件成立需要对Pitch进行映射才是正确的角度。
  AO_Pitch = FMath::GetMappedRangeValueClamped(InRange, OutRange, AO_Pitch);
 }
}
```

### Pitch在网络传输中的压缩

![alt text](/img/image.png)
![alt text](/img/image-1.png)
![alt text](/img/image-2.png)
GetPackagedAngles函数的作用是为了去创建用于网络传输的包，这些包中存储着这些旋转的数据，但为了速度，这些数据都会进行压缩。PackYawAndPitchTo32函数就是为了去对这些值进行压缩，把float压缩为uint32类型的无符号整数。而使用的就是CompressAxisToShort函数。它会把Angle乘以65536.f再除以360.f，而为什么是65536，因为65535是16位能表达的最大值，所以注解当中使用的）而不是]。即代表不包括65536。再使用RoundToInt对结果进行四舍五入变为一个带符号的整数，再逐位进行按位与运算,把结果映射为[0->65536)这个范围的数值,结果就变为了一个这个范围的Uint16无符号整数。而为什么是16，因为Yaw加上Picth组合起来成为Rotation32进行传输，所以它俩各占16位，也就有(YawShort << 16) | PitchShort;这步操作，把YawShort左移16位。
这样就保证了结果在传输中速度和同步，所以我们需要对结果进行重新映射才能显示为正确的角度。我们不管是使用瞄准偏移还是网络复制，这个Pitch的转换都是每帧运行的原因就是为了保证角度时刻都是正确的值。

### AimOffset

```cpp
void ABlasterCharacter::AimOffset(float DeltaTime)
{
 if (Combat && Combat->EquippedWeapon == nullptr) return;
 //获得速度和是否在空中。
 float Speed = CalculateSpeed();
 bool bIsInAir = GetCharacterMovement()->IsFalling();

 if (Speed == 0.f && !bIsInAir) // 静止不动，不跳跃
 {
   //设置旋转根骨头为true，以保证角色不会进行旋转，获得当前的瞄准旋转，计算当前真的瞄准旋转和开始时的瞄准旋转的增量，让AO_Yaw等于增量的Yaw，AO_Yaw就是动画实例中用来判断在混合空间如何进行动画过渡的值瞄准偏移。
  bRotateRootBone = true;
  FRotator CurrentAimRotation = FRotator(0.f, GetBaseAimRotation().Yaw, 0.f);
  FRotator DeltaAimRotation = UKismetMathLibrary::NormalizedDeltaRotator(CurrentAimRotation, StartingAimRotation);
  AO_Yaw = DeltaAimRotation.Yaw;
  //如果转向枚举等于不转向，就让插值AO_Yaw等于AO_Yaw。
  if (TurningInPlace == ETurningInPlace::ETIP_NotTurning)
  {
   InterpAO_Yaw = AO_Yaw;
  }
  //角色使用控制器旋转，这样加上设置在蓝图中的旋转根骨头的值可以让角色朝向保持静止。再调用原地转向。
  bUseControllerRotationYaw = true;
  TurnInPlace(DeltaTime);
 }
 if (Speed > 0.f || bIsInAir) // 跑或者跳
 {
   //如果速度大于零或者在空中，就不使用旋转根骨头，角色使用控制器旋转，这样做的目的就可以让角色跟随着镜头瞄准的朝向。设置开始瞄准旋转等于当前旋转，这样角色在静止时的朝向正好指向中心对应于蓝图混合空间中的默认状态动画，再通过上面的计算增量的方法就可以合理的计算出瞄准偏移的AO_Yaw，通过混合空间过渡动画。再设置转向枚举为不旋转代表角色不进行转向。
  bRotateRootBone = false;
  StartingAimRotation = FRotator(0.f, GetBaseAimRotation().Yaw, 0.f);
  AO_Yaw = 0.f;
  bUseControllerRotationYaw = true;
  TurningInPlace = ETurningInPlace::ETIP_NotTurning;
 }
 //时刻调用Pitch纠正函数。
 CalculateAO_Pitch();
}
```

### TurnInPlace

```cpp
void ABlasterCharacter::TurnInPlace(float DeltaTime)
{
   //如果AO_Yaw大于90度，代表角色的偏移已经足够大，改进行转向，所以改变转向枚举的值，蓝图就会根据这个改变的值播放状态机中的右转动画。
 if (AO_Yaw > 90.f)
 {
  TurningInPlace = ETurningInPlace::ETIP_Right;
 }
 //如果AO_Yaw大于-90度，代表角色的偏移已经足够大，改进行转向，所以改变转向枚举的值，蓝图就会根据这个改变的值播放状态机中的左转动画。
 else if (AO_Yaw < -90.f)
 {
  TurningInPlace = ETurningInPlace::ETIP_Left;
 }
 //如果转向枚举不等于不转向，代表角色正在进行转向的逻辑当中。所以在游戏的角度，角色处于一个瞄准朝向和身体朝向慢慢回正的过程，也就是身体朝瞄准的方向去旋转，代表这个瞄准偏移越来越小，所以我们这里对之前的AO_Yaw进行插值，而进行插值的参数就是之前保存的专门用于插值的InterpAO_Yaw，再设置AO_Yaw等于InterpAO_Yaw，这样角色的瞄准偏移就会越来越小，混合空间中的动画也会随之改变，当AO_Yaw小于设置的15.f时，就设置转向等于不转向代表转向过程结束，并把转向后现在角色的朝向设置为开始瞄准旋转。
 if (TurningInPlace != ETurningInPlace::ETIP_NotTurning)
 {
  InterpAO_Yaw = FMath::FInterpTo(InterpAO_Yaw, 0.f, DeltaTime, 4.f);
  AO_Yaw = InterpAO_Yaw;
  if (FMath::Abs(AO_Yaw) < 15.f)
  {
   TurningInPlace = ETurningInPlace::ETIP_NotTurning;
   StartingAimRotation = FRotator(0.f, GetBaseAimRotation().Yaw, 0.f);
  }
 }
}
```

### HideCameraIfCharacterClose

```cpp
void ABlasterCharacter::HideCameraIfCharacterClose()
{
  //判断角色是否是本地控制再执行后续操作，计算相机的位置减去角色的位置并求值，如果小于设置的阈值，就隐藏角色的网格，如果装备主武器和服务器就把武器网格也隐藏。
 if (!IsLocallyControlled()) return;
 if ((FollowCamera->GetComponentLocation() - GetActorLocation()).Size() < CameraThreshold)
 {
  GetMesh()->SetVisibility(false);
  if (Combat && Combat->EquippedWeapon && Combat->EquippedWeapon->GetWeaponMesh())
  {
   Combat->EquippedWeapon->GetWeaponMesh()->bOwnerNoSee = true;
  }
  if (Combat && Combat->SecondaryWeapon && Combat->SecondaryWeapon->GetWeaponMesh())
  {
   Combat->SecondaryWeapon->GetWeaponMesh()->bOwnerNoSee = true;
  }
 }
 //如果是大于就把角色和武器的网格显示回来。这样做的目的就是在角色在一切墙角或者视角转动过大太靠近角色时遮挡玩家的镜头。
 else
 {
  GetMesh()->SetVisibility(true);
  if (Combat && Combat->EquippedWeapon && Combat->EquippedWeapon->GetWeaponMesh())
  {
   Combat->EquippedWeapon->GetWeaponMesh()->bOwnerNoSee = false;
  }
  if (Combat && Combat->SecondaryWeapon && Combat->SecondaryWeapon->GetWeaponMesh())
  {
   Combat->SecondaryWeapon->GetWeaponMesh()->bOwnerNoSee = false;
  }
 }
}
```

### PollInit

```cpp
void ABlasterCharacter::PollInit()
{
  //如果角色状态为空就获得角色状态并转换为ABlasterPlayerState类型。
 if (BlasterPlayerState == nullptr)
 {
  BlasterPlayerState = GetPlayerState<ABlasterPlayerState>();
  //如果角色状态存在,执行角色状态初始化。
  if (BlasterPlayerState)
  {
   OnPlayerStateInitialized();

//从游戏玩法静态类中获取游戏状态并转换为ABlasterGameState类型。如果当前的玩家状态和游戏状态中当前得分最高的玩家一直，就调用多播RPCMulticastGainedTheLead为所有客户端和服务器上的这个角色添加皇冠。
   ABlasterGameState* BlasterGameState = Cast<ABlasterGameState>(UGameplayStatics::GetGameState(this));
   if (BlasterGameState && BlasterGameState->TopScoringPlayers.Contains(BlasterPlayerState))
   {
    MulticastGainedTheLead();
   }
  }
 }
 //如果角色的控制器为空，通过把Controller转换为ABlasterPlayerController类型获得控制器，否则就就等于它本身。
 if (BlasterPlayerController == nullptr)
 {
  BlasterPlayerController = BlasterPlayerController == nullptr ? Cast<ABlasterPlayerController>(Controller) : BlasterPlayerController;
  //如果角色控制器存在就执行相关HUD的更新。
  if (BlasterPlayerController)
  {
   UpdateHUDAmmo();
   UpdateHUDHealth();
   UpdateHUDShield();
   UpdateHUDGrenade();
  }
 }
}
```

### OnPlayerStateInitialized

```cpp
//调用增加分数0.f，调用死亡次数0次。设置团队颜色，设置重生点。
void ABlasterCharacter::OnPlayerStateInitialized()
{
 BlasterPlayerState->AddToScore(0.f);
 BlasterPlayerState->AddToDefeats(0);
 SetTeamColor(BlasterPlayerState->GetTeam());
 SetSpawnPoint();
}
```

### SetTeamColor

```cpp
void ABlasterCharacter::SetTeamColor(ETeam Team)
{
  //判断角色的网格和初始材质是否为空。
 if (GetMesh() == nullptr || OriginalMaterial == nullptr) return;
 //判断Team的值，根据值的不同设置不同的网格材质和溶解材质。
 switch (Team)
 {
 case ETeam::ET_RedTeam:
  GetMesh()->SetMaterial(0, RedMaterial);
  DissolveMaterialInstance = RedDissolveMathInst;
  break;
 case ETeam::ET_BlueTeam:
  GetMesh()->SetMaterial(0, BlueMaterial);
  DissolveMaterialInstance = BlueDissolveMathInst;
  break;
 case ETeam::ET_NoTeam:
  GetMesh()->SetMaterial(0, OriginalMaterial);
  DissolveMaterialInstance = BlueDissolveMathInst;
  break;
 }
}
```

### SetSpawnPoint

```cpp
void ABlasterCharacter::SetSpawnPoint()
{
  //如果具有权限，玩家状态获得的团队不等于无团队。
 if (HasAuthority() && BlasterPlayerState->GetTeam() != ETeam::ET_NoTeam)
 {
  //创建一个玩家开始数组,获得所有玩家开始。
  TArray<AActor*> PlayerStarts;
  UGameplayStatics::GetAllActorsOfClass(this, ATeamPlayerStart::StaticClass(), PlayerStarts);
  //创建一个团队玩家开始数组，循环玩家开始数组，把玩家开始数组转换为ATeamPlayerStart类型，判断转换的TeamStar的团队和当前角色状态的团队是否一致，一致代表是该团队的出生点，就把这个TeamStar添加到团队玩家开始的数组中。
  TArray<ATeamPlayerStart*> TeamPlayerStarts;
  for (auto Start : PlayerStarts)
  {
   ATeamPlayerStart* TeamStart = Cast<ATeamPlayerStart>(Start);
   if (TeamStart && TeamStart->Team == BlasterPlayerState->GetTeam())
   {
    TeamPlayerStarts.Add(TeamStart);
   }
  }
  //如果团队玩家开始数组大于0，代表有可以移动的团队出生点，随机选择一个团队出生点，设置玩家的旋转和位置等于这个出生点的旋转和位置，也就是传送玩家到出生点。
  if (TeamPlayerStarts.Num() > 0)
  {
   ATeamPlayerStart* ChosenPlayerStart = TeamPlayerStarts[FMath::RandRange(0, TeamPlayerStarts.Num() - 1)];
   SetActorLocationAndRotation(
    ChosenPlayerStart->GetActorLocation(),
    ChosenPlayerStart->GetActorRotation()
   );
  }
 }
}
```

### MulticastGainedTheLead

```cpp
void ABlasterCharacter::MulticastGainedTheLead_Implementation()
{
  //如果皇冠系统为空就返回。
 if (CrownSystem == nullptr) return;
 //如果皇冠组件为空，生成皇冠特效，附加到角色的网格上，插槽默认空，位置为角色的位置增加110.f，使用角色的旋转。不自动生成特效。
 if (CrownComponent == nullptr)
 {
  CrownComponent = UNiagaraFunctionLibrary::SpawnSystemAttached(
   CrownSystem,
   GetMesh(),
   FName(),
   GetActorLocation() + FVector(0.f, 0.f, 110.f),
   GetActorRotation(),
   EAttachLocation::KeepWorldPosition,
   false
  );
 }
 //如果存在，直接激活特效使其可见。
 if (CrownComponent)
 {
  CrownComponent->Activate();
 }
}
```

### MulticastLostTheLead

```cpp
//如果皇冠组件存在调用摧毁。
void ABlasterCharacter::MulticastLostTheLead_Implementation()
{
 if (CrownComponent)
 {
  CrownComponent->DestroyComponent();
 }
}
```

### UpdateHUDAmmo

```cpp
void ABlasterCharacter::UpdateHUDAmmo()
{
  //获得玩家控制器，调用设置携带弹药HUD，武器子弹HUD。
 BlasterPlayerController = BlasterPlayerController == nullptr ? Cast<ABlasterPlayerController>(Controller) : BlasterPlayerController;
 if (BlasterPlayerController && Combat && Combat->EquippedWeapon)
 {
  BlasterPlayerController->SetHUDCarriedAmmo(Combat->CarriedAmmo);
  BlasterPlayerController->SetHUDWeaponAmmo(Combat->EquippedWeapon->GetAmmo());
 }
}
```

### UpdateHUDHealth

```cpp
void ABlasterCharacter::UpdateHUDHealth()
{
  //获取玩家控制器，调用设置健康HUD。
 BlasterPlayerController = BlasterPlayerController == nullptr ? Cast<ABlasterPlayerController>(Controller) : BlasterPlayerController;
 if (BlasterPlayerController)
 {
  BlasterPlayerController->SetHUDHealth(Health, MaxHealth);
 }
}
```

### UpdateHUDShield

```cpp
void ABlasterCharacter::UpdateHUDShield()
{
  //获取玩家控制器，调用设置玩家护盾HUD。
 BlasterPlayerController = BlasterPlayerController == nullptr ? Cast<ABlasterPlayerController>(Controller) : BlasterPlayerController;
 if (BlasterPlayerController)
 {
  BlasterPlayerController->SetHUDShield(Shield, MaxShield);
 }
}
```

### UpdateHUDGrenade

```cpp
void ABlasterCharacter::UpdateHUDGrenade()
{
  //获得玩家控制器，调用设置手雷数量HUD。
 BlasterPlayerController = BlasterPlayerController == nullptr ? Cast<ABlasterPlayerController>(Controller) : BlasterPlayerController;
 if (BlasterPlayerController)
 {
  BlasterPlayerController->SetHUDGrenades(Combat->GetGrenades());
 }
}
```

### ReceiveDamage

```cpp
void ABlasterCharacter::ReceiveDamage(AActor* DamagedActor, float Damage, const UDamageType* DamageType, AController* InstigatorController, AActor* DamageCauser)
{
  //获得游戏模式，如果角色死亡或者游戏模式为空就提前返回。
 BlasterGameMode = BlasterGameMode == nullptr ? GetWorld()->GetAuthGameMode<ABlasterGameMode>() : BlasterGameMode;
 if (bElimmed || BlasterGameMode == nullptr) return;
 //从游戏模式中计算实际应该造成的伤害。
 Damage = BlasterGameMode->CalculateDamage(InstigatorController, Controller, Damage);

//定义对血量造成的伤害等于伤害。
 float DamageToHealth = Damage;
 //如果护盾大于0.f，代表伤害可以先由护盾承受。
 if (Shield > 0.f)
 {
  //如果护盾大于等于伤害，代表伤害全部由护盾承受，所以护盾值等于护盾减去伤害，限制计算后的值必须在0到最大护盾值之间。对血量造成的伤害等于0。
  if (Shield >= Damage)
  {
   Shield = FMath::Clamp(Shield - Damage, 0.f, MaxShield);
   DamageToHealth = 0.f;
  }
  //代表护盾值小于伤害，所以需要护盾和血量一起进行承受。对血量造成的伤害等于之前的值减去护盾值，并限制值必须在0和伤害之间，护盾值归零。
  else
  {
   DamageToHealth = FMath::Clamp(DamageToHealth - Shield, 0.f, Damage);
   Shield = 0.f;
  }
 }

//血量等于血量减去血量承受的伤害的值，结果限制在0到最大血量之间。
 Health = FMath::Clamp(Health - DamageToHealth, 0.f, MaxHealth);

//更新血量条，护盾条，播放被击中动画。
 UpdateHUDHealth();
 UpdateHUDShield();
 PlayHitReactMontage();

//如果血量等于0，并且游戏状态存在。
 if (Health == 0.f)
 {
  if (BlasterGameMode)
  {
    //获得玩家控制器，攻击者控制器。调用游戏模式中的玩家死亡方法并传入当前角色类，玩家控制器，攻击者控制器。执行相关的角色死亡逻辑。
   BlasterPlayerController = BlasterPlayerController == nullptr ? Cast<ABlasterPlayerController>(Controller) : BlasterPlayerController;
   ABlasterPlayerController* AttackerController = Cast<ABlasterPlayerController>(InstigatorController);
   BlasterGameMode->PlayerEliminated(this, BlasterPlayerController, AttackerController);
  }
 }
}
```

### OnRep_Health

```cpp
//血量通知函数，当网络复制参数血量改变时自动在客户端调用，虽然是不能携带参数的，但其实此函数会保留复制参数改变之前的值，我们可以使用这个值和改变之后的值进行比较来实现需要进行的更新逻辑。
void ABlasterCharacter::OnRep_Health(float LastHealth)
{
  //更新血量HUD。
 UpdateHUDHealth();
 //如果血量小于之前的血量，代表角色受到伤害，播放被击中动画。
 if (Health < LastHealth)
 {
  PlayHitReactMontage();
 }
}
```

### OnRep_Shield

```cpp
//护盾值改变时自动在客户端调用的通知函数。
void ABlasterCharacter::OnRep_Shield(float LastShield)
{
  //更新护盾HUD。
 UpdateHUDShield();
 //如果护盾值小于改变前的值，代表角色被击中，播放被击中动画。
 if (Shield < LastShield)
 {
  PlayHitReactMontage();
 }
}
```

### SpawnDefaultWeapon

```cpp
//生成默认武器，为什么要生成默认武器，因为现在游戏除了射击伤害就是手雷伤害，但需要拿起武器才能进行，如果只是把武器放在地图中，角色迟迟找到武器，就会被其他玩家打还没有还手机会挫败感很大，现在很多大逃杀游戏都加了开始的默认武器，比如使命召唤的战区不管是1还是2，都是有一把手枪的，所以我们也将使用手枪作为初始武器。
void ABlasterCharacter::SpawnDefaultWeapon()
{
  //获得游戏模式，获得当前世界。
 BlasterGameMode = BlasterGameMode == nullptr ? GetWorld()->GetAuthGameMode<ABlasterGameMode>() : BlasterGameMode;
 UWorld* World = GetWorld();
 //如果游戏模式和世界为真并且玩家没有死亡和存在默认武器类。那就生成开始武器，设置开始武器的摧毁武器为true，代表角色死亡时，武器跟随角色一起被摧毁。这样保证地图中不会因为角色死亡次数的积累而掉落大量的初始武器。
 if (BlasterGameMode && World && !bElimmed && DefaultWeaponClass) 
 {
  AWeapon* StartingWEeapon = World->SpawnActor<AWeapon>(DefaultWeaponClass);
  StartingWEeapon->bDestroyWeapon = true;
  //如果战斗组件存在，调用装备武器方法装备当前武器。
   if (Combat)
   {
    Combat->EquipWeapon(StartingWEeapon);
   }
 }
}
```

### UpdateDissolveMaterial

```cpp
//更新材质溶解。
void ABlasterCharacter::UpdateDissolveMaterial(float DissolveValue)
{
  //如果动态溶解材质是咧存在。
 if (DynamicDissolveMaterialInstance)
 {
  //调用设置ScalarParameterValue，改变名为Dissolve的值，这个值是用来控制使用材质溶解度的值。
  DynamicDissolveMaterialInstance->SetScalarParameterValue(TEXT("Dissolve"), DissolveValue);
 }
}
```

### StartDissolve

```cpp
//开始溶解。
void ABlasterCharacter::StartDissolve()
{
  //DissolveTrack是一个时间线浮点数的委托，用来绑定一个回调函数，用来在时间线更新的过程中，调用绑定函数并传递当前插值。这里是动态绑定UpdateDissolveMaterial作为回调函数
 DissolveTrack.BindDynamic(this, &ABlasterCharacter::UpdateDissolveMaterial);
 //如果溶解曲线，和时间线组件存在，把DissolveCurve溶解曲线添加到时间线中，和DissolveTrack委托绑定起来，并在时间线更新的过程中获取当前的插值并调用回调函数UpdateDissolveMateria传入插值。播放时间线DissolveTimeline。
 if (DissolveCurve && DissolveTimeline)
 {
  DissolveTimeline->AddInterpFloat(DissolveCurve, DissolveTrack);
  DissolveTimeline->Play();
 }
}
```

### SetOverlappingWeapon

```cpp
//设置重叠的武器。
void ABlasterCharacter::SetOverlappingWeapon(AWeapon* Weapon)
{
  //如果重叠武器存在,就把重叠武器的拾取小部件设置为flase不显示。改变重叠武器等于传入的武器。
 if (OverlappingWeapon)
 {
  OverlappingWeapon->ShowPickupWidget(false);
 }
 OverlappingWeapon = Weapon;
 //如果是本地控制，并且重叠武器存在，那么就显示武器的拾取小部件。
 if (IsLocallyControlled())
 {
  if (OverlappingWeapon)
  {
   OverlappingWeapon->ShowPickupWidget(true);
  }
 }
}
```

### OnRep_OverlappingWeapon

```cpp
//网络复制参数重叠武器变化时，自动在客户端进行调用。
void ABlasterCharacter::OnRep_OverlappingWeapon(AWeapon* LastWeapon)
{
//如果重叠武器存在，就显示武器的拾取小部件。这里为什么没有本地控制判断之类的条件呢？因为我们在对OverlappingWeapon注册的时候已经设置了这个参数只有拥有者才会复制，也就是说当服务器端的角色和武器发生了重叠事件更新了这个OverlappingWeapon，那么OverlappingWeapon的复制只会更新到这个角色的拥有者所属的客户端上，也就是拾取小组件只会显示在控制这个角色的客户端上。
 if (OverlappingWeapon)
 {
  OverlappingWeapon->ShowPickupWidget(true);
 }
 //如果改变之前的武器存在，则设置不显示之前的武器拾取小部件。
 if (LastWeapon)
 {
  LastWeapon->ShowPickupWidget(false);
 }
}
```

### IsWeaponEquipped

```cpp
//根据战斗组件的值判断是否装备武器。
bool ABlasterCharacter::IsWeaponEquipped()
{
 return (Combat && Combat->EquippedWeapon);
}
```

### IsAiming

```cpp
//从战斗组件的值判断是否在瞄准。
bool ABlasterCharacter::IsAiming()
{
 return (Combat && Combat->bAiming);
}
```

### GetEquippedWeapon

```cpp
//从战斗组件获得装备的武器。
AWeapon* ABlasterCharacter::GetEquippedWeapon()
{
 if (Combat == nullptr) return nullptr;
 return Combat->EquippedWeapon;
}
```

### GetHitTarget

```cpp
//从战斗组件得到击中目标。
FVector ABlasterCharacter::GetHitTarget() const
{
 if (Combat == nullptr) return FVector();
 return Combat->HitTarget;
}
```

### GetCombatState

```cpp
//从战斗组件得到战斗状态。
ECombatState ABlasterCharacter::GetCombatState() const
{
 if (Combat == nullptr) return ECombatState::ECS_MAX;
 return Combat->CombatState;
}
```

### IsLocallyReloading

```cpp
//从战斗组件得到是否正在本地换弹。
bool ABlasterCharacter::IsLocallyReloading()
{
 if (Combat == nullptr) return false;
 return Combat->bLocallyReloading;
}
```

### GetTeam

```cpp
//获得玩家状态，并从玩家状态获得所属团队。
ETeam ABlasterCharacter::GetTeam()
{
 BlasterPlayerState = BlasterPlayerState == nullptr ? GetPlayerState<ABlasterPlayerState>() : BlasterPlayerState;
 if (BlasterPlayerState == nullptr) return ETeam::ET_NoTeam;
 return BlasterPlayerState->GetTeam();
}
```

### IsHoldingTheFlag

```cpp
//从战斗组件获得是否正在举旗。
bool ABlasterCharacter::IsHoldingTheFlag() const
{
 if (Combat == nullptr) return false;
 return Combat->bHoldingTheFlag;
}
```

### SetHoldingTheFlag

```cpp
//设置正在举旗，设置战斗组件的正在举旗等于传入的bHolding。
void ABlasterCharacter::SetHoldingTheFlag(bool bHolding)
{
 if (Combat == nullptr) return;
 Combat->bHoldingTheFlag = bHolding;
}
```
