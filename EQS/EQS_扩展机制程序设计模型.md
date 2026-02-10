# UE EQS 扩展机制的程序设计模型

## 一、扩展架构概览

### 1.1 扩展点的本质定义

EQS 系统提供了**四个正交的扩展维度**，每个维度解决不同层次的抽象问题：

```
扩展维度                 抽象问题                    设计目标
──────────────────────────────────────────────────────────
Generator           "候选空间的定义"              开放集合生成
Test                "评价函数的定义"              多维度价值评估
Context             "参考系的定义"                动态上下文绑定
ItemType            "数据模型的定义"              类型系统扩展
```

### 1.2 扩展层次模型

```
┌─────────────────────────────────────────────────────────┐
│  用户代码层 (Application Layer)                          │
│  - 继承扩展点基类                                        │
│  - 实现特定领域逻辑                                      │
└────────────────┬────────────────────────────────────────┘
                 │ 依赖抽象接口
                 ↓
┌─────────────────────────────────────────────────────────┐
│  抽象接口层 (Abstract Interface Layer)                   │
│  - 定义扩展点契约                                        │
│  - 规范执行协议                                          │
└────────────────┬────────────────────────────────────────┘
                 │ 被调用
                 ↓
┌─────────────────────────────────────────────────────────┐
│  执行框架层 (Execution Framework Layer)                  │
│  - 调度扩展点                                            │
│  - 管理执行上下文                                        │
│  - 保证执行语义                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 二、Generator 扩展模型

### 2.1 抽象定义

**本质**: Generator 是一个**集合生成器 (Set Producer)**

```
Generator: Context → PowerSet(Candidates)

输入: 上下文空间 (Context Space)
输出: 候选项集合 (Candidate Set)
约束: 生成的集合必须同质 (Homogeneous Type)
```

### 2.2 设计模式分析

#### 策略模式 (Strategy Pattern)

```
抽象策略 (Abstract Strategy)
    ├── 接口: GenerateItems(QueryInstance)
    └── 契约: 生成候选集合，写入 QueryInstance

具体策略 (Concrete Strategies)
    ├── 空间采样策略 (Spatial Sampling)
    ├── 对象查询策略 (Object Query)
    ├── 关系推导策略 (Relational Inference)
    └── 用户定义策略 (User-Defined)
```

**设计意图**:
- **可替换性**: 任何 Generator 都可以互换
- **组合性**: 单个查询可以有多个 Option，每个 Option 一个 Generator
- **封装性**: 生成逻辑与查询框架解耦

#### 工厂模式 (Factory Pattern)

```
Generator 本质上是一个抽象工厂:

GenerateItems() {
    // 1. 读取上下文 (原材料)
    Context context = PrepareContext();

    // 2. 执行生成算法 (制造过程)
    Set<Candidate> candidates = ProduceSet(context);

    // 3. 输出产品 (成品)
    AddToInstance(candidates);
}
```

### 2.3 设计原则应用

#### 单一职责原则 (SRP)

```
Generator 只负责:
    ✅ 生成候选集合
    ❌ 不负责评估
    ❌ 不负责排序
    ❌ 不负责选择
```

#### 开闭原则 (OCP)

```
对扩展开放:
    - 任何人可以创建新的 Generator 子类
    - 添加新的生成算法

对修改封闭:
    - 不需要修改 EQS 框架代码
    - 不需要修改其他 Generator
```

#### 依赖倒置原则 (DIP)

```
高层模块 (QueryInstance):
    依赖 → Generator 抽象接口

低层模块 (具体 Generator):
    实现 → Generator 抽象接口

结果: 高层模块不依赖低层实现细节
```

### 2.4 扩展点设计特征

#### 最小化接口 (Minimal Interface)

```cpp
// 核心接口只有一个方法
virtual void GenerateItems(FEnvQueryInstance& QueryInstance) const = 0;

设计意图:
    - 降低实现负担
    - 明确单一职责
    - 易于理解和维护
