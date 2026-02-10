# EQS 程序模型完整调用流程

## 模型总览

```
┌─────────────────────────────────────────────────────────────────┐
│                         EQA 层级（数据配置）                      │
│  ┌──────────┬──────────┬──────────┬──────────┐                  │
│  │ Context  │Generator │  Test    │ ItemType │                  │
│  │"相对谁"  │"生成啥"  │"怎么评"  │"存储啥"  │                  │
│  └──────────┴──────────┴──────────┴──────────┘                  │
└─────────────────────────────────────────────────────────────────┘
                          ↓ 1. 用户发起查询
┌─────────────────────────────────────────────────────────────────┐
│                  执行调度层（UEnvQueryManager）                   │
│                      全局调度器/监考老师                          │
└─────────────────────────────────────────────────────────────────┘
         ↓ 2. 创建实例              ↓ 3. 驱动执行
┌──────────────────────┐   ┌─────────────────────────────────────┐
│ Environment          │   │  核心计算层（FEnvQueryInstance）     │
│ Query Asset          │   │     评估流程/做题学生                 │
│ (配置数据)           │   │                                      │
│  ├─ Context          │   │  ├─ Context  (调用)                 │
│  ├─ Generator        │   │  ├─ Generator (调用)                │
│  ├─ Test             │   │  ├─ Test      (调用)                │
│  └─ ItemType         │   │  └─ ItemType  (调用)                │
└──────────────────────┘   └─────────────────────────────────────┘
         ↓ 4. 提供配置              ↓ 5. 返回结果
                        用户回调处理结果
```

---

## 阶段 1: 用户发起查询

### 步骤 1.1 用户代码调用

**代码位置**: 用户项目（行为树/C++/蓝图）

**做了什么**:
```
1. 创建查询请求对象
   - 指定查询模板（Environment Query Asset）
   - 指定查询发起者（Owner，通常是 AI Controller）
   - 指定运行模式（SingleResult/RandomBest5Pct/AllMatching）

2. 设置运行时参数（可选）
   - 通过 NamedParams 传递动态值
   - 例: SetFloatParam("SearchRadius", 1000.0f)

3. 注册完成回调
   - 绑定委托函数
   - 查询完成时自动调用

4. 发起查询
   - 调用 UEnvQueryManager::RunQuery()
   - 获得 QueryID（用于追踪）
```

**输入**:
- `UEnvQuery* QueryTemplate` - 查询配置资产
- `UObject* Owner` - 查询发起者
- `EEnvQueryRunMode::Type` - 运行模式
- `FQueryFinishedSignature` - 完成回调

**输出**:
- `int32 QueryID` - 查询标识符

**关键函数**:
```cpp
int32 QueryID = UEnvQueryManager::RunEQSQuery(
    this,                          // Querier
    CoverQueryAsset,               // Query Template
    this,                          // Owner
    EEnvQueryRunMode::SingleBestItem,
    OnQueryFinishedDelegate
);
```

---

## 阶段 2: 执行调度层（Manager）处理

### 步骤 2.1 Manager 接收查询

**代码位置**: `EnvQueryManager.cpp:398`

**做了什么**:
```
1. 验证输入参数
   - 检查 QueryTemplate 有效性
   - 检查 Owner 有效性
   - 验证查询配置完整性

2. 创建查询实例
   → 调用 CreateQueryInstance()

3. 配置实例运行环境
   - 设置 World 指针（游戏世界引用）
   - 设置 Owner（查询发起者）
   - 设置 FinishDelegate（回调函数）
   - 设置 NamedParams（运行时参数）
   - 分配唯一 QueryID

4. 添加到运行队列
   - 加入 RunningQueries[] 数组
   - 等待 Tick 驱动执行

5. 返回 QueryID 给用户
```

**数据流**:
```
输入:
    FEnvQueryRequest {
        QueryTemplate,
        Owner,
        RunMode,
        NamedParams
    }

处理:
    → CreateQueryInstance(Template, RunMode)
    → 配置实例
    → 添加到队列

输出:
    QueryID (int32)
```

---

### 步骤 2.2 Manager 创建/复用实例

**代码位置**: `EnvQueryManager.cpp:531`

**做了什么**:
```
1. 查找实例缓存（享元模式）
   - 检查 InstanceCache 中是否有相同 Template
   - 如果找到 → 复制已排序的实例（性能优化）

2. 如果缓存未命中，创建新实例
   a. 分配 FEnvQueryInstance 对象

   b. 初始化基本信息
      - QueryName（查询名称）
      - Mode（运行模式）

   c. 从 Template 复制 Options
      - 遍历 Template->Options[]
      - 每个 Option 包含:
        · Generator（生成器）
        · Tests[]（测试数组）

   d. 自动排序测试（如果启用）
      → 调用 SortTests()
      - 规则: Filter 优先于 Score
      - 规则: Low Cost 优先于 High Cost

   e. 设置 ItemType
      - 从 Generator->ItemType 获取
      - 设置 ValueSize（数据大小）

   f. 缓存新创建的实例
      - 添加到 InstanceCache
      - 后续相同查询可复用

3. 返回实例智能指针
```

