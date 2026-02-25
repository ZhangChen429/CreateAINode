# 图节点作为Event容器的触发关系模型
> 重新理解SceneGraph：从"逻辑连接"到"触发关系"

---

## 🎯 核心洞察

**你的理解完全正确！**

```
SceneGraph的精确定义：

图节点 = Event容器（SectionNode包含m_events数组）
图连线 = 触发关系（完成后激活下游节点）

示例：

  [SectionNode_A]
       ↓ (OutputSocket::out)
  [SectionNode_B]

实际语义：
  "当SectionNode_A的所有Event执行完成后，
   发送Signal到OutputSocket::out，
   激活SectionNode_B的InputSocket::in，
   触发SectionNode_B开始执行其内部的Events"
```

---

## 📊 第一部分：代码验证

### 1.1 SectionNode = Event容器

```cpp
// 文件：scnsSectionNode.h
class SectionNode : public SceneGraphNode {
    // 核心：这是一个容器
    red::DynArray< THandle< SceneEvent > > m_events;  // 存储多个Event

    // 容器的时间范围
    SceneTime m_refrncDuration;  // 所有Event的时间范围

    // 输出插座：完成时发送信号
    struct OutputSocket {
        static const SocketName out = 0;  // 正常完成
        static const SocketName cancelFwd = 1;  // 取消传递
        static const SocketName transmitSignal = 2;  // 信号传输
    };
};
```

**关键证据**：
```cpp
// SectionNode的处理逻辑
void SectionNode::DoProcess(ProcessingResult& result, ...) {
    // 1. 遍历容器中的所有Event
    for (SceneEvent* event : m_events) {
        // 2. 根据当前时间激活Event
        if (ShouldActivate(event, currentTime)) {
            event->GenerateActionParts(...);
        }
    }

    // 3. 检查是否所有Event都完成
    if (AllEventsCompleted()) {
        // 4. 发送Signal到OutputSocket::out
        result.AddSignal(OutputSocket::out);
    }
}
```

**完全符合你的理解**：
- SectionNode = 容器
- m_events = 容器内的Event列表
- OutputSocket = 完成后触发下游

---

### 1.2 连线 = 触发关系

```cpp
// 文件：scnSceneGraphNode.h
class SceneGraphNode {
    struct OutputSocket {
        SocketName name;
        NodeId targetNodeId;  // 连线指向的目标节点
        SocketName targetSocketName;  // 目标节点的输入插座
    };

    red::DynArray< OutputSocket > m_outputSockets;
};
```

**触发机制**：
```
SectionNode_A完成：
  ↓
DoProcess() 检测所有Event完成
  ↓
AddSignal(OutputSocket::out)
  ↓
SceneGraph处理信号：
  查找OutputSocket::out连接的目标
  targetNodeId = SectionNode_B
  targetSocketName = InputSocket::in
  ↓
激活 SectionNode_B::InputSocket::in
  ↓
SectionNode_B 开始执行其m_events
```

**完全符合你的理解**：
- 连线不是抽象的"逻辑流程"
- 而是具体的"完成后触发"
- Signal系统实现触发机制

---

## 🔍 第二部分：这个理解的优势

### 2.1 语义更清晰

```
之前的理解（抽象）：
  "节点之间有逻辑连接"
  → 什么是逻辑连接？不清楚

你的理解（具体）：
  "节点A完成后触发节点B"
  → 触发语义明确：A的Event执行完 → 发Signal → B开始
```

### 2.2 可以部分表达时间

```
如果每个节点标注duration：

  ┌─────────────────────┐
  │ SectionNode_A       │
  │ duration = 5000ms   │ ← 可以看到这个容器持续5秒
  │ Events: 3个         │
  └──────────┬──────────┘
             ↓ (触发)
  ┌─────────────────────┐
  │ SectionNode_B       │
  │ duration = 3000ms   │ ← 可以看到这个容器持续3秒
  │ Events: 2个         │
  └─────────────────────┘

时间推导：
  SectionNode_A: 0ms - 5000ms
  SectionNode_B: 5000ms - 8000ms (A完成后触发)

优势：
  ✅ 图上可以显示每个容器的duration
  ✅ 可以通过路径累加推算总时长
  ✅ 触发顺序直观
```

### 2.3 符合实际执行流程

```
实际运行时：

Time = 0ms:
  → StartNode触发SectionNode_A
  → SectionNode_A开始执行m_events[0]

Time = 2000ms:
  → SectionNode_A执行m_events[1]

Time = 5000ms:
  → SectionNode_A所有Event完成
  → 发送Signal(OutputSocket::out)
  → 触发SectionNode_B
  → SectionNode_B开始执行m_events[0]

Time = 8000ms:
  → SectionNode_B所有Event完成
  → 发送Signal(OutputSocket::out)
  → 触发EndNode

你的理解完美映射这个过程：
  节点 = Event容器
  连线 = 触发信号
```

