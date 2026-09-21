# UE Damage 系统

UE 提供了一套 Damage System，主要数据链路为 —— 发生攻击 -> 生成攻击描述 -> 通知目标 Actor -> Actor 接收攻击描述，gameplay 消费描述

通过一个简单的例子来对这个系统进行基础使用串联 —— 场景：攻击键点击 -> Player 检测 Hit -> 如果命中了敌人 -> 敌人收到 20 点伤害

## 描述攻击

在进行攻击之前，我们需要对一次的攻击进行描述

```C++
USTRUCT(BlueprintType)
struct FSimpleAttackSpec
{
	GENERATED_BODY()
	UPROPERTY(EditAnywhere)
	float Damage = 20.0f;
	
	UPROPERTY(EditAnywhere)
	float Range = 150.0f;
	
	UPROPERTY(EditAnywhere)
	TSubclassOf<UDamageType> DamageType;
};
```

此处仅仅是描述一次“静态”的攻击，只是为了好配置（主打一个数据驱动），并非 Damage System 强制要求

## Attack Click

在 Player 发起攻击的时候，需要先进行攻击检测 —— 在检测规则中击中了什么东西，从普遍的实现来说，就是发起一次碰撞体的检测

比如说 ——

```C++
FVector Start = GetActorLocation();
FVector Direction = GetActorForwardVector();
FVector End = Start + Direction * AttackSpec.Range;
```

设置射线检测范围，然后查询碰撞到的物体 ——

```C++
FHitResult Hit;
FCollisionQueryParams Params;
Params.AddIgnoredActor(this);
bool bHit = GetWorld()->LineTraceSingleByChannel(
	Hit,
	Start,
	End,
	ECC_Pawn,
	Params
);
```

物理系统会计算从出发点到结束点第一个符合 Collision Channel 的物体是什么。如果 `bHit == true` 说明在攻击范围内碰撞到物体了，那么我们可以从 `Hit` 变量中获取所有想要的数据。

## 发送攻击描述

假设过滤清楚了（比如过滤自己，或者其他 Gameplay 上的规则），并且

```C++
AActor* HitActor = Hit.GetActor();
```

真实存在

那么进行调用 ——

```C++
UGameplayStatics::ApplyPointDamage(
	HitActor,
	AttackSpec.Damage,
	Direction,
	Hit,
	GetController(),
	this,
	AttackSpec.DamageType
);
```

很好理解的

- `HitActor` 打到谁了
- `Damage` 造成基础伤害是多少
- `Direction` 从什么方向攻击到敌人
- `Hit` 完整命中的结果（包括什么位置，之类的）
- `GetController` 谁需要对此次攻击”负责“ —— 谁发起 / 操作的这件事情
- `this` 哪个 Actor 实际造成了这次伤害
- `DamageType` 目标受到什么类型的伤害

这里就比较漂亮 —— 我们不是简单的转化 Hit 的 Actor 的类型，然后调用其实现的方法 —— 因为这样会使得攻击方必须知道受击方具体是什么类型，实现了什么受击方法需要调用等，虽然确乎可以“面向接口”来使得耦合关系仅仅是发起攻击方 -> 统一的受击接口；不过 UE 相当于已经写好了一个通用协议提供给我们直接使用

## 消费攻击描述

对于受击 Actor，会调用其 ——

```C++
virtual float TakeDamage(
	float DamageAmount,
	const FDamageEvent& DamageEvent,
	AController* EventInstigator,
	AActor* DamageCauser
);
```

这个方法，所以如果要实现一些特殊的 Gameplay 逻辑，需要自己 override 这个函数

（当然也可以不 override，而是监听 `OnTakeAnyDamage` / `OnTakePointDamage` / `OnTakeRadiaDamage` 事件）

比如 ——

```C++
float ATrainingDummy::TakeDamage(
	float DamageAmount,
	const FDamageEvent& DamageEvent,
	AController* EventInstigator,
	AActor* DamageCauser
) {
	const float ActualDamage = Super::TakeDamage(
	DamageAmount, DamageEvent, EventInstigator, DamageCauser);
	CurrentHealth = FMath::Clamp(CurrentHealth - ActualDamage, 0.0f, MaxHealth); 
	return ActualDamage;
}
```

其中值得注意的是，此处的返回值语义为“最终实际应用的 Damage” —— 即实际造成伤害