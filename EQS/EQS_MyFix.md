# UE EQS 架构设计文档

## 核心观点
- EQS 是数据驱动的空间决策评分框架
- 通过分离关注点和运行时推理
- 让策划独立配置 AI 的"找位置"行为
- 实现 70% 代码量削减和 10x 迭代速度提升

## Why：为什么需要 EQS

### 核心问题：传统硬编码的三大痛点
- 代码重复：每个 AI 行为都要写一遍"找位置"逻辑
- 迭代缓慢：改参数需要重新编译（策划依赖程序员）
- 难以扩展：新需求要改代码（硬编码的代价）

### 设计哲学：运行时推理 vs 预计算
- EQS 的选择：牺牲部分性能，换取灵活性和维护性
- 动态游戏世界的特点
  - 环境动态变化（掩体被破坏、玩家位置转移）
  - 情景动态变化（玩家从敌对→中立，AI 状态切换）
  - 传统预计算无法应对这种多变性
- 对比维度
  - 灵活性：预计算❌ 环境变化要重算 | EQS✅ 自动适应动态环境
  - 性能：预计算✅ 查表即可 | EQS⚠️ 每次重新计算
  - 维护性：预计算❌ 硬编码难改 | EQS✅ 配置化易调
  - 适用场景：预计算→静态世界（策略游戏）| EQS→动态世界（动作游戏）

### 替代方案对比
- 硬编码位置
  - 优势：性能极高、完全可控
  - 劣势：环境变化失效、策划依赖程序员
  - 为何不选：不适应动态世界
- 预计算导航网格
  - 优势：查表快、美术可参与
  - 劣势：动态变化要重烘焙、无法表达相对性、内存占用大
  - 为何不选：缺乏灵活性
- Utility AI
  - 优势：多维评分、可配置
  - 劣势：候选生成与评分耦合、难以复用
  - 为何不选：缺乏标准化流程
- 机器学习
  - 优势：可学习复杂策略
  - 劣势：训练成本高、运行慢、不可解释
  - 为何不选：策划无法调参

### 结论
- EQS 在动态游戏场景下
- 是"配置化 + 可扩展 + 可维护"的最优解

## How：如何实现 EQS

### 核心理念
- AI 不应该"知道答案在哪"
- 而应该"知道如何从多维评估标准中，对候选项打分，再自行选择最优项"
- 借鉴人类决策模型
  - 第一步：圈定范围（Context）"我附近2公里内的餐厅"
  - 第二步：生成候选（Generator）找到5家候选餐厅
  - 第三步：多维度评分（Test）
    - 距离：800m → 7分
    - 口味：川菜 → 9分（我喜欢辣）
    - 价格：人均80 → 6分
    - 环境：嘈杂 → 4分
  - 第四步：加权决策（Finalize）
    - 今天赶时间，距离权重0.6，口味0.3，其他0.1
    - 选最近的那家

### 三大支柱：分离关注点（SoC 原则）
- 执行流程
  - Generator（生成阶段）
  - ↓ Items（候选项集合）
  - ↓ Test + Context（评分阶段，Context在这里）
  - ↓ Normalization（归一化）
  - ↓ Weighted Sum（加权求和）
  - ↓ Output（输出最优项）

#### Generator：定义"从哪里找候选"
- 职责边界：引擎层语义 vs 业务层语义
  - 引擎层语义（✅ Generator 负责）
    - 职责：空间属性、基础类型、通用标签
    - 示例："可导航区域""带 Cover 标签的 Actor"
  - 业务层语义（❌ 交给 Test）
    - 职责：场景化决策逻辑
    - 示例："适合当前 AI 体型""能遮挡玩家视野"
- 设计原则
  - ✅ 使用 NavMesh Grid（过滤墙体、悬崖等无效空间）
  - ✅ 使用 Tagged Actors（采样带特定标签的对象）
  - ❌ 不在 Generator 中硬编码业务逻辑
- 效果
  - Generator 过滤掉 80% 无效选择
  - Test 仅处理几十到几百候选（算力可控）
  - 改需求只需调整标签和 Test，无需改 Generator（符合配置化）

#### Test：定义"如何评价候选"
- 粒度选择：细粒度原子化 + 配置层封装
- 为什么必须选细粒度？
  - 细粒度（EQS 选择）
    - 优势：灵活、可复用、配置化
    - 劣势：配置复杂度较高
  - 粗粒度（不推荐）
    - 优势：简单、语义清晰
    - 劣势：不灵活、每个场景要新 Test