```

#### 上下文注入 (Context Injection)

```cpp
// 通过 QueryInstance 注入执行上下文
QueryInstance {
    World,              // 游戏世界
    Owner,              // 查询发起者
    ContextCache,       // 上下文缓存
    NamedParams,        // 运行时参数
}

设计意图:
    - 避免全局状态
    - 支持并发查询
    - 便于测试和调试
```

#### 参数化配置 (Parameterization)

```cpp
// 配置与实现分离
class Generator {
    UPROPERTY(EditDefaultsOnly)
    FAIDataProviderFloat Parameter;  // 配置时声明

    void GenerateItems() {
        float value = Parameter.GetValue();  // 运行时获取
    }
}

设计意图:
    - 运行时动态绑定 (黑板值/常量)
    - 提升复用性
    - 支持数据驱动
```

---

## 三、Test 扩展模型

### 3.1 抽象定义

**本质**: Test 是一个**评价函数 (Evaluation Function)**

```
Test: Candidate × Context → Score ∪ {Filtered}

输入: (候选项, 上下文)
输出: 分数 或 过滤标记
性质: 可复合 (Composable)
```

### 3.2 函数式设计思想

#### 高阶函数 (Higher-Order Function)

```
Test 本质是一个可配置的高阶函数:

RunTest = Compose(
    ValueExtractor,      // 提取测试值
    ScoringEquation,     // 评分方程
    Normalization        // 归一化
)

配置决定行为:
    - TestPurpose: Filter | Score | FilterAndScore
    - FilterType: Match | Range
    - ScoringEquation: Linear | Square | InverseLinear | ...
```

#### 函数组合 (Function Composition)

```
一个查询可以有多个 Test:

FinalScore = Test1 ∘ Test2 ∘ Test3 ∘ ... ∘ TestN

执行语义:
    - 串行组合: Test 按顺序执行
    - 累积效应: 分数累加，过滤累积
    - 短路优化: 过滤后的项目跳过后续 Test
```

### 3.3 设计模式分析

#### 迭代器模式 (Iterator Pattern)

```
Test 通过迭代器访问候选集合:

for (ItemIterator It(this, QueryInstance); It; ++It) {
    // 自动跳过已过滤项目
    // 提供统一访问接口
    It.SetScore(...);
}

设计意图:
    - 隐藏存储结构 (类型擦除的 RawData)
    - 统一遍历接口
    - 支持过滤逻辑
```

#### 模板方法模式 (Template Method Pattern)

```
Test 基类定义执行骨架:

Execute() {
    // 1. 前置处理
    BindParameters();
    PrepareContext();

    // 2. 核心逻辑 (子类实现)
    RunTest();  ← 扩展点

    // 3. 后置处理
    NormalizeScores();
    UpdateStatistics();
}
```

#### 策略模式的参数化 (Parameterized Strategy)

```
Test 不仅是策略，还是可配置的策略:

Strategy = TestAlgorithm + Configuration

Configuration:
    - TestPurpose: 决定执行模式
    - ScoringEquation: 决定评分函数
    - Normalization: 决定归一化方式

同一个 Test 子类 + 不同配置 = 不同行为
```

### 3.4 设计原则应用

#### 单一职责原则 (SRP)

```
Test 只负责:
    ✅ 计算单一维度的价值
    ❌ 不负责生成候选
    ❌ 不负责最终决策
    ❌ 不负责执行调度
```

#### 开闭原则 (OCP)

```
扩展方式:
    1. 继承 Test 基类 → 新的评价维度
    2. 配置评分方程 → 新的评分曲线
    3. 组合多个 Test → 新的综合评价

无需修改:
    - 框架代码
    - 其他 Test
    - Generator/Context
```

#### 里氏替换原则 (LSP)

```
任何 Test 子类都可以替换 Test 基类:

Test* test = new MyCustomTest();
test->RunTest(queryInstance);  // 行为符合预期

前提条件:
    - 遵守 Test 执行协议
    - 正确使用 ItemIterator
    - 正确设置分数/过滤
