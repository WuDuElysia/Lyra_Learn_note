# GGYGO Content 目录约定

> 这是 Unreal `/Game` 资产目录的当前约定。代码模块目录见 [[模块参考]]；本文件只记录 Content 资产边界、迁移结果和引用验证结论。
>
> 目标工程：`f:\ue_project\GGYGO`
> Unreal 虚拟根：`/Game`
> 核对日期：2026-09-10

## 1. 顶层职责

| 目录 | 归属 | 约定 |
|---|---|---|
| `/Game/System/` | 全局系统 | `GGYGOGameData`、Experience、Teams 等跨角色配置 |
| `/Game/GameplayCues/` | GAS 表现 | GameplayCue 资产统一放这里；`DefaultGame.ini` 已扫描此路径 |
| `/Game/GameplayEffects/` | GAS 效果 | 按限制、冷却、伤害等语义分类，不按蓝图来源目录分类 |
| `/Game/Characters/` | 角色资产 | 角色专属资产按角色聚合；跨角色资产放 `Shared/` |
| `/Game/Input/` | 输入资产 | InputAction、InputMappingContext 和输入配置分层 |
| `/Game/PhysicsMaterials/` | 物理表现（预留） | 带表面/Gameplay Tag 的物理材质；当前尚未创建 |

这套结构按游戏概念组织资产。角色专属内容保留角色边界，便于后续按角色拆成 GameFeature；共享玩法内容不依赖原始导出目录的位置。`/Game/PhysicsMaterials/` 当前由 AssetTools 确认为不存在，只有实际出现物理材质资产时再创建。

## 2. 当前目录结构

```text
/Game/
  System/
    Experiences/
    Teams/
  GameplayCues/
  GameplayEffects/
    Damage/
    Restriction/
    Cooldown/
  Characters/
    Shared/
      Abilities/
        HitReact/
      Cameras/
      Movement/
      AbilitySets/
      Groups/
  Input/
    Actions/
    Mappings/
    Config/
```

上述目录已经由 Unreal Editor 的 AssetTools 创建或确认。目录本身可以先为空；只有出现对应资产时才放入内容。

系统路径与项目配置保持一致：

- `GameplayCueNotifyPaths`：`/Game/GameplayCues`
- `GGYGOGameDataPath`：`/Game/System/GGYGOGameData.GGYGOGameData`
- `UGGYGOGameData` 的 PrimaryAsset 扫描目录：`/Game/System`

## 3. 已确认的资产位置

### 输入

| 资产 | 当前路径 | 说明 |
|---|---|---|
| `IMC_Default` | `/Game/Input/Mappings/IMC_Default` | 默认输入映射 |
| `IMC_MouseLook` | `/Game/Input/Mappings/IMC_MouseLook` | 鼠标视角映射 |
| `IA_*` | `/Game/Input/Actions/` | InputAction 保持原有 Actions 目录 |

`BP_PlayerController` 对两个 IMC 都有引用，`BP_PC_Pyrios` 引用 `IMC_Default`。迁移后反向引用仍指向新路径。

### GameplayEffect

| 语义 | 资产 | 当前路径 |
|---|---|---|
| 限制 | `GE_BlockAll` | `/Game/GameplayEffects/Restriction/GE_BlockAll` |
| 限制 | `GE_BlockCombat` | `/Game/GameplayEffects/Restriction/GE_BlockCombat` |
| 限制 | `GE_BlockMoveOnly` | `/Game/GameplayEffects/Restriction/GE_BlockMoveOnly` |
| 限制 | `GE_Invincible` | `/Game/GameplayEffects/Restriction/GE_Invincible` |
| 冷却 | `GE_CooldownEvade` | `/Game/GameplayEffects/Cooldown/GE_CooldownEvade` |
| 冷却 | `GE_CooldownSkill` | `/Game/GameplayEffects/Cooldown/GE_CooldownSkill` |
| 受伤减速 | `GE_InjuredSlowdown` | `/Game/GameplayEffects/GE_InjuredSlowdown` |

这些 GE 的依赖主要是 `GameplayTags` 和 `GameplayAbilities`；`GE_InjuredSlowdown` 还依赖 `/Script/GGYGO`。核对时 7 个 GE 都没有资产反向引用，因此暂时不再拆分到角色目录。

## 4. 有意保留原位置的资产

以下内容不进行批量重排：

- `/Game/BP/GamePlay/BP_GameMode`
- `/Game/BP/GamePlay/BP_PlayerController`
- `/Game/BP/Character/Player/BP_PC_Pyrios`
- `/Game/Characters/Player/055_kuhara/animation/`
- `/Game/Player/055_kuhara/animation/`
- `/Game/Characters/Player/Pyrios/Animation/`
- `/Game/Model/Miyabi/`
- `/Game/Resourse/`
- `/Game/FoggyStreet/`
- `/Game/__ExternalActors__/`

`BP_GameMode` 被 `BP_PlayerController` 和 `BP_PC_Pyrios` 引用；`BP_PlayerController` 还被 `BP_GameMode` 引用；`BP_PC_Pyrios` 被 `BP_GameMode`、`DA_Pawn_Pyrios`、模型和 `ABP_Pyrios` 引用。入口蓝图需要在专门的引用清理窗口处理，不能与低风险的输入/GE 整理混在一起。

角色的 `Blueprint`、AnimBP、动画和模型继续保留在现有角色/导出目录，避免一次性生成大量 redirector，也避免把外部 Actor、地图和美术源资产带入概念重排。

## 5. 迁移后的核对约束

资产路径调整必须通过 Unreal Editor 的 AssetTools 完成，不直接在文件系统移动 `.uasset`。完成迁移后至少核对：

1. 新路径 `exists` 为真，旧路径 `exists` 为假。
2. `get_referencers` 在新路径上返回预期引用；没有引用的 GE 仍为空。
3. 迁移资产 `load_asset` 成功，保存后 `is_dirty` 为假。
4. 使用 `ObjectRedirector` 类型检查全项目重定向器，并区分本次迁移与既有遗留项。
5. PIE/Simulate PIE 能启动和停止，日志中没有指向本次新路径的加载错误。

本次核对中，9 个迁移资产全部成功加载并保存；两个 IMC 的引用已跟随到新路径；7 个 GE 的引用保持为空；Simulate PIE 成功加载 `/Game/Map/Untitled` 并正常停止。

项目当前仍有与本次目录整理无关的既有日志问题，包括 `ABP_Kuhara` 外部容器、`CharConfigData` 缺失、旧角色 AbilitySystem 找不到 `UGGYGOHealthSet` 等。这些问题不能归因于本次路径迁移。

## 6. 重定向器与版本控制说明

当前全项目可检出的 10 个 `ObjectRedirector` 位于既有角色、模型和动画路径，不包含本次迁移资产的旧路径。当前 MCP AssetTools 没有独立的 `fixup_redirectors` 接口，因此不在本次整理中清理这些遗留项。

本次核对还发现，项目根 `f:\ue_project\GGYGO` 实际是 Git 工作树，并且 Git 索引中仍跟踪部分原始 `Content` 路径；`Source/GGYGO` 子仓库当前干净。提交时应严格区分：

- 文档仓库只提交本文件，不带入已有 canvas、笔记和图片修改。
- 项目根的 Content/资产移动、旧路径删除和新路径出现必须单独审查，不能因为文档提交而误 add。
- `Source/GGYGO` 没有本次代码改动。

角色入口蓝图的后续移动应先重新获取完整引用图，再单独保存、PIE 和提交。
