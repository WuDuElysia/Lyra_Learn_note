# Lyra Ability 优先级与打断机制

## 一句话结论

Lyra 没有通用的 `Priority = 100` 数值优先级。Ability 之间的并行、阻塞和打断，是由三层规则组合出来的：

```text
ActivationGroup：控制大范围的并行、替换和阻塞
Ability Tag：精确指定谁阻塞谁、谁取消谁
ActivationRequiredTags / ActivationBlockedTags：判断当前 Ability 能不能启动
```

GameFeature 只负责把 Ability、AbilitySet、GameplayTag 配置和输入模块化加载进来；真正的激活和打断判断发生在 GAS Ability 与 ASC 中。

## 一、Lyra 的 Ability 激活链

```text
输入或 Gameplay Event
    ↓
ASC::TryActivateAbility
    ↓
ULyraGameplayAbility::CanActivateAbility
    ├─ GAS 原生：拥有、Tag、Cooldown、Cost 等检查
    └─ Lyra 自定义：检查 ActivationGroup 是否被阻塞
    ↓
Ability 激活成功
    ↓
ULyraAbilitySystemComponent::NotifyAbilityActivated
    ↓
加入激活组，并按规则取消可替换 Ability
```

关键源码：

- `Source/LyraGame/AbilitySystem/Abilities/LyraGameplayAbility.cpp`
  - `CanActivateAbility`
  - `SetCanBeCanceled`
- `Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.cpp`
  - `IsActivationGroupBlocked`
  - `NotifyAbilityActivated`
  - `AddAbilityToActivationGroup`
  - `CancelActivationGroupAbilities`

## 二、第一层：ActivationGroup

Lyra 在 `ULyraGameplayAbility` 中定义了三个激活组：

```cpp
enum class ELyraAbilityActivationGroup : uint8
{
    Independent,
    Exclusive_Replaceable,
    Exclusive_Blocking
};
```

### `Independent`

可以和其他 Ability 并行运行，不参与 Exclusive Ability 的互相阻塞和替换。

适合：

```text
被动能力
镜头能力
瞄准辅助
不影响主战斗流程的 Ability
```

### `Exclusive_Replaceable`

属于主要独占 Ability，但允许被新的 Exclusive Ability 取消和替换。

适合：

```text
普通攻击
连招攻击
蓄力攻击
普通技能
```

Lyra 要求这个组中的 Ability 必须允许取消，否则会破坏“可替换”语义。

### `Exclusive_Blocking`

激活后阻止其他 Exclusive Ability 激活。

适合：

```text
死亡
眩晕
过场
强制处决
不希望被普通技能打断的技能
```

Lyra 的 `IsActivationGroupBlocked` 规则可以简化为：

```text
当前有 Exclusive_Blocking：
    新的 Exclusive_Replaceable 不能激活
    新的 Exclusive_Blocking 不能激活

Independent：
    仍然可以激活
```

## 三、普通攻击如何被技能打断

假设：

```text
GA_NormalAttack
    ActivationGroup = Exclusive_Replaceable

GA_Skill
    ActivationGroup = Exclusive_Blocking
```

流程：

```text
普通攻击激活
    ↓
ASC 中存在 Exclusive_Replaceable

技能请求激活
    ↓
技能通过激活组检查
    ↓
技能激活成功
    ↓
Lyra 取消当前的 Exclusive_Replaceable
    ↓
普通攻击收到 CancelAbility
```

在 `ULyraAbilitySystemComponent::AddAbilityToActivationGroup` 中，新的 Exclusive Ability 会调用：

```cpp
CancelActivationGroupAbilities(
    ELyraAbilityActivationGroup::Exclusive_Replaceable,
    LyraAbility,
    false
);
```

因此，最简单的“技能打断普通攻击”配置就是：

```text
普通攻击：Exclusive_Replaceable
技能：Exclusive_Blocking
```

如果技能本身也允许被其他技能替换，则可以让技能使用 `Exclusive_Replaceable`。

## 四、第二层：Ability Tag 的 Block 和 Cancel

