# InteractiveScene系统核心元素关系图
> 7大核心元素及其交织关系的可视化解析

---

## 🎯 核心元素总览

InteractiveScene系统由**7个核心元素**构成，它们相互依赖、协同工作，形成一个完整的交互式叙事引擎。

```
核心元素清单：

1. 场景图 (Scene Graph)          - 叙事的逻辑结构
2. 执行流 (Execution Stream)     - 时间驱动的事件序列
3. Tier系统 (Tier System)        - 控制权分级管理
4. 信号系统 (Signal System)      - 节点间通信机制
5. 演员系统 (Actor System)       - 参与叙事的实体
6. 中断系统 (Interrupt System)   - 容错与分支处理
7. 资源管理 (Resource Manager)   - 预加载与流式管理
```

---

## 📊 第一层：宏观架构关系

### 系统全景图

```mermaid
graph TB
    subgraph "设计时 Design Time"
        A1[场景设计师]
        A2[场景图编辑器]
        A3[场景资源<br/>Scene Asset]
    end

    subgraph "运行时 Runtime - 核心元素"
        direction TB

        B1[场景图<br/>Scene Graph]
        B2[执行流<br/>Execution Stream]
        B3[Tier系统<br/>Tier System]
        B4[信号系统<br/>Signal System]
        B5[演员系统<br/>Actor System]
        B6[中断系统<br/>Interrupt System]
        B7[资源管理<br/>Resource Manager]
    end

    subgraph "游戏系统 Game Systems"
        C1[输入系统<br/>Input]
        C2[摄像机系统<br/>Camera]
        C3[动画系统<br/>Animation]
        C4[音频系统<br/>Audio]
        C5[AI系统<br/>AI]
        C6[物理系统<br/>Physics]
    end

    %% 设计时到运行时
    A1 --> A2
    A2 --> A3
    A3 -->|加载| B1

    %% 核心元素之间的关系
    B1 -->|生成| B2
    B1 -->|触发| B4
    B2 -->|控制| B3
    B2 -->|操作| B5
    B4 -->|激活节点| B1
    B6 -->|监控| B1
    B6 -->|修改| B2
    B7 -->|预加载| B1

    %% 运行时到游戏系统
    B3 -->|限制| C1
    B3 -->|约束| C2
    B2 -->|驱动| C3
    B2 -->|触发| C4
    B5 -->|控制| C5
    B5 -->|参与| C6

    %% 样式
    style B1 fill:#ff6b6b
    style B2 fill:#4ecdc4
    style B3 fill:#ffe66d
    style B4 fill:#95e1d3
    style B5 fill:#a8e6cf
    style B6 fill:#ffaaa5
    style B7 fill:#ffd3b6
```

---

## 🔍 第二层：核心元素详解

### 元素1：场景图 (Scene Graph)

```mermaid
graph LR
    subgraph "场景图结构"
        direction TB
        A[场景图<br/>Scene Graph]

        A --> B[节点集合<br/>Nodes]
        A --> C[连接集合<br/>Connections]
        A --> D[变量集合<br/>Variables]

        B --> B1[对话节点<br/>Dialogue Node]
        B --> B2[选择节点<br/>Choice Node]
        B --> B3[动作节点<br/>Action Node]
        B --> B4[分支节点<br/>Branch Node]
        B --> B5[等待节点<br/>Wait Node]

        C --> C1[输出插座<br/>Output Socket]
        C --> C2[输入插座<br/>Input Socket]
        C --> C3[连接关系<br/>Link]
    end

    style A fill:#ff6b6b
```

**核心职责**：
- 定义叙事的逻辑流程
- 组织节点的执行顺序
- 管理分支和条件判断

**与其他元素的关系**：
- → 执行流：场景图被解析生成执行流
- → 信号系统：节点间通过信号通信
- → 中断系统：中断时保存图状态
- → 资源管理：图中引用的资源需要预加载

---

### 元素2：执行流 (Execution Stream)

```mermaid
graph TB
    subgraph "执行流结构"
        A[执行流<br/>Execution Stream]

        A --> B[动作通道<br/>Action Channel]
        A --> C[控制通道<br/>Control Channel]
        A --> D[激活通道<br/>Activation Channel]

        B --> B1[时间轴事件<br/>Timed Events]
        B --> B2[播放动画<br/>Play Animation]
        B --> B3[切换Tier<br/>Change Tier]
        B --> B4[播放音效<br/>Play Audio]
        B --> B5[触发VFX<br/>Trigger VFX]

        C --> C1[启用节点<br/>Enable Node]
        C --> C2[禁用节点<br/>Disable Node]
        C --> C3[跳转节点<br/>Jump to Node]

        D --> D1[节点激活<br/>Node Activation]
        D --> D2[信号传播<br/>Signal Propagation]
    end

    style A fill:#4ecdc4
```

