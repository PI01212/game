# LagCompensationComponent

延迟补偿组件，客户端角色的组件，用于角色补偿相关功能的实现。

## LagCompensationComponent.h

```cpp
USTRUCT(BlueprintType)
struct FBoxInformation
{
 GENERATED_BODY()

 UPROPERTY()
 FVector Location;

 UPROPERTY()
 FRotator Rotation;

 UPROPERTY()
 FVector BoxExtent;
};

USTRUCT(BlueprintType)
struct FFramePackage
{
 GENERATED_BODY()

 UPROPERTY()
 float Time;

 UPROPERTY()
 TMap<FName, FBoxInformation> HitBoxInfo;

 UPROPERTY()
 ABlasterCharacter* Character;
};

USTRUCT(BlueprintType)
struct FServerSideRewindResult
{
 GENERATED_BODY()

 UPROPERTY()
 bool bHitConfirmed;

 UPROPERTY()
 bool bHeadShot;
};

USTRUCT(BlueprintType)
struct FShotgunServerSideRewindResult
{
 GENERATED_BODY()

 UPROPERTY()
 TMap<ABlasterCharacter*, uint32> HeadShots;

 UPROPERTY()
 TMap<ABlasterCharacter*, uint32> BodyShots;
};

UCLASS( ClassGroup=(Custom), meta=(BlueprintSpawnableComponent) )
class BLASTERLEARING_API ULagCompensationComponent : public UActorComponent
{
 GENERATED_BODY()

public: 
 ULagCompensationComponent();
 friend class ABlasterCharacter;
 virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;
 void ShowFramePackage(const FFramePackage& Package, const FColor& Color);

 /*
 * 扫描武器
 */
 FServerSideRewindResult ServerSideRewind(class ABlasterCharacter* HitCharacter, const FVector_NetQuantize& TraceStart, const FVector_NetQuantize& HitLocation, float HitTime);

 /*
 *射弹武器
 */
 FServerSideRewindResult ProjectileServerSideRewind(ABlasterCharacter* HitCharacter, const FVector_NetQuantize& TraceStart, const FVector_NetQuantize100& InitialVelocity, float HitTime);

 /*
 * 霰弹枪
 */
 FShotgunServerSideRewindResult ShotgunServerSideRewind(const TArray<ABlasterCharacter*>& HitCharacters, const FVector_NetQuantize& TraceStart, const TArray<FVector_NetQuantize>& HitLocations, float HitTime);

 UFUNCTION(Server, Reliable)
 void ServerScoreRequest(ABlasterCharacter* HitCharacter, const FVector_NetQuantize& TraceStart, const FVector_NetQuantize& HitLocation, float HitTime);

 UFUNCTION(Server, Reliable)
 void ProjectileServerScoreRequest(ABlasterCharacter* HitCharacter, const FVector_NetQuantize& TraceStart, const FVector_NetQuantize100& InitialVelocity, float HitTime);

 UFUNCTION(Server, Reliable)
 void ShotgunServerScoreRequest(const TArray<ABlasterCharacter*>& HitCharacters, const FVector_NetQuantize& TraceStart, const TArray<FVector_NetQuantize>& HitLocations, float HitTime);

protected:
 virtual void BeginPlay() override;
 void SaveFramePackage(FFramePackage& Package);
 FFramePackage InterpBetweenFrames(const FFramePackage& OlderFrame, const FFramePackage& YoungerFrame, float HitTime);
 void CacheBoxPositions(ABlasterCharacter* HitCharacter, FFramePackage& OutFramePackage);
 void MoveBoxes(ABlasterCharacter* HitCharacter, const FFramePackage& Package);
 void ResetHitBoxes(ABlasterCharacter* HitCharacter, const FFramePackage& Package);
 void EnableCharacterMeshCollision(ABlasterCharacter* HitCharacter, ECollisionEnabled::Type CollisionEnabled);
 void SaveFramePackageServer();
 FFramePackage GetFrameToCheck(ABlasterCharacter* HitCharacter, float HitTime);

 /*
 * 扫描武器
 */
 FServerSideRewindResult ConfirmHit(const FFramePackage& Package, ABlasterCharacter* HitCharacter, const FVector_NetQuantize& TraceStart, const FVector_NetQuantize& HitLocation);

 /*
 * 射弹武器
 */
 FServerSideRewindResult ProjectileConfirmHit(const FFramePackage& Package, ABlasterCharacter* HitCharacter, const FVector_NetQuantize& TraceStart, const FVector_NetQuantize100& InitialVelocity, float HitTime);

 /*
 * 霰弹枪倒带
 */
 FShotgunServerSideRewindResult ShotgunConfirmHit(const TArray<FFramePackage>& FramePackages, const FVector_NetQuantize& TraceStart, const TArray<FVector_NetQuantize>& HitLocations);

private:
 UPROPERTY()
 ABlasterCharacter* Character;

 UPROPERTY()
 class ABlasterPlayerController* Controller;

 TDoubleLinkedList<FFramePackage> FrameHistory;

 UPROPERTY(EditAnywhere)
 float MaxRecordTime = 4.f;

public: 
};
```

