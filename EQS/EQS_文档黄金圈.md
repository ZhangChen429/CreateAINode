

# 

## Why：为什么需要EQS？（信念层）

### 核心信念：AI不应该"知道答案"，而应该"知道如何评估"

**哲学思考**：
- ❌ **传统AI**：程序员告诉AI"掩体在坐标(100,200)"
- ✅ **EQS理念**：程序员告诉AI"好的掩体应该满足：距离适中+遮挡视线+后方安全"
- **本质区别**：从"给鱼"到"授渔"，从硬编码答案到定义评估标准

**底层动机**：适应动态世界的必然选择

游戏世界的三大动态特性：
1. **环境动态**：掩体被摧毁、地形变化、障碍物生成
2. **情境动态**：玩家从敌对变为友好、威胁等级变化
3. **能力动态**：不同AI有不同移动能力、感知范围

**传统方法的根本缺陷**：
```
硬编码位置：
  环境变化 → 失效 → 需要重新标记 → 维护成本爆炸

预计算方案：
  情境变化 → 失效 → 需要重新烘焙 → 迭代速度缓慢

每AI独立实现：
  能力不同 → 代码重复 → 维护噩梦
```

**EQS的信念**：
> "世界在变，规则不变。让AI在运行时根据规则重新评估，而非依赖过时的答案。"

### 痛点驱动：三大角色的真实困境

**程序员的痛点**：
- 问题：每个AI行为都要写一遍"找位置"的代码
- 现实：代码重复率>70%，改一个需求要改10处代码
- 后果：维护成本高、Bug多、重构困难

**策划的痛点**：
- 问题：想调整"掩体距离从30米改到50米"需要找程序员
- 现实：一个小参数调整要等1天（程序员档期+编译+测试）
- 后果：迭代周期长、创意受限、依赖程序员

**项目的痛点**：
- 问题：临近发布突然改需求（"老板说AI太笨了，改！"）
- 现实：硬编码方案需要大量返工，风险高
- 后果：延期/砍功能/质量下降

**EQS的承诺**：
> "让程序员写一次框架，策划配置无限种AI行为；让改需求从'噩梦'变成'改配置'。"

---

## How：EQS如何实现理念？（方法层）

### 方法论1：借鉴人类决策模型

**洞察**：人类如何选择餐厅？

```
第一步：圈定范围（Context）
  "我附近2公里内的餐厅"

第二步：生成候选（Generator）
  → 找到5家候选餐厅

第三步：多维度评分（Test）
  - 距离：800m → 7分
  - 口味：川菜 → 9分（我喜欢辣）
  - 价格：人均80 → 6分
  - 环境：嘈杂 → 4分

第四步：加权决策（Finalize）
  今天赶时间，距离权重0.6，口味0.3，其他0.1
  → 选最近的那家
```

**EQS的映射**：
- Context = "我附近2公里"
- Generator = "找到5家餐厅"  
- Test = "距离/口味/价格/环境"
- WeightedSum = "根据当前情境调整权重"

**为什么这样设计有效**：
- ✅ 符合直觉：程序员和策划都能快速理解
- ✅ 灵活适应：情境变化只需调整权重（赶时间vs悠闲聚餐）
- ✅ 可扩展：新增"停车方便"维度只需加一个Test

### 方法论2：工业流水线的标准化思维

**洞察**：为什么汽车工厂效率高？

```
[原材料仓库] → [工位1:外观检查] → [工位2:性能测试] → [工位3:安全评级] → [合格品出库]
```

**关键特征**：
- 每个工位职责单一（分离关注点）
- 所有产品走相同流程（标准化）
- 工位可独立升级（模块化）

**EQS的映射**：
```
[Context:划定搜索范围] → [Generator:生成候选] → [Test1:距离评分] → [Test2:视线评分] → [Test3:威胁评分] → [输出最优选项]
```

**流水线的三大收益**：
1. **质量稳定**：每个环节有标准，不会因人而异
2. **易于调试**：问题必然在某个工位，快速定位
3. **可扩展**：新增工位不影响现有流程