**关键优化**:
- **实例缓存**: 相同 Template 只初始化一次
- **测试排序**: 自动优化执行顺序
- **智能指针**: 自动内存管理

---

### 步骤 2.3 Manager 每帧驱动

**代码位置**: `EnvQueryManager.cpp:274`

**做了什么**:
```
每帧 Tick() 执行:

1. 计算时间预算
   - MaxAllowedTestingTime（默认 3ms）
   - 避免单帧执行时间过长

2. 遍历运行中的查询队列
   for (Query in RunningQueries):

3. 驱动查询执行一小步
   → Query->ExecuteOneStep(TimeRemaining)

4. 记录执行时间
   - 计算本次 Step 消耗的时间
   - 从 TimeRemaining 扣除

5. 检查完成状态
   if (Query->IsFinished()):
      - 触发完成回调
        → Query->FinishDelegate.Execute(Query)
      - 从队列移除查询

6. 检查时间预算
   if (TimeRemaining <= 0):
      - 本帧剩余查询下一帧继续
      - break 跳出循环
```

**时间分配示例**:
```
帧 1: Query1 执行 3ms → 未完成，保存进度
帧 2: Query1 执行 2ms → 完成
      Query2 执行 1ms → 未完成
帧 3: Query2 执行 3ms → 完成
```

**关键机制**: **时间切片** - 避免阻塞主线程

---

## 阶段 3: 核心计算层（Instance）执行

### 步骤 3.1 执行入口: ExecuteOneStep

**代码位置**: `EnvQueryInstance.cpp:292`

**做了什么**:
```
状态机驱动执行:

1. 验证前置条件
   - 检查 Owner 是否有效
   - 检查 OptionIndex 是否越界

2. 获取当前 Option
   - CurrentOption = Options[OptionIndex]

3. 状态机分发（基于 CurrentTest）

   if (CurrentTest == -1):
      ┌─────────────────────────────┐
      │  Generator 阶段              │
      │  → ExecuteGenerator()       │
      └─────────────────────────────┘

   else if (CurrentTest < Tests.Num()):
      ┌─────────────────────────────┐
      │  Test 阶段                   │
      │  → ExecuteTest()            │
      └─────────────────────────────┘

   else:
      ┌─────────────────────────────┐
      │  Finalize 阶段               │
      │  → FinalizeQuery()          │
      └─────────────────────────────┘

4. 状态转换
   - Generator 完成 → CurrentTest = 0
   - Test 完成 → CurrentTest++
   - 所有 Test 完成 → 进入 Finalize

5. 异常处理
   - 如果当前 Option 失败 → 尝试下一个 Option
   - 如果所有 Option 失败 → MarkAsFailed()
```

**状态转换图**:
```
[Initializing]
    ↓
CurrentTest = -1 → [Generator 阶段]
    ↓ FinalizeGeneration()
CurrentTest = 0 → [Test 0]
    ↓ FinalizeTest()
CurrentTest = 1 → [Test 1]
    ↓ FinalizeTest()
CurrentTest = N → [Test N]
    ↓ FinalizeTest()
CurrentTest >= Tests.Num() → [Finalize]
    ↓
[Finished]
```

---

### 步骤 3.2 Generator 阶段执行

**代码位置**: `EnvQueryInstance.cpp:338`

**做了什么**:
```
┌────────────────────────────────────────────────────┐
│ 步骤 1: 检查异步状态                                │
├────────────────────────────────────────────────────┤
│ if (Generator->IsCurrentlyRunningAsync()):        │
│     return; // 等待异步完成                        │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 2: 调用 Generator 核心方法                     │
├────────────────────────────────────────────────────┤
│ Generator->GenerateItems(*this)                   │
│                                                    │
│ 内部调用:                                           │
│   1. PrepareContext(ContextClass)                 │
│      → Context->ProvideContext()                  │
│      → ItemType::SetContextHelper()               │
│      → 获取参考位置/对象                            │
│                                                    │
│   2. 生成候选项（业务逻辑）                         │
│      - 计算候选位置/对象                            │
│      - 例: 网格点、特定类型 Actor                   │
│                                                    │
│   3. AddItemData<ItemType>(Value)                 │
│      → ItemType::SetValue()                       │
│      → 写入 RawData[] 数组                         │
│      → 添加 Items[] 元数据                         │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 3: 再次检查异步状态                            │
├────────────────────────────────────────────────────┤
│ if (Generator->IsCurrentlyRunningAsync()):       │
│     return; // 等待下一帧                          │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 4: 完成生成阶段                                │
├────────────────────────────────────────────────────┤
│ FinalizeGeneration()                              │
│   - 初始化 ItemDetails[]（调试用）                 │
│   - 统计 NumValidItems                            │
│   - 检查是否有有效候选                             │
│   - CurrentTest = 0（进入 Test 阶段）              │
└────────────────────────────────────────────────────┘
```