## LagCompensationComponen.cpp

### 构造函数

```cpp
ULagCompensationComponent::ULagCompensationComponent()
{
 PrimaryComponentTick.bCanEverTick = true;
}
```

### BeginPlay

```cpp
void ULagCompensationComponent::BeginPlay()
{
 Super::BeginPlay();
}
```

### TickComponent

```cpp
void ULagCompensationComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
 Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

//调用服务器保存帧数据包。
 SaveFramePackageServer();
}
```

### SaveFramePackageServer

```cpp
void ULagCompensationComponent::SaveFramePackageServer()
{
    //如果不是权威直接返回。
 if (Character == nullptr || !Character->HasAuthority()) return;
 //如果小于等于1代表帧历史中只保存了最多一帧。
 if (FrameHistory.Num() <= 1)
 {
    //创建此帧的帧数据包。
  FFramePackage ThisFrame;
  //调用保存帧数据包。
  SaveFramePackage(ThisFrame);
  //在帧历史中添加这一帧的帧数据包。
  FrameHistory.AddHead(ThisFrame);
 }
 else
 {
    //计算保存的帧数据包的时间长度，最新保存的帧时间也就是链表的开头减去链表保存最晚的帧时间也就是表尾。
  float HistoryLength = FrameHistory.GetHead()->GetValue().Time - FrameHistory.GetTail()->GetValue().Time;
  //while当这个时间大于设置的最大保存时间时。
  while (HistoryLength > MaxRecordTime)
  {
    //从帧数据包历史中移除表尾的帧历史。
   FrameHistory.RemoveNode(FrameHistory.GetTail());
   //再重新计算保存的时间长度。直至结果保存的时间长度不再大于规定的时间中。
   HistoryLength = FrameHistory.GetHead()->GetValue().Time - FrameHistory.GetTail()->GetValue().Time;
  }
  //创建这一帧的帧数据包。
  FFramePackage ThisFrame;
  //保存帧数据包。
  SaveFramePackage(ThisFrame);
  //添加到帧数据包历史中。
  FrameHistory.AddHead(ThisFrame);
 }
}
```

### SaveFramePackage

```cpp
void ULagCompensationComponent::SaveFramePackage(FFramePackage& Package)
{
    //获得所有者角色。
 Character = Character == nullptr ? Cast<ABlasterCharacter>(GetOwner()) : Character;
 if (Character)
 {
    //帧包的历史等于现在的时间。帧包所属的角色等于获得角色。
  Package.Time = GetWorld()->GetTimeSeconds();
  Package.Character = Character;
  //循环角色的碰撞盒。
  for (auto& BoxPair : Character->HitCollisionBoxes)
  {
    //创建碰撞盒信息结构体。
   FBoxInformation BoxInformation;
   //碰撞盒信息的位置等于角色此时的位置。旋转等于此时的旋转，碰撞盒的范围等于角色碰撞盒的范围。
   BoxInformation.Location = BoxPair.Value->GetComponentLocation();
   BoxInformation.Rotation = BoxPair.Value->GetComponentRotation();
   BoxInformation.BoxExtent = BoxPair.Value->GetScaledBoxExtent();
   //把碰撞盒信息存储在帧数据包中。
   Package.HitBoxInfo.Add(BoxPair.Key, BoxInformation);
  }
 }
}
```

### ServerScoreRequest

```cpp
//服务器分数得分请求，也就是客户端请求使用服务器倒带判断得分。
void ULagCompensationComponent::ServerScoreRequest_Implementation(ABlasterCharacter* HitCharacter, const FVector_NetQuantize& TraceStart, const FVector_NetQuantize& HitLocation, float HitTime)
{
    //创建服务器倒带结果结构体命中结果，调用服务器倒带判断是否命中。
 FServerSideRewindResult Confirm = ServerSideRewind(HitCharacter, TraceStart, HitLocation, HitTime);
 //如果角色和击中角色存在，并且装备武器和确认命中。
 if (Character && HitCharacter && Character->GetEquippedWeapon() && Confirm.bHitConfirmed)
 {
    //伤害等于判断是否击中头部，如果击中就使用角色装备的武器的爆头伤害，否则就是装备武器的基础伤害。
  const float Damage = Confirm.bHeadShot ? Character->GetEquippedWeapon()->GetHeadShotDamage() : Character->GetEquippedWeapon()->GetDamage();

//施加伤害。
  UGameplayStatics::ApplyDamage(
   HitCharacter,
   Damage,
   Character->Controller,
   Character->GetEquippedWeapon(),
   UDamageType::StaticClass()
  );
 }
}
```

### ServerSideRewind