### 方法论3：分离关注点的架构原则

**核心理念**：让每个模块只做一件事

**Generator的职责边界**：
```
✅ 应该做（引擎层语义）：
  - NavMeshGrid：只在可导航区域生成候选
  - TaggedActors：只返回带"Cover"标签的对象
  
❌ 不应该做（业务层语义）：
  - 判断"这个掩体是否适合当前AI体型"（交给Test）
  - 判断"这个位置是否能遮挡玩家视野"（交给Test）
```

**为什么这样拆分**：
- **灵活性**：改业务需求只改Test，不动Generator
- **复用性**：NavMeshGrid可用于找掩体/找狙击位/找逃跑点
- **可测试性**：Generator输出200个候选，Test评分逻辑独立测试

**Test的职责边界**：
```
✅ 应该做（原子化评分）：
  - Distance：计算距离
  - LineOfSight：计算视线遮挡
  - Height：计算高度差
  
❌ 不应该做（复合决策）：
  - "EvaluateSniperPosition"（这应该是多个Test的组合）
```

**为什么选择细粒度Test**：
- **可复用**：Distance可用于任何需要距离评分的场景
- **易理解**：策划能理解"距离+视线+高度"的组合逻辑
- **可调试**：每个Test独立调试，快速定位问题

**Context的职责边界**：
```
✅ 应该做（提供参考系）：
  - 单Context：Distance(Player) = 距离玩家多远
  - 多Context聚合：Distance(AllEnemies, Min) = 距离最近敌人多远
  
❌ 不应该做（耦合业务逻辑）：
  - Context不应该包含"如何评分"的逻辑（交给Test）
```

### 方法论4：运行时推理的技术路径

**选择**：牺牲性能换取灵活性

**性能优化的四大手段**：

**1. 分帧执行**
```
问题：1000个候选×10个Test = 10000次计算，单帧卡顿200ms
方案：分10帧执行，每帧计算1000次，单帧仅20ms
代价：决策延迟100ms（对动作游戏可接受）
```

**2. 多级剔除**
```
Generator阶段：
  - SimpleGrid生成10000个候选
  - NavMeshGrid过滤后剩1000个（过滤90%）
  - TaggedActors再过滤剩200个（再过滤80%）

Test阶段：
  - 仅需评估200个候选，计算量降低98%
```

**3. 异步计算**
```
主线程：
  - 提交查询请求
  - 继续执行其他逻辑

后台线程：
  - 执行Generator+Test计算
  
下一帧：
  - 主线程获取结果
```

**4. 按需触发**
```
❌ 传统AI：每帧重新计算位置（60fps = 60次/秒）
✅ EQS：仅在决策点触发（玩家改变位置、环境变化、威胁出现）
  - 计算频率降低10倍
```

**实测数据**：
- 优化前：单次查询200ms，无法用于实时游戏
- 优化后：单次查询20ms，支持100+个AI并发查询

---

## What：EQS具体是什么？（实践层）

### 组件清单：四层架构

**第一层：查询定义层（策划操作层）**

**UEnvQuery资产**：
- 功能：定义一次完整查询的配置文件
- 包含：Generator列表、Test列表、Context绑定、权重配置
- 格式：可视化编辑器（蓝图节点）

**Generator配置**：
```
类型：SimpleGrid | NavMeshGrid | PathingGrid | ActorsOfClass | TaggedActors
参数：
  - SearchRadius: 搜索半径
  - SpaceBetween: 候选点间距
  - FilterTag: 过滤标签
```

**Test配置**：
```
类型：Distance | Height | LineOfSight | Dot | PathLength...
参数：
  - TestPurpose: 评分目的
  - Normalization: 归一化方式（Linear/Square/Inverse）
  - Weight: 权重
```

**Context绑定**：
```
类型：Querier（查询者）| Player | AllEnemies | Teammates
聚合规则：Min | Max | Average | Sum
```

**第二层：执行调度层（引擎核心层）**

