# Kevin DemonBattle：战斗动画实施

> 当前：26 份派生动画、ABP、19 个技能 Montage 与 5 个表现 Montage 已保存并独立读回；正式单机战斗和受影响画面通过。[[Animation/结构]] · [[Animation/资产生产安全]] · [[GGYGO_流程_Kevin战斗动画.canvas]] · [[BossAI/计划_Kevin_DemonBattle战斗接入]]

## 来源与制作契约

75 条有效原片段和 4 条空辅助 FBX 保持只读。配置、动作清单和来源审计在项目 `AAADocs/Assets/BH3/KevinDemonBattle/`；没有原游戏 HitWindow／伤害事件，窗口与组合是项目人工设计。Weapon 刚性绑定 Bip001-Prop1，实际武器端点由原导入网格几何核对，不能从辅助 HitBox 名称推导判定。

`build_kevin_skill_animation.py` 经 Editor 的 KevinMotionAssetBuilder／KevinCombatAssetBuilder／KevinBossAnimBlueprintAuthor 制作。派生序列根轨迹原地化，完整累计 XYZ 单独交 CMC 执行，Montage 禁用 RootMotion，避免双重位移。多段 AirChain 保留原片段顺序与完整轨迹，不把各段独立当落地动作，不清零技能 Z。Slot、Section、GameplayEventWindow、FX／声音窗口与作者配置严格读回。

用户确认 Fly_Loop 原片段约 0.033 秒且根位置跳变约 186 cm，不能循环等待。派生 `presentation_last_frame` 复制最后一帧所有骨骼的平移／旋转／缩放，只有该纯表现姿态的角色位移曲线归零；原素材和技能轨迹不改。ABP 在原技能 Montage 退出、角色实际 Falling 时使用此姿态。

## 运行接缝

| 触发 | 实际结果 |
| --- | --- |
| GA 请求 Montage | 姿态／GameplayEventWindow／表现窗口由 Animation 发布，CMC 按对应原累计曲线移动胶囊；Combat 只查询，GA／GE 结算。 |
| 飞行组合结束 | GA 归还 Flying 模式；CMC 原生重力下降，ABP 使用末帧等待姿态；不合成 Z 或固定下降速度。 |
| 真实 Landed | 共享 BP 才请求 LandingRecovery（Main 2.45 s，Impact 0～0.35 s）；它只作表现，不执行位移，也不证明或代替落地。 |
| Health 死亡状态 | ABP 唯一播放 Die；死亡 GA 管资源和基于实际序列长度的保留时长，结束后共享 BossCharacter 处理销毁。 |
| 远距离接近 | 完整 Dash_F Montage 经 GAS→CMC 分段追击；不存在伪造向前走／跑循环。 |

已有且干净的自有资产严格读回；未知资产、脏包或不符的作者配置明确失败。已有 ABP 图不重建，不覆盖人工表现；本次明确授权的末帧派生、模板图及旋转字段更新只保存实际目标包，不 SaveAll。两项 KevinMotionOutputGuard／KevinPreserveAuthoredGraph 回归通过。

## 验证

项目 `Saved/Codex/KevinBossAssembly_20261111/Animation_Read/editor.log` 验证原轨迹、姿态、窗口和声音引用；R6 正式战斗包含两条 AirChain 原生落地；R7 验证完整 Dash_F 约 711.145 cm。R9 和 Visual_Fire_Air 的主要冰火／空中画面已核对。完整战斗与隔离表现复拍分别记录，不把资产读回当运行证据。

未覆盖 Kevin 联机、Cook、性能、逐时点中断完整矩阵或设备试听。素材与部分派生资产所在忽略目录只在本地交付。