```cpp
FServerSideRewindResult ULagCompensationComponent::ServerSideRewind(ABlasterCharacter* HitCharacter, const FVector_NetQuantize& TraceStart, const FVector_NetQuantize& HitLocation, float HitTime)
{
    //通过传入的击中角色和击中时间调用获得要检查是否击中的帧数据包。
 FFramePackage FrameToCheck = GetFrameToCheck(HitCharacter, HitTime);
 //调用确认是否击中并返回结果。
 return ConfirmHit(FrameToCheck, HitCharacter, TraceStart, HitLocation);
}
```

### GetFrameToCheck

```cpp
FFramePackage ULagCompensationComponent::GetFrameToCheck(ABlasterCharacter* HitCharacter, float HitTime)
{
    //是否直接返回的判断是依据击中角色为空，击中角色的延迟补偿组件为空，击中角色的帧历史数据包链表头为空，链表尾为空。
 bool bReturn =
  HitCharacter == nullptr ||
  HitCharacter->GetLagCompensation() == nullptr ||
  HitCharacter->GetLagCompensation()->FrameHistory.GetHead() == nullptr ||
  HitCharacter->GetLagCompensation()->FrameHistory.GetTail() == nullptr;
  //如果为true，直接返回。
 if (bReturn) return FFramePackage();
 // 使用帧包对命中进行验证。
 FFramePackage FrameToCheck;
 //设置应该使用插值。
 bool bShouldInterpolate = true;
 // 得到命中角色的历史帧包。
 const TDoubleLinkedList<FFramePackage>& History = HitCharacter->GetLagCompensation()->FrameHistory;
 //得到帧历史的最晚时间和最新时间。
 const float OldestHistoryTime = History.GetTail()->GetValue().Time;
 const float NewestHistoryTime = History.GetHead()->GetValue().Time;
 //如果最晚时间大于击中时间。
 if (OldestHistoryTime > HitTime)
 {
  // 延迟太大了，没必要倒带了。已经超了。
  return FFramePackage();
 }
 //如果最晚帧历史等于击中时间。
 if (OldestHistoryTime == HitTime)
 {
    //说明已经找到需要检查的帧历史数据包，就是帧历史的表尾节点。设置不需要使用插值。
  FrameToCheck = History.GetTail()->GetValue();
  bShouldInterpolate = false;
 }
 //如果最新时间小于等于命中时间。
 if (NewestHistoryTime <= HitTime)
 {
    //说明找到了需要检查的帧历史数据包就是帧历史的表头节点。
  FrameToCheck = History.GetHead()->GetValue();
  bShouldInterpolate = false;
 }

//设置最新指针为表头。最晚指针等于最新指针。
 TDoubleLinkedList<FFramePackage>::TDoubleLinkedListNode* Younger = History.GetHead();
 TDoubleLinkedList<FFramePackage>::TDoubleLinkedListNode* Older = Younger;
 while (Older->GetValue().Time > HitTime) // 最晚时间还是比命中时间早吗。
 {
  // 循环计算出临近的最晚和最早时间，使得命中时间正好在它俩之间准备插值。
  if (Older->GetNextNode() == nullptr) break;
  Older = Older->GetNextNode();
  if (Older->GetValue().Time > HitTime) Younger = Older;
 }
 //跳出循环就两种情况，一种是击中时间已经被夹在最晚时间和最早时间之间，一种是最晚时间等于击中时间。
 if (Older->GetValue().Time == HitTime) // 几乎不可能，但却找到了倒带的时间。
 {
    //需要检查的帧数据包找到。获得帧数据包的值。设置不再需要插值。
  FrameToCheck = Older->GetValue();
  bShouldInterpolate = false;
 }
 //如果需要插值，代表前面的步骤找到的击中时间在最晚时间和最早时间之间。
 if (bShouldInterpolate)
 {
  // 利用算出的最早和最晚时间给命中时间进行插值计算
  FrameToCheck = InterpBetweenFrames(Older->GetValue(), Younger->GetValue(), HitTime);
 }
 //计算出的插值帧数据包的角色设置为当前击中角色。并返回。
 FrameToCheck.Character = HitCharacter;
 return FrameToCheck;
}
```

### InterpBetweenFrames