**数据流**:
```
输入:
    无（从 QueryInstance 获取上下文）

处理:
    Context → 参考数据
    Generator 业务逻辑 → 候选项
    ItemType → 存储数据

输出:
    Items[] - 候选项元数据
    RawData[] - 候选项数据
    NumValidItems - 有效项计数
```

---

### 步骤 3.3 Context 调用详解

**代码位置**: `EnvQueryInstance.cpp:876`

**做了什么**:
```
PrepareContext(ContextClass, OutData) 调用流程:

┌────────────────────────────────────────────────────┐
│ 步骤 1: 检查缓存（享元模式）                        │
├────────────────────────────────────────────────────┤
│ for (CachedItem in ContextCache):                 │
│     if (CachedItem.Context == ContextClass):      │
│         提取缓存数据 → OutData                      │
│         return true; // 缓存命中                   │
└────────────────────────────────────────────────────┘
                     ↓ 缓存未命中
┌────────────────────────────────────────────────────┐
│ 步骤 2: 创建 Context 对象                           │
├────────────────────────────────────────────────────┤
│ ContextObject = Manager->PrepareLocalContext(Class)│
│   - 从对象池获取或新建                              │
│   - 轻量级对象，无状态                              │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 3: 调用 Context 核心方法                       │
├────────────────────────────────────────────────────┤
│ ContextData = {}                                  │
│ ContextObject->ProvideContext(*this, ContextData) │
│                                                    │
│ 内部逻辑（以 Querier 为例）:                       │
│   1. 获取查询发起者                                 │
│      Owner = Cast<AActor>(QueryInstance.Owner)    │
│                                                    │
│   2. 填充上下文数据                                 │
│      ItemType::SetContextHelper(ContextData, Owner)│
│        → 设置 ValueType                            │
│        → 分配内存 RawData                           │
│        → 调用 SetValue() 写入数据                   │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 4: 缓存结果                                    │
├────────────────────────────────────────────────────┤
│ ContextCache.Add({ContextClass, ContextData})     │
│   - 同一查询中，相同 Context 只计算一次             │
│   - 显著提升性能                                    │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 5: 提取数据到用户格式                          │
├────────────────────────────────────────────────────┤
│ ItemType::GetContextLocations(ContextData, OutData)│
│   - 从 RawData 提取位置/对象                        │
│   - 转换为 TArray<FVector> 或 TArray<AActor*>      │
└────────────────────────────────────────────────────┘
```

**Context 示例**:
- `Querier` → 查询发起者（AI）
- `Target` → 黑板中的目标
- `Item` → 当前被测试的候选项
- 自定义 Context → 游戏特定逻辑

---

### 步骤 3.4 AddItemData 调用详解

**代码位置**: `EnvQueryTypes.h:1176`

**做了什么**:
```
AddItemData<UEnvQueryItemType_Point>(FNavLocation) 调用流程:

┌────────────────────────────────────────────────────┐
│ 步骤 1: 类型验证                                    │
├────────────────────────────────────────────────────┤
│ check(ItemType->GetValueSize() == sizeof(Value)); │
│ check(ValueSize <= MaxValueSize);                 │
│   - 确保类型匹配                                    │
│   - 防止内存越界                                    │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 2: 更新内存统计（开始）                        │
├────────────────────────────────────────────────────┤
│ DEC_MEMORY_STAT_BY(STAT_AI_EQS_InstanceMemory, ...)│
│   - 临时减少统计（准备重新计算）                     │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 3: 分配内存                                    │
├────────────────────────────────────────────────────┤
│ DataOffset = RawData.AddZeroed(ValueSize);        │
│   - 在 RawData[] 数组末尾分配 ValueSize 字节        │
│   - 返回偏移量                                      │
│   - 例: FNavLocation 占 24 字节                    │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 4: 写入数据（类型擦除）                        │
├────────────────────────────────────────────────────┤
│ ItemType::SetValue(RawData + DataOffset, Value);  │
│                                                    │
│ 内部实现:                                           │
│   *reinterpret_cast<FNavLocation*>(RawData) = Value;│
│                                                    │
│ 本质: 将强类型数据写入字节数组                       │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 5: 添加元数据                                  │
├────────────────────────────────────────────────────┤
│ Items.Add(FEnvQueryItem(DataOffset));             │
│                                                    │
│ FEnvQueryItem 结构:                                │
│   - Score: 0.0f（初始分数）                        │
│   - DataOffset: 指向 RawData 的偏移                │
│   - bIsDiscarded: false（未过滤）                  │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 6: 更新内存统计（完成）                        │
├────────────────────────────────────────────────────┤
│ INC_MEMORY_STAT_BY(STAT_AI_EQS_InstanceMemory, ...)│
│   - 更新内存使用统计                                │
└────────────────────────────────────────────────────┘
```