---

## ⚠️ 第三部分：矛盾依然存在

### 3.1 并行问题

```
场景：一个节点触发两个并行节点

  ┌─────────────┐
  │ SectionNode │
  │ duration=5s │
  └──────┬──────┘
         ├──→ [Branch_A] (duration=3s)
         └──→ [Branch_B] (duration=4s)

触发关系：
  SectionNode完成 → 同时触发Branch_A和Branch_B

时间问题：
  Branch_A: 5s - 8s
  Branch_B: 5s - 9s

  它们是并行的！
  但如何在"触发关系图"上表达"并行持续到不同时间"？

  ┌─────────────┐
  │ HubNode     │ ← 这个节点何时被触发？
  └─────────────┘

  答案：等待Branch_A和Branch_B都完成
       → Time = 9s (取max)

  问题：
    图上只能看到"两条触发路径汇聚"
    看不到"Branch_B比Branch_A晚1秒完成"
```

**即使节点=Event容器，并行的时间关系依然隐式**

---

### 3.2 分支问题

```
场景：ChoiceNode

  ┌─────────────┐
  │ ChoiceNode  │
  │ duration=0  │ ← 等待玩家选择
  └──────┬──────┘
         ├──→ [Branch_A] (duration=5s)
         ├──→ [Branch_B] (duration=3s)
         └──→ [Branch_C] (duration=4s)

触发关系：
  ChoiceNode完成（玩家选择） → 触发一条分支

运行时（玩家选择A）：
  ChoiceNode → Branch_A → HubNode

时间：
  ChoiceNode: 0s - ?s (等待玩家)
  Branch_A:   ?s - (?+5)s

问题1：玩家何时选择？
  图上无法表达（不确定时间）

问题2：其他分支呢？
  Branch_B和Branch_C在图上存在
  但运行时不执行
  如何在"当前执行路径"上标注？

  观察运行时：
    需要高亮当前执行的节点
    但其他分支依然显示在图上（灰色？隐藏？）
```

**即使节点=Event容器，分支的运行时选择依然是图无法表达的动态性**

---

### 3.3 性能问题

```
查询需求：
  "Time = 5000ms时，应该执行哪些Event？"

图遍历方案：

  function FindEventsAtTime(graph, targetTime) {
      currentTime = 0;
      currentNode = graph.startNode;

      // 遍历图，累加时间
      while (currentNode != null) {
          nodeDuration = currentNode.GetDuration();
          nodeEndTime = currentTime + nodeDuration;

          if (targetTime >= currentTime && targetTime < nodeEndTime) {
              // 找到了！这个节点在执行
              // 但还需要遍历节点内的Events
              for (event in currentNode.events) {
                  eventTime = currentTime + event.localTime;
                  if (eventTime == targetTime) {
                      return event;
                  }
              }
          }

          currentTime = nodeEndTime;
          currentNode = GetNextNode(currentNode);  // 如果有分支？
      }
  }

复杂度：O(节点数 × 事件数)

ExecutionStream方案：

  ActionChannel（已排序）:
    [0ms]:    Event_A
    [2000ms]: Event_B
    [5000ms]: Event_C  ← 二分查找直接定位
    [5500ms]: Event_D
    ...

复杂度：O(log n)

即使理解为"触发关系"，图遍历依然慢于时间排序
```

**触发关系图优化查询"谁触发谁"，不优化查询"Time=X时执行什么"**

---

## 💡 第四部分：新的矛盾表述

### 4.1 更精确的矛盾定义

```
之前的表述：
  "图结构 vs 时间序列"

更精确的表述（基于你的理解）：
  "触发关系图 vs 时间查询流"

具体矛盾：

SceneGraph = 触发关系图
  优化查询："节点A完成后触发谁？"
  回答："触发节点B和节点C"
  ✅ 通过OutputSocket快速查找

  难以查询："Time=5000ms时应该执行什么？"
  回答："需要从StartNode遍历，累加duration，判断位置"
  ❌ O(n)遍历

ExecutionStream = 时间查询流
  优化查询："Time=5000ms时应该执行什么？"
  回答："Event_C和Event_D"
  ✅ 二分查找，O(log n)

  难以查询："Event_C完成后触发谁？"
  回答："ExecutionStream是扁平化的，没有'触发'概念"
  ❌ 信息丢失（编译时已展开）
```

**本质**：
- 触发关系（Who triggers Who）是图的优势
- 时间查询（What at When）是流的优势
- 两者无法兼得

---

### 4.2 为什么"节点=容器"不能解决矛盾

