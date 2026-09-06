# Lyra AbilityActorInfo 生命周期：Owner、Avatar 与 ASC

## 一句话结论

`AbilitySystemComponent` 的创建和 `AbilityActorInfo` 的完整绑定不是同一件事。标准 Lyra 玩家由 `PlayerState` 持有可跨死亡/重生复用的 ASC，`PlayerState` 是 `OwnerActor`，当前 Pawn 是 `AvatarActor`；Pawn 重生时通常不新建 ASC，而是把同一个 ASC 重新绑定到新 Pawn。

## 标准关系

```text
PlayerController
      │ Possess / 控制
      ▼
当前 Pawn（AvatarActor） ← PawnExtension / HeroComponent 协调初始化
      │ Controller、Mesh、Movement、AnimInstance 等 Avatar 数据
      ▼
PlayerState（OwnerActor）
      │ 持有
      ▼
LyraAbilitySystemComponent（AbilitySystemComponent）
```

在标准玩家路径中：

| ActorInfo 内容 | Lyra 中的典型来源/含义 | 生命周期 |
| --- | --- | --- |
| `OwnerActor` | `ALyraPlayerState`，能力归属、权限和持久数据对象 | 通常跨 Pawn 死亡/重生保持不变 |
| `AvatarActor` | 当前 `ALyraCharacter` / Pawn，能力实际表现和受伤对象 | Possess、重生、换 Pawn 时变化 |
| `AbilitySystemComponent` | PlayerState 上的 `ULyraAbilitySystemComponent` | 通常随 PlayerState 保持 |
| `PlayerController` | GAS 根据 Owner/Avatar 关系解析，Controller 变化后刷新 | 复制、Possess、UnPossess 时可能暂时为空或变化 |
| `SkeletalMeshComponent` | 从 Avatar 的 Character/Pawn 获取 | 随 Avatar 变化 |
| `AnimInstance` | 从 Avatar 的 SkeletalMesh 获取 | Mesh/Avatar 更换后刷新 |
| `MovementComponent` | 从 Avatar 的 Pawn 获取；LyraCharacter 使用自定义移动组件 | 随 Avatar 变化 |

`OwnerActor` 与 `AvatarActor` 不是同一个概念：

```text
OwnerActor  = 谁拥有这套能力系统、保存能力和属性
AvatarActor = 这套能力当前附着在哪个角色身上
```

## 典型初始化时序

### 1. PlayerState 创建 ASC

`ALyraPlayerState` 构造函数创建默认子对象：

```cpp
AbilitySystemComponent =
    ObjectInitializer.CreateDefaultSubobject<ULyraAbilitySystemComponent>(
        this,
        TEXT("AbilitySystemComponent")
    );
```

此时只是：

```text
PlayerState 上有一个 ASC
```

还不能简单理解成所有 ActorInfo 字段都已经完整有效。

### 2. PlayerState 早期建立 ActorInfo

`ALyraPlayerState::PostInitializeComponents` 调用：

```cpp
AbilitySystemComponent->InitAbilityActorInfo(
    this,
    GetPawn()
);
```

此时：

```text
OwnerActor = PlayerState
AvatarActor = 当时的 GetPawn()，可能为空或还不是最终 Pawn
```

这是一个早期初始化，主要先建立 ASC 的 Owner 关系；真正的当前 Pawn Avatar 还要由 Pawn 初始化流程再次确认。

### 3. AbilitySet 在 authority 上授予能力

经验加载后，`ALyraPlayerState::SetPawnData` 遍历 `PawnData->AbilitySets`：

```text
SetPawnData
  → AbilitySet::GiveToAbilitySystem
  → 创建 AttributeSet
  → GiveAbility 创建 AbilitySpec
  → 应用 GameplayEffect
```

`GiveToAbilitySystem` 只在 authority 上真正修改 ASC。AbilitySet 可能在 Avatar 已经绑定前授予，也可能在 Avatar 已经存在后通过装备、GameFeature 等系统动态授予，因此不能假设所有 AbilitySpec 都恰好在同一时刻产生。

