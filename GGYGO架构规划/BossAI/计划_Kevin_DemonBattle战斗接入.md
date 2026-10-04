# Kevin DemonBattle 战斗接入计划

> 当前阶段：近战防护、CMC 句柄接入、自动化测试和资产接线脚本已编码；统一完整构建通过，脚本和动态验收尚未执行。动画模块维护动作分组、Montage、骨架与编辑器 builder；根会话协调全局状态、构建、UE 操作与最终验收。此文件与 BossAI/Combat 局部图由战斗模块维护。禁止把暂定判定写成原游戏还原。
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
| 战斗模块 | `AI/Boss/Abilities/GGYGOBossMeleeAbility.h/.cpp` | 配置校验、主 Mesh/事件来源、服务端 Tick、BlendOut/中断/完成兜底清理；能力 BP 继续持有常改值 |
| 战斗模块 | `Combat/HitDetection/GGYGOMeleeTraceComponent.h/.cpp`、必要自动化测试 | Socket/数值无效时拒绝开窗；防止命中回调关闭/重开窗口后旧扫掠继续广播 |
| 动画模块 | Kevin 动作目录、Montage/Notify、骨架 Socket、最小 ABP、独立编辑器 builder | 优先复用 `BossStageCTestAssetBuilder.cpp` 的原生构造方式；不由战斗模块同时编辑 GGYGOEditor |
| 战斗模块准备、根会话协调执行 | `AAADocs/Scripts/wire_bh3_kevin_combat.py` 与 Wiring Config；新 Kevin GA BP、PawnData/AbilitySet/ActionSet/Definition | 依赖预检后才创建；既有受管资产只读比较并报告差异，不覆盖用户调参；不改旧 Test、主场景或 Pyrios |
| Movement 模块 | `UGGYGOActionMotionProfile`、CMC `BeginActionMotion/EndActionMotion` 与 RMS | 位移唯一执行者；GA 只校验配置、传入速率并持有句柄，不实现第二套轨迹采样 |

本轮不创建通用攻击图框架，不为每段动作写 C++。首批玩法是 Ice01/Ice02；具体 Montage、伤害、Socket、半径、速率与 Profile 放 GA BP，选招参数放 ActionSet。投射物、范围场、召唤、抓取/投技与飞行位移明确为后续执行器，不能全部套武器扫掠。

## 3. 窗口与判定契约

```text
BossMelee Activate:
  if Montage / 主 Mesh / Trace / Socket / 数值无效: 取消
  if 配有 MotionProfile: ValidateMotion，时长与 Montage 误差 <= 1ms，禁止原生 RootMotion
  if 实际速率无效 or Profile 所需 CMC 不存在/已有动作: 取消
  CommitAbility
  保存 Mesh Tick/URO → 服务器强制骨骼刷新
  Trace Tick 依赖 Mesh → AIController::StopMovement
  可选 BeginActionMotion(Profile, EffectiveRate) → 播放同速 Montage
  位移启动失败取消；绑定窗口/完成/混出/中断
HitWindowBegin:
  if 非当前 Montage/主 Mesh/播放实例 or 已混出: 丢弃
  if 当前已开窗: 不重置命中集合
  else BeginTraceWindow → 重置去重与采样基线
HitWindowEnd / BlendOut:
  EndTraceWindow
OnMeleeHit:
  if 能力不活跃 or 无服务器权限 or 窗口已关闭: 丢弃
  应用 Damage GE / Cue
EndAbility / Cancel / 完成回调超时:
  幂等关闭 Trace、解绑事件、清计时器
  MontageTask::TaskOwnerEnded 停止自有 Montage → 恢复 Mesh Tick/URO
  EndActionMotion(Handle) → 清引用 → 最后 Super::EndAbility
  OnAbilityEnded 可同步重新激活；Super 后不再修改实例成员
Trace 广播返回后:
  if 窗口已关闭 or 窗口代号已改变: 立即终止旧扫掠
```

原因：服务器不可见时仍要得到真实武器姿势；Mesh 未刷新会让判定使用旧位置。混出后不能等已被过滤的 NotifyEnd 才收尾。命中回调可能触发死亡/取消/新窗口，旧循环不能在回调返回后再发伤害。

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

## 6. 验收与当前状态

1. **审计完成**：源码、旧测试资产与 JSON 字段已核对；没有修改旧 UE 资产。
2. **完整构建通过，自动化测试待运行**：`GGYGO.Combat.MeleeTrace.SafetyAndCoverage` 覆盖无效 Socket、长武器采样、窗口去重/重开/结束、回调取消与重开、超预算关闭。`GGYGO.BossAI.Melee.EndReentry` 以真实 GAS 结束广播重新激活同一 Spec，验证清理先于广播；Boss 反射测试类位于 `AI/Boss/Tests`，Combat 测试不依赖 Boss 上层。
3. **脚本已准备，资产未生成**：Wiring Config/脚本通过 JSON/AST 静态检查；动画配置已有 Ice01/02 候选窗口和派生动画路径，尚待 builder 实际生成 ABP、原地动画、Profile、Montage/Socket，并在新构建中回读核验。构建和重启由根会话统一协调。
4. **运行待验收**：首个近战可播放、窗口外不伤害、同窗口只伤一次、下一窗口允许再伤、取消后无残留、服务器不可见仍更新骨骼、旧 Boss 测试与玩家窗口自动化回归。实际命中和血量变化必须给出日志/自动化证据。

新增动态验证方案 `AAADocs/BH3_Kevin_Combat_Runtime_Verification.md` 与只读观察脚本 `AAADocs/Scripts/observe_bh3_kevin_combat.py`，AST 已检查，尚未 UE 执行。观察器只订阅真实命中/血量事件并采样 Montage、Trace、CMC 与 Mesh 设置，不激活能力、不生成伤害。统一构建由根会话完成，UHT 与两个 DLL 链接成功；构建证据不代替上述运行验收。

本轮不提交 Git；素材资产处于项目美术忽略路径，后续需由根会话说明本机产物与可重建脚本的交付边界。
