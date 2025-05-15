# BlasterTypes

## Announcement

```cpp
//游戏结束时根据比分情况显示这些文本。
namespace Announcement
{
 const FString NewMatchStartsIn(TEXT("New match starts in:"));
 const FString ThereIsNoWinner(TEXT("There is no winner."));
 const FString YouAreTheWinner(TEXT("You are the winner!"));
 const FString PlayersTiedForTheWin(TEXT("Players tied for the win:"));
 const FString TeamsTiedForTheWin(TEXT("Teams tied for the win:"));
 const FString RedTeam(TEXT("Red team"));
 const FString BlueTeam(TEXT("Blue team"));
 const FString RedTeamWins(TEXT("Red team wins!"));
 const FString BlueTeamWins(TEXT("Blue team wins!"));
}
```

## CombatState

```cpp
UENUM()
//角色的战斗状态，用于判断角色现在处于什么状态执行什么逻辑。
enum class ECombatState : uint8
{
 ECS_Unoccuiped UMETA(DsiplayName = "Unoccpied"),
 ECS_Reloading UMETA(DisplayName = "Reloading"),
 ECS_ThrowingGrenade UMETA(DisplayName = "Throwing Grenade"),
 ECS_SwappingWeapon UMETA(DisplayName = "Swapping Weapon"),

 ECS_MAX UMETA(DsiplayName = "DefaultMax")
};
```

## Team

```cpp
//夺旗和团队模式两个队伍红蓝。
UENUM(BlueprintType)
enum class ETeam : uint8
{
 ET_RedTeam UMETA(DsiplayName = "RedTeam"),
 ET_BlueTeam UMETA(DsiplayName = "BlueTeam"),
 ET_NoTeam UMETA(DsiplayName = "NoTeam"),

 ET_MAX UMETA(DsiplayName = "DefaultMAX")
};
```

##

```cpp
UENUM()
//根据转向的值角色进行转向。
enum class ETurningInPlace : uint8
{
 ETIP_Left UMETA(DisplayName = "Turning Left"),
 ETIP_Right UMETA(DisplayName = "Turning Right"),
 ETIP_NotTurning UMETA(DsiplayName = "Not Turning"),

 ETIP_MAX UMETA(DisplayName = "DefaultMAX")
};
```