### 4. PawnExtension 在 Pawn 初始化状态满足后绑定 Avatar

标准玩家的 `ULyraHeroComponent::HandleChangeInitState` 在：

```text
DataAvailable → DataInitialized
```

时取得：

```text
当前 Pawn
当前 PlayerState
PlayerState 上的 Lyra ASC
```

然后调用：

```cpp
PawnExtensionComponent->InitializeAbilitySystem(
    LyraPlayerState->GetLyraAbilitySystemComponent(),
    LyraPlayerState
);
```

`ULyraPawnExtensionComponent::InitializeAbilitySystem` 再调用：

```cpp
InASC->InitAbilityActorInfo(
    InOwnerActor, // PlayerState
    Pawn          // 当前 Pawn
);
```

这一步才是标准玩家把：

```text
PlayerState ASC ↔ 当前 Pawn Avatar
```

正式绑定起来的关键入口。

## Lyra ASC 覆盖的 `InitAbilityActorInfo`

`ULyraAbilitySystemComponent::InitAbilityActorInfo` 的顺序是：

```text
1. 记录旧 Avatar 是否与新 Pawn 不同
2. 调用 GAS 父类 InitAbilityActorInfo
3. GAS 父类刷新 Owner、Avatar、Controller、Mesh、AnimInstance、Movement 等关系
4. 如果是新的 Pawn Avatar：
   4.1 通知已有 Lyra Ability 实例 OnPawnAvatarSet
   4.2 注册到 LyraGlobalAbilitySystem
   4.3 初始化 LyraAnimInstance
   4.4 TryActivateAbilitiesOnSpawn
```

Lyra 的判断是：

```cpp
const bool bHasNewPawnAvatar =
    Cast<APawn>(InAvatarActor) &&
    (InAvatarActor != ActorInfo->AvatarActor);
```

因此，普通的 `RefreshAbilityActorInfo` 不一定代表出现了新 Pawn Avatar；Lyra 的 `OnPawnAvatarSet` 主要针对新 Pawn。

## Owner、Avatar 变化时的几个函数

### `InitAbilityActorInfo(Owner, Avatar)`

用于建立或重新建立完整的 Owner/Avatar 关系。标准 Lyra 重生时通常再次调用：

```text
同一个 PlayerState ASC
    + 新的 PlayerState OwnerActor
    + 新的 Pawn AvatarActor
```

### `RefreshAbilityActorInfo()`

用于 Owner/Avatar 仍然是同一组对象，但派生字段可能变化的场景，例如：

```text
Controller 复制完成
Possess / Controller 改变
PlayerController 晚于 PlayerState 到达客户端
```

`ULyraPawnExtensionComponent::HandleControllerChanged` 会在当前 Pawn 仍是 Avatar 时调用它。它不等于重新创建 ASC，也不等于一定触发 Lyra 的 `OnPawnAvatarSet`。

### `SetAvatarActor(nullptr)`

用于解除当前 Pawn 与 ASC 的 Avatar 关系，但保留 OwnerActor 和 ASC：

```text
AvatarActor = nullptr
OwnerActor  = PlayerState
ASC         = 仍然存在
```

### `ClearActorInfo()`

当 OwnerActor 也无效时，才需要清除整套 ActorInfo。标准玩家死亡/换 Pawn 通常优先使用 `SetAvatarActor(nullptr)`，因为 PlayerState 和 ASC 仍然要保留。

## Pawn 解绑和重生

Pawn 销毁、死亡或 UnPossess 时，`ULyraPawnExtensionComponent::UninitializeAbilitySystem` 通常执行：

```text
1. 确认这个 Pawn 仍然是 ASC 的 Avatar
2. 取消普通 Ability，但保留 Ability.Behavior.SurvivesDeath
3. ClearAbilityInput
4. RemoveAllGameplayCues
5. Owner 有效：SetAvatarActor(nullptr)
6. Owner 无效：ClearActorInfo()
```