```
即使清楚地理解"节点=Event容器，连线=触发"：

问题1：容器内部的Event时间依然是相对的

  SectionNode_A.m_events = [
      Event_1 { localTime = 0ms },
      Event_2 { localTime = 2000ms }
  ]

  运行时需要绝对时间：
    Event_1: globalTime = ？需要累加所有前置节点的duration
    Event_2: globalTime = ？需要累加所有前置节点的duration + 2000ms

  每帧都要重新计算 → 性能问题

问题2：触发是动态的，时间是静态的

  ChoiceNode的触发：
    图上：3条箭头（3种可能）
    运行时：1条执行（1种实际）

  时间流的生成：
    需要在编译时"预测"或在运行时"生成"
    如果等运行时生成 → 卡顿
    如果编译时生成 → 需要预编译所有分支

问题3：并行触发的时间不确定性

  ┌────────┐
  │ Node_A │
  └────┬───┘
       ├──→ [Branch_1] (duration=3s)
       └──→ [Branch_2] (duration=5s)

  触发是确定的：Node_A → Branch_1 和 Branch_2
  但"何时汇聚"是隐式的：max(3s, 5s) = 5s

  图上看不到这个"max"运算
```

---

## 🎯 第五部分：双核心的必然性

### 5.1 触发关系图的价值（SceneGraph）

```
设计时的核心需求：

需求1："如果玩家选择A，会触发什么？"
  → 查看ChoiceNode的OutputSocket
  → 看到3个分支的触发目标
  ✅ 图清晰表达

需求2："这个Event是在哪个容器里？"
  → 点击Event，查看所属SectionNode
  → 看到节点的上下文（前置、后置节点）
  ✅ 图清晰表达

需求3："如何添加一个新的并行分支？"
  → 创建新SectionNode
  → 从前置节点拉一条连线（OutputSocket → InputSocket）
  ✅ 图直观操作

设计师需要理解：
  ✅ 触发关系（谁触发谁）
  ✅ 分支结构（有几条路径）
  ✅ 汇聚点（路径如何汇合）

设计师不需要关心：
  ❌ 精确的绝对时间（Time=5237ms执行Event_X）
  ❌ 时间窗口查询性能
  ❌ 二分查找算法
```

---

### 5.2 时间查询流的价值（ExecutionStream）

```
运行时的核心需求：

需求1："当前帧（Time=5000-5016ms）需要启动哪些Event？"
  → 二分查找5000ms的位置
  → 线性扫描到5016ms
  → 返回时间窗口内的ActionRecord
  ✅ O(log n + k)，极快

需求2："Event_A应该在什么时候启动？"
  → 直接读取ActionRecord.startTime
  ✅ O(1)，直接访问

需求3："快进10秒"
  → currentTime += 10000
  → 从新时间点开始查询
  ✅ 无需重新遍历图

运行时需要：
  ✅ 绝对时间（不是相对时间）
  ✅ 快速查询（O(log n)）
  ✅ 确定性（不是动态生成）

运行时不需要：
  ❌ 触发关系（已编译成绝对时间）
  ❌ 分支可能性（已选择一条路径）
  ❌ 节点边界（扁平化）
```

---

### 5.3 编译器的转换

```
编译过程：触发关系 → 时间序列

输入：SceneGraph（触发关系图）

  [StartNode]
      ↓ (触发)
  [SectionNode_A] (duration=5s, 3个Event)
      ↓ (完成后触发)
  [SectionNode_B] (duration=3s, 2个Event)
      ↓ (完成后触发)
  [EndNode]

编译算法：

  1. 从StartNode开始遍历触发链
  2. 累加绝对时间
  3. 展开每个容器的Event，计算绝对时间
  4. 按时间排序所有ActionRecord

输出：ExecutionStream（时间查询流）

  ActionChannel（时间排序）:
    [0ms]:    Event_A1
    [1000ms]: Event_A2
    [3000ms]: Event_A3
    [5000ms]: Event_B1
    [6000ms]: Event_B2

转换的本质：
  "A触发B" → "A的Event在0-5s，B的Event在5-8s"
  触发关系 → 时间先后

信息丢失：
  ❌ 触发关系（"B是A触发的"）
  ❌ 容器边界（"Event_B1属于SectionNode_B"）
  ❌ 分支可能性（"还有另外2条路径"）

信息保留：
  ✅ 绝对时间（"Event_B1在5000ms启动"）
  ✅ 执行顺序（"Event_A3先于Event_B1"）
  ✅ 时长（"Event_A1持续1000ms"）
```

---

## 🌟 第六部分：核心结论

### 6.1 你的理解是正确的

```
✅ 节点 = Event容器（SectionNode.m_events）
✅ 连线 = 触发关系（完成后发Signal激活下游）
✅ 这是SceneGraph的精确语义

这个理解的价值：
  ✅ 更清晰的概念模型
  ✅ 更容易理解Signal系统
  ✅ 更容易理解编译过程（触发 → 时间）
```

---

