# GGYGO Boss AI 计划蓝图

> **状态：已确认，实施中。**
>
> 2026-09-18：阶段 A–C 已完成。`GGYGOEditor Win64 Development` 与
> `GGYGO Win64 Development` 均通过 UHT、编译和链接；阶段 B 的单 Form/单 Phase
> 最小装配、阶段 C 的确定性选招、BT→GAS 激活桥及近战攻击竖切均通过单机 PIE。
> `GameplayCue.Hit.Flesh` 已由运行时 Cue Map 注册；命中 Notify 尚未配置可见/可听表现。
> Brain 暂停、死亡单次边沿与 Dedicated Server 复制仍待专项回归。
>
> 2026-09-30：14a E10/E11代码/测试已冻结，统筹完成UHT、完整C++构建及三项BossSelection自动化（`ModuleRepairGate_20260930_9`，项目38/38、0 warning/error）。14b E9回收源码/测试已冻结，统筹第13次门禁完整Editor构建及两项Encounter专项自动化通过（项目47/47、succeededWithWarnings/failed/notRun均0）；真实BT SafeStop/异步Abort、BT→GAS、PIE世界EndPlay与联机仍待验，不将夹具覆盖扩大为完整动态验收。
>
> 本文专门规划 Boss 战斗 AI、阶段/形态切换与持久 ASC 宿主。总体架构依据见
> [[计划蓝图]]，当前代码事实见 [[模块参考]]，配套图见 [[GGYGO_流程_BossAI.canvas]]。
>
> 参考资产：`F:\ue_project\GGYGO\Exports\HT` 中的 NTE 导出 JSON/C++ 描述。
> 导出内容能可靠说明资产关系与配置形态，但不能代替未导出的原生 `HTGame` 类源码。

---

## 0. 需求定义

这里的“切换形态”不是同一个 Pawn 上换 Mesh，而是允许：

- 第一形态与第二形态使用不同 PawnClass、骨骼、碰撞、移动和表现组件；
- 旧 Pawn 退场并销毁，新 Pawn 成为新的 Avatar；
- 切换前后的生命、韧性、Buff、Debuff、冷却和一次性机制继续存在；
- AIController、当前目标与仇恨不因换 Pawn 丢失；
- 所有实际战斗状态仍通过 GAS 改变，所有移动仍经 CMC/PathFollowing；
- 服务器是 Boss 决策、阶段变化、形态切换和伤害结算的唯一权威。

因此 Boss 的 ASC 必须与 Pawn 生命周期分离。需要复用的是
**ASC Owner/Avatar 的装配机制与 GAS 执行链**，不是玩家编队业务。

### 0.1 非目标

本轮规划不包含：

- 某一只 Boss 的具体数值、技能与动画内容；
- Mass AI、开放世界大规模 NPC 调度；
- 玩家锁定系统的完整交互与相机方案；
- 下场队友 AI；
- 立刻修改源码。本文审阅通过后再按第 13 章分阶段实施。

---

## 1. 从 NTE 导出资产确认到的结构

### 1.1 通用 Boss 外壳

`/Game/Blueprints/AI/BehaviorTree/BT_Boss` 负责通用生命周期：

- `HTBTService_CheckSense`：感知检查；
- `HTBTService_FindAggroTarget`：选择仇恨目标，配置为约 `0.6 ± 0.2s` 评估一次；
- `HTBTService_UpdateAggroList`：更新仇恨；
- `HTBTService_CheckLeaveBattle`：脱战；
- Controller 的 `HasEnemyTree` 指向每只 Boss 自己的战斗树；
- Idle、追踪、搜索、逃离战斗等行为拆成可复用子树。

这说明成熟项目也会把“通用战斗生命周期”与“具体 Boss 出招”分开。

### 1.2 Boss 专属决策树

Boss 33 的主战斗树按 60% 生命划分阶段，阶段子树使用：

- 距离与角度 Decorator；
- 冷却 Decorator；
- 动态权重随机 Composite；
- `HTBTTask_ActiveAbility` 作为叶节点；
- 追击子树处理暂时没有合适攻击的情况。

第一阶段约 7 个主动动作，第二阶段约 11 个；第二阶段包含独立的转阶段技能。

### 1.3 Ability 承担动作执行

`GA_boss_33_act07_BP` 的导出配置表明它继承 Montage 技能基类，并配置：

- Montage；
- Montage Event → 生成投射物；
- Montage Event → 目标选择 + GE 容器；
- 释放角度与技能冷却。

即：行为树只选择并激活 Ability，Montage、命中、投射物、GE 和结束语义都在 GA 内。
这与 GGYGO 已有的 `PlayMontageAndWaitForEvent`、MeleeTrace、DamageExecution 和组仲裁一致。

### 1.4 被动反应

Boss 09 的阶段树还监听：

- 玩家使用带 `Ability.Melee` 的技能；
- Boss 连续受到近战伤害；
- Boss 离出生点过远。

监听结果生成“近战反应”“击飞玩家”“回场中心”等被动行为请求，并各自带冷却。
说明正常攻击池与高优先级反应池应分开。

### 1.5 不照搬的部分

NTE 的 `BB_Monster` 已有约 68 个 Key，其中大量是 `ShouldXxx` 布尔量；不同阶段、
难度也复制了大量树节点。同时，BT Decorator 与 GA 配置里都出现了冷却。

GGYGO 不复制这些做法：

- Blackboard 只存短期决策上下文，不镜像 GameplayTag、阶段和冷却；
- 技能冷却只有 GAS/GE 一个真源；
- 技能条件与权重进入 ActionSet 数据资产，避免阶段树重复铺节点；
- 反应事件产生“请求”，不直接修改玩法状态。

---

## 2. 已选设计决策

