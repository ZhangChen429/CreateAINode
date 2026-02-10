# UE EQS 扩展设计：原则、反模式与决策指南

## 一、扩展点的核心问题

### 1.1 为什么需要扩展？

EQS 系统的核心问题：**"AI 如何在游戏世界中做空间决策？"**

分解为四个子问题：

```
问题 1: 候选项从哪来？           → 需要扩展 Generator
问题 2: 候选项怎么评价？         → 需要扩展 Test
问题 3: "好"是相对于什么？       → 需要扩展 Context
问题 4: 候选项存储什么数据？     → 需要扩展 ItemType
```

### 1.2 扩展决策树

```
开始使用 EQS
    ↓
内置 Generator 够用吗？
    ├─ YES → 使用内置的 (SimpleGrid, ActorsOfClass...)
    └─ NO  → 需要扩展 Generator
        ↓
        生成逻辑简单吗？
            ├─ YES → 用蓝图扩展 (Generator_BlueprintBase)
            └─ NO  → 用 C++ 扩展 (继承 UEnvQueryGenerator)

内置 Test 够用吗？
    ├─ YES → 使用内置的 (Distance, Trace, Dot...)
    └─ NO  → 需要扩展 Test
        ↓
        只能 C++ 扩展 (继承 UEnvQueryTest)

需要自定义参考对象吗？
    ├─ NO  → 用内置 Context (Querier, Target...)
    └─ YES → 扩展 Context
        ↓
        逻辑简单吗？
            ├─ YES → 用蓝图 (Context_BlueprintBase)
            └─ NO  → 用 C++ (继承 UEnvQueryContext)

需要存储自定义数据吗？
    ├─ NO  → 用内置 ItemType (Point, Actor, Direction)
    └─ YES → 扩展 ItemType (只能 C++ 继承)
```

---

## 二、Generator 扩展：何时、何地、如何

### 2.1 扩展原则

#### ✅ 应该扩展的情况

**原则 1: 当内置生成器无法表达你的候选空间时**

```
场景示例:
    ❌ 内置无法满足:
        - 需要查询游戏特定系统（任务系统、道具系统）
        - 需要特殊几何形状（螺旋、波浪线）
        - 需要基于复杂规则生成（游戏逻辑相关）

    ✅ 内置可以满足:
        - 规则网格、圆形、锥形区域
        - 特定类型的 Actor
        - 可见/感知到的对象
```

**原则 2: 当生成需要访问游戏自定义系统时**

```
典型场景:
    ✅ 应该扩展:
        - 查询任务系统中的任务目标
        - 查询装备系统中的可拾取物品
        - 查询建造系统中的可建造位置
        - 查询智能对象系统中的交互点

    ❌ 不应该扩展:
        - 只是过滤标准 Actor（用内置 ActorsOfClass + Test）
        - 只是改变查询半径（用参数配置）
```

**原则 3: 保持生成的纯粹性**

```
Generator 应该:
    ✅ 快速生成大量候选项
    ✅ 简单过滤（导航网格投影、基本可达性）
    ✅ 无副作用（不修改游戏状态）

Generator 不应该:
    ❌ 执行昂贵计算（寻路、射线检测）
    ❌ 进行复杂评分（留给 Test）
    ❌ 修改游戏状态（标记、占用资源）
```

### 2.2 扩展位置

#### 蓝图扩展（快速原型）

```
位置: Content Browser → 蓝图类 → EnvQueryGenerator_BlueprintBase

何时使用:
    ✅ 原型阶段，快速验证想法
    ✅ 设计师主导的简单逻辑
    ✅ 不需要高性能
    ✅ 逻辑不涉及复杂 C++ API

扩展点:
    - DoItemGeneration (从位置列表)
    - DoItemGenerationFromActors (从 Actor 列表)

调用辅助方法:
    - AddGeneratedVector(Location)
    - AddGeneratedActor(Actor)
    - GetQuerier() 获取查询发起者
```

#### C++ 扩展（生产级）

```
位置: YourModule/Public/EQS/ (你的模块目录)

创建文件:
    - MyGenerator.h
    - MyGenerator.cpp

基类:
    UEnvQueryGenerator

必须重载:
    virtual void GenerateItems(FEnvQueryInstance& QueryInstance) const override;

必须设置:
    构造器中: ItemType = UEnvQueryItemType_XXX::StaticClass();

何时使用:
    ✅ 需要高性能
    ✅ 需要访问 C++ 系统/API
    ✅ 逻辑复杂
    ✅ 生产环境使用
```

