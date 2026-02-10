# Unreal Engine EQS 程序设计模型

## 一、系统概述

### 1.1 设计目标

EQS (Environment Query System) 是一个**空间推理引擎**，其核心设计目标：

- **声明式查询**: 用户描述"想要什么"，而非"如何实现"
- **高性能**: 在复杂场景中保持实时响应
- **可扩展**: 易于添加新的查询类型和测试逻辑
- **数据驱动**: 查询配置与代码解耦
- **可视化**: 支持实时调试和结果可视化

### 1.2 核心概念模型

```
空间查询 = Generator(候选生成) → Test(评分筛选) → Result(结果选择)
                      ↓
                 Context(参考对象)
```

**三大核心组件**:
- **Generator**: 生成候选项（WHERE）
- **Test**: 评估候选项（HOW GOOD）
- **Context**: 提供参考对象（RELATIVE TO WHAT）

---

## 二、架构设计

### 2.1 分层架构

```
┌─────────────────────────────────────────────┐
│  Application Layer (行为树/AI Controller)   │  用户接口层
├─────────────────────────────────────────────┤
│  Manager Layer (UEnvQueryManager)           │  全局调度层
│  - 查询调度                                  │
│  - 生命周期管理                              │
│  - 资源缓存                                  │
├─────────────────────────────────────────────┤
│  Execution Layer (FEnvQueryInstance)        │  执行引擎层
│  - 状态机驱动                                │
│  - 时间切片执行                              │
│  - 数据流管理                                │
├─────────────────────────────────────────────┤
│  Component Layer (Generator/Test/Context)   │  逻辑组件层
│  - 策略模式实现                              │
│  - 可插拔设计                                │
├─────────────────────────────────────────────┤
│  Data Layer (ItemType/RawData)              │  数据抽象层
│  - 类型擦除                                  │
│  - 内存管理                                  │
└─────────────────────────────────────────────┘
```

### 2.2 职责分离

| 层次 | 职责 | 生命周期 |
|------|------|----------|
| **Manager** | 全局资源管理、调度策略 | 单例，游戏级 |
| **Instance** | 查询执行、状态维护 | 临时，查询级 |
| **Node** | 业务逻辑实现 | 无状态，可复用 |
| **ItemType** | 类型系统、序列化 | 静态，类型级 |

---

## 三、核心设计模式

### 3.1 策略模式 (Strategy Pattern)

**问题**: 如何支持多种候选生成和评分策略？

**解决方案**: Generator/Test/Context 作为可互换的策略

```
UEnvQueryNode (抽象策略)
    ├── UEnvQueryGenerator (生成策略)
    │       ├── Grid
    │       ├── ActorsOfClass
    │       └── Custom...
    ├── UEnvQueryTest (评分策略)
    │       ├── Distance
    │       ├── Trace
    │       └── Custom...
    └── UEnvQueryContext (上下文策略)
            ├── Querier
            ├── Target
            └── Custom...
```

**优势**:
- 运行时策略组合
- 易于扩展新策略
- 策略可独立测试

### 3.2 组合模式 (Composite Pattern)

**问题**: 如何灵活组合多个测试条件？

**解决方案**: Option = Generator + Tests[]

```
UEnvQueryOption (组合节点)
    ├── Generator (1个)
    └── Tests[] (N个)
            ├── Test1 (过滤)
            ├── Test2 (评分)
            └── Test3 (评分)
```

**特点**:
- 线性管道式处理
- 支持过滤和评分混合
- 自动优化执行顺序

### 3.3 状态模式 (State Pattern)

**问题**: 如何管理查询的执行阶段？

**解决方案**: Instance 作为状态机

```
States:
    [Initializing] → [Generating] → [Testing] → [Finalizing] → [Finished]
                          ↓            ↓            ↓
                     [AsyncWait]  [AsyncWait]  [Sorting]

Transitions:
    CurrentTest = -1  → Generator Phase
    CurrentTest >= 0  → Test Phase
    CurrentTest >= N  → Finalization
```

**状态数据**:
- `CurrentTest`: 当前测试索引 (-1 表示 Generator 阶段)
- `NumValidItems`: 有效项目计数
- `bIsFinished`: 完成标志

### 3.4 迭代器模式 (Iterator Pattern)

**问题**: 如何统一遍历不同类型的候选项？

**解决方案**: FItemIterator 抽象

```cpp
for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
{
    // 自动跳过被丢弃的项目
    // 提供评分接口 It.SetScore()
    // 提供数据访问 It.GetItemData()
}
```

**特点**:
- 隐藏内部存储结构
- 支持过滤逻辑
- 提供便捷的评分接口

### 3.5 模板方法模式 (Template Method Pattern)

**问题**: 如何标准化节点的执行流程？

**解决方案**: Node 基类定义执行骨架

