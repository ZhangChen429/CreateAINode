# EQS 核心设计问题深度解答

> 本文档回答EQS设计中的关键问题，揭示其底层设计哲学

---

## 🎯 问题1：为什么EPIC选择运行时动态推理？

### 预计算方案的致命缺陷

#### 缺陷1：无法应对动态环境

**现实场景：**
```
战场初始状态：
  掩体A: 完好 → 评分9.5（最佳）
  掩体B: 完好 → 评分7.0

玩家扔了个手榴弹，5秒后：
  掩体A: 被炸毁 ❌
  掩体B: 完好 → 评分变成9.5（现在最佳）

如果是预计算：
  AI仍然会冲向已被炸毁的掩体A（因为预计算时它是最佳）
```

**动态因素列表：**
- 可破坏建筑
- 移动的掩体（车辆）
- 动态生成的障碍物（技能墙体）
- 其他AI占据了位置

#### 缺陷2：情境依赖问题

**同一个点，不同情境下评分完全不同：**

```
点位X (100, 200, 50):

情境1：敌人在我前方
  - 距离敌人: 15米 → 8分
  - 掩护方向: 朝前有墙 → 9分
  - 总分: 8.5分 ✅ 很好

情境2：敌人在我后方
  - 距离敌人: 15米 → 8分
  - 掩护方向: 朝前有墙，背后空旷 → 2分 ❌
  - 总分: 5分 ❌ 很差
```

**如果预计算：**
- 需要预计算"所有可能的敌人位置"
- 100个候选点 × 100个敌人位置 = 10000种组合
- 再考虑多个敌人、动态障碍物 → 组合爆炸

#### 缺陷3：内存与计算量爆炸

**假设场景：**
- 关卡大小: 1000m × 1000m
- 每10米采样一个点 = 10000个候选点
- 每个点要考虑的情境：
  - 敌人位置：100种可能
  - 队友位置：50种可能
  - 障碍物状态：20种可能
  - 总情境数：100 × 50 × 20 = 100,000种

**预计算存储：**
```
10000个点 × 100000种情境 × 4字节(评分) = 4GB内存
```

这还只是一个关卡，且没考虑动态变化！

---

### 运行时推理的优势

#### 优势1：适应动态环境

```cpp
// 每次查询都是当前环境的快照
void OnNeedCover() {
    // 现在的敌人位置
    // 现在的掩体状态
    // 现在的队友位置
    RunQuery(FindCoverQuery);
}
```

掩体被炸毁？下次查询时自然就不会选它了（因为CoverTest会给0分）

#### 优势2：按需计算

```
不需要预计算所有可能：
  ✅ 只计算当前AI需要的查询
  ✅ 只计算当前情境下的评分
  ✅ 不需要的情境永远不计算
```

#### 优势3：灵活性

策划修改"掩体定义"（从"有墙"改成"有墙+高度差2米"）：
- ❌ 预计算：需要重新烘焙所有数据
- ✅ 运行时：下次查询时自动生效

---

### EPIC的权衡与解决方案

**核心矛盾：**
```
运行时推理 = 高灵活性 + 高计算成本
预计算   = 低灵活性 + 低运行成本
```

**EPIC的答案：用架构设计换取性能**

| 技术手段 | 目的 | 效果 |
|---------|------|------|
| **异步执行** | 不阻塞主线程 | 查询可能需要5ms，但不影响游戏60fps |
| **跨帧分摊** | 单帧只处理部分候选点 | 100个点分5帧处理，每帧20个 |
| **批量处理** | 一次查询处理多个点 | SIMD优化，向量化计算 |
| **结果缓存** | 短时间内复用结果 | 相同查询0.5秒内直接返回缓存 |
| **优先级队列** | 重要AI先计算 | Boss AI优先级高于小兵 |
| **Instance池** | 避免频繁分配 | 复用QueryInstance对象 |

**最终效果：**
```
✅ 灵活性：完全动态，适应所有情况
✅ 性能：异步+优化后，单帧开销可控（<1ms）
✅ 可扩展：支持上百个AI同时查询
```

---

## 🧠 问题2："AI不知道答案在哪，而是知道如何评估"的深层含义

### 你的理解完全正确！

这句话的核心是**范式转换**：

```
传统方式（AI知道答案）:
  关卡 → [掩体点列表] → AI直接获取

EQS方式（AI知道标准）:
  AI → [查询需求] → 系统评估 → 返回候选 → AI选择
```

---

### 深入类比：餐厅选择

#### 方式1：AI知道答案（传统预设点）

```
我：我想吃饭
AI助手：去"海底捞(坐标:116.4,39.9)"

问题：
  - 如果海底捞关门了呢？
  - 如果我今天不想吃火锅呢？
  - 如果我现在在北京五环外呢？（太远）
```

**本质：硬编码答案，无法适应变化**

---

#### 方式2：AI知道评估标准（EQS）

