# UE EQS 三层模型（修正版）

## 架构概览

```
┌──────────────────────────────────────────────────────┐
│  数据配置层 (Environment Query Asset)                 │
│  - 只定义规则，不参与运行时                            │
│  - 类比：试卷（题目 + 评分标准）                       │
└────────────────┬─────────────────────────────────────┘
                 │ 读取配置
                 ↓
┌──────────────────────────────────────────────────────┐
│  执行调度层 (UEnvQueryManager)                        │
│  - 只调度管控，不做具体计算                            │
│  - 类比：监考老师（分发试卷、限时、收卷）              │
└────────────────┬─────────────────────────────────────┘
                 │ 驱动执行
                 ↓
┌──────────────────────────────────────────────────────┐
│  核心计算层 (FEnvQueryInstance + Nodes)               │
│  - 只执行计算，按规则干活                              │
│  - 类比：做题学生（按题目要求解题）                    │
└──────────────────────────────────────────────────────┘
```

---

## 第一层：数据配置层（Environment Query Asset）

### 🎯 核心定位
**只定义规则，不碰运行时计算**

### 📦 核心类
- **UEnvQuery** (DataAsset)
  - 存储位置：`.uasset` 文件
  - 可视化编辑器：EQS Editor

### 🔧 配置内容

#### 1. 查询元信息
```
- QueryName: 查询标识
- RunMode: 结果返回模式
    • SingleResult: 返回最优单项
    • RandomBest5Pct: 从最佳 5% 随机选
    • RandomBest25Pct: 从最佳 25% 随机选
    • AllMatching: 返回全部匹配项
```

#### 2. 查询选项（Options）
每个 Option = 1 个 Generator + N 个 Tests

```cpp
UEnvQueryOption {
    UEnvQueryGenerator* Generator;  // 生成候选
    TArray<UEnvQueryTest*> Tests;   // 筛选评分
}
```

#### 3. 生成器配置（Generator）
定义 "生成哪些候选"

```
Generator 类型:
    - SimpleGrid: 规则网格位置
    - ActorsOfClass: 特定类型的 Actor
    - Donut: 环形区域位置
    - Cone: 锥形区域位置
    - ...

配置参数:
    - GenerateAround: 围绕哪个 Context
    - 范围参数: 半径、密度、数量等
    - 投影设置: 是否投影到导航网格
```

#### 4. 测试规则配置（Tests）
定义 "怎么筛选/打分"

```
Test 配置:
    - TestPurpose:
        • Filter: 只过滤
        • Score: 只评分
        • FilterAndScore: 先过滤再评分

    - 过滤条件:
        • FilterType: Match / Range
        • FloatValueMin/Max: 阈值范围

    - 评分配置:
        • ScoringEquation: Linear / Square / InverseLinear / ...
        • ScoringFactor: 权重系数
        • ReferenceValue: 参考值

    - 归一化:
        • NormalizationType: Absolute / RelativeToScores
```

#### 5. 上下文绑定（Context）
定义 "相对于什么"

```
Context 类型:
    - Querier: 查询发起者（通常是 AI）
    - Target: 目标对象（黑板中的敌人）
    - Item: 当前被测试的候选项
    - Custom: 蓝图自定义上下文
```

### 📝 类比说明
```
就像出一份考试题：
    ✏️ 题目类型（Generator）: 填空题 / 选择题 / 问答题
    📏 评分标准（Test）: 每题几分、怎么给分
    📋 参考答案（Context）: 标准答案是什么
    📊 结果要求（RunMode）: 取最高分 / 取前 10 名 / 全部及格者

但试卷本身不会自己批改，需要老师（Manager）组织学生（Instance）去答题。
```

---

## 第二层：执行调度层（UEnvQueryManager）

### 🎯 核心定位
**全局总指挥，只调度管控，不做具体计算**

### 📦 核心类
- **UEnvQueryManager** (全局单例)
  - 获取方式：`UEnvQueryManager::GetCurrent(World)`
  - 生命周期：整个游戏运行期间

### 🔧 核心职责