```
UEnvQueryNode::Execute() {
    1. PreExecute()      // 钩子: 初始化
    2. DoExecute()       // 抽象: 核心逻辑 (子类实现)
    3. PostExecute()     // 钩子: 清理
    4. NormalizeScores() // 钩子: 归一化
}
```

**扩展点**:
- `GenerateItems()` - Generator 实现
- `RunTest()` - Test 实现
- `ProvideContext()` - Context 实现

### 3.6 享元模式 (Flyweight Pattern)

**问题**: 如何避免重复创建相似的查询实例？

**解决方案**: 实例缓存复用

```
InstanceCache[] {
    {Template, PreConfiguredInstance}
}

CreateInstance(Template):
    if (Cached = Find(Template))
        return Clone(Cached.Instance)
    else
        return CreateNew(Template)
```

**共享数据**:
- 查询结构 (Options, Tests)
- 排序后的测试顺序
- 类型信息

**独立数据**:
- 执行上下文 (Owner, World)
- 运行时参数
- 查询结果

### 3.7 数据驱动设计 (Data-Driven Design)

**问题**: 如何让非程序员配置复杂查询？

**解决方案**: UEnvQuery 作为 DataAsset

```
QueryAsset.uasset (数据)
    ↓
UEnvQuery (运行时)
    ↓
FEnvQueryInstance (执行)
```

**优势**:
- 可视化编辑器
- 热重载支持
- 版本控制友好
- 复用性高

---

## 四、数据流设计

### 4.1 查询执行流

```
┌─────────────┐
│ RunQuery()  │ ← 输入: QueryTemplate, Owner, Params
└──────┬──────┘
       │
       ↓
┌─────────────────┐
│ CreateInstance  │ ← 缓存查找或创建
└──────┬──────────┘
       │
       ↓
┌─────────────────┐
│ Generator Phase │ → Items[] (候选集)
└──────┬──────────┘
       │
       ↓
┌─────────────────┐
│ Test Phase      │ → Items[].Score (评分)
│  - Test 1       │ → Items[].bDiscarded (过滤)
│  - Test 2       │
│  - Test N       │
└──────┬──────────┘
       │
       ↓
┌─────────────────┐
│ Finalize Phase  │ → 排序 + 选择
└──────┬──────────┘
       │
       ↓
┌─────────────────┐
│ Callback        │ ← 输出: Result
└─────────────────┘
```

### 4.2 数据存储模型

**类型擦除设计**:

```
FEnvQueryItem {
    Score: float           // 元数据
    DataOffset: int        // 指向 RawData
    bDiscarded: bool       // 状态标志
}

Items[] ─┐
         │
         ↓
RawData[] {              // 类型擦除存储
    [Offset0]: FNavLocation (24 bytes)
    [Offset1]: FNavLocation (24 bytes)
    [Offset2]: AActor*      (8 bytes)
    ...
}

ItemType ──────→ 类型信息
    ├── ValueSize
    ├── GetValue<T>()
    └── SetValue<T>()
```

**优势**:
- 统一内存管理
- 支持异构类型
- 缓存友好（连续内存）
- 类型安全（编译期检查）

### 4.3 参数绑定系统

**动态参数设计**:

```
FAIDataProvider {
    BindingSource:
        - Constant      → 直接值
        - BlackboardKey → 黑板查询
        - NamedParam    → 运行时注入
}

配置时:
    Test.Distance.Min = BlackboardKey("MinDistance")

运行时:
    float Value = DataProvider.GetValue()
        → Blackboard.GetValue("MinDistance")
        → 500.0f
```

**优势**:
- 配置与运行时解耦
- 支持动态参数
- 类型安全
- 性能优化（缓存绑定）

---

## 五、控制流设计

### 5.1 时间切片执行

**问题**: 如何避免复杂查询阻塞游戏主线程？

**解决方案**: 分步执行 + 时间限制

```
每帧 Manager.Tick():
    TimeRemaining = MaxAllowedTestingTime (3ms)

    for Query in RunningQueries:
        StartTime = Now()

        Query.ExecuteOneStep(TimeRemaining)

        TimeUsed = Now() - StartTime
        TimeRemaining -= TimeUsed

        if TimeRemaining <= 0:
            break  // 下一帧继续
```

**状态保存**:
- `CurrentTest`: 当前测试进度
- `CurrentTestStartingItem`: 测试项目进度
- `OptionIndex`: 当前选项进度

### 5.2 异步执行支持

**设计**:

```
Generator/Test:
    bCanRunAsync = true

执行流程:
    1. 提交异步任务 (寻路、物理查询等)
    2. 标记 bIsAsyncRunning = true
    3. 返回控制权
    4. 下一帧检查 IsFinished()
    5. 处理结果，继续下一步
```