```cpp
FFramePackage ULagCompensationComponent::InterpBetweenFrames(const FFramePackage& OlderFrame, const FFramePackage& YoungerFrame, float HitTime)
{
    //计算最新和最晚的帧的时间间距。击中时间到最晚时间在这段间距的占比。
 const float Distance = YoungerFrame.Time - OlderFrame.Time;
 const float InterpFraction = FMath::Clamp((HitTime - OlderFrame.Time) / Distance, 0.f, 1.f);

//创建插值的帧数据包。帧包的时间为击中时间。
 FFramePackage InterpFramePackage;
 InterpFramePackage.Time = HitTime;

//循环这个角色的最新帧保存的碰撞盒信息TMap。
 for (auto& YoungerPair : YoungerFrame.HitBoxInfo)
 {
    //创建当前碰撞盒的名字等于循环的Key。
  const FName& BoxInfoName = YoungerPair.Key;

//当前碰撞盒的最晚存储信息等于循环的最晚值。最新存储信息等于最新值。
  const FBoxInformation& OlderBox = OlderFrame.HitBoxInfo[BoxInfoName];
  const FBoxInformation& YoungerBox = YoungerFrame.HitBoxInfo[BoxInfoName];

//创建当前碰撞盒信息存储插值的结果。
  FBoxInformation InterpBoxInfo;

//对碰撞盒的位置盒旋转根据击中时间在最早和最晚时间的占比进行插值。
  InterpBoxInfo.Location = FMath::VInterpTo(OlderBox.Location, YoungerBox.Location, 1.f, InterpFraction);
  InterpBoxInfo.Rotation = FMath::RInterpTo(OlderBox.Rotation, YoungerBox.Rotation, 1.f, InterpFraction);
  //当前插值的碰撞盒范围直接等于最早碰撞盒的范围。
  InterpBoxInfo.BoxExtent = YoungerBox.BoxExtent;

//把插值完毕的碰撞盒信息按照Key碰撞盒名字，Value碰撞盒信息添加到插值的帧数据包的碰撞盒信息的TMap中。
  InterpFramePackage.HitBoxInfo.Add(BoxInfoName, InterpBoxInfo);
 }

//插值的帧数据包的击中时间和碰撞盒信息计算完毕，返回插值的结果插值帧数据包。
 return InterpFramePackage;
}
```

### ConfirmHit

```cpp
FServerSideRewindResult ULagCompensationComponent::ConfirmHit(const FFramePackage& Package, ABlasterCharacter* HitCharacter, const FVector_NetQuantize& TraceStart, const FVector_NetQuantize& HitLocation)
{
    //如果击中角色为空直接返回空结果。
 if (HitCharacter == nullptr) return FServerSideRewindResult();

//创建当前帧数据包。
 FFramePackage CurrentFrame;
 //缓存当前击中角色的碰撞盒帧数据包。用于恢复。
 CacheBoxPositions(HitCharacter, CurrentFrame);
 //通过传入要检查的帧数据包的碰撞盒信息移动击中角色的碰撞盒到要检测的时间角色碰撞盒所处的位置中。
 MoveBoxes(HitCharacter, Package);
 //禁用角色网格的碰撞。只通过碰撞盒进行判断。
 EnableCharacterMeshCollision(HitCharacter, ECollisionEnabled::NoCollision);

 // 首先把头的命中框设置碰撞
 UBoxComponent* HeadBox = HitCharacter->HitCollisionBoxes[FName("head")];
 HeadBox->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
 HeadBox->SetCollisionResponseToChannel(ECC_HitBox, ECollisionResponse::ECR_Block);

//创建击中结果存储确认命中结果。
 FHitResult ConfirmHitResult;
 //射线检测的结束位置等于射线开始位置加上命中位置减去开始位置的延伸确保射线检测的长度足够。
 const FVector TraceEnd = TraceStart + (HitLocation - TraceStart) * 1.25f;
 UWorld* World = GetWorld();
 if (World)
 {
    //使用ECC_HitBox通道进行碰撞检测。
  World->LineTraceSingleByChannel(
   ConfirmHitResult,
   TraceStart,
   TraceEnd,
   ECC_HitBox
  );
  if (ConfirmHitResult.bBlockingHit) // 击中说明命中了头部，不需要进行其余的检测了，因为只是一发子弹，所以提前返回。
  {
    //重置碰撞盒的位置。
   ResetHitBoxes(HitCharacter, CurrentFrame);
   //开启角色网格碰撞。
   EnableCharacterMeshCollision(HitCharacter, ECollisionEnabled::QueryAndPhysics);
   //返回结果为命中头部。
   return FServerSideRewindResult{ true, true };
  }
  else // 没击中头，检查其他命中框
  {
    //循环开启击中角色的碰撞盒碰撞。
   for (auto& HitBoxPair : HitCharacter->HitCollisionBoxes)
   {
    //如果碰撞盒不为空。
    if (HitBoxPair.Value != nullptr)
    {
        //开启碰撞盒的碰撞。设置对于ECC_HitBox通道的碰撞为阻挡。
     HitBoxPair.Value->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
     HitBoxPair.Value->SetCollisionResponseToChannel(ECC_HitBox, ECollisionResponse::ECR_Block);
    }
   }
   //使用ECC_HitBox通道进行碰撞检测。
   World->LineTraceSingleByChannel(
    ConfirmHitResult,
    TraceStart,
    TraceEnd,
    ECC_HitBox
   );
   //如果击中角色。
   if (ConfirmHitResult.bBlockingHit)
   {
    //重置角色碰撞盒的位置。
    ResetHitBoxes(HitCharacter, CurrentFrame);
    //开启角色的网格碰撞。
    EnableCharacterMeshCollision(HitCharacter, ECollisionEnabled::QueryAndPhysics);
    //返回击中身体。
    return FServerSideRewindResult{ true, false };
   }
  }
 }

//剩下的情况就是没有命中，重置角色碰撞盒的位置，开启角色网格的碰撞，返回未击中。
 ResetHitBoxes(HitCharacter, CurrentFrame);
 EnableCharacterMeshCollision(HitCharacter, ECollisionEnabled::QueryAndPhysics);
 return FServerSideRewindResult{ false, false };
}
```

