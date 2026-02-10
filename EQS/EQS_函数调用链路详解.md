# UE EQS 函数调用链路详解

## 一、调用链路总览

### 1.1 完整调用栈

```
用户代码
    ↓ RunQuery()
UEnvQueryManager
    ↓ CreateQueryInstance()
    ↓ Tick()
    ↓ Instance->ExecuteOneStep()
FEnvQueryInstance
    ↓ [Generator 阶段]
    ↓ Generator->GenerateItems()
        ↓ PrepareContext()
        ↓ Context->ProvideContext()
            ↓ ItemType::SetContextHelper()
        ↓ AddItemData()
            ↓ ItemType::SetValue()
    ↓ [Test 阶段]
    ↓ Test->RunTest()
        ↓ PrepareContext()
        ↓ Context->ProvideContext()
        ↓ ItemIterator
            ↓ ItemType::GetValue()
            ↓ GetItemLocation() / GetItemActor()
            ↓ SetScore()
    ↓ [Finalize 阶段]
    ↓ FinalizeQuery()
        ↓ SortScores()
        ↓ PickBestItem()
            ↓ ItemType::GetValue()
    ↓ FinishDelegate.Execute()
用户回调
    ↓ StoreInBlackboard()
        ↓ ItemType::StoreInBlackboard()
```

---

## 二、阶段 1: 查询发起

### 2.1 用户代码发起查询

**位置**: 行为树、C++ 代码、蓝图

```cpp
// === 用户代码 ===
// 文件: MyAIController.cpp

void AMyAIController::FindCoverLocation()
{
    // 1. 创建查询请求
    FEnvQueryRequest Request(CoverQueryAsset, this);

    // 2. 设置运行时参数
    Request.SetFloatParam("MinDistance", 500.0f);

    // 3. 发起查询
    FQueryFinishedSignature Delegate;
    Delegate.BindDynamic(this, &AMyAIController::OnCoverQueryFinished);

    QueryID = UEnvQueryManager::RunEQSQuery(
        this,
        CoverQueryAsset,
        this,
        EEnvQueryRunMode::SingleBestItem,
        Delegate
    );
}
```

**调用流程**:
```
AMyAIController::FindCoverLocation()
    ↓
UEnvQueryManager::RunEQSQuery() (静态便捷方法)
    ↓
Manager->RunQuery()
```

---

## 三、阶段 2: Manager 处理

### 3.1 Manager 接收查询

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/EnvQueryManager.cpp:398`

```cpp
// === UEnvQueryManager::RunQuery ===

int32 UEnvQueryManager::RunQuery(
    const FEnvQueryRequest& Request,
    EEnvQueryRunMode::Type RunMode,
    FQueryFinishedSignature FinishDelegate)
{
    // 步骤 1: 验证输入
    if (!Request.QueryTemplate || !Request.Owner.IsValid())
    {
        return INDEX_NONE;
    }

    // 步骤 2: 创建查询实例
    TSharedPtr<FEnvQueryInstance> QueryInstance =
        CreateQueryInstance(Request.QueryTemplate, RunMode);

    if (!QueryInstance.IsValid())
    {
        return INDEX_NONE;
    }

    // 步骤 3: 配置实例
    QueryInstance->World = Request.GetWorld();
    QueryInstance->Owner = Request.Owner;
    QueryInstance->FinishDelegate = FinishDelegate;

    // 步骤 4: 设置运行时参数
    QueryInstance->NamedParams = Request.NamedParams;

    // 步骤 5: 添加到运行队列
    const int32 QueryID = NextQueryID++;
    QueryInstance->QueryID = QueryID;
    RunningQueries.Add(QueryInstance);

    return QueryID;
}
```

**关键点**:
- `CreateQueryInstance()` - 创建或复用查询实例
- 实例被添加到 `RunningQueries[]` 队列
- 返回 QueryID 给用户

---

### 3.2 Manager 创建实例

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/EnvQueryManager.cpp:531`

```cpp
// === UEnvQueryManager::CreateQueryInstance ===

TSharedPtr<FEnvQueryInstance> UEnvQueryManager::CreateQueryInstance(
    const UEnvQuery* Template,
    EEnvQueryRunMode::Type RunMode)
{
    // 步骤 1: 查找缓存
    FEnvQueryInstanceCache* CachedQuery = InstanceCache.FindByPredicate(
        [Template](const FEnvQueryInstanceCache& Cache) {
            return Cache.Template == Template;
        }
    );

    if (CachedQuery)
    {
        // 步骤 2: 复用已排序的实例
        TSharedPtr<FEnvQueryInstance> NewInstance =
            MakeShareable(new FEnvQueryInstance(*CachedQuery->Instance));

        NewInstance->Mode = RunMode;
        return NewInstance;
    }

    // 步骤 3: 创建新实例
    TSharedPtr<FEnvQueryInstance> NewInstance =
        MakeShareable(new FEnvQueryInstance());

    NewInstance->QueryName = Template->GetName();
    NewInstance->Mode = RunMode;

    // 步骤 4: 初始化 Options
    for (UEnvQueryOption* Option : Template->Options)
    {
        FEnvQueryOptionInstance OptionInstance;
        OptionInstance.Generator = Option->Generator;
        OptionInstance.Tests = Option->Tests;

        // 步骤 5: 自动排序测试（如果启用）
        if (Option->Generator->bAutoSortTests)
        {
            SortTests(OptionInstance.Tests);
        }

        NewInstance->Options.Add(OptionInstance);
    }

    // 步骤 6: 设置 ItemType
    if (NewInstance->Options.Num() > 0)
    {
        NewInstance->ItemType = NewInstance->Options[0].Generator->ItemType;
        NewInstance->ValueSize = GetDefault<UEnvQueryItemType>(
            NewInstance->ItemType)->GetValueSize();
    }

    // 步骤 7: 缓存实例
    InstanceCache.Add({Template, *NewInstance, Template->GetFName()});

    return NewInstance;
}
```

