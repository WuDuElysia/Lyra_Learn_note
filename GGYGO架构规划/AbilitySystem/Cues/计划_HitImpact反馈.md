# HitImpact 反馈：决定、实施与验收边界

> [[AbilitySystem/Cues/结构|当前结构与契约]] · [[GGYGO_结构_HitImpact.canvas|结构图]] · [[GGYGO_流程_HitImpact.canvas|流程图]] · [[AbilitySystem/结构|父模块]] · [[计划蓝图|返回导航]]
>
> 本目录记录 AbilitySystem / Cues 的命中表现。Physics 只提供物理材质语义；负责本次工作线的会话不改变模块归属。
>
> 状态日期：2026-10-04。源码冻结、编译、原必要冒烟、生产资产与网络分别记录；不将代码或图文完成等同于整链验收。

## 根因与已选政策

旧实现把无 HitResult 统一替换为 Target 位置，将正常无 Hit 表现与载荷丢失混在一起；无表面语义、已有表面漏映射也统一选 DefaultEffect。仅把零坐标识别改成 HitResult 指针存在，仍不能表达这些输入契约。

已确认的位置模式为 HitResult / ParametersLocation / TargetCenter，默认 HitResult。只消费所选依赖，缺失或非有限值明确失败；原点与合法 Overlap 不因零坐标或 bBlockingHit=false 被判非法。普通材质无 SurfaceType Tag 默认使用明确的 NoSurfacePolicy=Generic 正常模式；已有表面 Tag 却无匹配默认拒绝，仅资产显式开启 bAllowUnmatchedSurfaceGeneric 时可选通用反馈。

没有继续保留旧自动 Target helper 或生产兼容壳。C++ 负责稳定入口、校验和资源生命周期，常改的模式、表面映射及资源选择留在 Cue BP 默认配置。伤害仍走 GA/GE/Execution/AttributeSet，Cue 不形成第二碰撞、能力、音频或移动执行链。

