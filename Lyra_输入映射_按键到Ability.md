# Lyra 输入映射：按键 → Tag → Ability

## 一句话结论

Lyra 不是让 GAS 直接监听键盘，而是把 `UInputAction` 和 `GameplayTag` 绑定，再用同一个 Tag 匹配 AbilitySpec 的动态源标签；匹配后保存 `FGameplayAbilitySpecHandle`，由 ASC 根据 Handle 找回 Spec，最后决定是激活 Ability，还是向已激活实例发送输入事件。

## 主链路

```text
物理键 Q
  → InputMappingContext：Q → IA_Ability_Q
  → LyraInputConfig：IA_Ability_Q → Input.Ability.Q
  → AbilitySet：GA_Fire → Input.Ability.Q
  → AbilitySpec.DynamicSpecSourceTags = { Input.Ability.Q }
  → AbilityInputTagPressed(Input.Ability.Q)
  → 匹配 Spec.Handle
  → ProcessAbilityInput
     ├─ Spec 未激活：TryActivateAbility
     └─ Spec 已激活：InvokeReplicatedEvent(InputPressed, Handle, PredictionKey)
```

## 三套映射不要混淆

| 层次 | 示例 | 归属 |
| --- | --- | --- |
| 物理键 → `UInputAction` | `Q → IA_Ability_Q` | UE Enhanced Input / 资产配置 |
| `UInputAction` → `GameplayTag` | `IA_Ability_Q → Input.Ability.Q` | Lyra 自定义 `ULyraInputConfig`、`BindAbilityActions` |
| Ability → 输入 Tag | `GA_Fire → Input.Ability.Q` | Lyra 自定义 `ULyraAbilitySet` 授予逻辑 |
| Tag → AbilitySpec | 精确匹配 `DynamicSpecSourceTags` | Lyra 输入处理使用 GAS 的 Spec 数据 |

两边的 Tag 必须一致：

```text
InputConfig：IA_Ability_Q → Input.Ability.Q
AbilitySet： GA_Fire      → Input.Ability.Q
```

当前 Lyra 使用 `HasTagExact`，所以不是父子 GameplayTag 的模糊匹配。

## 按下时如何找到 Ability

`ULyraInputComponent::BindAbilityActions` 将 `Action.InputTag` 作为参数绑定给 Triggered 回调：

```text
Q 按下
  → Input_AbilityInputTagPressed(Input.Ability.Q)
  → ULyraAbilitySystemComponent::AbilityInputTagPressed
```

ASC 会遍历 `ActivatableAbilities.Items`，对每个 AbilitySpec 检查：

```cpp
AbilitySpec.GetDynamicSpecSourceTags().HasTagExact(InputTag)
```

匹配后保存的是 Handle，而不是 Ability 指针：

```text
InputPressedSpecHandles = [SpecHandle]
InputHeldSpecHandles    = [SpecHandle]
```

之后 `ProcessAbilityInput` 使用：

```cpp
FindAbilitySpecFromHandle(SpecHandle)
```

找回 `FGameplayAbilitySpec`，再从 Spec 读取 Ability、激活状态和激活信息。

## 第一次按 Q 与再次按 Q

### 第一次按 Q

```text
Tag 匹配 Spec
  → Spec 未激活
  → 如果 ActivationPolicy == OnInputTriggered
  → TryActivateAbility(Spec.Handle)
```

触发激活的这次 Q 不会在同一帧自动再次发送为 `InputPressed` 事件。

### Ability 已激活时再次按 Q

```text
Tag 匹配同一个 Spec
  → Spec.IsActive() == true
  → AbilitySpecInputPressed(Spec)
  → Super::AbilitySpecInputPressed(Spec)
  → InvokeReplicatedEvent(InputPressed, Spec.Handle, PredictionKey)
  → WaitInputPress（如果存在）收到事件
```

`PredictionKey` 不是单独用来查找 Ability 的。事件实际通过：

```text
EventType + Spec.Handle + Activation PredictionKey
```

定位到这个 AbilitySpec 的这一次激活；`WaitInputPress` 监听的也是同一组信息。

## 释放时如何找到 Ability