```
我：我想吃饭
AI助手：你的需求是什么？
我：距离<2km，人均<100，评分>4.5，想吃辣

[系统查询大众点评]
返回100家候选：
  1. 川菜馆A: 距离0.8km(9分) + 人均80(8分) + 评分4.8(9分) + 辣度高(10分) = 综合9.0分
  2. 湘菜馆B: 距离1.2km(7分) + 人均90(7分) + 评分4.6(8分) + 辣度中(7分) = 综合7.3分
  3. 海底捞C: 距离0.5km(10分) + 人均150(0分) = 综合4分 ❌ 不合格

我：选A
```

**关键区别：**
- 我（AI）声明的是**需求**（特征）
- 系统负责**搜索+评估**
- 最终**我决定**选哪个

---

### 在EQS中的映射

```cpp
// ❌ 传统方式：AI知道答案
Vector3 coverPos = Level.CoverPoints[5]; // 硬编码第5号掩体
MoveTo(coverPos);

问题：
  - 如果5号掩体被占用了？
  - 如果5号掩体被炸毁了？
  - 如果5号掩体距离敌人太近了？
```

```cpp
// ✅ EQS方式：AI知道标准
UEnvQuery* FindCoverQuery = {
    Context: [我的位置, 敌人位置],
    Generator: Grid(半径30米),
    Tests: [
        Distance(敌人, 10-30米, 权重0.4),
        Trace(有墙遮挡, 权重0.4),
        Distance(我, <20米, 权重0.2)
    ]
};

// 系统运行时生成候选并评分
TArray<FEnvQueryResult> Results = RunQuery(FindCoverQuery);
// 返回：[点A:8.5分, 点B:7.2分, 点C:6.8分...]

// AI决策
Vector3 bestPos = Results[0].Location;
MoveTo(bestPos);
```

**优势：**
- ✅ 掩体被占用？下次查询会避开（加个"无人占用"的Test）
- ✅ 掩体被炸毁？Trace Test会失败，评分降低
- ✅ 距离变化？Distance Test自动重新计算

---

### 核心本质：控制反转（IoC）

**传统方式：**
```
[关卡数据] 控制 [AI行为]
  ↓
关卡说："这里是掩体点"
AI："好的，我去"
```

**EQS方式：**
```
[AI需求] 驱动 [环境查询]
  ↓
AI说："我需要一个满足X特征的位置"
系统："这里有符合的候选，你选"
```

**权力转移：**
- 传统：**环境告诉AI该去哪**（环境中心）
- EQS：**AI表达需求，环境提供选项**（AI中心）

---

## 📋 问题3：声明特征 vs 声明位置

### 你的理解精准到位！

这是EQS最核心的设计哲学：**从位置空间到特征空间的转换**

---

### 范式对比

#### 命令式（声明位置）

```cpp
// 策划在关卡中手动放置
CoverPoint coverPoints[] = {
    {位置: (100, 200, 50), 类型: "低掩体"},
    {位置: (150, 210, 50), 类型: "高掩体"},
    {位置: (120, 190, 52), 类型: "窗口"}
};

// AI使用
Vector3 pos = GetNearestCoverPoint(myPos);
```

**问题：**
- 策划需要手动放置1000个点？
- 关卡调整后，1000个点都要重新检查？
- 如何确保"这个点真的是好掩体"？

---

#### 声明式（声明特征）

```cpp
// 策划配置Query
Query: "找掩体" {
    Generator: Grid {
        中心: 我的位置,
        半径: 30米,
        间隔: 2米,
        投射到导航网格: true
    },

    Tests: [
        Test: "距离敌人合适" {
            测量对象: 敌人,
            理想范围: 10-30米,
            权重: 0.4
        },
        Test: "有墙遮挡" {
            射线检测: 从候选点到敌人,
            要求: 被阻挡,
            权重: 0.4
        },
        Test: "离我不太远" {
            测量对象: 我,
            最大距离: 20米,
            权重: 0.2
        }
    ]
}
```

**系统运行时：**
```
1. 在我周围30米内生成225个候选点（15×15网格）
2. 对每个点执行3个Test
3. 点A评分：
   - 距离敌人15米 → 9分（理想区间内）
   - 有墙遮挡 → 10分（完全阻挡）
   - 距离我8米 → 10分（很近）
   - 总分：9*0.4 + 10*0.4 + 10*0.2 = 9.6分 ✅

4. 返回Top 10候选点
```

**AI使用：**
```cpp
RunQuery("找掩体", OnComplete);

void OnComplete(Results) {
    Vector3 bestPos = Results[0]; // 9.6分的点A
    MoveTo(bestPos);
}
```

---

### 关键区别总结

| 维度 | 声明位置（传统） | 声明特征（EQS） |
|-----|----------------|----------------|
| **策划工作** | 手动放置1000个点 | 配置特征描述 |
| **数据存储** | 1000个Vector3 | 1个Query配置 |
| **适应变化** | 环境改变需重新放置 | 自动适应新环境 |
| **正确性** | 无法保证点真的好 | 系统保证符合标准 |
| **调试** | 不知道为什么选这个点 | 可以看到每个维度的评分 |
| **复用性** | 每个关卡要重新放点 | 同一个Query适用所有关卡 |