```

---

## 四、Context 扩展模型

### 4.1 抽象定义

**本质**: Context 是一个**坐标系提供者 (Reference Frame Provider)**

```
Context: QueryInstance → ReferenceFrame

输入: 查询实例
输出: 参考坐标系 (位置集合 或 对象集合)
语义: 定义"相对于什么"的参考点
```

### 4.2 设计模式分析

#### 依赖注入 (Dependency Injection)

```
Context 是一种运行时依赖注入:

Generator/Test 声明依赖:
    TSubclassOf<UEnvQueryContext> RequiredContext;

运行时注入:
    Context* ctx = Instantiate(RequiredContext);
    Data data = ctx->ProvideContext(queryInstance);

设计意图:
    - 解耦逻辑与数据源
    - 支持多态上下文
    - 便于测试 (Mock Context)
```

#### 享元模式 (Flyweight Pattern)

```
Context 数据被缓存复用:

QueryInstance {
    ContextCache: Map<ContextClass, ContextData>
}

PrepareContext(ContextClass):
    if (Cached)
        return Cache[ContextClass]
    else
        data = Context->ProvideContext()
        Cache[ContextClass] = data
        return data

设计意图:
    - 避免重复计算
    - 同一查询中多次使用相同上下文
    - 性能优化
```

#### 策略模式 (Strategy Pattern)

```
Context 是可替换的数据源策略:

接口: ProvideContext(QueryInstance, ContextData)

实现:
    - Querier: 提供查询发起者
    - Target: 提供目标对象
    - Custom: 用户自定义逻辑

同一个 Generator/Test 可以配置不同 Context
```

### 4.3 设计原则应用

#### 单一职责原则 (SRP)

```
Context 只负责:
    ✅ 提供参考数据
    ❌ 不负责生成候选
    ❌ 不负责评估逻辑
    ❌ 不负责数据处理
```

#### 接口隔离原则 (ISP)

```
Context 接口极简:
    virtual void ProvideContext(QueryInstance, ContextData) const = 0;

职责单一:
    - 输入: QueryInstance (包含查询上下文)
    - 输出: ContextData (填充数据)
    - 无副作用
```

#### 依赖倒置原则 (DIP)

```
Generator/Test 依赖 Context 抽象:
    TSubclassOf<UEnvQueryContext> contextClass;

而非具体实现:
    UEnvQueryContext_Querier* querier;  ❌

结果:
    - Generator/Test 可以使用任何 Context
    - Context 实现可以独立变化
```

---

## 五、ItemType 扩展模型

### 5.1 抽象定义

**本质**: ItemType 是一个**类型系统 (Type System)**

```
ItemType: 定义候选项的数据模型

职责:
    1. 类型标识 (Type Identity)
    2. 内存布局 (Memory Layout)
    3. 类型转换 (Type Conversion)
    4. 序列化 (Serialization)
```

### 5.2 设计模式分析

#### 类型擦除 (Type Erasure)

```
EQS 使用类型擦除统一存储异构数据:

存储层 (Storage):
    TArray<uint8> RawData;  // 无类型字节数组

类型层 (Type):
    template<typename T>
    T GetValue(const uint8* RawData) {
        return *reinterpret_cast<const T*>(RawData);
    }

优势:
    - 统一内存管理
    - 支持任意类型
    - 避免模板膨胀
```

#### 适配器模式 (Adapter Pattern)

```
ItemType 是类型系统的适配器:

C++ 类型 (Native Type)
    ↓ 适配
ItemType (EQS Type System)
    ↓ 适配
Blackboard (AI Type System)

每个 ItemType 实现三个适配接口:
    1. GetValue/SetValue: C++ 类型 ↔ RawData
    2. SetContextHelper: 创建 ContextData
    3. StoreInBlackboard: RawData → Blackboard
```

#### 抽象工厂模式 (Abstract Factory)

```
ItemType 定义了对象创建协议:

ItemType {
    CreateItem(value) {
        uint8* memory = AllocateMemory(ValueSize);
        SetValue(memory, value);
        return FEnvQueryItem(memory);
    }
}

