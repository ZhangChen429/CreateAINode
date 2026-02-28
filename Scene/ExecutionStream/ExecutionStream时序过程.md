# 2077 时序计算的真实机制
## 一、不是手写毫秒，而是 SceneGraph "编译"出来的
```mermaid
flowchart TD
    A["策划的工作流程（实际）"] --> B["节点画布操作"]
    B --> C["┌─────────┐      ┌─────────┐      ┌─────────┐
│  开始    │ ───► │ Hanako  │ ───► │ 对话    │
│         │      │ 坐下    │      │ 开始    │
└─────────┘      └─────────┘      └─────────┘
                  │
             属性面板：
             - Workspot: chair_01
             - 过渡动画: stand_to_sit
             - 等待完成: ✓

策划不需要写 '5000ms'
而是通过节点连接表达 '之后'"]
    C --> D["编译/Build"]
    D --> E["ExecutionStream（引擎自动生成）：
ActionChannel:
  [0] StartScene,     t=0ms
  [1] ChangeWork,     t=0ms,   duration=3200ms
  [2] DialogLine,     t=3200ms

时间是系统算出来的，不是策划填的"]
```

## 二、时序来源的几种机制
根据代码分析，2077 的时序计算有以下几种来源：

### 1. 节点连接 = 顺序关系
```mermaid
flowchart LR
    A[A] --> B[B]
    B --> C[C]
    note["SceneGraph 中的连接线决定执行顺序：
A → B → C 表示：
- B 在 A 之后执行
- C 在 B 之后执行

策划操作：拖拽连线
系统行为：自动计算 B.startTime = A.endTime"]
```

### 2. 动画/资源元数据 = 时长来源
```cpp
// 系统从动画资源中读取时长
AnimationClip clip = LoadAnim("stand_to_sit");
Msec duration = clip.GetDuration();  // 3200ms

// ChangeWorkEvent 的时长不是策划填的
// 而是从引用的 Workspot 和动画中读取的
```

```mermaid
note["策划看到的：
┌─────────────────────────────────────┐
│  ChangeWorkEvent                    │
│  ├── Workspot: restaurant_chair     │
│  ├── 入场动画: walk_to_chair        │  ← 时长自动读取
│  ├── 过渡动画: stand_to_sit         │  ← 时长自动读取
│  └── [系统计算] 总时长: 5.2秒       │
└─────────────────────────────────────┘"]
```

### 3. 信号等待 = 同步点
```mermaid
flowchart TD
    A["Hanako坐下"] --> B["WaitAll"]
    C["V坐下"] --> B
    B --> D["对话开始"]
    
    E["不用硬编码时间，而是等待信号：
WaitAll 节点：等待所有输入完成后才继续

编译后的 ExecutionStream：
StimulationChannel:
  等待信号: Hanako.WorkspotSeated
  等待信号: V.WorkspotSeated

ActionChannel:
  DialogStart.startTime = WaitAll.completionTime
  （运行时动态确定，不是编译时固定）"]
```

### 4. 手动时间偏移 = 微调
```mermaid
note["策划确实可以手动调整，但是作为'偏移'而不是'绝对时间'：
┌─────────────────────────────────────┐
│  DialogLineEvent                    │
│  ├── 说话人: Hanako                 │
│  ├── 台词: "你好，V"                │
│  ├── 延迟开始: +0.5秒               │  ← 这是手动微调
│  └── 提前结束: -0.2秒               │  ← 这也是微调
└─────────────────────────────────────┘

系统计算：
  baseTime = 前一个节点的结束时间
  actualStart = baseTime + 500ms（延迟）
  duration = 语音文件时长 - 200ms（提前结束）"]
```

## 三、策划真正需要理解的是什么？
```mermaid
flowchart TD
    A["策划需要理解的（概念层）：
1. 节点类型
   - DialogLineEvent（对话）
   - ChangeWorkEvent（切换Workspot）
   - CameraEvent（镜头）
   - WaitNode（等待）
   - ChoiceNode（选择）

2. 连接关系
   - 顺序执行（A → B）
   - 并行执行（A → B, A → C）
   - 同步点（B + C → D）

3. 信号和条件
   - 等待特定信号
   - 条件分支"] --> B["策划不需要理解的（实现层）：
❌ ExecutionStream 三通道
❌ 毫秒级时间计算
❌ 索引和排序算法
❌ 内存管理"]
```

## 四、SceneGraph 到 ExecutionStream 的编译过程
```mermaid
flowchart TD
    subgraph Step 1: 拓扑排序
        A["输入：SceneGraph（有向无环图）
[A]──►[B]──►[D]
 │          ▲
 └──►[C]────┘"] --> B["输出：节点执行顺序
排序结果：A, B, C, D（或 A, C, B, D）"]
    end
    subgraph Step 2: 时间计算
        C["遍历每个节点：
A: startTime = 0
   duration = GetNodeDuration(A)  // 从资源读取
   endTime = 0 + duration

B: startTime = A.endTime          // 继承前驱
   duration = GetNodeDuration(B)
   endTime = startTime + duration

C: startTime = A.endTime          // 和B并行
   ...

D: startTime = max(B.endTime, C.endTime)  // 等待两者
   ..."]
    end
    subgraph Step 3: 生成 ExecutionStream
        D["for each node in sortedNodes:
    ActionRecord record;
    record.m_startTime = node.calculatedStartTime;
    record.m_conclusionTime = node.calculatedEndTime;
    stream.GetActionChannel().StoreActionRecord(record);

stream.Reindex();  // 按时间排序"]
    end
    B --> C
    C --> D
```