---

### 类比：面向对象编程

这和OOP中的概念类似：

```python
# ❌ 面向具体（声明位置）
good_covers = [
    Vector3(100, 200, 50),
    Vector3(150, 210, 50)
]

# ✅ 面向接口（声明特征）
class GoodCover:
    def is_valid(self, pos):
        return (
            distance_to_enemy(pos) in range(10, 30) and
            has_cover(pos) and
            distance_to_me(pos) < 20
        )

good_covers = [p for p in all_positions if GoodCover.is_valid(p)]
```

**核心：从"是什么"到"满足什么"**

---

## 👨‍💻 问题4：策划的角色与系统的职责

### 你的理解完全正确！职责分工非常清晰

---

### 职责矩阵

| 角色 | 负责什么 | 不负责什么 | 输出 |
|-----|---------|-----------|------|
| **策划** | 定义"好位置"的标准 | 计算具体位置 | Query配置 |
| **系统** | 生成候选+评分+排序 | 决策选哪个 | 排序后的候选列表 |
| **AI** | 发起需求+最终选择 | 评分算法 | 行为决策 |
| **程序员** | 实现Test算法 | 配置具体参数 | Test类库 |

---

### 详细流程

#### 步骤1：策划配置（设计时）

**配置内容：**

```yaml
Query: "找掩体"
  # 1. 配置生成规则：在哪里生成候选点
  Generator:
    类型: Grid
    参数:
      中心: Context.Querier (我的位置)
      半径: 30米
      间隔: 2米
      数量: 约225个点
      投射到导航网格: true

  # 2. 配置特征描述：候选点应该有什么特性
  Tests:
    - Test: Distance
        目标: Context.Enemy (敌人)
        理想范围: 10-30米
        权重: 0.4

    - Test: Trace
        起点: 候选点
        终点: Context.Enemy
        要求: 被阻挡
        权重: 0.4

    - Test: Distance
        目标: Context.Querier
        最大值: 20米
        权重: 0.2

  # 3. 配置评分合并方式
  Normalization:
    类型: 加权求和
    范围: [0, 1]
```

**策划思考的是：**
- "我想要的掩体应该距离敌人10-30米"（领域知识）
- "必须有墙遮挡"（游戏规则）
- "不要离我太远"（玩法体验）

**策划不需要知道：**
- 具体哪个坐标是好位置
- 距离怎么计算（欧几里得？曼哈顿？）
- 射线检测怎么实现

---

#### 步骤2：系统运行（运行时）

```cpp
// AI发起查询
QueryManager->RunQuery(FindCoverQuery, MyContext);

// 系统内部流程
void UEnvQueryManager::RunQuery(Query, Context) {
    // 2.1 解析上下文
    UObject* Querier = Context.GetQuerier(); // 我的Actor
    UObject* Enemy = Context.GetEnemy();     // 敌人Actor

    // 2.2 生成候选点（按Generator规则）
    TArray<FVector> Items = Generator->Generate(Querier->GetLocation());
    // 结果：225个候选点

    // 2.3 执行每个Test，给每个候选点打分
    for (UEnvQueryTest* Test : Tests) {
        TArray<float> Scores = Test->Execute(Items, Context);

        // 归一化到[0, 1]
        Scores = Normalize(Scores);

        // 应用权重
        for (int i = 0; i < Items.Num(); i++) {
            Items[i].Score += Scores[i] * Test->Weight;
        }
    }

    // 2.4 排序
    Items.Sort([](A, B) { return A.Score > B.Score; });

    // 2.5 返回结果
    OnComplete.Execute(Items);
}
```

**系统确定：**
- 点A的DistanceTest得分：8.5分
- 点A的TraceTest得分：10分
- 点A的综合得分：8.5*0.4 + 10*0.4 + 9*0.2 = 9.2分

**系统输出：**
```cpp
Results = [
    {位置: (105, 198, 50), 综合评分: 9.2},
    {位置: (112, 205, 50), 综合评分: 8.7},
    {位置: (98, 190, 50), 综合评分: 8.3},
    // ... 其余222个点
]
```

---

#### 步骤3：AI决策（运行时）

```cpp
void OnQueryComplete(FEnvQueryResult Result) {
    if (Result.IsValid()) {
        // AI可以有多种策略

        // 策略1：选最优
        Vector3 bestPos = Result.GetItemAsLocation(0);

        // 策略2：在Top3中随机（增加变化性）
        int index = FMath::RandRange(0, 2);
        Vector3 pos = Result.GetItemAsLocation(index);

        // 策略3：根据AI性格（勇敢型选近的，谨慎型选远的）
        Vector3 pos = AIPersonality == Brave
            ? Result.GetClosestToEnemy()
            : Result.GetFarthestFromEnemy();

        // 执行移动
        MoveTo(pos);
    }
}
```