下一次重生时：

```text
旧 Pawn 被解除 Avatar
  ↓
同一个 PlayerState ASC 保留
  ↓
新 Pawn 创建
  ↓
新 Pawn 的 PawnExtension 初始化
  ↓
ASC.InitAbilityActorInfo(PlayerState, NewPawn)
  ↓
新 Pawn 成为 AvatarActor
```

如果客户端短时间同时看见旧 Pawn 和新 Pawn，`InitializeAbilitySystem` 会先找到旧 Avatar 对应的 PawnExtension 并解除旧绑定，避免同一个 ASC 同时挂在两个 Pawn 上。

## ActorInfo 和 Ability 实例的关系

要区分三个对象：

```text
Ability CDO
    AbilitySet 从 Ability 类取得的默认对象

FGameplayAbilitySpec
    ASC 中保存的运行时授予规格

Ability 实例
    激活后根据 InstancingPolicy 创建的运行时对象
```

当前 Lyra 默认：

```cpp
InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
```

因此一个 Spec 通常对应一个可复用的 Ability 实例。Ability 实例激活后可以从：

```cpp
CurrentActorInfo
```

读取当前 ASC、OwnerActor、AvatarActor、PlayerController 等上下文。

但 AbilitySet 中的：

```cpp
AbilitySpec.Ability
```

通常是 Ability CDO，不要把 CDO 的 `CurrentActorInfo` 当成当前角色实例的 ActorInfo。

Lyra 的 `OnGiveAbility` 接收显式的：

```cpp
const FGameplayAbilityActorInfo* ActorInfo
```

而 Ability 实例在实际激活后使用自己的 `CurrentActorInfo`。如果 Ability 在 Avatar 尚未有效时被授予，`OnGiveAbility` 可以发生，但依赖当前 Pawn 的 OnSpawn 激活可能要等新的 Avatar 绑定后由 Lyra 再次尝试。

## `OnGiveAbility`、`OnPawnAvatarSet` 与激活

### `OnGiveAbility`

表示：

```text
AbilitySpec 被授予 ASC
```

Lyra 的实现顺序：

```text
Super::OnGiveAbility
  → K2_OnAbilityAdded
  → TryActivateAbilityOnSpawn(ActorInfo, Spec)
```

它不是 Pawn 换 Avatar 的回调。

### `OnPawnAvatarSet`

这是 Lyra 自定义的能力实例回调，不是当前工程中名为 `OnAvatarSet` 的覆写。新 Pawn Avatar 绑定后，Lyra ASC 遍历已有实例并调用：

```cpp
LyraAbilityInstance->OnPawnAvatarSet();
```

适合在这里重新获取：

```text
Pawn 组件
CharacterMovementComponent
Mesh
Avatar 相关的运行时引用
```

### `TryActivateAbilitiesOnSpawn`

Lyra 在新 Pawn Avatar 绑定、全局 ASC 注册和动画初始化之后调用它。它遍历 AbilitySpec，根据 Ability CDO 的 `ActivationPolicy` 尝试激活 `OnSpawn` 能力。

## GAS 原生与 Lyra 自定义

### GAS 原生

- `FGameplayAbilityActorInfo` 及其 Owner/Avatar/ASC/Controller/Anim/Mesh/Movement 关系。
- `UAbilitySystemComponent::InitAbilityActorInfo`。
- `RefreshAbilityActorInfo`、`SetAvatarActor`、`ClearActorInfo`。
- `FGameplayAbilitySpec`、Ability 实例和 `CurrentActorInfo`。
- `OnGiveAbility`、Ability 激活、结束和取消生命周期。
- InstancingPolicy 对 Ability 实例数量和复用方式的影响。

### Lyra 自定义

