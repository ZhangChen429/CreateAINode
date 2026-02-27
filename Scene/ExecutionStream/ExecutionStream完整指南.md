# ExecutionStream 完整指南
> 从用户痛点到技术实现的全景解析

---

## 第一章：ExecutionStream是什么？

### 1.1 一句话定义

```
ExecutionStream = 将非线性分支剧情编译成可高效执行的时间流
```

### 1.2 它在系统中的位置

```
┌─────────────────────────────────────────────────────────────┐
│                        2077 叙事系统                         │
│                                                              │
│   ┌─────────────┐                                           │
│   │ Quest系统   │ ← 任务逻辑、触发条件                       │
│   └──────┬──────┘                                           │
│          │ 触发                                              │
│          ↓                                                   │
│   ┌─────────────────────────────────────────────────┐       │
│   │          InteractiveScene系统                    │       │
│   │                                                  │       │
│   │   ┌─────────────┐      ┌─────────────────┐      │       │
│   │   │ SceneGraph  │ ──→  │ ExecutionStream │      │       │
│   │   │ (设计时)     │ 编译  │ (运行时)        │      │       │
│   │   └─────────────┘      └─────────────────┘      │       │
│   │         ↑                       ↓                │       │
│   │     策划编辑              驱动执行               │       │
│   │                                 ↓                │       │
│   │                    ┌────────────────────┐       │       │
│   │                    │ Workspot / Dialog  │       │       │
│   │                    │ Camera / Audio     │       │       │
│   │                    └────────────────────┘       │       │
│   └─────────────────────────────────────────────────┘       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 核心职责

```
ExecutionStream的三个核心职责：

1. 时间转换
   输入：相对时间（Event在Section内的位置）
   输出：绝对时间（Event在Scene中的精确时刻）

2. 高效查询
   问题：当前帧应该执行什么？
   方案：时间排序 + 二分查找 = O(log n)

3. 动态合并
   问题：玩家选择时间不确定
   方案：预编译分支 + 运行时偏移合并
```

---

## 第二章：它解决什么问题？

### 2.1 用户体验层面的问题

#### 问题1：站桩对话

```
传统RPG对话：

  ┌─────────────────────────────────────┐
  │                                     │
  │     NPC站在原地                      │
  │         ↓                           │
  │     嘴巴动，播放语音                  │
  │         ↓                           │
  │     显示对话框                       │
  │         ↓                           │
  │     玩家选择回复                      │
  │         ↓                           │
  │     重复...                          │
  │                                     │
  └─────────────────────────────────────┘

  问题：
    ❌ NPC像木头人
    ❌ 不真实、不沉浸
    ❌ 2020年代玩家无法接受

2077的目标：

  ┌─────────────────────────────────────┐
  │                                     │
  │     NPC走向椅子                      │
  │         ↓                           │
  │     坐下、拿起酒杯                    │
  │         ↓                           │
  │     一边喝酒一边说话                  │
  │         ↓                           │
  │     根据对话情绪改变姿态              │
  │         ↓                           │
  │     玩家选择后，NPC反应不同           │
  │                                     │
  └─────────────────────────────────────┘

  需求：
    ✅ NPC有真实行为
    ✅ 行为和对话精确同步
    ✅ 毫秒级时序控制
```

**ExecutionStream如何解决**：
```
提供精确的时间轴控制：

  Time=0ms:    NPC开始走向椅子
  Time=2000ms: NPC坐下（触发Workspot）
  Time=3000ms: NPC拿起酒杯
  Time=3500ms: NPC开始说第一句话
  Time=8000ms: 显示玩家选择
  ...

  所有行为按绝对时间精确编排
```

---

#### 问题2：无法打断的过场

```
传统过场动画：

  玩家触发剧情
      ↓
  屏幕渐黑
      ↓
  播放预渲染/实时过场
      ↓
  玩家完全失去控制（不能移动、不能攻击）
      ↓
  即使敌人出现也必须看完
      ↓
  过场结束，恢复控制

  问题：
    ❌ 打断沉浸感
    ❌ 遇到危险无法反应
    ❌ 重玩时无法跳过

2077的目标：

  玩家触发剧情
      ↓
  无缝进入对话（无黑屏）
      ↓
  玩家保持部分控制（可以看周围）
      ↓
  如果敌人出现 → 可以立即中断 → 进入战斗
      ↓
  战斗结束 → 可以恢复对话（可选）
