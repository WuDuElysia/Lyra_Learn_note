# AbilitySystem 计划

> 返回 [[计划蓝图]] · 当前代码事实见 [[模块参考]]

## 3. AbilitySystem 层：照抄 Lyra 的部分

### 3.1 可以直接照抄的清单

| Lyra 文件 | GGYGO 对应位置 | 照抄程度 | 说明 |
|---|---|---|---|
| `AbilitySystem/LyraAbilitySystemComponent.*` | `AbilitySystem/GGYGOAbilitySystemComponent.*` | 照抄 + 扩展 | 输入 Tag 缓存、`ProcessAbilityInput`、TagRelationship 保留；激活组部分重写，见第 5 章 |
| `AbilitySystem/LyraAbilitySet.*` | `AbilitySystem/GGYGOAbilitySet.*` | 直接照抄 | 成组授予 Ability/Effect/AttributeSet 并可整组回收，队伍换角色必需 |
| `AbilitySystem/LyraAbilitySystemGlobals.*` | `AbilitySystem/GGYGOAbilitySystemGlobals.*` | 直接照抄 | 只为了让自定义 EffectContext 生效 |
| `AbilitySystem/LyraGameplayEffectContext.*` | `AbilitySystem/GGYGOGameplayEffectContext.*` | 直接照抄 | 携带 AbilitySource 与命中结果，伤害结算必需 |
| `AbilitySystem/LyraAbilitySourceInterface.*` | 同名迁移 | 直接照抄 | 配合上面的 EffectContext |
| `AbilitySystem/LyraAbilityTagRelationshipMapping.*` | 同名迁移 | 直接照抄 | Tag 级 Block/Cancel 关系表，与第 5 章的组优先级互补 |
| `AbilitySystem/Abilities/LyraGameplayAbility.*` | `AbilitySystem/Abilities/GGYGOGameplayAbility.*` | 照抄 + 扩展 | 保留 ActivationPolicy、AdditionalCosts、失败反馈、相机模式；扩展优先级与组 |
| `AbilitySystem/Abilities/LyraAbilityCost*.*` | `AbilitySystem/Abilities/GGYGOAbilityCost.h` | 照抄基类 | Lyra 的三个实现绑定 Inventory，GGYGO 的具体消耗待需求出现后放 `Costs/` |
| `AbilitySystem/Attributes/LyraAttributeSet.h` | `AbilitySystem/Attributes/GGYGOAttributeSet.h` | 直接照抄 | 基类 + `ATTRIBUTE_ACCESSORS` 宏 |
| `AbilitySystem/Attributes/LyraHealthSet.*` | `AbilitySystem/Attributes/GGYGOHealthSet.*` | 照抄 + 扩展 | Health/MaxHealth + Damage/Healing 元属性 + 死亡委托；再加韧性/削韧 |
| `AbilitySystem/Attributes/LyraCombatSet.*` | `AbilitySystem/Attributes/GGYGOCombatSet.*` | 直接照抄 | BaseDamage/BaseHeal |
| `AbilitySystem/Executions/LyraDamageExecution.cpp` | `AbilitySystem/Executions/GGYGODamageExecution.*` | 照抄结构 | 距离衰减换成动作游戏的部位倍率/防御结算 |
| `AbilitySystem/LyraGameplayCueManager.*` | `AbilitySystem/Cues/GGYGOGameplayCueManager.*` | 直接照抄 | 只做异步/预加载优化，不含 Cue 执行逻辑 |
| `AbilitySystem/LyraGlobalAbilitySystem.*` | 暂缓 | 待需求 | D1/D2 下价值很高（给整支队伍或全场角色广播 Buff），但目前没有全场 Buff 的玩法 |
| `AbilitySystem/Phases/LyraGamePhase*` | 暂缓 | 待需求 | 用 Ability 表达关卡阶段，现阶段无需求 |
| `Character/LyraCharacterWithAbilities.*` | 普通 Pawn 自持 ASC 及默认属性集时的参考 | 按需参考 | 玩家 Slot 与可换形态 Boss 都使用外置宿主；不再把“每个角色自带 ASC”当成全局规则 |

---

---

## 5. GA 配置层：优先级 + 激活组 + 组规则

### 5.1 Lyra 的事实与局限

Lyra 有 `ELyraAbilityActivationGroup`，只有三个值：

```cpp
Independent            // 永不阻塞，也不被阻塞
Exclusive_Replaceable  // 可被其他 Exclusive 取消替换
Exclusive_Blocking     // 阻止所有其他 Exclusive 激活
```