### 2.3 反模式与陷阱

#### ❌ 反模式 1: 在 Generator 中做复杂评估

```cpp
// ❌ 错误做法
void UMyGenerator::GenerateItems(FEnvQueryInstance& QueryInstance) const
{
    for (const FVector& Pos : CandidatePositions)
    {
        // 反模式: 在 Generator 中计算复杂分数
        float PathCost = ExpensivePathfinding(Pos);  // 昂贵操作
        float Visibility = RaycastCheck(Pos);         // 昂贵操作

        if (PathCost < Threshold && Visibility > 0.5f) {  // 过早评估
            QueryInstance.AddItemData<UEnvQueryItemType_Point>(Pos);
        }
    }
}

问题:
    1. Generator 阶段执行昂贵操作，阻塞生成
    2. 评估逻辑写死在 Generator 中，无法复用
    3. 无法利用 EQS 的测试排序优化
```

```cpp
// ✅ 正确做法
void UMyGenerator::GenerateItems(FEnvQueryInstance& QueryInstance) const
{
    // Generator 只负责生成
    for (const FVector& Pos : CandidatePositions)
    {
        QueryInstance.AddItemData<UEnvQueryItemType_Point>(Pos);
    }
}

// 评估逻辑放在 Test 中
class UEnvQueryTest_PathCost : public UEnvQueryTest { ... }
class UEnvQueryTest_Visibility : public UEnvQueryTest { ... }

// 在 EQA 配置中组合
Query:
    Generator: MyGenerator
    Tests:
        - PathCost (Filter)
        - Visibility (Score)
```

**设计原则**: **职责分离** - Generator 生成，Test 评估

---

#### ❌ 反模式 2: 在 Generator 中访问不稳定的上下文

```cpp
// ❌ 错误做法
void UMyGenerator::GenerateItems(FEnvQueryInstance& QueryInstance) const
{
    AActor* Target = QueryInstance.Owner->GetBlackboardComponent()
                         ->GetValueAsObject("Target");

    if (!Target) {
        // 反模式: Generator 依赖可能无效的上下文
        return;  // 直接返回，没有候选项
    }

    // 基于 Target 生成
}

问题:
    1. 如果 Target 无效，整个查询失败
    2. 不灵活，无法换成其他 Context
    3. 黑板访问写死在代码中
```

```cpp
// ✅ 正确做法
class UMyGenerator : public UEnvQueryGenerator
{
    UPROPERTY(EditAnywhere, Category = Generator)
    TSubclassOf<UEnvQueryContext> GenerateAround;  // 配置时指定
};

void UMyGenerator::GenerateItems(FEnvQueryInstance& QueryInstance) const
{
    TArray<FVector> ContextLocations;
    if (!QueryInstance.PrepareContext(GenerateAround, ContextLocations))
        return;  // 优雅失败

    // 使用 Context 系统
    for (const FVector& Center : ContextLocations) {
        // 生成逻辑
    }
}

// 配置时可以选择:
//   - Querier (自己周围)
//   - Target (目标周围)
//   - Custom (自定义)
```

**设计原则**: **依赖抽象** - 通过 Context 系统解耦

---

#### ❌ 反模式 3: Generator 修改游戏状态

```cpp
// ❌ 错误做法
void UMyGenerator::GenerateItems(FEnvQueryInstance& QueryInstance) const
{
    TArray<AResource*> Resources = FindResources();

    for (AResource* Resource : Resources)
    {
        Resource->MarkAsConsidered();  // 反模式: 修改状态
        QueryInstance.AddItemData<UEnvQueryItemType_Actor>(Resource);
    }
}

问题:
    1. Generator 可能被多次调用（缓存、重试）
    2. 副作用导致状态不一致
    3. 难以调试
```

