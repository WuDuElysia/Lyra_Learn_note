---
name: lyra-obsidian-notes
description: 为 LyraStarterGame 创建和维护 Obsidian 学习笔记；适用于记录 Lyra/GAS 的源码链路、输入映射、Ability 生命周期，并同时生成 Markdown 与 Canvas 可视化笔记。
---

# Lyra Obsidian 笔记 Skill

## 目标

把 Lyra 源码分析整理成简洁、可复习、可追溯的 Obsidian 笔记。默认使用当前笔记仓库根目录；如果用户指定其他目录，以用户指定目录为准。

## 输出规则

1. 对系统链路类主题默认生成一对同名文件：
   - `<主题>.md`：概念、调用链、关键代码、边界条件。
   - `<主题>.canvas`：用节点和箭头表达调用链。
2. 写入目标目录前，先检查目录和同名文件；已有文件不得直接覆盖，除非用户明确要求更新。
3. 使用 UTF-8 编码；文件名简短、稳定，避免把临时日期放进文件名。
4. Markdown 只保留能帮助复习的内容，不复制大段源码；源码引用使用项目相对路径、类名和函数名。
5. Canvas 使用 Obsidian Canvas JSON 格式，节点使用稳定的短 id；边必须通过 `fromNode` 和 `toNode` 连接已有节点。优先使用 text 节点，只有确定 Vault 相对路径时才使用 file 节点。
6. Canvas 只表达“源码文件/函数 → 下一个源码文件/函数”的主链路，不复述 Markdown 中的完整概念解释。

## 内容组织

系统链路笔记至少包含：

- 一句话结论。
- 一条从输入/事件到最终行为的主链路。
- “GAS 原生”与“Lyra 自定义”对照表。
- 第一次触发、重复触发、释放或取消等关键分支。
- 对容易混淆的概念做一句区分，例如 Input Release 不等于 EndAbility。
- 相关源码路径和关键函数。

## GAS 与 Lyra 的标注方式

每个关键节点或段落都要明确归类：

- `GAS 原生`：引擎提供的数据结构、ASC API、AbilitySpec/Handle、PredictionKey、AbilityTask 和通用 replicated event 机制。
- `Lyra 自定义`：Lyra 的 InputConfig、AbilitySet 输入 Tag、输入回调、输入 Handle 缓存、ProcessAbilityInput、激活策略以及对 GAS API 的组合/重载。
- `UE 输入框架`：Enhanced Input 的 InputMappingContext、UInputAction、Triggered/Completed 等，不把它误称为 GAS 功能。

## Lyra 输入笔记的固定检查点

分析 Lyra 输入时依次确认：

1. 物理按键如何映射到 `UInputAction`。
2. `ULyraInputConfig` 如何把 `UInputAction` 映射到 `FGameplayTag`。
3. `ULyraAbilitySet::GiveToAbilitySystem` 如何把输入 Tag 写入 `FGameplayAbilitySpec::GetDynamicSpecSourceTags()`。
4. `AbilityInputTagPressed/Released` 如何按 Tag 精确扫描 Spec，并缓存 `FGameplayAbilitySpecHandle`。
5. `ProcessAbilityInput` 如何用 Handle 找回 Spec，并区分未激活与已激活分支。
6. `AbilitySpecInputPressed/Released` 如何取得激活实例的 PredictionKey 并调用 `InvokeReplicatedEvent`。
7. `WaitInputPress/WaitInputRelease` 消费的是通用输入事件，不是输入 Tag；没有等待任务时，事件不会自动改变 Ability 行为。
8. 释放输入与 `EndAbility`、取消 Ability 的关系必须单独说明。

## Canvas 建议布局

Canvas 使用简洁的函数链路图：

```text
源码文件 + 函数
  → 源码文件 + 函数
  → 源码文件 + 函数
```

每个节点只写三项：

```text
[归属：GAS 原生 / Lyra 自定义 / UE 输入框架]
文件：Source/.../Example.cpp
函数：ExampleFunction
作用：一句话说明该函数在本链路中的职责
```

绘图规则：

1. 默认只保留 6～10 个关键函数节点；优先保留真正发生调用或状态转移的函数。
2. 一个节点可以包含同一文件中的相关函数，但不要把所有成员关系、字段列表、生命周期解释都画进图里。
3. 分支只保留用户必须理解的分支，例如“未激活→TryActivateAbility”和“已激活→InvokeReplicatedEvent”；详细条件写在 Markdown。
4. 不添加单独的复杂图例、对象关系全景图或重复 Markdown 的长段落；分类直接写在节点标题中。
5. 边只标注简短动词，例如“调用”“转发”“匹配”“刷新”“解绑”。
6. Markdown 负责详细解释，Canvas 负责让读者快速定位源码文件、函数和调用顺序。
