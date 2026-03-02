# ExecutionStream 在 AI Agent 时代的架构演进

---

## 核心发现：时间 + 触发器的混合架构

ExecutionStream 的本质并非纯粹的时间驱动，而是**时间 + 触发器的混合架构**。

```mermaid
graph TD
    subgraph "ActionRecord 核心结构"
        AR["ActionRecord"]
        AR --> RT["RecordType"]
        AR --> AI["ActiondefId"]
        AR --> EP["ExecutionPlan ← 关键"]
        AR --> ES["ExecutionStatus"]
        AR --> UD["UserData"]
    end

    subgraph "ExecutionPlan 内部"
        EP --> ST["m_startTime\n时间触发"]
        EP --> CT["m_conclusionTime"]
        EP --> TR1["m_startingTriggers\n开始时触发 ✅"]
        EP --> TR2["m_regularTriggers\n常规触发 ✅"]
        EP --> TR3["m_concludingTriggers\n结束时触发 ✅"]
        EP --> TR4["m_cancelingTriggers\n取消时触发 ✅"]
    end
```

**重要发现**：
- ✅ ExecutionStream 已经包含触发器（Trigger）机制
- ✅ 时间只是触发条件之一，不是唯一条件
- ✅ 架构已经具备事件驱动的基础设施

---

## 一、核心价值：流程编排，而非时间编排

```mermaid
graph LR
    subgraph "ExecutionStream 真正的价值"
        O["编排能力\nOrchestration"]
        I["索引能力\nIndexing"]
        T["触发能力\nTriggering"]
    end

    O --> O1["定义动作序列：A → B → C"]
    O --> O2["支持分支：A → B or C → D"]
    O --> O3["支持合并：CombineSubstream()"]
    O --> O4["支持取消：ControlChannel"]

    I --> I1["快速查询：满足条件X的动作"]
    I --> I2["高效迭代：时间窗口内的动作"]
    I --> I3["可扩展：当前按时间，未来按事件"]

    T --> T1["已有 TriggerList 机制"]
    T --> T2["支持多种触发器"]
    T --> T3["可扩展为事件触发"]
```

---

## 二、从时间驱动到事件驱动的转换

### 2.1 当前架构的局限

```mermaid
flowchart LR
    subgraph "当前（时间驱动）"
        A["ActionRecord\nm_startTime = 5000ms\n固定时间点"]
        B["问题"]
        A --> B
        B --> B1["❌ 必须预先知道精确时间"]
        B --> B2["❌ 玩家行为无法预测"]
        B --> B3["❌ AI 生成内容需提前编排"]
    end
```

### 2.2 未来架构设计

```mermaid
graph TD
    subgraph "EventTrigger 扩展类型"
        ET["EventTrigger.Type"]
        ET --> TM["Time\n时间触发（向后兼容）"]
        ET --> GE["GameEvent\n游戏事件触发"]
        ET --> AC["AICondition\nAI条件触发"]
        ET --> PA["PlayerAction\n玩家行为触发"]
        ET --> CU["Custom\n自定义触发"]
    end

    subgraph "多索引 EventChannel"
        EC["EventChannel"]
        EC --> TI["m_timeIndex\nMsec → Records"]
        EC --> EI["m_eventIndex\nCName → Records"]
        EC --> CI["m_conditionIndex\nConditionId → Records"]
    end

    subgraph "查询接口"
        EC --> Q1["IterateByTime(TimeWindow)"]
        EC --> Q2["IterateByEvent(CName)"]
        EC --> Q3["IterateByCondition(AIFunc)"]
    end
```

### 2.3 示例对比

```mermaid
flowchart LR
    subgraph "当前（时间驱动）"
        C1["ActionRecord\nm_startTime = 5000ms\naction = PlayDialogue\ntext = 预先写好的台词"]
    end

    subgraph "未来（事件驱动）"
        F1["ActionRecord\ntrigger = GameEvent('PlayerAsksQuestion')\naction = GenerateAIDialogue\nagent = HanakoAgent\ncontext = CurrentConversation"]
    end

    C1 -->|"演进"| F1
```

---

## 三、AI Agent 时代的三大价值场景

### 3.1 动态叙事生成

```mermaid
flowchart TD
    subgraph "当前（静态剧本）"
        S1["Time=0s：Hanako 说你好 V"]
        S2["Time=5s：V 选择友好或冷淡"]
        S3["Time=10s：Hanako 响应（预设文本）"]
        S1 --> S2 --> S3
        S3 --> SL["❌ 所有内容写死\n分支有限"]
    end

    subgraph "未来（AI 生成）"
        A1["ConversationStart 事件触发"]
        A2["AIGenerateDialogue\n传入：位置/情绪/关系/任务历史"]
        A3["LLM 实时生成对话内容"]
        A4["动态创建 ActionRecord\n动态合并到 ExecutionStream"]
        A1 --> A2 --> A3 --> A4
        A4 --> AL["✅ 无限对话可能\n✅ 响应玩家历史\n✅ 个性化体验"]
    end
```

