# Kevin DemonBattle 战斗接入计划

> 本计划由 Combat 玩家动作/GA 集成（战斗模块）唯一维护；BossAI 与 Combat 查询图文分别由其长期模块作者维护，本文通过链接引用。当前 BossMelee 已消费受控 Original、原资源批次、Owned Trace 和原终止接口，Gate79 NormalLifecycle 在隔离原生夹具中 Success/0E0W；正式 Kevin 接线、动作实战与联机不在该结果内。动画模块维护动作分组/Montage/骨架/builder，根会话统一安排构建、UE 与最终验收。禁止把人工暂定判定写成原游戏还原。
> [[BossAI/结构]] · [[Combat/结构]] · [[GGYGO_结构_BossAI.canvas]] · [[GGYGO_流程_Combat.canvas]] · [[Animation/BH3_Kevin_DemonBattle导入]]

## 1. 已核对的来源与现状

- 来源目录：`F:/AnimeStudio/Exports/BH3/Animator/Kevin/05_BOSS_411_DemonBattle`，79 份动画 JSON。递归字段检查未发现事件、伤害、碰撞体、Socket、半径或判定窗口字段；75 份有 RootT/RootQ 七条轨迹，其余 4 份无轨迹。
- `Add_HitBox`、`Normal_HitBox` 仅名称含 HitBox，JSON 没有判定尺寸/时机，FBX 为源空辅助片段；不能用名称推导伤害事件。
- 来源 Animator `applyRootMotion=false`，导出比例已烘焙 100。JSON 的 Avatar 骨数 358 不能替代 UE 实际 491 节点。导入报告中的 `Bip001-R-Hand`、`Weapon`、`Weapon_Effect`、`HitBox`、`CounterHitBox` 是候选挂点名称，不是已确认判定端点；最终使用 UE 回读骨名与武器空间位置。
- 可复用 `BossEncounter → BossState ASC + BossCharacter + AIController`、`BossDefinition/ActionSet/AbilitySet`、BT 选招和 GAS 激活结束桥、`UGGYGOBossMeleeAbility`、通用 Montage Task、GameplayEventWindow 和伤害 GE/Cue。
- 旧 `/Game/AI/Boss/Test/` 是 Pyrios 素材的测试链：`BP_GA_BossMelee_Test` 使用 `AM_BossMelee_Test`、`Ctr_Weapon_A_01→Ctr_Weapon_A_04`、半径 150、Damage 20、PoiseDamage 10、`GE_Damage_SetByCaller`。这些参数不能直接迁给 Kevin。旧测试 ActionSet 只有 `BossAction.Attack.Melee`，距离不限、角度 180，不能视为实战调参。

## 2. 最小实施与文件所有权

| 所属 | 允许修改的产物 | 边界 |
| --- | --- | --- |
| BossAI 模块 | `AI/Boss/Abilities/GGYGOBossMeleeAbility.h/.cpp` | 原动作/Task/窗口/Motion资源批次及终止请求；能力BP持常改值；本计划文档归战斗模块不授予源码写权 |
| Combat 命中查询模块 | `Combat/HitDetection/GGYGOMeleeTraceComponent.h/.cpp`、必要自动化测试 | 唯一窗口/采样/去重/Tick和Owned接口；无效配置明确失败，不拥有伤害或GA终止 |
| 动画模块 | Kevin 动作目录、Montage/Notify、骨架 Socket、最小 ABP、独立编辑器 builder | 优先复用 `BossStageCTestAssetBuilder.cpp` 的原生构造方式；不由战斗模块同时编辑 GGYGOEditor |
| 战斗模块准备、根会话协调执行 | `AAADocs/Scripts/wire_bh3_kevin_combat.py` 与 Wiring Config；新 Kevin GA BP、PawnData/AbilitySet/ActionSet/Definition | 依赖预检后才创建；既有受管资产只读比较并报告差异，不覆盖用户调参；不改旧 Test、主场景或 Pyrios |
| Movement 模块 | `UGGYGOActionMotionProfile`、CMC `BeginActionMotion/EndActionMotion` 与 RMS | 位移唯一执行者；GA 只校验配置、传入速率并持有句柄，不实现第二套轨迹采样 |