- 示例：评估"狙击位置"
  - 细粒度方案（EQS 推荐）
    - Test1: Distance to Player（距离玩家）
    - Test2: Height Difference（高度差）
    - Test3: Line of Sight（视线）
    - Test4: Cover from Behind（后方掩护）
    - → 4个原子Test组合
  - 粗粒度方案（不推荐）
    - Test: Sniper Position Score（狙击位置评分）
    - → 一个复杂Test搞定
- Test 依赖关系设计
  - 核心：有限依赖 + 共享上下文缓存
  - 问题场景
    - Test1: Pathfinding Cost（寻路代价）
    - Test2: Safe Path（路径是否暴露）→ 需要 Test1 的路径数据
  - 解决方案
    - 显式声明依赖关系
    - 固定执行顺序（先基础 Test，后依赖 Test）
    - 共享上下文缓存（避免重复计算）
  - 设计原则
    - 支持 Test 间数据共享（通过 Query Instance 上下文）
    - 按依赖顺序执行（拓扑排序）
    - 避免循环依赖（编辑器检查）

#### Context：定义"相对于什么评价"
- 核心能力：多 Context 集 + 上下文聚合规则
- 问题：Context 只能是单个对象吗？
  - 场景：AI 要找位置，同时考虑多个威胁
    - 玩家在A点
    - 队友在B点
    - 敌人1在C点
    - 敌人2在D点
    - Test: Distance → Context: ???
- 解决方案：两种模式
  - 模式1：多 Context → 多 Test（推荐）
    - Test1: Distance to Player（权重0.3）
    - Test2: Distance to Teammate（权重0.2）
    - Test3: Distance to Enemy1（权重0.3）
    - Test4: Distance to Enemy2（权重0.2）
    - 优势
      - 符合"分离关注点"设计哲学
      - 每个 Test 职责单一，易维护
      - 新增威胁只需加 1 个 Test
  - 模式2：多 Context → 1 个 Test（轻量场景）
    - Test: Distance
    - Context: [Player, Enemy1, Enemy2]
    - 聚合规则：Minimum（到最近威胁的距离）
    - 优势
      - 配置简单（只需 1 个 Test）
      - 适合次要决策（如选草丛隐蔽）
    - 劣势
      - 聚合逻辑固定（Min/Max/Avg）
      - 无法为不同威胁设置不同权重
- 架构建议
  - 优先选择模式1（多 Context → 多 Test）
  - 仅在轻量场景用模式2（Context 少、规则简单）
  - 避免混合模式（权重逻辑混乱，调试困难）

#### 数据基础：ItemType
- 作用：统一处理不同类型的候选项（位置/对象/方向）
- 内置类型覆盖 99% 场景
  - Point：位置坐标 → 找掩体、找集合点
  - Actor：游戏对象 → 选攻击目标
  - Direction：方向朝向 → 选逃跑方向
- 扩展能力
  - 当前：候选项 = 位置坐标
  - 扩展：候选项 = 任意类型（武器、技能、资源点）
  - 意义：从"空间查询"到"通用决策框架"
- 最佳实践
  - ✅ 优先使用内置类型（Point/Actor/Direction）
  - ✅ 需要额外数据时，用 Actor 的成员变量承载
  - ❌ 避免为存储信息而自定义 ItemType
- 详见附录 A1：ItemType 实现原理（面向程序员）

### 评分机制

#### 归一化（Normalization）
- 目的：统一不同 Test 的量纲到 [0, 1] 区间
- 归一化方式
  - Linear：线性映射 → 均衡评分
  - Square：平方惩罚低分 → 强化差异
  - InverseLinear：反向映射（越大越差）→ 距离、威胁等
- 示例过程
  - Test1原始分：距离22米
  - Test2原始分：高度1.2米
  - Test3原始分：bool值（能否看到）
  - ↓ 归一化 ↓
  - Test1标准分：0.78（Linear 映射）
  - Test2标准分：0.60（Linear 映射）
  - Test3标准分：1.0（看不到=好）

#### 加权求和（Weighted Sum）
- 计算公式
  - 候选点A最终得分 = Test1标准分 × Weight1 + Test2标准分 × Weight2 + Test3标准分 × Weight3
  - 示例：0.78 × 0.5 + 0.60 × 0.2 + 1.0 × 0.3 = 0.81