激活组只能表达比较粗的关系。更精确的关系使用 GAS 的 Ability Tag 配置：

```text
AbilityTags
BlockAbilitiesWithTag
CancelAbilitiesWithTag
ActivationRequiredTags
ActivationBlockedTags
```

例如：

```text
GA_NormalAttack
    AbilityTags = Ability.Attack.Normal

GA_Skill
    AbilityTags = Ability.Skill
    CancelAbilitiesWithTag = Ability.Attack.Normal
    BlockAbilitiesWithTag = Ability.Attack.Normal
```

两种 Tag 的区别：

```text
CancelAbilitiesWithTag：
    取消已经正在运行的匹配 Ability

BlockAbilitiesWithTag：
    当前 Ability 运行期间，阻止匹配 Ability 新激活
```

因此技能可以同时做到：

```text
已经在打的普通攻击 → 被取消
新的普通攻击请求 → 被阻止
```

## 五、ActivationRequiredTags 和 ActivationBlockedTags

这两个描述的是“当前 Ability 是否允许启动”：

```text
ActivationRequiredTags：
    ASC 必须拥有这些 Tag 才能激活

ActivationBlockedTags：
    ASC 拥有这些 Tag 时不能激活
```

例如：

```text
GA_NormalAttack
    ActivationBlockedTags:
        State.Stunned
        State.Dead
        State.Dodging
```

表示角色处于眩晕、死亡或闪避状态时，普通攻击不能启动。

注意不要混淆：

```text
ActivationBlockedTags：
    判断“我自己能不能启动”

BlockAbilitiesWithTag：
    判断“我启动后阻止谁”

CancelAbilitiesWithTag：
    判断“我启动后取消谁”
```

## 六、Lyra 的 TagRelationshipMapping

Lyra 还提供一个数据资产：

```text
ULyraAbilityTagRelationshipMapping
```

文件：

```text
Source/LyraGame/AbilitySystem/LyraAbilityTagRelationshipMapping.h
Source/LyraGame/AbilitySystem/LyraAbilityTagRelationshipMapping.cpp
```

一条关系可以表达：

```text
AbilityTag：
    Ability.Skill

AbilityTagsToBlock：
    Ability.Attack.Normal

AbilityTagsToCancel：
    Ability.Attack.Normal

ActivationRequiredTags：
    可选

ActivationBlockedTags：
    可选
```

`ULyraAbilitySystemComponent` 会在：

```text
GetAdditionalActivationTagRequirements
ApplyAbilityBlockAndCancelTags
```

中读取这份关系表，把额外规则合并进 GAS 的激活、阻塞和取消流程。

这样可以把“技能会打断普通攻击”的规则集中放在数据资产中，而不是分散写到每个 Ability 上。

## 七、绝区零风格战斗的推荐配置

### 普通攻击

```text
GA_NormalAttack
    ActivationGroup = Exclusive_Replaceable
    AbilityTags = Ability.Attack.Normal
    CanBeCanceled = true
```

普通攻击可以被技能、闪避或受击反应打断。

### 普通技能

如果技能释放期间不希望被其他普通技能打断：

```text
GA_Skill
    ActivationGroup = Exclusive_Blocking
    AbilityTags = Ability.Skill
    CancelAbilitiesWithTag = Ability.Attack.Normal
```

如果技能也可以被更高优先级动作替换：

```text
GA_Skill
    ActivationGroup = Exclusive_Replaceable
    CancelAbilitiesWithTag = Ability.Attack.Normal
```

### 闪避

闪避通常需要打断普通攻击，并在一小段时间内无敌：

```text
GA_Dodge
    AbilityTags = Ability.Dodge
    CancelAbilitiesWithTag = Ability.Attack.Normal
```

然后应用：

```text
GE_DodgeInvincible
    GrantedTag = State.Invincible
    Duration = 闪避无敌帧长度
```

如果闪避必须打断正在运行的技能，不能只依赖 `Exclusive_Blocking`：

```text
当前技能是 Exclusive_Blocking
    ↓
普通 Exclusive Ability 会被激活组直接拒绝
```