本轮不创建通用攻击图框架，不为每段动作写 C++。首批玩法是 Ice01/Ice02；具体 Montage、伤害、Socket、半径、速率与 Profile 放 GA BP，选招参数放 ActionSet。投射物、范围场、召唤、抓取/投技与飞行位移明确为后续执行器，不能全部套武器扫掠。

## 3. 窗口与判定契约

```text
BossAI / 调用方 → 受控 Try Accepted → 保存 Original
BossMelee InitializeAbilityActivation / ActivateAbilityBody:
  原资源批次固定 Original、ASC/Avatar/Mesh/World、Task/Montage、Trace/Motion
  校验 Montage、必需GE、Socket/数值，及 Profile/有效速率/CMC前置
  前置或动作资源失败 → FailOriginalAction → 原 End(true,true)
  Commit / 原 Motion / Task Ready 返回点均核对原批次和终止资格
HitWindowBegin:
  当前原 Task/实例、Montage、主 Mesh与服务器权限有效
  原窗口 Active时保留去重，不重新开窗
  TryOpenOwnedTraceWindow → 保存确切Window → SubscribeWindowHit
HitWindowEnd / BlendOut:
  退原hit订阅、CloseOwnedTraceWindow(原Window)，不清后继窗口
HandleOriginalMeleeHit:
  核对原激活、原批次、原窗口与服务器权限
  GE解析或Builder失败 → 诊断并return，拒绝该hit的damage/Cue
  合法Spec → SetByCaller → ASC Apply → 合法Cue；本分支不终止整动作
Completed / 中断 / watchdog:
  对固定Original发既有End或Cancel，不用当前激活猜原来源
CleanupAbilityResourcesForTermination(Context):
  匹配原批次后摘OriginalResources
  恢复原Mesh策略/prerequisite，清原Timer/Task callback，关原Owned窗口
  CMC.EndActionMotion(原Handle)，原Task.TaskOwnerEnded，父类清理
  native GA早通知：Task已Finished/OwnerFinished，UE登记可能仍持原引用
  native End后续Reset ActiveTasks → ASC本地通知
  原协议Completed/真实返回门禁成立后，由原调用方受控继续下一动作
Trace 每个原回调返回后:
  原窗口失效/替换立即终止旧分发/扫掠，不写后继基线
```

原因：服务器不可见时仍要得到真实武器姿势；Mesh 未刷新会让判定使用旧位置。混出后不能等已被过滤的 NotifyEnd 才收尾。命中回调可能触发死亡/取消/新窗口，旧循环不能在回调返回后再发伤害。

### 原终止阶段与业务失败响应

底层 Original/End/Cancel/延期/Completed 唯一归项目 GA/ASC 协议；Boss 只保存其原资源批次与清理句柄，Task/Combat/CMC各自执行和退出。`FailOriginalAction` 对生命周期前置或动作资源失败使用固定Original的正式End；原Cancel未被接收时不换一个请求强制成功。`CleanupAbilityResourcesForTermination` 在native Super End前释放匹配资源，派生在外调之后不写后继成员。

本机UE的 GA `OnGameplayAbilityEndedWithData` 是早通知，保留native End参数；Finished/OwnerFinished原Task仍可能登记，原生Reset在广播后才完成。ASC `OnAbilityEnded` 位于Reset之后，复制字段固定false。项目Completed还须通过原协议/调用跨度门禁，不能把早通知或资源已释放当成原请求已返回；正常后继在受控调用方的返回后继续。具体见 [[AbilitySystem/计划_原请求终止|终止契约]] 和 [[GGYGO_流程_原请求终止.canvas|原请求流程]]。玩家夹具中的“两原Task”数量不能套到Boss。

