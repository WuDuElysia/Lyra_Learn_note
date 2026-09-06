# Lyra GA 与 GE：创建、应用和 NotifyAbility 链路

## 一句话结论

`GameplayAbility`（GA）负责能力行为和生命周期；`GameplayEffect`（GE）负责属性修改、标签和状态。GA 先以 `FGameplayAbilitySpec` 授予到 ASC，再由输入或代码激活；GE 则由 CDO 生成本次应用的 `FGameplayEffectSpec`，应用后才成为 ASC 中的 `FActiveGameplayEffect`。`NotifyAbilityActivated/Failed/Ended` 是 ASC 接收 GAS 生命周期结果后，补充 Lyra 激活组和失败表现逻辑的回调，不是 GA 的创建函数。

## 一、GA 和 GE 的对象层级

### GameplayAbility

```text
ULyraGameplayAbility 类 / CDO
        ↓ GiveAbility
FGameplayAbilitySpec（保存在 ASC）
        ↓
FGameplayAbilitySpecHandle
        ↓ TryActivateAbility
Ability 实例
        ↓
ActivateAbility / EndAbility / CancelAbility
```

`FGameplayAbilitySpec` 是 ASC 中的授予记录，通常包含：

- Ability 类或 CDO；
- Ability 等级；
- `SourceObject`，例如武器或装备；
- 动态 Spec 源标签，例如 `Input.Ability.Q`；
- 当前激活状态；
- 激活实例和预测信息。

`FGameplayAbilitySpecHandle` 只定位 ASC 中的 AbilitySpec，不是 Ability 实例的指针。

### GameplayEffect

```text
UGameplayEffect 类 / CDO
        ↓ MakeOutgoingSpec
FGameplayEffectSpecHandle
        ↓ Data.Get()
FGameplayEffectSpec（本次应用的运行时配置）
        ↓ ApplyGameplayEffectSpecToSelf/Target
FActiveGameplayEffect（ASC 中的活动效果）
        ↓
FActiveGameplayEffectHandle
```

- `UGameplayEffect` CDO：效果定义模板，不等于已经应用的效果。
- `FGameplayEffectSpec`：本次应用的运行时数据，可以写入等级、Context、SetByCaller 和动态标签。
- `FActiveGameplayEffect`：应用后由 ASC 管理的活动效果记录。
- `FActiveGameplayEffectHandle`：定位活动效果，用于精确移除。

### 四种容易混淆的 Handle

| 类型 | 所在阶段 | 标识对象 | 常见用途 |
| --- | --- | --- | --- |
| `FGameplayAbilitySpecHandle` | GA 已授予 | ASC 中的 AbilitySpec | `TryActivateAbility`、查找、清除 Ability |
| `FGameplayEffectSpecHandle` | GE 尚未应用 | 待应用的 `FGameplayEffectSpec` | 修改 SetByCaller、动态标签等 |
| `FActiveGameplayEffectHandle` | GE 已应用 | ASC 中的 ActiveEffect | `RemoveActiveGameplayEffect` |
| `FGameplayEffectContextHandle` | Spec 的上下文 | 来源、归因、命中信息 | Instigator、Causer、HitResult、SourceObject |

`FGameplayEffectSpecHandle` 不能用来移除已经应用的 GE；只有应用后得到的 `FActiveGameplayEffectHandle` 才承担这个职责。

## 二、GA 的授予与激活

### `ULyraAbilitySet::GiveToAbilitySystem`

源码：`Source/LyraGame/AbilitySystem/LyraAbilitySet.cpp`

核心过程：

```cpp
ULyraGameplayAbility* AbilityCDO =
    AbilityToGrant.Ability->GetDefaultObject<ULyraGameplayAbility>();

FGameplayAbilitySpec AbilitySpec(
    AbilityCDO,
    AbilityToGrant.AbilityLevel
);

AbilitySpec.SourceObject = SourceObject;
AbilitySpec.GetDynamicSpecSourceTags().AddTag(
    AbilityToGrant.InputTag
);

FGameplayAbilitySpecHandle AbilitySpecHandle =
    LyraASC->GiveAbility(AbilitySpec);
```