不同 ItemType 创建不同类型的对象:
    - Point: 创建位置对象
    - Actor: 创建 Actor 引用
    - Custom: 创建自定义数据
```

### 5.3 设计原则应用

#### 开闭原则 (OCP)

```
类型系统可扩展:
    - 继承 ItemType 基类
    - 定义新的 ValueType
    - 实现类型操作接口

无需修改:
    - EQS 核心代码
    - 现有 ItemType
    - Generator/Test 逻辑
```

#### 里氏替换原则 (LSP)

```
任何 ItemType 子类可以替换基类:

ItemType* type = new CustomItemType();
FEnvQueryItem item = type->CreateItem(value);

前提:
    - 正确实现 GetValue/SetValue
    - 正确设置 ValueSize
    - 遵守内存管理约定
```

---

## 六、扩展机制的系统性设计

### 6.1 正交性设计 (Orthogonality)

```
四个扩展维度相互独立:

Generator ⊥ Test ⊥ Context ⊥ ItemType

意味着:
    - 可以独立扩展任意维度
    - 扩展之间无耦合
    - 组合爆炸可控
```

**数学表达**:
```
系统可能状态空间 = |Generators| × |Tests|^N × |Contexts|^M × |ItemTypes|

独立性保证:
    - 添加新 Generator 不影响 Test
    - 添加新 Test 不影响 Context
    - 修改 ItemType 不影响 Generator/Test 逻辑 (只要接口兼容)
```

### 6.2 组合优于继承 (Composition over Inheritance)

```
EQS 查询不是通过继承构建，而是通过组合:

Query = Option[]
Option = Generator + Test[]
Test = Algorithm + Configuration
Generator = Algorithm + Context

继承只用于:
    ✅ 扩展点的实现 (实现多态)
    ✅ 类型层次 (ItemType 继承树)

组合用于:
    ✅ 查询的构建 (运行时组合)
    ✅ 行为的配置 (策略组合)
```

### 6.3 依赖注入的层次结构

```
层次 1: Manager 注入 Instance
    Manager → CreateInstance(Query, Owner, Params)

层次 2: Instance 注入 Generator/Test
    Instance → Generator->GenerateItems(Instance)
    Instance → Test->RunTest(Instance)

层次 3: Generator/Test 注入 Context
    Generator → Instance->PrepareContext(ContextClass)
    Test → Instance->PrepareContext(ContextClass)

特点:
    - 单向依赖 (上层依赖下层抽象)
    - 运行时注入 (灵活配置)
    - 无全局状态 (并发安全)
```

### 6.4 接口隔离的层次设计

```
用户接口 (User Interface)
    ├── 配置接口: 编辑器可视化配置
    └── 执行接口: RunQuery(Query, Callback)

扩展接口 (Extension Interface)
    ├── Generator: GenerateItems(Instance)
    ├── Test: RunTest(Instance)
    └── Context: ProvideContext(Instance, Data)

框架接口 (Framework Interface)
    ├── Instance: ExecuteOneStep(TimeLimit)
    ├── Manager: Tick(DeltaTime)
    └── ItemType: GetValue/SetValue

原则:
    - 每层接口职责明确
    - 上层不依赖下层实现
    - 接口最小化
```

---

## 七、参数化设计模型

### 7.1 配置与逻辑分离

```
传统设计:
    硬编码参数 → 修改需要重新编译

EQS 设计:
    配置 (DataAsset) ← 分离 → 逻辑 (C++ 类)

优势:
    - 运行时可配置
    - 无需重新编译
    - 版本控制友好
    - 支持热重载
```

### 7.2 多级参数绑定

```
参数来源层次:

Level 1: 编辑器常量
    FAIDataProviderFloat { DefaultValue = 500.0f }

Level 2: 运行时参数
    QueryInstance.SetNamedParam("MaxDistance", 1000.0f)

Level 3: 黑板动态值
    FAIDataProviderFloat { BindingName = "PatrolRadius" }
    → Blackboard->GetValue("PatrolRadius")

