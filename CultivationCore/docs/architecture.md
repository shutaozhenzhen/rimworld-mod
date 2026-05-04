# 架构设计文档

## 设计原则

- **模块化**：每个系统独立可插拔，核心与子模组通过接口解耦
- **事件驱动**：系统间通过事件通信，避免直接依赖
- **配置驱动**：尽可能用 XML Def 定义，减少硬编码
- **原生扩展**：优先使用 RimWorld 原生系统（Hediff/Comp/Verb），避免过度自定义

---

## 分层架构

```
┌─────────────────────────────────────────────┐
│                 子模组层                      │
│  凡人修仙传  玄鉴仙族  仙葫  衡华  灭运图录  ... │
├─────────────────────────────────────────────┤
│                  API 层                       │
│  CultivationAPI  ICultivationMethod          │
│  CultivationEvents  扩展注册                  │
├─────────────────────────────────────────────┤
│                 核心系统层                     │
│  境界  灵根  灵力  功法  炼丹  炼器  阵法  灵植 │
├─────────────────────────────────────────────┤
│                 基础框架层                     │
│  Hediff  Comp  Verb  WorkGiver  JobDriver    │
├─────────────────────────────────────────────┤
│                RimWorld 原版                   │
│  Thing  Pawn  Map  Game  TickManager         │
└─────────────────────────────────────────────┘
```

---

## 核心系统设计

### 1. 境界系统（Realm）

**职责**：管理修仙者境界等级、提供属性加成、控制神通解锁。

```
Realm → HediffDef（stage 对应境界）
     ↓
HediffComp_Realm  // 自定义 Comp，管理境界数据
     ↓ 属性
 - RealmLevel: int    // 境界等级 0=凡人, 1=炼气, 2=筑基...
 - StageProgress: float // 当前境界修炼进度 0~1
 - BreakthroughHistory: List<BreakthroughRecord>
```

**数据流**：
```
修炼动作 → 增加 StageProgress → 满后触发突破校验 → 
成功→RealmLevel+1  失败→StageProgress 衰减 + 可能心魔
```

**与子模组交互**：
- `CultivationEvents.OnRealmBreakthrough(Pawn, int newRealm)` — 突破时通知
- 子模组可通过事件添加自定义突破动画/效果

---

### 2. 灵根系统（SpiritRoot）

**职责**：定义修仙者资质，影响修炼速度、功法兼容性、神通威力。

```
SpiritRoot → HediffDef（开局赋予）
          ↓
HediffComp_SpiritRoot
     ↓ 属性
 - RootType: SpiritRootType  // 金/木/水/火/土/异/无
 - RootQuality: int           // 1~10，越高速度越快
 - SubRoots: List<SpiritRootType>  // 杂灵根次要属性
```

**修炼速度公式**：
```
baseSpeed × rootQualityMultiplier × realmMultiplier × environmentMultiplier
```
- 单灵根：1.5×
- 双灵根：1.0×
- 三灵根：0.7×
- 四灵根：0.5×
- 五灵根：0.3×（但兼容所有功法）

---

### 3. 灵力系统（SpiritEnergy）

**职责**：修仙核心资源，用于释放神通、维持阵法、激活法器。

```
HediffComp_SpiritEnergy
     ↓ 属性
 - MaxEnergy: float    // 最大灵力（随境界增长）
 - CurrentEnergy: float // 当前灵力
 - RegenRate: float     // 每秒恢复量
```

**消耗与恢复**：
- 神通释放：一次性消耗
- 阵法维持：持续消耗
- 恢复：修炼/灵石吸收/丹药/灵气浓郁环境

---

### 4. 功法系统（CultivationMethod）

**职责**：定义修炼路径，影响神通、属性成长、突破倾向。

```csharp
public interface ICultivationMethod
{
    string MethodName { get; }
    string Description { get; }
    SpiritRootType[] CompatibleRoots { get; }  // 兼容灵根
    int Quality { get; }                       // 1~5 凡/精/上/极/仙
    
    bool CanLearn(Pawn pawn);
    void OnLearn(Pawn pawn);
    void OnBreakthrough(Pawn pawn, int realmLevel);
    float GetCultivationSpeedMultiplier(Pawn pawn);
    IEnumerable<AbilityDef> GetUnlockedAbilities(int realmLevel);
}
```

**注册方式**（子模组）：
```csharp
// 子模组在 Mod.Entry 中调用
CultivationAPI.RegisterMethod(new FanRenBasicMethod());
```

---

### 5. 炼丹系统（Alchemy）

**职责**：丹药制作，核心资源产出路径。

