# GameFeature 计划

> 返回 [[计划蓝图]] · 当前代码事实见 [[模块参考]]
>
> 当前验证：Gate57已编译并链接新运行时／Editor DLL，GF正式运行未验。原阶段标题保持历史锚点，交回时未编译与Build56失败原文在下文保留。

## 11. GameFeature 架构

### 11.1 Lyra 的模型

```
LyraExperienceDefinition (DataAsset)
  ├─ DefaultPawnData
  ├─ GameFeaturesToEnable[]
  └─ Actions[]
        ├─ AddComponents            ← 给指定 Actor 类加组件
        ├─ AddAbilities             ← 给 ASC 加 AbilitySet
        ├─ AddInputContextMapping   ← 加输入映射
        └─ AddWidgets               ← 加 UI

LyraExperienceManagerComponent（挂 GameState）
  └─ 加载 Experience → 激活 GameFeature → 执行 Actions → 广播就绪
```

**GameFeature 的价值是"按需装配"，代价是"初始化时序变复杂"。** Lyra 用 `UGameFrameworkComponentManager` 的 InitState 状态机（`ULyraPawnExtensionComponent`）协调"组件什么时候能用"，这是 Lyra 最难懂也最容易抄错的部分。

### 11.2 分两步走

**第一步（现在）：只抄 InitState 协调机制，不抄 GameFeature 插件化**

