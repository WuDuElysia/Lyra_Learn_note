# Kevin DemonBattle：材质导入与预览

> 本页只记录 `F:/AnimeStudio/Exports/BH3/Animator/Kevin/05_BOSS_411_DemonBattle` 的材质基础预览。16 张贴图、3 个预览主材质和 8 个材质实例已导入 UE，并绑定到 Kevin DemonBattle 骨骼网格体。角色架构见 [[Character/结构]]，Pyrios 的独立渲染实现见 [[Character/渲染实现]]。

## 来源与接线

`BOSS_411.fbx` 中可读到 8 个材质名，和 `Materials/*.json` 一一对应。8 份 JSON 实际引用 16 张 PNG，全部能在同一个导出目录找到。动画导入已生成 `/Game/Characters/Boss/Kevin/DemonBattle/Mesh/SK_Kevin_DemonBattle`，其 8 个原名槽位均已绑定到 `/Materials/MI_<槽位名>`。脚本 `F:/ue_project/GGYGO/AAADocs/Scripts/setup_bh3_kevin_materials.py` 只处理这张明确路径的 Mesh，先核对槽位，再导入贴图到 `Textures`、建立预览材质到 `Materials`。未找到 Mesh 或槽位时不会开始修改资产。源 JSON/PNG 没有改写。

| FBX 槽位 / JSON | `_MainTex` | 预览类型 | 其他已引用数据 |
| --- | --- | --- | --- |
| `BOSS_411_Material_Body01` | `Monster_Boss_411_Texture_Body_Color` | Opaque | `_LightMapTex` = `...Body_Lightmap` |
| `BOSS_411_Material_Body02` | `Monster_Boss_411_Texture_Body_Wings_Color` | Masked | `_LightMapTex` = `...Body_Wings_LightMap` |
| `BOSS_411_Material_Eye` | `Monster_Boss_411_Texture_Eye` | Masked | `_EyeEffectTex` = `Avatar_Bronya_C9_Eye_Color02` |
| `BOSS_411_Material_Face` | `Monster_Boss_411_Texture_Face_Color` | Opaque | `_FaceMapTex` = `NPC_Kevin_FaceMap`；`_LightMapTex` = `...Face_Lightmap` |
| `BOSS_411_Material_Hair` | `Monster_Boss_411_Texture_Hair_Color` | Masked | `_LightMapTex` = `...Hair_Lightmap` |
| `BOSS_411_Material_Wing` | `Monster_Boss_411_Texture_Body_Wings_Color` | Masked | `_LightMapTex` = `...Body_Wings_LightMap` |
| `BOSS_411_Material_Wings_Blue` | `Monster_Boss_411_Texture_Body_Wings_Blue_Color` | Translucent / Unlit | `_LightMapTex` = `...Body_Wings_Blue_LightMap` |
| `BOSS_411_Weapon_Blade_Material` | `Monster_Boss_411_Texture_Weapon_Color` | Opaque | `_SPTex` = `b1`；`_SPNoiseTex` = `noise_rough` |

预览材质使用三个 UE 主材质：`M_Kevin_Preview_Opaque`、`M_Kevin_Preview_Masked`、`M_Kevin_Preview_Translucent`。每槽一个 `MI_<原材质名>`。Opaque/Masked 用 UE Default Lit 只展示基础色与可调粗糙度；Masked/Translucent 使用 `_MainTex.A`。蓝色翼片的 Alpha 在贴图中普遍低于 1，故单独用双面半透明 Unlit 预览。普通翼片有透明裁切区域，眼图有透明背景；Hair 的 Alpha 变化较小，但预览中保留 Masked 通道。源 `Wing` JSON 的 `_AlphaClip=1` 与这项选择一致；`Wings_Blue` 的 `_AlphaClip=0`。

## 数据边界和预览结果

导出名为 `_LightMapTex` 的图片是原 BH3 角色 Shader 的逐通道输入，尚未确认每个通道的语义。脚本将其及 `_FaceMapTex`、武器 SP 纹理以线性 BC7 导入，保留通道和 Alpha，但没有把它们当作 UE 烘焙光照、法线、金属度或粗糙度接入。`_EyeEffectTex`、武器的 `_SPTex` / `_SPNoiseTex`、特效/阴影/描边等原版逻辑也暂不驱动基础预览。武器 JSON 有较高的 Emission/Transition 标量，但缺少运行时激活状态，预览中没有让整把武器常亮。

这是方便检查 Mesh、UV、材质槽和透明部位的**基线预览**，不是崩坏3 Toon Shader 等价还原。资产编辑器中可见白色头发与衣片、黑色翼骨、青色翼膜和武器贴图；画面中仍缺少原作光照、描边、眼/面阴影、高光与 FX。蓝翼透明排序及翼片裁切阈值仍需进一步判断。无光照模式截图 `F:/ue_project/GGYGO/Saved/Codex/bh3_kevin_material_preview.png` 用于检查贴图、几何与透明区域；光照模式截图 `F:/ue_project/GGYGO/Saved/Codex/bh3_kevin_material_preview_lit.png` 显示 UE Default Lit 材质在资产预览环境下可接受光照并投影。两张截图均不能证明已匹配原作的照明。脚本只绑定 `/Game/Characters/Boss/Kevin/DemonBattle/Mesh/SK_Kevin_DemonBattle`，不会扫描或修改同目录的其他 Mesh。没有修改 Pyrios 材质、主关卡或 BossAI 接线。

## 当前状态

- **已核对**：8 个 JSON 与 FBX 材质名一致；16 个被引用的 PNG 均存在。PNG Alpha 抽样显示普通翼片有透明空区、蓝翼大面积半透明、眼图有透明背景。
- **已导入并保存**：16 张贴图、3 个预览主材质、8 个材质实例。UE 日志含 `BH3_KEVIN_PREVIEW_OK textures=16 materials=8 meshes=1`；本次导入后的日志区间没有材质或 Shader 编译错误。MCP 回读 8 个 Mesh 槽位，全部指向对应 `MI_<槽位名>`；8 个实例的 `MainTex` 也全部回读到对应源贴图。
- **已目视检查**：`SK_Kevin_DemonBattle` 资产编辑器无光照及光照模式截图均已保存。基础颜色与透明部位可见；光照模式下可见阴影和材质受光。预览可用不等于原版 Shader 的观感已还原。
- **动画只读核验**：`F:/ue_project/GGYGO/Saved/Codex/bh3_demonbattle_verification.json` 报告 75 条动画、491 根骨骼、26,295 个网格顶点、8 个材质槽，动画共享同一 Skeleton；另有 4 个源 FBX 没有动画曲线。资产目录总计 104 项，含动画、Mesh、材质和纹理等。