### CacheBoxPositions

```cpp
void ULagCompensationComponent::CacheBoxPositions(ABlasterCharacter* HitCharacter, FFramePackage& OutFramePackage)
{
    //如果角色为空直接返回。
 if (HitCharacter == nullptr) return;
 //循环击中角色的碰撞盒信息。
 for (auto& HitBoxPair : HitCharacter->HitCollisionBoxes)
 {
    //如果碰撞盒不为空。
  if (HitBoxPair.Value != nullptr)
  {
    //创建碰撞盒信息结构体。
   FBoxInformation BoxInfo;
   //存储击中角色的碰撞盒信息位置，旋转，碰撞盒范围。
   BoxInfo.Location = HitBoxPair.Value->GetComponentLocation();
   BoxInfo.Rotation = HitBoxPair.Value->GetComponentRotation();
   BoxInfo.BoxExtent = HitBoxPair.Value->GetScaledBoxExtent();
   //添加到输出的帧数据包的碰撞盒信息的TMap中。
   OutFramePackage.HitBoxInfo.Add(HitBoxPair.Key, BoxInfo);
  }
 }
}
```

### MoveBoxes

```cpp
void ULagCompensationComponent::MoveBoxes(ABlasterCharacter* HitCharacter, const FFramePackage& Package)
{
    //如果角色为空直接返回。
 if (HitCharacter == nullptr) return;
 //循环角色的碰撞盒信息。
 for (auto& HitBoxPair : HitCharacter->HitCollisionBoxes)
 {
    //如果碰撞盒信息不为空。
  if (HitBoxPair.Value != nullptr)
  {
    //检查并存储命中那帧的帧数据包中对应碰撞盒名字的碰撞盒的信息。
   const FBoxInformation* BoxValue = Package.HitBoxInfo.Find(HitBoxPair.Key);
   if (BoxValue)
   {
    //把角色的碰撞盒位置，旋转，碰撞盒全部设置为要检查命中那帧的帧数据包碰撞盒信息。
    HitBoxPair.Value->SetWorldLocation(BoxValue->Location);
    HitBoxPair.Value->SetWorldRotation(BoxValue->Rotation);
    HitBoxPair.Value->SetBoxExtent(BoxValue->BoxExtent);
   }
  }
 }
}
```

### ResetHitBoxes

```cpp
void ULagCompensationComponent::ResetHitBoxes(ABlasterCharacter* HitCharacter, const FFramePackage& Package)
{
 if (HitCharacter == nullptr) return;
 //循环命中角色的碰撞盒信息。
 for (auto& HitBoxPair : HitCharacter->HitCollisionBoxes)
 {
  if (HitBoxPair.Value != nullptr)
  {
    //检查并存储缓存起来的当前帧数据包中对应名字的碰撞盒信息。
   const FBoxInformation* BoxValue = Package.HitBoxInfo.Find(HitBoxPair.Key);
   if (BoxValue)
   {
    //把角色对应的碰撞盒的位置，旋转，碰撞盒范围全部设置回移动前的当前帧信息。
    HitBoxPair.Value->SetWorldLocation(BoxValue->Location);
    HitBoxPair.Value->SetWorldRotation(BoxValue->Rotation);
    HitBoxPair.Value->SetBoxExtent(BoxValue->BoxExtent);
    //禁用碰撞盒的碰撞。
    HitBoxPair.Value->SetCollisionEnabled(ECollisionEnabled::NoCollision);
   }
  }
 }
}
```

### ShotgunServerScoreRequest

```cpp
void ULagCompensationComponent::ShotgunServerScoreRequest_Implementation(const TArray<ABlasterCharacter*>& HitCharacters, const FVector_NetQuantize& TraceStart, const TArray<FVector_NetQuantize>& HitLocations, float HitTime)
{
  //调用霰弹枪服务器倒带确认击中结果。
 FShotgunServerSideRewindResult Confirm = ShotgunServerSideRewind(HitCharacters, TraceStart, HitLocations, HitTime);

//循环击中的角色数组。
 for (auto& HitCharacter : HitCharacters)
 {
  //如果击中的角色为空或攻击的角色没有装备武器，跳过这次循环。
  if (HitCharacter == nullptr || Character == nullptr || Character->GetEquippedWeapon() == nullptr) continue;
  //创建总伤害设置为0。
  float TotalDamage = 0.f;
  //如果确认击中的结果中被爆头的角色有当前击中的角色。
  if (Confirm.HeadShots.Contains(HitCharacter))
  {
    //定义爆头伤害等于确认命中结果中对应当前击中角色的爆头次数乘以攻击角色装备的武器的爆头伤害。
   float HeadShotDamage = Confirm.HeadShots[HitCharacter] * Character->GetEquippedWeapon()->GetHeadShotDamage();
   //总伤害等于加上爆头伤害。
   TotalDamage += HeadShotDamage;
  }
  //如果确认击中的结果中被击中身体的角色有当前击中的角色。
  if (Confirm.BodyShots.Contains(HitCharacter))
  {
    //定义身体伤害等于确认命中结果中对应当前击中角色的击中身体次数乘以攻击角色装备的武器的基础伤害。
   float BodyShotDamage = Confirm.BodyShots[HitCharacter] * Character->GetEquippedWeapon()->GetDamage();
   //总伤害等于再加上身体伤害。
   TotalDamage += BodyShotDamage;
  }
  //施加伤害。
  UGameplayStatics::ApplyDamage(
   HitCharacter,
   TotalDamage,
   Character->Controller,
   Character->GetEquippedWeapon(),
   UDamageType::StaticClass()
  );
 }
}
```

