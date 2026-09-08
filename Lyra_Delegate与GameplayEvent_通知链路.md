# Lyra Delegate 与 GAS Gameplay Event：通知链路

## 一句话结论

Lyra 的死亡流程实际上经过两层通知：第一层是 C++ Delegate，`HealthSet` 通过 `OnOutOfHealth.Broadcast(...)` 通知已经注册的 `HealthComponent::HandleOutOfHealth`；第二层是 GAS Gameplay Event，`HandleOutOfHealth` 构造 `FGameplayEventData` 后调用 `ASC->HandleGameplayEvent(...)`，GAS 再根据 Ability 的 `FAbilityTriggerData` 尝试激活死亡 Ability。

```text
C++ Delegate：
HealthSet.Broadcast
  → HealthComponent::HandleOutOfHealth

GAS Gameplay Event：
HandleOutOfHealth
  → ASC.HandleGameplayEvent
  → Death Ability
```

## 一、C++ Delegate 是什么

Delegate 可以理解为一个“可保存函数列表的事件对象”：

```text
声明事件
  → 注册对象和成员函数
  → 事件发生时 Broadcast
  → 调用所有已注册函数
```

### 1. 声明 Delegate 类型

文件：`Source/LyraGame/AbilitySystem/Attributes/LyraAttributeSet.h`

```cpp
DECLARE_MULTICAST_DELEGATE_SixParams(
    FLyraAttributeEvent,
    AActor*,
    AActor*,
    const FGameplayEffectSpec*,
    float,
    float,
    float
);
```

它定义了一个可以广播六个参数的原生多播 Delegate。六个参数对应：

```text
EffectInstigator
EffectCauser
EffectSpec
EffectMagnitude
OldValue
NewValue
```

### 2. 声明具体事件

文件：`Source/LyraGame/AbilitySystem/Attributes/LyraHealthSet.h`

```cpp
mutable FLyraAttributeEvent OnOutOfHealth;
```

这表示 `HealthSet` 内部有一个名为 `OnOutOfHealth` 的事件对象。

它不是普通函数，因此不能这样调用：

```cpp
OnOutOfHealth();
```

它要通过：

```cpp
OnOutOfHealth.Broadcast(...);
```

触发。

### 3. 注册接收函数：`AddUObject`

文件：`Source/LyraGame/Character/LyraHealthComponent.cpp`

函数：`ULyraHealthComponent::InitializeWithAbilitySystem`

```cpp
HealthSet->OnOutOfHealth.AddUObject(
    this,
    &ThisClass::HandleOutOfHealth
);
```

含义是：

```text
把当前 HealthComponent 对象的 HandleOutOfHealth 函数
注册到 HealthSet 的 OnOutOfHealth 事件上。
```

可以想象成 Delegate 内部保存了：

```text
监听列表：
    HealthComponent 对象 → HandleOutOfHealth 函数
```

`this` 是当前的 `ULyraHealthComponent` 对象；`&ThisClass::HandleOutOfHealth` 是该对象的成员函数地址。

### 4. 触发事件：`Broadcast`

文件：`Source/LyraGame/AbilitySystem/Attributes/LyraHealthSet.cpp`

函数：`ULyraHealthSet::PostGameplayEffectExecute`

当生命值跨过 0：

```cpp
if ((GetHealth() <= 0.0f) && !bOutOfHealth)
{
    OnOutOfHealth.Broadcast(
        Instigator,
        Causer,
        &Data.EffectSpec,
        Data.EvaluatedData.Magnitude,
        HealthBeforeAttributeChange,
        GetHealth()
    );
}
```

`Broadcast` 会遍历已经注册的监听者，概念上相当于：

```cpp
HealthComponent->HandleOutOfHealth(
    Instigator,
    Causer,
    &Data.EffectSpec,
    Data.EvaluatedData.Magnitude,
    HealthBeforeAttributeChange,
    GetHealth()
);
```

如果有多个对象通过 `AddUObject` 注册，它们都会收到通知。

### 5. 解除注册