```

**ExecutionStream如何解决**：
```
支持中断和恢复：

  正常执行：
    Time=5000ms → 执行Event_A
    Time=6000ms → 执行Event_B
    Time=7000ms → 执行Event_C

  中断发生（Time=6500ms，战斗开始）：
    保存状态：currentTime=6500ms
    暂停ExecutionStream
    进入战斗

  恢复（战斗结束）：
    恢复状态：currentTime=6500ms
    继续执行：Event_C在Time=7000ms执行

  因为ExecutionStream是时间排序的：
    任意时间点都可以暂停
    任意时间点都可以恢复
    不需要从头重放
```

---

#### 问题3：虚假的选择

```
传统分支对话：

  玩家选择A → 台词变化 → 结果相同
  玩家选择B → 台词变化 → 结果相同
  玩家选择C → 台词变化 → 结果相同

  问题：
    ❌ 选择没有意义
    ❌ 玩家感到被欺骗
    ❌ 重玩价值低

2077的目标：

  玩家选择A → 分支剧情A → NPC好感+10 → 解锁隐藏任务
  玩家选择B → 分支剧情B → NPC好感-5  → 常规结局
  玩家选择C → 分支剧情C → NPC死亡    → 特殊结局

  选择真正改变：
    ✅ 后续对话内容
    ✅ NPC行为和态度
    ✅ 任务走向和结局
```

**ExecutionStream如何解决**：
```
支持真正的分支执行：

  预编译所有分支：
    branchA_Stream: [分支A的所有Event]
    branchB_Stream: [分支B的所有Event]
    branchC_Stream: [分支C的所有Event]

  玩家选择时（假设选择A）：
    mainStream.CombineSubstream(branchA_Stream, currentTime)

  分支A的Event被合并到主流：
    真正执行分支A的剧情
    不是换一句台词
    是完全不同的行为序列
```

---

#### 问题4：失去控制感

```
传统过场中的玩家：

  状态：完全被动
  能做：什么都不能做
  感受：在看电影，不是在玩游戏

  问题：
    ❌ 第一人称游戏尤其严重
    ❌ 玩家和角色割裂
    ❌ "这不是我在行动"

2077的目标（Tier系统）：

  Tier1：完全自由（开放世界）
  Tier2：限制移动，可以观察（重要对话）
  Tier3：限制移动和视角范围（关键剧情）
  Tier4：完全锁定视角（电影时刻）
  Tier5：黑屏过渡

  根据剧情需要动态调整玩家控制权
  而不是一刀切的"完全失去控制"
```

**ExecutionStream如何解决**：
```
Tier切换作为Event执行：

  Time=0ms:    ChangeTier(Tier2)  ← 进入对话，限制移动
  Time=500ms:  Camera(...)
  Time=1000ms: Workspot(NPC坐下)
  ...
  Time=30000ms: ChangeTier(Tier4) ← 关键时刻，锁定视角
  Time=35000ms: 播放关键剧情
  Time=40000ms: ChangeTier(Tier2) ← 恢复部分控制
  ...
  Time=60000ms: ChangeTier(Tier1) ← 对话结束，完全自由

  Tier变化是时间轴上的Event
  精确控制何时给予/收回多少控制权
```

---

### 2.2 技术层面的问题

#### 问题1：相对时间 vs 绝对时间

```
设计时（策划视角）：

  Section_A:
    Event_1 @ 0ms    ← "在这个段落开始时"
    Event_2 @ 2000ms ← "在这个段落的第2秒"

  Section_B:
    Event_3 @ 0ms    ← "在这个段落开始时"
    Event_4 @ 1000ms ← "在这个段落的第1秒"

  策划不关心Section_B在全局的第几秒开始
  只关心段落内部的相对时间
  这样Section可以移动位置而不影响内部

运行时（游戏视角）：

  游戏循环每帧问："现在是第5000ms，该执行什么？"

  需要绝对时间才能回答：
    Event_1 @ 0ms（绝对）
    Event_2 @ 2000ms（绝对）
    Event_3 @ 5000ms（绝对）← 需要知道Section_A持续5秒
    Event_4 @ 6000ms（绝对）

问题：
  设计用相对时间（模块化）
  执行用绝对时间（精确定位）
  如何转换？