**核心职责**：
- 将场景图转化为时间序列
- 精确控制事件触发时机
- 支持回放和快进

**与其他元素的关系**：
- ← 场景图：从图中生成
- → Tier系统：通过动作通道切换Tier
- → 演员系统：向演员发送动作指令
- ← 中断系统：中断时修改执行流

---

### 元素3：Tier系统 (Tier System)

```mermaid
graph TB
    subgraph "Tier系统结构"
        A[Tier系统<br/>Tier System]

        A --> B[Tier定义<br/>Tier Definitions]
        A --> C[Tier堆栈<br/>Tier Stack]
        A --> D[约束应用<br/>Constraint Application]

        B --> B1[Tier 1<br/>完全自由]
        B --> B2[Tier 2<br/>轻度限制]
        B --> B3[Tier 3<br/>中度限制]
        B --> B4[Tier 4<br/>重度限制]
        B --> B5[Tier 5<br/>完全控制]

        C --> C1[优先级队列<br/>Priority Queue]
        C --> C2[激活Tier<br/>Active Tier]

        D --> D1[输入限制<br/>Input Restrictions]
        D --> D2[视角约束<br/>Camera Constraints]
        D --> D3[移动限制<br/>Movement Restrictions]
        D --> D4[UI调整<br/>UI Changes]
    end

    style A fill:#ffe66d
```

**核心职责**：
- 管理玩家控制权
- 动态调整输入映射
- 约束摄像机和移动

**与其他元素的关系**：
- ← 执行流：接收Tier切换指令
- → 游戏系统：应用约束到输入/摄像机
- ← 中断系统：中断可能改变Tier

---

### 元素4：信号系统 (Signal System)

```mermaid
graph LR
    subgraph "信号系统流程"
        A[信号源<br/>Signal Source]
        B[输出插座<br/>Output Socket]
        C[信号队列<br/>Signal Queue]
        D[信号分发<br/>Signal Dispatcher]
        E[输入插座<br/>Input Socket]
        F[目标节点<br/>Target Node]

        A -->|发送| B
        B -->|入队| C
        C -->|轮询| D
        D -->|路由| E
        E -->|激活| F
        F -.->|产生新信号| A
    end

    style D fill:#95e1d3
```

**核心职责**：
- 节点间通信
- 激活下游节点
- 传播执行流

**与其他元素的关系**：
- ← 场景图：插座定义在节点上
- ← 执行流：信号触发通过激活通道
- → 场景图：信号激活下一个节点

---

### 元素5：演员系统 (Actor System)

```mermaid
graph TB
    subgraph "演员系统结构"
        A[演员系统<br/>Actor System]

        A --> B[演员注册<br/>Actor Registry]
        A --> C[演员控制<br/>Actor Control]
        A --> D[状态管理<br/>State Management]

        B --> B1[玩家<br/>Player]
        B --> B2[NPC列表<br/>NPCs]
        B --> B3[道具列表<br/>Props]
        B --> B4[载具<br/>Vehicles]

        C --> C1[动画控制<br/>Animation Control]
        C --> C2[AI接管<br/>AI Override]
        C --> C3[位置控制<br/>Position Control]
        C --> C4[注视控制<br/>Look-At Control]

        D --> D1[保存状态<br/>Save State]
        D --> D2[恢复状态<br/>Restore State]
        D --> D3[绑定/解绑<br/>Bind/Unbind]
    end

    style A fill:#a8e6cf
```

**核心职责**：
- 管理场景参与者
- 接管实体控制权
- 保存和恢复状态

**与其他元素的关系**：
- ← 场景图：定义需要哪些演员
- ← 执行流：接收动作指令
- ← 中断系统：中断时保存演员状态
- → 游戏系统：控制AI、动画、物理

---

### 元素6：中断系统 (Interrupt System)