文件：`Source/LyraGame/Character/LyraHealthComponent.cpp`

函数：`ULyraHealthComponent::UninitializeFromAbilitySystem`

```cpp
HealthSet->OnOutOfHealth.RemoveAll(this);
```

表示当前 HealthComponent 不再接收这个 Delegate 的通知，避免对象解绑后继续被调用。

### 6. Delegate 不是网络广播

这里的：

```cpp
AddUObject
Broadcast
```

是同一进程内的 C++ 函数回调：

```text
不是 RPC
不是网络同步
不是 GAS Gameplay Event
```

它只是让一个对象发生事件时调用另一个对象的函数。

## 二、GAS Gameplay Event 是什么

Gameplay Event 是 GAS 使用的另一套事件机制，核心数据是：

```cpp
FGameplayEventData
```

它至少包含：

```text
EventTag
Instigator
Target
OptionalObject
ContextHandle
EventMagnitude
```

可以理解为：

```text
事件名称 + 事件参数
```

### 1. 构造 Gameplay Event

文件：`Source/LyraGame/Character/LyraHealthComponent.cpp`

函数：`ULyraHealthComponent::HandleOutOfHealth`

```cpp
FGameplayEventData Payload;
Payload.EventTag = LyraGameplayTags::GameplayEvent_Death;
Payload.Instigator = DamageInstigator;
Payload.Target = AbilitySystemComponent->GetAvatarActor();
Payload.OptionalObject = DamageEffectSpec->Def;
Payload.ContextHandle = DamageEffectSpec->GetEffectContext();
Payload.EventMagnitude = DamageMagnitude;
```

这里的：

```cpp
Payload.EventTag = GameplayEvent.Death;
```

表示这是一个死亡事件。

### 2. 把事件交给 ASC

```cpp
AbilitySystemComponent->HandleGameplayEvent(
    Payload.EventTag,
    &Payload
);
```

`HandleGameplayEvent` 是 GAS 的事件入口，不是普通 Delegate 的 `Broadcast`。

它会让 GAS 根据事件 Tag 查找当前 ASC 中配置了对应 Gameplay Event Trigger 的 AbilitySpec，并尝试激活匹配的 Ability。

## 三、`FAbilityTriggerData` 与 Gameplay Event 的关系

文件：`Source/LyraGame/AbilitySystem/Abilities/LyraGameplayAbility_Death.cpp`

死亡 Ability 的构造函数中：

```cpp
FAbilityTriggerData TriggerData;

TriggerData.TriggerTag =
    LyraGameplayTags::GameplayEvent_Death;

TriggerData.TriggerSource =
    EGameplayAbilityTriggerSource::GameplayEvent;

AbilityTriggers.Add(TriggerData);
```

`FAbilityTriggerData` 是 Ability 的触发规则记录，核心信息是：

```text
TriggerTag
TriggerSource
```

这段代码不是在此刻激活 Ability，而是在 Ability CDO 上登记规则：

```text
如果收到 GameplayEvent.Death，
就尝试激活这个 Ability。
```

`EGameplayAbilityTriggerSource::GameplayEvent` 表示触发来源是 Gameplay Event；另一类常见来源是 `OwnedTagAdded`，表示 ASC 获得指定 Owned Tag 时触发。

## 四、两层死亡通知完整链路

```text
1. 伤害 GameplayEffect 修改 Health
        ↓
2. LyraHealthSet::PostGameplayEffectExecute
        ↓
3. Health <= 0，OnOutOfHealth.Broadcast(...)
        ↓
4. HealthComponent::HandleOutOfHealth(...)
        ↓
5. 构造 FGameplayEventData
        ↓
6. ASC->HandleGameplayEvent(GameplayEvent.Death, Payload)
        ↓
7. GAS 匹配 Death Ability 的 FAbilityTriggerData
        ↓
8. 尝试激活 Death Ability
        ↓
9. ULyraGameplayAbility_Death::ActivateAbility
```

所以两次“通知”不是同一个系统：