```

**ExecutionStream的解决方案**：
```
编译时累加转换：

  输入：
    Section_A (duration=5000ms)
      Event_1 @ 0ms
      Event_2 @ 2000ms
    Section_B (duration=3000ms)
      Event_3 @ 0ms
      Event_4 @ 1000ms

  编译过程：
    currentTime = 0
    Event_1: 0 + 0 = 0ms
    Event_2: 0 + 2000 = 2000ms
    currentTime += 5000  // Section_A结束
    Event_3: 5000 + 0 = 5000ms
    Event_4: 5000 + 1000 = 6000ms

  输出（ExecutionStream）：
    [0ms]:    Event_1
    [2000ms]: Event_2
    [5000ms]: Event_3
    [6000ms]: Event_4

  已转换为绝对时间，可直接查询执行
```

---

#### 问题2：图遍历 vs 时间查询

```
SceneGraph的结构（图）：

  StartNode
      ↓
  Section_A ──→ Section_B ──→ Section_C
      ↓
  内部Event列表

查询"Time=5000ms执行什么"：

  方法：从StartNode遍历
    → 累加Section_A的duration
    → 判断5000ms是否在Section_A内
    → 如果不是，继续遍历Section_B
    → 累加duration，判断...
    → 找到对应Section后，遍历内部Event

  复杂度：O(节点数 × Event数)

  问题：
    每帧都要遍历
    60fps = 每秒60次遍历
    性能不可接受
```

**ExecutionStream的解决方案**：
```
预编译成时间排序数组：

  ActionChannel = [
    {time: 0,    event: E1},
    {time: 2000, event: E2},
    {time: 5000, event: E3},  ← 二分查找直接定位
    {time: 6000, event: E4},
    ...
  ]

查询"Time=5000ms执行什么"：

  方法：二分查找
    → 比较中间元素
    → 缩小范围
    → 7次比较找到位置（100个Event）

  复杂度：O(log n)

  性能对比：
    图遍历：~2ms/帧
    二分查找：~0.02ms/帧
    提升100倍
```

---

#### 问题3：动态选择的时间不确定性

```
问题场景：

  Section_A (5秒)
      ↓
  ChoiceNode（玩家选择）← 玩家思考多久？不知道！
      ↓
  Branch_A / Branch_B / Branch_C

编译时的困境：

  Section_A: 0-5秒（确定）
  ChoiceNode: 5秒-???（不确定）
  Branch_A: ???开始（依赖玩家选择时间）

  无法在编译时确定Branch_A的绝对时间
```

**ExecutionStream的解决方案**：
```
分段编译 + 运行时合并：

Step 1: 编译确定部分
  mainStream: Section_A的Event（0-5秒）

Step 2: 独立编译每个分支（从0开始）
  branchA_Stream: Branch_A的Event（相对时间0开始）
  branchB_Stream: Branch_B的Event（相对时间0开始）
  branchC_Stream: Branch_C的Event（相对时间0开始）

Step 3: 运行时合并
  玩家在Time=12000ms选择了Branch_A

  执行：mainStream.CombineSubstream(branchA_Stream, 12000)

  结果：
    branchA_Stream中的Event_X @ 0ms
    → 合并后变成 @ 12000ms

    branchA_Stream中的Event_Y @ 3000ms
    → 合并后变成 @ 15000ms

  核心公式：
    绝对时间 = 分支相对时间 + 玩家选择时刻
```

---

#### 问题4：分支取消和资源管理

```
问题场景：

  玩家选择了Branch_A
  那么Branch_B和Branch_C的Event不应该执行

  但它们可能已经预加载了资源：
    Branch_B的音频文件
    Branch_C的动画数据

  问题：
    如何取消未选分支？
    如何释放无用资源？
```

**ExecutionStream的解决方案**：
```
ControlChannel处理取消：

三通道设计：
  ActionChannel:      存储要执行的动作
  ControlChannel:     存储取消请求  ← 解决这个问题
  StimulationChannel: 存储预加载提示

玩家选择Branch_A时：
  1. 合并branchA_Stream到mainStream
  2. 向ControlChannel添加取消请求：
     - 取消branchB_Stream的所有ActionId
     - 取消branchC_Stream的所有ActionId
  3. 释放Branch_B和Branch_C的预加载资源

ControlChannel的作用：
  ✅ 取消未选分支的Event
  ✅ 中断时取消正在执行的Event
  ✅ 快进时跳过中间的Event