判定实现（`LyraAbilitySystemComponent.cpp::IsActivationGroupBlocked`）是按枚举索引的固定数组 `ActivationGroupCounts[3]`，逻辑只有一句：**只要当前有任何 Blocking 能力，就拒绝所有新的 Exclusive**。

局限：

1. **只有一个"独占"概念，没有"组"。** 不能表达"技能之间互斥，但技能和普攻可以共存"。
2. **没有优先级数值。** `Exclusive_Replaceable` 之间是后到者无条件取消先到者——大招会被普攻打断。
3. **策略耦合在 GA 上。** 想改"技能组只能同时一个"这条规则，得改每个技能 GA 的配置。

### 5.2 方案：三个正交维度

| 维度 | 配在哪 | 类型 | 职责 |
|---|---|---|---|
| **组身份** `GroupTag` | GA 资产 | `FGameplayTag` | 这个 GA 属于哪个组。如 `AbilityGroup.Attack` / `.Skill` / `.Ultimate` / `.Dodge` / `.HitReact` |
| **组规则** `GroupRule` | 独立 DataAsset | 枚举 + 参数 | 该组内部的并发规则。**配在组上，不配在 GA 上**——改规则只改一处 |
| **自身策略** `SelfPolicy` + `Priority` | GA 资产 | 枚举 + int32 | 这个 GA 在组规则之上的个体行为，以及冲突时的强弱 |

**组规则**（`EGGYGOAbilityGroupRule`）：

| 值 | 语义 | 典型用途 |
|---|---|---|
| `Coexist` | 组内任意多个可同时激活 | Buff 类、被动 |
| `SingleInstance` | 组内同时只能一个，按优先级决定去留 | 技能组、大招组 |
| `SingleInstanceQueued` | 组内同时只能一个，新请求进缓冲队列而不是取消旧的 | **普攻连段**（等前一段结束再出下一段） |

**自身策略**（`EGGYGOAbilitySelfPolicy`）：

| 值 | 语义 |
|---|---|
| `Coexist` | 不主动排斥任何人，只受组规则约束 |
| `Exclusive` | 激活期间排斥**所有**组的低优先级 GA（不只是同组）。用于大招、死亡、被击倒 |

**优先级** `Priority`（int32，越大越强）。分段留空隙便于插值：

```
死亡 / 强制状态       1000
被击倒 / 强硬直        800
大招                  600
技能                  400
闪避                  300
重攻击                200
轻攻击                100
被动 / Buff             0
```

### 5.3 仲裁算法（含 D4）

把 Lyra 的固定数组换成按组 Tag 索引的实例表：

```cpp
// 取代 Lyra 的 ActivationGroupCounts[3]
TMap<FGameplayTag, TArray<TWeakObjectPtr<UGGYGOGameplayAbility>>> ActiveAbilitiesByGroup;
```

`CanActivateAbility` 阶段（只判断，不改状态）：

1. 取请求 GA 的 `GroupTag` / `Priority` / `SelfPolicy`
2. **全局排斥检查**：遍历所有已激活 GA，若存在 `SelfPolicy == Exclusive` 且 `Priority > 请求者Priority` 的实例 → 拒绝
3. **同组规则检查**：查 `GroupTag` 对应的 `GroupRule`
   - `Coexist` → 通过
   - `SingleInstance` → 若组内已有实例且其 `Priority > 请求者Priority` → 拒绝；否则通过（标记待取消列表）
   - `SingleInstanceQueued` → 若组内已有实例 → 拒绝激活，但把意图压回输入缓冲队列
4. **Tag 关系检查**：交给照抄来的 `GGYGOAbilityTagRelationshipMapping`，处理与组无关的 Tag 级 Block/Cancel

`NotifyAbilityActivated` 阶段（改状态）：

5. 把新实例登记进 `ActiveAbilitiesByGroup`
6. 执行第 3 步标记的取消列表（`CancelAbilityHandle`）
7. 若 `SelfPolicy == Exclusive`，取消所有 `Priority <` 自己的其他组实例

`OnAbilityEnded` 阶段：

8. 从 `ActiveAbilitiesByGroup` 摘除
9. 若该组规则是 `SingleInstanceQueued`，通知输入缓冲队列尝试出队

**D4 的落点在第 2、3 步：比较用 `>` 而不是 `>=`，即同优先级时后来者胜出并打断先激活者。**