统一访问接口:
    float value = Parameter.GetValue();
```

### 7.3 惰性求值 (Lazy Evaluation)

```
参数绑定采用惰性求值:

配置时:
    Parameter.BindingName = "SomeKey"  // 只记录绑定关系

运行时:
    Parameter.BindData(Owner, QueryID)  // 建立绑定
    float value = Parameter.GetValue()  // 实际求值

优势:
    - 支持运行时动态值
    - 避免过早求值
    - 支持参数变化
```

---

## 八、执行语义保证

### 8.1 状态机的不变量 (Invariants)

```
EQS 执行过程维护以下不变量:

Invariant 1: 单调进度
    CurrentTest 只能递增或重置 (不能回退)

Invariant 2: 项目不可变性
    Generator 执行后，Items[] 不再添加
    (直到进入下一个 Option)

Invariant 3: 分数累积性
    Test 只能修改分数和过滤标记
    不能修改候选项数据

Invariant 4: 上下文一致性
    同一查询实例中，相同 Context 返回相同数据
```

### 8.2 执行顺序的可交换性

```
Test 执行顺序的语义:

串行语义:
    Test1 → Test2 → Test3
    (后续 Test 看到前面的过滤结果)

但允许自动排序优化:
    FilterTest1 → FilterTest2 → ScoreTest1 → ScoreTest2

可交换条件:
    - Filter Test 可以任意排序 (交换律)
    - Score Test 可以任意排序 (交换律)
    - Filter 必须在 Score 前 (偏序关系)
```

### 8.3 异步执行的语义

```
同步执行语义:
    Generate() → Test1() → Test2() → ... → Done

异步执行语义:
    Generate() → Suspend
        ↓
    Resume → Test1() → Suspend
        ↓
    Resume → Test2() → Done

保证:
    - 最终结果一致性 (与同步执行相同)
    - 状态恢复正确性 (可从任意点恢复)
    - 无竞态条件 (单线程执行模型)
```

---

## 九、扩展点的通用设计原则

### 9.1 最小化接口原则 (Minimal Interface Principle)

```
扩展点接口设计准则:

✅ 只暴露必需的方法
    Generator: 只有 GenerateItems()
    Test: 只有 RunTest()
    Context: 只有 ProvideContext()

✅ 通过注入传递依赖
    不是: virtual void Run() { /* 需要访问全局状态 */ }
    而是: virtual void Run(ExecutionContext& ctx) { /* 通过参数传递 */ }

✅ 单一抽象级别
    接口中的方法处于同一抽象层次
    不混合高层逻辑和底层细节
```

### 9.2 契约式设计 (Design by Contract)

```
每个扩展点定义清晰的契约:

Generator 契约:
    前置条件: QueryInstance 已初始化
    后置条件: Items[] 包含生成的候选项
    不变量: ItemType 与声明一致

Test 契约:
    前置条件: Items[] 非空
    后置条件: 每个 Item 有分数或被过滤
    不变量: 不修改 Items[] 的数据内容

Context 契约:
    前置条件: QueryInstance.Owner 有效
    后置条件: ContextData 包含有效数据
    不变量: 无副作用 (纯函数)
```

### 9.3 开闭原则的多层实现

```
Level 1: 类继承扩展
    继承 Generator/Test/Context 基类
    → 添加新的算法实现

Level 2: 配置参数扩展
    调整 Test 的 ScoringEquation
    → 改变评分行为

Level 3: 组合扩展
    Option = Generator + Test[]
    → 创建新的查询模式

设计意图:
    - 多个扩展层次
    - 不同场景选择不同扩展方式
    - 最大化灵活性
```

### 9.4 依赖倒置的一致应用

```
依赖关系图:

    Manager
       ↓ (依赖抽象)
    Instance
       ↓ (依赖抽象)
  Generator / Test / Context
       ↑ (实现抽象)
    用户扩展类

特征:
    - 所有依赖都指向抽象
    - 高层模块不依赖低层实现
    - 便于替换和扩展
