# AbilitySystem 计划

> 返回 [[计划蓝图]] · 当前代码事实见 [[模块参考]]

> 当前：ASC两项清理来源及Host H1/H2已有限静态接受冻结；未成功整链编译。Build55为 Failed (OtherCompilationError)，UHT通过并写33个生成文件，新运行时未链接、未UE冒烟，构建中353源/45保护保持。Character Optional路径、Task委托拼写、RMS定义include三项机械修正已原组长冻结并根接受；Hero旧三调用尚未迁移。Input混合注入、GA、Camera真实取舍待用户选择，仅相关线等待。

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

## 当前Avatar生命周期增量

### 本轮局部图文预检（N2，2026-10-03）

- 唯一目标：同步 ASC 已冻结的已提交 cleanup 来源与失败 Init 原写入证明两个不同契约，以及本次读取时 Host 消费状态；不把局部来源完成写成生产链或动态验收完成。
- 精确四文件（本目录）：`结构.md`、`计划_AbilitySystem.md`、`GGYGO_结构_AbilitySystem.canvas`、`GGYGO_流程_AbilitySystem.canvas`。AbilitySystem 长期组长直接写入，gpt-6.1-sol / xhigh；无子代理。
- 只读依赖：已冻结 ASC h/cpp、共享 Types、两项源码统筹验收及 N2 租约基线（`F:/ue_project/GGYGO/Saved/ValidationRecords/ASCCleanupDocumentation_20261003_LeaseBefore.json`）；Host H1验收/H2租约与实际消费者、计划蓝图/模块参考、Combatants/Character 结构。Host 未冻结新内容只读，不替其实施。
- 顺序与原子结果：已冻结 ASC 契约 → 核对 Host H1/H2 当前事实 → 两 Markdown 同步职责/输入输出/缺口 → 两既有 Canvas 展开实际来源与关键分支 → JSON/ID/端点/标签/几何/链接及保存回读。四文件共同说明同一清理来源契约，没有第二状态/执行生命周期。
- 非目标：源码、项目记录、全局入口/模块参考、相邻模块、第五文档/子图、B0/输入算法、资产、Build/UHT/UE/Git。保留 K3/GA 实质政策待决及全部历史失败/未验边界。
- 验收断言：两 ASC 步骤仅 `finite_static_accepted_uncompiled`；cleanup不授工作/Ready，失败 Init cleanup不commit/Receipt/Notice且原失败保留；旧ID/锚点与可复用布局保持，无重叠/悬空边/空标签，新链接可定位；本轮仅统筹统一编译＋必要 UE 冒烟，不扩严格矩阵。
- 停止点：四文件保存回读、hash交回并冻结；没有 Obsidian UI 验收。需第五文件或新业务取舍时具体交回，不自行扩权。

ActorInfo三Try／一次发布、历史C1a Gate47～48与C1b Gate49R1有编译及有限专项；这些原结果继续保留，成功仅为各自真实同步返回，不证明全GA/Cue/网络结束。当前新增的两种ASC清理来源契约均已统筹接受／冻结，状态仅 `finite_static_accepted_uncompiled`，不沿用旧专项作为新证据。

| 原子交付 | 当前事实 | 下一边界 |
| --- | --- | --- |
| 原已提交cleanup | CheckAvatarBindingCleanupContext纯GT核对原Current/Revoked Context、LastWrite/Issuer/allocation/完整快照；撤销保留来源，真实typed/legacy写前退休；Cancel/Cue/Clear复用原native窗口 | 新工作仍Current/live；关闭Owner/宿主明确拒绝PreserveOwner，调用方显式ClearActorInfo，无错误后换模式。 |
| 失败Init原写入Proof | 完整native Init返回后、未commit失败保留不可变原WrittenActual；TryCleanupFailedAvatarActorInfoInit消费原Operation＋必需同步纯原scope查询，先消费再唯一Clear并核真实GAS后置条件 | 清理不重捕来源、回滚/排队/重试；Succeeded不commit/Receipt/Notice/Ready，原Init仍失败。 |
| Host H1消费 | 有限静态接受／冻结，清理分支已调用cleanup查询并显式选择Clear模式 | 未成功整链编译／UE；普通Ready/Released合法后继不被全局Busy挡住。 |
| Host H2消费 | 已根有限静态接受冻结；Initialize及owner-only失败分支消费原Operation/pure原scope，原Init失败保留 | 与H1均只为源码有限接受，不称整链运行完成。 |
| 生产／验证 | Health/Base已消费原H，Hero仍未迁移，整链未闭合 | Build55已统一编译：UHT通过33生成文件，最终Failed (OtherCompilationError)，新运行时未链接／未UE；353源45保护保持。三项机械修正已冻结接受，Hero旧三调用未迁移；不扩严格矩阵，R0历史失败及原断言保留。 |