### 3.2 响应式 NPC 行为

```mermaid
flowchart TD
    subgraph "当前（预设动画）"
        P1["Time=0ms：Hanako 坐下"]
        P2["Time=1000ms：Hanako 看向 V"]
        P3["Time=3000ms：Hanako 开始说话"]
        P1 --> P2 --> P3
        P3 --> PL["❌ 固定剧本\n无法适应玩家行为"]
    end

    subgraph "未来（AI 实时决策）"
        R1["AICondition 持续观察游戏状态"]
        R2{"玩家拔出武器?"}
        R3["AIAgent.decide()\n选项：逃跑 / 谈判 / 呼叫保镖"]
        R4["动态执行 NPC 响应动作"]
        R1 --> R2
        R2 -->|"是"| R3 --> R4
        R2 -->|"否"| R1
        R4 --> RL["✅ NPC 行为真实可信\n✅ 玩家行为有意义\n✅ 涌现式玩法"]
    end
```

### 3.3 动态剧情分支

```mermaid
flowchart TD
    subgraph "当前（有限分支）"
        B1["编剧预设 3~5 个分支"]
        B2["每个分支预先编排完整内容"]
        B1 --> B2 --> BL["❌ 内容受人力限制"]
    end

    subgraph "未来（无限分支）"
        D1["QuestPointReached 事件触发"]
        D2["StoryAIAgent\n传入：玩家所有选择历史\n世界状态 / 叙事目标 / 游戏风格"]
        D3["AI 生成完整剧情分支"]
        D4["CompileGeneratedQuest()\n转换为 ExecutionStream"]
        D5["CombineSubstream()\n动态合并到主流"]
        D1 --> D2 --> D3 --> D4 --> D5
        D5 --> DL["✅ 真正的玩家选择影响\n✅ 无限重玩价值\n✅ 每次游玩独特"]
    end
```

---

## 四、向后兼容的三阶段扩展方案

### 阶段 1：扩展触发条件（向后兼容）

```mermaid
graph TD
    subgraph "TriggerCondition 扩展"
        TC["TriggerCondition.Type"]
        TC --> TM["Time\n原有：时间触发\nm_time: Msec"]
        TC --> EV["Event\n新增：事件触发\nm_eventName: CName"]
        TC --> AI["AICondition\n新增：AI条件\nm_condition: function"]
        TC --> PA["PlayerAction\n新增：玩家行为\nm_playerAction: enum"]
        TC --> CO["Composite\n新增：组合条件\nm_composite: vector"]
    end

    subgraph "向后兼容保证"
        GS["GetStartTime() 接口保留\nType==Time → 返回 m_time\nType!=Time → 返回 unknownTime"]
    end
```

### 阶段 2：多索引支持

```mermaid
graph TD
    subgraph "ActionChannel 扩展"
        AC["ActionChannel"]
        AC --> OS["原有：时间索引\nDynArray 按时间排序"]
        AC --> NI["新增：MultiIndex"]
        NI --> TI["m_timeIndex\nMsec → vector&lt;RecordIdx&gt;"]
        NI --> EI["m_eventIndex\nCName → vector&lt;RecordIdx&gt;"]
        NI --> CI["m_conditionIndex\nHash → vector&lt;RecordIdx&gt;"]
    end

    subgraph "新增查询接口"
        Q1["IterateByTime(TimeWindow)\n原有逻辑，向后兼容"]
        Q2["IterateByEvent(CName)\n查 m_eventIndex"]
        Q3["IterateByCondition(func)\n遍历条件索引"]
    end

    AC --> Q1
    AC --> Q2
    AC --> Q3
```

### 阶段 3：运行时动态生成

```mermaid
sequenceDiagram
    participant AI as AIAgentManager
    participant DEM as DynamicExecutionManager
    participant ES as ExecutionStream
    participant PL as Player

    PL->>DEM: OnPlayerAction(action)
    DEM->>ES: IterateByEvent(action.GetEventName())
    ES-->>DEM: 匹配的 ActionRecord[]
    DEM->>DEM: ExecuteAction(record)
    DEM->>AI: NotifyPlayerAction(action)

    AI->>AI: LLM 生成响应动作
    AI->>DEM: OnAIActionGenerated(aiAction)
    DEM->>DEM: 创建新 ActionRecord
    DEM->>ES: StoreActionRecord(record)
    DEM->>ES: Reindex()
```

---

## 五、完整工作流：AI 驱动的餐厅对话