| # | 决策 | 内容 |
|---|---|---|
| **B1** | **生命周期决定 ASC 宿主** | 玩家角色由 CharacterSlot 持有；换形态 Boss 由 BossState 持有；无需跨 Pawn 保留状态的普通怪物允许 Pawn 自持。 |
| **B2** | **抽取最薄的 CombatantState** | 只抽 ASC、基础 AttributeSet、Avatar 引用与绑定；不抽玩家编队、AI、阶段或 PawnData 业务。 |
| **B3** | **Behavior Tree 只做决策** | 感知、目标、追击、动作选择和等待；不播放 Montage、不施加伤害、不直接切阶段。 |
| **B4** | **GAS 是动作执行唯一入口** | AI 不伪造 InputTag，BT Task 直接请求 ASC 激活已授予的 AbilitySpec。 |
| **B5** | **目标与仇恨分离** | CombatTarget 提供唯一当前目标；Threat 维护候选与分数；GA、转向和未来 Motion Warping 读取同一目标接口。 |
| **B6** | **阶段与形态是两个概念** | Phase 是战斗规则阶段；Form 是当前 Avatar 形态。阶段可以不换形态，也可以触发形态更换。 |
| **B7** | **形态切换采用 Exit → Handoff → Enter** | 依赖旧 Pawn Montage 的 Ability 必须先结束，再更换 Avatar；新 Pawn 就绪后再播放入场 Ability。 |
| **B8** | **形态切换不是死亡** | 旧 Pawn 走 RetireAvatar，不触发死亡、掉落、击杀统计或 Encounter 结束。 |
| **B9** | **Boss 能力默认一次性授予** | 所有阶段/形态 Ability 在 BossState 初始化时授予，通过 Form/Phase Tag 限制；避免换 Avatar 时动态增删 Spec。 |
| **B10** | **冷却只归 GAS** | ActionSet、BT 不保存技能冷却；选择动作时调用 ASC 的可激活性检查。 |
| **B11** | **服务器决策** | Boss GA 使用 `ServerInitiated`（或明确的服务器策略），AIController 只存在于服务器；客户端只接收 Pawn、属性、Tag、Montage 与 Cue。 |
| **B12** | **不新增逐帧全局调度器** | 感知走引擎回调，仇恨定时低频评估，BT 由 BrainComponent 驱动，动作逐帧部分仍属于 CMC/GAS/AnimInstance。 |

### 2.1 对总计划 D1 的修订范围

原 D1“ASC 挂 Slot、不挂 Pawn”只对**玩家编队角色**成立。更一般的规则改为：

> ASC Owner 的生命周期必须覆盖它承载的玩法状态；Pawn 只是可替换 Avatar 时，ASC 外置；
> 状态与 Pawn 同生共死时，允许 Pawn 自持 ASC。

这不是放弃 Slot，而是把 Slot 从“所有角色唯一布局”收窄为“玩家角色的正确布局”。

---

## 3. 总体结构

```text
AGGYGOBossEncounter / 生成方
│
├─ AGGYGOBossState                         [持久、复制]
│  ├─ UGGYGOAbilitySystemComponent
│  ├─ UGGYGOHealthSet / UGGYGOCombatSet
│  ├─ BossDefinition
│  ├─ CurrentPhaseTag / CurrentFormTag
│  └─ AvatarPawn ───────────────────────────────┐
│                                               │
├─ AGGYGOBossAIController                       │
│  ├─ AIPerception                              │
│  ├─ UGGYGOThreatComponent                     │
│  ├─ UGGYGOCombatTargetComponent               │
│  └─ BehaviorTree                              │
│       └─ ChooseAction → ActivateAbility → Wait│
│                                               ▼
└──────────────────────────────────── AGGYGOBossCharacter
                                        PawnExtension
                                        HealthComponent
                                        CMC
                                        形态 Mesh / Anim / Trace
```

依赖方向：AI 决策层依赖 GAS 的公开请求接口；GAS 不依赖 BehaviorTree。
BossState 不依赖具体 Boss Pawn 子类，只依赖 PawnExtension 契约。

---

## 4. 运行时类设计

### 4.1 `AGGYGOCombatantState`：外置 ASC 的最小公共基类

计划目录：`Source/GGYGO/Combatants/`

```cpp
UCLASS(Abstract)
class AGGYGOCombatantState : public AInfo, public IAbilitySystemInterface
{
    UGGYGOAbilitySystemComponent* AbilitySystemComponent;
    UGGYGOHealthSet* HealthSet;
    UGGYGOCombatSet* CombatSet;

    UPROPERTY(ReplicatedUsing=OnRep_AvatarPawn)
    APawn* AvatarPawn;

    void AttachAvatar(APawn* NewAvatar);
    void DetachAvatar(APawn* ExpectedAvatar);
};
```

只负责：

- ASC 与基础 AttributeSet 默认子对象；
- Owner 恒为 CombatantState、Avatar 可变；
- Avatar 引用复制与客户端重绑；
- 保证 `InitAbilityActorInfo` 只有一个装配入口；
- 提供 Avatar 更换前取消能力的公共钩子。

明确不负责：PawnData、AbilitySet 业务、玩家 Owner、AI、阶段、形态、生成和销毁。

### 4.2 `AGGYGOCharacterSlot`

改为继承 `AGGYGOCombatantState`，保留现有玩家语义：

- 固定一份 PawnData，只初始化一次；
- 从 PawnData 授予玩家角色 AbilitySet；
- `Owner = PlayerController`，保证预测键；
- ASC 使用 `Mixed`；
- SquadComponent 继续持有并切换 Slot；
- 对外行为不变，属于结构性搬迁而非玩家功能重写。

### 4.3 `AGGYGOBossState`

计划目录：`Source/GGYGO/AI/Boss/`

