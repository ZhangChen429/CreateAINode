# ExecutionStream 问题定义与 CDPR 解决方案

---

## 一、核心问题定义

> **一句话定义问题：如何在开放世界游戏中，让"精心编排的叙事"与"玩家自由行为"共存？**

```mermaid
graph LR
    subgraph 精心编排
        A1[对话有顺序]
        A2[动作有时序]
        A3[镜头有节奏]
        A4[NPC有表演]
    end

    subgraph 玩家自由
        B1[随时走开]
        B2[随时攻击]
        B3[随时跳过]
        B4[随时做别的]
    end

    精心编排 <-->|矛盾| 玩家自由

    subgraph CDPR解法
        C1[ActionChannel 做什么]
        C2[ControlChannel 停什么]
        C3[StimulationChannel 等什么]
        C1 --- C2 --- C3
    end

    玩家自由 --> CDPR解法
    精心编排 --> CDPR解法
```

**传统解法（二选一）：**

- **过场动画**：完全剥夺玩家控制 → 打断沉浸
- **纯脚本**：完全交给玩家 → 叙事混乱

**CDPR 的挑战（两者都要）：**

- 保持精确的叙事编排
- 同时允许玩家有限度的自由
- 自由行为能被优雅处理

---

## 二、问题分解：五个子问题

```mermaid
flowchart LR
    subgraph 问题层
        P1[时序编排\n何时做什么]
        P2[中断处理\n如何停止]
        P3[系统协调\n等谁完成]
        P4[动态组合\n运行时拼接]
        P5[性能\n快速查询]
    end

    subgraph 方案层
        S1[ActionChannel\n+ 时间索引]
        S2[ControlChannel\n+ Target机制]
        S3[StimulationChannel\n+ 信号机制]
        S4[CombineSubstream\n+ 时间平移]
        S5[延迟索引 + 二分查找\n+ 小对象优化]
    end

    subgraph 核心思想
        T1[时间 = 数组索引\n时间关系隐含在顺序中]
        T2[做与停分离\n独立控制平面]
        T3[事件总线模式\n解耦间接通信]
        T4[预编译 + 动态组合]
        T5[批量优于频繁\n栈优于堆]
    end

    P1 --> S1 --> T1
    P2 --> S2 --> T2
    P3 --> S3 --> T3
    P4 --> S4 --> T4
    P5 --> S5 --> T5
```

### 问题 1：时序编排问题

**需求**："Hanako 说完话后，V 坐下，镜头切换，然后对话继续"

**困难**：Hanako 说话要多久取决于语音文件，V 坐下要多久取决于动画长度，镜头何时切取决于前面两个的完成时间，如果某个动画变了，后面全部要重算。

**本质**：如何表达和计算"事件之间的时间关系"？

### 问题 2：中断处理问题

**需求**："对话进行中，玩家突然拔枪攻击 NPC"

**困难**：当前对话要立即停止，NPC 动画要中断并切换到战斗状态，已播放的内容不能回滚，后续内容不能继续执行，资源要正确清理。

**本质**：如何优雅地"停止正在做的事"？

### 问题 3：系统协调问题

**需求**："等 NPC 坐好后再开始对话"

**困难**："坐好"是 Workspot 系统的事，"对话"是 Dialog 系统的事，两个系统互不知道对方，需要某种协调机制。

**本质**：如何让"不同系统"知道"彼此的状态"？

### 问题 4：动态组合问题

**需求**："根据玩家选择，播放不同的后续内容"

**困难**：选项 A/B/C 的内容不同，每个分支的时长不同，后续内容的时间要动态调整，可能有嵌套分支。

**本质**：如何"运行时拼接"多段内容？

### 问题 5：性能问题

**需求**："复杂场景有数百个事件，仍需 60fps"

**困难**：每帧要检查哪些事件该执行，遍历数百个事件太慢，频繁插入/删除影响性能，内存分配要最小化。

**本质**：如何高效地"查询和管理大量时间事件"？

---

## 三、CDPR 的解决方案

### ExecutionStream 整体架构

```mermaid
graph TD
    ES[ExecutionStream 执行流]

    ES --> AC[ActionChannel 动作通道]
    ES --> CC[ControlChannel 控制通道]
    ES --> SC[StimulationChannel 刺激通道]

    AC --> AR["ActionRecord[] 按 startTime 排序"]
    AR --> |"二分查找 O(log n)"| QRY[查询当前窗口事件]

    CC --> CR["ControlRequest[] cancelAction / cancelStream"]
    CR --> |Target 匹配| INT[中断目标动作]

    SC --> SR["StimulationRecord[] pending → delivered"]
    SR --> |信号检测| TRG[触发下一动作]

    ES --> CS[CombineSubstream 子流合并]
    CS --> |时间偏移 offset| MRG[合并后重新索引]
```