**AI决定：**
- 要不要用这个结果（可能现在不需要掩体了）
- 用哪个候选点（最优？Top3随机？）
- 如何使用这个位置（直接移动？还是先观察？）

---

### 角色互动图

```
┌──────────┐                    ┌──────────┐                    ┌──────────┐
│  策划     │                    │  系统     │                    │   AI      │
└────┬─────┘                    └────┬─────┘                    └────┬─────┘
     │                                │                                │
     │ 1. 配置Query Asset              │                                │
     │   "掩体应该距敌10-30米"          │                                │
     ├────────────────────────────────>│                                │
     │                                │                                │
     │                                │  2. AI运行时发起查询             │
     │                                │<───────────────────────────────┤
     │                                │                                │
     │                                │ 3. 生成候选点(225个)            │
     │                                │ 4. 执行Test评分                 │
     │                                │ 5. 归一化+合并                  │
     │                                │ 6. 排序                        │
     │                                │                                │
     │                                │ 7. 返回Top候选列表              │
     │                                ├────────────────────────────────>│
     │                                │                                │
     │                                │                8. AI选择点A     │
     │                                │                9. 执行移动      │
     │                                │                                │
```

---

### 关键设计优势

**1. 解耦：策划不需要懂算法**
```
策划：我要"距离10-30米"
程序：我实现DistanceTest（欧几里得距离）
改进：我把算法改成"路径距离"（沿导航网格）
→ 策划的配置不用改
```

**2. 复用：一个Query适配所有关卡**
```
同一个"找掩体"Query：
  - 在城市关卡 → 在建筑间生成候选点
  - 在森林关卡 → 在树木间生成候选点
  - 在室内关卡 → 在房间内生成候选点
→ 系统根据导航网格自动适应
```

**3. 调试：可视化每个维度的贡献**
```
点A评分9.2分，为什么？
  - DistanceTest: 8.5分 (权重0.4) → 贡献3.4分
  - TraceTest: 10分 (权重0.4) → 贡献4.0分
  - CloseTest: 9分 (权重0.2) → 贡献1.8分
  → 可以看到是"掩护"和"距离"占主导
```

---

## 📊 问题5：打分系统的配置与合并

### 谁配置打分系统？

**三层配置模型：**

| 层级 | 配置者 | 配置内容 | 示例 |
|-----|-------|---------|------|
| **框架层** | 引擎程序员 | Test基类、归一化算法、合并策略 | `UEnvQueryTest`基类 |
| **组件层** | Gameplay程序员 | 具体Test实现 | `DistanceTest`, `TraceTest` |
| **实例层** | 策划 | 具体Query中的Test组合和权重 | "找掩体"Query配置 |

---

### 详细配置流程

#### 层级1：引擎程序员提供框架

```cpp
// UEnvQueryTest基类
class UEnvQueryTest {
public:
    // 核心接口
    virtual void RunTest(TArray<FEnvQueryItem>& Items, Context) = 0;

    // 归一化方式
    enum ENormalizationType {
        Absolute,      // 绝对值归一化 [min, max] → [0, 1]
        RelativeToMax, // 相对最大值
        RelativeToMin  // 相对最小值
    };

    // 评分合并方式
    enum EScoreCombine {
        WeightedSum,   // 加权求和
        Multiply,      // 乘积
        Min,           // 取最小
        Max            // 取最大
    };

    // 配置项
    float Weight = 1.0f;
    ENormalizationType NormalizationType;
};
```

---

#### 层级2：Gameplay程序员实现具体Test

```cpp
// 距离Test
class UEnvQueryTest_Distance : public UEnvQueryTest {
public:
    // 策划可配置的参数
    UPROPERTY(EditAnywhere)
    float MinDistance = 0.0f;

    UPROPERTY(EditAnywhere)
    float MaxDistance = 1000.0f;

    UPROPERTY(EditAnywhere)
    TEnumAsByte<EDistanceMode> DistanceMode; // 3D, 2D, Z轴

    // 实现打分逻辑
    virtual void RunTest(TArray<FEnvQueryItem>& Items, Context) override {
        UObject* Target = Context.GetTarget(); // 测量目标

        for (FEnvQueryItem& Item : Items) {
            // 计算距离
            float Distance = (Item.Location - Target->GetLocation()).Size();

            // 评分逻辑：理想区间内得分高
            float Score;
            if (Distance < MinDistance) {
                Score = 0.0f; // 太近
            } else if (Distance > MaxDistance) {
                Score = 0.0f; // 太远
            } else {
                // 在理想区间内，越接近中点越高分
                float Mid = (MinDistance + MaxDistance) / 2;
                float Range = (MaxDistance - MinDistance) / 2;
                Score = 1.0f - abs(Distance - Mid) / Range;
            }

            Item.Score = Score;
        }
    }
};
```