## 五、策划实际的工作界面
```mermaid
flowchart TD
    A["Scene Editor（场景编辑器）"] --> B["┌─────────────────────────────────────────────────────────┐
│                    节点画布                              │
│                                                          │
│   ┌──────┐     ┌──────────┐     ┌──────────┐            │
│   │ 开始 │────►│ Hanako   │────►│ 对话     │            │
│   └──────┘     │ ChangeWork│     │ Section1 │            │
│                └──────────┘     └────┬─────┘            │
│                      │               │                   │
│                      │          ┌────▼─────┐            │
│                      │          │ 选择节点 │            │
│                      │          └────┬─────┘            │
│                      │         ┌─────┼─────┐            │
│                      │         ▼     ▼     ▼            │
│                      │      [选项A][选项B][选项C]       │
│                                                          │
└─────────────────────────────────────────────────────────┘"]
    A --> C["┌─────────────────────────────────────────────────────────┐
│                    属性面板                              │
│  选中节点: Hanako ChangeWork                             │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Performer:     [Hanako        ▼]                   │  │
│  │  Workspot:      [restaurant_chair_01 ▼]             │  │
│  │  Transition:    [stand_to_sit  ▼]                   │  │
│  │                                                     │  │
│  │  ☑ 等待完成后继续                                   │  │
│  │  延迟开始:  [0.0] 秒                                 │  │
│  │                                                     │  │
│  │  [预计时长: 3.2秒]  ← 系统自动计算并显示             │  │
│  └────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘"]
    A --> D["┌─────────────────────────────────────────────────────────┐
│                    时间轴预览                            │
│  0s    1s    2s    3s    4s    5s    6s    7s    8s      │
│  ├─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤      │
│  │████████████████│                                      │
│  │  Hanako坐下    │                                      │
│  │                │████████████████████████│             │
│  │                │     对话 Section1      │             │
│                                                          │
│  ← 这个时间轴是自动生成的预览，策划不需要手动排          │
└─────────────────────────────────────────────────────────┘"]
```

## 六、但策划仍然需要理解的"隐性知识"
```mermaid
flowchart TD
    A["虽然系统自动计算时间，但策划仍需理解："] --> B["1. 动画时长的影响
'为什么对话开始得这么慢？'
→ 因为选了一个5秒的入场动画
→ 需要选择更快的动画或跳过入场"]
    A --> C["2. 并行与串行的选择
串行：A → B → C（总时长 = A + B + C）
并行：A → B 和 A → C → D（总时长可能更短）

策划需要判断：这两件事能同时发生吗？"]
    A --> D["3. 等待点的设置
'对话应该等两人都坐好，还是Hanako坐好就开始？'
→ 影响节点连接方式
→ 影响玩家体验的节奏感"]
    A --> E["4. 中断处理的配置
'玩家在这个阶段拔枪会怎样？'
→ 需要配置中断条件和跳转目标
→ 这部分仍需策划理解业务逻辑"]
```

## 七、总结：2077 时序计算的分层
```mermaid
flowchart TD
    subgraph 策划层（SceneGraph）
        A["- 拖拽节点
- 连接关系
- 选择动画/Workspot
- 配置等待条件
- 微调延迟（可选）"]
    end
    subgraph 编译层（SceneGraph → ExecutionStream）
        B["- 拓扑排序
- 时间计算（从资源元数据）
- 依赖分析
- 生成 ActionRecord"]
    end
    subgraph 执行层（ExecutionStream）
        C["- 按时间触发动作
- 处理信号
- 处理中断"]
    end
    A -->|不需要理解| B
    B -->|自动完成| C
    D["答案：策划不需要手写毫秒，但需要理解逻辑关系
时序是从 SceneGraph 结构 + 资源元数据 编译出来的"]
```

## 八、AI 可以进一步简化什么？
```mermaid
flowchart TD
    A["现状：策划用 SceneGraph，系统自动算时间
已经比手写毫秒好很多了"] --> B["AI 可以进一步做的："]
    B --> C["1. 从文本生成 SceneGraph
输入：'Hanako先坐下，V跟着坐，然后开始对话'
输出：自动生成节点和连线
→ 策划连图都不用画了"]
    B --> D["2. 智能选择动画
输入：'Hanako优雅地坐下'
AI：从动画库中选择最符合'优雅'的坐下动画
→ 策划不用翻动画列表"]
    B --> E["3. 节奏建议
AI分析：这段对话节奏太紧凑，建议在选择前增加停顿
AI建议：当前配置可能导致镜头切换过于频繁
→ 策划得到智能反馈"]
```

### 核心结论
2077 的时序主要靠 SceneGraph 拓扑结构 + 资源元数据自动计算，策划需要理解的是"逻辑关系"而不是"具体毫秒"。这已经是一个相当成熟的抽象层，AI 能进一步简化的是"连图都不用画，用自然语言描述即可"。

### 总结
1. 2077 中策划无需手写毫秒级时间，而是通过 SceneGraph 节点连接表达逻辑关系，系统自动编译生成时序数据；
2. 时序的核心来源包括节点连接关系、动画/资源元数据、信号同步点，手动调整仅作为微调手段；
3. AI 优化的核心方向不是替代现有编译逻辑，而是进一步降低策划的操作成本（文本生成节点、智能选动画、节奏建议）。