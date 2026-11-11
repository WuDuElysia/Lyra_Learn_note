# Kevin DemonBattle 战斗接入

> 当前为已保存并通过单机自验的完整 DemonBattle 单形态、两阶段组装；负责人为 BossAI 原会话。项目权威需求：`AAADocs/需求文档/kevin_boss_assembly.md`；配置和验证入口在 `AAADocs/Assets/BH3/KevinDemonBattle/`。
> [[BossAI/结构]] · [[GGYGO_结构_BossAI.canvas]] · [[GGYGO_流程_BossAI.canvas]] · [[Animation/Kevin战斗动画实施]] · [[Movement/结构]] · [[Combat/结构]]

## 正式入口与配置

`/Game/Map/BH3/L_KevinBoss_P3` 的 Encounter 创建 BossState、Controller 与继承共享 `BP_BossCharacter` 的 Kevin Pawn；Definition 引用 PawnData、AbilitySet、BB、BT 和两阶段动作池。蓝图 BTS 获取／失去目标；BossState 蓝图消费生命事实，在 50% 时切换第二阶段。C++ 只校验／发布状态、筛选／加权选招、请求 GAS，不编码 Kevin 具体技能策略。

第一阶段为 Ice01～05、Counter 与 DashF；第二阶段增加 Fire01～03、AirChain01／02。11 个攻击入口加无伤害追击入口；组合内部 Fly01／02／03 和 Fire04 不独立抽签。16 个动作子类与死亡 GA、三份共享流程模板、初始化／冷却 GE 共 37 包；AI 装配 10 包。

BT Face→ChooseBossAction→ActivateAbility→Wait；Controller 使用原生 AAIController Tick 更新 Focus／ControlRotation，CMC 执行转向。没有向前走／跑循环，用户选完整 Dash_F 分段接近：约 1.07 秒／7.11 m，不缩放、裁剪原轨迹；此 Boss 的 BT 不使用 MoveTo。Dash_F 两阶段均可选，650～5000 cm、朝向偏差不超过 15°、无伤害，参数仍由数据资产调节。

## 蓝图流程与原生资源接口

| 输入与触发 | 接口、结果与归还 |
| --- | --- |
| GAS 原激活 | `FGGYGOBossActionToken` 只封装原 Activation，不发行第二生命周期。共享地面／飞行模板在 `PrepareBossAction` 绑定目标并选择是否 `AcquireBossFlight`；具体子类提供 Montage、窗口和数值。 |
| Montage 命中窗 | BP `BossHitWindow` 给出空间基准／弹体瞄准点，`ExecuteBossWindow` 消费原窗口。近战用 Owned Trace；范围／弹体用 Combat 原生查询；GA 构造 GE／Cue，GAS 结算。 |
| 飞行 Montage 完成 | BP `MontageActionFinished`→`ReleaseBossFlight(...,true)`→`WaitBossLanding`。原动作先退出，再归还 Flying；CMC 原生重力下降，ABP 用 Fly_Loop 末帧全身静止姿态等待。 |
| 真实 Landed | BP `BossLanded`→`PlayBossRecoveryPresentation`；LandingRecovery 只播姿态和表现。`RecoveryMontageFinished`→`FinishBossAction`，再回 BT 地面池。超时／目标失效走原取消和清理。 |
| 死亡 | 共享死亡 BP 调 `CancelOtherBossActions`→`StartBossDeathState`→`WaitBossDeathPresentation`→`FinishBossDeath`；ABP 唯一播 Die，不另播死亡 Montage。共享 BossCharacter 死亡结束事件下一帧请求 `UninitAndDestroy`。保留时长来自 Die 资产，不宣称计时器证明 GPU 动画完成。 |

GA 清理先摘原资源批次，再归还原 Task、Motion、Flying 句柄、Query、弹体、持续声音和订阅；每个外调返回后按实际原身份重验，不影响后继。持续声由原 GA 持有的 `AGGYGOBossAudioActor` 承载，原生 ProjectileMovement 推进弹体，Combat 查询物理碰撞；不新增帧调度器。

## 来源、制作与验收边界

75 条有效原片段与 4 条空辅助 FBX 只读。窗口、伤害、权重和阶段是项目设计；JSON 没有原游戏伤害事件／HitBox 参数，不能把人工配置称作原版。FX 根据已查看武器与翼膜贴图配色／轮廓创作；音效已复用 55 个声音包，未知原事件和时点的候选保持未知。

Editor 制作工具生成和读回 BP 图、GE、动画与材质资源，不进入运行时模块。旋转采用具名 Pitch／Yaw／Roll，避免 Python 位置参数顺序把地面 FX 竖起。只保存明确本次创建或授权更新的资产，不 SaveAll。

`Saved/Codex/KevinBossAssembly_20261111/`：R6 正式 BT 同场覆盖 11 攻击入口、两阶段、两条 AirChain 真实落地与胜利，死亡销毁后 5 秒无残留；该轮追击缺口由 R7 单独验证完整 Dash_F 两段各 711.145 cm，玩家静止、双方无伤害。R9 冰圈与破环、Visual_Fire_Air 五能力专项画面已查看，专项不代替自动选招。原失败和玩家阵亡轮次保留。

联机、Cook、性能、设备试听，以及逐时点中断／窗口外零伤害完整矩阵未覆盖。忽略目录的角色与场景素材仍是本地资产；Git 中保留制作脚本／配置和已跟踪场景，不放开素材忽略范围。
