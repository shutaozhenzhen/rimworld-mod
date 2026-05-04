# 逐步开发计划

从纯 XML 模组到完整 C# 修仙框架的分阶段实施路线。

---

## 阶段 0.1：基础境界与修炼（纯 XML）

**目标**：验证核心循环——修炼→积累→突破，全部由原版系统支撑。

### 内容

| 功能 | 实现方式 | 文件位置 |
|---|---|---|
| 炼气/筑基/金丹三境界 | HediffDef（3 个 stage） | `Defs/HediffDefs/Realms.xml` |
| 修炼工作类型 | WorkTypeDef + WorkGiverDef | `Defs/WorkTypeDefs/Cultivation.xml` |
| 打坐垫建筑 | ThingDef（可坐，美工 2） | `Defs/ThingDefs/Buildings_Furniture.xml`（已有） |
| 境界提升配方 | RecipeDef（消耗时间+修为值） | `Defs/RecipeDefs/Breakthrough.xml` |
| 贴图 | PNG，AI 生成（通义） | `Textures/Things/Building/MeditationCushion.png`（已有） |

### 验收标准
- [ ] 小人可找到打坐垫并执行修炼工作
- [ ] 修炼后获得 Hediff（修为进度条可见）
- [ ] 修为满后可手动触发境界突破（通过 Recipe/Bill 系统）
- [ ] 境界 Hediff 正确显示中文标签和图标
- [ ] 存档/读档后境界和修为不丢失

### 技术要点
- Hediff 使用 `HediffWithComps`，stage 对应境界
- 境界提升通过 `HediffComp_Disappears` + 新 Hediff 叠加实现
- 修炼进度用 severity（0~1）表示当前境界百分比

---

## 阶段 0.2：灵根与丹药（XML + RecipeDef）

**目标**：引入差异化（灵根）和资源循环（采集→炼丹→服用）。

### 内容

| 功能 | 实现方式 | 文件位置 |
|---|---|---|
| 五行灵根（金木水火土） | HediffDef（开局随机或固定） | `Defs/HediffDefs/SpiritRoots.xml` |
| 灵石矿脉 | ThingDef（可开采矿石） | `Defs/ThingDefs/Items_Resources.xml` |
| 灵石 | ThingDef（堆叠资源） | `Defs/ThingDefs/Items_Resources.xml` |
| 聚气丹 | ThingDef + RecipeDef + IngestionOutcomeDoer | `Defs/ThingDefs/Items_Pills.xml`, `Defs/RecipeDefs/Alchemy.xml` |
| 筑基丹 | 同上，稀有材料需求 | 同上 |
| 简易丹炉 | ThingDef（工作台） | `Defs/ThingDefs/Buildings_Production.xml` |
| 灵草（1-2 种） | PlantDef（可种植/野生） | `Defs/ThingDefs/Plants_Cultivation.xml` |
| 丹炉贴图、灵石贴图、丹药贴图 | AI 生成 | `Textures/Things/`, `Textures/Items/` |

### 验收标准
- [ ] 开局小人随机获得灵根 Hediff
- [ ] 灵根影响修炼速度（通过 statOffset 或 hediffComp）
- [ ] 地图生成灵石矿脉
- [ ] 丹炉可制作聚气丹（消耗灵草+灵石）
- [ ] 服用聚气丹加速修为增长
- [ ] 筑基丹作为突破前置条件

### 技术要点
- 灵根对修炼速度的影响通过 `HediffCompProperties_StatFactor` 实现
- 丹药效果用 `IngestionOutcomeDoer_GiveHediff` 赋予临时修炼加速
- 丹炉配方使用 `researchPrerequisite` 锁定在早期科技后

---

## 阶段 0.3：神通与功法接口（C# 起步）

**目标**：第一个 C# 汇编，实现基础战斗神通和功法系统接口。

### 内容