#### 1. 接收查询请求
```cpp
// 来源 1: 行为树节点
BTTask_RunEQSQuery → Manager->RunQuery()

// 来源 2: C++ 代码
FEnvQueryRequest Request(QueryTemplate, AIController);
int32 QueryID = Manager->RunQuery(Request, RunMode, Callback);

// 来源 3: 蓝图
RunEQSQuery(QueryAsset, Querier, OnFinished)
```

#### 2. 资源管控
```cpp
class UEnvQueryManager {
    // 运行中的查询队列
    TArray<TSharedPtr<FEnvQueryInstance>> RunningQueries;

    // 性能限制
    float MaxAllowedTestingTime = 0.003f;  // 每帧 3ms
    int32 QueryCountWarningThreshold = 10; // 告警阈值

    // 实例缓存（复用已配置的实例）
    TArray<FEnvQueryInstanceCache> InstanceCache;

    // 上下文对象池
    TMap<FName, UEnvQueryContext*> LocalContextMap;
};
```

#### 3. 时间切片调度
```cpp
void UEnvQueryManager::Tick(float DeltaTime) {
    float TimeRemaining = MaxAllowedTestingTime;

    for (auto& Query : RunningQueries) {
        double StartTime = FPlatformTime::Seconds();

        // 驱动查询执行一小步
        Query->ExecuteOneStep(TimeRemaining);

        TimeRemaining -= (FPlatformTime::Seconds() - StartTime);

        // 检查完成状态
        if (Query->IsFinished()) {
            // 触发回调（直接通知调用方，Manager 不参与）
            Query->FinishDelegate.Execute(Query);
            RunningQueries.Remove(Query);
        }

        // 时间用尽，下一帧继续
        if (TimeRemaining <= 0) break;
    }
}
```

#### 4. 实例生命周期管理
```
创建: CreateQueryInstance(Template)
    ├── 检查缓存: 是否有已配置的实例？
    ├── 复用: Clone 缓存的实例
    └── 新建: 创建并初始化新实例

运行: 添加到 RunningQueries 队列

完成: 触发回调后移除（实例由智能指针管理）
```

#### 5. 性能监控与调试
```cpp
#if USE_EQS_DEBUGGER
    FEQSDebugger EQSDebugger;
    TMap<FName, FStatsInfo> DebuggerStats;
#endif

// 统计信息
- 每个查询的执行时间
- 生成的候选项数量
- 每个测试的耗时
- 可视化调试接口
```

### 📝 类比说明
```
就像监考老师：
    📥 收试卷: 接收来自各处的查询请求
    ⏱️ 限时: 每帧最多花 3ms 处理查询
    👥 分配: 把试卷分配给学生（Instance）
    🔄 督促: 每帧检查学生做题进度（ExecuteOneStep）
    ✅ 收卷: 检测到完成后通知成绩（触发回调）
    📊 统计: 记录每场考试的耗时、及格率

但老师自己不答题，也不批改试卷，只负责组织流程。
```

---

## 第三层：核心计算层（FEnvQueryInstance + Nodes）

### 🎯 核心定位
**执行引擎，只按规则计算，不管调度**

### 📦 核心类

#### 主体：FEnvQueryInstance
```cpp
struct FEnvQueryInstance : public FEnvQueryResult {
    // === 执行上下文 ===
    UWorld* World;
    TWeakObjectPtr<UObject> Owner;  // 通常是 AI Controller
    FQueryFinishedSignature FinishDelegate;

    // === 配置数据（来自 EQA） ===
    TArray<FEnvQueryOptionInstance> Options;
    int32 OptionIndex;  // 当前执行的 Option

    // === 执行状态（状态机） ===
    int32 CurrentTest;  // -1=Generator阶段, >=0=Test阶段
    int32 CurrentTestStartingItem;  // 时间切片的进度保存

    // === 候选数据 ===
    TArray<FEnvQueryItem> Items;  // 候选项元数据
    TArray<uint8> RawData;        // 类型擦除的原始数据
    int32 NumValidItems;          // 有效项计数

    // === 上下文缓存 ===
    TArray<FEnvQueryInstanceContextCacheItem> ContextCache;

    // === 运行时参数 ===
    TMap<FName, float> NamedParams;
};
```