这一步完成的是：

```text
Ability 类
  → Ability CDO
  → FGameplayAbilitySpec
  → 写入 InputTag / SourceObject
  → GiveAbility
  → 返回 AbilitySpecHandle
```

它不是激活 Ability，也不是创建 Ability 实例。输入映射完成后，Lyra 的 `ProcessAbilityInput` 才会用这个 Handle 找回 Spec，并在 Spec 未激活时调用 `TryActivateAbility`。

GA 的输入链路见：[[Lyra_输入映射_按键到Ability]]。

## 三、GE 的两种创建和应用方式

### 1. 直接应用静态 GE

Lyra 的 AbilitySet 对固定效果采用直接应用：

```cpp
const UGameplayEffect* GameplayEffect =
    EffectToGrant.GameplayEffect
        ->GetDefaultObject<UGameplayEffect>();

const FActiveGameplayEffectHandle GameplayEffectHandle =
    LyraASC->ApplyGameplayEffectToSelf(
        GameplayEffect,
        EffectToGrant.EffectLevel,
        LyraASC->MakeEffectContext()
    );
```

`ApplyGameplayEffectToSelf` 会在 GAS 内部完成 Spec 创建和应用，适合：

- 装备提供的固定属性加成；
- AbilitySet 的常驻效果；
- 不需要在应用前修改数值或标签的 GE。

返回的 `GameplayEffectHandle` 是活动效果句柄，Lyra 会把它保存到 `FLyraAbilitySet_GrantedHandles`，在 AbilitySet 移除时精确回收。

### 2. 先创建 Spec，再修改并应用

当前 Lyra 动态标签代码：

```cpp
const FGameplayEffectSpecHandle SpecHandle =
    MakeOutgoingSpec(
        DynamicTagGE,
        1.0f,
        MakeEffectContext()
    );

FGameplayEffectSpec* Spec = SpecHandle.Data.Get();

if (!Spec)
{
    return;
}

Spec->DynamicGrantedTags.AddTag(Tag);

ApplyGameplayEffectSpecToSelf(*Spec);
```

适合：

- SetByCaller 动态伤害或治疗；
- `DynamicGrantedTags` 动态授予标签；
- `AddDynamicAssetTag` 动态资产标签；
- 根据命中、武器、距离或 Ability 等级写入运行时数据。

## 四、逐行理解动态 Spec

### `MakeOutgoingSpec`

```cpp
MakeOutgoingSpec(DynamicTagGE, 1.0f, MakeEffectContext())
```

参数含义：

| 参数 | 含义 |
| --- | --- |
| `DynamicTagGE` | GameplayEffect 类，提供效果定义模板 |
| `1.0f` | Effect Level，不是持续时间 |
| `MakeEffectContext()` | 创建本次效果的来源和归因上下文 |

它只创建待应用的 `FGameplayEffectSpec`，此时 GE 还没有进入 ASC。

### `MakeEffectContext`

`EffectContext` 不是 GE，也不是 Spec；它是 Spec 中的来源说明。

常见内容：

- `Instigator`：谁施加效果；
- `EffectCauser`：什么对象实际造成效果；
- `SourceObject`：武器、装备等来源；
- `HitResult`：命中位置、骨骼、物理材质；
- Lyra 的 `AbilitySource`；
- Lyra 的 `CartridgeID`。

Lyra 在 `ULyraAbilitySystemGlobals::AllocGameplayEffectContext` 中返回 `FLyraGameplayEffectContext`，因此 Lyra 的 Context 能扩展 GAS 默认上下文。

`ULyraGameplayAbility::MakeEffectContext(Handle, ActorInfo)` 还会补充：