**UEnvQueryManager（单例）**：
- 职责：管理所有查询的生命周期
- 功能：
  - 接收查询请求
  - 分配执行资源（线程/帧预算）
  - 缓存查询结果（避免重复计算）
  - 监控性能指标

**查询实例池**：
```cpp
TArray<FEnvQueryInstance*> ActiveQueries;  // 正在执行的查询
TArray<FEnvQueryInstance*> PendingQueries; // 等待执行的查询
TMap<int32, FCachedResult> ResultCache;     // 结果缓存
```

**第三层：核心计算层（标准化流程）**

**FEnvQueryInstance（查询实例）**：
```
生命周期：
  1. Initialize(QueryAsset, Querier, Context)
  2. ExecuteGenerators() → 生成Items
  3. ExecuteTests() → 逐Test评分
  4. NormalizeScores() → 归一化
  5. CalculateFinalScores() → 加权合并
  6. FindBestItem() → 输出结果
```

**数据流**：
```
Input: QueryAsset + Querier + Context
  ↓
Generator → Items[0..N]
  ↓
Test1 → Items[0..N].Scores[0]
Test2 → Items[0..N].Scores[1]
...
TestM → Items[0..N].Scores[M-1]
  ↓
Normalize → Items[0..N].NormalizedScores[0..M-1]
  ↓
WeightedSum → Items[0..N].FinalScore
  ↓
Output: BestItem
```

**第四层：扩展组件层（插件机制）**

**自定义Generator**：
```cpp
class UEnvQueryGenerator_Custom : public UEnvQueryGenerator {
  virtual void GenerateItems(FEnvQueryInstance& QueryInstance) override {
    // 自定义生成逻辑
  }
};
```

**自定义Test**：
```cpp
class UEnvQueryTest_Custom : public UEnvQueryTest {
  virtual void RunTest(FEnvQueryInstance& QueryInstance) override {
    // 自定义评分逻辑
  }
};
```

**自定义Context**：
```cpp
class UEnvQueryContext_Custom : public UEnvQueryContext {
  virtual void ProvideContext(FEnvQueryInstance& QueryInstance) override {
    // 自定义参考系
  }
};
```

### 典型使用场景：配置模式库

**场景1：AI找掩体躲避玩家**

**需求分析**：
- 目标：找到能遮挡玩家视线的位置
- 约束：距离玩家20-50米、在可导航区域、后方相对安全

**EQS配置**：
```
Generator: NavMeshGrid
  - SearchRadius: 50m
  - SpaceBetween: 2m
  
Test1: Distance (Context: Player)
  - Min: 20m, Max: 50m
  - Normalization: Linear
  - Weight: 0.3
  
Test2: LineOfSight (Context: Player)
  - TestPurpose: Inverse (遮挡视线得高分)
  - Normalization: Binary
  - Weight: 0.5
  
Test3: Dot (Context: Player)
  - TestPurpose: 后方得高分
  - Normalization: Linear
  - Weight: 0.2
```

**运行流程**：
1. NavMeshGrid在50米内生成200个候选点
2. Distance评分：30米处得满分，20米/50米得0分
3. LineOfSight评分：完全遮挡得1分，完全暴露得0分
4. Dot评分：在玩家后方180°得1分，正前方得0分
5. 加权合并：FinalScore = 0.3×Distance + 0.5×LineOfSight + 0.2×Dot
6. 输出：FinalScore最高的位置

**场景2：狙击手找制高点**

**需求分析**：
- 目标：找到高处、视野开阔、后方有掩护的位置
- 约束：高度差>10米、能看到玩家、后方有墙体

**EQS配置**：
```
Generator: NavMeshGrid + HeightFilter
  - SearchRadius: 100m
  - MinHeight: 10m (相对地面)
  
Test1: HeightDifference (Context: Player)
  - Normalization: Square (高度优势递增)
  - Weight: 0.4
  
Test2: LineOfSight (Context: Player)
  - TestPurpose: Normal (能看到得高分)
  - Weight: 0.4
  
Test3: Trace (Context: Querier, Direction: Backward)
  - TestPurpose: 检测后方是否有掩体
  - Weight: 0.2
```