#### 工具：Generator / Test / Context
```
UEnvQueryGenerator  → 生成候选项
UEnvQueryTest       → 评分/过滤
UEnvQueryContext    → 提供参考对象
```

### 🔧 执行流程（状态机）

#### 状态转换图
```
[初始化]
    ↓
[Generator 阶段] (CurrentTest = -1)
    ↓ GenerateItems()
    ↓ FinalizeGeneration()
    ↓
[Test 阶段] (CurrentTest = 0, 1, 2, ...)
    ↓ RunTest()
    ↓ FinalizeTest()
    ↓ CurrentTest++
    ↓ (循环直到所有 Test 完成)
    ↓
[排序阶段]
    ↓ SortScores()
    ↓ PickBestItem(RunMode)
    ↓
[完成]
    ↓ MarkAsFinished()
    ↓ 触发 FinishDelegate
```

#### 核心方法：ExecuteOneStep()
```cpp
void FEnvQueryInstance::ExecuteOneStep(double TimeLimit) {
    const double StartTime = FPlatformTime::Seconds();

    // === 阶段 1: Generator ===
    if (CurrentTest < 0) {
        UEnvQueryGenerator* Generator = Options[OptionIndex].Generator;

        // 生成候选项
        Generator->GenerateItems(*this);

        // 检查异步状态
        if (!Generator->IsCurrentlyRunningAsync()) {
            FinalizeGeneration();  // 初始化 ItemDetails
            CurrentTest = 0;       // 进入 Test 阶段
        }
        return;
    }

    // === 阶段 2: Tests ===
    if (CurrentTest < Tests.Num()) {
        UEnvQueryTest* Test = Tests[CurrentTest];

        // 执行测试（可能只处理部分项目，时间切片）
        Test->RunTest(*this);

        // 检查异步状态
        if (!Test->IsCurrentlyRunningAsync()) {
            FinalizeTest();  // 归一化分数
            CurrentTest++;   // 下一个测试
        }

        // 检查时间限制
        if (FPlatformTime::Seconds() - StartTime > TimeLimit) {
            return;  // 保存进度，下一帧继续
        }
        return;
    }

    // === 阶段 3: 完成 ===
    if (NumValidItems > 0) {
        FinalizeQuery();  // 排序并选择最终结果
    } else {
        // 尝试下一个 Option，或标记失败
        TryNextOption();
    }
}
```

### 🔧 各组件的工作

#### 1. Generator 生成候选池
```cpp
// 例：SimpleGrid 生成器
void UEnvQueryGenerator_SimpleGrid::GenerateItems(FEnvQueryInstance& QueryInstance) {
    // 1. 获取参考位置（Context）
    TArray<FVector> ContextLocations;
    QueryInstance.PrepareContext(GenerateAround, ContextLocations);

    // 2. 围绕每个参考位置生成网格点
    for (const FVector& Center : ContextLocations) {
        for (int32 X = 0; X < GridSize; X++) {
            for (int32 Y = 0; Y < GridSize; Y++) {
                FVector GridPoint = Center + Offset(X, Y);

                // 3. 投影到导航网格
                FNavLocation NavLoc;
                if (ProjectToNavigation(GridPoint, NavLoc)) {
                    // 4. 添加到候选列表
                    QueryInstance.AddItemData<UEnvQueryItemType_Point>(NavLoc);
                }
            }
        }
    }
}
```

#### 2. Test 筛选/评分
```cpp
// 例：Distance 测试
void UEnvQueryTest_Distance::RunTest(FEnvQueryInstance& QueryInstance) {
    // 1. 获取参考对象（Context）
    TArray<FVector> ContextLocations;
    QueryInstance.PrepareContext(DistanceTo, ContextLocations);

    // 2. 遍历所有候选项（自动跳过已被过滤的）
    for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
    {
        const FVector ItemLocation = GetItemLocation(QueryInstance, It.GetIndex());

        // 3. 计算距离
        float MinDistance = MAX_FLT;
        for (const FVector& ContextLoc : ContextLocations) {
            float Distance = FVector::Dist(ItemLocation, ContextLoc);
            MinDistance = FMath::Min(MinDistance, Distance);
        }

        // 4. 设置分数（自动处理过滤 + 评分逻辑）
        It.SetScore(TestPurpose, FilterType, MinDistance,
                    FloatValueMin, FloatValueMax);
    }
}
```