```mermaid
graph TB
    subgraph "中断系统结构"
        A[中断系统<br/>Interrupt System]

        A --> B[条件监控<br/>Condition Monitoring]
        A --> C[中断处理<br/>Interrupt Handling]
        A --> D[返回管理<br/>Return Management]

        B --> B1[距离条件<br/>Distance]
        B --> B2[战斗条件<br/>Combat]
        B --> B3[超时条件<br/>Timeout]
        B --> B4[事件条件<br/>Event]
        B --> B5[自定义条件<br/>Custom]

        C --> C1[保存上下文<br/>Save Context]
        C --> C2[跳转分支<br/>Jump to Branch]
        C --> C3[触发中断内容<br/>Execute Interrupt]

        D --> D1[返回条件检查<br/>Check Return]
        D --> D2[恢复上下文<br/>Restore Context]
        D --> D3[继续主线<br/>Resume Main]
        D --> D4[永久分支<br/>Permanent Branch]
    end

    style A fill:#ffaaa5
```

**核心职责**：
- 监控中断条件
- 保存和恢复场景状态
- 管理分支跳转

**与其他元素的关系**：
- ← 场景图：监控节点执行
- ← 执行流：可以修改执行流
- → 演员系统：保存演员状态
- → Tier系统：可能需要临时改变Tier

---

### 元素7：资源管理 (Resource Manager)

```mermaid
graph TB
    subgraph "资源管理结构"
        A[资源管理器<br/>Resource Manager]

        A --> B[预加载<br/>Preloading]
        A --> C[流式加载<br/>Streaming]
        A --> D[卸载<br/>Unloading]

        B --> B1[静态分析<br/>Static Analysis]
        B --> B2[依赖扫描<br/>Dependency Scan]
        B --> B3[后台加载<br/>Background Load]

        C --> C1[预测加载<br/>Predictive Load]
        C --> C2[优先级队列<br/>Priority Queue]
        C --> C3[距离触发<br/>Distance Trigger]

        D --> D1[引用计数<br/>Reference Count]
        D --> D2[垃圾回收<br/>Garbage Collection]
        D --> D3[内存管理<br/>Memory Management]
    end

    style A fill:#ffd3b6
```

**核心职责**：
- 预测资源需求
- 异步加载资源
- 管理内存占用

**与其他元素的关系**：
- ← 场景图：分析图中引用的资源
- ← 执行流：预测未来需要的资源
- ← 演员系统：加载演员相关资源

---

## 🔗 第三层：核心交互流程

### 流程1：场景启动流程

```mermaid
sequenceDiagram
    participant User as 玩家触发
    participant SG as 场景图
    participant RM as 资源管理器
    participant ES as 执行流
    participant AS as 演员系统
    participant TS as Tier系统
    participant SS as 信号系统

    User->>SG: 进入触发区域
    activate SG

    SG->>RM: 请求加载资源
    activate RM
    RM-->>SG: 资源就绪
    deactivate RM

    SG->>AS: 绑定演员实体
    activate AS
    AS-->>SG: 演员准备完成
    deactivate AS

    SG->>ES: 生成执行流
    activate ES

    ES->>TS: 切换到Tier 2
    activate TS
    TS-->>ES: Tier应用完成
    deactivate TS

    ES->>AS: 发送动作指令
    AS-->>ES: 动作执行中
    deactivate AS

    ES->>SS: 完成信号
    activate SS
    SS->>SG: 激活下一节点
    SS-->>ES: 信号已传播
    deactivate SS

    deactivate ES
    deactivate SG
```

---

### 流程2：中断处理流程

```mermaid
sequenceDiagram
    participant Player as 玩家行为
    participant IS as 中断系统
    participant SG as 场景图
    participant ES as 执行流
    participant AS as 演员系统
    participant TS as Tier系统

    Player->>IS: 触发中断条件<br/>（如：走太远）
    activate IS

    IS->>IS: 检测条件满足

    IS->>AS: 保存演员状态
    activate AS
    AS-->>IS: 状态已保存
    deactivate AS

    IS->>SG: 保存当前节点
    activate SG
    SG-->>IS: 节点状态已保存

    IS->>SG: 跳转到中断分支
    SG-->>IS: 跳转完成
    deactivate SG

    IS->>ES: 修改执行流
    activate ES
    ES-->>IS: 执行流已更新
    deactivate ES

    loop 中断内容执行
        IS->>IS: 检查返回条件
    end

    alt 返回条件满足
        IS->>SG: 恢复到保存节点
        IS->>AS: 恢复演员状态
        IS->>ES: 继续主线执行流
    else 永不返回
        IS->>SG: 进入失败分支
    end

    deactivate IS
```

---

### 流程3：Tier切换流程