raw非虚Try/CallActivate也可在真实NotifyActivated签原身份并清理资源；缺原激活外层受控Try见证时最终UnsupportedEntry、不发布协议Completed。所有direct End/Cancel并非因此一律Unsupported；核心记录详述该来源门禁。旧“OnAbilityEnded内立即raw重新激活同一Spec”的严格验收仍独立保留红测。

**Boss命中失败与Combo不同**：当前 `HandleOriginalMeleeHit` 必须取得ResolvedDamageEffect与有效必需Spec；解析不可用、Builder失败或必需Spec缺失只拒绝该hit的伤害与Cue，不调用原动作End。共享Builder支持空GE不等于Boss选择了无GE正常模式；Gate79无GE正常Case仅验证Combo，正式Boss命中尚未动态验。Combo E14-C才在这两类运行命中故障中使用End(Original,true,true)中止整动作。该响应由各原消费者决定，共享Builder/资源协议不会统一业务策略；本次没有迁移正式Kevin资产。

当前一次 GA 只有一组 Socket/半径/Damage/PoiseDamage；不重叠的多个窗口可各命中同一目标一次。首版禁止重叠窗口，不支持每个窗口不同伤害或多套并行 Trace；确有需求时再提出带窗口 ID 的配置增量。ActionTag 是 ActionSet 查找键，多攻击 BP 必须与各自动作行一一匹配，不能重复同一 Tag 导致首条匹配歧义。

判定按上帧/本帧最大武器长度与半径推导分段，间距不超过半径；默认 `MaxTraceSegments=64`，超过预算关闭窗口并告警。无效 Socket 不回退到 Mesh 原点。首帧只建立基线，后续逐点跨帧球扫掠；低帧率大角度旋转仍采用直线路径近似。专用通道、敌我过滤和精确身体受击体未完成；首个场景限定一 Boss 对一个有 ASC 的验证目标，不能据此声称完整群战或原游戏 HitBox 还原。

动作位移首版只支持从零开始、固定速率、不跳 Section 的线性 Montage；有效速率包括 GA 速率、全局调试缩放与 Montage RateScale，CMC 与 Montage 使用同一时间尺度。BlendOut 只关判定，最终结束/取消释放动作句柄；寻路经现有 Controller 停止，后续移动由 BT 新请求决定。Profile 是独立累计位移曲线，Montage 必须引用派生原地动画，防止双重位移。

## 4. 可复用数据与暂定数据

| 数据 | 来源与用法 |
| --- | --- |
| 动画、FPS、源片段范围、循环元数据、骨架层级 | 可由导出读取；核对 UE 实际帧范围，多出一帧不得悄悄裁掉 |
| RootT/RootQ、BS/Loop/AS 名称 | 用于审计与动作分组；名称不能证明攻击类型，不能据此直接启用循环或胶囊位移 |
| Montage Section/Slot 与动作分组 | 由动画模块产出资产契约和回读报告 |
| 命中起止帧、Socket 偏移/端点、半径、Damage、PoiseDamage、选招距离 | 导出缺失，全部标注人工暂定；在 Montage/GA BP/ActionSet 中调整 |

动画预览的候选窗口：Ice01 45→55 帧；Ice02 8→24 帧为 `Bip001-Prop1` 剑离手横扫/收回。最终窗口仍以动画模块资产回读为准。Ice02 当前设计仅沿骨骼动画轨迹做判定，没有实现独立弹体；Socket 不绑定手骨。Fire01/02 大幅轨迹与其他技能暂不接入这一近战执行器。

## 5. 隔离资产接线契约