```cpp
// 射线检测Test
class UEnvQueryTest_Trace : public UEnvQueryTest {
public:
    UPROPERTY(EditAnywhere)
    bool bRequireBlocked = true; // 要求被阻挡（掩护）还是通畅（视野）

    virtual void RunTest(TArray<FEnvQueryItem>& Items, Context) override {
        UObject* Target = Context.GetTarget();

        for (FEnvQueryItem& Item : Items) {
            // 执行射线检测
            FHitResult Hit;
            bool bHit = World->LineTraceSingle(
                Hit,
                Item.Location,
                Target->GetLocation(),
                ECC_Visibility
            );

            // 评分
            if (bRequireBlocked) {
                Item.Score = bHit ? 1.0f : 0.0f; // 有遮挡=满分
            } else {
                Item.Score = bHit ? 0.0f : 1.0f; // 无遮挡=满分
            }
        }
    }
};
```

---

#### 层级3：策划配置具体Query

**在编辑器中配置：**

```
Query Asset: "找掩体"

┌─ Generator: Grid ───────┐
│ 半径: 30米               │
│ 间隔: 2米                │
└─────────────────────────┘

┌─ Test 1: Distance ──────┐
│ 类型: UEnvQueryTest_Distance    │
│ 目标: Enemy                      │
│ 最小距离: 10米                   │
│ 最大距离: 30米                   │
│ 权重: 0.4                        │
│ 归一化方式: Absolute             │
└─────────────────────────────────┘

┌─ Test 2: Trace ─────────┐
│ 类型: UEnvQueryTest_Trace       │
│ 起点: Item (候选点)             │
│ 终点: Enemy                     │
│ 要求阻挡: true                  │
│ 权重: 0.4                       │
└─────────────────────────────────┘

┌─ Test 3: Distance ──────┐
│ 类型: UEnvQueryTest_Distance    │
│ 目标: Querier (我)              │
│ 最大距离: 20米                  │
│ 权重: 0.2                       │
└─────────────────────────────────┘

┌─ 合并方式 ──────────────┐
│ 类型: 加权求和            │
└─────────────────────────┘
```

---

### 打分系统的核心目标

**目标1：多维度量化"好位置"**

```
人类思维："这个掩体不错，距离合适，掩护也好"
  ↓ 量化
系统评分：
  - 距离维度：8.5分
  - 掩护维度：10分
  - 综合：9.2分
```

**目标2：支持权衡（Trade-off）**

```
场景A：追击战（重视距离）
  权重：Distance=0.7, Cover=0.3
  → 选择"较近但掩护一般"的点

场景B：防守战（重视掩护）
  权重：Distance=0.3, Cover=0.7
  → 选择"较远但掩护很好"的点

同一组候选点，不同权重 → 不同选择
```

**目标3：归一化不同量纲**

```
问题：
  - Distance返回：15米（原始值）
  - Trace返回：true/false（布尔值）
  - Angle返回：45度（角度值）

如何合并？

解决：归一化到[0, 1]
  - Distance: 15米 → 在[10, 30]区间内 → 0.75分
  - Trace: true → 1.0分
  - Angle: 45度 → 在[0, 90]区间内 → 0.5分

现在可以加权求和：0.75*0.4 + 1.0*0.4 + 0.5*0.2 = 0.8分
```

---

### 不同维度的分值如何合并

#### 方法1：加权求和（最常用）

```cpp
// 每个Test返回[0, 1]的分数
float score1 = DistanceTest->Run(item); // 0.8
float score2 = CoverTest->Run(item);    // 1.0
float score3 = HeightTest->Run(item);   // 0.6

// 应用权重
float weight1 = 0.4;
float weight2 = 0.4;
float weight3 = 0.2;

// 加权求和
float finalScore = score1 * weight1 + score2 * weight2 + score3 * weight3;
// = 0.8*0.4 + 1.0*0.4 + 0.6*0.2 = 0.84
```

**适用场景：**
- 各维度相对独立
- 允许"某维度差，其他维度补"

---

#### 方法2：乘积（AND逻辑）

```cpp
float finalScore = score1 * score2 * score3;
// = 0.8 * 1.0 * 0.6 = 0.48
```

**特点：**
- 任何一个维度为0 → 总分为0
- 类似"必须同时满足"
- 得分会偏低（需要所有维度都好）

**适用场景：**
- 硬约束（如"必须有掩护 AND 必须可到达"）

---

#### 方法3：最小值（最弱项）

```cpp
float finalScore = FMath::Min3(score1, score2, score3);
// = min(0.8, 1.0, 0.6) = 0.6
```

**特点：**
- "木桶效应"：短板决定总分
- 避免"某维度特别差但总分还行"的情况

**适用场景：**
- 安全性评估（最弱环节决定安全性）

---

#### 方法4：最大值（最强项）

```cpp
float finalScore = FMath::Max3(score1, score2, score3);
// = max(0.8, 1.0, 0.6) = 1.0
```

**适用场景：**
- "任一条件满足即可"的OR逻辑