**关键点**:
- 实例缓存复用（享元模式）
- 测试自动排序（性能优化）
- ItemType 从 Generator 获取

---

### 3.3 Manager 驱动执行

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/EnvQueryManager.cpp:274`

```cpp
// === UEnvQueryManager::Tick ===

void UEnvQueryManager::Tick(float DeltaTime)
{
    // 步骤 1: 计算时间预算
    const double MaxAllowedSeconds = MaxAllowedTestingTime;
    double TimeLeft = MaxAllowedSeconds;

    // 步骤 2: 遍历运行中的查询
    for (int32 Index = RunningQueries.Num() - 1; Index >= 0; Index--)
    {
        TSharedPtr<FEnvQueryInstance>& QueryInstance = RunningQueries[Index];

        if (!QueryInstance.IsValid())
        {
            RunningQueries.RemoveAtSwap(Index, 1, false);
            continue;
        }

        // 步骤 3: 驱动查询执行一步
        const double StepStartTime = FPlatformTime::Seconds();

        QueryInstance->ExecuteOneStep(TimeLeft);

        const double StepDuration = FPlatformTime::Seconds() - StepStartTime;
        TimeLeft -= StepDuration;

        // 步骤 4: 检查是否完成
        if (QueryInstance->IsFinished())
        {
            // 步骤 5: 触发回调
            QueryInstance->FinishDelegate.ExecuteIfBound(QueryInstance);

            // 步骤 6: 从队列移除
            RunningQueries.RemoveAtSwap(Index, 1, false);
        }

        // 步骤 7: 时间预算用尽，下一帧继续
        if (TimeLeft <= 0.0)
        {
            break;
        }
    }
}
```

**关键点**:
- 时间切片执行（默认 3ms）
- `ExecuteOneStep()` - 核心执行入口
- 完成后触发回调

---

## 四、阶段 3: Instance 执行（核心）

### 4.1 执行入口：ExecuteOneStep

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/EnvQueryInstance.cpp:292`

```cpp
// === FEnvQueryInstance::ExecuteOneStep ===

void FEnvQueryInstance::ExecuteOneStep(double TimeLimit)
{
    // 步骤 1: 验证 Owner
    if (!Owner.IsValid())
    {
        MarkAsFailed();
        return;
    }

    // 步骤 2: 获取当前 Option
    if (OptionIndex >= Options.Num())
    {
        MarkAsFailed();  // 所有 Option 都失败
        return;
    }

    FEnvQueryOptionInstance& CurrentOption = Options[OptionIndex];

    // 步骤 3: 状态机分发
    if (CurrentTest < 0)
    {
        // ========== Generator 阶段 ==========
        ExecuteGenerator(CurrentOption.Generator);
    }
    else if (CurrentTest < CurrentOption.Tests.Num())
    {
        // ========== Test 阶段 ==========
        ExecuteTest(CurrentOption.Tests[CurrentTest]);
    }
    else
    {
        // ========== 完成阶段 ==========
        FinalizeQuery();
    }
}
```

**状态机**:
```
CurrentTest = -1  → Generator 阶段
CurrentTest >= 0  → Test 阶段 (0, 1, 2, ...)
CurrentTest >= N  → Finalize 阶段
```

---

### 4.2 Generator 阶段执行

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/EnvQueryInstance.cpp:338`

```cpp
// === FEnvQueryInstance::ExecuteGenerator ===

void FEnvQueryInstance::ExecuteGenerator(UEnvQueryGenerator* Generator)
{
    if (!Generator)
    {
        MarkAsFailed();
        return;
    }

    // 步骤 1: 检查异步状态
    if (Generator->IsCurrentlyRunningAsync())
    {
        // 异步未完成，等待下一帧
        return;
    }

    // 步骤 2: 首次执行 Generator
    if (CurrentTest == -1 && !bFoundSingleResult)
    {
        const double StartTime = FPlatformTime::Seconds();

        // ========== 调用 Generator 的核心方法 ==========
        Generator->GenerateItems(*this);

        const double EndTime = FPlatformTime::Seconds();
        TotalExecutionTime += (EndTime - StartTime);
    }

    // 步骤 3: 检查是否仍在异步执行
    if (Generator->IsCurrentlyRunningAsync())
    {
        return;
    }

    // 步骤 4: Generator 完成，进入 Finalize
    FinalizeGeneration();
}
```

---

### 4.3 Generator 内部调用链

**文件**: 用户自定义 Generator

```cpp
// === UMyCustomGenerator::GenerateItems ===
// 这是用户重载的方法