```mermaid
sequenceDiagram
    participant ES as 执行流
    participant TS as Tier系统
    participant Input as 输入系统
    participant Camera as 摄像机系统
    participant UI as UI系统
    participant Player as 玩家实体

    ES->>TS: 动作：切换到Tier 3
    activate TS

    TS->>TS: 检查Tier堆栈
    TS->>TS: Push新Tier到堆栈
    TS->>TS: 计算激活Tier

    par 并行应用约束
        TS->>Input: 禁用移动输入
        and
        TS->>Camera: 应用视角限制<br/>（Yaw: ±60°）
        and
        TS->>UI: 隐藏战斗HUD
        and
        TS->>Player: 播放过渡动画<br/>（如：坐下）
    end

    Input-->>TS: 输入映射已更新
    Camera-->>TS: 约束已应用
    UI-->>TS: UI已调整
    Player-->>TS: 动画播放中

    TS-->>ES: Tier切换完成
    deactivate TS

    Note over ES,Player: 玩家现在处于Tier 3<br/>可以观察，但无法移动
```

---

## 🧩 第四层：数据流关系

### 核心数据流向图

```mermaid
flowchart TD
    subgraph "设计数据"
        A1[场景图定义<br/>Graph Definition]
        A2[节点配置<br/>Node Config]
        A3[资源引用<br/>Resource Refs]
    end

    subgraph "运行时数据"
        B1[当前节点<br/>Current Node]
        B2[执行时间<br/>Playback Time]
        B3[变量状态<br/>Variable State]
        B4[演员绑定<br/>Actor Bindings]
    end

    subgraph "约束数据"
        C1[激活Tier<br/>Active Tier]
        C2[输入映射<br/>Input Map]
        C3[视角限制<br/>Camera Clamp]
    end

    subgraph "中断数据"
        D1[中断条件<br/>Conditions]
        D2[保存上下文<br/>Saved Context]
        D3[返回条件<br/>Return Conditions]
    end

    A1 -->|解析| B1
    A2 -->|实例化| B4
    A3 -->|预加载| ResourceCache[资源缓存]

    B1 -->|生成事件| ExecutionQueue[执行队列]
    ExecutionQueue -->|触发| TierChange[Tier切换]
    TierChange -->|更新| C1
    C1 -->|应用| C2
    C1 -->|应用| C3

    B1 -->|监控| D1
    D1 -->|满足时| D2
    D2 -->|保存| B1
    D2 -->|保存| B3
    D2 -->|保存| B4

    style B1 fill:#ff6b6b
    style ExecutionQueue fill:#4ecdc4
    style C1 fill:#ffe66d
    style D2 fill:#ffaaa5
```

---

## 🎯 第五层：关键依赖关系

### 依赖强度矩阵

| 元素 ↓ 依赖 → | 场景图 | 执行流 | Tier | 信号 | 演员 | 中断 | 资源 |
|--------------|-------|-------|------|------|------|------|------|
| **场景图** | - | 生成 | - | 定义 | 引用 | 被监控 | 引用 |
| **执行流** | 来自 | - | 控制 | 触发 | 指令 | 被修改 | - |
| **Tier** | - | 被控制 | - | - | - | - | - |
| **信号** | 激活 | 记录 | - | - | - | - | - |
| **演员** | 被引用 | 被指令 | - | - | - | 被保存 | 需要 |
| **中断** | 监控 | 修改 | 改变 | - | 保存 | - | - |
| **资源** | 分析 | 预测 | - | - | 关联 | - | - |

**依赖强度说明**：
- 🔴 **强依赖** (生成、控制)：A必须有B才能工作
- 🟡 **中依赖** (使用、调用)：A经常使用B
- 🟢 **弱依赖** (可选、优化)：A可以不用B

---

## 🔄 第六层：生命周期关系

### 场景完整生命周期

```mermaid
stateDiagram-v2
    [*] --> 设计时

    设计时 --> 加载中: 运行时触发

    加载中 --> 准备中: 资源就绪
    note right of 加载中
        资源管理器工作
        场景图加载
    end note

    准备中 --> 就绪: 初始化完成
    note right of 准备中
        演员系统绑定
        执行流生成
    end note

    就绪 --> 执行中: 开始播放
    note right of 就绪
        Tier系统初始化
        信号系统准备
    end note

    执行中 --> 执行中: 正常流程
    执行中 --> 中断处理: 中断条件触发
    中断处理 --> 执行中: 返回主线
    中断处理 --> 完成: 永久分支

    执行中 --> 暂停: 暂停请求
    暂停 --> 执行中: 恢复

    执行中 --> 完成: 正常结束
    完成 --> 清理: 释放资源

    清理 --> [*]

    note left of 中断处理
        中断系统接管
        保存所有状态
    end note
```