```

---

## 第三章：ExecutionStream的架构

### 3.1 三通道设计

```
┌─────────────────────────────────────────────────────────────┐
│                      ExecutionStream                         │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                   ActionChannel                        │  │
│  │                                                        │  │
│  │  职责：存储所有需要执行的动作                           │  │
│  │                                                        │  │
│  │  数据结构：                                            │  │
│  │    DynArray<ActionRecord> m_stream; // 按时间排序      │  │
│  │                                                        │  │
│  │  ActionRecord = {                                      │  │
│  │    SceneTime startTime;    // 开始时间                 │  │
│  │    SceneTime duration;     // 持续时间                 │  │
│  │    ActionDef actionDef;    // 要执行的动作定义         │  │
│  │    ActionId actionId;      // 唯一标识（用于取消）     │  │
│  │  }                                                     │  │
│  │                                                        │  │
│  │  核心方法：                                            │  │
│  │    IterateStartingRecords(timeWindow, callback)        │  │
│  │    → 查询时间窗口内需要启动的动作                      │  │
│  │                                                        │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                  ControlChannel                        │  │
│  │                                                        │  │
│  │  职责：处理取消和中断                                  │  │
│  │                                                        │  │
│  │  数据结构：                                            │  │
│  │    DynArray<Request> m_requests; // 取消请求队列       │  │
│  │                                                        │  │
│  │  Request = {                                           │  │
│  │    Type type;              // cancelAction/cancelAll   │  │
│  │    Target target;          // 要取消的目标             │  │
│  │    Msec occurrenceTime;    // 请求发生时间             │  │
│  │  }                                                     │  │
│  │                                                        │  │
│  │  使用场景：                                            │  │
│  │    - 分支选择时取消未选分支                            │  │
│  │    - 中断时取消当前动作                                │  │
│  │    - 快进时批量取消                                    │  │
│  │                                                        │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                StimulationChannel                      │  │
│  │                                                        │  │
│  │  职责：预测和预加载                                    │  │
│  │                                                        │  │
│  │  数据结构：                                            │  │
│  │    DynArray<Stimulation> m_stimulations;               │  │
│  │                                                        │  │
│  │  Stimulation = {                                       │  │
│  │    Target target;          // 通知哪个系统             │  │
│  │    Hint hint;              // 预加载什么资源           │  │
│  │    Msec actualTime;        // 实际使用时间             │  │
│  │    Msec stimulationTime;   // 提前通知时间             │  │
│  │  }                                                     │  │
│  │                                                        │  │
│  │  示例：                                                │  │
│  │    "在第8秒会播放audio_001.wem"                        │  │
│  │    "提前1秒（第7秒）通知AudioSystem预加载"             │  │
│  │                                                        │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 核心方法

```cpp
class ExecutionStream {
public:
    // 获取通道
    ActionChannel& GetActionChannel();
    ControlChannel& GetControlChannel();
    StimulationChannel& GetStimulationChannel();

    // 核心方法1：合并分支流
    void CombineSubstream(
        const ExecutionStream& branchStream,  // 要合并的分支
        Uint32 offsetMsec                      // 时间偏移量
    );

    // 核心方法2：重新索引（合并后排序）
    void Reindex();

    // 核心方法3：清空
    void Clear();
};
```

### 3.3 CombineSubstream详解

```cpp
// 伪代码实现
void ExecutionStream::CombineSubstream(
    const ExecutionStream& branch,
    Uint32 offset
) {
    // 1. 合并ActionChannel
    for (const ActionRecord& record : branch.m_actionChannel) {
        ActionRecord newRecord = record;
        newRecord.startTime += offset;  // 关键：加上偏移量
        this->m_actionChannel.PushBack(newRecord);
    }

    // 2. 合并ControlChannel
    for (const Request& request : branch.m_controlChannel) {
        Request newRequest = request;
        newRequest.occurrenceTime += offset;
        this->m_controlChannel.PushBack(newRequest);
    }

    // 3. 合并StimulationChannel
    for (const Stimulation& stim : branch.m_stimulationChannel) {
        Stimulation newStim = stim;
        newStim.stimulationTime += offset;
        newStim.actualTime += offset;
        this->m_stimulationChannel.PushBack(newStim);
    }

    // 4. 重新按时间排序
    this->Reindex();
}
```

---

## 第四章：工作流程

### 4.1 编译流程