void UMyCustomGenerator::GenerateItems(FEnvQueryInstance& QueryInstance) const
{
    // ========== 调用点 1: 准备 Context ==========
    TArray<FVector> ContextLocations;
    QueryInstance.PrepareContext(GenerateAround, ContextLocations);

    // 内部调用: PrepareContext() → Context->ProvideContext()

    // 生成候选位置
    TArray<FVector> CandidateLocations;
    for (const FVector& Center : ContextLocations)
    {
        // 生成逻辑
        for (int32 i = 0; i < GridSize; i++)
        {
            FVector Point = Center + FVector(i * Spacing, 0, 0);
            CandidateLocations.Add(Point);
        }
    }

    // ========== 调用点 2: 添加候选项 ==========
    for (const FVector& Location : CandidateLocations)
    {
        QueryInstance.AddItemData<UEnvQueryItemType_Point>(FNavLocation(Location));
    }

    // 内部调用: AddItemData() → ItemType::SetValue()
}
```

---

### 4.4 Context 调用链

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/EnvQueryInstance.cpp:876`

```cpp
// === FEnvQueryInstance::PrepareContext ===

bool FEnvQueryInstance::PrepareContext(
    TSubclassOf<UEnvQueryContext> ContextClass,
    TArray<FVector>& ContextLocations)
{
    // 步骤 1: 检查缓存
    for (const FEnvQueryInstanceContextCacheItem& CachedItem : ContextCache)
    {
        if (CachedItem.Context == ContextClass)
        {
            // 缓存命中，提取数据
            UEnvQueryItemType_Point::GetContextLocations(
                CachedItem.ContextData,
                ContextLocations
            );
            return true;
        }
    }

    // 步骤 2: 缓存未命中，创建 Context 对象
    UEnvQueryContext* ContextObject =
        QueryManager->PrepareLocalContext(ContextClass);

    if (!ContextObject)
    {
        return false;
    }

    // 步骤 3: 准备数据容器
    FEnvQueryContextData ContextData;

    // ========== 调用 Context 的核心方法 ==========
    ContextObject->ProvideContext(*this, ContextData);

    // 步骤 4: 缓存结果
    FEnvQueryInstanceContextCacheItem CacheItem;
    CacheItem.Context = ContextClass;
    CacheItem.ContextData = ContextData;
    ContextCache.Add(CacheItem);

    // 步骤 5: 提取位置数据
    UEnvQueryItemType_Point::GetContextLocations(ContextData, ContextLocations);

    return true;
}
```

---

### 4.5 Context 内部实现

**文件**: 用户自定义 Context

```cpp
// === UEnvQueryContext_Querier::ProvideContext ===
// 文件: Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/Contexts/EnvQueryContext_Querier.cpp:11

void UEnvQueryContext_Querier::ProvideContext(
    FEnvQueryInstance& QueryInstance,
    FEnvQueryContextData& ContextData) const
{
    // 步骤 1: 获取查询发起者
    AActor* QueryOwner = Cast<AActor>(QueryInstance.Owner.Get());

    if (!QueryOwner)
    {
        return;
    }

    // ========== 调用 ItemType 的辅助方法 ==========
    UEnvQueryItemType_Actor::SetContextHelper(ContextData, QueryOwner);
}
```

---

### 4.6 ItemType 在 Context 中的调用

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/Items/EnvQueryItemType_Actor.cpp`

```cpp
// === UEnvQueryItemType_Actor::SetContextHelper ===

void UEnvQueryItemType_Actor::SetContextHelper(
    FEnvQueryContextData& ContextData,
    const AActor* SingleActor)
{
    // 步骤 1: 设置类型信息
    ContextData.ValueType = UEnvQueryItemType_Actor::StaticClass();
    ContextData.NumValues = 1;

    // 步骤 2: 分配内存
    ContextData.RawData.SetNumUninitialized(sizeof(FWeakObjectPtr));

    // 步骤 3: 写入数据
    FWeakObjectPtr ActorPtr(const_cast<AActor*>(SingleActor));

    // ========== 调用 ItemType 的 SetValue ==========
    SetValue(ContextData.RawData.GetData(), ActorPtr);
}

// === UEnvQueryItemType_Actor::SetValue ===

void UEnvQueryItemType_Actor::SetValue(
    uint8* RawData,
    const FWeakObjectPtr& Value)
{
    // 类型擦除：将 FWeakObjectPtr 写入字节数组
    *reinterpret_cast<FWeakObjectPtr*>(RawData) = Value;
}
```

**关键概念**:
- 类型擦除：所有数据存储为 `uint8*`
- `SetValue()` 负责类型安全的写入

---

### 4.7 AddItemData 调用链

**文件**: `Engine/Source/Runtime/AIModule/Public/EnvironmentQuery/EnvQueryTypes.h:1176`

```cpp
// === FEnvQueryInstance::AddItemData ===