```cpp
// ✅ 正确做法
void UMyGenerator::GenerateItems(FEnvQueryInstance& QueryInstance) const
{
    TArray<AResource*> Resources = FindResources();

    // Generator 只读取，不修改
    for (AResource* Resource : Resources)
    {
        QueryInstance.AddItemData<UEnvQueryItemType_Actor>(Resource);
    }
}

// 修改状态在查询完成后的回调中
OnQueryFinished(TSharedPtr<FEnvQueryResult> Result)
{
    AActor* SelectedResource = Result->GetItemAsActor(0);
    SelectedResource->MarkAsUsed();  // 在这里修改状态
}
```

**设计原则**: **无副作用** - Generator 是纯函数

---

## 三、Test 扩展：何时、何地、如何

### 3.1 扩展原则

#### ✅ 应该扩展的情况

**原则 1: 当需要新的评价维度时**

```
场景示例:
    ✅ 应该扩展:
        - 游戏特定的评分逻辑（声望、任务相关性）
        - 复杂的几何计算（视野锥、声音传播）
        - 游戏系统集成（库存、技能冷却）
        - 多对象关系（团队协作、区域控制）

    ❌ 不应该扩展:
        - 简单距离计算（用内置 Distance）
        - 简单方向检查（用内置 Dot）
        - 简单射线检测（用内置 Trace）
```

**原则 2: 当内置 Test 的评分方程不够用时**

```
内置评分方程:
    - Linear
    - Square
    - InverseLinear
    - SquareRoot
    - Constant

需要扩展的情况:
    ✅ 需要自定义评分曲线（指数、对数、分段函数）
    ✅ 需要多变量评分（不是单一数值）
    ✅ 需要条件评分（根据状态改变评分逻辑）
```

**原则 3: Test 应该是无状态的**

```
Test 应该:
    ✅ 基于输入计算输出
    ✅ 可重复执行（幂等性）
    ✅ 无副作用

Test 不应该:
    ❌ 存储查询之间的状态
    ❌ 修改候选项数据
    ❌ 修改游戏世界状态
```

### 3.2 扩展位置

#### C++ 扩展（唯一方式）

```
位置: YourModule/Public/EQS/Tests/

创建文件:
    - MyTest.h
    - MyTest.cpp

基类:
    UEnvQueryTest

必须重载:
    virtual void RunTest(FEnvQueryInstance& QueryInstance) const override;

必须设置:
    构造器中:
        ValidItemType = UEnvQueryItemType_XXX::StaticClass();
        Cost = EEnvTestCost::Low / Medium / High;

为什么没有蓝图扩展？
    - Test 是性能热点
    - 需要精确控制执行
    - 通常涉及底层 API
```

### 3.3 反模式与陷阱

#### ❌ 反模式 1: Test 中生成新的候选项

```cpp
// ❌ 错误做法
void UMyTest::RunTest(FEnvQueryInstance& QueryInstance) const
{
    for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
    {
        const FVector ItemLoc = GetItemLocation(QueryInstance, It.GetIndex());

        // 反模式: 在 Test 中生成新候选
        if (ShouldGenerateNearby(ItemLoc))
        {
            QueryInstance.AddItemData<UEnvQueryItemType_Point>(ItemLoc + Offset);
        }
    }
}

问题:
    1. 违反职责分离
    2. 破坏执行语义（Test 阶段不应该修改候选集）
    3. 难以预测结果
```

```cpp
// ✅ 正确做法
// 如果需要动态生成，在 Generator 中做
void UMyGenerator::GenerateItems(FEnvQueryInstance& QueryInstance) const
{
    TArray<FVector> BasePositions = GetBasePositions();

    for (const FVector& Base : BasePositions)
    {
        QueryInstance.AddItemData<UEnvQueryItemType_Point>(Base);

        // 可以在 Generator 中生成相关候选
        if (bGenerateNearby)
        {
            QueryInstance.AddItemData<UEnvQueryItemType_Point>(Base + Offset);
        }
    }
}
```

**设计原则**: **只读访问** - Test 只读取和评分，不修改集合

---

#### ❌ 反模式 2: 不使用迭代器，直接访问数组

```cpp
// ❌ 错误做法
void UMyTest::RunTest(FEnvQueryInstance& QueryInstance) const
{
    // 反模式: 直接遍历数组
    for (int32 i = 0; i < QueryInstance.Items.Num(); i++)
    {
        if (QueryInstance.Items[i].IsDiscarded())
            continue;  // 手动跳过

        // 访问数据
        const uint8* RawData = QueryInstance.RawData.GetData() +
                               QueryInstance.Items[i].DataOffset;
        // 手动类型转换...
    }
}

问题:
    1. 代码冗长
    2. 容易出错（忘记跳过 Discarded）
    3. 打分逻辑需要手动实现
    4. 不利用框架提供的辅助
```