这时可以使用显式取消规则、让技能变为 `Exclusive_Replaceable`，或者扩展 ASC 实现真正的高优先级仲裁。

### 眩晕和死亡

```text
GA_Stunned
    ActivationGroup = Exclusive_Blocking

GA_Death
    ActivationGroup = Exclusive_Blocking
```

普通攻击、技能和闪避都可以配置：

```text
ActivationBlockedTags:
    State.Stunned
    State.Dead
```

## 八、Lyra 没有数值 Priority

Lyra 没有一个 GAS 自动读取的：

```cpp
Priority = 100;
```

它使用的是规则组合：

```text
ActivationGroup：
    粗粒度控制并行、替换和阻塞

AbilityTag / BlockTag / CancelTag：
    精确控制谁打断谁

RequiredTag / BlockedTag：
    控制当前能否激活

Cooldown / Cost：
    控制资源和冷却条件
```

如果普通攻击和技能在同一帧都请求激活，Lyra 本身没有一个通用数字系统保证“技能一定先处理”；最终还会受到请求顺序、激活条件和服务器权威结果影响。

## 九、是否应该给自己的 GA 增加数值优先级

可以增加：

```cpp
UPROPERTY(EditDefaultsOnly)
int32 InterruptPriority = 0;
```

但仅添加字段没有效果。还需要在 ASC 或统一激活入口中实现：

```text
请求激活新 Ability
    ↓
查找当前活动 Ability
    ↓
比较双方 InterruptPriority
    ↓
新 Ability 更高：取消旧 Ability 并激活
新 Ability 更低：激活失败
```

多人联机时，这套判断必须在客户端预测路径和服务器权威路径中保持一致，否则可能出现：

```text
客户端认为技能打断了普通攻击
服务器认为技能不能激活
```

因此第一版建议优先使用：

```text
ActivationGroup
+ AbilityTags
+ BlockAbilitiesWithTag
+ CancelAbilitiesWithTag
```

它们已经足够实现：

```text
闪避 > 技能 > 普通攻击
```

需要更多等级时，再扩展自定义 ASC 优先级仲裁。

## 十、GameFeature 在这里负责什么

```text
GameFeature：
    加载 Combat 模块
    注册输入映射
    授予 Ability、AbilitySet、AttributeSet
    在停用时回收功能

ASC / GameplayAbility：
    判断 Ability 能否激活
    处理激活组
    应用 Block / Cancel 规则
    处理网络激活和取消
```

所以不要把优先级判断写进 `GameFeatureAction_AddAbilities`。GameFeature 负责“把能力装进角色”，ASC 负责“能力之间如何运行”。

## 源码索引

- `Source/LyraGame/AbilitySystem/Abilities/LyraGameplayAbility.h`
  - `ELyraAbilityActivationGroup`
  - `ActivationGroup`
- `Source/LyraGame/AbilitySystem/Abilities/LyraGameplayAbility.cpp`
  - `CanActivateAbility`
  - `SetCanBeCanceled`
- `Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.cpp`
  - `IsActivationGroupBlocked`
  - `NotifyAbilityActivated`
  - `AddAbilityToActivationGroup`
  - `CancelActivationGroupAbilities`
  - `ApplyAbilityBlockAndCancelTags`
- `Source/LyraGame/AbilitySystem/LyraAbilityTagRelationshipMapping.h`
  - `FLyraAbilityTagRelationship`
  - `ULyraAbilityTagRelationshipMapping`
- `Source/LyraGame/AbilitySystem/LyraAbilityTagRelationshipMapping.cpp`
  - `GetAbilityTagsToBlockAndCancel`
  - `GetRequiredAndBlockedActivationTags`
- `Source/LyraGame/GameFeatures/GameFeatureAction_AddAbilities.cpp`
  - GameFeature 注入和回收 Ability 的位置

相关笔记：

- [[Lyra_GA与GE_创建应用_NotifyAbility]]
- [[Lyra_Delegate与GameplayEvent_通知链路]]
- `Lyra_Ability优先级与打断机制.canvas`