template<typename TypeItem>
void FEnvQueryInstance::AddItemData(typename TypeItem::FValueType ItemValue)
{
    // 步骤 1: 验证类型
    check(GetDefault<TypeItem>()->GetValueSize() == sizeof(typename TypeItem::FValueType));
    check(GetDefault<TypeItem>()->GetValueSize() <= ValueSize);

    // 步骤 2: 更新统计
    DEC_MEMORY_STAT_BY(STAT_AI_EQS_InstanceMemory,
                       RawData.GetAllocatedSize() + Items.GetAllocatedSize());

    // 步骤 3: 分配内存
    const int32 DataOffset = RawData.AddZeroed(ValueSize);

    // ========== 调用 ItemType 的 SetValue ==========
    TypeItem::SetValue(RawData.GetData() + DataOffset, ItemValue);

    // 步骤 4: 添加元数据
    Items.Add(FEnvQueryItem(DataOffset));

    // 步骤 5: 更新统计
    INC_MEMORY_STAT_BY(STAT_AI_EQS_InstanceMemory,
                       RawData.GetAllocatedSize() + Items.GetAllocatedSize());
}
```

**示例调用**:
```cpp
// Generator 中
QueryInstance.AddItemData<UEnvQueryItemType_Point>(FNavLocation(Location));

// 展开为:
// 1. ValueSize = sizeof(FNavLocation)
// 2. RawData.AddZeroed(24)  // FNavLocation 是 24 字节
// 3. UEnvQueryItemType_Point::SetValue(RawData + Offset, Location)
// 4. Items.Add(FEnvQueryItem(Offset))
```

---

### 4.8 Finalize Generation

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/EnvQueryInstance.cpp:438`

```cpp
// === FEnvQueryInstance::FinalizeGeneration ===

void FEnvQueryInstance::FinalizeGeneration()
{
    // 步骤 1: 初始化 ItemDetails（调试用）
#if USE_EQS_DEBUGGER
    ItemDetails.SetNum(Items.Num());
#endif

    // 步骤 2: 统计有效项
    NumValidItems = 0;
    for (const FEnvQueryItem& Item : Items)
    {
        if (!Item.IsDiscarded())
        {
            NumValidItems++;
        }
    }

    // 步骤 3: 检查是否有候选项
    if (NumValidItems == 0)
    {
        // 当前 Option 失败，尝试下一个
        OptionIndex++;
        CurrentTest = -1;
        Items.Reset();
        RawData.Reset();
        return;
    }

    // 步骤 4: 进入 Test 阶段
    CurrentTest = 0;
}
```

---

## 五、阶段 4: Test 执行

### 5.1 Test 执行入口

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/EnvQueryInstance.cpp:502`

```cpp
// === FEnvQueryInstance::ExecuteTest ===

void FEnvQueryInstance::ExecuteTest(UEnvQueryTest* Test)
{
    if (!Test)
    {
        CurrentTest++;
        return;
    }

    // 步骤 1: 检查异步状态
    if (Test->IsCurrentlyRunningAsync())
    {
        return;
    }

    // 步骤 2: 首次执行
    if (!bFoundSingleResult)
    {
        // 单结果优化
        const bool bWantsSingleResult = (Mode == EEnvQueryRunMode::SingleResult);
        Test->bPassOnSingleResult = bWantsSingleResult;

        const double StartTime = FPlatformTime::Seconds();

        // ========== 调用 Test 的核心方法 ==========
        Test->RunTest(*this);

        const double EndTime = FPlatformTime::Seconds();
        TotalExecutionTime += (EndTime - StartTime);
    }

    // 步骤 3: 检查是否仍在异步
    if (Test->IsCurrentlyRunningAsync())
    {
        return;
    }

    // 步骤 4: Test 完成
    FinalizeTest();
}
```

---

### 5.2 Test 内部调用链

**文件**: 用户自定义 Test

```cpp
// === UEnvQueryTest_Distance::RunTest ===
// 文件: Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/Tests/EnvQueryTest_Distance.cpp:15