```cpp
// ✅ 正确做法
void UMyTest::RunTest(FEnvQueryInstance& QueryInstance) const
{
    // 使用迭代器
    for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
    {
        // 自动跳过 Discarded
        // 提供便捷的访问接口
        const FVector ItemLoc = GetItemLocation(QueryInstance, It.GetIndex());

        float Score = CalculateScore(ItemLoc);

        // 自动处理过滤和评分
        It.SetScore(TestPurpose, FilterType, Score, MinThreshold, MaxThreshold);
    }
}
```

**设计原则**: **使用抽象** - 利用框架提供的迭代器

---

#### ❌ 反模式 3: 忘记设置 Cost 导致性能问题

```cpp
// ❌ 错误做法
class UMyExpensiveTest : public UEnvQueryTest
{
public:
    UMyExpensiveTest()
    {
        ValidItemType = UEnvQueryItemType_Point::StaticClass();
        // 反模式: 忘记设置 Cost，默认为 Low
    }

    virtual void RunTest(FEnvQueryInstance& QueryInstance) const override
    {
        for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
        {
            // 执行昂贵的寻路操作
            float PathCost = ExpensivePathfinding(...);
            It.SetScore(...);
        }
    }
};

问题:
    1. EQS 认为这是低成本 Test
    2. 可能不会优先执行 Filter
    3. 做了很多不必要的昂贵计算
```

```cpp
// ✅ 正确做法
class UMyExpensiveTest : public UEnvQueryTest
{
public:
    UMyExpensiveTest()
    {
        ValidItemType = UEnvQueryItemType_Point::StaticClass();
        Cost = EEnvTestCost::High;  // 明确标记为高成本
        bCanRunAsync = true;         // 可选: 支持异步
    }

    virtual void RunTest(FEnvQueryInstance& QueryInstance) const override
    {
        // 昂贵操作
    }
};

结果:
    - EQS 会自动排序，先执行 Filter Test
    - 减少需要执行昂贵 Test 的候选项数量
    - 性能显著提升
```

**设计原则**: **成本标注** - 准确标注 Test 的成本

---

#### ❌ 反模式 4: Test 之间有隐式依赖

```cpp
// ❌ 错误做法
class UTestA : public UEnvQueryTest
{
    void RunTest(FEnvQueryInstance& QueryInstance) const override
    {
        for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
        {
            // 计算 Score，并存储到某个全局缓存
            GlobalCache[It.GetIndex()] = CalculatedValue;
            It.SetScore(...);
        }
    }
};

class UTestB : public UEnvQueryTest
{
    void RunTest(FEnvQueryInstance& QueryInstance) const override
    {
        for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
        {
            // 反模式: 依赖 TestA 的缓存
            float ValueFromA = GlobalCache[It.GetIndex()];
            It.SetScore(...);
        }
    }
};

问题:
    1. TestB 依赖 TestA 先执行
    2. 但 EQS 可能自动排序改变顺序
    3. 结果不可预测
```

```cpp
// ✅ 正确做法
// 每个 Test 独立计算
class UTestA : public UEnvQueryTest
{
    void RunTest(FEnvQueryInstance& QueryInstance) const override
    {
        for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
        {
            float Score = CalculateA(...);
            It.SetScore(...);
        }
    }
};

class UTestB : public UEnvQueryTest
{
    void RunTest(FEnvQueryInstance& QueryInstance) const override
    {
        for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
        {
            float Score = CalculateB(...);  // 独立计算
            It.SetScore(...);
        }
    }
};

// 如果确实需要共享数据，使用 ItemType
class UMyGenerator : public UEnvQueryGenerator
{
    void GenerateItems(FEnvQueryInstance& QueryInstance) const override
    {
        FMyData Data;
        Data.PrecomputedValue = ExpensiveCalculation();

        QueryInstance.AddItemData<UEnvQueryItemType_MyData>(Data);
    }
};
```

**设计原则**: **Test 独立** - 每个 Test 应该独立工作