### ShotgunServerSideRewind

```cpp
FShotgunServerSideRewindResult ULagCompensationComponent::ShotgunServerSideRewind(const TArray<ABlasterCharacter*>& HitCharacters, const FVector_NetQuantize& TraceStart, const TArray<FVector_NetQuantize>& HitLocations, float HitTime)
{
  //因为霰弹枪的弹丸是一次发射多发弹丸，所以不一定只击中一个角色。那么需要移动的角色就不会是一个人，而可能是多个人。所以要检查的帧数据包是个数组。
 TArray<FFramePackage> FrameToCheck;
 //循环击中的角色数组。
 for (ABlasterCharacter* HitCharacter : HitCharacters)
 {
  //通过击中的角色和击中的时间获得要检查的帧数据包并添加到检查帧数据包数组中。
  FrameToCheck.Add(GetFrameToCheck(HitCharacter, HitTime));
 }
 //调用霰弹枪确认命中方法判断要检查的帧数据包的命中结果并返回。
 return ShotgunConfirmHit(FrameToCheck, TraceStart, HitLocations);
}
```

### ShotgunConfirmHit

```cpp
FShotgunServerSideRewindResult ULagCompensationComponent::ShotgunConfirmHit(const TArray<FFramePackage>& FramePackages, const FVector_NetQuantize& TraceStart, const TArray<FVector_NetQuantize>& HitLocations)
{
  //我们需要确保帧包中的角色没有空指针，所以循环对当中的角色进行判断，如果有空指针直接返回空的霰弹枪服务器倒带结果。
 for (auto& Frame : FramePackages)
 {
  if (Frame.Character == nullptr) return FShotgunServerSideRewindResult();
 }

//创建霰弹枪服务器倒带结果。和当前帧数据包数组。
 FShotgunServerSideRewindResult ShotgunResult;
 TArray<FFramePackage> CurrentFrames;
 //循环要检查的帧数据包数组。
 for (auto& Frame : FramePackages)
 {
  //创建当前帧数据包。
  FFramePackage CurrentFrame;
  //当前帧的角色等于循环的帧角色。
  CurrentFrame.Character = Frame.Character;
  //缓存循环的帧角色当前帧的信息。用于检查击中之后重置回原来的位置。
  CacheBoxPositions(Frame.Character, CurrentFrame);
  //移动循环帧角色的碰撞盒到要检查的帧包中的位置。
  MoveBoxes(Frame.Character, Frame);
  //禁用角色网格的碰撞。
  EnableCharacterMeshCollision(Frame.Character, ECollisionEnabled::NoCollision);
  //把缓存的当前帧添加当前帧数组中。
  CurrentFrames.Add(CurrentFrame);
 }

//循环要检查的帧数据包。
 for (auto& Frame : FramePackages)
 {
  // 首先把头的命中框设置碰撞
  UBoxComponent* HeadBox = Frame.Character->HitCollisionBoxes[FName("head")];
  HeadBox->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
  HeadBox->SetCollisionResponseToChannel(ECC_HitBox, ECollisionResponse::ECR_Block);
 }

 UWorld* World = GetWorld();
 //循环命中位置数组。也就是传入的命中目标数组。
 for (auto& HitLocation : HitLocations)
 {
  //创建命中检测结果。
  FHitResult ConfirmHitResult;
  //设置射线的结束位置为射线开始加上命中位置减去射线开始位置的延伸。
  const FVector TraceEnd = TraceStart + (HitLocation - TraceStart) * 1.25f;
  if (World)
  {
    //进行ECC_HitBox通道的射线检测。
   World->LineTraceSingleByChannel(
    ConfirmHitResult,
    TraceStart,
    TraceEnd,
    ECC_HitBox
   );
   //从命中结果中的命中角色获得命中的玩家角色。
   ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(ConfirmHitResult.GetActor());
   if (BlasterCharacter)
   {
    //如果霰弹枪命中结果中的爆头TMap中有这个角色。
    if (ShotgunResult.HeadShots.Contains(BlasterCharacter))
    {
      //这个角色的爆头命中次数+1。
     ShotgunResult.HeadShots[BlasterCharacter]++;
    }
    else
    {
      //没有就在爆头TMap中添加这个角色，设置值为1。
     ShotgunResult.HeadShots.Emplace(BlasterCharacter, 1);
    }
   }
  }
 }

 //循环开启其他命中框碰撞，但要禁用头，以免造成二次伤害判断
 for (auto& Frame : FramePackages)
 {
  //循环帧包中角色的碰撞盒信息。
  for (auto& HitBoxPair : Frame.Character->HitCollisionBoxes)
  {
    //如果碰撞盒不为空。
   if (HitBoxPair.Value != nullptr)
   {
    //开启碰撞设置对于ECC_HitBox通道的碰撞为阻挡。
    HitBoxPair.Value->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
    HitBoxPair.Value->SetCollisionResponseToChannel(ECC_HitBox, ECollisionResponse::ECR_Block);
   }
  }
  //通过名字获得角色的头部碰撞盒。禁用其碰撞。
  UBoxComponent* HeadBox = Frame.Character->HitCollisionBoxes[FName("head")];
  HeadBox->SetCollisionEnabled(ECollisionEnabled::NoCollision);
 }

 //循环检查其他命中框命中。
 for (auto& HitLocation : HitLocations)
 {
  //创建命中检测结果。
  FHitResult ConfirmHitResult;
  const FVector TraceEnd = TraceStart + (HitLocation - TraceStart) * 1.25f;
  if (World)
  {
    //调用射线检测碰撞。
   World->LineTraceSingleByChannel(
    ConfirmHitResult,
    TraceStart,
    TraceEnd,
    ECC_HitBox
   );
   //获得命中的角色。
   ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(ConfirmHitResult.GetActor());
   if (BlasterCharacter)
   {
    //如果身体命中TMap中有这个角色。
    if (ShotgunResult.BodyShots.Contains(BlasterCharacter))
    {
      //就把这个角色的值+1。
     ShotgunResult.BodyShots[BlasterCharacter]++;
    }
    else
    {
      //没有就天机这个角色的键值对。
     ShotgunResult.BodyShots.Emplace(BlasterCharacter, 1);
    }
   }
  }
 }

//循环缓存的当前帧数据包。
 for (auto& Frame : CurrentFrames)
 {
  //重置命中角色的碰撞盒。开启角色的网格碰撞。
  ResetHitBoxes(Frame.Character, Frame);
  EnableCharacterMeshCollision(Frame.Character, ECollisionEnabled::QueryAndPhysics);
 }
 //返回霰弹枪的命中结果。
 return ShotgunResult;
}
```