void UEnvQueryTest_Distance::RunTest(FEnvQueryInstance& QueryInstance) const
{
    // 步骤 1: 获取 Owner（用于参数绑定）
    UObject* QueryOwner = QueryInstance.Owner.Get();
    if (!QueryOwner)
    {
        return;
    }

    // 步骤 2: 绑定数据提供者参数
    FloatValueMin.BindData(QueryOwner, QueryInstance.QueryID);
    FloatValueMax.BindData(QueryOwner, QueryInstance.QueryID);

    float MinThresholdValue = FloatValueMin.GetValue();
    float MaxThresholdValue = FloatValueMax.GetValue();

    // ========== 调用点 1: 准备 Context ==========
    TArray<FVector> ContextLocations;
    if (!QueryInstance.PrepareContext(DistanceTo, ContextLocations))
    {
        return;
    }

    // ========== 调用点 2: 迭代候选项 ==========
    for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
    {
        // ========== 调用点 3: 获取项目位置 ==========
        const FVector ItemLocation = GetItemLocation(QueryInstance, It.GetIndex());

        // 内部调用: GetItemLocation() → ItemType::GetItemLocation()

        // 步骤 3: 计算距离
        float MinDistance = BIG_NUMBER;
        for (const FVector& ContextLocation : ContextLocations)
        {
            const float Distance = FVector::Dist(ItemLocation, ContextLocation);
            MinDistance = FMath::Min(MinDistance, Distance);
        }

        // ========== 调用点 4: 设置分数 ==========
        It.SetScore(TestPurpose, FilterType, MinDistance,
                    MinThresholdValue, MaxThresholdValue);
    }
}
```

---

### 5.3 ItemIterator 工作原理

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/EnvQueryTypes.cpp:1023`

```cpp
// === FEnvQueryInstance::ItemIterator 构造器 ===

FEnvQueryInstance::ItemIterator::ItemIterator(
    const UEnvQueryTest* QueryTest,
    FEnvQueryInstance& QueryInstance,
    int32 StartingItemIndex)
    : Instance(QueryInstance)
    , CurrentItem(StartingItemIndex == INDEX_NONE ? 0 : StartingItemIndex)
    , Test(QueryTest)
{
    // 跳过已被过滤的项目
    while (CurrentItem < Instance.Items.Num() &&
           Instance.Items[CurrentItem].IsDiscarded())
    {
        CurrentItem++;
    }
}

// === operator++ ===

FEnvQueryInstance::ItemIterator& FEnvQueryInstance::ItemIterator::operator++()
{
    // 移动到下一个有效项
    do
    {
        CurrentItem++;
    }
    while (CurrentItem < Instance.Items.Num() &&
           Instance.Items[CurrentItem].IsDiscarded());

    return *this;
}
```

**关键点**:
- 自动跳过 `Discarded` 的项目
- 提供 `GetIndex()` 获取当前索引

---

### 5.4 GetItemLocation 调用链

**文件**: `Engine/Source/Runtime/AIModule/Classes/EnvironmentQuery/EnvQueryTest.h:357`

```cpp
// === UEnvQueryTest::GetItemLocation ===

FVector UEnvQueryTest::GetItemLocation(
    FEnvQueryInstance& QueryInstance,
    int32 ItemIndex) const
{
    // 验证索引
    check(QueryInstance.Items.IsValidIndex(ItemIndex));

    // 获取项目的数据偏移
    const FEnvQueryItem& Item = QueryInstance.Items[ItemIndex];
    const uint8* RawData = QueryInstance.RawData.GetData() + Item.DataOffset;

    // ========== 调用 ItemType 的 GetItemLocation ==========
    return GetDefault<UEnvQueryItemType>(QueryInstance.ItemType)
        ->GetItemLocation(RawData);
}
```

---

### 5.5 ItemType 在 Test 中的调用

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/Items/EnvQueryItemType_Point.cpp`

```cpp
// === UEnvQueryItemType_Point::GetItemLocation ===

FVector UEnvQueryItemType_Point::GetItemLocation(const uint8* RawData) const
{
    // ========== 调用 GetValue 获取数据 ==========
    const FNavLocation& NavLoc = GetValue(RawData);
    return NavLoc.Location;
}

// === UEnvQueryItemType_Point::GetValue ===

FNavLocation UEnvQueryItemType_Point::GetValue(const uint8* RawData)
{
    // 类型擦除逆向：从字节数组读取 FNavLocation
    return *reinterpret_cast<const FNavLocation*>(RawData);
}
```

---

### 5.6 ItemIterator::SetScore 调用链

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/EnvQueryTypes.cpp:1050`

```cpp
// === FEnvQueryInstance::ItemIterator::SetScore ===

void FEnvQueryInstance::ItemIterator::SetScore(
    EEnvTestPurpose::Type TestPurpose,
    EEnvTestFilterType::Type FilterType,
    float Score,
    float MinThreshold,
    float MaxThreshold)
{
    FEnvQueryItem& Item = Instance.Items[CurrentItem];

    // 步骤 1: 过滤逻辑
    if (TestPurpose == EEnvTestPurpose::Filter ||
        TestPurpose == EEnvTestPurpose::FilterAndScore)
    {
        bool bPassedFilter = false;

        switch (FilterType)
        {
        case EEnvTestFilterType::Match:
            bPassedFilter = (Score == MinThreshold);
            break;

        case EEnvTestFilterType::Range:
            bPassedFilter = (Score >= MinThreshold && Score <= MaxThreshold);
            break;
        }

        if (!bPassedFilter)
        {
            Item.bIsDiscarded = true;
            Instance.NumValidItems--;

            // 单结果模式：找到第一个满足条件的就返回
            if (Test->bPassOnSingleResult && Instance.NumValidItems == 1)
            {
                Instance.bFoundSingleResult = true;
            }

            return;
        }
    }

    // 步骤 2: 评分逻辑
    if (TestPurpose == EEnvTestPurpose::Score ||
        TestPurpose == EEnvTestPurpose::FilterAndScore)
    {
        // 根据评分方程计算分数
        float NormalizedScore = Score;

        switch (Test->ScoringEquation)
        {
        case EEnvTestScoreEquation::Linear:
            NormalizedScore = (Score - MinThreshold) / (MaxThreshold - MinThreshold);
            break;

        case EEnvTestScoreEquation::InverseLinear:
            NormalizedScore = (MaxThreshold - Score) / (MaxThreshold - MinThreshold);
            break;

        case EEnvTestScoreEquation::Square:
            NormalizedScore = FMath::Square((Score - MinThreshold) / (MaxThreshold - MinThreshold));
            break;

        case EEnvTestScoreEquation::SquareRoot:
            NormalizedScore = FMath::Sqrt((Score - MinThreshold) / (MaxThreshold - MinThreshold));
            break;

        case EEnvTestScoreEquation::Constant:
            NormalizedScore = 1.0f;
            break;
        }

        // 应用权重
        NormalizedScore *= Test->ScoringFactor.GetValue();

        // 累加分数
        Item.Score += NormalizedScore;
    }
}
```

