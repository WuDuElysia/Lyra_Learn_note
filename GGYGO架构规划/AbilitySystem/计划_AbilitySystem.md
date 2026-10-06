# AbilitySystem 计划

> 返回 [[计划蓝图]] · 当前代码事实见 [[模块参考]]

> 历史 Build55：Failed (OtherCompilationError)，UHT通过并写33个生成文件，新运行时未链接、未UE冒烟，构建中353源/45保护保持；当时三项机械修正已冻结，Hero旧三调用尚未迁移。保留该失败记录，不作为当前源码状态。
>
> 上一检查点（2026-10-05）：原生原身份发行、final生命周期、Initialize/Body/Context Cleanup、Combo/Boss/Admission及Process/OnSpawn/Boss BT受控消费已落盘并编译。Gate79四RuntimeHit Case行为PASS，原四普通叶0E0W；七叶报告5 Success/2 Fail/0 Warning，故障叶只保留真实生产Error。当前有限验收及剩余入口见[[AbilitySystem/计划_原请求终止|原请求终止计划]]、[[GGYGO_流程_原请求终止.canvas|子图]]；严格红测、正式资产与网络不关闭。

> 当前（2026-10-06）：必需姿态 Ticket/typed OnFailed、Combo Main 动作请求与 End 原移动事实中断已接齐，统一 Editor 构建 Succeeded。Gate101-R2 六叶原报告为3 Success/3 Fail，原Error/Warning不滤掉；分项及有限 BodyZ 见 [[计划_玩家普攻连段#本轮编译与必要冒烟证据|集中证据]]。原生命周期/Busy契约不改变，严格红测、完整姿态混合、Cook/联机/HID继续开放，C12普通换人政策仍待用户决定。

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

> 以下保留N2交回时的范围、判断与停止点；其中未编译、未迁移和政策待决均为当时状态。当前事实以本节表格及T2增量为准。

- 唯一目标：同步 ASC 已冻结的已提交 cleanup 来源与失败 Init 原写入证明两个不同契约，以及本次读取时 Host 消费状态；不把局部来源完成写成生产链或动态验收完成。
- 精确四文件（本目录）：`结构.md`、`计划_AbilitySystem.md`、`GGYGO_结构_AbilitySystem.canvas`、`GGYGO_流程_AbilitySystem.canvas`。AbilitySystem 长期组长直接写入，gpt-6.1-sol / xhigh；无子代理。
- 只读依赖：已冻结 ASC h/cpp、共享 Types、两项源码统筹验收及 N2 租约基线（`F:/ue_project/GGYGO/Saved/ValidationRecords/ASCCleanupDocumentation_20261003_LeaseBefore.json`）；Host H1验收/H2租约与实际消费者、计划蓝图/模块参考、Combatants/Character 结构。Host 未冻结新内容只读，不替其实施。
- 顺序与原子结果：已冻结 ASC 契约 → 核对 Host H1/H2 当前事实 → 两 Markdown 同步职责/输入输出/缺口 → 两既有 Canvas 展开实际来源与关键分支 → JSON/ID/端点/标签/几何/链接及保存回读。四文件共同说明同一清理来源契约，没有第二状态/执行生命周期。
- 非目标：源码、项目记录、全局入口/模块参考、相邻模块、第五文档/子图、B0/输入算法、资产、Build/UHT/UE/Git。保留 K3/GA 实质政策待决及全部历史失败/未验边界。
- 验收断言：两 ASC 步骤仅 `finite_static_accepted_uncompiled`；cleanup不授工作/Ready，失败 Init cleanup不commit/Receipt/Notice且原失败保留；旧ID/锚点与可复用布局保持，无重叠/悬空边/空标签，新链接可定位；本轮仅统筹统一编译＋必要 UE 冒烟，不扩严格矩阵。
- 停止点：四文件保存回读、hash交回并冻结；没有 Obsidian UI 验收。需第五文件或新业务取舍时具体交回，不自行扩权。

ActorInfo三Try／一次发布、历史C1a Gate47～48与C1b Gate49R1有编译及有限专项；成功仅为各自真实同步返回，不证明全GA/Cue/网络结束。两种ASC清理来源及Host H1/H2现已实施并编译，不能沿用旧专项或Gate66既有冒烟作为新增清理场景的动态证据。

| 原子交付 | 当前事实 | 下一边界 |
| --- | --- | --- |
| 原已提交cleanup | CheckAvatarBindingCleanupContext纯GT核对原Current/Revoked Context、LastWrite/Issuer/allocation/完整快照；撤销保留来源，真实typed/legacy写前退休；Cancel/Cue/Clear复用原native窗口 | 新工作仍Current/live；关闭Owner/宿主明确拒绝PreserveOwner，调用方显式ClearActorInfo，无错误后换模式。 |
| 失败Init原写入Proof | 完整native Init返回后、未commit失败保留不可变原WrittenActual；TryCleanupFailedAvatarActorInfoInit消费原Operation＋必需同步纯原scope查询，先消费再唯一Clear并核真实GAS后置条件 | 清理不重捕来源、回滚/排队/重试；Succeeded不commit/Receipt/Notice/Ready，原Init仍失败。 |
| Host H1消费 | 已实施并编译：清理分支调用cleanup查询，按捕获的关闭事实显式选择Clear模式 | 原提交Destroy／完整换绑动态验收仍开放；普通Ready/Released合法后继不被全局Busy挡住。 |
| Host H2消费 | 已实施并编译：Initialize及owner-only失败分支消费原Operation/pure原scope，原Init失败保留 | 编译不证明失败Init完整生产清理；两种来源分别保留动态边界。 |
| 生产／验证 | Health/Base已消费原H；Hero A已接原输入ID/完整Retry，B1已接typed本地H，B2已接Source→CMC装配；Gate59及后续编译已覆盖 | 历史Build55失败及R0 Gate42的2 Fail／12错误保留；未重跑的原失败不能标关闭。Gate66既有三冒烟通过不替代Host／Extension整链、资产或联机验收。 |

两个ASC清理来源、Host H1/H2与Hero消费已实施并编译，来源职责仍见[[AbilitySystem/结构#原已提交清理来源（Destroy）|提交来源]]、[[AbilitySystem/结构#失败Init原写入清理（未提交证明）|失败Init来源]]及[[AbilitySystem/结构#清理调用方与本轮验证边界|调用方门禁]]。GA/派生共同生命周期与受控生产请求点现已接齐，普通/故障必要冒烟有限验收；Avatar完整换绑/Destroy与原严格失败、正式资产/联机仍独立留账。

## 当前原请求输入增量（07E2：已接Hero并编译，动态边界保留）

ASC已替换旧Tag生产入口为Receive(Tag,PreviousIdentity,OriginalDeadline)、End(Identity,Released/Invalidated)、Queue(const OriginalRetry&)；OnAbilityInputRetryable改为const完整单载荷，Hero A已实际消费并编译。原weak ASC／revision／serial由ASC签发，完整Tag／ID／原绝对deadline精确匹配，不用-1补窗口。唯一Spec held／queued聚合、有序边沿及按边沿冻结来源已实现，final Can／Notify只消费本次来源。

首按Action来源与原deadline由Input／Hero提供；多来源任一held则Spec保持，最后真实释放才Released，失效用Invalidated。deadline到期不伪松键；Query/raw不借来源。Hero A已接EnhancedInput ActionInstance的Triggered/Completed/Canceled，保存原ID并核原Tag/deadline/Binding后Queue完整Retry；局部失效只End自己的ID。Hero B1已注册并回放typed本地H通知，Released只退原会话；B2已装配Source→CMC原移动输入会话。旧-1断言/运行夹具、真实混合来源及资产/联机未由编译证明，不把Hero迁移写成这些专项已通过。

当前消费：Process在原InputScope内一次受控Try→bNativeAccepted消费，Origin/Can/Retryable规则保持；OnSpawn和Boss BT也已接一次受控入口并编译，不raw重试，remote接受不是本地Activation证明。Combo/Boss/Admission已迁薄扩展点及原资源Context Cleanup；Gate79普通链有限通过。原B0/输入合同与新的原激活来源分责，RPC/网络、混合来源及原strict/raw失败仍开放。见[[AbilitySystem/结构#单次 Can 评估与输入失败来源（B0）|输入接口]]及[[AbilitySystem/计划_原请求终止|终止契约]]。

## 当前精确Montage增量（K3）

已按用户选择实现项目ASC精确归属入口TryPlay／Check／Capture／TryClear，不改UE／GAS库。完整Super／Guard接受且真实Local写入与来源一致才签不透明Handle；唯一provenance不存进度、播放活动或第二执行状态。停止不抹归属，清空保留原混出时机，仅清经认证的原GA／Local字段，不Stop或延期。

| 阶段 | 当前事实／剩余交付 |
| --- | --- |
| ASC原播放接口 | 已实施、冻结并编译；同GA同资产A／B按原非复用Guard调用及ASC私有Proof区别。正式资产/联机与原严格失败动态边界保留。 |
| GA原资源捕获／统一终止 | 已共同启用并编译：实际NotifyActivated签Original，受控Try记录外层返回来源；final Activate/End/Cancel管理实际调用跨度，Initialize→原准入/相机→Body，一次Context Cleanup→native End。GA唯一原记录，ASC仅见证并发布；Combo/Boss/Admission已迁。raw外层缺证据与PendingRemove/锁内teardown仍明确失败，不补Completed。 |
| Task原资源消费者 | 精确ASC Result/Handle/Guard、原五native单播及SectionReceived/OnFailed已编译；Combo/Boss在Ready前注册原包，闭包持Original/原Task，Context收尾精确脱token。Task唯一持播放/停止/scale/监听，无全局尝试表或新实例扫描。Task自然Completed不是GA原Notice；native/捕获析构极端重入、旧夹具与资产网络仍未专项验收。 |
| 验证与生产 | Gate79有限正常/故障链已验：四RuntimeHit Case行为PASS，原Combo纠正/Boss Mesh/输入/移动普通叶0E0W。原报告5 Success/2 Fail/0 Warning，两故障仅有三条真实生产Error，不过滤；正式Boss战斗、BP/Montage接线、Execution数值、Cue表现、网络与原strict/raw红仍开放。历史构建/错误见既有模块验证记录。 |

Init／Clear的真实Local重置退休原证明；同端点Refresh仅来源实际改变才退休，不用Publication修订猜播放变化。裸限定基类写入与未观察Owner ABA不作完整保障声明；销毁提交来源／Init失败写入Proof与Host H1/H2、Hero A/B1/B2已实施并编译，完整生产清理动态验收另列。它们与K3/GA终止分责。接口与跨模块边界见[[AbilitySystem/结构#精确Montage播放归属（K3：已编译，派生与生产未闭合）|播放契约]]、[[Animation/结构|Guard]]。

T2现有接口：ASC `TryActivateAbilityWithTerminationBoundary`提供真实Try完整退出见证，`OnAbilityTerminationCompleted`发布带原来源的完成历史；GA `CaptureCurrentActivation`读取既有原身份，`RequestAbilityEnd/Cancel`受理原请求，`CleanupAbilityResourcesForTermination`只清自身原资源。GA持唯一首个不可变Context与原终止记录；ASC关联原Try/Ended见证，不持第二终止状态机。原生Cancel广播前捕资源；原生End、完整虚End/Cancel及关联Try全部退出后先封存结果、释放原Busy，再通知，回调后继不能覆写原结果。原生WaitingToExecute仅一次Continuation；解锁发生在原虚调用内时留Ready，完整返回后消费。

受控项目入口与显式End→原Completed→新请求政策已共同启用；Activate/End/Cancel为final，派生通过Initialize/Body/Context Cleanup保持业务。原生NotifyActivated签身份和受控Try提供外层来源是两项不同证明；raw非虚Try/CallActivate缺后者，原清理不冒充协议Completed。活动同实例或原Busy拒绝重开，PerExecution不同实例不按同Spec一概封禁；无自动重启/第二队列。详细结果与边界见[[AbilitySystem/计划_原请求终止|子计划]]、[[GGYGO_流程_原请求终止.canvas|子图]]。

## 必需姿态契约与Main/End集成

本轮完整链由 Animation 提供姿态能力与固定动作曲线来源，Task 管原播放/订阅/Ticket，Combo 管段与原资源生命周期，CMC 唯一执行胶囊 XYZ 位移并认证原移动请求。ASC仍只提供原播放与原终止证明，不解析角色动作或复制姿态执行。

| 边界 | 已实现契约 | 清理/验收边界 |
| --- | --- | --- |
| Guard → Task | Activate实际播放前Acquire；ExplicitNotRequired普通模式，RequiredReady原Ticket；仅Required在引擎Task Tick只读Poll，Failed/Invalidated明确失败 | 不每帧Acquire，不消费最终混合曲线，不另建Tick/播放时钟；Ticket由OnDestroy退休 |
| Task → Combo | pre-Ready一包注册SectionReceived与typed OnFailed；原Section快照/事实取自确切实例，失败先精确Stop/EndTask再历史分发 | Startup NONE / Playback原ID；native垃圾标记后的Get(true)只用于历史发送者核验，GA另核自身原身份；native结束原激活则不发旧BP |
| Combo → CMC | 原Snapshot固定ID/Section/Position/Rate；Main调用BeginMontageActionMotion并观察原失败；SectionReceived进入End释放Main，再Query/Subscribe原QualifiedMovementIntent | 只消费原Scope及复核一致的Intent；CancelMontageActionMotionForMovement后RequestAbilityEnd(Original,true,true)。不可用/执行失败明确结束原动作；普通Interrupted/Cancel不换语义 |
| 生产接线/必要门禁 | GA三段字段冷读回；ABP_Pyrios Required FullBody ActionPoseSlot保存回读；Gate101-R2 End两叶、XYZ叶Success，姿态两故障behavior PASS但各2Error/Fail，恢复负例3Error/Fail | BodyZ只Normal01 Main；End姿态/完整混合、真实HID、完整三段命中表现、Cook/网络及strict/raw仍未验 |

实际接口输入/输出见 [[AbilitySystem/结构#Task原实例与资源消费（已编译，专项与迁移边界保留）|Task契约]]；Main/End/失败资源流与唯一集中证据见 [[计划_玩家普攻连段]]。姿态执行见 [[Animation/动作姿态修正|Animation契约]]，胶囊执行见 [[Movement/结构|Movement职责]]。Gate101/R1实现失败经本轮修正和R2复测关闭，原失败报告保留；不把负例报告Fail改成全通过。

C12普通换人退出：真实业务政策尚未决定，候选共享接口只读配对冻结、生产未实施；本轮没有扩大到队伍切换或修改已确认的原生命周期/Busy归属。

## 当前连段输入Task增量（W0）

W0与Combo native输入消费者已落盘并编译。固定Original/原Task/原订阅贯穿创建、Ready、OnPress及Context Cleanup；Gate79正常纠正与必要资源收尾有限通过。Gate69/70旧失败保留在模块验证记录，native捕获析构重入/remote同步回放/正式输入资产与网络仍未由本次普通链关闭。

| 范围 | 当前实现与剩余边界 |
| --- | --- |
| 原输入资源 | WaitComboInput首次Activate捕原弱ASC/Ability、Spec/key、remote/predict模式及自身handle；GAS事件桶与监听唯一归属保持。零key合法且只作桶定位，原Activation仅在调用方闭包；工厂、SetSourceStep、OnPress BP与65535整数载荷/预测发送规则保持。 |
| native与清理契约 | 可选pre-Ready单播RegisterNativeCallback返回精确FDelegateHandle，UnregisterNativeCallback只按原token先脱后释放；空/重复/迟到明确拒绝。原Consume→native/捕获析构返回→弱原Task/原订阅核验→BP；同步回放结束不能写Waiting。OnDestroy先关/脱资源，再原桶remove自身handle，不消费后继事件，捕获释放后无旧Task写尾。 |
| 验收／迁移 | Combo已在原Task Ready前注册native回调，精确token与原桶资源已随Context清理；Gate79有限纠正及原Task资源断言通过。未单独运行native/析构重入或同步remote回放专项，正式资产和网络继续开放；不新增严格矩阵。 |

实际接口输入/作用/输出、清理与失败诊断见[[AbilitySystem/结构#原事件桶与输入Task订阅（W0）|W0契约]]，节点位于[[GGYGO_结构_AbilitySystem.canvas|结构图]]及[[GGYGO_流程_AbilitySystem.canvas|流程图]]。

## 当前GA激活接缝与OnSpawn入口

原准备接缝现已与原身份发行、final生命周期及派生迁移共同启用并编译。以下记录当前职责；旧准备阶段及Gate70不能作为新动态证据，实际有限验证见原请求终止计划。

| 范围 | 当前事实与后继门禁 |
| --- | --- |
| 激活薄hook | Initialize在组终检/相机/Body前一次调用，初始化原批次；外调后原来源结束、Busy或改变则不进入旧Body。Body默认一次native Super→既有BP，派生只管招式资源，核心Activate为final。Original由实际NotifyActivated签发，Capture只读、不从Spec/key补造。 |
| OnSpawn请求点 | 保留原策略、Spec/Avatar及网络侧选择，live项目ASC在原请求位置一次既有受控Try；非法或非项目ASC诊断拒绝，无raw回退/新队列。NativeAccepted可仅为remote请求接受，不等于本地原激活、Commit或Completed。 |
| 共同启用／验收 | GA/ASC实际签发/见证与Combo/Boss/Admission扩展点共同启用；final End/Cancel统一一次Context Cleanup/native End、实际返回义务与ASC完成通知。Gate79必要正常/故障行为有限验收；raw/native严格、移除/极端重入、正式资产和网络仍开放。 |

真实接口与失败诊断见[[AbilitySystem/结构#激活薄hook与OnSpawn受控入口|结构契约]]；本节不复写终止子图或将计划写成已实现。

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
6. GAS完成ActiveCount递增后进入final项目Activate，先核实际Original并Initialize原批次；若其结束、Busy或来源改变，不执行旧Body。对原仍有效的attempt再做最终裁决，已被较新尝试拒绝者不能回头取消胜出实例。
7. 未被预先拒绝的尝试调用 `ASC::FinalizeAbilityGroupAdmission(Ability, AdmissionSequence)`。ASC 先按原同组优先级/Exclusive 冲突谓词比较 pending 的全局 sequence：较新尝试拒绝较旧尝试，较旧 resolver 恢复后识别自身已拒绝并退出；settled 冲突再走原取消，并只从 `ActiveAbilitiesByGroup` 权威登记复核同步回调后的占用，不能把已注销登记但仍在 GAS 结束广播栈中的 Spec 实例当成竞争者。返回后基类按捕获 sequence 消费拒绝并完成该 attempt，失败则在相机/BP 业务前安全结束。sequence 由 ASC 跨实例分配，独立于 PredictionKey 和 Spec 总 ActiveCount。

`OnAbilityEnded` 阶段：

8. 项目 ASC 在 Super `NotifyAbilityEnded` 前摘除旧组登记，但暂不广播组空。
9. Super 减少 ActiveCount 并广播结束；legacy/raw路由可发生重入登记，返回后仅当组仍为空才发 `OnAbilityGroupFreed`，Hero据此核原ID/Tag/deadline缓冲。T2受控路由的同实例原Busy须待全部关联调用退出后释放；组空通知本身不是原请求Completed，也不授新激活许可。

**D4 对应 `SingleInstance` 的 `bNewcomerWinsOnTie=true` 默认值：可取消旧实例的平手由后来者取代；不可取消旧实例仍保留槽位。** `UncancelableActive` 使用现有通用组失败 Tag，不归类为 Queued 重试。

历史组仲裁验收：当轮完整C++构建成功，`GGYGO.AbilitySystem.Admission.GroupLifecycle` 单项1/1，随后统一自动化38/38，0 warning、0 error。单项诊断确认C sequence 18正确拒绝B sequence 17；旧post-cancel扫描全部Spec误命中已注销但仍处GAS结束广播栈的A sequence 0。终检改为权威组登记后，C活跃、B安全退出，唯一实例/ActiveCount/组登记严格断言通过。保留原结果，不将其当作新T2协议动态证据；相机资源修复属于第05批。

历史回归入口：`GGYGO.AbilitySystem.Admission.GroupLifecycle`与旧38/38保留原当轮结果。共同生命周期启用后的Gate71原strict/raw复现仍红，旧立即重开期待与缺外层见证明确留账，不能据历史绿或Gate79普通链关红。当前Combo/Task有限生产与必要冒烟见[[计划_玩家普攻连段]]，双端RPC与联机另验。

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

### 当前Builder契约（E14-B/C：已编译，必要行为有限验收）

`BuildHitEffectPayload`保留bool接口：显式空DamageEffectClass是正常Context/Cue模式；非空必需GE须有有效Spec/Def/Context与原Ability/Source/Target。MakeOutgoingSpec、ApplyAbilityTags、BP_EditSpecValues及Cue初始化外调后重核原来源，只全部成功才赋输出；失败false/空输出并定位诊断，不降级无GE。Combo命中期间必需依赖或Builder失败直接RequestAbilityEnd(Original,true,true)，CanBeCanceled=false也不改为Cancel→End；Boss要求有效GE，失败保留拒绝该hit的响应。正常Spec应用免疫/拒绝仍可碰撞Cue。Gate79正常无GE/有效GE与两故障中止行为有限通过，真实Error/Fail及Shared资产失效/底层MakeOutgoingSpec失败/正式资产网络未验边界保留。

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