### 6.2 矛盾依然存在

```
即使清楚"节点=容器，连线=触发"：

矛盾依然存在：
  触发关系图 ≠ 时间查询流

原因：
  1. 触发是动态的（分支、并行）
  2. 时间是静态的（绝对值）
  3. 容器内Event的相对时间 → 绝对时间转换有成本
  4. 图遍历（O(n)）慢于二分查找（O(log n)）

因此：
  ❌ 不能只用SceneGraph运行（性能差）
  ❌ 不能只用ExecutionStream设计（无法表达触发关系）
  ✅ 双核心：SceneGraph设计，ExecutionStream执行
```

---

### 6.3 更精确的矛盾表述

```
旧表述：
  "图结构 vs 时间序列"
  → 太抽象

新表述（基于你的理解）：
  "触发关系优化 vs 时间查询优化"

具体：
  SceneGraph优化：
    ✅ 查询"节点A触发谁"（OutputSocket）
    ✅ 查询"谁触发节点B"（InputSocket）
    ✅ 表达分支、并行、汇聚
    ❌ 查询"Time=5s执行什么"（需要遍历）

  ExecutionStream优化：
    ✅ 查询"Time=5s执行什么"（二分查找）
    ✅ 查询"Event_A何时启动"（直接读取）
    ✅ 时间窗口查询
    ❌ 查询"Event_A触发了谁"（信息丢失）

这是数据结构的经典权衡：
  无法同时优化两种查询
  需要两种数据结构 + 编译转换
```

---

## 📚 附录：实际案例

### 案例：q115_00b_hanako

```
SceneGraph视角（触发关系）：

  [StartNode]
      ↓ (触发)
  [SectionNode_Greeting]
    容器内容：
      - ChangeTier Event (Time=0ms)
      - Camera Event (Time=500ms)
      - PlayWorkspot Event (Time=1000ms, Hanako)
      - PlayWorkspot Event (Time=1000ms, V)
    duration = 2000ms
      ↓ (所有Event完成后触发)
  [SectionNode_Dialog]
    容器内容：
      - DialogLine Event (Time=0ms, Hanako说话)
      - ChoiceHub Event (Time=5000ms, 玩家选择)
    duration = 8000ms
      ↓ (选择后触发)
  [ChoiceNode]
      ├──→ [Branch_Polite] (礼貌回应)
      ├──→ [Branch_Cold] (冷淡回应)
      └──→ [Branch_Joke] (开玩笑)
      ↓ (分支完成后触发)
  [HubNode]
      ↓ (汇聚后触发)
  [SectionNode_Farewell]
    容器内容：
      - StopWorkspot Event (Time=0ms, Hanako)
      - StopWorkspot Event (Time=0ms, V)
    duration = 2000ms
      ↓ (完成后触发)
  [EndNode]

设计师看到：
  ✅ 触发链条清晰
  ✅ 容器内容可见（双击查看Events）
  ✅ 分支结构明确（3个选择）
  ✅ 汇聚点清楚（HubNode）

ExecutionStream视角（玩家选择Branch_Polite）：

  ActionChannel（绝对时间）:
    [0ms]:    ChangeTier(Tier=2)
    [500ms]:  Camera(...)
    [1000ms]: PlayWorkspot(Hanako, chair)
    [1000ms]: PlayWorkspot(V, chair)
    [2000ms]: DialogLine(Hanako, "你好V")
    [7000ms]: ShowChoiceHub(...)
    [10000ms]: DialogLine(Hanako, "很高兴你这么说")  ← Branch_Polite的内容
    [12000ms]: DialogLine(V, "...")
    [15000ms]: StopWorkspot(Hanako)
    [15000ms]: StopWorkspot(V)

运行时看到：
  ✅ 精确时间（每个动作何时发生）
  ✅ 快速查询（当前时间应该做什么）
  ✅ 确定性（这次执行的确切序列）
  ❌ 触发关系（看不出"DialogLine是由Branch_Polite触发的"）
  ❌ 其他分支（Branch_Cold和Branch_Joke在ExecutionStream中不存在）
```

---

**版本**: 1.0
**日期**: 2026-02-25
**关键词**: 触发关系, Event容器, SceneGraph, ExecutionStream, Signal系统

---

*本文档确认了你的理解：SceneGraph的节点是Event容器，连线是触发关系。这是一个更精确的概念模型。*

*然而，即使清楚了"触发关系图"的语义，矛盾依然存在："触发关系优化"（图的优势）与"时间查询优化"（流的优势）无法兼得。双核心架构依然是必然的。*

*你的理解让我们对矛盾的本质有了更清晰的认识：不是"图 vs 时间轴"的抽象冲突，而是"触发查询 vs 时间查询"的具体权衡。这是数据结构设计的经典问题：无法用一种结构同时高效支持两种查询。*