---

### 归一化的重要性

#### 为什么必须归一化？

**问题场景：**

```cpp
// 未归一化
Test1: Distance = 15米 (原始值)
Test2: Angle = 45度 (原始值)

// 如果直接加权求和
Score = 15 * 0.5 + 45 * 0.5 = 30

问题：
  1. 数值大小不可比（15米 vs 45度）
  2. 权重失效（角度值天然大，主导了结果）
  3. 无法判断"30分"是好是坏
```

**归一化后：**

```cpp
// 归一化到[0, 1]
Test1: Distance = 15米 → 在[10, 30]区间 → 0.75分
Test2: Angle = 45度 → 在[0, 90]区间 → 0.5分

// 加权求和
Score = 0.75 * 0.5 + 0.5 * 0.5 = 0.625

优势：
  ✅ 数值可比（都是[0, 1]）
  ✅ 权重有效（真正是5:5）
  ✅ 结果可解释（0.625 = 62.5%满意度）
```

---

#### 归一化方法

**方法1：线性归一化**

```cpp
// 原始值在[min, max]区间
float Normalize(float value, float min, float max) {
    if (value < min) return 0.0f;
    if (value > max) return 1.0f;
    return (value - min) / (max - min);
}

// 示例
Normalize(15米, 10, 30) = (15-10) / (20) = 0.25
```

**方法2：反向归一化**

```cpp
// 越小越好（如距离）
float NormalizeInverse(float value, float min, float max) {
    return 1.0f - Normalize(value, min, max);
}

// 示例：距离越近越好
NormalizeInverse(5米, 0, 20) = 1.0 - 0.25 = 0.75 (近=高分)
```

**方法3：高斯曲线**

```cpp
// 有理想值，偏离理想值扣分
float NormalizeGaussian(float value, float ideal, float range) {
    float diff = abs(value - ideal);
    return exp(-(diff * diff) / (2 * range * range));
}

// 示例：理想距离15米
NormalizeGaussian(15米, 15, 5) = 1.0 (完美)
NormalizeGaussian(20米, 15, 5) = 0.6 (偏离)
```

---

### 实际案例：找掩体的完整评分

```cpp
候选点A的评分过程：

┌──────────────────────────────────────────┐
│ Step 1: 执行各Test                       │
├──────────────────────────────────────────┤
│ Test: Distance to Enemy                  │
│   原始值: 15米                           │
│   理想范围: [10, 30]                     │
│   归一化: (15-10)/20 = 0.25              │
│   反向: 1.0 - 0.25 = 0.75 (因为越远越好) │
│   权重: 0.4                              │
│   得分: 0.75 ✓                           │
├──────────────────────────────────────────┤
│ Test: Has Cover                          │
│   射线检测: 被阻挡                       │
│   归一化: true → 1.0                     │
│   权重: 0.4                              │
│   得分: 1.0 ✓                            │
├──────────────────────────────────────────┤
│ Test: Distance to Me                     │
│   原始值: 8米                            │
│   最大距离: 20米                         │
│   归一化: 1.0 - (8/20) = 0.6 (越近越好)  │
│   权重: 0.2                              │
│   得分: 0.6 ✓                            │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│ Step 2: 加权合并                         │
├──────────────────────────────────────────┤
│ 最终得分 = 0.75*0.4 + 1.0*0.4 + 0.6*0.2 │
│          = 0.3 + 0.4 + 0.12              │
│          = 0.82                          │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│ Step 3: 解释                             │
├──────────────────────────────────────────┤
│ 总分82%，其中：                          │
│   - 掩护贡献最大 (40%)                   │
│   - 距离敌人合适 (30%)                   │
│   - 距离我较近 (12%)                     │
│ → 这是一个"掩护优秀、距离合理"的位置    │
└──────────────────────────────────────────┘
```

---

## 🎨 问题6：声明式配置在设计上的优势

### 声明式 vs 命令式对比

#### 命令式（Imperative）- 告诉"怎么做"

```cpp
// 命令式：详细描述步骤
Vector3 FindCoverPoint() {
    // 步骤1：生成候选点
    TArray<Vector3> candidates;
    for (float x = MyPos.X - 30; x <= MyPos.X + 30; x += 2) {
        for (float y = MyPos.Y - 30; y <= MyPos.Y + 30; y += 2) {
            Vector3 pos(x, y, MyPos.Z);
            if (NavMesh->IsWalkable(pos)) {
                candidates.Add(pos);
            }
        }
    }

    // 步骤2：评分
    float bestScore = 0;
    Vector3 bestPos;
    for (Vector3& pos : candidates) {
        // 计算距离敌人
        float distToEnemy = (pos - EnemyPos).Size();
        float distScore = (distToEnemy >= 10 && distToEnemy <= 30) ? 1.0 : 0.0;

        // 检测掩护
        FHitResult hit;
        bool hasCover = World->LineTrace(pos, EnemyPos, hit);
        float coverScore = hasCover ? 1.0 : 0.0;

        // 计算距离我
        float distToMe = (pos - MyPos).Size();
        float closeScore = 1.0 - (distToMe / 20.0);

        // 合并
        float totalScore = distScore * 0.4 + coverScore * 0.4 + closeScore * 0.2;

        if (totalScore > bestScore) {
            bestScore = totalScore;
            bestPos = pos;
        }
    }

    return bestPos;
}
```