**应用场景**:
- 批量寻路测试
- 大量射线追踪
- 导航网格查询

### 5.3 测试执行优化

**自动排序策略**:

```
排序规则:
    1. TestPurpose: Filter → FilterAndScore → Score
    2. Cost: Low → Medium → High

目标:
    - 尽早过滤无效候选
    - 减少昂贵测试的执行次数
```

**早停机制**:

```
bPassOnSingleResult = true:
    找到第一个满足条件的项目 → 立即返回

应用场景:
    - 是否存在有效掩体？
    - 是否有可见敌人？
```

---

## 六、扩展性设计

### 6.1 插件式架构

**扩展点**:

```
继承点                    扩展类型              难度
─────────────────────────────────────────────────
UEnvQueryGenerator    → 自定义生成逻辑        中
UEnvQueryTest         → 自定义评分/过滤       易
UEnvQueryContext      → 自定义参考对象        易
UEnvQueryItemType     → 自定义数据类型        难
```

**最小化扩展**:

```cpp
// 自定义 Test 只需实现一个方法
class UMyCustomTest : public UEnvQueryTest {
    virtual void RunTest(FEnvQueryInstance& QueryInstance) const override {
        for (ItemIterator It(this, QueryInstance); It; ++It) {
            float Score = CalculateScore(It);
            It.SetScore(TestPurpose, FilterType, Score, Min, Max);
        }
    }
};
```

### 6.2 蓝图扩展支持

**设计**:

```
UEnvQueryContext_BlueprintBase:
    ProvideSingleActor() [BlueprintImplementable]
    ProvideSingleLocation() [BlueprintImplementable]
    ProvideActorsSet() [BlueprintImplementable]

用户蓝图:
    继承 Context_BlueprintBase
    实现 ProvideXXX 函数
    无需 C++ 代码
```

**优势**:
- 快速原型开发
- 设计师友好
- 运行时调试

### 6.3 类型系统扩展

**ItemType 设计**:

```
抽象层次:
    UEnvQueryItemType (抽象)
        ├── ValueSize: 类型大小
        ├── GetValue<T>(): 类型安全访问
        └── SetValue<T>(): 类型安全写入

扩展新类型:
    1. 定义 FValueType
    2. 实现 Get/Set 方法
    3. 注册类型信息
    4. 实现黑板转换（可选）
```

---

## 七、性能优化设计

### 7.1 内存优化

**策略**:

| 优化点 | 技术 | 收益 |
|--------|------|------|
| **连续存储** | TArray<uint8> RawData | 缓存友好，减少内存碎片 |
| **类型擦除** | 模板特化 + void* | 统一管理异构数据 |
| **位域压缩** | int32:31 + bool:1 | 节省 50% 标志位内存 |
| **对象池** | Instance Cache | 避免频繁分配/释放 |
| **延迟分配** | ItemDetails 按需创建 | 非调试模式零开销 |

### 7.2 计算优化

**策略**:

| 优化点 | 技术 | 收益 |
|--------|------|------|
| **时间切片** | 每帧限制 3ms | 避免帧率下降 |
| **测试排序** | Filter 优先 | 减少候选项数量 |
| **早停** | 单结果模式 | 最优情况 O(1) |
| **Context 缓存** | 实例级缓存 | 避免重复计算 |
| **异步执行** | 批量寻路 | 并行化昂贵操作 |
| **实例复用** | Template 缓存 | 避免重复配置 |

### 7.3 算法复杂度分析

```
假设:
    N = 候选项数量
    T = 测试数量
    C = 上下文数量

理想情况:
    Generator: O(N)
    Tests: O(N * T * C)
    总复杂度: O(N * T * C)

优化后:
    测试排序 → 平均减少 50% 候选项
    Context 缓存 → C 项计算变为 O(1)
    实际复杂度: O(N/2 * T + C)
```

---

## 八、调试与监控设计

### 8.1 调试数据收集

**设计**:

```
#if USE_EQS_DEBUGGER
    FEnvQueryInstance {
        DebugItems[]          // 完整快照
        DebugItemDetails[]    // 每个测试的评分
        PerformedTestNames[]  // 执行历史
        OptionStats[]         // 性能统计
    }
#endif
```

**零调试成本**: 宏控制，发布版完全移除

### 8.2 可视化支持

**组件**:

```
EQSTestingPawn:
    - 实时查询可视化
    - 候选项分数显示
    - 测试步骤回放
    - 性能分析图表

EQSDebugger:
    - 历史查询记录
    - 统计信息收集
    - 热点分析
```

### 8.3 性能分析

**监控指标**:

```
Per Query:
    - TotalExecutionTime: 总耗时
    - StepData[]: 每步耗时
    - NumGeneratedItems: 候选项数量
    - NumValidItems: 有效项数量

Per Test:
    - NumRuns: 执行次数
    - TotalTime: 累计耗时
    - AvgTime: 平均耗时
```