**内存布局**:
```
Items[]:
    [0] {Score:0, DataOffset:0,  bDiscarded:false}
    [1] {Score:0, DataOffset:24, bDiscarded:false}
    [2] {Score:0, DataOffset:48, bDiscarded:false}
          ↓ DataOffset 指向 ↓
RawData[]:
    [0-23]   FNavLocation #0 (24 bytes)
    [24-47]  FNavLocation #1 (24 bytes)
    [48-71]  FNavLocation #2 (24 bytes)
```

**类型擦除的意义**:
- 统一内存管理（不需要为每种类型写不同逻辑）
- 支持异构类型（Point、Actor、Direction 等）
- 内存连续（缓存友好）

---

### 步骤 3.5 FinalizeGeneration

**代码位置**: `EnvQueryInstance.cpp:438`

**做了什么**:
```
Generator 完成后的清理工作:

┌────────────────────────────────────────────────────┐
│ 步骤 1: 初始化调试数据                              │
├────────────────────────────────────────────────────┤
│ #if USE_EQS_DEBUGGER                              │
│     ItemDetails.SetNum(Items.Num());              │
│     // 为每个候选项分配调试记录                     │
│ #endif                                            │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 2: 统计有效项                                  │
├────────────────────────────────────────────────────┤
│ NumValidItems = 0;                                │
│ for (Item in Items):                              │
│     if (!Item.IsDiscarded()):                     │
│         NumValidItems++;                          │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 3: 检查是否有候选项                            │
├────────────────────────────────────────────────────┤
│ if (NumValidItems == 0):                          │
│     // 当前 Option 失败                            │
│     OptionIndex++;  // 尝试下一个 Option           │
│     CurrentTest = -1;  // 重新开始 Generator       │
│     Items.Reset();  // 清空数据                    │
│     RawData.Reset();                              │
│     return;                                       │
└────────────────────────────────────────────────────┘
                     ↓ 有有效候选项
┌────────────────────────────────────────────────────┐
│ 步骤 4: 进入 Test 阶段                              │
├────────────────────────────────────────────────────┤
│ CurrentTest = 0;                                  │
│   - 状态转换到第一个 Test                           │
│   - 下次 ExecuteOneStep 将执行 Test                │
└────────────────────────────────────────────────────┘
```

---

### 步骤 3.6 Test 阶段执行

**代码位置**: `EnvQueryInstance.cpp:502`

**做了什么**:
```
┌────────────────────────────────────────────────────┐
│ 步骤 1: 检查异步状态                                │
├────────────────────────────────────────────────────┤
│ if (Test->IsCurrentlyRunningAsync()):             │
│     return; // 等待异步完成（例: 批量寻路）         │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 2: 设置单结果优化标志                          │
├────────────────────────────────────────────────────┤
│ if (Mode == SingleResult):                        │
│     Test->bPassOnSingleResult = true;             │
│     // 找到第一个满足条件的立即返回                  │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 3: 调用 Test 核心方法                          │
├────────────────────────────────────────────────────┤
│ Test->RunTest(*this)                              │
│                                                    │
│ 内部调用（以 Distance 为例）:                       │
│                                                    │
│ 3.1 获取 Owner（用于参数绑定）                      │
│     Owner = QueryInstance.Owner.Get()             │
│                                                    │
│ 3.2 绑定数据提供者参数                              │
│     FloatValueMin.BindData(Owner, QueryID);       │
│     FloatValueMax.BindData(Owner, QueryID);       │
│     float Min = FloatValueMin.GetValue();         │
│     float Max = FloatValueMax.GetValue();         │
│     // 从黑板或配置获取阈值                          │
│                                                    │
│ 3.3 准备 Context 数据                              │
│     → PrepareContext(DistanceTo, ContextLocations)│
│     → 获取参考位置（例: 敌人位置）                   │
│                                                    │
│ 3.4 遍历所有候选项                                  │
│     for (ItemIterator It(this, Instance); It; ++It)│
│     {                                             │
│         // 自动跳过 Discarded 的项                 │
│                                                    │
│ 3.5 获取候选项位置                                  │
│         → GetItemLocation(Instance, It.GetIndex())│
│         → ItemType::GetItemLocation(RawData)      │
│         → ItemType::GetValue(RawData)             │
│         → 返回 FVector                             │
│                                                    │
│ 3.6 计算测试值                                      │
│         float Distance = CalculateDistance(...)   │
│         // 业务逻辑：计算距离                        │
│                                                    │
│ 3.7 设置分数/过滤                                   │
│         → It.SetScore(Purpose, FilterType,        │
│                       Distance, Min, Max);        │
│         // 自动处理过滤和评分                        │
│     }                                             │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 4: 再次检查异步状态                            │
├────────────────────────────────────────────────────┤
│ if (Test->IsCurrentlyRunningAsync()):             │
│     return; // 等待下一帧                          │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 5: 完成 Test                                   │
├────────────────────────────────────────────────────┤
│ FinalizeTest()                                    │
│   - 归一化分数                                      │
│   - 更新调试数据                                    │
│   - 检查有效项                                      │
│   - CurrentTest++（进入下一个 Test）                │
└────────────────────────────────────────────────────┘
```