#### 3. Context 提供参考对象
```cpp
// 例：Querier Context
void UEnvQueryContext_Querier::ProvideContext(
    FEnvQueryInstance& QueryInstance,
    FEnvQueryContextData& ContextData) const
{
    // 提供查询发起者的 Actor
    AActor* QueryOwner = Cast<AActor>(QueryInstance.Owner.Get());
    UEnvQueryItemType_Actor::SetContextHelper(ContextData, QueryOwner);
}

// 缓存机制
bool FEnvQueryInstance::PrepareContext(TSubclassOf<UEnvQueryContext> ContextClass,
                                        FEnvQueryContextData& ContextData)
{
    // 1. 检查缓存
    if (auto* Cached = ContextCache.FindByKey(ContextClass)) {
        ContextData = Cached->ContextData;
        return true;
    }

    // 2. 创建并调用
    UEnvQueryContext* ContextObj = Manager->PrepareLocalContext(ContextClass);
    ContextObj->ProvideContext(*this, ContextData);

    // 3. 缓存结果
    ContextCache.Add({ContextClass, ContextData});
    return true;
}
```

#### 4. 最终排序与选择
```cpp
void FEnvQueryInstance::FinalizeQuery() {
    // 1. 按分数降序排序
    Items.Sort([](const FEnvQueryItem& A, const FEnvQueryItem& B) {
        return A.Score > B.Score;
    });

    // 2. 根据 RunMode 选择结果
    switch (Mode) {
        case EEnvQueryRunMode::SingleResult:
            ResultItems.Add(Items[0]);  // 最优单项
            break;

        case EEnvQueryRunMode::RandomBest5Pct:
            int32 TopN = FMath::Max(1, Items.Num() * 0.05f);
            int32 RandomIndex = FMath::RandRange(0, TopN - 1);
            ResultItems.Add(Items[RandomIndex]);
            break;

        case EEnvQueryRunMode::AllMatching:
            ResultItems = Items;  // 全部
            break;
    }

    // 3. 标记完成
    MarkAsFinished();
}
```

### 📝 类比说明
```
就像做题的学生：
    📝 第一步（Generator）: 在草稿纸上写出所有可能的答案候选
        • SimpleGrid: 在地图上画出所有可能的位置点
        • ActorsOfClass: 列出所有敌人对象

    ✏️ 第二步（Tests）: 逐个检查候选，打分或划掉
        • Distance Test: 测量每个位置距离目标多远，打分
        • Trace Test: 检查是否被遮挡，不合格的划掉
        • Dot Test: 检查是否在视野内，打分

    📊 第三步（Finalize）: 把剩余候选排序，选出最优答案
        • SingleResult: 选最高分的
        • RandomBest5Pct: 从前 5% 里随机选一个

    ✅ 第四步（Callback）: 把答案交给老师（Manager），老师转交给出题人（AI）

学生不管什么时候做题、做多久，只管按题目要求一步步算。
```

---

## 🔄 三层协作流程

### 完整执行流程

```
1. [配置层] 设计师在编辑器中创建 EQA
   ↓ 保存为 QueryAsset.uasset

2. [调用方] 行为树 / C++ 发起查询请求
   ↓ RunQuery(QueryAsset, AIController, Callback)

3. [调度层] Manager 接收请求
   ↓ CreateInstance(QueryAsset)
   ↓ 添加到 RunningQueries[]

4. [调度层] Manager.Tick() 每帧驱动
   ↓ Instance->ExecuteOneStep(3ms)

5. [计算层] Instance 执行状态机
   ↓ Generator 生成候选
   ↓ Test1 筛选/评分
   ↓ Test2 筛选/评分
   ↓ TestN 筛选/评分
   ↓ 排序并选择结果

6. [计算层] Instance 标记完成
   ↓ FinishDelegate.Execute(Result)

7. [调用方] 收到回调
   ↓ 使用查询结果（移动到位置 / 攻击目标）
```