---

### 5.7 Finalize Test

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/EnvQueryInstance.cpp:568`

```cpp
// === FEnvQueryInstance::FinalizeTest ===

void FEnvQueryInstance::FinalizeTest()
{
    // 步骤 1: 归一化分数（如果需要）
    if (CurrentTest >= 0 && CurrentTest < Options[OptionIndex].Tests.Num())
    {
        UEnvQueryTest* Test = Options[OptionIndex].Tests[CurrentTest];

        if (Test->TestPurpose != EEnvTestPurpose::Filter)
        {
            Test->NormalizeItemScores(*this);
        }
    }

    // 步骤 2: 更新调试数据
#if USE_EQS_DEBUGGER
    StoreDebugInfo();
#endif

    // 步骤 3: 检查是否还有有效项
    if (NumValidItems == 0)
    {
        // 当前 Option 失败，尝试下一个
        OptionIndex++;
        CurrentTest = -1;
        Items.Reset();
        RawData.Reset();
        ContextCache.Reset();
        return;
    }

    // 步骤 4: 进入下一个 Test
    CurrentTest++;
}
```

---

## 六、阶段 5: Finalize Query

### 6.1 最终排序和选择

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/EnvQueryInstance.cpp:642`

```cpp
// === FEnvQueryInstance::FinalizeQuery ===

void FEnvQueryInstance::FinalizeQuery()
{
    // 步骤 1: 排序候选项（按分数降序）
    Items.Sort([](const FEnvQueryItem& A, const FEnvQueryItem& B) {
        return A.Score > B.Score;
    });

    // 步骤 2: 根据 RunMode 选择结果
    switch (Mode)
    {
    case EEnvQueryRunMode::SingleResult:
        // 取最高分的单项
        if (Items.Num() > 0 && !Items[0].IsDiscarded())
        {
            PickSingleItem(0);
        }
        break;

    case EEnvQueryRunMode::RandomBest5Pct:
        {
            const int32 NumTop5Pct = FMath::Max(1, FMath::CeilToInt(Items.Num() * 0.05f));
            const int32 RandomIndex = FMath::RandRange(0, NumTop5Pct - 1);
            PickSingleItem(RandomIndex);
        }
        break;

    case EEnvQueryRunMode::RandomBest25Pct:
        {
            const int32 NumTop25Pct = FMath::Max(1, FMath::CeilToInt(Items.Num() * 0.25f));
            const int32 RandomIndex = FMath::RandRange(0, NumTop25Pct - 1);
            PickSingleItem(RandomIndex);
        }
        break;

    case EEnvQueryRunMode::AllMatching:
        // 返回所有未被过滤的项
        for (int32 i = 0; i < Items.Num(); i++)
        {
            if (!Items[i].IsDiscarded())
            {
                PickSingleItem(i);
            }
        }
        break;
    }

    // 步骤 3: 标记完成
    MarkAsFinished();
}

// === FEnvQueryInstance::PickSingleItem ===

void FEnvQueryInstance::PickSingleItem(int32 ItemIndex)
{
    // 将选中的项添加到结果列表
    FEnvQueryItem ItemCopy = Items[ItemIndex];

    // 复制关联的数据
    const int32 NewDataOffset = ResultRawData.AddZeroed(ValueSize);
    FMemory::Memcpy(
        ResultRawData.GetData() + NewDataOffset,
        RawData.GetData() + ItemCopy.DataOffset,
        ValueSize
    );

    ItemCopy.DataOffset = NewDataOffset;
    ResultItems.Add(ItemCopy);
}
```

---

## 七、阶段 6: 用户回调

### 7.1 回调触发

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/EnvQueryManager.cpp:308`

```cpp
// === UEnvQueryManager::Tick (回调部分) ===