> InitState 机制与角色组件如何拆分，详见 [[计划_角色与组件#14. 角色组件化与 InitState|角色与组件计划第 14 章]]。

- 抄 `LyraPawnExtensionComponent` 的 InitState 状态机，协调 ASC / PawnData / HeroComponent 的初始化顺序
- 抄 `LyraHeroComponent` 的 `NAME_BindInputsNow` 扩展点
- 不建 GameFeature 插件，Ability 和输入直接由 PawnData + AbilitySet 静态配置

InitState 机制本身就有独立价值，而且它是 GameFeature 的前置。它要解决的是这类竞态：
HeroComponent 想绑输入，但 ASC 还没建好；CMC 想读 `Restriction.CantMove`，但 PawnData 还没分发。
把这些顺序写成 `BeginPlay` 里一串手工排列的调用，在联机下会因为 PlayerController
到达时机不确定而失效——客户端上 Controller 可能比 Pawn 晚到几帧。

**第二步：把角色做成 GameFeature 插件**

- 每个角色一个 GameFeature 插件（角色 Ability、动画资产、输入配置、Cue）
- 编队决定激活哪些插件
- D2 下这一步收益明显：4 人房 12 个角色，不做按需加载会把全部角色资产常驻内存

**什么时候动手：下面任一条成立。**

| 触发条件 | 为什么它是判据 |
|---|---|
| 角色数达到 4 个以上，且各自带独立的 Montage、Cue、动画资产 | 按需加载省下的内存要先真实存在。角色只有一两个时，全部常驻和按需加载的差别测不出来 |
| 要把角色作为独立内容包交付（不重新打包主程序就能上线新角色） | 这是 GameFeature 唯一无法用别的手段替代的能力。用静态 PawnData 做不到 |
| 不同模式需要不同的能力集（PvP 削弱版、试玩版限技能） | `GameFeatureAction` 正是为"同一角色在不同上下文下装不同东西"设计的；用 PawnData 表达要为每种组合复制一份资产 |

反过来说，**只是角色变多但资产共用、也不需要热更**，那就不该动 ——
代价是同时调试 GameFeature 时序、GAS 时序、CMC 时序三套异步链路。

**前置项：原Session所有权、当前资格与退场边界已有源码。**
09-G4-2已由GameMode消费Experience成功输入，先保存原Session再Start，替换旧裸激活等待计数；
原World／AuthGameMode／GI／同Session资格决定是否装配，Destroyed／EndPlay共用原资源关闭。
Gate57已统一编译并链接新双DLL；“源码有限接受尚未经过新编译／UHT／UE”是GF-N4交回时历史状态。来源资产、Teams内部创建责任与实际插件生命周期仍待验收，再接入Action。生命周期目标见11.4。

### 11.3 第一步的落地结构

```
GameModes/
  GGYGOExperienceDefinition.h/.cpp      一局配置：GF根列表 + 显式来源 + 队伍成员
  GGYGOGameFeatureSession.h/.cpp        原World Active资源（09-G4-1冻结，GameMode已源码消费）
  GGYGOGameMode.h/.cpp                  Experience输入／原Session请求／当前资格／双入口关闭（09-G4-2冻结）
Character/
  Components/GGYGOPawnExtensionComponent.h/.cpp   InitState 协调
  Data/GGYGOPawnData.h/.cpp                       角色定义 DataAsset
Teams/
  GGYGOSquadComponent.h/.cpp            队伍成员与出战切换（挂 PlayerState）
```

**请求在`InitGame`发起，装配消费当前原场景资格。** `InitializeGameFeatureSession`先核父错误及原World／实际AuthGameMode／GI、有效Experience，再调用`TryBuildGameFeatureInput`。转换成功且roots／sources均空才进入显式NoGF；非空成功Input由`TryCreate`返回原Session，先保存`GameFeatureSession`再`Start`，完成可同步。工厂／配置失败明确拒绝，失败空输出不能变成NoGF。

`AreGameFeaturesReady`／`CanAssembleForGameFeatureContext`核调用方开放、同一个原World／实际AuthGameMode、GI当前World和同一Session，再读`IsReadyFor(OriginalWorld)`；成功NoGF配置只免除Session请求，仍须原场景资格。插件Action晚于装配、历史Ready或仅pending结清都不能授予业务成功。

两个必须处理对的细节：

- **同步完成前已保存原资源。** 旧逐根计数曾在首根同步callback时提前归零；现在原Session先存入GameMode，真实根成功／Prepare及整批dispatch返回／pending排空／全集合当前native Active由Session唯一判断。GameMode弱观察者仍重核原World及同Session，通知本身不是资格。
- **真实pending、业务失败与退场分开。** 非Ready结果保留真实错误并封闭原调用方，Session负责自己失败排空；不失败放行、不自动重试。`Destroyed`／`EndPlay`在各自Super前共用`CloseGameFeatureSession`：先关调用方、清NoGF并移出原Session，再Close；外部调用后不重置成员，重复入口无第二份资源。覆盖未BeginPlay的原生Destroy，不能把Close请求当实际清理完成或撤回GI Loaded。

`SpawnSquadForPendingPlayers`从当前`GetPlayerControllerIterator`取局部`TWeakObjectPtr<APlayerController>`快照，不保存等待名单；每个存活同World玩家在外部Spawn前后核同原Session及资格。`HandleStartingNewPlayer_Implementation`同样使用原World／Player弱身份；`IsSquadAssembled()`保留既有已装配跳过。三段Spawn内部的借用登记、部分失败处理与创建责任未迁移；GF外层门禁不构成Teams装配验收。

**还有一个 GameFeature 的前提在 AssetManager 上**：`DefaultGame.ini` 必须注册
`GameFeatureData` 这个 PrimaryAssetType，否则 GameFeatures 子系统在启动阶段就报
"Asset manager settings do not include a rule for assets of type GameFeatureData"，
插件功能整体不可用。它的 `Directories` 留空即可 —— 那些资产在各插件自己的根目录下，
由插件系统发现，不需要 AssetManager 去 `/Game` 里扫。

### 11.4 最终策略：只停用、不卸载（目标，未实施）

2026-09-30 用户最终约定：**同一 GameInstance 正常运行及切图/返回菜单期间，保留原生 Loaded 需求；对局关闭只撤回自己的 Active 需求。** 中途提出的自动卸载选择已撤销，不能作为实施依据。

这项策略只针对实际 GameFeature 内容，不表示当前 Kevin 内容或 Camera、GAS 等项目 C++ 模块已经插件化。11.2 的角色插件化仍是后续内容组织计划。

#### 当前事实与缺口

- 当前链为`InitGame → InitializeGameFeatureSession → Experience.TryBuildGameFeatureInput`：成功两空才NoGF；非空`TryCreate`后先保存原完整Session再`Start`。GameMode不直接调用native裸激活、不持native handle、不维护插件等待计数。
- Ready观察者及两个装配入口都核原World／实际AuthGameMode／GI当前World／同Session和调用方开放；非空再查`IsReadyFor(OriginalWorld)`。当前Controller局部弱快照逐调用前后重核；Destroyed／EndPlay在Super前共用移出原Session的关闭消费者。
- GI Loaded宿主、完整Managed加载、Experience显式配置、不可变激活请求值及Session已有冻结源码，09-G4-2 GameMode消费者也已冻结并获统筹有限静态接受。Gate57统一编译Succeeded（9 actions／37.13秒），更新7个UHT生成文件并链接新运行时与Editor DLL；GF消费者与Host H3已进入新编译，GF正式运行仍未验。来源资产、Teams内部创建及实际生命周期未验。保留11.4标题锚点，“目标，未实施”是原计划标题，当前已有部分源码但整体生产目标尚未验收；历史核心构建不覆盖新增代码。
- GF-N4交回历史原状态：最新Build56仍Failed (OtherCompilationError)，仅Hero旧三调用；该GF消费者与Host H3增量未新编译／UHT／UE。

#### 已编码底层，不等于生产接线

| 底层 | 已有接口与实现 | 尚未证明 |
| --- | --- | --- |
| `FGGYGOGameFeatureRetention`（09-G1冻结时未成功整链编译；现Gate57已编译） | 同一handle逐URL真实Load；真实回调与调用返回排空后才Loaded；关闭停止未发项、排空已发项再原生释放，保留Submitted非ack | GI已调用新加载，GameMode经Session间接消费；真实插件、跨图Loaded及外部借用未验。 |
| `FGGYGOGameFeatureClosureResolver`（G0-2历史已编译） | 当前policy同步引用＋按值roots／来源，完整enabled依赖交叉；Rejected无部分候选 | Experience→GameMode→Session→GI的成功FInput已源码接入；候选仍非Loaded／权限，来源资产及借用保护未验。 |
| `UGGYGOGameFeatureSubsystem`／原租期（09-G2冻结时未成功整链编译；现Gate57已编译） | 正常Game／PIE GI collection创建ordinary owner，Prepare同步解析全Managed→真实Loaded→原World租期；GI跨图保活、关闭不提前释放 | GameMode只经原Session传入成功配置，不直接释放GI；Borrowed拒绝，不保存policy、不新增Ready或引用计数，G0-3 PostInit候选不接入。 |
| 租期激活请求值（09-G4-0冻结时未成功整链编译；现Gate57已编译） | `GetActivationPluginURLs`返回私有不可变数组只读view；根＋true可达依赖，先验证闭包、失败无部分输出；完整Nodes仍Loaded | Session消费全请求平铺Active、真实根激活及全集合当前Active，GameMode已源码消费该Session；值非状态／引用权限，不关闭C15。 |
| `FGGYGOGameFeatureSession`（09-G4-1冻结时未成功整链编译；现Gate57已编译） | `TryCreate / Start / IsReadyFor / Close`；原World／GI弱身份，持原Loaded租期及一个None Active GUID；真实根与调用返回门禁、原生release门禁、独立析构清理 | GameMode已保存／Start／当前资格查询／双入口Close；真实停用／切图／其他持有者／PIE Action隔离未验。Unconfirmed／丢失回调可能保留至进程结束。 |
| `UGGYGOExperienceDefinition`（09-G3冻结时未成功整链编译；现Gate57已编译） | 反射来源数组＋`TryBuildGameFeatureInput`纯值转换；IsDataValid与GameMode共用，两空为明确NoGF、缺失／非法／冲突失败 | 实际来源资产未迁移；GameMode新增消费者已进入Gate57；交回时未新编译／UHT／UE为历史状态，GF正式运行未验。输出值不是Loaded／Active或借用保护。 |
| `AGGYGOGameMode`（09-G4-2交回时未新编译／UHT／UE；现Gate57已编译） | InitGame请求原Session、当前原身份资格、局部弱Controller派发；Destroyed／EndPlay共用移出原资源后Close | 三段Spawn内部仍借用登记、部分失败／创建责任未迁移；GF源码有限接受不代替Teams、资产或动态验收。 |

GF-N4交回历史原文：旧Loaded core/R1第21次、解析器第25次构建成功；09-G1／G2／G3／G4-0／G4-1未成功整链编译／UE，09-G4-2新增消费者未新编译／UHT／UE。实际源码接线与已跑通插件生命周期分别记录；核心输入、提交、同步／异步释放及失败合同见[[GameFeature/结构|实际接口说明]]。

当前验证更新：Gate57统一编译Succeeded（9 actions／37.13秒），更新7个UHT生成文件并链接新运行时与Editor DLL；GF消费者与Host H3已进入新编译，GF正式运行仍未验。Gate57后新UE40416已连接原生MCP；统筹18:06:35记录的`ActorInfoTransaction.NativeLifecycleAndHistory`单叶Success，事件段0错误／0警告。该单叶范围不覆盖GF插件Session或GameMode→Teams正式链；GF正式运行、来源资产、C15／Borrowed、PIE Action隔离及网络继续未验。

#### 目标生命周期与唯一归属

| 生命周期 / 资源 | 目标归属 | 关闭或终止语义 |
| --- | --- | --- |
| 每局pending／Active资源、完成通知 | `FGGYGOGameFeatureSession`（已编码，GameMode已源码消费，运行未验） | 关准入／撤观察者／停止未发项，真实工作及调用返回排空后撤自己Active；callback、返回、GUID注销门禁后归还原租期，旧World不能授新World Ready。 |
| 原场景请求、装配资格与自有Session句柄 | `AGGYGOGameMode`（09-G4-2冻结，未新门禁） | 先保存再Start；每次核原身份及IsReadyFor。Destroyed／EndPlay共用封闭调用方→移出原Session→Close→各自Super；不管理native状态或GI Loaded释放。 |
| 跨地图原生Loaded需求 | `UGGYGOGameFeatureSubsystem`／ordinary owner（Gate57已编译，GF正式运行未验） | 正常切图保GI持有；GI结束关准入／撤自身持有，最后原World租期归还才释放，原core排空真实pending。 |
| 插件状态、引用合并、状态迁移 | UE 原生 handle / reference controller | 不新增项目使用计数、插件状态机或第二套调度器。 |

释放自己的 Active 需求之前，必须建立所需 Loaded 保留；只剩 Loaded 需求时由原生机制停用至 Loaded。其他原生 Active 使用者不受影响，插件也可能因原生更高需求继续 Active；Loaded 是最低需求，不是强制降级命令。

薄GI子系统已实现，无自定义GI或UEngineSubsystem；原租期给根URL／完整请求URL值及原World准入，不暴露handle／Release。GameMode现在先保存完整原Session再Start，Session实际Prepare并保租期到自身Active／pending收尾。GameMode双关闭入口仅移出并Close原Session，GI继续唯一持跨图Loaded。完整合同见[[GameFeature/结构#GI薄宿主与原World租期（09-G2：已编码，未编译）|GI／租期合同]]及[[GameFeature/结构#原World Active会话（09-G4-1：源码冻结，未生产接线）|Session合同]]。

独立 PIE 的 GameInstance 各持有自己的保留资源，原生插件状态仍由进程共享；结束一个实例只能撤回本实例需求，不能影响其他原生 Active 使用者。GameInstance 结束后允许原生机制按剩余需求清理，**不承诺编辑器 PIE 停止后永不卸载**，也不把进程共享插件状态等同于各 World 的 Action 隔离。

#### 实施前仍须明确的契约

- **依赖 Loaded 保留：**UE原生`TrackDependencies`对根Loaded需求不保证全部依赖Loaded。GI已连接全Managed闭包至同一Loaded handle，Experience→GameMode→Session成功输入已有源码；实际来源资产、插件运行及Borrowed保护仍未证明，不能把解析值或图上已接线当依赖已被动态保护。
- **外部裸 API 归属：**`IsActive` 不能证明所有者。原生 handle 使用者可由原生引用机制保护；外部裸 API 预激活不能被自动判为本局所有。实施前必须明确接入原生引用或外部借用边界，不能擅自取得生命周期后撤回并误停外部使用者。
- **初始化与终止：**GI／原租期在同步完成前构造，GameMode在Start前保存完整原Session。真实根全成功、GI Prepare／整批dispatch返回、pending空、全集合native Active及原准入才Ready；GameMode另核原World／AuthGM／GI／同Session当前资格。非Ready封闭原调用方，Session保native失败优先并自己收尾；Destroyed／EndPlay Super前共用移出原资源后Close，旧观察者不通知新World。GI Deinitialize撤自身持有，Session真实排空及Active释放门禁后归还租期，最后owner才释放Loaded。受限正常Game／PIE及exact AssetManager启动尝试完成窗口保持，不支持任意policy就绪探测／手工RPC／动态reload或元数据变化。
- **无法确认的清理：**Close原生true才Closed；正常守卫下native false为Failed且自有引用已撤才归还租期；原生不可用／守卫丢失为Unconfirmed并保留，实际回调未到仍Pending，无假成功或自动native重试。析构只移交同一handle／租期给独立清理记录，无裸this／World／GI捕获；丢失回调或未确认资源可能持续到进程结束，须保留可见诊断。

#### 待验收与实施停止点

GF-N4交回历史原文：待验证：同步完成、双入口幂等／未BeginPlay销毁、关闭时真实pending、切图／回菜单保持Loaded、其他Active持有者、独立PIE终止／Action隔离、实际依赖保留及外部裸API边界。Gate55历史失败、UHT写33生成文件及353源码／45保护hash保持留账，Character／Task／RMS机械修正已冻结并获统筹接受；最新Build56失败仅Hero旧三调用，未链接新运行时／UE。GF消费者与Host H3增量未新编译／UHT／UE。Input硬件因果根范围已纠正、Hero A直接迁移，不再把注入兼容列为用户决策；GA兼容／Camera可见失败仅相关线等待。历史记录`Saved/ValidationRecords/ModuleRepairGate_20261003_55_Result.json`及`Saved/ValidationRecords/ModuleRepairGate_20261003_55_MechanicalFixes.json`保留，最新Build56以统筹记录为准，未执行上述动态验收。

当前Gate57：Gate57统一编译Succeeded（9 actions／37.13秒），更新7个UHT生成文件并链接新运行时与Editor DLL；GF消费者与Host H3已进入新编译，GF正式运行仍未验。Gate57后新UE40416已连接原生MCP；统筹18:06:35记录的`ActorInfoTransaction.NativeLifecycleAndHistory`单叶Success，事件段0错误／0警告。该单叶范围不覆盖GF插件Session或GameMode→Teams正式链；GF正式运行、来源资产、C15／Borrowed、PIE Action隔离及网络继续未验。

本GF-N4仅四份既有局部图文同步已接受09-G4-2消费者及双入口关闭。来源资产迁移、Teams三段Spawn内部创建／部分失败、插件动态运行、C15与网络仍开放；另租Squad原创建Actor终止消费者不构成装配创建完成。Source README另一步维护，源码／项目记录／全局入口／资产无本次写权；后续仅统筹统一编译＋必要UE冒烟，不扩严格矩阵。普通Loaded历史构建、静态接受或本次图文保存不能作为整模块完成。

#### Experience配置与后继接线顺序

09-G3已经提供`FGGYGOGameFeatureSourceDeclaration`（PluginName／Source／ExternalOwnerLabel）、`GameFeatureSources`及同步`TryBuildGameFeatureInput`，不需要为每个模式增加C++业务。Managed不接受Label，Borrow要求有效Label，Unspecified不默认托管；相同声明去重，冲突与缺根声明拒绝并定位原索引。完整依赖声明及URL仍由GI／Resolver验证。详细合同见[[GameFeature/结构#Experience显式来源配置（09-G3：已编码，未编译）|配置接口]]。

当前源码顺序：Experience成功转换 → GameMode仅成功两空选择NoGF；非空TryCreate后保存完整原Session再Start → GI原Loaded租期 → Session全请求Active登记／真实根激活／Ready门禁 → GameMode同原World／AuthGM／GI／Session且当前IsReadyFor才装配 → Destroyed／EndPlay共用原资源Close。来源资产与真实运行未验，Teams内部三个Spawn仍须创建责任迁移；失败空输出非NoGF，C15保护落实前仍拒绝Borrowed。每局启用项及来源属于数据资产，底层权限与资源清理归C++唯一责任。

Active依赖升级可能不重走已遍历子树是只读源码推导，未动态复现。09-G4-0将根＋true可达完整请求URL写入原租期，false-only仍Loaded，错误在native前拒绝；该阶段统筹有限复核／hash／范围diff通过，未新编译／UE。Session已消费全请求登记→真实根激活→全集合当前Active，09-G4-2 GameMode已实际源码消费原Session；请求集合或根URL仍不等于Active保护证据。接口见[[GameFeature/结构#不可变激活请求URL（09-G4-0：源码冻结，未生产消费）|请求集合接口]]，保留阶段锚点，不修改引擎。

---