释放不是 ASC 主动轮询键盘，而是 Enhanced Input 的 `Completed` 回调把同一个输入 Tag 传回来：

```text
Q 松开
  → Input_AbilityInputTagReleased(Input.Ability.Q)
  → AbilityInputTagReleased(Input.Ability.Q)
```

Lyra ASC 再次遍历当前 AbilitySpec，找出动态源标签精确匹配的 Spec：

```text
InputReleasedSpecHandles.AddUnique(AbilitySpec.Handle)
InputHeldSpecHandles.Remove(AbilitySpec.Handle)
```

随后 `ProcessAbilityInput` 的 release 阶段使用 Handle 找回 Spec：

```text
FindAbilitySpecFromHandle(SpecHandle)
  → Spec.InputPressed = false
  → 如果 Spec 仍激活：AbilitySpecInputReleased(Spec)
  → InvokeReplicatedEvent(InputReleased, Handle, PredictionKey)
  → WaitInputRelease（如果存在）收到事件
```

释放输入不等于结束 Ability。这里没有自动调用 `EndAbility`；是否结束由 Ability 自己决定，例如在 `WaitInputRelease` 回调中显式调用 `EndAbility`。

## GAS 原生与 Lyra 自定义

### GAS 原生

- `FGameplayAbilitySpec`：Ability 的运行时授予规格。
- `FGameplayAbilitySpecHandle`：唯一定位一个 Spec。
- `TryActivateAbility`：尝试激活 Spec。
- Active Ability 实例和 `ActivationPredictionKey`：定位一次具体激活。
- `InvokeReplicatedEvent`：按事件类型、Spec Handle 和 PredictionKey 分发通用 Ability 事件。
- `AbilityTask_WaitInputPress` / `AbilityTask_WaitInputRelease`：监听对应通用输入事件。
- Ability 是否结束，以及 `EndAbility` / `CancelAbility` 的生命周期语义。

### Lyra 自定义

- `ULyraInputConfig`：维护 InputAction 与 InputTag 的配对。
- `ULyraInputComponent::BindAbilityActions`：把 Enhanced Input 的 Triggered/Completed 转成带 Tag 的 Lyra 回调。
- `ULyraAbilitySet::GiveToAbilitySystem`：把 AbilitySet 的 InputTag 写入 Spec 动态源标签。
- `AbilityInputTagPressed/Released`：按 Tag 扫描 Spec，并缓存 Handle。
- `InputPressedSpecHandles`、`InputHeldSpecHandles`、`InputReleasedSpecHandles`：Lyra 的输入缓存。
- `ProcessAbilityInput`：按 held、pressed、激活、released 的顺序组织输入处理。
- `ULyraAbilitySystemComponent::AbilitySpecInputPressed/Released`：调用 GAS 基类后，用 `InvokeReplicatedEvent` 连接 Lyra 输入与 `WaitInputPress/Release`。
- Lyra 的 `ActivationPolicy`，例如 `OnInputTriggered`、`WhileInputActive`。

## 关键边界

1. `WaitInputPress` 不知道是 Q 还是 E，它只知道某个 Spec 收到了 `InputPressed`。
2. 如果 Q Ability 的 Spec 只有 `Input.Ability.Q`，E 的 `Input.Ability.E` 不会唤醒它的 `WaitInputPress`。
3. 同一个 InputTag 可以匹配多个 Spec，因此多个 Ability 可能同时进入输入处理。
4. 如果 Ability 在按键释放前已经结束，release 阶段仍可能把 `Spec.InputPressed` 置为 `false`，但因为 Spec 不再 active，不会发送 `InputReleased` replicated event。
5. `InputReleased` 事件没有自动调用 `EndAbility`；要实现“松开按键结束 Ability”，需要 Ability 显式等待释放并结束自己。

## 源码索引

- `Source/LyraGame/Input/LyraInputComponent.h`
  - `ULyraInputComponent::BindAbilityActions`
- `Source/LyraGame/Character/LyraHeroComponent.cpp`
  - `InitializePlayerInput`
  - `Input_AbilityInputTagPressed`
  - `Input_AbilityInputTagReleased`