```
┌─────────────────────────────────────────────────────────────┐
│                        编译阶段                              │
│                                                              │
│   输入：SceneGraph（策划设计的分支图）                       │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Step 1: 解析图结构                                   │   │
│   │                                                      │   │
│   │   识别：                                             │   │
│   │     - 确定段（StartNode到第一个ChoiceNode）          │   │
│   │     - 选择点（ChoiceNode, ConditionNode）            │   │
│   │     - 分支段（每个选择后的路径）                     │   │
│   │     - 汇聚点（HubNode, AndNode）                     │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Step 2: 编译确定段                                   │   │
│   │                                                      │   │
│   │   遍历确定段的所有节点                               │   │
│   │   累加duration，计算绝对时间                         │   │
│   │   生成mainStream                                     │   │
│   │                                                      │   │
│   │   mainStream = {                                     │   │
│   │     [0ms]:    Event_1,                               │   │
│   │     [2000ms]: Event_2,                               │   │
│   │     [5000ms]: ChoiceNode标记                         │   │
│   │   }                                                  │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Step 3: 编译每个分支                                 │   │
│   │                                                      │   │
│   │   对每个分支独立编译（时间从0开始）                  │   │
│   │                                                      │   │
│   │   branchA_Stream = {                                 │   │
│   │     [0ms]:    Event_A1,  // 相对时间                 │   │
│   │     [3000ms]: Event_A2,                              │   │
│   │   }                                                  │   │
│   │                                                      │   │
│   │   branchB_Stream = { ... }                           │   │
│   │   branchC_Stream = { ... }                           │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Step 4: 递归处理嵌套分支                             │   │
│   │                                                      │   │
│   │   如果分支内还有选择点                               │   │
│   │   递归执行Step 2-3                                   │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│   输出：                                                     │
│     - mainStream（主流）                                     │
│     - branchStreams[]（所有预编译的分支流）                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 执行流程

```
┌─────────────────────────────────────────────────────────────┐
│                        执行阶段                              │
│                                                              │
│   每帧执行（60fps = 每16.6ms执行一次）：                     │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Step 1: 更新时间                                     │   │
│   │                                                      │   │
│   │   prevTime = currentTime                             │   │
│   │   currentTime += deltaTime                           │   │
│   │   timeWindow = [prevTime, currentTime]               │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Step 2: 查询ActionChannel                            │   │
│   │                                                      │   │
│   │   actionChannel.IterateStartingRecords(              │   │
│   │     timeWindow,                                      │   │
│   │     [](ActionRecord& record) {                       │   │
│   │       ExecuteAction(record);  // 执行动作            │   │
│   │     }                                                │   │
│   │   );                                                 │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Step 3: 处理ControlChannel                           │   │
│   │                                                      │   │
│   │   controlChannel.IterateRequests(                    │   │
│   │     timeWindow,                                      │   │
│   │     [](Request& request) {                           │   │
│   │       if (request.type == CancelAction) {            │   │
│   │         CancelAction(request.target);                │   │
│   │       }                                              │   │
│   │     }                                                │   │
│   │   );                                                 │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Step 4: 处理StimulationChannel                       │   │
│   │                                                      │   │
│   │   stimulationChannel.IterateStimulations(            │   │
│   │     timeWindow,                                      │   │
│   │     [](Stimulation& stim) {                          │   │
│   │       NotifySystem(stim.target, stim.hint);          │   │
│   │     }                                                │   │
│   │   );                                                 │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Step 5: 检查选择点                                   │   │
│   │                                                      │   │
│   │   if (到达ChoiceNode) {                              │   │
│   │     显示选择UI                                       │   │
│   │     等待玩家选择                                     │   │
│   │     // 玩家选择后触发OnChoiceSelected               │   │
│   │   }                                                  │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Step 6: 检查结束                                     │   │
│   │                                                      │   │
│   │   if (到达EndNode || 所有ActionRecord执行完毕) {     │   │
│   │     OnSceneEnded();                                  │   │
│   │   }                                                  │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 选择处理流程