- `ALyraPlayerState` 持有持久 ASC。
- `ULyraPawnExtensionComponent` 协调 ASC 与 Pawn Avatar 的绑定/解绑。
- `ULyraHeroComponent` 用 Lyra 初始化状态推动绑定时机。
- `ULyraAbilitySystemComponent::InitAbilityActorInfo` 的新 Pawn 检测、`OnPawnAvatarSet`、全局 ASC 注册、动画初始化和 OnSpawn 尝试。
- `ULyraGameplayAbility::OnPawnAvatarSet`。
- 死亡/换 Pawn 时保留 `Ability_Behavior_SurvivesDeath` 的取消策略。
- `PlayerState::SetPawnData` 调用 AbilitySet 授予默认能力。

### UE 框架层

- PlayerController 的 Possess/UnPossess。
- Pawn、PlayerState、Controller 的复制和 `OnRep`。
- ActorComponent 的初始化状态链。
- Character 的 Mesh、MovementComponent、AnimInstance 生命周期。

## 一条完整标准玩家时序

```text
PlayerState 构造
  → 创建 Lyra ASC
  → PostInitializeComponents：InitAbilityActorInfo(PlayerState, GetPawn())
  → ExperienceLoaded / SetPawnData：Authority 授予 AbilitySet
  → GameMode 生成新 Pawn 并设置 PawnData
  → HeroComponent 推进 DataAvailable → DataInitialized
  → PawnExtension.InitializeAbilitySystem(ASC, PlayerState)
  → ASC.InitAbilityActorInfo(PlayerState, NewPawn)
  → GAS 刷新 ActorInfo 派生字段
  → Lyra 实例 OnPawnAvatarSet
  → 注册全局 ASC、初始化动画
  → TryActivateAbilitiesOnSpawn
  → PlayerController.PostProcessInput 驱动同一个 ASC

死亡 / UnPossess：
  → UninitializeAbilitySystem
  → Cancel 普通 Ability
  → ClearAbilityInput / GameplayCue
  → SetAvatarActor(nullptr)

重生：
  → 复用 PlayerState 和 ASC
  → 新 Pawn 重新成为 AvatarActor
```

## 源码索引

- `Source/LyraGame/Player/LyraPlayerState.cpp`
  - `ALyraPlayerState::ALyraPlayerState`
  - `ALyraPlayerState::PostInitializeComponents`
  - `ALyraPlayerState::SetPawnData`
  - `ALyraPlayerState::GetAbilitySystemComponent`
- `Source/LyraGame/Character/LyraPawnExtensionComponent.cpp`
  - `InitializeAbilitySystem`
  - `UninitializeAbilitySystem`
  - `HandleControllerChanged`
  - `CheckDefaultInitialization`
- `Source/LyraGame/Character/LyraHeroComponent.cpp`
  - `HandleChangeInitState`
  - `InitializePlayerInput`
- `Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.cpp`
  - `InitAbilityActorInfo`
  - `TryActivateAbilitiesOnSpawn`
  - `EndPlay`
- `Source/LyraGame/AbilitySystem/Abilities/LyraGameplayAbility.cpp`
  - 构造函数：`InstancedPerActor`
  - `OnGiveAbility`
  - `OnRemoveAbility`
  - `OnPawnAvatarSet`
  - `TryActivateAbilityOnSpawn`
- `Source/LyraGame/AbilitySystem/LyraAbilitySet.cpp`
  - `GiveToAbilitySystem`
- `Source/LyraGame/Player/LyraPlayerController.cpp`
  - `OnRep_PlayerState`
  - `OnUnPossess`
  - `PostProcessInput`
- `Source/LyraGame/Character/LyraCharacter.cpp`
  - `PossessedBy`
  - `UnPossessed`
  - `OnRep_Controller`
  - `UninitAndDestroy`

相关笔记：

- [[Lyra_输入映射_按键到Ability]]
- `Lyra_AbilityActorInfo_生命周期.canvas`

## 相关能力与效果笔记

- [[Lyra_GA与GE_创建应用_NotifyAbility]]：记录 GA/GE 的创建、应用、Handle 区别，以及 ASC 的 `NotifyAbility...` 生命周期回调。