# TeamPlayerStart

团队模式和夺旗模式角色根据团队生成的地点。

## TeamPlayerStart.h

```cpp
//只有一个团队枚举用来设置复活点属于那个队伍，当角色复活时就会根据这个枚举随机挑选属于自己队伍的复活点。
UCLASS()
class BLASTERLEARING_API ATeamPlayerStart : public APlayerStart
{
 GENERATED_BODY()
 
public:
 UPROPERTY(EditAnywhere)
 ETeam Team;
};
```