---

## 四、Context 扩展：何时、何地、如何

### 4.1 扩展原则

#### ✅ 应该扩展的情况

**原则 1: 当需要游戏特定的参考对象时**

```
场景示例:
    ✅ 应该扩展:
        - 任务系统中的任务目标
        - 团队系统中的队友/队长
        - 游戏模式特定的目标（夺旗点、占领区域）
        - 动态计算的位置（预测位置、重心）

    ❌ 不应该扩展:
        - 查询发起者（用 Querier）
        - 黑板中的目标（用 Target + 黑板配置）
        - 当前项目（用 Item）
```

**原则 2: Context 应该是轻量的**

```
Context 应该:
    ✅ 快速返回数据
    ✅ 可缓存结果
    ✅ 无副作用

Context 不应该:
    ❌ 执行昂贵计算
    ❌ 进行寻路或射线检测
    ❌ 修改游戏状态
```

### 4.2 扩展位置

#### 蓝图扩展（推荐）

```
位置: Content Browser → 蓝图类 → EnvQueryContext_BlueprintBase

何时使用:
    ✅ 逻辑简单
    ✅ 只需要访问蓝图 API
    ✅ 设计师主导

扩展点（选择其一实现）:
    - ProvideSingleActor
    - ProvideSingleLocation
    - ProvideActorsSet
    - ProvideLocationsSet
```

#### C++ 扩展

```
位置: YourModule/Public/EQS/Contexts/

基类:
    UEnvQueryContext

必须重载:
    virtual void ProvideContext(FEnvQueryInstance& QueryInstance,
                                 FEnvQueryContextData& ContextData) const override;
```

### 4.3 反模式与陷阱

#### ❌ 反模式 1: Context 中执行昂贵操作

```cpp
// ❌ 错误做法
void UMyContext::ProvideContext(FEnvQueryInstance& QueryInstance,
                                 FEnvQueryContextData& ContextData) const
{
    AAIController* AI = Cast<AAIController>(QueryInstance.Owner.Get());

    // 反模式: 在 Context 中执行昂贵操作
    TArray<FVector> PatrolPoints;
    for (const FVector& Point : AllPatrolPoints)
    {
        FPathFindingQuery Query(AI, Point);
        if (NavSys->TestPathSync(Query))  // 昂贵的寻路测试
        {
            PatrolPoints.Add(Point);
        }
    }

    UEnvQueryItemType_Point::SetContextHelper(ContextData, PatrolPoints);
}

问题:
    1. Context 可能被多个 Generator/Test 调用
    2. 每次调用都会执行昂贵操作（虽然有缓存）
    3. 阻塞查询初始化
```

```cpp
// ✅ 正确做法
void UMyContext::ProvideContext(FEnvQueryInstance& QueryInstance,
                                 FEnvQueryContextData& ContextData) const
{
    AAIController* AI = Cast<AAIController>(QueryInstance.Owner.Get());

    // Context 只快速返回数据
    TArray<FVector> PatrolPoints = AI->GetCachedPatrolPoints();

    // 或者返回所有点，让 Test 去筛选
    TArray<FVector> AllPoints = GetAllPatrolPoints();

    UEnvQueryItemType_Point::SetContextHelper(ContextData, AllPoints);
}

// 昂贵的筛选在 Test 中做
class UEnvQueryTest_Reachable : public UEnvQueryTest
{
    void RunTest(...) {
        // 在这里做寻路测试
    }
};
```

**设计原则**: **轻量快速** - Context 只提供数据，不做复杂计算

---

#### ❌ 反模式 2: Context 返回不稳定的数据

```cpp
// ❌ 错误做法
void UMyContext::ProvideContext(FEnvQueryInstance& QueryInstance,
                                 FEnvQueryContextData& ContextData) const
{
    // 反模式: 每次调用返回不同结果
    FVector RandomLocation = FMath::VRand() * 1000.0f;

    UEnvQueryItemType_Point::SetContextHelper(ContextData, RandomLocation);
}

问题:
    1. Context 被缓存，但第一次调用时是随机的
    2. 同一查询中，不同 Generator/Test 会看到相同的"随机"结果
    3. 结果不可复现
```