```text
AbilitySource
SourceLevel
SourceObject
Instigator
EffectCauser
```

当前 `AddDynamicTagGameplayEffect` 位于 ASC 中，调用的是 ASC 的 `MakeEffectContext`；在 Ability 创建效果的路径中，则可能进入 `ULyraGameplayAbility` 的重写版本。

### `SpecHandle.Data.Get()`

```cpp
FGameplayEffectSpec* Spec = SpecHandle.Data.Get();
```

它的含义是：

```text
FGameplayEffectSpecHandle
    → Data 内部智能指针
    → Get()
    → FGameplayEffectSpec*
```

`Get()` 只取得一个不转移所有权的原始指针：

- 不会复制 Spec；
- 不需要也不能手动 `delete`；
- 不代表 GE 已经应用；
- 不应该在 `SpecHandle` 生命周期外长期保存。

因此 Lyra 会判空：

```cpp
if (!Spec)
{
    return;
}
```

### 修改 Spec

```cpp
Spec->SetSetByCallerMagnitude(
    LyraGameplayTags::SetByCaller_Damage,
    DamageAmount
);
```

表示把运行时伤害值写入 Spec。GE 资产只规定从哪个 SetByCaller Tag 读取，具体值由本次应用决定。

```cpp
Spec->DynamicGrantedTags.AddTag(Tag);
```

表示本次应用动态授予一个标签。要让标签持续存在，承载它的 `DynamicTagGE` 通常需要配置为 Duration 或 Infinite，而不是依赖 Instant 效果长期提供标签。

### `ApplyGameplayEffectSpecToSelf`

```cpp
ApplyGameplayEffectSpecToSelf(*Spec);
```

`*Spec` 把指针解引用为 `FGameplayEffectSpec&`，然后把 Spec 应用到当前 ASC。

GAS 会根据 GE 定义和 Spec 处理：

- Modifier 与 Attribute 修改；
- Granted Tags；
- Duration、Period；
- Stacking；
- GameplayCue；
- 权限、预测和复制。

应用后，Duration/Infinite 等效果会成为 ASC 中的 `FActiveGameplayEffect`。如果需要精确移除，应保存函数返回的 `FActiveGameplayEffectHandle`。

## 五、Self、Target 和 GE 生命周期

### 应用到自己

```cpp
ASC->ApplyGameplayEffectSpecToSelf(*Spec);
```

效果进入当前 ASC，适合自身加 Buff、扣血、加状态标签或添加被动。

### 应用到目标

可以在目标 ASC 上应用：

```cpp
TargetASC->ApplyGameplayEffectSpecToSelf(*Spec);
```

也可以从源 ASC 指定目标：

```cpp
SourceASC->ApplyGameplayEffectSpecToTarget(*Spec, TargetASC);
```

Context 仍然可以记录攻击者和武器，但 ActiveEffect 最终由目标 ASC 管理。

### Instant、Duration、Infinite、Periodic

| 类型 | 运行时行为 | 典型用途 |
| --- | --- | --- |
| Instant | 立即结算，通常不作为长期 ActiveEffect 保留 | 一次伤害、治疗、扣资源 |
| Duration | 持续指定时间，到期结束 | 加速、燃烧、减伤 |
| Infinite | 一直持续，直到主动移除 | 装备被动、常驻状态、动态标签 |
| Periodic | 按 Period 周期执行，通常与 Duration/Infinite 配合 | 每秒伤害、周期治疗 |

### 移除方式

精确句柄移除：

```cpp
ASC->RemoveActiveGameplayEffect(ActiveHandle);
```

Lyra 的 `FGlobalAppliedEffectList` 和 `FLyraAbilitySet_GrantedHandles` 都保存 `FActiveGameplayEffectHandle`，这样只会移除自己创建的实例。

查询移除：

```cpp
FGameplayEffectQuery Query =
    FGameplayEffectQuery::MakeQuery_MatchAnyOwningTags(
        FGameplayTagContainer(Tag)
    );

Query.EffectDefinition = DynamicTagGE;
RemoveActiveEffects(Query);
```