---

### 步骤 3.7 ItemIterator 工作机制

**代码位置**: `EnvQueryTypes.cpp:1023`

**做了什么**:
```
ItemIterator 构造和遍历:

┌────────────────────────────────────────────────────┐
│ 构造器: ItemIterator(Test, Instance)               │
├────────────────────────────────────────────────────┤
│ 1. 初始化索引                                       │
│    CurrentItem = StartingItemIndex ?? 0           │
│                                                    │
│ 2. 跳过已过滤的项                                   │
│    while (CurrentItem < Items.Num() &&            │
│           Items[CurrentItem].IsDiscarded()):      │
│        CurrentItem++;                             │
│                                                    │
│ 结果: CurrentItem 指向第一个有效项                  │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ operator++: 移动到下一项                            │
├────────────────────────────────────────────────────┤
│ do {                                              │
│     CurrentItem++;                                │
│ } while (CurrentItem < Items.Num() &&             │
│          Items[CurrentItem].IsDiscarded());       │
│                                                    │
│ 自动跳过 Discarded 项                               │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ GetIndex(): 获取当前索引                            │
├────────────────────────────────────────────────────┤
│ return CurrentItem;                               │
│   - 用于访问 Items[CurrentItem]                    │
│   - 用于访问 RawData[Items[CurrentItem].DataOffset]│
└────────────────────────────────────────────────────┘
```

**使用示例**:
```cpp
// Test 中的标准用法
for (ItemIterator It(this, QueryInstance); It; ++It)
{
    // It 自动跳过 Discarded 项
    int32 Index = It.GetIndex();
    FVector Loc = GetItemLocation(QueryInstance, Index);

    float Score = CalculateScore(Loc);
    It.SetScore(TestPurpose, FilterType, Score, Min, Max);
}
```

---

### 步骤 3.8 SetScore 详解

**代码位置**: `EnvQueryTypes.cpp:1050`

**做了什么**:
```
SetScore(Purpose, FilterType, Value, Min, Max) 处理流程:

┌────────────────────────────────────────────────────┐
│ 步骤 1: 获取当前项引用                              │
├────────────────────────────────────────────────────┤
│ FEnvQueryItem& Item = Items[CurrentItem];         │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 2: 过滤逻辑（如果需要）                        │
├────────────────────────────────────────────────────┤
│ if (Purpose == Filter || Purpose == FilterAndScore):│
│                                                    │
│     判断是否通过过滤:                               │
│     switch (FilterType):                          │
│         case Match:                               │
│             Pass = (Value == Min)                 │
│         case Range:                               │
│             Pass = (Value >= Min && Value <= Max) │
│                                                    │
│     if (!Pass):                                   │
│         Item.bIsDiscarded = true;  // 标记过滤     │
│         NumValidItems--;           // 减少计数     │
│                                                    │
│         单结果优化:                                 │
│         if (bPassOnSingleResult && NumValidItems==1):│
│             bFoundSingleResult = true;            │
│             // 找到唯一满足条件的，立即返回          │
│                                                    │
│         return;  // 过滤掉，不继续评分              │
└────────────────────────────────────────────────────┘
                     ↓ 通过过滤
┌────────────────────────────────────────────────────┐
│ 步骤 3: 评分逻辑（如果需要）                        │
├────────────────────────────────────────────────────┤
│ if (Purpose == Score || Purpose == FilterAndScore):│
│                                                    │
│     应用评分方程:                                   │
│     switch (ScoringEquation):                     │
│                                                    │
│         case Linear:                              │
│             Normalized = (Value - Min) / (Max - Min)│
│                                                    │
│         case InverseLinear:                       │
│             Normalized = (Max - Value) / (Max - Min)│
│                                                    │
│         case Square:                              │
│             Normalized = ((Value - Min) / Range)^2│
│                                                    │
│         case SquareRoot:                          │
│             Normalized = sqrt((Value - Min) / Range)│
│                                                    │
│         case Constant:                            │
│             Normalized = 1.0f                     │
│                                                    │
│     应用权重:                                       │
│     FinalScore = Normalized * ScoringFactor       │
│                                                    │
│     累加分数:                                       │
│     Item.Score += FinalScore;                     │
└────────────────────────────────────────────────────┘
```