### ProjectileServerScoreRequest

```cpp
//射弹服务器分数请求。
void ULagCompensationComponent::ProjectileServerScoreRequest_Implementation(ABlasterCharacter* HitCharacter, const FVector_NetQuantize& TraceStart, const FVector_NetQuantize100& InitialVelocity, float HitTime)
{
  //调用射弹服务器倒带确认命中结果。
 FServerSideRewindResult Confirm = ProjectileServerSideRewind(HitCharacter, TraceStart, InitialVelocity, HitTime);

//如果角色使用武器击中命中角色。
 if (Character && HitCharacter && Confirm.bHitConfirmed && Character->GetEquippedWeapon())
 {
  //伤害根据是否爆头使用爆头伤害，否则就是基础伤害。
  const float Damage = Confirm.bHeadShot ? Character->GetEquippedWeapon()->GetHeadShotDamage() : Character->GetEquippedWeapon()->GetDamage();

//施加伤害。
  UGameplayStatics::ApplyDamage(
   HitCharacter,
   Damage,
   Character->Controller,
   Character->GetEquippedWeapon(),
   UDamageType::StaticClass()
  );
 }
}
```

### ProjectileServerSideRewind

```cpp
FServerSideRewindResult ULagCompensationComponent::ProjectileServerSideRewind(ABlasterCharacter* HitCharacter, const FVector_NetQuantize& TraceStart, const FVector_NetQuantize100& InitialVelocity, float HitTime)
{
  //通过击中角色和击中时间获得要检查的帧数据包。
 FFramePackage FrameToCheck = GetFrameToCheck(HitCharacter, HitTime);
 //调用射弹命中检查并返回命中结果。
 return ProjectileConfirmHit(FrameToCheck, HitCharacter, TraceStart, InitialVelocity, HitTime);
}
```

### ProjectileConfirmHit

