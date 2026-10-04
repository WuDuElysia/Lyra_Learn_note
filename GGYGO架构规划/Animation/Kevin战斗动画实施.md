# Kevin DemonBattle：战斗动画实施

> 当前：源数据审计与原始动画逐帧取样预览完成；Montage、Socket、Notify和ABP生成器已准备。A1–A5安全修复完整链接通过，3项Animation安全自动化冷启动通过，零错误警告。见 [[Animation/BH3_Kevin_DemonBattle导入|素材导入]]、[[Animation/结构|Animation结构]]、[[Animation/资产生产安全|安全契约]]、[[GGYGO_流程_Kevin战斗动画.canvas|生成与运行接缝]]。资产生成与运行验收仍待执行。

## 来源证据与边界

79份JSON/FBX没有导出伤害事件、碰撞体参数或HitWindow；75有效片段与4空片段用途清单位于项目 `AAADocs/BH3_Kevin_Combat_Action_Manifest.json`，审计说明为 `AAADocs/BH3_Kevin_Combat_Source_Audit.md`。55条拟独立Montage准备，15状态/4姿势/1测试素材保留；4空片段跳过。

`Weapon`网格3064顶点全部权重1绑定`Bip001 Prop1`（UE名`Bip001-Prop1`）；`Weapon`、`HitBox`、`CounterHitBox`、`Weapon_Effect`、`LanceThrowAttach`没有本地动画轨道。节点运动不能解释为伤害窗口。

## 生成器与资产契约（已编码，待执行）

- Animation负责`Source/GGYGOEditor/Private/KevinCombatAssetBuilder.cpp`及人工配置 `AAADocs/BH3_Kevin_Combat_Montage_Config.json`，生成Montage/Slot/Socket/Notify及独立ABP。它是编辑器资产生产工具，无运行时招式分支、计时器、伤害或位移执行器。
- 源审计脚本只重建证据与用途建议，不读写人工配置。未确认的伤害窗口为空且disabled；来源标为`manual_candidate`才允许按人工标定帧生成Notify。
- 每份源片段先独立Montage，`DefaultSlot`、一个`Main` Section自动结束，不按AS/BS/Loop名称拼接或无限循环。
- 首批Ice01/02已目视取样，人工候选窗口见下表；Fire01/02及远程/召唤/投技没有伤害窗口。
- 两个不同武器Socket依据UE导入LOD0的武器section顶点和Prop1参考骨变换生成局部端点，并输出几何证据；不使用同一个Weapon占位点充当线段两端。
- Notify使用现有`UGGYGOAnimNotifyState_GameplayEventWindow`的`Event.Montage.HitWindowBegin/End`，同Montage窗口按序、无重叠。
- [[Movement/结构|Movement]]独占曲线权威运动与Ice01/02派生原地动画；源序列保留。Montage配置支持派生路径覆盖，不在Animation生成器改骨盆或开启RootMotion。
- `ABP_Kevin_DemonBattle`由Animation唯一生成：StandBy SequencePlayer循环 → DefaultSlot → OutputPose；Combat仅引用。当前最小表现拓扑没有移动状态机，不能宣称完整Boss locomotion已实现。
- [[Combat/结构|Combat]]负责现有BossMelee能力、GA/测试接线；接口保持`AttackMontage`、`TraceStartSocket/EndSocket`、`TraceRadius`、`Damage`、`PoiseDamage`、`HitCueTag`。动画仅发事件，不扣血或移动胶囊。

## 人工窗口与预览证据

| 片段 | 60fps候选窗口 | 观察依据与限制 |
| --- | --- | --- |
| Ice_Attack_01 | 45–55帧，0.75–0.916667秒 | 查看20/30/45/48/51/55/65/80帧；48帧剑横出，51–55帧转入收势。仅作为初版候选，不是源游戏命中数据。 |
| Ice_Attack_02 | 8–24帧，0.133333–0.4秒 | 查看0/8/15/21/30/45帧；剑随Prop1离手扫出再收回。24帧边界是21与30帧观察之间的人工估计，待游戏命中验收调整。 |

两条均标`manual_candidate`。Ice02是骨骼动画驱动的离手武器轨迹，不代表已有独立弹体生命周期、追踪或碰撞系统。预览原序列不等于Movement派生序列和胶囊位移的组合已验证。

Fire01另查看80帧姿态；Fire01/02的8.6m/18.7m源整体位移来自FBX轨道审计，不能从单帧截图证明距离或命中语义，继续禁用伤害。

## 构建接口与保护

1. 统筹先执行Movement的`GGYGO.BuildKevinMotionAssets audit/build`，生成`AS_Kevin_Ice_Attack_01_InPlace`和`AS_Kevin_Ice_Attack_02_InPlace`；Animation只读取它们，必须RootMotion关闭。
2. 执行`GGYGO.BuildKevinCombatAssets [可选配置路径]`，默认读取上述JSON。55个独立片段各有一个`Main` Section，下一节None；循环源也只播放一次，不自动拼接。
3. 校验窗口有序、间隔、有限数值且结束早于混出；混入/混出上限为片段长度25%，短姿态片段不会被过长混合覆盖。仅2条配置damage_enabled，其他53条只准备资产。
4. `BuildSockets`验证LOD0指定材质section所有顶点刚性绑定`Bip001-Prop1`，转换到该骨参考局部空间，协方差主轴投影两端生成mesh-only的`KevinWeaponTraceStart/End`。输出实际UE顶点数、端点、长度、最大径向距离。PCA几何端点仍需场景检查；半径由GA配置。
5. 已有自有且干净的Montage/ABP/Socket只读核验，保留人工图节点、表现Notify、混合值及Socket位置；metadata不代表覆盖授权。未知资产、脏包、配置差异或半组Socket均停止。缺失ABP才创建三节点姿态链并`CompileBlueprint`；已有ABP只核验编译状态及Slot存在，不能证明有效输出接线。
6. 只保存本次新建资产；报告`Saved/Codex/kevin_combat_asset_report.json`区分`success`、`status`、`saved_this_run`，记录序列/窗口数/Slot/停止关系、Socket几何及ABP检查。验证通过不等于本次保存。Movement派生产物同样先核验类型、脏包、磁盘及生成基线，再创建缺失输出。

依赖仅增加在`GGYGOEditor`：Json、AnimGraph、AnimGraphRuntime、BlueprintGraph、KismetCompiler；运行时模块没有反向依赖编辑器。业务动作、混合值、窗口在配置/资产中，权威位移在Movement，命中查询在Combat，生命周期与GE在GA。

## 尚未验收

统筹最新完整链接成功（5个动作，3.05秒），冷启动单独复测2/2及全套GGYGO自动化10/10通过，零错误警告；包含3项Animation安全测试和RootMotion加强断言。Python离线回归8/8通过。生成器执行、派生原地动画接线、Socket实际空间位置、独立测试关卡中的命中/取消清理均未完成。当前窗口只具备原序列预览证据；应以资产回读和运行验收补齐，不能将候选写成最终命中语义。