| 功能 | 实现方式 | 技术难点 |
|---|---|---|
| 火球术 | Verb_CastAbility 扩展 | Verb 系统，Projectile 定义 |
| 金刚罩 | HediffComp（临时护盾） | Hediff 叠加 + 伤害拦截 |
| 灵力值 | HediffComp_SpiritEnergy（自定义） | 自定义 Comp |
| 神通解锁面板 | Gizmo + Command_Ability | UI 交互 |
| 功法接口 `ICultivationMethod` | C# 接口定义 | 扩展性设计 |
| 基础功法 2-3 种 | 实现 ICultivationMethod | 策略模式 |
| 修炼工作改为 C# | WorkGiver_Cultivate + JobDriver_Cultivate | Job 系统 |

### C# 项目结构
```
Source/
├── CultivationCore.csproj
├── Comps/
│   ├── CompCultivator.cs          # 修仙者核心组件
│   └── CompProperties_Cultivator.cs
├── HediffComps/
│   ├── HediffComp_SpiritEnergy.cs  # 灵力值
│   └── HediffComp_Realm.cs         # 境界状态
├── Verbs/
│   └── Verb_CastFireball.cs        # 火球术
├── WorkGivers/
│   └── WorkGiver_Cultivate.cs      # 修炼工作
├── JobDrivers/
│   └── JobDriver_Cultivate.cs      # 修炼行为
├── Interfaces/
│   └── ICultivationMethod.cs       # 功法接口
└── Utilities/
    └── CultivationUtility.cs       # 工具函数
```

### 验收标准
- [ ] 火球术可对目标释放并造成伤害
- [ ] 金刚罩可吸收接下来 N 次伤害
- [ ] 灵力值 UI 显示并随修炼恢复
- [ ] 功法接口可被外部 Mod 实现
- [ ] 存档兼容（IExposable 正确实现）

### 技术要点
- **不要自定义 Pawn 类**，用 `ThingComp` 附加修仙行为
- 神通用 `Verb` 系统，参考原版 `Verb_CastAbility`
- 功法接口用 C# `interface`，子模组通过 `[DefOf]` 注册
- 所有自定义 Comp 必须实现 `IExposable`

---

## 阶段 0.4：丹符阵器基础（C# 深化）

| 功能 | 关键点 |
|---|---|
| 炼丹品质系统 | 凡品/灵品/仙品，成功率+品质随机 |
| 符箓（一次性消耗） | ThingDef + Verb（使用时触发） |
| 聚灵阵 | Building + Comp（范围 buff） |
| 法器（飞剑） | ThingWithComps + Verb_MeleeAttack 扩展 |
| 丹毒/耐药系统 | Hediff 叠加 + 衰减 |

---

## 阶段 0.5+：模块化扩展

每个小说子模组基于 CultivationCore API 独立开发：

| 子模组 | 特色系统 |
|---|---|
| 凡人修仙传 | 掌天瓶（灵植催熟）、虚天殿事件链、稳健发育曲线 |
| 玄鉴仙族 | 家族血脉传承、族谱系统、家族兴衰事件 |
| 仙葫 | 洞天法宝（小地图）、本命法宝成长、先天神禁 |
| 衡华 | 悟道推演、功法改良、知识树 |
| 灭运图录 | 道心选择、业力追踪、因果系统 |

---

## 版本与里程碑

| 版本 | 类型 | 关键交付 |
|---|---|---|
| 0.1.0 | XML | 境界+修炼+打坐垫 |
| 0.2.0 | XML | 灵根+丹药+灵石+灵草 |
| 0.3.0 | C# | 神通+功法接口+灵力值 |
| 0.4.0 | C# | 丹符阵器基础 |
| 1.0.0 | 完整 | 核心框架发布，支持子模组开发 |

---

## 风险与应对

| 风险 | 应对 |
|---|---|
| 纯 XML 阶段难以实现复杂逻辑 | 用 RecipeDef/IngestionOutcomeDoer/HediffComp 组合 |
| C# 初期学习曲线 | 先参考原版 `WorkGiver`/`JobDriver` 实现 |
| 存档兼容性 | 每个阶段新建存档测试，不覆盖旧档 |
| 与其他 Mod 冲突 | 使用独特 defName 前缀（如 `CC_`），Harmony patch 加条件判断 |