**问题：**
- ❌ 策划无法修改（需要改代码）
- ❌ 复用困难（下次找不同类型的位置要重写）
- ❌ 调试困难（无法可视化中间步骤）
- ❌ 优化困难（异步、跨帧改动大）
- ❌ 扩展困难（加新维度要改很多地方）

---

#### 声明式（Declarative）- 描述"要什么"

```yaml
# Query Asset配置文件
Query: "找掩体"
  Generator:
    Type: Grid
    Center: Querier
    Radius: 30
    Spacing: 2

  Tests:
    - Type: Distance
      Target: Enemy
      Range: [10, 30]
      Weight: 0.4

    - Type: Trace
      From: Item
      To: Enemy
      RequireBlocked: true
      Weight: 0.4

    - Type: Distance
      Target: Querier
      Max: 20
      Weight: 0.2
```

**优势：**
- ✅ 策划可以在编辑器中修改
- ✅ 同一套配置系统复用
- ✅ 可视化调试（编辑器插件）
- ✅ 引擎负责优化执行
- ✅ 添加新Test只需插件

---

### 声明式配置的六大设计优势

#### 优势1：解耦"做什么"与"怎么做"

```
┌─────────────┐          ┌─────────────┐
│  策划层      │          │  程序层      │
│ (What)      │          │ (How)       │
├─────────────┤          ├─────────────┤
│ 配置需求：   │    独立    │ 实现算法：   │
│"距离10-30米"│   ◄──►   │ 欧几里得距离 │
│             │          │ 或路径距离   │
└─────────────┘          └─────────────┘
```

**举例：**
```
策划配置：
  Test: Distance, Min: 10, Max: 30

程序实现V1：
  float dist = (A - B).Size(); // 直线距离

程序实现V2（优化）：
  float dist = NavMesh->GetPathLength(A, B); // 路径距离

策划配置完全不需要改！
```

---

#### 优势2：组合优于继承

**命令式困境（继承地狱）：**

```cpp
class FindCoverPoint { }
class FindHighCoverPoint : public FindCoverPoint { }
class FindLowCoverPoint : public FindCoverPoint { }
class FindCoverPointNearAlly : public FindCoverPoint { }
class FindHighCoverPointNearAlly : public FindHighCoverPoint, FindCoverPointNearAlly { } // 多重继承噩梦
```

**声明式解决（组合）：**

```yaml
Query: "找高掩体"
  Tests: [Distance, Trace, Height]

Query: "找低掩体"
  Tests: [Distance, Trace, Height(reversed)]

Query: "找队友附近的掩体"
  Tests: [Distance, Trace, DistanceToAlly]

Query: "找队友附近的高掩体"
  Tests: [Distance, Trace, Height, DistanceToAlly]
  # 只需组合现有Test，不需要新类
```

**优势：**
- ✅ N个Test = 2^N种组合
- ✅ 不需要写新代码
- ✅ 策划自由组合

---

#### 优势3：数据驱动（配置即行为）

```
传统方式：
  修改AI行为 → 改代码 → 编译 → 测试 → 发现不对 → 改代码 → 编译...
  ⏱️ 每次迭代：5-10分钟

声明式方式：
  修改AI行为 → 调整Query配置 → 保存 → 立即生效 → 发现不对 → 调整 → 立即生效
  ⏱️ 每次迭代：10秒
```

**迭代效率提升30-60倍！**

---

#### 优势4：可视化调试

```
命令式：
  代码返回一个Vector3
  ✘ 不知道为什么选这个点
  ✘ 不知道其他候选点是什么
  ✘ 不知道每个维度的贡献

声明式（编辑器插件）：
  ✓ 在场景中显示所有225个候选点
  ✓ 颜色表示评分（绿=高分，红=低分）
  ✓ 选中一个点，显示详细评分：
    - Distance: 0.75 (权重0.4) → 贡献0.3
    - Cover: 1.0 (权重0.4) → 贡献0.4
    - Close: 0.6 (权重0.2) → 贡献0.12
    - Total: 0.82
```

**UE编辑器中的实际效果：**
- 可以在Viewport中看到EQS查询的可视化调试
- 每个候选点是一个小球体，颜色表示评分
- 点击后显示详细breakdown

---

#### 优势5：复用性

**一个Query，多个场景复用：**

```yaml
Query: "找掩体"
  # 配置不包含任何关卡特定的坐标

使用场景：
  - 城市关卡 → 在建筑间找掩体
  - 森林关卡 → 在树木间找掩体
  - 室内关卡 → 在柱子、家具间找掩体
  - 沙漠关卡 → 在岩石、沙丘间找掩体

同一个Query Asset，适配所有环境！
```