```
丹炉 Building_WorkTable → RecipeDef（丹方）
     ↓
Bill_Production → 开始工作
     ↓
CompAlchemyFurnace（管理火候/药性 minigame）
     ↓ 产出
ThingDef（丹药） ← IngestionOutcomeDoer
     ↓ 服用
Hediff（临时 Buff）/ HediffComp（属性增长）
```

**品质体系**：
```
成功率 = baseChance × (炼丹技能 × 0.5 + 丹房加成 + 丹炉品阶)
品质 = 随机(0~100) + 炼丹技能加成
  - 0~40: 凡品（基础效果）
  - 40~70: 灵品（1.3× 效果）
  - 70~90: 仙品（1.6× 效果，降低丹毒）
  - 90~100: 神品（2× 效果，无丹毒）
```

---

### 6. 炼器系统（Artifact）

**职责**：法器制作与成长。

```
炼器鼎 Building_WorkTable → RecipeDef（器方）
     ↓
Artifact ThingWithComps
     ├── CompArtifact  // 法器核心组件
     │    - Quality: int
     │    - Runes: List<ArtifactRune>  // 器纹
     │    - Soul: ArtifactSoul         // 器魂（可选）
     └── Verb（法器自带技能）
```

---

### 7. 阵法系统（Formation）

**职责**：区域性 buff/debuff，地图级影响。

```
FormationCore（阵眼 Building）
     ← 半径内 →
FormationNode（阵旗 Building，可多个）
     ↓ 效果
MapComponent_Formation
     - FormationType: enum（聚灵/防护/杀伐/传送）
     - Active: bool（需要灵石供能）
     - Coverage: CellRect（影响范围）
```

**运转消耗**：每秒消耗 N 灵石（根据阵法等级和规模）

---

### 8. 灵植系统（SpiritPlant）

**职责**：特殊植物，炼丹材料来源。

```
SpiritPlant → PlantDef
     ↓
CompSpiritPlant
     - GrowthStage: int (0~4)
     - Quality: int
     - MutationChance: float  // 变异概率
     ↓ 事件
成熟时触发：正常收获 / 良性变异 / 恶性变异 / 引来妖兽
```

---

## 事件系统

```csharp
public static class CultivationEvents
{
    // 境界
    public static event Action<Pawn, int> OnRealmBreakthrough;
    
    // 功法
    public static event Action<Pawn, string> OnMethodLearn;
    public static event Action<Pawn, string> OnMethodAbandon;
    
    // 灵力
    public static event Action<Pawn, float, float> OnQiChanged;  // (pawn, new, delta)
    
    // 炼丹
    public static event Action<Pawn, Thing> OnPillCreated;
    public static event Action<Pawn, Thing> OnPillConsumed;
    
    // 炼器
    public static event Action<Pawn, ThingWithComps> OnArtifactCreated;
    
    // 阵法
    public static event Action<Building, bool> OnFormationActivated;
    
    // 战斗
    public static event Action<Pawn, Pawn, DamageInfo> OnAbilityCast;
}
```

---

## API 设计

```csharp
public static class CultivationAPI
{
    // === 功法注册（子模组调用） ===
    public static void RegisterMethod(ICultivationMethod method);
    public static IEnumerable<ICultivationMethod> GetRegisteredMethods();
    
    // === 查询 ===
    public static CompCultivator GetCultivator(Pawn pawn);
    public static bool IsCultivator(Pawn pawn);
    public static int GetRealmLevel(Pawn pawn);
    public static float GetCurrentQi(Pawn pawn);
    public static SpiritRootType GetSpiritRoot(Pawn pawn);
    
    // === 灵力操作 ===
    public static bool ConsumeQi(Pawn pawn, float amount);
    public static void RestoreQi(Pawn pawn, float amount);
    
    // === 境界 ===
    public static bool TryBreakthrough(Pawn pawn, out int newRealm);
}
```

---

## 存档兼容设计

- 所有自定义 `ThingComp`、`HediffComp`、`MapComponent` 必须实现 `IExposable`
- `ExposeData()` 使用版本号前缀：
  ```csharp
  public override void ExposeData()
  {
      base.ExposeData();
      Scribe_Values.Look(ref _version, "cc_version", 1);
      Scribe_Values.Look(ref realmLevel, "realmLevel");
      // ...
  }
  ```
- 新增字段使用 `Scribe_Values.Look` 的默认值参数保证向后兼容
- 不删除已有字段，废弃字段保留但不再使用

---

## 性能考虑

- 修炼 tick 计算：每 60 tick 执行一次（不每帧计算）
- 灵力恢复：每 250 tick（Rare tick）
- 阵法效果：每 250 tick 扫描范围内 Pawn 并应用 buff
- 灵植生长：使用原版 Plant tick 系统（已优化）
- 大范围查询：使用 `Map.listerThings` 而非遍历