```cpp
// ✅ 正确做法
void UMyContext::ProvideContext(FEnvQueryInstance& QueryInstance,
                                 FEnvQueryContextData& ContextData) const
{
    // 返回稳定的数据源
    AAIController* AI = Cast<AAIController>(QueryInstance.Owner.Get());
    FVector TargetLocation = AI->GetBlackboardComponent()
                                ->GetValueAsVector("LastSeenLocation");

    UEnvQueryItemType_Point::SetContextHelper(ContextData, TargetLocation);
}

// 如果确实需要随机，在 Test 中做
class UEnvQueryTest_Random : public UEnvQueryTest { ... }
```

**设计原则**: **稳定一致** - Context 在同一查询中应该返回相同数据

---

## 五、ItemType 扩展：何时、何地、如何

### 5.1 扩展原则

#### ✅ 应该扩展的情况

**原则 1: 当内置类型无法表达你的数据时**

```
内置类型:
    - Point (FNavLocation): 位置
    - Actor (TWeakObjectPtr<AActor>): Actor 引用
    - Direction (FRotator): 方向

需要扩展的情况:
    ✅ 复合数据（位置 + 属性）
    ✅ 自定义游戏对象（非 Actor 的对象）
    ✅ 需要预计算的数据（性能优化）
```

**原则 2: ItemType 扩展是最后的选择**

```
先尝试:
    1. 能用 Point 吗？（大多数情况）
    2. 能用 Actor 吗？（对象查询）
    3. 能在 Test 中计算吗？（按需计算）

只在必需时扩展 ItemType:
    ✅ Generator 中预计算昂贵数据，Test 中复用
    ✅ 需要存储非 Actor 的游戏对象
    ✅ 需要特殊的黑板集成
```

### 5.2 扩展位置

#### C++ 扩展（唯一方式）

```
位置: YourModule/Public/EQS/Items/

基类:
    - UEnvQueryItemType_VectorBase (位置类)
    - UEnvQueryItemType_ActorBase (对象类)

必须实现:
    1. typedef FValueType (定义存储类型)
    2. 构造器中设置 ValueSize
    3. static GetValue(RawData)
    4. static SetValue(RawData, Value)
    5. static SetContextHelper(...)

为什么没有蓝图扩展？
    - 涉及底层内存管理
    - 需要精确控制内存布局
    - 性能关键
```

### 5.3 反模式与陷阱

#### ❌ 反模式 1: 过度使用自定义 ItemType

```cpp
// ❌ 错误做法
// 为每个小的数据变化都创建新类型

struct FPositionWithDistance { FVector Pos; float Distance; };
class UEnvQueryItemType_PositionDistance : public UEnvQueryItemType { ... }

struct FPositionWithAngle { FVector Pos; float Angle; };
class UEnvQueryItemType_PositionAngle : public UEnvQueryItemType { ... }

struct FPositionWithHeight { FVector Pos; float Height; };
class UEnvQueryItemType_PositionHeight : public UEnvQueryItemType { ... }

问题:
    1. 类型爆炸
    2. 每个类型需要独立维护
    3. Generator 和 Test 需要匹配特定类型
```

```cpp
// ✅ 正确做法
// 使用内置 Point 类型，在 Test 中计算

// Generator 使用标准 Point
QueryInstance.AddItemData<UEnvQueryItemType_Point>(Location);

// Test 中按需计算
class UEnvQueryTest_Distance : public UEnvQueryTest
{
    void RunTest(...) {
        float Distance = CalculateDistance(...);
        It.SetScore(...);
    }
};

// 只在真正需要时才创建复合类型
struct FTacticalPosition {
    FNavLocation Location;
    float CoverQuality;
    float ElevationAdvantage;
    TArray<FVector> FiringAngles;
    bool bHasEscapeRoute;
};

// 当这些数据在 Generator 中预计算，并在多个 Test 中复用时
```

**设计原则**: **优先使用内置类型** - 自定义 ItemType 是最后的手段

---

#### ❌ 反模式 2: ItemType 中存储指针/引用

```cpp
// ❌ 错误做法
struct FBadItemData {
    FVector Location;
    AActor* CachedActor;  // 反模式: 裸指针
    TArray<FVector>* Positions;  // 反模式: 指针
};

问题:
    1. RawData 是按值复制的
    2. 指针在复制后可能失效
    3. 内存管理混乱
```