用户决定记录：`F:/ue_project/GGYGO/Saved/ValidationRecords/HitImpactExplicitPolicy_20261004_UserDecision.json`。接口与具体默认值见 [[AbilitySystem/Cues/结构#Blueprint 默认值与正常模式|资产政策]]。

## 已实施的关键机制

1. **Handle 在 Blueprint 前校验。** 原生 Static 派发先 K2_HandleGameplayCue，再 OnExecute，因此 Executed 覆盖 HandleGameplayCue；输入失败在 K2 / 父子反馈之前返回。其它事件保留原路由。
2. **OnExecute 在 Super 前再校验。** 直接调用原生实现也受保护。通过后复制 Location/Normal/旋转、参数、所选粒子/声音强引用以及原 Target/World 弱身份；借用 Effect 不跨外调。参数副本保留原 EffectContext，没有伪造或重写 HitResult。
3. **父类兼容先检查。** Resolve 内只读检查已配置的空间输出、DefaultPlacementInfo / 每项 Override 与原生附着后落点。预检不调用有随机概率/缓存副作用的 ShouldSpawn。冲突明确拒绝整次反馈，反射字段不存在或类型变化明确失败。
4. **外调后只继续自己的合法反馈。** Super / OnBurst 返回后，以及粒子外调后，重检原目标、World、所选资源。失效返回 false 并中止剩余自身反馈；不换目标/资源，也不回滚已发生效果。
5. **结果与实测分开。** Burst 原生 OnExecute 播放后也固定返回 false；本类 true 只描述校验通过与同步派发结束，不能证明实际可见/可听或复制完成。每 Cue 对象每失败原因最多一条定位诊断。

当前有限父类定位检查在 Cue 私有实现内读取固定反射配置；这不是新的共享接口或状态所有者。资源只在调用栈持有，诊断标记不拥有业务状态。单行 FScriptArrayHelper 修正只适配原生 GetRawPtr 的非 const wrapper API，Effects 与元素访问仍 const，没有写入反射数据。

## 当前兼容边界

本机引擎原生定位仍是 BlockingHit → 非零 Parameters.Location → TargetComponent Socket。合法原点 Overlap 可被父类放到目标位置；显式 Parameters/Target 模式也可能被原 Context 的 BlockingHit 覆盖。附着与 Socket 配置可能进一步改变落点。当前方案报告 InheritedPlacementConflict 并拒绝整次反馈，**没有完整支持所有冲突的父类配置**。

需要保留这些配置时，后续应先冻结项目内定位适配与资产迁移契约，再按文件/生命周期拆分实施；本步不修改 UE/GAS 源码，不伪造 BlockingHit，不静默跳过父类来声称成功。不存在截止不明的重试或另一调度器。

原资产须核对：确需命中点的 Cue 提供真实 HitResult；正常无 Hit 表现明确选位置模式；表面映射顺序/层级与通用策略有意配置；继承的空间效果与所选位置兼容。实际资产路径尚未盘点或迁移，本目录不填写虚构 Cue BP 名称。

蓝图覆写 OnExecute、K2/OnBurst 中自行播放或修改配置的表现拓扑需单独验收。原生资源快照防止数组重入造成借用悬空，不等于保证整个反馈事务原子完成。

## 实施与验证状态

| 阶段 | 当前证据 / 状态 | 尚未覆盖 |
| --- | --- | --- |
| P1 两生产源 | 显式政策、两入口门禁、父类兼容检查、调用期资源已实现，有限静态接受并冻结。 | 原资产配置/接线、完整父类冲突配置适配、实际播放和网络。 |
| P2 原测试迁移 | 仅既有 ContextAndCueLocation 叶迁移；原 Context 断言块及其余测试叶比对未变，无新测试路径，源码冻结。 | 未以合成载荷冒称真实碰撞；原叶不证明 K2/OnBurst 实际播放。 |
| Gate64 原失败（历史保留） | UHT 通过，8 个生成文件；Editor 构建 Failed (OtherCompilationError)，exit1，58.88 秒。唯一记录的 error：Cue.cpp84 C2662，const FScriptArrayHelper 不能调用非 const GetRawPtr。 | 该次未完成新运行时链接，未运行旧 DLL 冒烟；不删除原失败。ASC.cpp759 的 NonInstanced C4996 warning 保留。 |
| C2662 单源修正 | 仅移除局部 wrapper 的 const；反向还原后 SHA256 与修前基线完全匹配。const Effects / const EffectType 访问保留；已重新冻结并进入 Gate64R1。 | 不改变政策/接口或反射数据；源码与图文写入分阶段。 |
| Gate64R1 修后统一编译 | Succeeded，exit0，23.86 秒、4 actions；新 runtime DLL 已链接，哈希见下文。 | C4996 warning 保留；编译成功不代替真实资产或网络。 |
| Gate64R1 原必要冒烟 | 五个现有叶均 Success，各 0 error / 0 warning；本目录相关 ContextAndCueLocation、PayloadTargetAndSurfaceTags 均通过。编译后及 UE 后 353 源保持，45 保护仅预期 runtime DLL 变化；UE 已退出，无残留构建/UE 进程。 | 完整烟日志仍有启动期 13 Error / 2 Warning，未称全日志无错。原解析/载荷叶不证明实际父类、BP、媒体或网络；旧 Boss 清理叶不证明新初始 BT 路径。 |
| 生产 / 专项 / 资产 / 网络 | 本次未验收。 | 真实命中、原点/Overlap 的实际落点、父类配置、BP 拓扑、声音听感、Player/Boss 接线与网络。 |
| P3 本目录图文 | 四份局部文档对应已冻结接口；有限 JSON、ID、边引用及新增 wikilinks 核对通过，四份图文交回冻结。 | 父级/全局入口由统筹或原唯一写入者接入；不是 Obsidian UI 或游戏动态验收。 |

官方记录：`F:/ue_project/GGYGO/Saved/ValidationRecords/ModuleRepairGate_20261004_64_Result.json` 保留原失败；修后最终结果为 `F:/ue_project/GGYGO/Saved/ValidationRecords/ModuleRepairGate_20261004_64R1_Result.json`，门禁编号是 **64R1**。

修后原烟报告：`Saved/AutomationReports/ModuleRepairGate_20261004_64R1_Smoke/index.json`；完整日志：`Saved/Logs/GGYGO_Gate64R1_Smoke_20261004.log`。报告 SHA256 为 `F93A6088EA13933C5DE14DA2323AB921A0EBED4728F401CD5D3893B1F1AE12E3`，新 `Binaries/Win64/UnrealEditor-GGYGO.dll` SHA256 为 `E924FA1C24161C21518EFFD3AE286F0F571B1FAEC2E1BBE8D50A134CFFE8C216`。启动期 13 条 LogAutomationTest Condition failed 在叶派发前；两项 Warning 为 DDC TestData 写路径及 Python 反射名冲突，均保留，不列为这五叶错误也不宣称已修复。

### 当前冻结哈希

| 文件（相对 F:/ue_project/GGYGO） | SHA256 |
| --- | --- |
| Source/GGYGO/AbilitySystem/Cues/GGYGOGameplayCueNotify_HitImpact.h | `C27C5626D5F08875F52FD10C622C46E5D0BFA098CDD51759752A519D1C8B54E5` |
| Source/GGYGO/AbilitySystem/Cues/GGYGOGameplayCueNotify_HitImpact.cpp（单行修正后） | `37810A13F1D409356DA54ED781891E3702E553A170B65CB807E02A6F78CA1ECE` |
| Source/GGYGO/AbilitySystem/Tests/GGYGOHitSemanticsTest.cpp | `354EACC2EF9F73AEF5B3CF31BA859887E40E30A16F594FAFB2A53AD7F9697E2C` |

Gate64 失败时 Cue.cpp 的旧冻结哈希为 `225CB88D978234A43D053F824D3945A7D3411A2825A111BD17037227936F3C77`；该历史失败保留。其它两个文件未因本次机械修正改变。

## 诊断与排查

| 现象 | 优先检查 |
| --- | --- |
| 默认 Cue 不播放，MissingHitResult | GA 是否交付原 Context；正常无 Hit 表现是否明确配置 ParametersLocation / TargetCenter。 |
| 无表面 Tag 反馈不符合预期 | NoSurfacePolicy 与 DefaultEffect；普通材质本来不提供项目 Tags。 |
| UnmatchedSurfaceTag | 实际 AggregatedTargetTags、SurfaceEffects 顺序/层级及漏映射；未显式允许时不得通用替代。 |
| InheritedPlacementConflict | 父类 DefaultPlacementInfo / 每项 Override、Socket 和附着配置；合法原点/Overlap 可能触发原生位置兼容限制。 |
| 原生回调后剩余反馈中止 | 原 Target/World 与所选资源生命周期；不会换目标、World 或资产继续播放。 |
| 后续同类失败没有重复日志 | 每个 Cue 对象每种原因只报告一次；查首条包含资产/对象/模式/Tag 的诊断。 |

## 必要原烟检查与验收顺序

原路径：`GGYGO.AbilitySystem.HitSemantics.ContextAndCueLocation`。统筹已在 Gate64R1 新运行时执行 Success、0 error / 0 warning；原 `GGYGO.AbilitySystem.HitSemantics.PayloadTargetAndSurfaceTags` 同样通过，没有新增叶或扩穷尽矩阵。

该叶保留 Context 深拷贝、显式 Origin、距离及严格世界原点断言；最少新增场景为：

- 默认无 HitResult 的精确 MissingHitResult 拒绝。
- 有效测试 World/Actor/SceneRoot 下，TargetCenter 使用实际非零目标位置；无表面 Generic 是正常模式。
- ParametersLocation 保留传入非零位置与世界原点。
- HitResult 模式保留合成世界原点 Overlap；同时断言解析成功与零坐标，并保留原 HitResult 身份/非 Blocking 标志。
- 已知 Metal SurfaceType Tag 漏映射时精确 UnmatchedSurfaceTag 拒绝，不隐式成功。

本轮顺序为修正源码冻结 → 统一编译新运行时 → 原必要冒烟 → 正式资产/接线与实际落点 → 网络。Gate64R1 已完成统一编译与原必要冒烟，正式资产/实际反馈与网络尚未运行；失败历史及不在夹具范围的场景继续分别可见，不降低原断言把阶段交回写成根因全部关闭。

## 图文范围与交接

本次仅新建本目录 `结构.md`、`GGYGO_结构_HitImpact.canvas`、`GGYGO_流程_HitImpact.canvas`、`计划_HitImpact反馈.md`；没有修改父目录/全局入口或相邻模块。本目录类接口静态关系与运行时序分图说明，反射逐字段实现细节留在源码，不展开成测试矩阵。

统筹随后完成父级与跨模块桥接：AbilitySystem 父结构／伤害流程、Physics 材质入口、Audio 结构／流程及总览反馈块已删去旧隐式 Target／DefaultEffect 描述，并接入本目录链接；模块参考与计划导航同步。其他 GAS 历史图文尚未完整迁移受控激活／终止接口，以总览当前状态为准。本目录原作者交回冻结后由统筹更新本交接段，没有扩展源码／资产权限。
