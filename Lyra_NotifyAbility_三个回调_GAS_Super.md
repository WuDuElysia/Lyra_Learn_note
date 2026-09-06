# NotifyAbility 三个回调：GAS Super 与 Lyra 扩展

## 一句话结论

`NotifyAbilityActivated`、`NotifyAbilityFailed`、`NotifyAbilityEnded` 是 `UAbilitySystemComponent` 观察 Ability 生命周期结果的三个 ASC 回调。Lyra 重写它们时先调用 `Super`，让 GAS 完成原生广播、Spec/实例清理和结束状态维护，再追加 Lyra 的激活组计数、失败表现路由和激活组回收。

这三个函数不是三个“激活 Ability 的函数”，而是 GAS 在 Ability 生命周期变化后通知 ASC 的统一扩展点。

## 一、先明确调用关系

```text
GAS 内部激活流程
      │
      ├─ 成功：虚调用 LyraASC::NotifyAbilityActivated
      │             ├─ Super → GAS 原生回调广播
      │             └─ Lyra → 加入 ActivationGroup
      │
      ├─ 失败：虚调用 LyraASC::NotifyAbilityFailed
      │             ├─ Super → GAS 原生失败回调广播
      │             └─ Lyra → 路由 FailureReason
      │
      └─ 结束：虚调用 LyraASC::NotifyAbilityEnded
                    ├─ Super → GAS 清理 Spec/实例/结束状态
                    └─ Lyra → 移出 ActivationGroup
```

这里的 `Super` 不是“再执行一次 Ability”，而是 C++ 父类实现：

```text
GAS 调用虚函数
  → 实际进入 ULyraAbilitySystemComponent 的 override
  → Lyra override 调用 Super
  → Super 进入 UAbilitySystemComponent 的 GAS 原生实现
  → 返回 Lyra override
  → Lyra 执行自己的项目逻辑
```

## 二、当前工程能确认什么，哪些属于 Engine 内部

### 当前 Lyra 工程可以直接确认

`Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.cpp` 中三个 override 都明确先调用父类：

```cpp
Super::NotifyAbilityActivated(...);
Super::NotifyAbilityFailed(...);
Super::NotifyAbilityEnded(...);
```

之后分别执行：

- Activated：`AddAbilityToActivationGroup`；
- Failed：`ClientNotifyAbilityFailed` 或 `HandleAbilityFailed`；
- Ended：`RemoveAbilityFromActivationGroup`。

### GAS Engine 内部

当前工作区只包含 Lyra 项目，UE Engine 源码不在工作区，因此不能把本地行号假装成 Engine 源码行号。下面的 `Super` 内容依据 `UAbilitySystemComponent` 的公开接口和公开 UE 源码实现契约整理；不同 UE 小版本的内部顺序或附加字段可能变化，但“回调广播、活动计数、实例清理、Spec 脏标记”这些职责是理解 Lyra 的关键。

## 三、`NotifyAbilityActivated` 的 GAS Super

### 1. 什么时候调用

```text
TryActivateAbility
  → GAS 检查 Ability 条件
  → Ability 激活成功
  → ASC.NotifyAbilityActivated(Handle, Ability)
```

它代表：

> GAS 已经确认这次 Ability 激活成功，现在通知 ASC 及其监听者。

它不是：

- `GiveAbility`；
- `CanActivateAbility`；
- `ActivateAbility`；
- 输入按下事件。

### 2. GAS Super 的核心工作

GAS 父类实现的核心是广播原生 Ability 激活回调：

```cpp
AbilityActivatedCallbacks.Broadcast(Ability);
```

这意味着其他 GAS 系统可以订阅：

```text
某个 Ability 已经开始运行
```

父类的重点不是给 Lyra 做激活组，而是把“激活成功”作为 GAS 通用事件向外广播。

`Handle` 仍然作为回调参数传入父类接口，便于 ASC 的生命周期契约和派生类使用；GAS 的基础激活广播主要把 `Ability` 传给 `AbilityActivatedCallbacks`。

### 3. Lyra 在 Super 之后做什么

```cpp
if (ULyraGameplayAbility* LyraAbility =
    Cast<ULyraGameplayAbility>(Ability))
{
    AddAbilityToActivationGroup(
        LyraAbility->GetActivationGroup(),
        LyraAbility
    );
}
```

Lyra 追加的是项目规则：