```cpp
// ✅ 正确做法
// 使用值类型或智能指针
struct FGoodItemData {
    FVector Location;
    TWeakObjectPtr<AActor> ActorRef;  // 智能指针
    TArray<FVector> Positions;        // 值类型
};
```

**设计原则**: **值语义** - ItemType 应该是可复制的值类型

---

## 六、扩展决策流程图

### 6.1 我应该扩展什么？

```
遇到问题: EQS 无法满足需求

    ↓

候选项从哪来？
    ├─ 内置 Generator 够用 → 使用内置
    └─ 不够用 → 扩展 Generator
        ├─ 简单逻辑 → 蓝图扩展
        └─ 复杂/性能要求 → C++ 扩展

    ↓

候选项怎么评价？
    ├─ 内置 Test 够用 → 使用内置
    └─ 不够用 → 扩展 Test (只能 C++)
        ├─ 成本低 → Cost = Low
        ├─ 成本中等 → Cost = Medium
        └─ 成本高 → Cost = High + 考虑异步

    ↓

参考对象是什么？
    ├─ 内置 Context 够用 → 使用内置
    └─ 不够用 → 扩展 Context
        ├─ 简单逻辑 → 蓝图扩展
        └─ 复杂逻辑 → C++ 扩展

    ↓

需要自定义数据类型？
    ├─ Point/Actor 够用 → 使用内置
    └─ 不够用 → 扩展 ItemType (只能 C++)
        ├─ 需要预计算 → 复合 ItemType
        └─ 需要非 Actor 对象 → 自定义 ItemType
```

### 6.2 蓝图 vs C++ 决策

```
Generator:
    蓝图 ✅  快速原型、简单逻辑、设计师主导
    C++  ✅  性能要求、复杂逻辑、生产环境

Test:
    蓝图 ❌  不支持
    C++  ✅  唯一选择

Context:
    蓝图 ✅  推荐（大多数情况）
    C++  ✅  需要访问底层 API

ItemType:
    蓝图 ❌  不支持
    C++  ✅  唯一选择
```

---

## 七、扩展的最佳实践

### 7.1 通用原则

#### 原则 1: 单一职责

```
✅ 好的设计:
    Generator: 生成候选
    Test: 评估候选
    Context: 提供数据
    ItemType: 定义类型

❌ 坏的设计:
    Generator 中评估
    Test 中生成
    Context 中修改状态
```

#### 原则 2: 无副作用

```
✅ 所有扩展点应该是"纯"的:
    - 相同输入 → 相同输出
    - 不修改全局状态
    - 不修改游戏世界
    - 可重复执行

❌ 避免副作用:
    - 修改黑板
    - 标记对象
    - 发送事件
    - 修改 AI 状态
```

#### 原则 3: 依赖抽象

```
✅ 通过 Context 系统解耦:
    UPROPERTY(EditAnywhere)
    TSubclassOf<UEnvQueryContext> DataSource;

❌ 硬编码依赖:
    AActor* Target = GetBlackboardComponent()->GetValueAsObject("Target");
```

#### 原则 4: 成本标注

```
✅ 准确标注 Test 成本:
    Cost = EEnvTestCost::Low;    // 简单计算
    Cost = EEnvTestCost::Medium;  // 单次射线
    Cost = EEnvTestCost::High;    // 寻路、批量射线

❌ 忘记标注或标注错误:
    默认 Low，但实际很昂贵
```

#### 原则 5: 优先组合

```
✅ 通过组合解决问题:
    Query:
        Generator: SimpleGrid
        Tests:
            - Distance (Filter)
            - Trace (Filter)
            - Pathfinding (Score)

❌ 创建新的巨型组件:
    GeneratorWithDistanceAndTraceAndPathfinding
```

### 7.2 命名规范

```
Generator:
    EnvQueryGenerator_<功能>
    例: EnvQueryGenerator_SmartObjects
        EnvQueryGenerator_TacticalPositions

Test:
    EnvQueryTest_<评价维度>
    例: EnvQueryTest_CoverQuality
        EnvQueryTest_TeamValue

Context:
    EnvQueryContext_<参考对象>
    例: EnvQueryContext_TeamLeader
        EnvQueryContext_PredictedLocation

ItemType:
    EnvQueryItemType_<数据类型>
    例: EnvQueryItemType_TacticalPosition
        EnvQueryItemType_Resource
```