职责：

- 持有 `UGGYGOBossDefinition`；
- 初始化 Boss 的持久 AbilitySet；
- 保存并复制 `CurrentPhaseTag`、`CurrentFormTag`；
- 监听 HealthComponent/HealthSet 的生命边沿，服务器判断阶段阈值；
- 协调 Exit、Avatar Handoff、Enter；
- ASC 使用 `Minimal`；
- Boss 可配置 `bAlwaysRelevant`，确保 Boss UI、Tag 和 Cue 对参战客户端可见；
- 不持有 BehaviorTree 和仇恨。

### 4.4 `AGGYGOBossCharacter`

继承 `AGGYGOCharacterBase`，是纯 Avatar：

- 不创建第二套 ASC/AttributeSet；
- 不挂 HeroComponent；
- 通过 PawnExtension 接收 BossState ASC；
- 挂载形态所需的 Mesh、碰撞、CMC、Trace 和表现组件；
- `GetAbilitySystemComponent()` 仍只转发 PawnExtension；
- 提供 `RetireAvatarForFormChange()`，不得复用死亡流程。

具体形态可以是不同 Blueprint 子类，不要求共享骨骼或 AnimBP。

### 4.5 `AGGYGOBossAIController`

职责：

- AI Perception；
- 持有 Threat 与 CombatTarget；
- 运行通用 Boss BehaviorTree；
- 换形态时保持 Controller 实例，只重新 Possess；
- Avatar 切换期间暂停普通战斗分支；
- 新 Pawn Possess 后更新 SelfPawn，但保留目标与仇恨。

Controller 不持有生命、阶段、冷却或 Buff。

### 4.6 `AGGYGOBossEncounter`

当前已实现：按Definition生成State、Controller与初始Avatar，提供EncounterSeed及公开实例查询。14b新增显式创建责任与失败/EndPlay统一回收；源码和两项测试已冻结，统筹完整Editor构建及两项专项自动化通过，真实BT/PIE/联机仍待验。

- 三份弱创建记录由实际Spawn返回值登记，独立于公开引用、Actor Owner和当前Avatar；只回收自己创建的State/Controller/初始Avatar，不回收替换进来的外部Avatar。
- 实际装配为State初始化 → Controller生成/Seed初始化 → Deferred Avatar/SetPawnData → State `AttachAvatar` → `FinishSpawningActor` → Controller `Possess` → 核对创建记录、对象与关联后发布三引用；回调清理/EndPlay后旧装配不得继续发布成功。
- `EndPlay`先标记结束，再调用protected `CleanupCreatedBoss`；装配失败`FailSpawn`也调用同一入口。清理先快照、清公开引用/创建记录，再`ClearActionSelection`/Brain `StopLogic` → `UnPossess` → State `DetachAvatar(CurrentAvatar)` → 仅Destroy创建Avatar/Controller/State。每步复核有效性，重复清理无重复副作用。
- State当前Avatar若是外部Pawn，只经宿主接口解绑并解除Controller关系；Owner指向Encounter也不销毁。初始创建Avatar即使Owner改变仍回收。ExpectedASC/ActorInfo清理归CombatantState/PawnExtension，Encounter不重复实现。
- 同步装配/清理守卫拒绝回调重入Spawn；EndPlay结束标志拒绝后续生成。失败回滚后可重试；不新增阶段/动作状态机或第二帧调度器。

完整Director的后续目标（未实现）：

- 保存出生点、战斗区域和参与者；
- 开战、脱战重置、胜利和奖励；
- BossState 的生命周期上限由 Encounter 决定；
- 不参与逐个技能选择。