---

## 💡 第七层：设计关键点

### 7大核心元素的设计要点

#### 1. 场景图 - 逻辑清晰
```
✅ 必须：
  - 支持分支和循环
  - 节点可重用
  - 可视化编辑

⚠️ 注意：
  - 避免过度复杂（建议<100节点）
  - 提供调试视图
  - 支持热重载
```

#### 2. 执行流 - 确定性
```
✅ 必须：
  - 精确到毫秒的时间控制
  - 支持回放
  - 可合并子流

⚠️ 注意：
  - 避免浮点数累积误差
  - 提供时间缩放功能
  - 优化查询性能
```

#### 3. Tier系统 - 渐进性
```
✅ 必须：
  - 至少3个层级
  - 平滑过渡
  - 有叙事理由

⚠️ 注意：
  - 避免过度限制（Tier4不超过2分钟）
  - 提供软约束而非硬墙
  - 给玩家"补偿自由"
```

#### 4. 信号系统 - 可靠性
```
✅ 必须：
  - 保证送达
  - 有序传播
  - 可追踪调试

⚠️ 注意：
  - 避免信号风暴
  - 防止循环依赖
  - 限制传播深度
```

#### 5. 演员系统 - 状态完整
```
✅ 必须：
  - 保存完整状态
  - 无缝接管/释放
  - 支持多实体同步

⚠️ 注意：
  - 处理实体已销毁情况
  - 避免状态爆炸
  - 提供状态快照
```

#### 6. 中断系统 - 鲁棒性
```
✅ 必须：
  - 覆盖主要中断场景
  - 提供返回机制
  - 保护关键状态

⚠️ 注意：
  - 避免过于敏感（误触发）
  - 给玩家反馈
  - 设计兜底方案
```

#### 7. 资源管理 - 预测性
```
✅ 必须：
  - 提前预加载
  - 异步加载
  - 内存监控

⚠️ 注意：
  - 避免加载峰值
  - 提供遮罩方案
  - 优雅降级
```

---

## 🎨 第八层：集成模式

### 模式A：最小集成

```
只实现核心三元素：

┌─────────┐
│ 场景图  │ ← 定义逻辑
└────┬────┘
     ↓
┌─────────┐
│ 执行流  │ ← 驱动时间轴
└────┬────┘
     ↓
┌─────────┐
│ Tier系统│ ← 控制自由度
└─────────┘

适用：小团队、短周期、轻叙事
```

### 模式B：标准集成

```
实现5个核心元素：

    场景图 ──→ 执行流 ──→ Tier系统
       │         │            │
       ↓         ↓            ↓
    信号系统   演员系统   游戏系统
       ↑         ↑
       └─────┬───┘
         中断系统

适用：中型团队、标准周期、中度叙事
```

### 模式C：完整集成

```
实现全部7个元素：

         ┌─ 资源管理 ─┐
         ↓             ↓
    场景图 ──→ 执行流 ──→ Tier系统
       │         │            │
       ↓         ↓            ↓
    信号系统   演员系统   游戏系统
       ↑         ↑            ↑
       └─────┬───┘            │
         中断系统 ←─────────────┘

适用：大型团队、长周期、重度叙事
```

---

## 📋 总结：关键要点

### 7大核心元素的本质

1. **场景图** = 大脑（决策中心）
2. **执行流** = 神经（传递指令）
3. **Tier系统** = 肌肉（执行动作）
4. **信号系统** = 神经递质（细胞通信）
5. **演员系统** = 器官（具体功能）
6. **中断系统** = 免疫系统（应对异常）
7. **资源管理** = 循环系统（供给营养）

### 彼此关系的本质

```
场景图定义"做什么"
  ↓
执行流决定"何时做"
  ↓
Tier系统控制"怎么做"
  ↓
信号系统传递"谁来做"
  ↓
演员系统执行"正在做"
  ↓
中断系统处理"做不了"
  ↓
资源管理保证"能做完"
```

### 设计黄金法则

```
1. 解耦原则：每个元素可独立测试
2. 单一职责：每个元素只做一件事
3. 最小依赖：减少元素间强耦合
4. 可扩展性：预留扩展点
5. 可调试性：提供完整日志
```

---

*本文档展示InteractiveScene的核心元素关系*
*建议打印并贴在工作区作为参考*
*设计时常回顾，确保理解正确*

**版本**: 1.0
**日期**: 2026-02-09