if (QueryInstance->IsFinished())
{
    // 触发用户注册的回调
    QueryInstance->FinishDelegate.ExecuteIfBound(QueryInstance);

    // 从运行队列移除
    RunningQueries.RemoveAtSwap(Index, 1, false);
}
```

---

### 7.2 用户回调处理

**文件**: 用户代码

```cpp
// === AMyAIController::OnCoverQueryFinished ===

void AMyAIController::OnCoverQueryFinished(TSharedPtr<FEnvQueryResult> Result)
{
    // 步骤 1: 检查查询是否成功
    if (!Result->IsSuccessful())
    {
        UE_LOG(LogAI, Warning, TEXT("Cover query failed"));
        return;
    }

    // ========== 调用点 1: 获取结果位置 ==========
    FVector BestCoverLocation = Result->GetItemAsLocation(0);

    // 内部调用: GetItemAsLocation() → ItemType::GetItemLocation()

    // 步骤 2: 使用结果
    MoveToCover(BestCoverLocation);

    // ========== 调用点 2: 存储到黑板（可选） ==========
    UBlackboardComponent* Blackboard = GetBlackboardComponent();

    FBlackboardKeySelector CoverLocationKey;
    CoverLocationKey.SelectedKeyName = "CoverLocation";

    Result->StoreInBlackboard(CoverLocationKey, Blackboard);

    // 内部调用: StoreInBlackboard() → ItemType::StoreInBlackboard()
}
```

---

### 7.3 GetItemAsLocation 调用链

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/EnvQueryTypes.cpp:88`

```cpp
// === FEnvQueryResult::GetItemAsLocation ===

FVector FEnvQueryResult::GetItemAsLocation(int32 Index) const
{
    if (!Items.IsValidIndex(Index))
    {
        return FVector::ZeroVector;
    }

    // 获取项目数据
    const FEnvQueryItem& Item = Items[Index];
    const uint8* RawData = RawData.GetData() + Item.DataOffset;

    // ========== 调用 ItemType 的 GetItemLocation ==========
    return GetDefault<UEnvQueryItemType>(ItemType)->GetItemLocation(RawData);
}
```

---

### 7.4 StoreInBlackboard 调用链

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/EnvQueryTypes.cpp:146`

```cpp
// === FEnvQueryResult::StoreInBlackboard ===

bool FEnvQueryResult::StoreInBlackboard(
    FBlackboardKeySelector& KeySelector,
    UBlackboardComponent* Blackboard) const
{
    if (Items.Num() == 0 || !Blackboard)
    {
        return false;
    }

    // 获取第一个项目的数据
    const FEnvQueryItem& Item = Items[0];
    const uint8* RawData = RawData.GetData() + Item.DataOffset;

    // ========== 调用 ItemType 的 StoreInBlackboard ==========
    return GetDefault<UEnvQueryItemType>(ItemType)
        ->StoreInBlackboard(KeySelector, Blackboard, RawData);
}
```

---

### 7.5 ItemType::StoreInBlackboard

**文件**: `Engine/Source/Runtime/AIModule/Private/EnvironmentQuery/Items/EnvQueryItemType_Point.cpp`

```cpp
// === UEnvQueryItemType_Point::StoreInBlackboard ===

bool UEnvQueryItemType_Point::StoreInBlackboard(
    FBlackboardKeySelector& KeySelector,
    UBlackboardComponent* Blackboard,
    const uint8* RawData) const
{
    // ========== 调用 GetValue 读取数据 ==========
    const FNavLocation& NavLoc = GetValue(RawData);

    // 存储到黑板
    Blackboard->SetValueAsVector(KeySelector.SelectedKeyName, NavLoc.Location);

    return true;
}
```

---

## 八、时序图总结

### 8.1 完整时序图

```
用户代码                  Manager                  Instance                Generator/Test/Context        ItemType
   |                         |                        |                             |                       |
   |-- RunQuery() ---------->|                        |                             |                       |
   |                         |-- CreateInstance() --->|                             |                       |
   |                         |                        |-- 初始化                      |                       |
   |                         |<-----------------------|                             |                       |
   |<-- QueryID -------------|                        |                             |                       |
   |                         |                        |                             |                       |

   每帧 Tick:
                             |-- Tick() ------------->|                             |                       |
                             |                        |-- ExecuteOneStep()          |                       |
                             |                        |                             |                       |

   [Generator 阶段]
                             |                        |-- GenerateItems() --------->|                       |
                             |                        |                             |-- PrepareContext() -->|
                             |                        |<----------------------------|                       |
                             |                        |                             |-- ProvideContext() ---|----->
                             |                        |                             |<----------------------|      |
                             |                        |                             |                       |<-----|
                             |                        |<----------------------------|                       |
                             |                        |                             |                       |
                             |                        |-- AddItemData() ----------->|                       |----->
                             |                        |                             |                       |      |
                             |                        |<----------------------------|                       |<-----|
                             |                        |                             |                       |

   [Test 阶段]
                             |                        |-- RunTest() --------------->|                       |
                             |                        |                             |-- PrepareContext() -->|
                             |                        |                             |-- ItemIterator ------>|
                             |                        |                             |    GetItemLocation -->|----->
                             |                        |                             |<----------------------|      |
                             |                        |                             |                       |<-----|
                             |                        |                             |-- SetScore()          |
                             |                        |<----------------------------|                       |

   [Finalize 阶段]
                             |                        |-- FinalizeQuery()           |                       |
                             |                        |    SortScores()             |                       |
                             |                        |    PickBestItem()           |                       |
                             |                        |-- MarkAsFinished()          |                       |
                             |<-----------------------|                             |                       |

   回调:
   |<-- FinishDelegate ------|                        |                             |                       |
   |-- GetItemAsLocation() --|----------------------->|                             |                       |----->
   |                         |                        |                             |                       |      |
   |<-- FVector -------------|<-----------------------|                             |                       |<-----|
   |                         |                        |                             |                       |
   |-- StoreInBlackboard() --|----------------------->|                             |                       |----->
   |                         |                        |                             |                       |      |
   |<-- bool ---------------|<-----------------------|                             |                       |<-----|