**评分示例**:
```
假设:
    Value = 600
    Min = 500
    Max = 1500
    ScoringFactor = 2.0
    ScoringEquation = InverseLinear

计算:
    Normalized = (1500 - 600) / (1500 - 500)
               = 900 / 1000
               = 0.9

    FinalScore = 0.9 * 2.0 = 1.8

    Item.Score += 1.8
```

---

### 步骤 3.9 FinalizeTest

**代码位置**: `EnvQueryInstance.cpp:568`

**做了什么**:
```
Test 完成后的处理:

┌────────────────────────────────────────────────────┐
│ 步骤 1: 归一化分数（如果需要）                      │
├────────────────────────────────────────────────────┤
│ if (Test->TestPurpose != Filter):                 │
│     Test->NormalizeItemScores(*this);             │
│                                                    │
│ 归一化类型:                                         │
│   - Absolute: 归一化到 [0, 1]                      │
│   - RelativeToScores: 相对于当前所有项分数          │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 2: 记录调试数据                                │
├────────────────────────────────────────────────────┤
│ #if USE_EQS_DEBUGGER                              │
│     for (Item in Items):                          │
│         ItemDetails[i].TestResults.Add(Score);    │
│     PerformedTestNames.Add(Test->GetName());      │
│ #endif                                            │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 3: 检查是否还有有效项                          │
├────────────────────────────────────────────────────┤
│ if (NumValidItems == 0):                          │
│     // 所有候选都被过滤了                           │
│     OptionIndex++;  // 尝试下一个 Option           │
│     CurrentTest = -1;  // 重新开始                 │
│     Items.Reset();                                │
│     RawData.Reset();                              │
│     ContextCache.Reset();                         │
│     return;                                       │
└────────────────────────────────────────────────────┘
                     ↓ 还有有效项
┌────────────────────────────────────────────────────┐
│ 步骤 4: 进入下一个 Test                             │
├────────────────────────────────────────────────────┤
│ CurrentTest++;                                    │
│   - 如果 CurrentTest < Tests.Num()                 │
│     下次执行下一个 Test                             │
│   - 如果 CurrentTest >= Tests.Num()                │
│     进入 Finalize 阶段                             │
└────────────────────────────────────────────────────┘
```

---

### 步骤 3.10 Finalize Query

**代码位置**: `EnvQueryInstance.cpp:642`

**做了什么**:
```
所有 Test 完成后的最终处理:

┌────────────────────────────────────────────────────┐
│ 步骤 1: 排序候选项                                  │
├────────────────────────────────────────────────────┤
│ Items.Sort([](const FEnvQueryItem& A,             │
│               const FEnvQueryItem& B) {           │
│     return A.Score > B.Score;  // 降序             │
│ });                                               │
│                                                    │
│ 结果: Items[0] 是最高分                            │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 2: 根据 RunMode 选择结果                       │
├────────────────────────────────────────────────────┤
│ switch (Mode):                                    │
│                                                    │
│ case SingleResult:                                │
│     PickSingleItem(0);  // 最高分的单项            │
│                                                    │
│ case RandomBest5Pct:                              │
│     NumTop = CeilToInt(Items.Num() * 0.05);       │
│     RandomIndex = RandRange(0, NumTop-1);         │
│     PickSingleItem(RandomIndex);                  │
│     // 从最佳 5% 中随机选择                         │
│                                                    │
│ case RandomBest25Pct:                             │
│     NumTop = CeilToInt(Items.Num() * 0.25);       │
│     RandomIndex = RandRange(0, NumTop-1);         │
│     PickSingleItem(RandomIndex);                  │
│     // 从最佳 25% 中随机选择                        │
│                                                    │
│ case AllMatching:                                 │
│     for (i = 0; i < Items.Num(); i++):            │
│         if (!Items[i].IsDiscarded()):             │
│             PickSingleItem(i);                    │
│     // 返回所有未过滤的项                           │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 3: PickSingleItem 处理                         │
├────────────────────────────────────────────────────┤
│ void PickSingleItem(int32 ItemIndex):             │
│                                                    │
│   1. 复制 Item 元数据                              │
│      FEnvQueryItem ItemCopy = Items[ItemIndex];   │
│                                                    │
│   2. 复制关联数据                                   │
│      NewOffset = ResultRawData.AddZeroed(ValueSize);│
│      FMemory::Memcpy(                             │
│          ResultRawData + NewOffset,               │
│          RawData + ItemCopy.DataOffset,           │
│          ValueSize                                │
│      );                                           │
│                                                    │
│   3. 更新偏移                                       │
│      ItemCopy.DataOffset = NewOffset;             │
│                                                    │
│   4. 添加到结果                                     │
│      ResultItems.Add(ItemCopy);                   │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 4: 标记完成                                    │
├────────────────────────────────────────────────────┤
│ MarkAsFinished()                                  │
│   - 设置 bIsFinished = true                       │
│   - 记录总执行时间                                  │
│   - 准备触发回调                                    │
└────────────────────────────────────────────────────┘
```