- 输出目录：`/Game/Characters/Boss/Kevin/DemonBattle/Gameplay`；ABP 使用动画模块的 `/Animation/ABP_Kevin_DemonBattle`。
- Ice01/02 使用各自 GA BP、Montage 与 `/Motion/DA_Kevin_Ice_Attack_01_Motion`、`_02_Motion`；`BossAction.Attack.Melee.Ice01` / `.Ice02` 已加入 `System/GGYGOGameplayTags.h/.cpp` 原生统一词汇表，本次完整构建已通过。仅注册语义键，未添加业务分支；不能重复基础 Melee Tag。
- 脚本默认只读预检，`--apply` 才创建；核验实际 ABP/骨架、Prop1 Socket、Montage Slot/派生动画段、Notify 类/标签/起止时间与 Profile 时长。UE Python API 尚未执行验证，运行时另有 `ValidateMeleeConfiguration/ValidateMotion`。
- 已有资产必须带本工具所有权元数据；已有受管资产只读比较配置、报告差异，保留人工调参。所有伤害、范围、胶囊、朝向及选择权重均标记人工暂定。
- `behavior_tree` 暂为空，只支持计划中的手动触发隔离验收；不能宣称 Target 初始化或自动选招已接线。

## 6. 当前有限验收与正式战斗边界（2026-10-05）

| 范围 | 实际证据与当前结论 | 剩余门禁 |
| --- | --- | --- |
| 共享原生构建 | Gate79 Editor Succeeded（4 actions、12.03秒、exit0） | 编译不证明Kevin资产/实战 |
| Boss正常生命周期 | Gate79 `GGYGO.BossAI.Melee.NormalLifecycle` Success/0E0W；隔离夹具的受控原请求/原Task/Motion/Owned资源正常链有限通过 | 本叶不证明Kevin Ice01/02真实伤害/Cue、素材配置或所有失败响应 |
| 玩家运行命中 | Gate79四Case行为PASS；两故障叶仍Fail并保留1/2条生产Error，稳定说明见[[计划_玩家普攻连段]] | 不推导Boss命中故障会结束整动作 |
| raw/native严格边界 | Gate71 `GGYGO.BossAI.Melee.EndReentry` 实际Fail，旧结束回调内raw重激活及后继保持断言未关闭；Gate79未选择该叶 | 保留原报告/日志和严格场景，不用NormalLifecycle替代 |
| Combat查询 | Owned接口与独立查询证据由[[Combat/结构]]唯一记录；本计划只说明实际消费者 | 正式Kevin窗口/Socket/敌我/TraceChannel/完整实战另验 |
| 正式资产与网络 | 本需求只改原生链及笔记，没有生成/迁移Kevin GA/ABP/Montage/Socket/GE/Cue/Definition/BT | 由各资产作者回读，完整动作、实际数值和Cue表现、双端/延迟仍未验 |

本次七叶整体仍是5 Success/2 Fail/0 Warning、exit1；局部有限接受不改写Gate71/75/77/78历史失败或原诊断。原始证据集中在项目 `AAADocs/Modules/CombatActions/Module_Repair_04_Validation.md` 与各模块验证记录；不在本计划复制逐次日志。

### 历史准备检查点（本批未重新资产回读）

此前已审计源码、旧测试资产和JSON字段，准备隔离Wiring Config、接线脚本与只读观察器，完成JSON/AST和当时构建。旧记录的“资产未生成、脚本未UE执行”仅表示当时检查点，不能用本次原生叶替代后续资产作者的实际结果。当前文件位置：

- `AAADocs/Assets/BH3/KevinDemonBattle/BH3_Kevin_Combat_Wiring_Config.json`：隔离配置，参数保留人工暂定。
- `AAADocs/Scripts/wire_bh3_kevin_combat.py`：默认预检、显式apply创建；既有受管资产保留人工调参。
- `AAADocs/Assets/BH3/KevinDemonBattle/BH3_Kevin_Combat_Runtime_Verification.md` 与 `AAADocs/Scripts/observe_bh3_kevin_combat.py`：只读观察，不激活能力、不生成伤害。

正式首个场景仍须按实际资产证明播放、窗口外不伤害、同窗一次/新窗可再命中、取消后无残留、服务端不可见骨骼刷新、真实GE/Execution血量与Cue、既有Boss/玩家回归；不能靠素材名称、脚本静态通过或本机合成夹具还原原游戏HitBox。

本轮不提交 Git；素材资产处于项目美术忽略路径，后续需由根会话说明本机产物与可重建脚本的交付边界。