`RemoveDynamicTagGameplayEffect` 使用这种方式，因为它需要按“动态标签 + DynamicTagGE 类”找到并移除匹配实例，而不是只保存单个句柄。

## 六、NotifyAbility：GAS 生命周期到 Lyra 逻辑的桥

`NotifyAbility...` 是 ASC 收到 GAS 能力生命周期结果后的通知入口。

它们不是：

- Ability 授予函数；
- `ActivateAbility` 本身；
- GE 应用函数；
- 输入 Tag 匹配函数。

它们是 Lyra 在 GAS 原生流程之后插入项目规则的位置。

### `NotifyAbilityActivated`

源码：`Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.cpp`

```cpp
void ULyraAbilitySystemComponent::NotifyAbilityActivated(
    const FGameplayAbilitySpecHandle Handle,
    UGameplayAbility* Ability)
{
    Super::NotifyAbilityActivated(Handle, Ability);

    if (ULyraGameplayAbility* LyraAbility =
        Cast<ULyraGameplayAbility>(Ability))
    {
        AddAbilityToActivationGroup(
            LyraAbility->GetActivationGroup(),
            LyraAbility
        );
    }
}
```

调用顺序：

```text
TryActivateAbility 成功
  → GAS 调用 NotifyAbilityActivated
  → Super::NotifyAbilityActivated
  → Lyra AddAbilityToActivationGroup
```

`Super` 代表 GAS 原生的激活通知和活动状态维护。它不是重新激活 Ability，而是通知 ASC：

> 这次 Ability 已经被 GAS 确认激活了。

Lyra 随后把能力加入激活组，维护 `ActivationGroupCounts`。

### Lyra 激活组

`ELyraAbilityActivationGroup` 有三种有效策略：

| 激活组 | 行为 |
| --- | --- |
| `Independent` | 不阻塞、不取消其他能力 |
| `Exclusive_Replaceable` | 可以被新的独占能力取消和替换 |
| `Exclusive_Blocking` | 阻止其他 Exclusive 能力激活 |

`CanActivateAbility` 会先通过 `IsActivationGroupBlocked` 检查阻塞状态；激活成功后 `NotifyAbilityActivated` 才增加组计数，并可能调用 `CancelActivationGroupAbilities` 取消旧的 Replaceable 能力。

这是一套“独立/可替换/阻塞”的关系系统，不是带数值的优先级系统。Lyra 默认没有“优先级 10 一定打断优先级 5”的通用规则。

### `NotifyAbilityFailed`

源码：`Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.cpp`

```cpp
void ULyraAbilitySystemComponent::NotifyAbilityFailed(
    const FGameplayAbilitySpecHandle Handle,
    UGameplayAbility* Ability,
    const FGameplayTagContainer& FailureReason)
{
    Super::NotifyAbilityFailed(Handle, Ability, FailureReason);

    if (APawn* Avatar = Cast<APawn>(GetAvatarActor()))
    {
        if (!Avatar->IsLocallyControlled() &&
            Ability->IsSupportedForNetworking())
        {
            ClientNotifyAbilityFailed(Ability, FailureReason);
            return;
        }
    }

    HandleAbilityFailed(Ability, FailureReason);
}
```

它表示：

```text
GAS 尝试激活 Ability，但 CanActivateAbility 或其他条件失败
```

失败原因可能包含：

- 冷却未结束；
- 成本不足；
- 缺少 Required Tags；
- 被 Blocked Tags 阻塞；
- Lyra 激活组阻塞；
- Ability 当前不允许激活。

处理顺序：

```text
GAS 判定激活失败
  → Super::NotifyAbilityFailed
  → 远程控制且支持网络：ClientNotifyAbilityFailed
  → 否则：HandleAbilityFailed
  → LyraGameplayAbility::OnAbilityFailedToActivate
```

