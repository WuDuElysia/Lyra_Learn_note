# BH3 Kevin DemonBattle：独立资产导入

> 当前阶段：独立模型、Skeleton 和 75 条动画已导入保存；StandBy 已在 Persona 预览。79 条索引中 4 条 FBX 没有动画曲线，未生成 AnimSequence；可导入的 75/75 成功，不代表 79 条全部生成。材质已绑定，具体边界见 [[Character/Kevin_DemonBattle_材质导入]]。
> 入口：[[Animation/结构|Animation 结构]] · [[GGYGO_流程_BH3_DemonBattle导入.canvas|导入流程子图]]

## 范围与资产契约

- 先清点 `F:/AnimeStudio/Exports/BH3` 全目录；实际只导入 `Animator/Kevin/05_BOSS_411_DemonBattle` 及本包必需依赖。其他 Kevin 形态和 Stage 保持源文件状态。
- 目标根目录 `/Game/Characters/Boss/Kevin/DemonBattle/`。Animation 模块负责 `Mesh/` 与 `Animation/`；渲染模块负责 `Materials/`、`Textures/` 和材质接线。
- 模型已保存为 `Mesh/SK_Kevin_DemonBattle`；由该模型新建同目录独立 `SK_Kevin_DemonBattle_Skeleton`，75 条动画均引用这一骨架，导入时逐条回读确认。
- 动画采用 `Animation/AS_<源clip名>`；非法字符替换为下划线并预检名称碰撞，保留完整源路径映射。包括本包索引内的 `BOS_411_Ani_Counter`、`BOSS_340_Ani_Stun`，不以名称差异排除。
- 本轮没有 BossAI、主关卡、ABP、Montage 接线；不复用 Pyrios 骨架，不重定向，不运行 Pyrios 去根脚本。物理资产仅在动画预览或本轮验证确有需要时创建。

## 源文件已确认事实

- BH3 共 931 文件：320 FBX、424 JSON、187 PNG，1,417,389,516 字节。
- DemonBattle 包共 184 文件：1 模型 FBX、79 动画 FBX、79 动画 JSON、8 材质 JSON、1 总索引及 16 PNG。
- `anim_settings.json`：`scaleFactorBakedIntoFbx=100`、`applyRootMotion=false`、`rootMotionBone=Bip001`、Avatar 元数据骨骼数 358。该数不是 UE 最终骨骼数。
- FBX 7300；坐标记录 Y-up，单位 `UnitScaleFactor=1`（厘米）；不再额外乘 100。导入计划为统一比例 1，启用坐标/单位转换，模型和动画使用相同选项。
- 模型有 484 个非网格 Model 节点，根为 `BOSS_411`，`Bip001` 为其子节点；另有 7 个网格节点。79 动画以 LimbNode 占位保留这 7 个网格名称，全部 491 个 Model 的名称和父子关系一致。UE Skeleton 实际回读 491 节点。名为 `Root` 的辅助节点也保留为 `BOSS_411` 子节点，不改写为最顶层根。
- 模型有 8 个材质名，导入阶段禁用材质和贴图自动导入，由渲染模块按实际槽名接线。

## 分阶段实施与验收

1. `AAADocs/Scripts/import_bh3_kevin_demonbattle.py` 已提供 `--stage audit/smoke/all`；未知已有资产停止，脚本自有资产须来源哈希及 Skeleton 一致后跳过。已存 StandBy 在批量阶段通过此复用校验；整批重复执行尚未另跑一轮。
2. 模型和 `BOSS_411_Ani_StandBy` 先导入并预览：人物直立、武器/翼部可见，无明显蒙皮爆开。Mesh 参考包围盒约 399.56 × 460.36 × 335.91 cm（包含武器/翼部，不是人体身高）。使用默认 FBX 坐标转换，未人为增加朝向旋转；玩法前向尚未接线验收。
3. StandBy：60 fps、132 个采样键、131 帧区间、2.1833334 秒。源 FBX 比 JSON stopTime 多一帧，保留 FBX 导出区间，未按 JSON 裁剪。其余动画按各自 JSON 采样率导入；`Fly_Attack_03_AS/Summon/Between/Shoot/BS` 为 30 fps，其余 60 fps。
4. 批量新增 74 条，复核跳过已存 StandBy，合计 75 条；无层级不匹配。`BOS_411_Ani_Counter`、`BOSS_340_Ani_Stun` 已成功导入。`Wing_Defence_Break(80-200)` 映射为 `AS_BOSS_411_Ani_Wing_Defence_Break_80_200_`。
5. 导入结束时磁盘有 77 个资产：1 SkeletalMesh、1 Skeleton、75 AnimSequence。8 材质槽保持来源名称，未自动制作材质/贴图，未创建 PhysicsAsset。每条动画回读根运动提取和强制锁根均为 false，保留源变换。

## 4 条源空片段与导入提示

| 索引名 | 实际源数据与处理 |
| --- | --- |
| `BOSS_411_Ani_Add_HitBox` | FBX 无 AnimationCurve，只有骨架和 1 秒空 Take；未生成动画。 |
| `BOSS_411_Ani_Normal_HitBox` | 同上。 |
| `BOSS_411_Ani_No_Weapon` | 同上。 |
| `BOSS_411_Ani_Have_Weapon` | 同上。 |

初次批量中 UE 对上述四项返回“未找到网格体或动画轨道”，原始执行报告保留这 4 个失败。最终分类报告结合源审计将它们明确归为 `source_empty`；脚本已增加导入前空曲线检查。没有合成替代动画，也没有把空片段记成骨架不兼容。

模型日志提示无平滑组、部分骨骼缺失绑定姿势；UE 随后报告自动重建绑定姿势成功。StandBy 预览通过，未把这条提示等同于全动作/渲染质量验收。

## 报告与渲染交接

本地项目 `Saved/Codex/`：`bh3_demonbattle_source_audit.json`、`bh3_demonbattle_smoke_report.json`、`bh3_demonbattle_all_report.json`、`bh3_demonbattle_final_report.json`。原始批量报告 `complete=false` 表示索引未全部生成；最终报告 `importable_complete=true`、`index_fully_imported=false`。项目核对说明见 `AAADocs/BH3_Kevin_DemonBattle_Import.md`。

独立只读核验已落盘 `bh3_demonbattle_verification.json`：75 条动画共用骨架、491 节点、原始根运动标记保持关闭；加入材质与纹理后全目录 104 资产。`.uasset` 遵循现有 `Content/Characters` Git 忽略约定，本机保存不等于已经上传；跨机器复现需要源文件和项目导入脚本。

8 槽依次为 `BOSS_411_Weapon_Blade_Material`、`BOSS_411_Material_Face`、`BOSS_411_Material_Eye`、`BOSS_411_Material_Hair`、`BOSS_411_Material_Wing`、`BOSS_411_Material_Body02`、`BOSS_411_Material_Wings_Blue`、`BOSS_411_Material_Body01`。渲染模块据此配置材质，不依赖猜测的槽位名称。

## 当前缺口

- 除 StandBy 外，75 条动画尚未逐条进行视觉验收；游戏内前向、Root Motion 消费及 Boss 行为尚未接线。
- Unity 的 loop、root-motion JSON 参数只作为来源信息；不能直接等同 UE Root Motion 设置。已禁用 UE 根运动提取并保留原始变换轨道，不改写为玩法位移；loopTime 保存为来源元数据，未赋予状态机循环策略。
- 材质、物理行为及运行时 Boss 接线不属于本轮 Animation 已实现状态。