```

---

## 九、调用频率统计

### 9.1 各组件调用次数

假设：
- 1 个查询
- 1 个 Option
- 100 个候选项
- 3 个 Test

| 组件/方法 | 调用次数 | 调用时机 |
|----------|---------|---------|
| **Manager::RunQuery** | 1 | 查询发起时 |
| **Manager::Tick** | N 帧 | 每帧（直到完成） |
| **Instance::ExecuteOneStep** | N 帧 | 每帧（直到完成） |
| **Generator::GenerateItems** | 1 | Generator 阶段 |
| **Context::ProvideContext** | 4 次 | Generator(1) + Test(3) |
| **ItemType::SetValue** | 100 | AddItemData 时 |
| **Test::RunTest** | 3 | 每个 Test |
| **ItemIterator** | 3 个迭代器 | 每个 Test |
| **ItemType::GetValue** | 300 | Test 中访问（100×3） |
| **ItemType::GetItemLocation** | 300 | Test 中访问 |
| **ItemIterator::SetScore** | 300 | Test 中评分 |
| **ItemType::StoreInBlackboard** | 1 | 用户回调 |

---

## 十、关键路径分析

### 10.1 性能热点

```
热点 1: Test::RunTest (300 次调用)
    - ItemIterator 遍历
    - ItemType::GetValue (每个项目)
    - 业务逻辑计算
    - SetScore

热点 2: ItemType::GetValue (300 次)
    - 类型转换
    - 内存访问

热点 3: Context::ProvideContext (4 次，但可能昂贵)
    - 取决于 Context 实现
    - 缓存机制减轻

热点 4: FinalizeQuery (1 次，但可能昂贵)
    - 排序 (O(N log N))
    - 选择结果
```

### 10.2 优化点

```
优化 1: Context 缓存
    - 相同 Context 只计算一次
    - PrepareContext 中实现

优化 2: 时间切片
    - ExecuteOneStep 限制时间
    - 避免单帧阻塞

优化 3: 测试排序
    - Filter 优先执行
    - 减少需要评分的项目数

优化 4: 早停机制
    - bPassOnSingleResult
    - 找到满足条件的项目立即返回

优化 5: 异步执行
    - bCanRunAsync
    - 昂贵操作异步化
```

---

## 十一、总结

### 11.1 核心调用链

```
1. 用户发起查询
   RunQuery() → 创建 Instance → 添加到队列

2. Manager 每帧驱动
   Tick() → ExecuteOneStep()

3. Generator 阶段
   GenerateItems()
   → PrepareContext() → Context::ProvideContext()
                      → ItemType::SetContextHelper() → SetValue()
   → AddItemData() → ItemType::SetValue()

4. Test 阶段 (循环 N 次)
   RunTest()
   → PrepareContext() → Context::ProvideContext()
   → ItemIterator
      → GetItemLocation() → ItemType::GetItemLocation() → GetValue()
      → SetScore()

5. Finalize 阶段
   FinalizeQuery() → 排序 → 选择

6. 用户回调
   FinishDelegate
   → GetItemAsLocation() → ItemType::GetItemLocation() → GetValue()
   → StoreInBlackboard() → ItemType::StoreInBlackboard()
```

### 11.2 四大组件的调用时机

| 组件 | 被谁调用 | 何时调用 | 调用次数 |
|------|---------|---------|---------|
| **Generator** | Instance | Generator 阶段开始 | 1 次/Option |
| **Test** | Instance | Test 阶段每个测试 | N 次/Option |
| **Context** | Generator/Test | 需要参考数据时 | 多次（缓存） |
| **ItemType** | 所有组件 | 读写候选数据时 | 频繁 |

### 11.3 设计精髓

1. **分层清晰**: Manager → Instance → Node
2. **职责分离**: 生成、评估、提供、存储
3. **依赖注入**: 通过 QueryInstance 传递上下文
4. **缓存优化**: Context 缓存、Instance 缓存
5. **时间切片**: 避免阻塞主线程
6. **类型擦除**: 统一内存管理

这就是 EQS 的完整函数调用链路！