- 策划可调
  - ✅ 每个 Test 的权重
  - ✅ 归一化方式和参考值域
  - ❌ 合并算法本身（固定为加权求和）

### 性能优化手段

#### 1. 按需触发（不是每帧运行）
- 行为树结构
  - Selector
    - [条件] 已有掩体 且 掩体仍有效 → MoveTo 已缓存的掩体
    - [否则] 重新运行EQS查询 ← 只在这时才跑
- 触发频率
  - 新目标出现（玩家换位置）
  - 环境变化（掩体被破坏）
  - 当前目标失效
  - → 通常几秒才触发一次

#### 2. Generator 阶段剔除
- 初始候选：1000个网格点
- ↓ Generator内置过滤 ↓
- 剔除不可导航的点（碰撞检测）
- 剔除超出范围的点（距离阈值）
- ↓ 剩余200个候选 ↓
- 进入Test阶段

#### 3. Test 阶段提前淘汰
- Test: Trace
  - FilterType: Minimum
  - FloatValueMin: 0.5
  - 含义：分数<0.5直接淘汰，不参与后续Test
- 优化效果
  - Test1: 200个候选 → 淘汰150个 → 50个进入下一轮
  - Test2: 50个候选 → 淘汰30个 → 20个进入下一轮
  - 实际计算量：200 + 50 + 20 ≈ 300次（而非 200×5=1000次）

#### 4. 异步计算
- EQS查询是异步的
- RunEQSQuery(QueryTemplate, Owner, FinishDelegate)
- AI在等待期间继续执行其他行为（不会卡住）

#### 5. 空间分区（场景级优化）
- 大地图场景
  - 地图划分为 100×100米 Grid
  - 每个 Grid 预生成候选点
  - 运行时只在当前 Grid 内跑 Test
  - 效果：候选数从 10000 降到 100

#### 性能数据总结
- 触发频率：2-5秒/次（不是每帧）
- 单次耗时：0.5-2ms（200候选×3Test）
- 并发能力：50个AI × 100候选 × 3Test = 可接受
- 优化效果：算力消耗可控，不会成为瓶颈

## What：EQS 具体是什么

### 系统定位
- "环境查询系统"（Environment Query System）= 空间决策评分框架

#### 为什么叫"环境查询"？
- 命名本质："查询"是起点（先查，后评，后选）
- 历史演进：UE 早期 EQS 雏形是"环境状态查询"，后扩展为评分
- 范围界定："环境"指 AI 决策所需的所有上下文，而非仅指场景
- 设计原则：引擎命名应贴合底层能力，而非具体业务场景
- 结论："空间评分"只是 EQS 的一个应用场景，系统本身覆盖更广

### 架构分层

#### 层级1：查询定义层
- 职责：定义"查什么"和"怎么评"的规则
- 动词：配置 / 声明 / 绑定
- 特点：数据驱动（策划可独立操作）

#### 层级2：执行调度层
- 职责：管理查询的整个生命周期
- 动词：调度 / 分配 / 缓存 / 监控
- 解决：并发 + 性能 + 资源管理

#### 层级3：核心计算层
- 职责：执行具体的查询计算
- 动词：生成 / 筛选 / 打分 / 排序
- 标准化查询流程

#### 层级4：扩展组件层
- 职责：提供可插拔的策略组件
- 动词：扩展 / 定制 / 组合
- 拓展修改路径

### 核心价值量化

#### 对程序员
- 代码量 ↓ 70%（无需为每个行为写查询逻辑）
- 维护成本 ↓ 80%（参数调整不需要重新编译）

#### 对策划
- 独立调参（不依赖程序员）
- 迭代速度 ↑ 10x（5分钟配好，而非2天）

#### 对项目
- 适应需求变化（配置化实现）
- 扩展成本接近 0（组合现有 Test）

## 附录

### A1. ItemType 实现原理（面向程序员）

#### 核心问题：异构数据的统一处理
- 场景
  - 查询最佳位置点 → 返回 FVector
  - 查询最佳目标敌人 → 返回 AActor*
  - 查询最佳逃跑方向 → 返回 FRotator
- 矛盾
  - Generator/Test/Context 需要统一接口处理 Items[]
  - 但 Items 的实际类型各不相同
  - C++ 是静态类型语言，如何统一存储？

#### 解决方案：类型擦除（Type Erasure）
- 类型擦除层（统一存储）
  - struct FEnvQueryItem { uint8* RawData; float Score; }
  - TArray Items 统一存储