---

## 八、常见问题与决策

### Q1: Generator 还是 Test？

```
问题: 某个逻辑应该放在 Generator 还是 Test？

决策规则:
    Generator:
        ✅ 决定候选集的范围
        ✅ 快速、轻量的过滤
        ✅ 基于空间/对象查询的逻辑

    Test:
        ✅ 昂贵的计算
        ✅ 评分逻辑
        ✅ 需要多维度组合的逻辑

示例:
    导航网格可达性 → Generator (快速投影)
    实际寻路成本 → Test (昂贵计算)
```

### Q2: 什么时候需要自定义 ItemType？

```
决策流程:
    1. Point 够用吗？ (90% 的情况)
        YES → 使用 Point
        NO → 继续

    2. Actor 够用吗？
        YES → 使用 Actor
        NO → 继续

    3. 需要在 Generator 中预计算并在多个 Test 中复用？
        YES → 考虑自定义 ItemType
        NO → 在 Test 中按需计算

只有第 3 步 YES 时才扩展 ItemType
```

### Q3: Context 还是 Test 参数？

```
问题: 参考数据应该通过 Context 还是 Test 参数提供？

Context:
    ✅ 动态的、运行时决定的数据
    ✅ 需要在多个 Generator/Test 中复用
    ✅ 可能有多个值（数组）

Test 参数 (FAIDataProvider):
    ✅ 配置时决定的常量
    ✅ 黑板中的简单值
    ✅ 单一数值

示例:
    敌人位置 → Context (动态、多个)
    安全距离阈值 → Test 参数 (配置常量)
```

### Q4: 蓝图还是 C++？

```
蓝图适合:
    ✅ 原型阶段
    ✅ 设计师主导
    ✅ 简单逻辑（< 10 个节点）
    ✅ 不需要高性能
    ✅ 只有 Generator/Context（Test 不支持）

C++ 适合:
    ✅ 生产环境
    ✅ 性能要求高
    ✅ 复杂逻辑
    ✅ 需要访问底层 API
    ✅ Test 和 ItemType（必须 C++）

建议:
    原型用蓝图 → 验证想法 → 性能优化时改 C++
```

---

## 九、反模式总结表

| 反模式 | 问题 | 正确做法 |
|--------|------|---------|
| **Generator 中做评估** | 职责混乱、无法复用 | 评估放 Test |
| **Generator 修改状态** | 副作用、难调试 | 保持纯函数 |
| **Test 中生成候选** | 破坏执行语义 | 生成放 Generator |
| **Test 直接访问数组** | 代码冗长、易错 | 使用 ItemIterator |
| **Test 忘记标注 Cost** | 性能问题 | 准确标注成本 |
| **Test 之间有依赖** | 执行顺序不可控 | 保持 Test 独立 |
| **Context 昂贵操作** | 阻塞初始化 | 保持轻量快速 |
| **Context 不稳定数据** | 结果不可预测 | 返回稳定数据 |
| **过度自定义 ItemType** | 类型爆炸 | 优先内置类型 |
| **ItemType 存指针** | 内存管理混乱 | 使用值类型 |

---

## 十、总结：设计原则金字塔

```
                    可复用
                   /      \
                 灵活      高效
                /            \
           可组合            无副作用
          /                        \
     单一职责                   依赖抽象
    /                                    \
职责分离  成本标注  纯函数  轻量快速  值语义  正交设计
```

### 核心要点

1. **职责分离**: Generator 生成、Test 评估、Context 提供、ItemType 定义
2. **单一职责**: 每个组件只做一件事
3. **无副作用**: 所有扩展点都是纯函数
4. **依赖抽象**: 通过 Context 系统解耦
5. **成本标注**: 准确标注 Test 成本
6. **优先组合**: 通过组合而非继承构建复杂性
7. **最小化接口**: 扩展点接口越小越好
8. **优先内置**: 先用内置，必要时才扩展

### 决策口诀

```
问自己三个问题:
1. 内置够用吗？ → 90% 的情况用内置
2. 职责对吗？ → Generator 生成，Test 评估
3. 有副作用吗？ → 必须是纯函数

遵循这三点，就不会出大错。
```