**原因：**
- Generator基于运行时的导航网格（动态）
- Test基于运行时的Context（我、敌人的位置）
- 不依赖硬编码坐标

---

#### 优势6：渐进式优化（Progressive Enhancement）

**引擎可以在不改配置的情况下优化执行：**

```
版本1.0（同步执行）：
  RunQuery() → 等待5ms → 返回结果
  ✘ 卡顿

版本2.0（引擎优化：异步）：
  RunQueryAsync(OnComplete) → 立即返回 → 后台计算 → 回调
  ✓ 不卡顿
  策划配置完全不用改！

版本3.0（引擎优化：跨帧）：
  第1帧：生成候选点
  第2帧：执行前50个点的Test
  第3帧：执行后50个点的Test
  第4帧：合并排序返回
  策划配置完全不用改！

版本4.0（引擎优化：SIMD）：
  向量化计算，性能提升4倍
  策划配置完全不用改！
```

---

### 类比：SQL的成功

EQS的声明式设计和SQL非常相似：

```sql
-- SQL（1974年发明，50年后仍在用）
SELECT * FROM positions
WHERE distance < 100 AND has_cover = true
ORDER BY score DESC
LIMIT 1
```

**SQL成功的原因：**
1. ✅ **声明式**：描述"要什么"，不管"怎么做"
2. ✅ **优化器**：数据库引擎负责优化查询计划
3. ✅ **数据独立性**：查询语句不依赖存储细节
4. ✅ **复用性**：同一个查询适用不同数据集
5. ✅ **可组合**：WHERE子句可以任意组合条件

**EQS借鉴了这些优势：**
- Query Asset = SQL语句
- EQS Manager = 查询优化器
- Test = WHERE条件
- Generator = FROM子句
- Finalize = ORDER BY + LIMIT

---

### 反面教材：如果EQS用命令式

**假设EQS是命令式的：**

```cpp
// 策划想找掩体
class MyCustomCoverFinder : public ICoverFinder {
public:
    Vector3 FindCover(Context ctx) override {
        // 策划需要写C++代码
        TArray<Vector3> points;
        for (float x = -30; x <= 30; x += 2) {
            for (float y = -30; y <= 30; y += 2) {
                Vector3 p = ctx.Querier->GetLocation() + Vector3(x, y, 0);
                // ... 一大堆距离、射线检测、评分代码
                points.Add(p);
            }
        }
        // ... 排序、选择逻辑
        return points[0];
    }
};
```

**后果：**
- ❌ 策划写不了（需要程序员）
- ❌ 每个AI行为要写一个类
- ❌ 程序员成为瓶颈
- ❌ 代码重复（每个类都有类似逻辑）
- ❌ 难以维护（散落在100个类中）

---

## 📝 总结：EQS设计哲学的本质

### 核心理念

```
传统AI：环境告诉AI去哪
  关卡 → [预设点] → AI
  ✘ 静态、脆弱、不智能

EQS：AI表达需求，环境响应
  AI → [查询需求] → 系统 → [候选+评分] → AI选择
  ✓ 动态、灵活、智能
```

---

### 设计精髓（用一句话概括）

> **EQS将"空间决策问题"转化为"声明式查询问题"，通过运行时评估多维特征，让AI基于当前情境做出智能选择。**

---

### 关键设计决策总结

| 问题 | EPIC的选择 | 原因 |
|-----|----------|------|
| 预计算 vs 运行时 | **运行时推理** | 适应动态环境、情境依赖 |
| 位置 vs 特征 | **声明特征** | 复用性、适应性 |
| 命令式 vs 声明式 | **声明式配置** | 生产力、迭代效率 |
| 同步 vs 异步 | **异步+跨帧** | 性能、不卡顿 |
| 单维度 vs 多维度 | **多维度评分** | 真实决策的复杂性 |
| 硬编码 vs 数据驱动 | **数据驱动** | 策划可配置 |

---

### 对其他系统的启发

**当你的系统有以下特征时，考虑EQS模式：**

1. ✅ 需要从大量候选中选"最优"
2. ✅ 有多个独立的评价维度
3. ✅ 非技术人员需要配置
4. ✅ 需要运行时动态决策
5. ✅ 性能敏感（需要异步优化）

**示例应用：**
- **RTS建筑选址**：Generator生成候选位置，Test评估资源、防御、距离
- **开放世界任务生成**：Generator生成任务候选，Test评估难度、奖励、玩家偏好
- **动态难度调整**：Generator生成敌人配置候选，Test评估玩家能力、挑战性
- **程序化关卡生成**：Generator生成房间布局候选，Test评估连通性、趣味性

---

**最后的关键：**
> 声明式设计不是为了炫技，而是为了解决真实的生产力问题：让策划能够自主配置复杂的AI行为，而不依赖程序员。这才是EQS设计的最大价值。