**场景3：小队协同占领据点**

**需求分析**：
- 目标：找到靠近目标点、分散分布、互相支援的位置
- 约束：距离目标点<30米、队友间距>10米、能互相看到

**EQS配置**：
```
Generator: ActorsOfClass(CapturePoint)
  → OnActorLocation (在据点周围生成候选)
  
Test1: Distance (Context: CapturePoint)
  - Max: 30m
  - Weight: 0.4
  
Test2: Distance (Context: AllTeammates)
  - Min: 10m (惩罚过近位置)
  - Normalization: Inverse
  - Weight: 0.3
  
Test3: LineOfSight (Context: AllTeammates)
  - Aggregation: Average (平均能看到多少队友)
  - Weight: 0.3
```

### 扩展点：可定制的策略

**归一化方式扩展**：
```
Linear: Score = (Value - Min) / (Max - Min)
Square: Score = ((Value - Min) / (Max - Min))²
Inverse: Score = 1 - (Value - Min) / (Max - Min)
Custom: 自定义归一化函数
```

**聚合规则扩展**：
```
多Context聚合：
  - Min: 距离最近的威胁
  - Max: 距离最远的队友
  - Average: 到所有敌人的平均距离
  - Weighted: 根据威胁等级加权
```

**依赖管理扩展**：
```
显式依赖声明：
Test_CoverQuality:
  Depends: [Test_LineOfSight, Test_Distance]
  
执行顺序：
  1. Test_LineOfSight（基础数据）
  2. Test_Distance（基础数据）
  3. Test_CoverQuality（使用1和2的结果）
```

---

## 启示：从Why到What的完整闭环

### 理念落地的关键

**1. Why驱动What**：
- 信念："让AI评估而非记答案"
- 落地：运行时推理架构 + 配置化评分规则

**2. How保证质量**：
- 方法：分离关注点 + 标准化流程
- 保证：灵活性与性能的平衡

**3. What支撑迭代**：
- 实践：四层架构 + 可扩展组件
- 结果：代码量-70%，迭代速度+10倍

### 学习者的思维转变

**初级阶段（会用）**：
- 看到：Generator/Test/Context配置界面
- 能做：照着教程配置"找掩体"查询
- 局限：不理解为什么这样设计

**中级阶段（理解）**：
- 看到：分离关注点、运行时推理的设计哲学
- 能做：为新场景设计合理的Generator/Test组合
- 局限：不知道如何权衡性能与灵活性

**高级阶段（设计）**：
- 看到：Why→How→What的完整思维链
- 能做：为新引擎设计类似的可扩展架构
- 突破：从"使用手册"到"成为造手册的人"

### 工程哲学的启示

**好的架构是"做减法"**：
- EQS没有提供1000个Test，只有20+个原子Test
- 复杂度在组合中产生，而非单个模块中

**配置化的本质是"权力转移"**：
- 从"程序员独占决策权"到"策划拥有调参权"
- 释放创造力，加速迭代

**命名的背后是覆盖范围的野心**：
- "环境查询"而非"空间评分"
- 暗示系统可扩展到更广的应用领域

**性能优化的正确顺序**：
- 先保证架构正确（分离关注点）
- 再用工程手段优化（分帧/异步/缓存）
- 不要为了性能牺牲灵活性

---

## 总结：黄金圈的螺旋上升

**Why（信念）** → 适应动态世界，让AI学会评估而非记忆答案

**How（方法）** → 借鉴人类决策+工业流水线+分离关注点+运行时推理

**What（实践)** → 四层架构+原子化Test+多Context聚合+可扩展组件

**最终启示**：
> 真正理解一个系统，不是记住它的API，而是理解它为什么这样设计、如何权衡的、具体实现了什么。从Why到What，从信念到实践，这才是从使用者到设计者的思维跃迁。