```
┌─────────────────────────────────────────────────────────────┐
│                    玩家选择处理                              │
│                                                              │
│   触发：玩家在ChoiceNode做出选择                            │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Step 1: 获取选择的分支流                             │   │
│   │                                                      │   │
│   │   choiceName = GetPlayerChoice();  // 如"Branch_A"   │   │
│   │   branchStream = GetPrecompiledStream(choiceName);   │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Step 2: 合并到主流                                   │   │
│   │                                                      │   │
│   │   offset = currentTime;  // 玩家选择的时刻           │   │
│   │   mainStream.CombineSubstream(branchStream, offset); │   │
│   │                                                      │   │
│   │   // branchStream中的Event时间都加上offset           │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Step 3: 取消其他分支                                 │   │
│   │                                                      │   │
│   │   for (otherBranch in unselectedBranches) {          │   │
│   │     // 添加取消请求到ControlChannel                  │   │
│   │     controlChannel.AddCancelRequest(otherBranch);    │   │
│   │                                                      │   │
│   │     // 释放预加载的资源                              │   │
│   │     ReleasePreloadedResources(otherBranch);          │   │
│   │   }                                                  │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                         ↓                                    │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Step 4: 继续执行                                     │   │
│   │                                                      │   │
│   │   // 下一帧会自动从合并后的流中执行                  │   │
│   │   // 因为branchStream的Event已经有了绝对时间         │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 第五章：实际案例

### 5.1 案例：与Hanako的餐厅对话

```
场景：q115_00b_hanako
描述：V与Hanako在餐厅讨论合作

SceneGraph设计：

  [StartNode]
      ↓
  [Section_Greeting]
    - ChangeTier(Tier2) @ 0ms
    - Camera(餐厅全景) @ 500ms
    - Workspot(Hanako坐下) @ 1000ms
    - Workspot(V坐下) @ 1000ms
    duration = 3000ms
      ↓
  [Section_Dialog1]
    - DialogLine(Hanako: "你好，V") @ 0ms
    - LookAt(Hanako看V) @ 0ms
    duration = 5000ms
      ↓
  [ChoiceNode]
    ├─→ [Branch_Polite] "礼貌回应"
    │     - DialogLine(V: "很高兴见到你") @ 0ms
    │     - Hanako好感度+10
    │     duration = 4000ms
    │
    ├─→ [Branch_Cold] "冷淡回应"
    │     - DialogLine(V: "说重点") @ 0ms
    │     - Hanako好感度-5
    │     duration = 3000ms
    │
    └─→ [Branch_Joke] "开玩笑"
          - DialogLine(V: "没想到大小姐会约我") @ 0ms
          - Workspot(Hanako微笑) @ 2000ms
          - Hanako好感度+5
          duration = 5000ms
      ↓
  [HubNode] (汇聚)
      ↓
  [Section_Farewell]
    - DialogLine(Hanako: "期待我们的合作") @ 0ms
    - StopWorkspot(Hanako) @ 3000ms
    - StopWorkspot(V) @ 3000ms
    - ChangeTier(Tier1) @ 4000ms
    duration = 5000ms
      ↓
  [EndNode]
```

### 5.2 编译结果

```
mainStream（确定段）：

  [0ms]:     ChangeTier(Tier2)
  [500ms]:   Camera(餐厅全景)
  [1000ms]:  Workspot(Hanako坐下)
  [1000ms]:  Workspot(V坐下)
  [3000ms]:  DialogLine(Hanako: "你好，V")
  [3000ms]:  LookAt(Hanako看V)
  [8000ms]:  ChoiceNode标记 ← 等待玩家选择


branchPolite_Stream（从0开始）：

  [0ms]:     DialogLine(V: "很高兴见到你")
  [0ms]:     SetFact(Hanako好感度+10)
  [4000ms]:  后续Event...


branchCold_Stream（从0开始）：

  [0ms]:     DialogLine(V: "说重点")
  [0ms]:     SetFact(Hanako好感度-5)
  [3000ms]:  后续Event...


branchJoke_Stream（从0开始）：

  [0ms]:     DialogLine(V: "没想到大小姐会约我")
  [2000ms]:  Workspot(Hanako微笑)
  [0ms]:     SetFact(Hanako好感度+5)
  [5000ms]:  后续Event...


afterChoice_Stream（汇聚后，从0开始）：

  [0ms]:     DialogLine(Hanako: "期待我们的合作")
  [3000ms]:  StopWorkspot(Hanako)
  [3000ms]:  StopWorkspot(V)
  [4000ms]:  ChangeTier(Tier1)
```

### 5.3 运行时执行

```
Time = 0ms:
  → 执行 ChangeTier(Tier2)
  → V进入对话模式，限制移动

Time = 500ms:
  → 执行 Camera(餐厅全景)
  → 镜头切换到餐厅全景