- `Source/LyraGame/AbilitySystem/LyraAbilitySet.cpp`
  - `ULyraAbilitySet::GiveToAbilitySystem`
- `Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.cpp`
  - `AbilityInputTagPressed`
  - `AbilityInputTagReleased`
  - `ProcessAbilityInput`
  - `AbilitySpecInputPressed`
  - `AbilitySpecInputReleased`
- `Source/LyraGame/Player/LyraPlayerController.cpp`
  - `PostProcessInput`

相关 Canvas：`Lyra_输入映射_按键到Ability.canvas`

## 相关前置

输入事件最终作用于哪个 ASC、AbilitySpec 和 Avatar，取决于 ActorInfo 的 Owner/Avatar 绑定关系：

- [[Lyra_AbilityActorInfo_生命周期]]
- `Lyra_AbilityActorInfo_生命周期.canvas`

## `InputPressed` 与 `InvokeReplicatedEvent` 的分工

Lyra 的 `AbilitySpecInputPressed` 同时走两条路径，不是二选一：

```text
AbilitySpecInputPressed(Spec)
  ├─ Super::AbilitySpecInputPressed(Spec)
  │    ├─ 更新 Spec.InputPressed = true
  │    └─ 调用活动 Ability 实例的 Ability::InputPressed(...)
  │
  └─ InvokeReplicatedEvent(InputPressed, Spec.Handle, PredictionKey)
       └─ 唤醒 WaitInputPress 等 AbilityTask
```

### `InputPressed`：直接 Ability 回调

`Super::AbilitySpecInputPressed` 是 GAS 原生的 ASC 输入处理入口。它主要负责：

- 更新 `FGameplayAbilitySpec::InputPressed`。
- 如果 Spec 已激活，调用 `UGameplayAbility::InputPressed`。
- 对实例化 Ability，GAS 可以把这个回调分发给当前实例集合。

它是立即执行的函数调用：

```text
ASC 收到输入 → 直接调用 Ability 实例的 InputPressed
```

它适合 C++ Ability 自己重写 `InputPressed` 后立即处理，但本身不是一个用于蓝图异步等待的事件通道。

### `InvokeReplicatedEvent`：AbilityTask 通用事件

Lyra 在调用 GAS 父类后，继续执行：

```cpp
InvokeReplicatedEvent(
    EAbilityGenericReplicatedEvent::InputPressed,
    Spec.Handle,
    OriginalPredictionKey
);
```

`WaitInputPress` 会使用下列信息注册监听：

```text
EventType + SpecHandle + ActivationPredictionKey
```

它的作用是：

- 让 `WaitInputPress` 这种异步 AbilityTask 可以注册等待。
- 即使事件先到、Task 后创建，也可以通过 GAS 事件缓存检查并触发。
- 让客户端预测输入可以由 Task 通过 `ServerSetReplicatedEvent` 同步到服务器。
- 用 SpecHandle 和 PredictionKey 区分具体 AbilitySpec 及其某一次激活。

`InvokeReplicatedEvent` 自己首先是本地事件分发，不是这一行直接完成网络复制；`WaitInputPress` 的任务回调负责后续的预测/服务器同步。

### 为什么不只使用 `InputPressed`

`InputPressed` 能做到“立即通知 Ability 实例”，但它没有直接提供：

```text
AbilityTask 注册/等待
事件先到时的缓存与回放
按 SpecHandle + PredictionKey 匹配事件
WaitInputPress 的客户端/服务器预测同步
```

因此，Lyra 保留 GAS 原生的 `InputPressed` 回调，同时使用通用 replicated event 连接蓝图 AbilityTask：

```text
InputPressed
    给 C++ Ability 或原生回调使用

InvokeReplicatedEvent
    给 WaitInputPress / WaitInputRelease 等 AbilityTask 使用
```

Lyra 源码明确不使用 `bReplicateInputDirectly`，而是采用 `InvokeReplicatedEvent` 配合 `WaitInputPress`。释放路径完全对应：`AbilitySpecInputReleased` 先调用 GAS 原生 `Ability::InputReleased`，再发送 `InputReleased` 通用事件供 `WaitInputRelease` 消费。