```text
GAS 广播“激活成功”
  → Lyra 增加 ActivationGroupCounts
  → Independent 不影响其他能力
  → Exclusive_Replaceable 可被替换
  → Exclusive_Blocking 阻塞其他 Exclusive
```

把激活组放在这个回调中，意味着输入激活、GameplayEvent 激活、OnSpawn 激活和代码激活都能统一计数，而不需要每个 Ability 自己写一份规则。

### 4. 这一步不负责什么

`NotifyAbilityActivated` 不负责：

- 创建 AbilitySpec；
- 创建 Ability 实例；
- 执行 Ability 的攻击、移动或动画逻辑；
- 应用 GameplayEffect；
- 发送 `WaitInputPress` 事件。

技能行为仍然由 Ability 的 `ActivateAbility` 负责。

## 四、`NotifyAbilityFailed` 的 GAS Super

### 1. 什么时候调用

```text
TryActivateAbility
  → GAS 检查失败
  → 收集 FailureReason
  → ASC.NotifyAbilityFailed(Handle, Ability, FailureReason)
```

失败原因可能来自：

- 冷却还没有结束；
- Cost 不足；
- Required Tags 缺失；
- Blocked Tags 存在；
- Lyra ActivationGroup 被阻塞；
- Ability 当前不允许激活。

### 2. GAS Super 的核心工作

GAS 父类实现的核心是把失败结果广播给原生失败回调：

```cpp
AbilityFailedCallbacks.Broadcast(
    Ability,
    FailureReason
);
```

因此订阅 `AbilityFailedCallbacks` 的系统可以知道：

```text
哪个 Ability 失败了
为什么失败
```

`FailureReason` 是 `FGameplayTagContainer`，它把失败原因作为标签集合传递，而不是只返回一个无意义的 `false`。

GAS Super 不会：

- 重新尝试激活；
- 强制绕过冷却或 Cost；
- 调用 `ActivateAbility`；
- 自动播放 Lyra 的失败 Montage。

它主要完成原生失败通知。

### 3. Lyra 在 Super 之后做什么

Lyra 代码根据 Avatar 是否本地控制、Ability 是否支持网络来决定失败表现发送位置：

```text
Super::NotifyAbilityFailed
  → 远程控制且支持网络
      → ClientNotifyAbilityFailed
          → 拥有客户端
  → 否则
      → HandleAbilityFailed
          → OnAbilityFailedToActivate
```

这样服务器判定的失败可以在正确的玩家客户端显示：

- 冷却提示；
- 资源不足提示；
- 失败音效；
- 失败动画；
- UI 反馈。

`NotifyAbilityFailed` 是“统一处理一次失败尝试”的入口；它不等同于 `CanActivateAbility`。

### 4. 为什么不直接在 `CanActivateAbility` 里做表现

`CanActivateAbility` 是检查函数，可能被多次查询：

```text
UI 查询能否释放
输入系统查询
客户端预测查询
服务器真正激活时查询
```

如果在里面播放音效或修改 UI，就会产生重复表现。

`NotifyAbilityFailed` 则表示：

```text
这次真正的激活尝试已经失败
GAS 已经给出了最终的 FailureReason
```

所以它更适合做失败表现路由。

## 五、`NotifyAbilityEnded` 的 GAS Super

### 1. 什么时候调用

```text
Ability::EndAbility 或 CancelAbility
  → Ability 告诉 ASC 自己结束
  → ASC.NotifyAbilityEnded(Handle, Ability, bWasCancelled)
```

它不是结束 Ability 的入口，而是 Ability 结束后由 GAS 进行收尾的入口。

参数：

| 参数 | 作用 |
| --- | --- |
| `Handle` | 找到对应的 AbilitySpec |
| `Ability` | 已经结束的 Ability 实例或对象 |
| `bWasCancelled` | 区分正常结束和被取消结束 |

### 2. GAS Super 的核心清理步骤

公开 GAS 实现可以概括为以下顺序：

```text
1. 根据 Handle 找 AbilitySpec
2. 如果 Spec 已被移除，直接结束清理
3. 广播 AbilityEndedCallbacks
4. 如果它正在驱动 Montage，清除 AnimatingAbility
5. 减少 Spec.ActiveCount
6. InstancedPerExecution：移除并清理本次实例
7. Authority：处理 RemoveAfterActivation 或 MarkAbilitySpecDirty
8. 广播带结束信息的 OnAbilityEnded
```

这些步骤可以分别理解。