```cpp
//射弹命中检测。射弹武器的命中检测和射线武器的检测有所不同。因为射线武器都是通过射线检测，而射弹武器需要去通过发射射弹来进行检测。所以射线武器是传入击中位置来进行射线检测，而射弹武器则是传入射弹的初始速度来进行模拟检测，因为射弹从发射到击中目标是需要时间的。
FServerSideRewindResult ULagCompensationComponent::ProjectileConfirmHit(const FFramePackage& Package, ABlasterCharacter* HitCharacter, const FVector_NetQuantize& TraceStart, const FVector_NetQuantize100& InitialVelocity, float HitTime)
{
  //创建当前帧数据包。
 FFramePackage CurrentFrame;
 //缓存击中角色的当前帧数据。
 CacheBoxPositions(HitCharacter, CurrentFrame);
 //移动命中角色的碰撞盒到检查位置。
 MoveBoxes(HitCharacter, Package);
 //禁用角色的网格碰撞。
 EnableCharacterMeshCollision(HitCharacter, ECollisionEnabled::NoCollision);

 // 首先把头的命中框设置碰撞
 UBoxComponent* HeadBox = HitCharacter->HitCollisionBoxes[FName("head")];
 HeadBox->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
 HeadBox->SetCollisionResponseToChannel(ECC_HitBox, ECollisionResponse::ECR_Block);

//创建检测抛射物路径的结构体。
 FPredictProjectilePathParams PathParams;
 //开启射线检测。
 PathParams.bTraceWithCollision = true;
 //设置模拟的最大时间为设置的时间。
 PathParams.MaxSimTime = MaxRecordTime;
 //设置抛射物的初速度为初始速度。
 PathParams.LaunchVelocity = InitialVelocity;
 //设置开始位置为射线开始位置。
 PathParams.StartLocation = TraceStart;
 //设置模拟频率为15.f。
 PathParams.SimFrequency = 15.f;
 //设置抛射物的半径为5.f。
 PathParams.ProjectileRadius = 5.f;
 //设置检测通道为ECC_HitBox。
 PathParams.TraceChannel = ECC_HitBox;
 //设置抛射物检测忽略所有者，就是防止命中自己。
 PathParams.ActorsToIgnore.Add(GetOwner());

//创建抛射物检测的结果。
 FPredictProjectilePathResult PathResult;
 //调用抛射物检测传入抛射物和检测结果。
 UGameplayStatics::PredictProjectilePath(this, PathParams, PathResult);

 if (PathResult.HitResult.bBlockingHit) // 击中头,更早返回
 {
  //重置击中角色的碰撞盒。
  ResetHitBoxes(HitCharacter, CurrentFrame);
  //开启击中角色网格的碰撞。
  EnableCharacterMeshCollision(HitCharacter, ECollisionEnabled::QueryAndPhysics);
  //返回击中结果为击中头部。
  return FServerSideRewindResult{ true, true };
 }
 else//没有击中头部，检查其他命中框
 {
  //循环角色其他碰撞盒。
  for (auto& HitBoxPair : HitCharacter->HitCollisionBoxes)
  {
   if (HitBoxPair.Value != nullptr)
   {
    //开启其他碰撞盒的碰撞，对于ECC_HitBox通道的碰撞为阻挡。
    HitBoxPair.Value->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
    HitBoxPair.Value->SetCollisionResponseToChannel(ECC_HitBox, ECollisionResponse::ECR_Block);
   }
  }
  //再次调用抛射物检测。
  UGameplayStatics::PredictProjectilePath(this, PathParams, PathResult);
  //如果命中结果中确认命中。
  if (PathResult.HitResult.bBlockingHit)
  {
    //重置击中角色的碰撞盒。
   ResetHitBoxes(HitCharacter, CurrentFrame);
   //开启角色的网格碰撞。
   EnableCharacterMeshCollision(HitCharacter, ECollisionEnabled::QueryAndPhysics);
   //返回击中角色的身体。
   return FServerSideRewindResult{ true, false };
  }
 }

//否则就是没有命中角色。开启角色的网格碰撞。
 ResetHitBoxes(HitCharacter, CurrentFrame);
 EnableCharacterMeshCollision(HitCharacter, ECollisionEnabled::QueryAndPhysics);
 //返回没有命中角色。
 return FServerSideRewindResult{ false, false };
}
```

### EnableCharacterMeshCollision

```cpp
//设置角色网格的碰撞。
void ULagCompensationComponent::EnableCharacterMeshCollision(ABlasterCharacter* HitCharacter, ECollisionEnabled::Type CollisionEnabled)
{
  //如果击中角色和网格存在。
 if (HitCharacter && HitCharacter->GetMesh())
 {
  //设置击中角色的网格碰撞为传入的碰撞类型。
  HitCharacter->GetMesh()->SetCollisionEnabled(CollisionEnabled);
 }
}
```

### ShowFramePackage

```cpp
//显示角色的帧数据包，用于调式时使用。
void ULagCompensationComponent::ShowFramePackage(const FFramePackage& Package, const FColor& Color)
{
  //循环帧数据包中的角色碰撞盒。
 for (auto& BoxInfo : Package.HitBoxInfo)
 {
  //绘制DebugBox。使用碰撞盒的位置，范围，旋转。设置颜色，只绘制4s中就消失。
  DrawDebugBox(
   GetWorld(),
   BoxInfo.Value.Location,
   BoxInfo.Value.BoxExtent,
   FQuat(BoxInfo.Value.Rotation),
   Color,
   false,
   4.f
  );
 }
}
```