两个ASC源码收尾的技术方案与接口已实施，不再记“待用户决定/尚未实现”。完整来源契约及作用域、寿命/释放责任见[[AbilitySystem/结构#原已提交清理来源（Destroy）|提交来源]]、[[AbilitySystem/结构#失败Init原写入清理（未提交证明）|失败Init来源]]及[[AbilitySystem/结构#清理调用方与本轮验证边界|调用方/门禁]]。K3/GA实质政策、资产与联机未验边界独立保留。

## 当前原请求输入增量（07E2：已编码、未编译）

ASC已替换旧Tag生产入口为Receive(Tag,PreviousIdentity,OriginalDeadline)、End(Identity,Released/Invalidated)、Queue(const OriginalRetry&)；OnAbilityInputRetryable改为const完整单载荷。原weak ASC／revision／serial由ASC签发，完整Tag／ID／原绝对deadline精确匹配，不用-1补窗口。唯一Spec held／queued聚合、有序边沿及按边沿冻结来源已编码，final Can／Notify继续只消费本次来源。

首按真实来源与原deadline由Input／Hero提供；多来源任一held则Spec保持，最后真实释放才Released，失效用Invalidated。deadline到期不伪松键；Query/raw不借来源。07E2源码有限接受不等于已编译：Hero A/B与Action观察端口尚未迁移；两诊断仅const完整单载荷签名适配已冻结，旧-1断言／运行来源未迁移。Input内部raw与Movement恢复义务已分离，Action物理来源此前已获原则授权；原观察端口未接；混合注入真实技术方案待用户选择，仅相关线等待。

接力：已授权来源合同的技术接缝→Input原观察端口→Hero原ID消费A／typed H与Source→CMC装配B→核已冻结两诊断签名与未迁移运行合同→全链冻结后统一编译与必要UE冒烟。原Gate41仅旧B0历史，不扩大严格矩阵、不撤失败场景。ASC精确Montage归属首步现已源码实现、未成功整链编译；Task消费者已源码接入冻结；GA统一取消／结束未实施，不能关闭生产根因。详见[[AbilitySystem/结构#单次 Can 评估与输入失败来源（B0）|实际接口与重要实现]]。

## 当前精确Montage增量（K3）

已按用户选择实现项目ASC精确归属入口TryPlay／Check／Capture／TryClear，不改UE／GAS库。完整Super／Guard接受且真实Local写入与来源一致才签不透明Handle；唯一provenance不存进度、播放活动或第二执行状态。停止不抹归属，清空保留原混出时机，仅清经认证的原GA／Local字段，不Stop或延期。

| 阶段 | 当前事实／剩余交付 |
| --- | --- |
| ASC原播放接口 | 四文件已交回冻结、统筹有限静态接受，未成功整链编译／UE；同GA同资产A／B按原非复用Guard调用及ASC私有Proof区别。 |
| GA原资源捕获／统一终止 | 精确只读预检已交回，原终止身份／资源／Busy／外层退出候选已明确；非虚原生激活的自动Retrigger缺完整退出钩子，受控入口取舍待用户决定。PlayerCombo／BossMelee迁薄扩展点仍未改源，不冒称方案已实施。 |
| Task原资源消费者 | h/cpp已交回冻结、统筹全文有限静态接受（52EC85F0…／35F04002…，八依赖保持）。消费ASC单一Result／原Handle／Guard，删除全局尝试表／token／新实例扫描；仅自身委托／缩放lease及栈共持原停止义务。原混出精确Clear；Completed只为原实例Ended事实，不Check清空后当前ASC、不授End后继GA权力。未成功整链编译／UE，旧nonGuard／after-Super／手工ID夹具未适配，原严格失败未复测；GA业务原生命周期认证另步未接。 |
| 验证与生产 | 全链冻结后统一编译＋必要UE冒烟，不扩严格矩阵；原失败复现／断言保留，未运行不标通过。正式AnimClass／资产、网络与完整K3仍开放。 |

Init／Clear的真实Local重置退休原证明；同端点Refresh仅来源实际改变才退休，不用Publication修订猜播放变化。裸限定基类写入与未观察Owner ABA不作完整保障声明；销毁提交来源／Init失败写入Proof两个ASC源码步骤现已实现并有限静态接受冻结，Host H1已消费、H2已有限静态接受冻结，整链仍未成功整链编译／冒烟；它们与K3/GA终止政策分责，不从K3扩权。具体接口、重要实现和跨模块边界见 [[AbilitySystem/结构#精确Montage播放归属（K3：ASC与Task源码冻结，GA未迁移）|接口契约]]、[[Animation/结构|Guard]]。

GA终止的本轮精确预检已零写入交回：原生Cancel在虚End前广播；原生End在NotifyAbilityEnded之后仍写CurrentEventData；TryActivate／InternalTryActivate／CallActivate非虚，不能由项目四源完整截获所有原生父调用退出。候选由GA持原激活与唯一终止资源、ASC持真实受控激活作用域和完成通知，业务清理迁薄扩展点；这些新合同仍未实施。受控项目入口／拒绝同调用自动Retrigger、显式End→带原来源Completed→新请求的可见行为已提请用户决策；不得将候选签名记为源码存在，不改UE／GAS库，不自动排队或用延迟通知掩盖退出缺口。Task消费者不依赖这项新终止接口，已按独立两源租约接入精确播放Result并冻结；GA业务结束／接续的原生命周期认证仍是后续责任。

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
| `SingleInstanceQueued` | 组内同时只能一个，被拒绝的激活由调用方缓冲重试 | 独立攻击实例之间的准入；玩家 GA 内段序由窗口与单请求缓存管理 |

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
   - `SingleInstance` → 不可取消旧实例直接阻断；可取消时比较优先级，平手按 `bNewcomerWinsOnTie`。查询只判断，不持有另一份取消队列。
   - `SingleInstanceQueued` → 若组内已有实例 → 拒绝激活并返回可重试原因；意图缓冲归 HeroComponent。
4. **Tag 关系检查**：交给照抄来的 `GGYGOAbilityTagRelationshipMapping`，处理与组无关的 Tag 级 Block/Cancel

`NotifyAbilityActivated` 阶段（改状态）：

5. `NotifyAbilityActivated` 开始本实例的准入尝试，只把新实例登记为 `ActiveAbilitiesByGroup` 预留，再让 GAS 广播激活通知；此时不能取消或 End，因为本次 Spec ActiveCount 尚未递增。
6. GAS 完成 ActiveCount 递增后进入项目基类 `ActivateAbility`。若当前尝试已被同步递归中的更新尝试标记拒绝，立即跳过最终裁决，不能回头取消胜出的更新实例。
7. 未被预先拒绝的尝试调用 `ASC::FinalizeAbilityGroupAdmission(Ability, AdmissionSequence)`。ASC 先按原同组优先级/Exclusive 冲突谓词比较 pending 的全局 sequence：较新尝试拒绝较旧尝试，较旧 resolver 恢复后识别自身已拒绝并退出；settled 冲突再走原取消，并只从 `ActiveAbilitiesByGroup` 权威登记复核同步回调后的占用，不能把已注销登记但仍在 GAS 结束广播栈中的 Spec 实例当成竞争者。返回后基类按捕获 sequence 消费拒绝并完成该 attempt，失败则在相机/BP 业务前安全结束。sequence 由 ASC 跨实例分配，独立于 PredictionKey 和 Spec 总 ActiveCount。

`OnAbilityEnded` 阶段：

8. 项目 ASC 在 Super `NotifyAbilityEnded` 前摘除旧组登记，但暂不广播组空。
9. Super 减少 ActiveCount 并广播结束；允许回调重新登记新激活。返回后仅当组仍为空才发 `OnAbilityGroupFreed`，HeroComponent 据此处理自己的缓冲。

**D4 对应 `SingleInstance` 的 `bNewcomerWinsOnTie=true` 默认值：可取消旧实例的平手由后来者取代；不可取消旧实例仍保留槽位。** `UncancelableActive` 使用现有通用组失败 Tag，不归类为 Queued 重试。

最终完整 C++ 构建成功。`GGYGO.AbilitySystem.Admission.GroupLifecycle` 单项 1/1 通过，随后统一自动化 38/38 全部通过，0 warning、0 error。单项诊断确认 C sequence 18 正确拒绝 B sequence 17；旧 post-cancel 曾扫描全部 Spec 实例并误命中已从 `ActiveAbilitiesByGroup` 注销、但仍处于 GAS 结束广播栈的 A sequence 0。终检改为只遍历权威组登记后，C 保持活跃、B 安全退出，唯一实例/ActiveCount/组登记严格断言通过。相机资源修复属于第05批。

回归入口：`GGYGO.AbilitySystem.Admission.GroupLifecycle` 覆盖组规则与取消/结束/组空重入及同 Spec PerExecution 计数，严格断言“旧 pending 尝试退出后只留一个活动实例”已通过；`GGYGO.AbilitySystem.Admission.InvalidPredictionKeyReentry` 和 `GGYGO.AbilitySystem.Admission.CorrectionRpc` 同在最终 38/38 回归中通过。本地测试不代替双端 RPC 验收。Montage Task 与 Combo 专项见 [[计划_玩家普攻连段]]。

玩家普攻采用一个 GA 内管理多段：同 Priority 100，NotifyState 控制窗口，GA 消费单请求缓冲并切换逐段 Main→End Montage。`SingleInstanceQueued` 只约束其他攻击实例；它不负责内部段序或动画窗口。具体分阶段实施和网络/清理契约见 [[计划_玩家普攻连段]]。

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