#### 找回 Spec

GAS 需要通过：

```cpp
FindAbilitySpecFromHandle(Handle)
```

找到这次 Ability 所属的 `FGameplayAbilitySpec`。

如果 Ability 结束时 Spec 已经被清除，GAS 会认为相关授予记录已经完成回收，不再继续依赖这个 Spec。

#### 广播结束回调

GAS 会广播原生结束回调，使其他系统可以知道：

```text
这个 Ability 已经结束
```

这和 Lyra 的激活组清理不同。GAS 广播是通用生命周期通知，Lyra 计数是项目规则。

#### 清理动画引用

如果结束的 Ability 正是 ASC 当前记录的 `AnimatingAbility`，GAS 会清掉这份引用，避免 ASC 继续认为已经结束的 Ability 正在驱动 Montage。

#### 减少 `Spec.ActiveCount`

一个 AbilitySpec 可能有活动实例或多次激活记录，因此 GAS 不只是保存一个简单的 `bool`。

结束时父类会减少：

```text
Spec.ActiveCount
```

并进行下溢检查。这个计数关系到：

- `Spec.IsActive()` 的判断；
- 是否还有活动实例；
- 是否可以执行 RemoveAfterActivation；
- 后续输入分支应该把它当作活动 Ability 还是未激活 Ability。

#### 清理 `InstancedPerExecution`

如果 Ability 的实例策略是：

```text
InstancedPerExecution
```

那么每次激活可能产生一个独立实例。

该实例结束后，GAS 会根据复制策略和 authority：

- 从 ReplicatedInstances 或 NonReplicatedInstances 移除；
- 在适当条件下标记待销毁；
- 不让已经结束的执行实例继续留在 ASC 的实例列表中。

这也是为什么 `NotifyAbilityEnded` 不能只理解成一个“打印日志的通知”。它承担真正的运行时回收工作。

#### Authority 上处理 Spec

如果当前是 authority，GAS 还会处理：

```text
Spec.RemoveAfterActivation
```

当该 Spec 被标记为激活结束后移除，并且已经没有其他活动实例时，GAS 可以清除这个 AbilitySpec。

否则，GAS 会标记 AbilitySpec 已经脏了：

```text
MarkAbilitySpecDirty
```

这样复制系统才能把 Spec 的活动状态变化同步出去。

#### 广播最终结束信息

GAS 还会通过带有 Ability、Handle 和取消状态的信息广播结束结果，让监听者知道：

```text
哪个 Ability
属于哪个 Spec
是否被取消
```

### 3. Lyra 在 Super 之后做什么

Lyra 的代码在父类完成清理后执行：

```cpp
RemoveAbilityFromActivationGroup(
    LyraAbility->GetActivationGroup(),
    LyraAbility
);
```

也就是：

```text
GAS 先减少 Spec.ActiveCount、清理实例和复制状态
  → Lyra 再减少 ActivationGroupCounts
```

这样可以避免 Ability 已经结束，但 Lyra 激活组还一直认为它在运行。

### 4. `EndAbility` 和 `NotifyAbilityEnded` 的区别

```text
EndAbility / CancelAbility
    Ability 主动结束或被取消

NotifyAbilityEnded
    GAS 接到结束消息后，清理自己的运行时状态并通知监听者
```

不要写成：

```text
NotifyAbilityEnded 调用了 EndAbility
```

更准确的是：

```text
EndAbility 完成
  → GAS 处理 NotifyAbilityEnded
```

## 六、三个 GAS Super 的对照

| 回调 | GAS Super 的主要职责 | Lyra override 的主要职责 |
| --- | --- | --- |
| `NotifyAbilityActivated` | 广播 `AbilityActivatedCallbacks` | 增加激活组计数，执行独占/替换规则 |
| `NotifyAbilityFailed` | 广播 `AbilityFailedCallbacks(Ability, FailureReason)` | 通过 RPC 或本地函数路由失败表现 |
| `NotifyAbilityEnded` | 查找 Spec、广播结束、清理 Montage、减少 `ActiveCount`、清理实例和复制状态 | 减少激活组计数 |

共同结构是：

```cpp
Super::NotifyAbilityXxx(...);
LyraProjectSpecificLogic(...);
```

这不是形式主义，而是为了保留 GAS 的核心状态机，再叠加 Lyra 的项目规则。

## 七、和输入事件的区别