当前关卡Actor已承担最小生成/回收；待多Boss遭遇需求出现后再评估完整Director。契约与验收边界见[[BossAI/结构#14b：Encounter创建责任与回收（完整构建、两项专项自动化通过）|Encounter回收说明]]，时序见[[GGYGO_流程_BossAI.canvas|主流程撤场分支]]。

---

## 5. 数据资产

### 5.1 `UGGYGOBossDefinition`

```text
InitialFormTag
InitialPhaseTag
Forms[]
Phases[]
PersistentAbilitySets[]
BehaviorTree
ThreatConfig
LeashConfig
```

它回答“这只 Boss 是什么”，不保存运行时状态。

### 5.2 Form 与 Phase

```cpp
FGGYGOBossFormDefinition
    FormTag
    AvatarPawnData
    ExitAbility
    EnterAbility

FGGYGOBossPhaseDefinition
    PhaseTag
    EnterHealthThreshold
    TargetFormTag          // 可与当前相同
    ActionSet
    TransitionAbility     // 阶段演出；若换形态则负责发出 Handoff 请求
    EnterEffects[]
```

区分示例：

- `Phase.2` 可能只让 Boss 狂暴，不换 Pawn；
- `Phase.3` 可以从人形 Form 切到巨兽 Form；
- 同一个巨兽 Form 内仍可有多个 Phase；
- Form Tag 描述当前 Avatar，Phase Tag 描述当前规则集合。

### 5.3 `UGGYGOBossActionSet`

```cpp
FGGYGOBossActionDefinition
    ActionTag
    AbilityClass
    BaseWeight
    MinDistance / MaxDistance
    MaxFacingAngle
    RequiredTags / BlockedTags
    RepeatPenalty
    UnusedWeightGain / MaxWeight
    bRequiresLineOfSight
```

ActionSet 不包含：冷却、伤害、Montage、GE、移动实现。

14a 的 `ValidateConfiguration(OutError)` 与编辑器 `IsDataValid` 共用全集规则：Tag 唯一且与有效能力类的 ActionTag 匹配，所有数值有限，权重/距离非负，角度0–180，惩罚0–1，最远距离为0或不小于最近距离。坏行令整集拒绝，包括基础权重为零的行；空集/全基础零合法。有效上限保持 `max(MaxWeight, BaseWeight)`。`FindAction` 不能在重复 Tag 时返回第一条掩盖歧义。

- 冷却与资源：GA/GE；
- 伤害和表现：GA；
- 移动：MoveTo/CMC 或 GA 发起的 RootMotion；
- 阶段/形态是否允许：GameplayTag + Action 条件。

### 5.4 Boss 能力授予策略

BossState 初始化时将所有阶段会用到的 AbilitySet 去重后一次性授予。各 GA 用：

```text
ActivationRequiredTags: State.Boss.Form.X / State.Boss.Phase.Y
ActivationBlockedTags : State.Boss.Transforming / State.Dying / State.Dead
```

好处：

- Avatar 更换不改变 AbilitySpecHandle；
- 不在网络敏感的 Handoff 中增删能力；
- 冷却能自然跨形态持续；
- BT 只需询问该能力当前是否可激活。

如果未来单只 Boss 的全部形态资产过大，再增加软引用与分阶段预加载；不要先用动态授予解决加载问题。

---

## 6. AI 决策层

### 6.1 通用 BehaviorTree 骨架

```text
Root Selector
├─ Dead / EncounterFinished                     → 停止
├─ State.Boss.Transforming                      → 等待 Handoff/Enter 完成
├─ PendingReaction                              → 选择并请求反应 GA
├─ ShouldLeaveCombat                            → 回场/重置
├─ NoValidTarget                                → AcquireTarget / Search
└─ Combat
   ├─ NeedApproach                              → MoveTo(Target)
   ├─ NeedReposition                            → EQS/导航调整位置
   └─ ChooseAction → ActivateAbility → WaitAbilityEnd
```

“阶段 1/2/3”不需要复制三棵结构相同的树。`ChooseAction` 根据 ASC 的 Phase/Form Tag
选择对应 ActionSet。

### 6.2 Blackboard 最小集合

建议只保留：

```text
SelfPawn
TargetActor
LastKnownTargetLocation
SelectedActionTag（短期）
MoveGoal（短期）
```

以下内容不进入 Blackboard：

- CurrentPhase / CurrentForm：读 BossState/ASC Tag；
- Cooldown：读 ASC；
- IsDead / IsStunned / IsTransforming：读 ASC Tag；
- ThreatTable：归 ThreatComponent；
- Health：读 HealthComponent/ASC；
- 大量 `ShouldXxx`：改成组件查询、Tag 或一次性 Request。

### 6.3 `UBTTask_GGYGOChooseBossAction`

服务器执行：

1. 清旧请求和 Blackboard 选择，从当前 Phase 取得 ActionSet，进行全集校验；
2. 排除 BaseWeight=0，过滤 RequiredTags/BlockedTags（可包含 Form/Phase/状态）；
3. 过滤目标、距离、角度、视线；
4. 查询对应 AbilitySpec 是否存在；
5. 复制原 SpecHandle 后调用 GAS 可激活性检查；保存请求前复核初始 Phase、Set、ASC 和 Pawn，冷却、资源和组冲突不重复实现；
6. `Controller::SelectAction` 对合格项加权抽签；合格项的惩罚权重全部耗尽时只恢复它们的基础权重，随后继续选择；
7. `StoreActionSelection` 记录来源 Set/Phase/ASC/Tag/Class/Spec/Avatar，Blackboard 仍只写 SelectedAction 的 Tag 名。

14a 将 `RepeatPenalty=0` 定义为软避重，不让唯一合法候选永久饿死。BaseWeight=0 保持禁用，不能被恢复逻辑重新启用；无合法候选不抽签。权重更新与求和使用双精度中间值防溢出，随机流只在成功选择时消费一次；ActionSet 来源变化清旧派生权重。

当前使用 EncounterSeed 初始化服务器 `FRandomStream`，后续选择顺序推进该随机流；不另存第二个决策序号。完整候选淘汰原因与最终权重的开发日志仍属后续诊断增强。

### 6.4 `UBTTask_GGYGOActivateAbility`

职责：

1. `ConsumeActionSelection` 先消费一次请求并核对当前 Set/Phase/ASC/Tag/Class/Spec/Avatar，再清 Blackboard；失败也消费，成功只返回选择时的原 SpecHandle。清黑板可能触发观察者，激活前再次检查来源和原 Spec；
2. TargetContext/CombatTarget 仍属阶段D目标接口，当前不实现；
3. 调用 `TryActivateAbility`；
4. 按原 SpecHandle 筛选 AbilityEnded；当前未引入独立激活实例标识；
5. 激活失败立即返回 Failed；当前未输出最终激活失败 Tag；
6. 能力正常结束返回 Succeeded，取消返回 Failed；同步结束也走相同判定；
7. BT 被 Abort 时解除监听，不默认强制取消不可取消的 GA。

短期记录是请求身份，不是第二个动作/阶段状态机；重选、种子重置、附身变化和 EndPlay 清理，来源变化或原 Spec 被移除后拒绝旧请求，不改 BB 资产，也不绕过 `TryActivateAbility` 的最终准入。14a已完整构建并通过三项专项自动化；真实BT→GAS集成和生产资产/PIE验证仍另列。

AI 不调用：

```text
AbilityInputTagPressed
AbilityInputTagReleased
ProcessAbilityInput
```

这些 API 属于玩家输入协议，不是语义动作协议。

### 6.5 反应请求

Threat/Health/目标技能事件可以生成 `FGGYGOAIReactionRequest`：

```text
ReactionTag
SourceActor
TargetActor
Priority
ExpireTime
```

BT 的反应分支只消费请求并尝试激活反应 GA。是否能顶掉当前动作仍由现有
GroupTag + ActivationPriority + SelfPolicy 仲裁，AI 不自己实现第二套打断规则。

---

## 7. 目标、仇恨与移动

### 7.1 `UGGYGOCombatTargetComponent`

提供单一当前目标与变化事件：

```text
CurrentTarget
SetTarget / ClearTarget
OnTargetChanged
IsTargetValid
GetTargetLocation / GetAimLocation
```

未来玩家锁定与 AI 目标都实现同一查询接口。GA、投射物、朝向、Motion Warping 和相机
不直接读取 Blackboard。

### 7.2 `UGGYGOThreatComponent`

服务器维护候选：

- 视觉/听觉感知加入候选；
- 受到伤害增加来源仇恨；
- 治疗、嘲讽等以后通过明确事件增加；
- 目标死亡、离场、不可攻击时移除或降权；
- 事件到达时立即重评，平稳期用约 0.5～0.8 秒低频校正；
- 切换 Avatar 时组件不销毁，仇恨自然保留。

仇恨表不复制。客户端只需要最终目标带来的可见结果，不参与 Boss 决策。

### 7.3 移动边界

- 接近、绕行、回场：AIController `MoveTo` / PathFollowing → CMC；
- 攻击突进、击退、精确落点：GA → RootMotion/RootMotionSource/Motion Warping → CMC；
- 禁止 `SetActorLocation` 做普通战斗移动；
- `Restriction.CantMove` 仍是 locomotion 能否移动的唯一 GameplayTag 真源。

---

## 8. 阶段与形态切换

### 8.1 阈值触发

BossState 在服务器监听 Health 变化，用“跨越阈值边沿”而不是每帧比较：

```text
OldHealthPct > Threshold && NewHealthPct <= Threshold
```

每个 Phase 记录是否已经进入，避免治疗后再次跌破造成重复转阶段。

触发后请求高优先级 Exclusive 的 Transition Ability。阶段状态最终以 ASC Tag/BossState
复制字段为准，BehaviorTree 不自行写 Phase。

### 8.2 不换形态的阶段变化

```text
GA_PhaseTransition 激活
→ State.Boss.Transforming
→ 播 Montage / Cue / 应用 EnterEffects
→ 替换 Phase Tag
→ EndAbility
→ 清 Transforming
→ BT 重新读取 ActionSet
```

### 8.3 换形态的两段式流程

```text
阶段阈值跨越
    ↓
Exit/Transition Ability（旧 Avatar）
    ├─ 加 State.Boss.Transforming
    ├─ 高优先级 Exclusive，阻止普通动作
    └─ 旧 Pawn 退场 Montage 完成后结束
    ↓
BossState::BeginAvatarHandoff
    ├─ 暂停普通 BT 分支与 PathFollowing
    ├─ 取消不允许跨 Avatar 存活的能力
    ├─ Deferred Spawn 新 Form Pawn
    ├─ 注入新 Form PawnData
    ├─ AIController UnPossess 旧 Pawn → Possess 新 Pawn
    ├─ BossState::AttachAvatar(NewPawn)
    ├─ PawnExtension 绑定同一个 ASC
    ├─ 更新 Form/Phase Tag 并 ForceNetUpdate
    └─ 旧 Pawn Retire 后销毁（不走死亡）
    ↓
Enter Ability（新 Avatar）
    ├─ 入场 Montage / Cue / EnterEffects
    └─ 完成后清 State.Boss.Transforming
    ↓
BehaviorTree 恢复正常决策
```

### 8.4 唯一写入者

`AGGYGOCombatantState::AttachAvatar` 是外部更换 Avatar 的唯一入口。它负责调用
PawnExtension 的 ASC 初始化路径并更新复制引用；其它系统不得直接调用
`InitAbilityActorInfo`。

该入口已在阶段 A 收口：玩家 Slot 与 BossState 都继承 `AttachAvatar`，
装配方只调用宿主入口；`DetachAvatar(ExpectedAvatar)` 防止旧 Pawn 的延迟销毁
误清新 Avatar。

### 8.5 活跃 Ability 的处理

默认规则：动作 GA 不允许跨 Avatar 存活。切换前统一取消，TransitionOut 必须已结束。

如未来确有纯逻辑被动能力需要跨 Avatar：

- 它必须不缓存 Mesh、MovementComponent、AnimInstance 或旧 Pawn 指针；
- 在 `OnPawnAvatarSet` 重新绑定；
- 用明确的 `Ability.Behavior.SurvivesAvatarChange` 标记；
- 未标记能力一律结束。

第一版不开放例外，全部结束最安全。

### 8.6 形态切换失败回滚

- 新 Pawn Spawn 失败：保留旧 Pawn，不销毁，清 Transforming 并记录错误；
- 新 Pawn 缺 PawnExtension：同上，拒绝 Handoff；
- 新 Pawn 初始化未到 GameplayReady：保持隐藏/无碰撞，不开始 Enter；
- Boss 在 Handoff 中死亡：死亡优先级高于转阶段，取消切换并走唯一死亡链；
- Controller Possess 失败：恢复旧 Pawn Possess，不能留下有 ASC 无 Avatar 的活 Boss。

---

## 9. GameplayAbility 约束

### 9.1 AI Ability 网络策略

- Boss 主动技能默认 `ServerInitiated`；
- 客户端不预测 Boss 决策；
- ASC 复制激活状态、Montage、Tag 与 Cue；
- 伤害与阶段只由服务器落地。

不要直接沿用玩家动作 GA 默认的 `LocalPredicted` 而不审阅网络策略。

### 9.2 通用战斗动作基类

计划增加 `UGGYGOCombatActionAbility`，将 NTE 的数据化优点接到现有能力任务上：

```text
Montage
GameplayEvent → Begin/End MeleeTrace
GameplayEvent → Apply EffectContainer
GameplayEvent → Spawn Projectile
GameplayEvent → Action Movement / Facing
GameplayEvent → Jump Montage Section
```

玩家攻击与 Boss 攻击都可以复用这个基类。具体 GA 资产主要填数据，避免每个技能复制一张
EventGraph。

### 9.2.1 当前Boss近战共享伤害接缝（15F）

当前`UGGYGOBossMeleeAbility`已保留`DamageEffect`覆盖，并提供蓝图默认false的`bUseSharedDamageEffectWhenUnset`。配置校验`ValidateMeleeConfiguration`与命中`HandleMeleeHit`复用`UGGYGOGameData::ResolveDamageGameplayEffect(覆盖, 显式bool)`：非空覆盖始终优先；空覆盖且false失败；空覆盖且true只有共享GE快照可用才选择成功，共享缺失仍失败。原Montage/Socket/数值/动作配置和激活门禁保留，命中经过原能力状态、窗口、ASC及Authority门禁后解析；解析类缺失在`BuildHitEffectPayload`前报错返回，不GE、不Cue，不能成为Boss无伤害成功。选择矩阵见[[BossAI/结构#15F：Boss近战伤害GE选择（已构建，未专项动态验证）|当前结构契约]]。

[[System/结构|System]]独占启动预载与共享快照；GA仅持有本次解析结果，不新增加载器、重试或跨帧缓存。类可用后继续由[[AbilitySystem/结构|共享载荷构建接口]]生成`BuildHitEffectPayload`，GA填SetByCaller并请求GAS执行。非空类后Spec失败/GE免疫等原执行语义未在15F改变；CMC位移、Animation窗口与CombatTrace查询/去重及清理责任保持原接口。此接缝已经实现，不把上节计划中的通用基类写成已实现。

统筹第15次GGYGOEditor完整构建Succeeded（6 actions/22.10秒）；常规原47项Success，但新增`GGYGO.Input.Fixture.LocalSessionReady`为1 Fail，不是48/48。尚无Boss选择/命中矩阵专项动态证明；实际蓝图默认值回读、PIE与专用服务器/cook继续待验。第15次日志/自动化JSON及本项目`AAADocs/Module_Repair_14b_Validation.md`记录证据。15F结构图与主流程已按独立租约同步实际选择/校验/命中边界并冻结；Boss选择/命中矩阵仍未专项动态验证。

### 9.3 优先级建议

```text
Death                  最高、Exclusive
Boss Form Transition   高、Exclusive
PoiseBreak / HardReact 高
Counter / Reaction     中高
Ultimate / Special     中
Normal Attack          普通
Passive                Coexist
```

精确数值仍由现有 ActivationPriority 规则表配置；这里只定义相对关系。

---

## 10. 复制与相关性

| 对象 | 存在端 | 复制策略 |
|---|---|---|
| BossAIController / BT / Threat | 仅服务器 | 不复制决策内部状态 |
| BossState | 服务器 + 客户端 | Boss 默认 AlwaysRelevant；复制 Avatar、Phase、Form、属性 |
| Boss ASC | BossState 上 | `Minimal`；Tag/Cue 对观察者可见，AI 不需要拥有客户端预测 |
| Boss Pawn | 服务器 + 相关客户端 | 正常空间相关性与移动复制 |
| CombatTarget | 服务器权威 | 第一版不复制完整仇恨，只按表现需求复制当前目标 |
| Encounter | 服务器权威 | 只复制 UI/流程必需状态 |

晚加入客户端必须仅靠当前复制状态恢复：BossState 的 Phase/Form/Avatar 与 ASC Tag 是真源，
不能依赖“当时播放过一次”的本地事件推断当前形态。

---

## 11. 时序与可观测性

每次关键变化输出结构化日志，至少包含：

```text
EncounterId
BossState
Old/New Phase
Old/New Form
Old/New Avatar
AbilitySpecHandle / ActionTag
Target
TransitionReason
```

开发期调试命令建议：

```text
GGYGODumpBossState
GGYGODumpBossThreat
GGYGOForceBossPhase <PhaseTag>
GGYGOForceBossAction <ActionTag>
GGYGOShowBossDecision 0|1
```

`ForceBossPhase` 仍走正常 Transition/Handoff 链，不能直接改成员变量，否则调试路径与正式路径不同。

关键事件委托：

```text
OnBossPhaseChanging / Changed
OnBossFormChanging / Changed
OnBossAvatarChanging / Changed
OnBossTargetChanged
OnBossActionSelected / Failed / Ended
```

这些用于调试、UI 与自动化等待，不再额外建立逐帧调度器。

---

## 12. 计划目录

```text
Source/GGYGO/
├─ Combatants/
│  ├─ GGYGOCombatantState.h/.cpp
│  └─ GGYGOCombatantTypes.h
├─ AI/
│  ├─ GGYGOAIController.h/.cpp
│  ├─ Targeting/
│  │  ├─ GGYGOCombatTargetComponent.h/.cpp
│  │  └─ GGYGOThreatComponent.h/.cpp
│  ├─ BehaviorTree/
│  │  ├─ BTTask_GGYGOChooseBossAction.h/.cpp
│  │  ├─ BTTask_GGYGOActivateAbility.h/.cpp
│  │  └─ BTDecorator_GGYGOCanActivateAbility.h/.cpp
│  └─ Boss/
│     ├─ GGYGOBossState.h/.cpp
│     ├─ GGYGOBossCharacter.h/.cpp
│     ├─ GGYGOBossAIController.h/.cpp
│     ├─ GGYGOBossDefinition.h/.cpp
│     └─ GGYGOBossActionSet.h/.cpp
└─ AbilitySystem/Abilities/
   └─ GGYGOCombatActionAbility.h/.cpp
```

资产建议：

```text
Content/AI/Boss/Common/
Content/AI/Boss/<BossId>/
Content/Characters/Boss/<BossId>/<FormId>/
Content/Abilities/Boss/<BossId>/
```

---

## 13. 实施阶段与验收

### 阶段 A：只抽宿主，不改行为

- 新建 `AGGYGOCombatantState`；
- `AGGYGOCharacterSlot` 迁移到基类；
- 玩家队伍装配、切人、输入、属性复制行为保持不变；
- 收口 Avatar 唯一写入点。

验收：现有玩家单机流程无回归；专用服务器下预测键、属性、Cue 与切人正常。

**实施进度（2026-09-17）**：

- [x] 新建 `AGGYGOCombatantState`，持有 ASC、基础属性集与复制 Avatar；
- [x] `AGGYGOCharacterSlot` 迁移到基类，Mixed 复制、PawnData、能力授予和玩家 Owner 保持在 Slot；
- [x] GameMode 改为只调用 `CombatantState::AttachAvatar`，删除生成方对 PawnExtension 的直接写入；
- [x] PawnExtension 的幂等判断同时核对 ASC、Owner 与 Avatar；
- [x] Editor Target 与独立 Game Target 编译通过；
- [x] 无界面单机验证默认地图、GameMode、玩家 Slot 与出战 Pawn 装配；
- [ ] 交互式 PIE 验证输入、属性与 Cue；
- [ ] Dedicated Server 验证预测键、属性复制与切人。

### 阶段 B：Boss 最小装配

- BossState + BossCharacter + BossAIController；
- BossDefinition 只配一个 Form、一个 Phase；
- Minimal ASC 复制；
- 复用 Health、CMC、死亡链。

验收：Boss 能生成、被 AIController Possess、客户端看见血量/Tag/Cue、死亡只触发一次。

**实施进度（2026-09-17）**：

- [x] `UGGYGOBossDefinition`：Form/Phase、持久 AbilitySet 与 BehaviorTree 静态定义；
- [x] `AGGYGOBossState`：Minimal ASC、Form/Phase 复制、全部形态能力去重后一次授予；
- [x] `AGGYGOBossCharacter`：复用 Health/CMC/PawnExtension 的纯 Avatar；
- [x] `AGGYGOBossAIController`：显式 Possess，换 Avatar 时暂停但不清理 Brain；
- [x] `AGGYGOBossEncounter`：当前按 State → Controller → Deferred Avatar 的顺序生成，再AttachAvatar/FinishSpawning/Possess并核对装配；14b回收增量完整构建与两项专项自动化通过，真实BT/PIE/联机仍待验；
- [x] Avatar 销毁时 `CombatantState` 自动 Detach，死亡后不残留旧 Avatar；
- [x] Editor Target 与独立 Game Target 编译通过；
- [x] 创建单 Form/单 Phase 的 `DA_Boss_Test`、`DA_Pawn_Boss_Test` 与 `BP_Boss_Test`；
- [x] 建立独立 `L_BossAI_Test` 测试图，不污染主场景；
- [x] 单机 PIE 验证生成与 AIController Possess，装配日志确认 Form/Phase Tag 与 `Health=100/100`；
- [ ] 单机补验 GameplayCue 实际播放与死亡只触发一次；
- [ ] Dedicated Server 验证客户端可见性与 Minimal 复制。

### 阶段 C：一条完整攻击竖切

- `UGGYGOCombatActionAbility`；
- 一个近战 GA；
- ChooseAction + ActivateAbility 两个 BT Task；
- Montage Event → Trace → DamageExecution → Cue。

验收：AI 只请求能力；停用 BT 后不会有攻击；GA 单独激活仍能完成同一动作。

**实施进度（2026-09-17）**：

- [x] 新建 `UGGYGOCombatActionAbility`，用 `BossAction.*` 语义标签连接决策与执行；
- [x] 新建 `UGGYGOBossActionSet`，只保存候选条件和动态权重参数，不复制冷却、伤害或 Montage 配置；
- [x] `ChooseBossAction` 先按距离/角度/Tag/LOS 过滤，再调用 GAS `CanActivateAbility`，最后由 Encounter Seed 驱动确定性加权选择；
- [x] `ActivateBossAbility` 只调用 `TryActivateAbility` 并精确等待该 Spec 结束；BT Abort 只解绑等待，不越权取消已进入执行段的 GA；
- [x] `UGGYGOBossMeleeAbility` 串通 Montage → GameplayEvent NotifyState → MeleeTrace → `GE_Damage_SetByCaller` → `GameplayCue.Hit.Flesh`；
- [x] 编辑器命令 `GGYGO.BuildBossStageCTestAssets` 可重复生成 `BB_Boss_Test`、`BT_Boss_Test`、`AM_BossMelee_Test` 与命中窗口；
- [x] 创建 `BP_GA_BossMelee_Test`、`DA_BossActionSet_Test`、`DA_AbilitySet_Boss_Test` 并接入 `DA_Boss_Test`；
- [x] 测试图补专用地板，修复角色从出生高度持续掉出 KillZ、导致近战命中不可复现的问题；
- [x] 单机 PIE 实测：同一 Spec 循环完成“选招 → 激活 → Montage → 开窗 → 命中 → 伤害 → 关窗”，玩家生命 `100 → 0`；
- [x] 将 `GCN_BossHit_Flesh_Test` 放入配置扫描路径 `/Game/GameplayCues/Test/`；运行时 `GameplayCue.PrintGameplayCueNotifyMap` 确认 `GameplayCue.Hit.Flesh -> 1`；
- [x] Editor Target 编译通过；
- [ ] 组装一条可见/可听的命中 Cue 表现资产（当前已发 Cue，但测试 Notify 尚未配置 Niagara/音效）；
- [ ] 专项自动化验证“暂停 Brain 后不再发起新攻击、已激活 GA 能自行收尾”；
- [ ] Dedicated Server 验证能力、伤害与 Cue 的权威/复制链。

### 整改增量：14a / 14b（2026-09-30）

- [x] 14a E10/E11：全集配置校验、候选耗尽恢复、一次性来源/原Spec身份契约已冻结；统筹完整构建与三项BossSelection自动化通过。
- [ ] 14a真实BT→GAS、黑板观察者重入、生产资产与PIE集成验收。
- [x] 14b E9：显式创建记录、生成失败/EndPlay共用幂等回收及回调重入防护源码已冻结。
- [x] 两项Encounter测试源码及结构/主流程图文已冻结：`GGYGO.BossAI.Encounter.CleanupLifecycle`、`GGYGO.BossAI.Encounter.ExplicitCreationOwnership`。
- [x] 14b UHT、完整GGYGOEditor构建与上述两项自动化通过；真实夹具断言覆盖成功/重复生成、部分失败/重试、装配与清理重入、提前失效、Owner变化及外部Avatar保留，不扩大为实际BT或PIE/联机结论。
- [ ] 实际BT SafeStop/异步Abort及BT→GAS结束顺序；可观测测试Brain只观察StopLogic调用入口，不验证异步停止完成。
- [ ] PIE/世界卸载的Encounter EndPlay派发；当前无BeginPlay夹具显式调用真实EndPlay，不以Destroy替代此门禁。
- [ ] 联机/专用服务器的Encounter生命周期与复制验证。

14b的调用顺序为清引用/记录 → StopLogic → UnPossess → DetachAvatar → 只销毁创建对象，详见[[GGYGO_流程_BossAI.canvas|主流程]]及[[BossAI/结构|接口职责]]。第13次门禁证据：项目`Saved/Logs/ModuleRepairBuildGate_20260930_13.log`完整构建Succeeded、25.49秒、UHT写入12份生成文件、6 actions；`Saved/AutomationReports/ModuleRepairGate_20260930_13/index.json`项目47/47、succeededWithWarnings/failed/notRun均0、totalDuration为0.472851783秒，两项Encounter各Success且errors/warnings0，UE进程exit0由统筹报告。旧RootMotionBake预期拒绝与启动诊断不视为项目测试失败。项目详细记录见`AAADocs/Module_Repair_14b_Validation.md`；E9源码/专项自动化通过，完整动态验收仍有上列缺口。两张Canvas已按独立租约同步完整构建/两项专项通过并再冻结；真实BT/PIE/联机仍未验，最新门禁以本节和验证记录为准。完整形态切换、目标/仇恨、Kevin接线保持后续范围。

### 阶段 D：目标、仇恨与移动

- Perception → Threat → CombatTarget；
- MoveTo/PathFollowing；
- 目标死亡、丢失、多人仇恨切换；
- 脱战回场。

验收：无逐帧全场扫描；换 Avatar 后目标和仇恨保留。

### 阶段 E：阶段与形态切换

- 两个 Form、两个 Phase；
- Exit → Handoff → Enter；
- RetireAvatar；
- 失败回滚与晚加入客户端恢复。

验收：切换前后 HP、Buff、Debuff、冷却和 AbilitySpecHandle 保留；旧 Pawn 不触发死亡/掉落；
AIController 与目标不重建；客户端最终只看到一个有效 Avatar。

### 阶段 F：反应与数据化扩展

- ReactionRequest；
- 动态权重与反重复；
- 远程、召唤、回中心、受击反应；
- 决策日志与自动化测试。

---

## 14. 必测场景

| 场景 | 预期 |
|---|---|
| 阶段阈值被一次巨额伤害跨过 | 只进入一次正确的新阶段 |
| 治疗回阈值上方后再次跌破 | 不重复已完成的阶段切换 |
| Exit 动画中受到致死伤害 | Death 抢占，取消 Handoff，只走一次死亡链 |
| 新形态 Spawn 失败 | 回滚到旧形态，Boss 不消失、不变成无 Avatar 活状态 |
| 换形态前有冷却和 Debuff | 新 Pawn 上继续计时 |
| 换形态时目标正在移动 | Controller/Threat 保留目标，Enter 完成后重新导航 |
| 客户端在第二形态中途加入 | 直接恢复正确 Pawn、Phase/Form Tag、生命和 Cue 状态 |
| BT 请求冷却中的技能 | GAS 拒绝，BT 重新选择，不建立第二份冷却计时 |
| 当前目标在 GA 中途死亡 | GA 按自身规则结束/改目标；BT 下一轮重新选择 |
| Dedicated Server + 两个玩家 | Boss 决策只执行一次，伤害与阶段权威一致 |

---

## 15. 审阅后冻结的检查项

在改代码前确认以下条目：

- [x] 接受 `CombatantState` 只抽 Owner/Avatar/ASC，不包含 AI 与 PawnData；
- [x] 接受玩家 Slot、BossState、简单怪物 Pawn-owned ASC 三种布局并存；
- [x] 接受 BehaviorTree 只产出请求，GAS 执行动作；
- [x] 接受 Phase 与 Form 分离；
- [x] 接受 Boss 能力一次性授予、Tag 门控；
- [x] 接受 Exit 与 Enter 是两个 Ability，中间才发生 Avatar Handoff；
- [x] 接受换形态不走死亡流程；
- [x] 接受冷却只保存在 GAS；
- [x] 接受 AIController/Threat 在换形态期间保持实例；
- [x] 接受第一版先做一只两形态 Boss 的完整竖切，再扩展通用性。

本轮“开始计划”视为以上架构项确认；若后续改变任一项，先更新本节再继续代码阶段。