```text
第一层：C++ Delegate
    OnOutOfHealth.Broadcast
    → HandleOutOfHealth

第二层：GAS Gameplay Event
    HandleGameplayEvent
    → Ability Trigger
    → Death Ability
```

## 五、Delegate 和 Gameplay Event 对比

| 项目 | C++ Delegate | GAS Gameplay Event |
| --- | --- | --- |
| 声明 | `DECLARE_MULTICAST_DELEGATE...` | GAS 原生事件接口 |
| 注册 | `AddUObject` | Ability Trigger 或 AbilityTask 等 GAS 机制 |
| 发送 | `Broadcast(...)` | `HandleGameplayEvent(Tag, Payload)` |
| 接收者 | 具体 C++ 成员函数 | 匹配的 Ability 或等待事件的 AbilityTask |
| 数据 | 函数参数 | `FGameplayEventData` |
| 作用 | 对象之间的本地回调 | 触发/通知 Ability |
| 网络含义 | 默认没有网络语义 | 是否同步取决于 GAS 激活、预测和复制流程 |

## 六、`WaitGameplayEvent` 与这两者的关系

如果一个 Ability 已经激活，可以使用 GAS 的：

```text
AbilityTask_WaitGameplayEvent
```

概念流程：

```text
Ability 激活
  → WaitGameplayEvent 注册 EventTag
  → 其他系统调用 HandleGameplayEvent
  → GAS 将 FGameplayEventData 交给等待任务
  → Ability 收到事件回调
```

它和 `OnOutOfHealth.AddUObject` 的共同点是：

```text
都要先注册监听，再等待事件发生
```

但它们不是同一个 Delegate：

```text
AddUObject
    绑定 C++ 对象成员函数

WaitGameplayEvent
    注册 GAS AbilityTask 对 Gameplay Event 的监听
```

## 七、不要混淆 Gameplay Event 与 Gameplay Message

Lyra 的死亡代码中还可能调用：

```cpp
UGameplayMessageSubsystem::BroadcastMessage(...);
```

这是 Lyra 的 Gameplay Message 系统，不是 GAS Gameplay Event。

```text
C++ Delegate：
    OnOutOfHealth.Broadcast

GAS Gameplay Event：
    ASC.HandleGameplayEvent

Lyra Gameplay Message：
    GameplayMessageSubsystem.BroadcastMessage
```

它们都叫“通知”或“消息”，但接收机制不同。

## 八、源码索引

### C++ Delegate

- `Source/LyraGame/AbilitySystem/Attributes/LyraAttributeSet.h`
  - `FLyraAttributeEvent`
- `Source/LyraGame/AbilitySystem/Attributes/LyraHealthSet.h`
  - `OnOutOfHealth`
- `Source/LyraGame/AbilitySystem/Attributes/LyraHealthSet.cpp`
  - `PostGameplayEffectExecute`
  - `OnOutOfHealth.Broadcast`
- `Source/LyraGame/Character/LyraHealthComponent.cpp`
  - `InitializeWithAbilitySystem`
  - `UninitializeFromAbilitySystem`
  - `HandleOutOfHealth`

### GAS Gameplay Event

- `Source/LyraGame/Character/LyraHealthComponent.cpp`
  - `HandleOutOfHealth`
  - `AbilitySystemComponent->HandleGameplayEvent`
- `Source/LyraGame/AbilitySystem/Abilities/LyraGameplayAbility_Death.cpp`
  - 构造函数中的 `FAbilityTriggerData`
  - `ActivateAbility`
- `Source/LyraGame/AbilitySystem/Abilities/LyraGameplayAbility_Death.h`
  - `ULyraGameplayAbility_Death`
- `Content/Characters/Heroes/Abilities/GA_Hero_Death.uasset`
  - 实际死亡 Ability 资产

相关笔记：

- [[Lyra_GA与GE_创建应用_NotifyAbility]]
- [[Lyra_NotifyAbility_三个回调_GAS_Super]]
- `Lyra_Delegate与GameplayEvent_通知链路.canvas`