Time = 1000ms:
  → 执行 Workspot(Hanako坐下)
  → 执行 Workspot(V坐下)
  → 两人开始坐下动画

Time = 3000ms:
  → 执行 DialogLine(Hanako: "你好，V")
  → 执行 LookAt(Hanako看V)
  → Hanako开始说话，看向V

Time = 8000ms:
  → 到达 ChoiceNode
  → 显示选择UI：[礼貌回应] [冷淡回应] [开玩笑]
  → 等待玩家选择...

... 玩家思考5秒 ...

Time = 13000ms:
  → 玩家选择 [开玩笑]
  → 执行合并：mainStream.CombineSubstream(branchJoke_Stream, 13000)

合并后的mainStream：
  [0ms-8000ms]:   原有Event（已执行）
  [13000ms]:      DialogLine(V: "没想到大小姐会约我")  ← 0+13000
  [13000ms]:      SetFact(Hanako好感度+5)
  [15000ms]:      Workspot(Hanako微笑)                 ← 2000+13000
  [18000ms]:      后续汇聚Event...                     ← 5000+13000

Time = 13000ms:
  → 执行 DialogLine(V: "没想到大小姐会约我")
  → 执行 SetFact(Hanako好感度+5)

Time = 15000ms:
  → 执行 Workspot(Hanako微笑)
  → Hanako播放微笑动画

Time = 18000ms:
  → 合并 afterChoice_Stream
  → mainStream.CombineSubstream(afterChoice_Stream, 18000)

Time = 18000ms:
  → 执行 DialogLine(Hanako: "期待我们的合作")

Time = 21000ms:
  → 执行 StopWorkspot(Hanako)
  → 执行 StopWorkspot(V)
  → 两人开始站起动画

Time = 22000ms:
  → 执行 ChangeTier(Tier1)
  → V恢复完全控制

Time = 23000ms:
  → 到达 EndNode
  → Scene结束
```

---

## 第六章：总结

### 6.1 ExecutionStream解决的问题一览

```
┌─────────────────────────────────────────────────────────────┐
│                   用户体验问题                               │
├──────────────────────┬──────────────────────────────────────┤
│ 站桩对话             │ → 精确时序控制，NPC真实行为          │
├──────────────────────┼──────────────────────────────────────┤
│ 无法打断的过场       │ → 支持中断和恢复                     │
├──────────────────────┼──────────────────────────────────────┤
│ 虚假的选择           │ → 真正的分支执行                     │
├──────────────────────┼──────────────────────────────────────┤
│ 失去控制感           │ → Tier系统动态调整                   │
└──────────────────────┴──────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     技术问题                                 │
├──────────────────────┬──────────────────────────────────────┤
│ 相对时间             │ → 编译时转换为绝对时间               │
├──────────────────────┼──────────────────────────────────────┤
│ 图遍历性能           │ → 时间排序 + 二分查找                │
├──────────────────────┼──────────────────────────────────────┤
│ 选择时间不确定       │ → 预编译分支 + 运行时偏移合并        │
├──────────────────────┼──────────────────────────────────────┤
│ 分支取消             │ → ControlChannel处理                 │
├──────────────────────┼──────────────────────────────────────┤
│ 资源预加载           │ → StimulationChannel通知             │
└──────────────────────┴──────────────────────────────────────┘
```

### 6.2 核心价值

```
ExecutionStream的核心价值：

  将"设计师的分支思维"转化为"计算机的时间执行"

  设计师说："如果玩家选A，执行这些Event"
  ExecutionStream说："在第13000ms执行Event_X，在第15000ms执行Event_Y"

  桥接两个世界：
    人类世界：逻辑、选择、分支
    机器世界：时间、顺序、执行
```

### 6.3 一句话总结

```
ExecutionStream =
  非线性剧情的时间编译器 +
  动态选择的运行时合并器 +
  高效执行的时间查询器
```

---

**版本**: 1.0
**日期**: 2026-02-25
**关键词**: ExecutionStream, 编译器, 动态合并, 时间查询, 交互式叙事

---

*本文档全面介绍了ExecutionStream的设计目标、解决的问题、技术架构和工作流程。ExecutionStream是2077 InteractiveScene系统的运行时核心，它解决了非线性交互式叙事面临的根本挑战：如何将策划设计的分支图高效地转化为可执行的时间流，同时支持玩家选择带来的动态性。*