---

## 九、设计原则总结

### 9.1 SOLID 原则应用

| 原则 | 应用 |
|------|------|
| **单一职责** | Manager(调度)、Instance(执行)、Node(逻辑) 分离 |
| **开闭原则** | 通过继承扩展，无需修改核心代码 |
| **里氏替换** | 所有 Generator/Test 可互换 |
| **接口隔离** | Generator、Test、Context 独立接口 |
| **依赖倒置** | 依赖抽象 Node，而非具体实现 |

### 9.2 性能设计原则

| 原则 | 实践 |
|------|------|
| **数据局部性** | 连续内存存储 (RawData[]) |
| **预计算** | 实例缓存、测试排序 |
| **懒加载** | ItemDetails 按需创建 |
| **批处理** | 异步批量寻路 |
| **时间分摊** | 时间切片执行 |

### 9.3 可维护性原则

| 原则 | 实践 |
|------|------|
| **清晰职责** | 每个类职责单一明确 |
| **一致性** | 统一的节点执行模型 |
| **可测试性** | 节点无状态，易于单元测试 |
| **可调试性** | 完善的调试数据收集 |
| **文档化** | 自描述的接口设计 |

---

## 十、架构权衡与反思

### 10.1 设计权衡

| 方面 | 选择 | 代价 |
|------|------|------|
| **类型系统** | 类型擦除 | 增加复杂度，牺牲类型安全 |
| **执行模式** | 时间切片 | 增加状态管理复杂度 |
| **扩展性** | 策略模式 | 运行时多态开销 |
| **缓存策略** | 实例复用 | 内存常驻开销 |
| **调试支持** | 宏隔离 | 调试版与发布版差异 |

### 10.2 适用场景

**最佳实践**:
- 空间推理决策（移动、掩体、位置选择）
- 目标评估（敌人优先级、资源选择）
- 区域查询（巡逻点、侦查位置）

**不适用场景**:
- 实时高频查询（每帧执行）
- 简单距离判断（直接计算更快）
- 确定性逻辑（不需要评分）

### 10.3 扩展方向

**可能的改进**:
- 多线程并行执行
- GPU 加速评分计算
- 增量更新查询结果
- 机器学习辅助评分
- 分布式查询系统

---

## 十一、核心设计启示

### 11.1 分层抽象

```
抽象程度: Application → Manager → Instance → Node → Data
职责分离: 使用 → 调度 → 执行 → 逻辑 → 存储
```

**启示**: 清晰的层次划分是复杂系统的基础

### 11.2 组合优于继承

```
UEnvQueryOption = Generator + Tests[]
灵活性 > 单一继承树
```

**启示**: 通过组合实现灵活配置，避免继承爆炸

### 11.3 数据驱动配置

```
逻辑代码 (C++) ← 分离 → 配置数据 (UAsset)
```

**启示**: 将"是什么"与"如何做"解耦

### 11.4 性能优化哲学

```
1. 先使其工作
2. 再使其正确
3. 最后使其快速
   ├── 减少计算量 (测试排序)
   ├── 分摊开销 (时间切片)
   └── 复用结果 (缓存)
```

**启示**: 性能优化应基于 profile，而非猜测

### 11.5 扩展性设计

```
核心系统: 稳定、抽象
扩展点: 明确、简单
用户代码: 最小化实现
```

**启示**: 好的扩展性来自清晰的抽象边界

---

## 十二、总结

EQS 的程序设计展示了一个**工业级空间查询系统**应该具备的特质：

### 核心设计特征

1. **清晰的架构分层** - 职责明确，边界清晰
2. **灵活的策略组合** - 运行时可配置
3. **高效的执行引擎** - 时间切片 + 异步 + 缓存
4. **强大的扩展能力** - 插件式架构
5. **完善的调试支持** - 零成本抽象

### 设计模式运用

- ✅ 策略模式 - Generator/Test/Context
- ✅ 组合模式 - Option 组合
- ✅ 状态模式 - Instance 执行流
- ✅ 迭代器模式 - 统一遍历
- ✅ 享元模式 - 实例缓存
- ✅ 模板方法 - 执行骨架

### 性能优化策略

- ⚡ 时间切片 - 避免卡顿
- ⚡ 测试排序 - 减少计算
- ⚡ 实例缓存 - 避免重建
- ⚡ 类型擦除 - 内存高效
- ⚡ 异步执行 - 并行计算

### 核心价值

EQS 不仅是一个功能系统，更是一个**设计范式**的实践：

> 通过**清晰的抽象**、**灵活的组合**和**精细的优化**，
> 构建一个**易用、高效、可扩展**的复杂系统。

这种设计思想可以应用到其他领域的系统架构设计中。
