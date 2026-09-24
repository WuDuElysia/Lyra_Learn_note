# GameFeature 计划

> 返回 [[计划蓝图]] · 当前代码事实见 [[模块参考]]

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

**前置项：先补激活等待，再写 Action。**
见 11.3 末尾。没有等待机制时，插件的 Action 可能在角色生成之后才执行完，
症状是"技能偶发缺失"，且只在慢盘或大插件上出现。
这一项与角色数量无关，是插件化的第一步而不是收尾。

### 11.3 第一步的落地结构

```
GameModes/
  GGYGOExperienceDefinition.h/.cpp      一局的玩法定义：GameFeature 列表 + 队伍成员
  GGYGOGameMode.h/.cpp                  按 Experience 装配，接管角色生成
Character/
  Components/GGYGOPawnExtensionComponent.h/.cpp   InitState 协调
  Data/GGYGOPawnData.h/.cpp                       角色定义 DataAsset
Teams/
  GGYGOSquadComponent.h/.cpp            队伍成员与出战切换（挂 PlayerState）
```

**激活在 `InitGame` 发起，装配等它完成。** `AGGYGOGameMode` 用一个计数
（`PendingGameFeatureCount`）跟踪未完成的插件数：`HandleStartingNewPlayer` 在计数非零时
不生成队伍，等最后一个插件回调把计数归零，再遍历当前所有 PlayerController 补做装配。

不做等待的话，插件里的 `GameFeatureAction`（往角色类注入组件、授予能力）
可能在角色生成之后才执行完，角色就少了那部分内容且不报错。
插件列表为空时计数恒为零，装配路径与没有这套机制时完全一致。

两个必须处理对的细节：

- **计数在发起前递增**。插件此前已加载时回调会同步触发，先调用后递增会把计数打到负数。
- **加载失败也要放行**。失败只报错、照常递减，否则一个装不上的插件会让所有玩家
  永远停在等待里 —— "进游戏没有角色"比"缺某个插件的内容"严重得多。

补做装配时遍历 `GetPlayerControllerIterator` 而不是维护一份等待名单：
等待期间玩家可能断线，名单里会留下悬垂指针。已有队伍的玩家按
`IsSquadAssembled()` 跳过，避免重复装配出两套位置。

**还有一个 GameFeature 的前提在 AssetManager 上**：`DefaultGame.ini` 必须注册
`GameFeatureData` 这个 PrimaryAssetType，否则 GameFeatures 子系统在启动阶段就报
"Asset manager settings do not include a rule for assets of type GameFeatureData"，
插件功能整体不可用。它的 `Directories` 留空即可 —— 那些资产在各插件自己的根目录下，
由插件系统发现，不需要 AssetManager 去 `/Game` 里扫。

---