`NotifyAbilityFailed` 不会重试或强制激活 Ability，它主要把失败原因路由到正确的本地端，用于播放失败动画、提示、音效或 UI。

`ClientNotifyAbilityFailed` 是不可靠 Client RPC；它只同步失败表现，不改变服务器已经做出的激活结果。

### `NotifyAbilityEnded`

源码：`Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.cpp`

```cpp
void ULyraAbilitySystemComponent::NotifyAbilityEnded(
    FGameplayAbilitySpecHandle Handle,
    UGameplayAbility* Ability,
    bool bWasCancelled)
{
    Super::NotifyAbilityEnded(Handle, Ability, bWasCancelled);

    if (ULyraGameplayAbility* LyraAbility =
        Cast<ULyraGameplayAbility>(Ability))
    {
        RemoveAbilityFromActivationGroup(
            LyraAbility->GetActivationGroup(),
            LyraAbility
        );
    }
}
```

调用顺序：

```text
Ability::EndAbility 或 CancelAbility
  → GAS 清理活动状态
  → NotifyAbilityEnded
  → Super::NotifyAbilityEnded
  → Lyra 减少激活组计数
```

它的作用是让已经结束的 Ability 从激活组中退出，否则 `Exclusive_Blocking` 的计数可能一直存在，后续 Ability 会被错误阻塞。

`NotifyAbilityEnded` 不是调用 `EndAbility`；它是 Ability 已经结束后 ASC 收到的通知。

## 七、完整 GA → GE → Notify 链路

```text
AbilitySet::GiveToAbilitySystem
  → 创建 FGameplayAbilitySpec 并 GiveAbility
  → 输入系统通过 Tag 找到 Spec.Handle
  → ProcessAbilityInput
  → TryActivateAbility
      ├─ 失败：NotifyAbilityFailed
      │       → FailureReason → Lyra 失败表现
      └─ 成功：NotifyAbilityActivated
              → Super → AddAbilityToActivationGroup
              → Ability::ActivateAbility
              → MakeOutgoingSpec
              → 修改 SetByCaller / DynamicGrantedTags
              → ApplyGameplayEffectSpecToSelf/Target
              → ASC 创建 ActiveGameplayEffect
              → EndAbility/CancelAbility
              → NotifyAbilityEnded
              → RemoveAbilityFromActivationGroup
```

其中：

- GA 的 Handle 用于定位 AbilitySpec；
- GE 的 SpecHandle 用于应用前修改数据；
- GE 应用后的 ActiveHandle 用于移除效果；
- NotifyAbility 是 ASC 对 GAS 生命周期的项目级桥接；
- InputPressed/InputReleased 和 NotifyAbilityActivated/Ended 属于不同层次：前者是输入事件，后者是 Ability 生命周期通知。

## 八、GAS 原生与 Lyra 自定义

### GAS 原生

- `UGameplayAbility`、`UGameplayEffect`；
- `FGameplayAbilitySpec` 和各类 GAS Handle；
- `UAbilitySystemComponent::GiveAbility`、`TryActivateAbility`；
- `MakeOutgoingSpec`、`ApplyGameplayEffectToSelf`、`ApplyGameplayEffectSpecToSelf`；
- `FGameplayEffectContextHandle`、`FGameplayEffectSpec`、`FActiveGameplayEffect`；
- Ability 的激活、失败、结束通知机制；
- Prediction、复制、AbilityTask 和通用事件。

### Lyra 自定义

- `ULyraAbilitySet::GiveToAbilitySystem` 的批量授予和句柄回收；
- `ULyraAbilitySystemComponent` 的输入缓存和 `ProcessAbilityInput`；
- `FLyraGameplayEffectContext` 与 `AllocGameplayEffectContext`；
- `AddDynamicTagGameplayEffect` / `RemoveDynamicTagGameplayEffect`；
- `NotifyAbilityActivated` 的激活组计数；
- `NotifyAbilityFailed` 的失败表现路由；
- `NotifyAbilityEnded` 的激活组计数回收；
- `Independent`、`Exclusive_Replaceable`、`Exclusive_Blocking`；
- `ULyraGameplayAbility::MakeEffectContext` 对 AbilitySource 和 SourceObject 的补充。