---

## 阶段 4: 用户回调处理

### 步骤 4.1 Manager 触发回调

**代码位置**: `EnvQueryManager.cpp:308`

**做了什么**:
```
在 Manager::Tick() 中:

┌────────────────────────────────────────────────────┐
│ if (QueryInstance->IsFinished()):                 │
├────────────────────────────────────────────────────┤
│ 1. 触发用户回调                                     │
│    QueryInstance->FinishDelegate.ExecuteIfBound(  │
│        QueryInstance                              │
│    );                                             │
│    // 执行用户注册的回调函数                         │
│                                                    │
│ 2. 从运行队列移除                                   │
│    RunningQueries.RemoveAtSwap(Index);            │
│    // 释放资源                                      │
└────────────────────────────────────────────────────┘
```

---

### 步骤 4.2 用户回调函数

**代码位置**: 用户代码

**做了什么**:
```
void OnQueryFinished(TSharedPtr<FEnvQueryResult> Result):

┌────────────────────────────────────────────────────┐
│ 步骤 1: 检查查询是否成功                            │
├────────────────────────────────────────────────────┤
│ if (!Result->IsSuccessful()):                     │
│     // 查询失败处理                                 │
│     return;                                       │
└────────────────────────────────────────────────────┘
                     ↓ 成功
┌────────────────────────────────────────────────────┐
│ 步骤 2: 获取结果数据                                │
├────────────────────────────────────────────────────┤
│ 方式 1: 获取位置                                    │
│   FVector Location = Result->GetItemAsLocation(0);│
│   → ItemType::GetItemLocation(RawData)            │
│   → ItemType::GetValue(RawData)                   │
│   → 返回 FVector                                   │
│                                                    │
│ 方式 2: 获取 Actor                                 │
│   AActor* Actor = Result->GetItemAsActor(0);      │
│   → ItemType::GetActor(RawData)                   │
│   → 返回 AActor*                                   │
│                                                    │
│ 方式 3: 获取所有结果                                │
│   for (int32 i = 0; i < Result->GetItemsNum(); i++)│
│       Process(Result->GetItemAsLocation(i));      │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 3: 存储到黑板（可选）                          │
├────────────────────────────────────────────────────┤
│ FBlackboardKeySelector Key;                       │
│ Key.SelectedKeyName = "CoverLocation";            │
│                                                    │
│ Result->StoreInBlackboard(Key, Blackboard);       │
│   → ItemType::StoreInBlackboard(Key, BB, RawData) │
│   → Blackboard->SetValueAsVector(Key, Location);  │
└────────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────┐
│ 步骤 4: 使用结果                                    │
├────────────────────────────────────────────────────┤
│ // 根据业务逻辑使用查询结果                          │
│ AIController->MoveToLocation(Location);           │
│ // 或                                              │
│ BlackboardComponent->SetValueAsObject("Target", Actor);│
└────────────────────────────────────────────────────┘
```

---

## 完整数据流总结

### 数据流向图