---

### 解决问题 1：时序编排 → ActionChannel + 时间索引

**方案**：ActionChannel = 按时间排序的事件数组

```mermaid
sequenceDiagram
    participant ES as ExecutionStream
    participant AC as ActionChannel
    participant EXE as 执行器

    Note over AC: 初始化：插入动作记录（m_indexed=false）
    AC->>AC: StoreActionRecord(Act1, start=0ms)
    AC->>AC: StoreActionRecord(Act2, start=100ms)
    AC->>AC: StoreActionRecord(Act3, start=500ms)

    Note over AC: 查询前：Reindex() 一次性排序
    EXE->>AC: Reindex()
    AC->>AC: std::sort by startTime

    loop 每帧 tick
        EXE->>AC: IterateStartingRecords(TimeWindow)
        AC-->>EXE: 返回窗口内事件（二分查找 O(log n)）
        EXE->>EXE: 执行到期事件
    end
```

**数据结构**：

```cpp
ActionRecord {
    m_executionPlan.m_startTime      // 开始时间
    m_executionPlan.m_conclusionTime // 结束时间
}

ActionChannel {
    DynArray<ActionRecord> m_stream; // 按 startTime 排序
}
```

**为什么有效**：时间关系隐含在数组顺序中；二分查找快速定位当前事件；Plan/Status 分离支持"计划 vs 实际"的对比。

---

### 解决问题 2：中断处理 → ControlChannel

**方案**：ControlChannel = 独立的控制请求队列，"做什么"和"停止什么"是两件独立的事。

```mermaid
sequenceDiagram
    participant Player as 玩家
    participant CC as ControlChannel
    participant AC as ActionChannel
    participant EXE as 执行器

    Note over AC: 正在执行对话 [0ms ~ 5000ms]
    EXE->>AC: 执行 Dialog_Action（开始于 0ms）

    Player->>CC: 拔枪触发取消请求（2500ms）
    CC->>CC: 记录 Request{type=cancelAction, time=2500ms}

    EXE->>CC: 检测控制请求
    CC-->>EXE: 返回 cancelAction@2500ms
    EXE->>AC: 中断 Dialog_Action（2500ms）
    EXE->>EXE: 清理资源，切换战斗状态
    Note over AC: 后续动作不再执行
```

**数据结构**：

```cpp
ControlChannel::Request {
    Type m_type;           // cancelActionInstance / cancelExecutionStream
    Target m_target;       // 取消哪个动作/哪个流
    Msec m_occurrenceTime; // 何时执行这个取消
}
```

**为什么有效**：取消逻辑和执行逻辑分离，互不干扰；可以预设取消条件（定时取消）；可以运行时动态添加取消请求（玩家行为触发）；Target 机制灵活，可取消单个动作或整个流。

---

### 解决问题 3：系统协调 → StimulationChannel

**方案**：StimulationChannel = 统一的信号/事件总线，不让系统直接依赖彼此，通过"刺激/信号"间接通信。

```mermaid
sequenceDiagram
    participant WS as Workspot系统
    participant SC as StimulationChannel
    participant EXE as Scene执行器
    participant DLG as Dialog系统

    WS->>WS: NPC 开始坐下动画
    WS->>WS: 动画播放中...
    WS->>SC: 发送 Stimulation{type=Seated, time=3200ms}
    SC->>SC: 记录信号，status=pending

    Note over SC: 时间推进到 3200ms
    SC->>SC: status → delivered

    EXE->>SC: 检测信号 Seated
    SC-->>EXE: 信号已 delivered
    EXE->>DLG: 触发对话开始

    Note over WS,DLG: Workspot 与 Dialog 完全解耦，互不知晓
```

**数据结构**：

```cpp
StimulationRecord {
    Stimulation m_stimulation;         // 信号类型
    ExecutionPlan m_executionPlan;     // 何时发生
    ExecutionStatus m_executionStatus; // pending → delivered
}
```

**为什么有效**：系统解耦，Workspot 不知道 Dialog，Dialog 不知道 Workspot；统一接口，所有系统用同样方式发送/接收信号；可追溯，信号有时间戳；状态明确，`pending` vs `delivered` 清晰表达信号生命周期。

---

### 解决问题 4：动态组合 → CombineSubstream

**方案**：CombineSubstream = 子流合并 + 时间偏移

```mermaid
flowchart TD
    subgraph 主流
        M1["A1 (0ms~100ms)"] --> M2["A2 (100ms~300ms)"]
    end

    subgraph 子流_原始
        S1["B1 (0ms~200ms)"] --> S2["B2 (200ms~400ms)"]
    end

    subgraph 子流_偏移后_offset_300ms
        O1["B1 (300ms~500ms)"] --> O2["B2 (500ms~700ms)"]
    end

    subgraph 合并结果
        R1["A1 (0ms)"] --> R2["A2 (100ms)"] --> R3["B1 (300ms)"] --> R4["B2 (500ms)"]
    end

    子流_原始 -->|所有时间 + 300ms| 子流_偏移后_offset_300ms
    主流 --> 合并结果
    子流_偏移后_offset_300ms --> 合并结果
```