```mermaid
flowchart TD
    subgraph "① 初始化"
        I1["加载基础场景结构（骨架）\n不是完整剧本"]
        I2["StartNode → SectionNode → AIDecisionPoint → EndNode"]
        I1 --> I2
    end

    subgraph "② 玩家进入触发"
        T1["Event: PlayerEnterRestaurant"]
        T2["SetupRestaurantScene\n加载场景资源"]
        T3["Event: RestaurantReady"]
        T1 --> T2 --> T3
    end

    subgraph "③ AI 生成开场白"
        G1["HanakoAgent 接收 RestaurantReady"]
        G2["LLM 生成：\n输入：角色设定+场景上下文+关系状态\n约束：专业但温暖/2-3句话/日式商务礼仪"]
        G3["动态创建 ActionRecord\n播放对话+面部动画+情绪"]
        G4["等待：PlayerSpeak 或 Timeout(10s)"]
        G1 --> G2 --> G3 --> G4
    end

    subgraph "④ 玩家发言响应"
        P1["语音/文字输入"]
        P2["IntentRecognition 分析意图"]
        P3["HanakoAgent 生成响应\n传入：对话历史+情绪状态+意图"]
        P4["播放响应对话"]
        P5{"是否触发任务?"}
        P1 --> P2 --> P3 --> P4 --> P5
    end

    subgraph "⑤ AI 动态生成任务"
        Q1["StoryAIAgent\n传入：玩家等级+选择历史+世界状态"]
        Q2["LLM 生成任务内容\n（基于玩家擅长玩法定制）"]
        Q3["CompileGeneratedQuest()\n转为 ExecutionStream"]
        Q4["CombineSubstream(questStream, currentTime)"]
        Q1 --> Q2 --> Q3 --> Q4
    end

    I2 --> T1
    T3 --> G1
    G4 --> P1
    P5 -->|"是"| Q1
    P5 -->|"否"| G4
```

---

## 六、ExecutionStream 为何天然适配 AI 时代

```mermaid
graph TD
    subgraph "已有的基础设施（无需重建）"
        F1["TriggerList 机制\n只需扩展触发类型"]
        F2["CombineSubstream()\nAI 生成内容动态插入"]
        F3["Reindex()\n运行时修改后重新组织"]
        F4["三通道分离\nAction / Control / Stimulation"]
    end

    subgraph "与 AI 的天然契合"
        F1 --> A1["事件触发 → AI 决策触发"]
        F2 --> A2["预设分支 → AI 生成分支"]
        F3 --> A3["静态内容 → 动态内容"]
        F4 --> A4["ActionChannel: AI 生成动作\nControlChannel: AI 决策流程控制\nStimulationChannel: AI 预测资源需求"]
    end
```

---

## 七、当前 vs 未来：全面对比

```mermaid
graph LR
    subgraph "当前（时间驱动）"
        C1["内容生成：编剧手工创作"]
        C2["触发机制：固定时间点"]
        C3["分支数量：3~10 个预设"]
        C4["NPC 行为：预设动画序列"]
        C5["玩家影响：有限选择点"]
        C6["重玩价值：固定剧本"]
        C7["开发成本：高（手工编排）"]
        C8["内容规模：受限于人力"]
    end

    subgraph "未来（事件 + AI 驱动）"
        F1["内容生成：AI 实时生成"]
        F2["触发机制：事件+条件+AI决策"]
        F3["分支数量：无限动态分支"]
        F4["NPC 行为：AI 实时决策"]
        F5["玩家影响：持续的行为影响"]
        F6["重玩价值：每次独特"]
        F7["开发成本：中（AI 辅助）"]
        F8["内容规模：几乎无限"]
    end

    C1 -->|"演进"| F1
    C2 -->|"演进"| F2
    C3 -->|"演进"| F3
    C4 -->|"演进"| F4
    C5 -->|"演进"| F5
    C6 -->|"演进"| F6
    C7 -->|"演进"| F7
    C8 -->|"演进"| F8
```

---

## 八、实施路线图

```mermaid
flowchart LR
    subgraph "Phase 1（1~2 个月）\n向后兼容扩展"
        P1A["扩展 TriggerCondition\n支持事件触发"]
        P1B["添加 EventChannel 多索引"]
        P1C["保持现有时间驱动完全兼容"]
        P1A --> P1B --> P1C
    end

    subgraph "Phase 2（2~3 个月）\nAI 集成"
        P2A["集成 LLM API"]
        P2B["实现 AIAgent 基础框架"]
        P2C["动态生成简单对话"]
        P2A --> P2B --> P2C
    end

    subgraph "Phase 3（3~6 个月）\n高级 AI 特性"
        P3A["AI 生成完整剧情分支"]
        P3B["NPC AI 实时决策"]
        P3C["玩家行为意图识别"]
        P3A --> P3B --> P3C
    end

    subgraph "Phase 4（持续）\n生产化"
        P4A["性能优化"]
        P4B["内容质量控制"]
        P4C["A/B 测试系统"]
        P4A --> P4B --> P4C
    end

    P1C --> P2A
    P2C --> P3A
    P3C --> P4A
```

---

## 结论

> **CDPR 设计 ExecutionStream 时可能没有预见 AI 时代，但这个架构意外地完美适配了未来的需求。**

三个关键洞察：

1. **核心价值不是时间编排**，而是**动作序列的灵活编排和执行**——时间只是触发维度之一
2. **架构已经具备基础**：TriggerList、CombineSubstream、Reindex 三个机制天然支持 AI 驱动
3. **转换成本可控**：向后兼容扩展，无需推倒重来，三阶段渐进演进