```
┌─────────────────────────────────────────────────────┐
│ 用户请求                                             │
│   QueryTemplate + Owner + Params + Callback        │
└────────────┬────────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────────┐
│ Manager 处理                                         │
│   → CreateInstance(Template)                        │
│   → 配置 Instance (Owner, Params, Callback)         │
│   → 添加到 RunningQueries[]                          │
└────────────┬────────────────────────────────────────┘
             ↓ 每帧 Tick
┌─────────────────────────────────────────────────────┐
│ Instance 执行                                        │
│   → ExecuteOneStep(TimeLimit)                       │
└────────────┬────────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────────┐
│ Generator 阶段                                       │
│   Context → ContextData (参考位置/对象)              │
│   Generator 业务逻辑 → Candidates (候选项列表)        │
│   ItemType::SetValue → RawData[] (存储候选数据)       │
│   → Items[] (元数据) + RawData[] (数据)              │
└────────────┬────────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────────┐
│ Test 阶段 (循环 N 次)                                │
│   Context → ContextData (参考数据)                   │
│   ItemType::GetValue ← RawData[] (读取候选数据)       │
│   Test 业务逻辑 → TestValue (计算值)                 │
│   SetScore → Items[].Score (累加分数)                │
│             Items[].bDiscarded (过滤标记)            │
└────────────┬────────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────────┐
│ Finalize 阶段                                        │
│   → Items.Sort() (按分数排序)                        │
│   → PickBestItem(RunMode) (选择结果)                 │
│   → ResultItems[] + ResultRawData[] (最终结果)       │
└────────────┬────────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────────┐
│ 用户回调                                             │
│   FinishDelegate.Execute(Result)                    │
│   → GetItemAsLocation() / GetItemAsActor()          │
│   → ItemType::GetValue ← ResultRawData[]            │
│   → StoreInBlackboard() (可选)                       │
│   → 使用结果 (移动/攻击/等)                           │
└─────────────────────────────────────────────────────┘
```

---

## 调用频率与性能分析

### 调用频率表（假设 100 候选，3 Test）

| 阶段 | 方法 | 调用次数 | 说明 |
|------|------|---------|------|
| **用户层** | RunQuery | 1 | 发起查询 |
| **Manager** | Tick | N 帧 | 每帧驱动 |
| **Manager** | CreateInstance | 1 | 创建实例（可能缓存命中） |
| **Instance** | ExecuteOneStep | N 帧 | 状态机驱动 |
| **Generator** | GenerateItems | 1 | 生成候选 |
| **Generator** | PrepareContext | 1 | 获取生成中心 |
| **Generator** | AddItemData | 100 | 每个候选 |
| **Context** | ProvideContext | 4 (缓存) | Generator(1) + Test(3) |
| **ItemType** | SetValue | 100 | Generator 写入 |
| **Test** | RunTest | 3 | 每个 Test |
| **Test** | PrepareContext | 3 | 每个 Test |
| **ItemIterator** | 构造/遍历 | 3×100 | 每个 Test 遍历所有项 |
| **ItemType** | GetValue | 300+ | Test 读取（100×3） |
| **ItemType** | GetItemLocation | 300+ | Test 访问位置 |
| **Test** | SetScore | 300 | 评分/过滤 |
| **Finalize** | Sort | 1 | 排序 O(N log N) |
| **Finalize** | PickBestItem | 1-100 | 根据 RunMode |
| **用户回调** | GetItemAsLocation | 1-N | 获取结果 |
| **ItemType** | StoreInBlackboard | 0-1 | 可选存储 |

---

## 性能热点与优化

### 热点分析

```
🔥 热点 1: Test::RunTest (300 次调用)
   - 包含业务逻辑计算
   - ItemType::GetValue 频繁调用
   - 占总时间 60-80%

🔥 热点 2: ItemType::GetValue/GetItemLocation (300+ 次)
   - 类型转换开销
   - 内存访问模式
   - 占总时间 10-20%

🔥 热点 3: Finalize Sort (1 次，但 O(N log N))
   - 候选项多时开销大
   - 占总时间 5-15%

💰 缓存优化: Context 缓存
   - 原本可能调用 4 次 ProvideContext
   - 缓存后只计算 1-4 次（取决于不同 Context 数量）

💰 时间切片优化
   - 避免单帧执行时间过长
   - 3ms 限制保证帧率稳定

💰 测试排序优化
   - Filter Test 优先执行
   - 减少需要评分的候选项数量
   - 可减少 50%+ 计算量

💰 早停优化
   - SingleResult 模式
   - 找到第一个满足条件的立即返回
   - 最优情况节省 90%+ 时间
```

---

## 设计精髓总结

### 核心机制

1. **状态机驱动**
   - CurrentTest 控制执行阶段
   - 清晰的状态转换

2. **时间切片**
   - ExecuteOneStep(TimeLimit)
   - 避免阻塞主线程

3. **依赖注入**
   - 通过 QueryInstance 传递上下文
   - 无全局状态，支持并发

4. **类型擦除**
   - RawData[] 统一存储
   - ItemType 提供类型安全访问

5. **缓存优化**
   - Instance 缓存（Template 复用）
   - Context 缓存（单次查询复用）

6. **迭代器抽象**
   - ItemIterator 自动跳过无效项
   - 统一遍历接口

7. **策略组合**
   - Generator + Tests[] 灵活组合
   - 评分方程可配置

### 职责分离

```
Manager    → 全局调度、资源管理
Instance   → 执行引擎、状态管理
Generator  → 生成候选
Test       → 评估候选
Context    → 提供参考
ItemType   → 类型系统
```

每个组件职责单一，边界清晰，易于扩展和维护。