## 九、常见误区

1. `GiveAbility` 是授予 GA，不是激活 GA。
2. `GameplayAbility` 不会直接“应用到 ASC”；ASC 保存的是 AbilitySpec，激活后才运行 Ability。
3. `UGameplayEffect` CDO 是定义模板，不是已经生效的 ActiveEffect。
4. `FGameplayEffectSpecHandle` 不是移除句柄；移除要用 `FActiveGameplayEffectHandle`。
5. `MakeEffectContext` 不是创建 GE，它只创建效果来源/命中上下文。
6. `SpecHandle.Data.Get()` 只取出待应用 Spec 的指针，不会自动应用效果。
7. `NotifyAbilityActivated` 不是 `ActivateAbility`，而是 GAS 确认激活后通知 ASC 的回调。
8. `NotifyAbilityFailed` 不会重新激活 Ability，只负责失败处理和表现路由。
9. `NotifyAbilityEnded` 不等于调用 `EndAbility`，它发生在 Ability 结束流程之后。
10. 输入释放不会自动调用 `EndAbility`；是否结束由 Ability 自己决定。

## 十、源码索引

- `Source/LyraGame/AbilitySystem/LyraAbilitySet.cpp`
  - `ULyraAbilitySet::GiveToAbilitySystem`
  - `FLyraAbilitySet_GrantedHandles::AddAbilitySpecHandle`
  - `FLyraAbilitySet_GrantedHandles::AddGameplayEffectHandle`
- `Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.cpp`
  - `ULyraAbilitySystemComponent::ProcessAbilityInput`
  - `ULyraAbilitySystemComponent::NotifyAbilityActivated`
  - `ULyraAbilitySystemComponent::NotifyAbilityFailed`
  - `ULyraAbilitySystemComponent::NotifyAbilityEnded`
  - `ULyraAbilitySystemComponent::AddAbilityToActivationGroup`
  - `ULyraAbilitySystemComponent::RemoveAbilityFromActivationGroup`
  - `ULyraAbilitySystemComponent::AddDynamicTagGameplayEffect`
  - `ULyraAbilitySystemComponent::RemoveDynamicTagGameplayEffect`
- `Source/LyraGame/AbilitySystem/Abilities/LyraGameplayAbility.cpp`
  - `ULyraGameplayAbility::CanActivateAbility`
  - `ULyraGameplayAbility::MakeEffectContext`
  - `ULyraGameplayAbility::OnAbilityFailedToActivate`
- `Source/LyraGame/AbilitySystem/LyraGameplayEffectContext.h`
  - `FLyraGameplayEffectContext`
- `Source/LyraGame/AbilitySystem/LyraAbilitySystemGlobals.cpp`
  - `ULyraAbilitySystemGlobals::AllocGameplayEffectContext`
- `Source/LyraGame/Player/LyraCheatManager.cpp`
  - `ULyraCheatManager::ApplySetByCallerDamage`
  - `ULyraCheatManager::ApplySetByCallerHeal`
- `Source/LyraGame/Character/LyraHealthComponent.cpp`
  - `ULyraHealthComponent::DamageSelfDestruct`
- `Source/LyraGame/AbilitySystem/LyraGlobalAbilitySystem.cpp`
  - `FGlobalAppliedEffectList::AddToASC`
  - `FGlobalAppliedEffectList::RemoveFromASC`

相关笔记：

- [[Lyra_输入映射_按键到Ability]]
- [[Lyra_AbilityActorInfo_生命周期]]
- `Lyra_GA与GE_创建应用_NotifyAbility.canvas`