**核心接口**：

```cpp
void ExecutionStream::CombineSubstream(
    const ExecutionStream& other,  // 要合并的子流
    Uint32 combinationOffsetMsec   // 时间偏移量
);
```

**实现细节**：

```cpp
for (ActionRecord record : other.m_stream) {
    record.m_executionPlan.m_startTime += offsetMsec;
    record.m_executionPlan.m_conclusionTime += offsetMsec;
    StoreActionRecord(std::move(record));
}
m_indexed = false;  // 标记需要重新排序
```

**为什么有效**：子流可以预编译，每个分支独立编译运行时组合；时间偏移只需加法 O(n)；三通道统一处理，Action/Control/Stimulation 一起偏移；合并时不排序，查询前一次性排序。

---

### 解决问题 5：性能 → 延迟索引 + 小对象优化

```mermaid
graph TD
    PERF[性能优化体系]

    PERF --> LZ[延迟索引\nLazy Indexing]
    PERF --> BS[二分查找\nBinary Search]
    PERF --> SO[小对象优化\nTriggerList N]
    PERF --> VO[值对象编码\nUserData]

    LZ --> LZ1["插入: m_indexed = false，O(1)"]
    LZ --> LZ2["查询前: Reindex()，O(n log n) 一次"]

    BS --> BS1[数组已排序]
    BS --> BS2["std::lower_bound，O(log n)"]
    BS --> BS3["只处理窗口内事件，O(k)"]

    SO --> SO1["StaticArray 栈预分配\n≤N 个：零堆分配"]
    SO --> SO2[超出后：DynArray 堆分配]

    VO --> VO1["FixedArray Uint64 x5 = 40字节固定"]
    VO --> VO2[无指针，支持 memcpy / hash / compare]
```

| 技术 | 问题 | 解决 | 效果 |
|---|---|---|---|
| 延迟索引 | 频繁插入导致频繁排序 | 插入标记 `false`，查询前统一 `Reindex()` | 插入 O(1)，排序 O(n log n) 一次 |
| 二分查找 | 每帧遍历所有事件太慢 | `std::lower_bound` 定位时间窗口 | 查询 O(log n + k) |
| 小对象优化 | 触发器频繁堆分配 | `StaticArray<N>` 栈预分配 | 常见情况零堆分配 |
| 值对象编码 | 额外信息需要指针 | `FixedArray<Uint64, 5>` 固定 40 字节 | 无指针，可直接 memcpy |

---

## 四、解决方案总结

```mermaid
graph TD
    subgraph 架构哲学
        PH1[关注点分离]
        PH2[数据驱动]
        PH3[延迟求值]
    end

    subgraph 三通道
        AC2[ActionChannel\n做什么]
        CC2[ControlChannel\n停什么]
        SC2[StimulationChannel\n等什么]
    end

    subgraph 能力扩展
        CS2[CombineSubstream\n预编译 + 动态组合]
        PE2[延迟索引 + 二分查找\n+ 小对象优化]
    end

    PH1 --> 三通道
    PH2 --> 三通道
    PH3 --> PE2
    PH3 --> CS2
    三通道 --> CS2
    CS2 --> PE2
```

| 问题 | CDPR 方案 | 核心思想 |
|---|---|---|
| 时序编排 "何时做什么" | ActionChannel + 时间排序 | 时间 = 数组索引，时间关系隐含在顺序中 |
| 中断处理 "如何停止" | ControlChannel + Target 机制 | "做"和"停"分离，独立的控制平面 |
| 系统协调 "等谁完成" | StimulationChannel + 信号机制 | 事件总线模式，解耦的间接通信 |
| 动态组合 "运行时拼接" | CombineSubstream + 时间平移 | 预编译 + 动态组合 |
| 性能 "快速查询" | 延迟索引 + 二分查找 + 小对象优化 | 批量优于频繁，空间换时间，栈优于堆 |

---

## 五、一句话总结

> **问题**：开放世界游戏中，如何让"精确编排的叙事"与"玩家自由行为"共存？
>
> **CDPR 的答案**：
>
> ```
> ExecutionStream = ActionChannel（做什么）
>                + ControlChannel（停什么）
>                + StimulationChannel（等什么）
> ```
>
> 三个独立的通道，各自解决一类问题，通过统一的时间索引协调工作，用 `CombineSubstream` 支持动态组合。
>
> **这是一个"把复杂问题分解为正交子问题"的经典案例。**
