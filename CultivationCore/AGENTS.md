# AGENTS.md — CultivationCore（仙途·核心）

## 项目概述

- **类型**：RimWorld 模组（当前阶段纯 XML/贴图，后续将加入 C# 汇编）
- **目标版本**：RimWorld 1.6
- **packageId**：`TheBook.RimCultivation.CultivationCore`
- **主题**：中式修仙/修真，作为后续子模组的基础核心模组
- **贴图来源**：AI 生成（通义）

## 目录结构

```
CultivationCore/
├── About/           # 模组元数据 (About.xml, ModIcon.png, Preview.png)
├── Defs/            # XML 定义文件
│   ├── ThingDefs/   # ThingDef（建筑、物品）
│   ├── HediffDefs/  # 状态效果（修为、境界）
│   ├── WorkTypeDefs/# 工作类型
│   ├── RecipeDefs/  # 配方（炼丹、炼器）
│   └── ...          # 按需创建子目录
├── Textures/        # 贴图资源，路径需与 texPath 匹配
└── Source/          # C# 源码（预留，后续加入）
```

## 开发约定

### Def 命名规范
- **defName**：英文字母 PascalCase（如 `MeditationCushion`）
- **label**：中文显示名（如 `蒲团`）
- **texPath**：相对于 `Textures/` 的路径，不含扩展名（如 `Things/Building/MeditationCushion`）

### 文件命名
- Def 文件按类别存放，如 `Defs/ThingDefs/Buildings_Furniture.xml`
- 文件名使用英文下划线命名法，反映类别和内容

### 模组继承
- 作为核心模组，其他子模组通过 `packageId` 声明依赖
- 变更时需注意向后兼容，避免破坏依赖模组

## 关键路径对应关系

| 定义中的字段 | 实际文件/目录 |
|---|---|
| `<texPath>Things/Building/Foo</texPath>` | `Textures/Things/Building/Foo.png` |
| ThingDef 定义 | `Defs/ThingDefs/*.xml` |
| HediffDef 等 | `Defs/HediffDefs/*.xml` |

## 注意事项

- **当前阶段无编译**：纯 XML 模组，修改后重启 RimWorld 即可生效
- **仓库根目录**：Git 仓库在 `Mods/` 层，提交时注意路径（`git add CultivationCore/...`）
- **贴图路径**：RimWorld 对路径大小写敏感，需与 `texPath` 严格一致
- **XML 标签**：必须保留 `<Defs>` 根元素包裹所有定义
- **XML 注释**：使用 `<!-- -->` 语法

---

## 开发路线

### MVP 三阶段

| 阶段 | 内容 | 技术手段 |
|---|---|---|
| 0.1 | 基础境界（炼气→筑基→金丹）、打坐垫、修炼工作 | XML Defs + Hediff |
| 0.2 | 简化灵根（五行）、基础丹药（聚气丹/筑基丹）、灵石 | XML + RecipeDef |
| 0.3 | 基础神通（火球术/金刚罩）、功法接口 | C# Comp + Verb |

### 后续模块化架构

核心框架（CultivationCore）提供基础 API，每本小说作为独立子模组扩展：

```
CultivationCore/          # 核心：境界、灵根、灵力、事件API
├── 凡人修仙传/           # 资源争夺 + 稳健发育
├── 玄鉴仙族/             # 血脉传承 + 家族经营
├── 仙葫/                 # 洞天法宝 + 本命法宝
└── ...                   # 其他小说模组
```

子模组通过 `CultivationAPI` 注册功法和事件，核心通过事件系统（`CultivationEvents.OnRealmBreakthrough` 等）通知子模组。

### RimWorld C# 开发要点

- **Hediff 系统**：用 `HediffWithComps` 跟踪修为/境界/灵根状态（非自定义类）
- **ThingComp**：修仙行为用 `CompCultivator` 等组件附加到 Pawn 上
- **WorkGiver + JobDriver**：修炼等新工作的标准实现路径
- **Harmony Patch**：需要修改原版行为时使用，避免直接修改基类
- **IExposable**：所有自定义数据必须正确实现，确保存档兼容
- **Verb 系统**：神通技能通过扩展 Verb 或 Projectile 实现

---

## 参考链接

- [RimWorld Modding Tutorials](https://rimworldwiki.com/wiki/Modding_Tutorials)
- [RimWorld Modding 基础教程](https://blog.csdn.net/qq_29799917/article/details/104692076)
- [泰南官方贴图资源](https://www.dropbox.com/scl/fo/fi49a1wj4iaao72ndhlp9/AK4AoWZg0WCSeRKbZ1WY6kI?rlkey=6zgtptylc7cigr139m75z741m&e=2&dl=0%7COfficial)
- [Modding 电话簿](https://spdskatr.github.io/RWModdingResources/telephonebook)
- [Ludeon 官方 Mod 论坛](https://ludeon.com/forums/index.php?board=12.0)

---

## RimWorld 源码架构速览

### 核心入口（理解游戏基础）
- **`Game.cs`**：游戏主类，单例 `Current.Game`，管理核心组件（TickManager、Storyteller、ResearchManager 等）
- **`World.cs`**：世界/星球管理
- **`Map.cs`**：地图管理，提供 `listerThings` 等查询接口

### 实体体系（理解游戏对象）
- **`Thing.cs`**：所有物体基类（物品/建筑/Pawn），实现 `ISelectable`、`IExposable`、`ILoadReferenceable`
- **`Pawn.cs`**：角色基类，通过 Tracker（Health/Job/Equipment/Inventory）组件化管理状态
- **`Building_WorkTable.cs`**：工作台基类

### 核心机制
- **`TickManager.cs`**：游戏 tick 调度（Tick/Rare/Long 三级频率）
- **`Storyteller.cs`**：故事讲述者/事件驱动
- **`IncidentWorker.cs`**：事件基类

### 工具与模式
- **Def 系统**：所有游戏数据通过 XML Def 定义，运行时通过 `DefDatabase<T>.GetNamed()` 查询
- **Comp 系统**：`ThingComp` 附加行为组件，是 Mod 最常用的扩展点
- **`DefOf` 模式**：`[DefOf]` 注解自动注入常用 Def 引用

### 学习路径建议
1. 先理解 `Thing` / `Pawn` / `ThingComp` — 这是 Mod 开发的核心
2. 再看 `WorkGiver` / `JobDriver` — 新增工作类型的关键
3. 需要修改原版时看 Harmony Patch 示例