### 数据流向图

```
ConfigAsset.uasset
    ↓ (读取)
UEnvQuery (配置数据)
    ↓ (实例化)
FEnvQueryInstance (运行时状态)
    ↓ (生成)
Items[] + RawData[] (候选数据)
    ↓ (筛选/评分)
Items[].Score (打分结果)
    ↓ (排序)
SortedItems[] (排序结果)
    ↓ (选择)
FEnvQueryResult (最终结果)
    ↓ (回调)
AI/行为树 (使用结果)
```

### 时间线图

```
Frame 1:
    Manager.Tick()
        → Instance->ExecuteOneStep(3ms)
            → Generator->GenerateItems()
            → 生成 100 个候选点
        → 用时 2ms，返回

Frame 2:
    Manager.Tick()
        → Instance->ExecuteOneStep(3ms)
            → Test1->RunTest()
            → 处理 50 个候选项
        → 用时 3ms，保存进度，返回

Frame 3:
    Manager.Tick()
        → Instance->ExecuteOneStep(3ms)
            → Test1->RunTest()
            → 处理剩余 50 个候选项
            → FinalizeTest()
        → 用时 2.5ms，返回

Frame 4:
    Manager.Tick()
        → Instance->ExecuteOneStep(3ms)
            → Test2->RunTest()
            → 处理全部候选项
            → FinalizeTest()
            → FinalizeQuery()
            → MarkAsFinished()
        → 触发 FinishDelegate
        → 从队列移除
```

---

## 🎯 关键设计点

### 1. 职责边界清晰

| 层次 | 职责 | 不做什么 |
|------|------|----------|
| **配置层** | 定义规则 | ❌ 不参与运行时 |
| **调度层** | 管理生命周期 | ❌ 不执行具体计算 |
| **计算层** | 执行算法 | ❌ 不管调度策略 |

### 2. 数据流单向

```
配置层 → 调度层 → 计算层 → 结果回调
(只读)   (驱动)   (执行)   (通知)
```

### 3. 时间控制分层

```
调度层: 控制"每帧总时间" (3ms)
计算层: 控制"单步执行时间" (保存进度)
```

### 4. 状态保存机制

```
Instance 保存:
    - CurrentTest: 当前测试索引
    - CurrentTestStartingItem: 当前项目索引
    - Items[]: 中间结果

使得查询可以跨多帧执行，不阻塞主线程
```

---

## 📚 总结

### 三层模型的核心价值

1. **关注点分离**
   - 配置层：业务规则
   - 调度层：资源管理
   - 计算层：算法实现

2. **单一职责**
   - 每层只做一件事
   - 职责边界清晰
   - 易于理解和维护

3. **灵活组合**
   - 配置层可视化编辑
   - 调度层性能可控
   - 计算层易于扩展

### 类比总结

```
┌─────────────┬────────────────┬─────────────────┐
│    层次     │      类比      │    核心工作      │
├─────────────┼────────────────┼─────────────────┤
│  配置层     │   试卷（题目）  │  定义 What       │
│  调度层     │   监考老师      │  管理 When/Who   │
│  计算层     │   做题学生      │  执行 How        │
└─────────────┴────────────────┴─────────────────┘
```

### 扩展理解

这个三层模型不仅适用于 EQS，也是很多系统的通用模式：

```
配置驱动系统 = 数据配置 + 调度框架 + 执行引擎
```

例如：
- **行为树**: BehaviorTree Asset + BTComponent + Task Nodes
- **动画蓝图**: AnimBlueprint + AnimInstance + AnimNodes
- **材质系统**: Material Asset + MaterialInstance + ShaderNodes

这种分层思想是 UE 引擎架构的核心设计哲学之一。