- 类型恢复层（ItemType 负责）
  - class UEnvQueryItemType_Point
  - FVector GetItemLocation(const FEnvQueryItem& Item)

#### ItemType 的三大职责
- 类型定义：定义 RawData 的实际类型（FVector/AActor*）
- 内存管理：分配/释放 RawData 内存
- 类型转换：RawData ↔ 具体类型的安全转换

#### 内存布局：分离存储
- Items[] - 元数据数组（固定结构）
  - Item[0]: Score + RawPtr
  - Item[1]: Score + RawPtr
  - ...
- RawData[] - 实际数据块（类型相关）
  - FVector（12字节）
  - FVector（12字节）
  - ...
- 优势
  - 类型无关性：Items[] 结构固定
  - 内存连续性：RawData[] 连续分配，缓存友好
  - 延迟绑定：运行时才确定 ItemType

#### ItemType 在执行流中的作用
- 初始化：Manager→CreateInstance() → ItemType→GetValueSize() 预分配内存
- 生成候选：Generator→GenerateItems() → ItemType→SetValue() 写入 RawData[]
- 评估打分：Test→RunTest() → ItemType→GetItemLocation(Item) 读取数据（热点路径：300+次调用）
- 结果返回：Result.GetItemAsLocation(0) → ItemType→GetItemLocation(BestItem)

#### 内置 ItemType 设计
- Point：FVector（12字节）→ 空间位置查询（90% 场景）
- Actor：TWeakObjectPtr（8字节）→ 目标选择（8% 场景）
- Direction：FRotator（12字节）→ 方向决策（2% 场景）
- 为什么只有 3 种？最小必要集原则
  - Point 覆盖所有"去哪里"
  - Actor 覆盖所有"找谁"
  - Direction 覆盖所有"朝哪"

#### Actor 类型的特殊设计
- 使用 TWeakObjectPtr 而非裸指针
- 解决问题
  - EQS 查询是异步的，可能跨多帧
  - 查询开始时 Actor 存在，结束时可能已销毁
  - WeakPtr 自动处理生命周期，避免悬空指针崩溃

#### 何时需要自定义 ItemType？
- 决策树
  - 查询空间位置？ → ✅ 使用 Point
  - 查询游戏对象？ → ✅ 使用 Actor
  - 查询方向朝向？ → ✅ 使用 Direction
  - 以上都不满足？ → 能否拆分为 Point+Actor 组合？
    - 能 → ✅ 用多次查询代替
    - 不能 → ⚠️ 可能需要自定义（罕见）
- 自定义成本
  - ❌ 需要实现 ItemType + Generator + 可能的自定义 Test
  - ❌ 内存占用增加
  - ❌ 失去内置 Test 的兼容性
- 推荐做法：优先使用 Actor 承载额外数据
  - 方案1：自定义 ItemType（不推荐）
    - struct FBuildingSlot { FVector Location; ETerrainType TerrainType; float ResourceDensity; }
  - 方案2：使用 Actor 承载数据（推荐）
    - class ABuildingSlotMarker : public AActor { UPROPERTY() ETerrainType TerrainType; }
    - 使用 UEnvQueryItemType_Actor + Cast

#### ItemType 的架构定位
- ItemType 不是"功能组件"，而是"基础设施"
- 对比
  - Generator：生产候选项（功能组件 = 工厂）
  - Test：评估候选项（功能组件 = 质检员）
  - Context：提供参考系（功能组件 = 坐标系）
  - ItemType：定义候选项"是什么"（基础设施 = 类型系统）
- 核心洞察
  - ItemType 不是 EQS 的"功能组件"，而是"基础设施"
  - 就像 C++ 的类型系统支撑着整个语言，ItemType 支撑着整个 EQS

#### 从"空间查询"到"通用决策框架"
- 应用场景扩展
  - 空间决策：Point → 找掩体、找集合点
  - 目标选择：Actor → 选攻击目标
  - 方向决策：Direction → 选逃跑方向
  - 武器选择：自定义 Weapon → 选最佳武器
  - 技能选择：自定义 Action → 选释放技能
- 意义
  - 原本：EQS = "空间查询系统"（局限于位置）
  - 现在：EQS = "通用候选评估框架"（适用于任何多选一决策）
  - 未来：可扩展到战略层 AI、资源管理、战术规划等