```

---

## 十、元设计模式：框架设计的设计

### 10.1 好莱坞原则 (Hollywood Principle)

```
"Don't call us, we'll call you"

传统方式:
    User → Manager.Run()
        → Generator.Generate()
        → Test.Evaluate()

EQS 方式:
    User → Manager.RunQuery()
    Manager → Instance.Execute()
        ↓
    Instance → Generator.GenerateItems(Instance)
               Test.RunTest(Instance)

控制反转:
    - 用户不直接调用扩展点
    - 框架调用用户代码
    - 保证执行语义
```

### 10.2 模板-钩子模式 (Template-Hook Pattern)

```
框架提供执行模板:

Execute() {
    // 模板流程 (框架控制)
    Initialize()
    for each Option:
        ExecuteGenerator()  ← Hook 1
        for each Test:
            ExecuteTest()    ← Hook 2
        FinalizeOption()
    SelectResult()
}

用户提供钩子实现:
    Generator->GenerateItems()
    Test->RunTest()

分离:
    - 执行流程 (框架负责)
    - 业务逻辑 (用户负责)
```

### 10.3 分层架构的抽象泄漏控制

```
理想情况:
    上层不需要知道下层实现

实际设计:
    有限的抽象泄漏 (Controlled Leakage)

示例:
    Test 可以访问 ItemType:
        GetItemLocation(QueryInstance, ItemIndex)

    但只通过抽象方法:
        不是: *(FVector*)RawData  ❌
        而是: ItemType->GetItemLocation(RawData)  ✅

原则:
    - 泄漏是有限和受控的
    - 通过接口泄漏，而非实现
    - 便于优化性能
```

### 10.4 可测试性设计

```
扩展点天然支持测试:

单元测试:
    Generator/Test/Context 可独立测试
    无需完整的 EQS 环境

Mock 对象:
    QueryInstance 可以 Mock
    Context 可以 Mock

隔离性:
    每个扩展点无副作用
    可以重复执行
    结果可预测
```

---

## 十一、设计权衡分析

### 11.1 性能 vs 灵活性

```
灵活性设计:
    - 运行时多态 (virtual 调用)
    - 动态参数绑定
    - 上下文缓存

性能开销:
    - 虚函数调用开销
    - 参数查找开销
    - 缓存查找开销

权衡:
    ✅ 接受适度的性能开销
    ✅ 通过其他优化补偿 (时间切片、异步)
    ✅ 灵活性带来的价值 > 性能损失
```

### 11.2 类型安全 vs 内存效率

```
类型擦除设计:
    优势:
        - 统一内存管理
        - 避免模板膨胀
        - 支持序列化

    代价:
        - 编译期类型检查减弱
        - 需要运行时类型验证
        - 潜在的类型转换错误

权衡:
    ✅ 通过 ItemType 系统补偿类型安全
    ✅ ValidItemType 检查
    ✅ 明确的类型契约
```

### 11.3 简单性 vs 表达力

```
最小化接口:
    优势:
        - 易于理解
        - 降低实现难度
        - 减少出错可能

    限制:
        - 表达力受限
        - 复杂逻辑难以实现

权衡:
    ✅ 提供辅助方法 (GetItemLocation)
    ✅ 提供配置选项 (ScoringEquation)
    ✅ 90% 场景用简单接口
    ✅ 10% 场景可以绕过抽象
```

---

## 十二、设计模式的协同效应

### 12.1 模式组合产生的系统特性

```
策略模式 + 组合模式 + 迭代器模式
    ↓
运行时可配置的查询管道

模板方法 + 依赖注入
    ↓
框架控制的扩展点执行

类型擦除 + 适配器模式
    ↓
类型安全的异构数据管理

享元模式 + 工厂模式
    ↓
高效的对象复用机制
```

### 12.2 设计原则的协同效应

```
单一职责 + 接口隔离
    ↓
清晰的组件边界

开闭原则 + 依赖倒置
    ↓
无限的扩展可能性