```text
AbilitySpecInputPressed
    输入按下后的直接回调

InvokeReplicatedEvent
    给 WaitInputPress / WaitInputRelease 的通用事件

NotifyAbilityActivated
    激活结果：成功

NotifyAbilityFailed
    激活结果：失败

NotifyAbilityEnded
    Ability 生命周期：结束/取消
```

例如第二次按 Q：

```text
AbilitySpecInputPressed
  → Ability::InputPressed
  → InvokeReplicatedEvent(InputPressed)
  → WaitInputPress
```

这不会再次触发：

```text
NotifyAbilityActivated
```

只有一次新的激活尝试成功时，才会进入激活成功通知路径。

## 八、和 GameplayEffect 的区别

GE 没有这三个 `NotifyAbility` 回调，因为 GE 不属于 Ability 生命周期。

```text
GA：激活 → 执行 → 结束

GE：创建 Spec → 应用 → ActiveEffect 生效 → 到期/移除
```

GE 侧对应的观察机制包括：

- GameplayEffect Applied / ActiveEffect Added Delegate；
- ActiveEffect Removed Delegate；
- ActiveEffect Stack Change Delegate；
- `AttributeSet::PreAttributeChange`；
- `AttributeSet::PostGameplayEffectExecute`；
- GameplayCue；
- `FActiveGameplayEffectHandle`。

因此：

```text
NotifyAbility
    观察和扩展 Ability 生命周期

GameplayEffect Delegate / AttributeSet 回调
    观察 GE 应用、属性结算和 ActiveEffect 生命周期
```

一个 Ability 结束，也不代表它创建的 GE 同时结束：

```text
攻击 Ability
  → 应用燃烧 GE
  → Ability EndAbility
  → NotifyAbilityEnded
  → 燃烧 GE 仍然持续
```

## 九、当前 Lyra 源码对照

### `NotifyAbilityActivated`

文件：`Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.cpp`

```text
Super::NotifyAbilityActivated
  → Cast<ULyraGameplayAbility>
  → AddAbilityToActivationGroup
```

### `NotifyAbilityFailed`

文件：`Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.cpp`

```text
Super::NotifyAbilityFailed
  → 非本地控制且支持网络：ClientNotifyAbilityFailed
  → 否则：HandleAbilityFailed
  → ULyraGameplayAbility::OnAbilityFailedToActivate
```

### `NotifyAbilityEnded`

文件：`Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.cpp`

```text
Super::NotifyAbilityEnded
  → Cast<ULyraGameplayAbility>
  → RemoveAbilityFromActivationGroup
```

### 当前工程的头文件声明

文件：`Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.h`

- `NotifyAbilityActivated`：能力开始激活后调用；父类先处理通用状态；
- `NotifyAbilityFailed`：激活失败时调用；Lyra 决定本地处理或 RPC；
- `NotifyAbilityEnded`：结束或取消后调用；父类清理后减少组计数；
- `ClientNotifyAbilityFailed`：客户端失败表现 RPC；
- `HandleAbilityFailed`：转发到 Lyra Ability 的失败回调。

## 十、最终记忆模型

```text
TryActivateAbility
    申请激活

CanActivateAbility
    检查资格

NotifyAbilityActivated Super
    GAS 广播“激活成功”

NotifyAbilityActivated Lyra
    记录激活组、执行独占/替换规则

NotifyAbilityFailed Super
    GAS 广播“激活失败 + FailureReason”

NotifyAbilityFailed Lyra
    把失败表现发送到正确客户端

EndAbility / CancelAbility
    Ability 结束或取消

NotifyAbilityEnded Super
    GAS 清理 Spec、ActiveCount、实例、动画引用和复制状态

NotifyAbilityEnded Lyra
    清理激活组计数
```

## 十一、引擎源码边界与参考

当前项目工作区不包含 UE Engine 源码，不能把 Engine 内部实现的具体行号当作当前本地源码行号。笔记中的 GAS Super 行为按公开 `UAbilitySystemComponent` API 和公开源码实现契约整理；如果 UE 小版本改变了内部广播或实例清理顺序，应以对应版本的 Engine 源码为准。

参考：

- [Epic Games：UAbilitySystemComponent API](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/GameplayAbilities/UAbilitySystemComponent)
- [公开 UE 源码镜像：AbilitySystemComponent_Abilities.cpp](https://github.com/ylyking/UnrealEngineNiv/blob/master/Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp)
- [[Lyra_GA与GE_创建应用_NotifyAbility]]