这带来一个必须注意的后果：**普攻连段不能靠"同级先到先得"实现了**。第二段普攻如果和第一段同 Priority，会直接打断第一段，动画从头切。两个解法：

- 用 `SingleInstanceQueued` 组规则，让第二段进队列等第一段结束（推荐，语义清晰）
- 或给连段各段递增 Priority（100 / 101 / 102），靠优先级递增实现顺序推进

### 5.4 落地结构

```
AbilitySystem/
  GGYGOAbilitySystemComponent.h/.cpp     组表 + 仲裁实现
  Abilities/
    GGYGOGameplayAbility.h/.cpp          GroupTag / Priority / SelfPolicy 字段
  Groups/                                （新目录）
    GGYGOAbilityGroupConfig.h/.cpp       DataAsset：Map<GroupTag, FGGYGOAbilityGroupRule>
    GGYGOAbilityGroupTypes.h             枚举 + 规则结构体
```

---

## 6. GE 层

原本认为没有可抄的，实际上有三个值得抄：

1. **`LyraGameplayEffectContext`**：自定义 EffectContext，携带 `AbilitySource` 和命中的物理材质。"打到金属出火花、打到肉出血"就靠这个。
2. **`LyraDamageExecution`**：伤害结算用 `ExecutionCalculation` 而不是简单 Modifier。价值是能同时读源和目标的属性（攻击力 vs 防御力）、能读 EffectContext（命中部位）、能一次算完再写回。动作游戏的伤害公式必然需要。
3. **元属性模式**（`LyraHealthSet` 的 `Damage`/`Healing`）：GE 改的是 `Damage` 元属性，`PostGameplayEffectExecute` 里转换成 `Health` 扣减并清零。伤害可以被拦截、修正、触发事件，而不是直接扣血。GGYGO 的 `UGGYGOHealthSet` 用同样的模式，并多一个 `PoiseDamage` 元属性走韧性。

GGYGO 需要的 GE 分类（都是编辑器里的蓝图资产，代码侧只提供 Execution 与 Tag）：

| 分类 | 用途 | 归属 |
|---|---|---|
| 伤害 / 治疗 | 走 `GGYGODamageExecution`，用 `SetByCaller` 传数值 | `UGGYGOGameData`（15.4） |
| 硬直 / 受击 | 施加 `Restriction.*` Tag + 持续时间，动作游戏的核心 | 施加它的能力 |
| 削韧 / 破防 | 韧性属性的消耗与恢复 | 施加它的能力 |
| 冷却 | GAS 原生的 `CooldownGameplayEffectClass` | 能力自己 |
| 无敌帧 | 闪避的 i-frame，`Restriction.ImmuneDamage` | 闪避能力 |

**不做全局 GE 注册表**（一个集中保存所有 GE 软引用路径、启动时同步加载的命名空间）。
路径拼错只能在运行时发现、引用关系在编辑器的资产引用图里不可见、
全局可变变量无法按玩法模式差异化、启动时同步加载全部 GE 会拖慢启动。
共享的那几个 GE 走 `UGGYGOGameData` 资产（15.4），其余归属施加它们的能力。

---

## 7. GameplayEvent 层

直接照抄。GAS 原生的 `SendGameplayEventToActor` + `AbilityTask_WaitGameplayEvent` + `FGameplayEventData` 已经够用，Lyra 没有做额外封装。

动作游戏要额外接的两处：

1. **AnimNotify → GameplayEvent**：动画帧上的判定窗口、连段窗口、可取消点，用 AnimNotify 发 GameplayEvent，GA 内用 `WaitGameplayEvent` 接。这是动作游戏最常用的模式。动作类 Notify 一律走这条路，不要让 AnimInstance 直连调用逻辑层的具体类型——那样动画层就成了逻辑层的上游，且被调用方换实现时动画层要跟着改。
2. **命中 → GameplayEvent**：命中判定组件检测到命中后发事件，GA 决定要不要应用伤害 GE。

`GameplayMessageRuntime`（已在 `GGYGO.Build.cs` 声明）和 GameplayEvent 是两个不同东西：

| | GameplayEvent | GameplayMessage |
|---|---|---|
| 目标 | 特定 Actor 的 ASC | 全局广播，无目标 |
| 用途 | 驱动 Ability 逻辑 | 通知 UI / 音频 / 成就等旁观者 |
| 载荷 | `FGameplayEventData`（固定结构） | 任意 USTRUCT |

---