里氏替换 + 契约式设计
    ↓
可靠的多态行为
```

---

## 十三、通用性与可移植性

### 13.1 扩展机制的通用模型

EQS 的扩展机制可以抽象为通用框架：

```
通用查询系统 (Generic Query System)
    = 集合生成器 (Generator)
    + 评价函数集 (Tests)
    + 上下文系统 (Context)
    + 类型系统 (ItemType)
    + 执行框架 (Manager/Instance)

适用领域:
    - 空间推理 (EQS 原始用途)
    - 资源选择 (装备、技能选择)
    - 任务推荐 (任务系统)
    - 社交推荐 (NPC 互动对象)
    - 路径规划 (路径候选评估)
```

### 13.2 设计模式的可迁移性

```
这套设计模式可以应用到其他系统:

行为树系统:
    Task = 可扩展的行为节点 (类似 Generator/Test)
    Decorator = 可组合的修饰器 (类似 Test 组合)
    Blackboard = 上下文系统 (类似 Context)

材质系统:
    MaterialNode = 可扩展的节点 (类似 Generator/Test)
    MaterialInstance = 参数化配置 (类似 DataProvider)
    ShaderCompiler = 执行框架 (类似 Manager)

任何需要"生成-评估-选择"模式的系统
```

---

## 十四、总结：设计哲学

### 14.1 核心设计理念

```
1. 正交分解 (Orthogonal Decomposition)
   将复杂问题分解为独立的维度

2. 抽象分层 (Layered Abstraction)
   清晰的层次边界和依赖方向

3. 组合优于继承 (Composition over Inheritance)
   通过组合构建灵活的行为

4. 开放扩展，封闭修改 (OCP)
   系统可扩展，核心不可变

5. 依赖抽象 (DIP)
   所有依赖都指向抽象接口

6. 契约明确 (Design by Contract)
   每个接口都有清晰的前后置条件

7. 配置与逻辑分离 (Separation of Concerns)
   数据驱动的参数化设计
```

### 14.2 扩展机制的价值

```
对开发者:
    ✅ 学习成本低 (最小化接口)
    ✅ 扩展简单 (单一职责)
    ✅ 组合灵活 (正交设计)
    ✅ 调试容易 (无状态、可测试)

对系统:
    ✅ 可维护性强 (清晰边界)
    ✅ 可扩展性好 (开闭原则)
    ✅ 性能可控 (时间切片、缓存)
    ✅ 复用性高 (数据驱动)
```

### 14.3 设计启示

这套扩展机制展示了如何设计一个**工业级可扩展框架**：

1. **从抽象问题入手** - Generator/Test/Context 分别解决不同抽象问题
2. **最小化核心接口** - 只暴露最必需的方法
3. **通过组合构建复杂性** - 而非通过继承层次
4. **控制抽象泄漏** - 有限的、受控的泄漏以优化性能
5. **保证执行语义** - 框架控制执行流程
6. **支持多层次扩展** - 继承、配置、组合三个层次

这些原则可以应用到任何需要高度可扩展性的系统设计中。

---

## 十五、扩展点设计检查清单

设计新的扩展点时，检查以下原则：

### 基础原则
- [ ] 接口最小化 (1-3 个核心方法)
- [ ] 单一抽象层次
- [ ] 清晰的前后置条件
- [ ] 无副作用 (或副作用受控)
- [ ] 可独立测试

### 扩展性
- [ ] 对扩展开放
- [ ] 对修改封闭
- [ ] 支持运行时组合
- [ ] 参数可配置
- [ ] 上下文可注入

### 性能
- [ ] 避免全局状态
- [ ] 支持缓存
- [ ] 支持异步 (如需要)
- [ ] 可以优化执行顺序
- [ ] 内存布局友好

### 可用性
- [ ] 蓝图支持 (如适用)
- [ ] 编辑器集成
- [ ] 调试可视化
- [ ] 错误信息清晰
- [ ] 文档完善

遵循这些原则，可以设计出优雅、高效、易用的扩展系统。
