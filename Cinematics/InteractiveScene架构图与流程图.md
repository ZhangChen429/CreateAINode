# InteractiveScene系统架构图与流程图
> 可视化技术架构 - Mermaid图表集合

---

## 📊 图表索引

1. [系统层次架构](#系统层次架构)
2. [场景实例生命周期](#场景实例生命周期)
3. [ExecutionStream执行流](#executionstream执行流)
4. [Tier切换状态机](#tier切换状态机)
5. [中断处理流程](#中断处理流程)
6. [信号传播机制](#信号传播机制)
7. [系统集成关系](#系统集成关系)
8. [类继承关系](#类继承关系)

---

## 系统层次架构

### 整体架构层次图

```mermaid
graph TB
    subgraph "应用层 Application Layer"
        Quest[Quest System]
        UI[UI System]
        Audio[Audio Director]
    end

    subgraph "场景系统层 Scene System Layer"
        SceneSystem[ISceneSystem<br/>场景系统接口]
        SceneInstance[SceneInstance<br/>场景实例]
        TierSystem[TierSystem<br/>Tier管理]

        subgraph "执行核心 Execution Core"
            ExeStream[ExecutionStream<br/>执行流]
            ActionCh[ActionChannel<br/>动作通道]
            ControlCh[ControlChannel<br/>控制通道]
            StimCh[StimulationChannel<br/>刺激通道]
        end

        subgraph "场景图 Scene Graph"
            SceneGraph[SceneResource<br/>场景资源]
            Nodes[NodeDefinitions<br/>节点定义]
            Sockets[Socket System<br/>插座系统]
        end
    end

    subgraph "仿真层 Simulation Layer"
        Physics[Physics Engine<br/>物理引擎]
        AI[AI System<br/>AI系统]
        Animation[Animation System<br/>动画系统]
        Renderer[Renderer<br/>渲染器]
    end

    subgraph "实体层 Entity Layer"
        Player[PlayerPuppet<br/>玩家]
        NPCs[NPC Entities<br/>NPC实体]
        Props[Props<br/>道具]
    end

    Quest --> SceneSystem
    UI --> SceneSystem
    Audio --> SceneSystem

    SceneSystem --> SceneInstance
    SceneSystem --> TierSystem

    SceneInstance --> ExeStream
    ExeStream --> ActionCh
    ExeStream --> ControlCh
    ExeStream --> StimCh

    SceneInstance --> SceneGraph
    SceneGraph --> Nodes
    Nodes --> Sockets

    ActionCh --> Physics
    ActionCh --> AI
    ActionCh --> Animation
    ActionCh --> Renderer

    TierSystem --> Player

    SceneInstance --> Player
    SceneInstance --> NPCs
    SceneInstance --> Props

    style SceneSystem fill:#ff6b6b
    style ExeStream fill:#4ecdc4
    style TierSystem fill:#ffe66d
```

### 数据流架构

```mermaid
graph LR
    subgraph "编辑器 Editor"
        A1[场景编辑器<br/>Scene Editor]
        A2[节点图<br/>Node Graph]
        A3[时间轴<br/>Timeline]
    end

    subgraph "编译 Compilation"
        B1[场景编译器<br/>Compiler]
        B2[优化器<br/>Optimizer]
    end

    subgraph "运行时 Runtime"
        C1[SceneResource<br/>场景资源]
        C2[SceneInstance<br/>场景实例]
        C3[ExecutionStream<br/>执行流]
        C4[ActionsExecutor<br/>动作执行器]
    end

    subgraph "效果 Effects"
        D1[实体变换<br/>Transform]
        D2[动画播放<br/>Animation]
        D3[音频触发<br/>Audio]
        D4[VFX效果<br/>VFX]
    end

    A1 --> A2
    A2 --> A3
    A3 --> B1
    B1 --> B2
    B2 --> C1
    C1 -->|Load & Instance| C2
    C2 -->|Generate| C3
    C3 -->|Execute| C4
    C4 --> D1
    C4 --> D2
    C4 --> D3
    C4 --> D4

    style C3 fill:#4ecdc4
    style C4 fill:#95e1d3
```

---

## 场景实例生命周期

### 完整生命周期状态图

```mermaid
stateDiagram-v2
    [*] --> Creating: CreateSceneInstance()

    Creating --> Preparing: 资源加载
    Preparing --> Ready: 初始化完成

    Ready --> Started: Play Request
    Ready --> Disposed: Stop Request

    Started --> Playing: 正常执行

    Playing --> Paused: Pause Request
    Playing --> Interrupted: 中断条件触发
    Playing --> Finished: 执行完成
    Playing --> Stopped: Stop Request

    Paused --> Playing: Unpause Request
    Paused --> Stopped: Stop Request

    Interrupted --> InterruptBranch: 执行中断分支
    InterruptBranch --> Returned: 返回条件满足
    InterruptBranch --> Finished: 中断分支结束

    Returned --> Playing: 恢复主线

    Finished --> Disposed: 清理资源
    Stopped --> Disposed: 清理资源

    Disposed --> [*]

    note right of Creating
        通知: instancePreparing
    end note

    note right of Ready
        通知: instanceReady
    end note

    note right of Started
        通知: instanceStarted
    end note

    note right of Interrupted
        通知: instanceInterrupted
    end note

    note right of Returned
        通知: instanceReturned
    end note

    note right of Finished
        通知: instanceFinished
    end note
```

### 场景创建序列图

```mermaid
sequenceDiagram
    participant Quest as Quest System
    participant SS as SceneSystem
    participant Loader as Resource Loader
    participant SI as SceneInstance
    participant Listener as SceneListener

    Quest->>SS: CreateSceneInstance(sceneId, params)
    activate SS

    SS->>Loader: LoadSceneResource(sceneId)
    activate Loader
    Loader-->>SS: SceneResource*
    deactivate Loader

    SS->>SI: new SceneInstance(resource, params)
    activate SI

    SI->>SI: 解析场景图
    SI->>SI: 绑定演员
    SI->>SI: 生成ExecutionStream

    SI-->>SS: SceneInstanceId
    deactivate SI

    SS->>Listener: Notify(instancePreparing)
    SS->>Listener: Notify(instanceReady)

    SS-->>Quest: SceneInstanceId
    deactivate SS

    Quest->>SS: QueueRequest(Play)
    SS->>SI: StartPlayback()
    SS->>Listener: Notify(instanceStarted)
```

---

## ExecutionStream执行流

### 三通道架构详解

```mermaid
graph TB
    subgraph "ExecutionStream 执行流"
        direction TB

        subgraph "ActionChannel 动作通道"
            A1[ActionRecord<br/>时间: 1000ms<br/>类型: SetTier<br/>目标: Player]
            A2[ActionRecord<br/>时间: 2000ms<br/>类型: LookAt<br/>目标: NPC]
            A3[ActionRecord<br/>时间: 3000ms<br/>类型: PlayAnim<br/>目标: NPC]

            A1 --> A2 --> A3
        end

        subgraph "ControlChannel 控制通道"
            C1[ControlRequest<br/>时间: 1500ms<br/>操作: EnableNode<br/>节点: Dialog_01]
            C2[ControlRequest<br/>时间: 3500ms<br/>操作: DisableNode<br/>节点: Dialog_01]

            C1 --> C2
        end

        subgraph "StimulationChannel 刺激通道"
            S1[Stimulation<br/>时间: 2500ms<br/>节点: Choice_Node<br/>插座: OnComplete]
            S2[Stimulation<br/>时间: 4000ms<br/>节点: Exit_Node<br/>插座: OnStart]

            S1 --> S2
        end
    end

    subgraph "执行器 Executors"
        ActExec[ActionsExecutor<br/>执行动作记录]
        CtrlExec[GraphController<br/>控制节点状态]
        StimExec[GraphExecutor<br/>激活节点]
    end

    A3 --> ActExec
    C2 --> CtrlExec
    S2 --> StimExec

    ActExec --> Effect1[应用游戏效果]
    CtrlExec --> Effect2[更新图状态]
    StimExec --> Effect3[触发节点逻辑]

    style A1 fill:#ff6b6b
    style C1 fill:#4ecdc4
    style S1 fill:#ffe66d
```

### 时间推进机制

```mermaid
sequenceDiagram
    participant Tick as Game Tick
    participant SI as SceneInstance
    participant ES as ExecutionStream
    participant AC as ActionChannel
    participant Exec as ActionsExecutor

    Tick->>SI: Update(deltaTime = 16ms)
    activate SI

    SI->>ES: TranslatePos(16ms)
    activate ES

    Note over ES: 当前时间: 1000ms<br/>新时间: 1016ms

    ES->>AC: GetActionsInRange(1000, 1016)
    activate AC
    AC-->>ES: ActionRecord[]
    deactivate AC

    ES-->>SI: 到期动作列表
    deactivate ES

    loop 对每个到期动作
        SI->>Exec: ExecuteAction(actionRecord)
        activate Exec
        Exec->>Exec: 应用效果到实体
        Exec-->>SI: 完成
        deactivate Exec
    end

    SI-->>Tick: 完成
    deactivate SI
```

---

## Tier切换状态机

### Tier层级关系

```mermaid
graph TD
    T0[Tier 0<br/>未定义]
    T1[Tier 1<br/>完全自由<br/>Full Gameplay]
    T2[Tier 2<br/>受限移动<br/>Staged Gameplay]
    T3[Tier 3<br/>限制移动+视角<br/>Limited Gameplay]
    T4[Tier 4<br/>极度受限<br/>FPP Cinematic]
    T5[Tier 5<br/>完全控制<br/>Cinematic]

    T0 -->|默认状态| T1
    T1 <-->|场景请求| T2
    T2 <-->|叙事深入| T3
    T3 <-->|关键剧情| T4
    T4 <-->|TPP过场| T5

    T5 -.->|跳过| T1
    T4 -.->|紧急退出| T1
    T3 -.->|中断| T1

    style T1 fill:#a8e6cf
    style T2 fill:#ffd3b6
    style T3 fill:#ffaaa5
    style T4 fill:#ff8b94
    style T5 fill:#ff6b6b
```

### Tier切换序列

```mermaid
sequenceDiagram
    participant Scene as SceneInstance
    participant Action as ActionSetSceneTier
    participant TS as TierSystem
    participant Player as PlayerPuppet
    participant Input as InputSystem
    participant Camera as CameraSystem

    Scene->>Action: 执行Tier切换动作
    activate Action

    Action->>TS: RequestTier(playerId, Tier3Data)
    activate TS

    TS->>TS: Push到Tier堆栈
    TS->>TS: 计算激活Tier (最高优先级)

    TS->>Player: SendEvent(TierChangeEvent)
    activate Player

    Player->>Input: UpdateInputContext(Tier3)
    Note over Input: 禁用移动<br/>限制武器切换

    Player->>Camera: ApplyCameraConstraints(Tier3)
    Note over Camera: 设置Yaw限制: [-60, 60]<br/>设置Pitch限制: [-30, 30]<br/>启用软衰减

    Player-->>TS: 确认
    deactivate Player

    TS-->>Action: Tier切换完成
    deactivate TS

    Action-->>Scene: 动作完成
    deactivate Action
```

### Tier堆栈管理

```mermaid
graph TB
    subgraph "Tier Stack (从底到顶)"
        direction BT
        Stack1[Tier1 - Quest请求<br/>优先级: 10]
        Stack2[Tier2 - 场景A请求<br/>优先级: 20]
        Stack3[Tier3 - 场景B请求<br/>优先级: 30]
        Stack4[Tier3 - 临时限制<br/>优先级: 40]

        Stack1 -.->|被覆盖| Stack2
        Stack2 -.->|被覆盖| Stack3
        Stack3 -.->|被覆盖| Stack4
    end

    ActiveTier[当前激活Tier<br/>Tier3 优先级40]

    Stack4 ==>|最高优先级| ActiveTier

    ClearOp[ClearTier操作]
    ClearOp -->|移除Stack4| Stack3

    NewActive[新激活Tier<br/>Tier3 优先级30]
    Stack3 ==>|成为最高| NewActive

    style Stack4 fill:#ff6b6b
    style ActiveTier fill:#4ecdc4
    style NewActive fill:#95e1d3
```

---

## 中断处理流程

### 中断检测与处理

```mermaid
flowchart TD
    Start([场景正常执行]) --> CheckInt{每帧检查<br/>中断条件}

    CheckInt -->|条件未满足| Continue[继续执行]
    Continue --> CheckInt

    CheckInt -->|条件满足| SaveState[保存当前状态<br/>Circumstance]

    SaveState --> FindBranch{查找中断分支}

    FindBranch -->|找到分支| ExecuteInt[执行中断节点]
    FindBranch -->|无分支| DefaultInt[执行默认中断]

    ExecuteInt --> CheckReturn{检查返回条件}
    DefaultInt --> CheckReturn

    CheckReturn -->|条件未满足| ExecuteInt
    CheckReturn -->|条件满足| RestoreState[恢复保存状态]
    CheckReturn -->|永不返回| Finish[结束场景]

    RestoreState --> Resume[恢复主线执行]
    Resume --> End([继续场景])
    Finish --> End2([场景结束])

    style SaveState fill:#ffe66d
    style ExecuteInt fill:#ff6b6b
    style RestoreState fill:#4ecdc4
```

### 中断条件类型决策树

```mermaid
graph TD
    Root{中断条件类型}

    Root --> Distance[距离类]
    Root --> State[状态类]
    Root --> Event[事件类]

    Distance --> DistPlayer[玩家距离实体]
    Distance --> DistNode[玩家距离节点]
    Distance --> DistSpeaker[说话者距离]

    State --> Combat[玩家战斗状态]
    State --> Distracted[角色分心]
    State --> Vehicle[载具状态]

    Event --> Trigger[触发器激活]
    Event --> Fact[Fact变化]
    Event --> Custom[自定义事件]

    DistPlayer --> Check1{距离 > 阈值?}
    Combat --> Check2{进入战斗?}
    Trigger --> Check3{触发器激活?}

    Check1 -->|是| Interrupt1[触发中断]
    Check2 -->|是| Interrupt2[触发中断]
    Check3 -->|是| Interrupt3[触发中断]

    Check1 -->|否| NoInt1[不中断]
    Check2 -->|否| NoInt2[不中断]
    Check3 -->|否| NoInt3[不中断]

    style Interrupt1 fill:#ff6b6b
    style Interrupt2 fill:#ff6b6b
    style Interrupt3 fill:#ff6b6b
```

---

## 信号传播机制

### 节点激活与信号流

```mermaid
graph LR
    subgraph "Node A 对话节点"
        A_Start([Start])
        A_Logic[执行对话逻辑]
        A_Out[OutputSocket<br/>'OnComplete']

        A_Start --> A_Logic --> A_Out
    end

    A_Out -->|发送信号| Stim[Stimulation<br/>目标: Node B<br/>插座: Input1<br/>节点点: start]

    subgraph "StimulationChannel"
        Stim --> Queue[信号队列]
    end

    Queue --> Dispatch[信号分发器]

    Dispatch -->|查找目标| B_In

    subgraph "Node B 选择节点"
        B_In[InputSocket<br/>'Input1']
        B_Start([Start])
        B_Logic[显示选择UI]
        B_Out1[OutputSocket<br/>'Choice_A']
        B_Out2[OutputSocket<br/>'Choice_B']

        B_In --> B_Start
        B_Start --> B_Logic
        B_Logic --> B_Out1
        B_Logic --> B_Out2
    end

    B_Out1 -->|玩家选择A| StimC1[信号 → Node C]
    B_Out2 -->|玩家选择B| StimC2[信号 → Node D]

    style Stim fill:#ffe66d
    style Queue fill:#4ecdc4
    style Dispatch fill:#95e1d3
```

### 系统插座触发机制

```mermaid
sequenceDiagram
    participant Node as Dialog Node
    participant Graph as Scene Graph
    participant Sys as System Sockets
    participant Exec as Executor

    Node->>Graph: 对话行播放完成
    activate Graph

    Graph->>Sys: Trigger(commandCompleted)
    activate Sys

    Note over Sys: SystemSocket ID: 1025

    Sys->>Graph: 查找连接到此插座的节点
    Graph-->>Sys: 目标节点列表

    loop 对每个目标节点
        Sys->>Exec: SendStimulation(nodeId, start)
        Exec->>Exec: 激活目标节点
    end

    deactivate Sys
    deactivate Graph

    alt 有节点请求取消
        Node->>Sys: Trigger(cancelExecution)
        Sys->>Exec: SendStimulation(nodeId, cancel)
        Exec->>Node: 取消当前节点
    end
```

---

## 系统集成关系

### 跨系统协作图

```mermaid
graph TB
    subgraph "Quest System 任务系统"
        QP[QuestPhase]
        QG[Quest Graph]
    end

    subgraph "Scene System 场景系统"
        SS[SceneSystem]
        SI[SceneInstance]
        ES[ExecutionStream]
    end

    subgraph "Tier System 层级系统"
        TS[TierSystem]
        TD[TierData]
    end

    subgraph "AI System AI系统"
        AICmd[AICommand]
        BehaviorTree[Behavior Tree]
    end

    subgraph "Audio System 音频系统"
        AudioDir[Audio Director]
        JALI[JALI System]
        DialogCtrl[Dialogue Controller]
    end

    subgraph "Animation System 动画系统"
        AnimGraph[Animation Graph]
        Workspot[Workspot System]
        IK[IK System]
    end

    subgraph "Camera System 摄像头系统"
        CamCtrl[Camera Controller]
        CamConstraints[Camera Constraints]
    end

    QP -->|创建场景| SS
    QG -->|场景资源引用| SI

    SI -->|生成| ES
    ES -->|Tier动作| TS
    ES -->|AI动作| AICmd
    ES -->|音频动作| AudioDir
    ES -->|动画动作| AnimGraph
    ES -->|摄像头动作| CamCtrl

    TS -->|限制输入| QP
    TS -->|视角约束| CamConstraints

    AICmd --> BehaviorTree
    AudioDir --> JALI
    JALI --> DialogCtrl

    AnimGraph --> Workspot
    AnimGraph --> IK

    DialogCtrl -->|同步| AnimGraph

    style SS fill:#ff6b6b
    style ES fill:#4ecdc4
    style TS fill:#ffe66d
```

### 资源依赖关系

```mermaid
graph TD
    subgraph "Quest Phase Resource"
        QPR[questPhaseResource]
        QG[questGraphDefinition]
        PP[phasePrefabs[]]
    end

    subgraph "Scene Resources"
        SR1[SceneResource A<br/>q110_02_placide.scene]
        SR2[SceneResource B<br/>q110_03_market.scene]
    end

    subgraph "Scene Dependencies"
        Prefab1[场景预制件<br/>Environment]
        Prefab2[工作点<br/>Workspots]
        Com1[社区<br/>Communities]
        Anim1[动画<br/>Animations]
    end

    subgraph "Asset Resources"
        Mesh1[网格<br/>Meshes]
        Tex1[纹理<br/>Textures]
        Audio1[音频<br/>Voice Lines]
        VFX1[特效<br/>VFX]
    end

    QPR --> QG
    QPR --> PP

    PP --> SR1
    PP --> SR2

    SR1 --> Prefab1
    SR1 --> Prefab2
    SR1 --> Com1
    SR1 --> Anim1

    Prefab1 --> Mesh1
    Prefab1 --> Tex1
    Anim1 --> Audio1
    SR1 --> VFX1

    style QPR fill:#a8e6cf
    style SR1 fill:#ffd3b6
    style Prefab1 fill:#ffaaa5
```

---

## 类继承关系

### Action类层次结构

```mermaid
classDiagram
    class ActionDefinition {
        <<abstract>>
        +GetDuration() Msec
        +SetDuration(Msec)
        +CreateSimActionInstance()
    }

    class ActionSetSceneTier {
        +Params m_params
        +Msec m_duration
    }

    class ActionUseWorkspot {
        +SceneWorkspotDataId dataId
        +Bool forceBlendIn
    }

    class ActionLookAt {
        +PerformerId target
        +Float weight
        +EasingType easing
    }

    class ActionPlayerLookAt {
        +WorldPosition targetPos
        +Float intensity
    }

    class ActionTeleportPuppet {
        +WorldPosition position
        +Quaternion rotation
    }

    class ActionVfx {
        +CName effectName
        +PerformerId attachTo
    }

    class ActionAICommand {
        +CName commandName
        +AICommandParams params
    }

    ActionDefinition <|-- ActionSetSceneTier
    ActionDefinition <|-- ActionUseWorkspot
    ActionDefinition <|-- ActionLookAt
    ActionDefinition <|-- ActionPlayerLookAt
    ActionDefinition <|-- ActionTeleportPuppet
    ActionDefinition <|-- ActionVfx
    ActionDefinition <|-- ActionAICommand
```

### Interrupt类层次结构

```mermaid
classDiagram
    class IInterruptCondition {
        <<interface>>
        +GetType() InterruptConditionType
        +CheckCondition(context) Bool
        +Initialize(context)
        +Deinitialize(context)
    }

    class InterruptConditionDistance {
        <<abstract>>
        #Float m_distance
    }

    class InterruptConditionDistancePlayerEntity {
        +ent::EntityID m_entity
    }

    class InterruptConditionDistancePlayerNode {
        +NodeRef m_nodeRef
    }

    class InterruptConditionPlayerCombat {
        +Bool m_triggerOnEnter
    }

    class InterruptConditionFact {
        +CName m_factName
        +Int32 m_factValue
        +InterruptFactConditionType m_type
    }

    class InterruptConditionTrigger {
        +NodeRef m_triggerRef
    }

    IInterruptCondition <|.. InterruptConditionDistance
    InterruptConditionDistance <|-- InterruptConditionDistancePlayerEntity
    InterruptConditionDistance <|-- InterruptConditionDistancePlayerNode

    IInterruptCondition <|.. InterruptConditionPlayerCombat
    IInterruptCondition <|.. InterruptConditionFact
    IInterruptCondition <|.. InterruptConditionTrigger
```

### TierData类层次结构

```mermaid
classDiagram
    class SceneTierData {
        <<abstract>>
        +GameplayTier tier
        +Bool emptyHands
    }

    class SceneTier1Data {
    }

    class SceneTier2Data {
        +Tier2WalkType walkType
    }

    class SceneTierDataMotionConstrained {
        <<abstract>>
        +Bool usePlayerWorkspot
    }

    class SceneTier3Data {
        +Tier3CameraSettings cameraSettings
        +Float yawLeftLimit
        +Float yawRightLimit
        +Float pitchTopLimit
        +Float pitchBottomLimit
    }

    class SceneTier4Data {
        +NodeRef splineRef
        +Bool lockToSpline
    }

    class SceneTier5Data {
        +NodeRef splineRef
        +Bool overrideFOV
        +Float customFOV
    }

    SceneTierData <|-- SceneTier1Data
    SceneTierData <|-- SceneTier2Data
    SceneTierData <|-- SceneTierDataMotionConstrained

    SceneTierDataMotionConstrained <|-- SceneTier3Data
    SceneTierDataMotionConstrained <|-- SceneTier4Data
    SceneTierDataMotionConstrained <|-- SceneTier5Data
```

---

## 数据结构详解

### ExecutionStream内部结构

```mermaid
classDiagram
    class ExecutionStream {
        -ActionChannel m_actionChannel
        -ControlChannel m_controlChannel
        -StimulationChannel m_stimulationChannel
        +GetActionChannel() ActionChannel&
        +GetControlChannel() ControlChannel&
        +GetStimulationChannel() StimulationChannel&
        +IsEmpty() Bool
        +Clear()
        +Reserve(Uint32, Uint32, Uint32)
        +CombineSubstream(ExecutionStream&, Uint32)
        +TranslatePos(Msec)
        +TranslateNeg(Msec)
        +IsIndexed() Bool
        +Reindex(Bool)
    }

    class ActionChannel {
        -DynArray~ActionRecord~ m_records
        -Index m_index
        +AddRecord(ActionRecord&)
        +GetActionsAtTime(SceneTime) ActionRecord[]
        +IsIndexed() Bool
    }

    class ControlChannel {
        -DynArray~ControlRequest~ m_requests
        -Index m_index
        +AddRequest(ControlRequest&)
        +GetRequestsAtTime(SceneTime) ControlRequest[]
    }

    class StimulationChannel {
        -DynArray~Stimulation~ m_stimulations
        -Index m_index
        +AddStimulation(Stimulation&)
        +GetStimulationsAtTime(SceneTime) Stimulation[]
    }

    class ActionRecord {
        +ActionId actionId
        +SceneTime startTime
        +Msec duration
        +PerformerId performer
    }

    class ControlRequest {
        +NodeId targetNode
        +ControlOp operation
        +SceneTime timestamp
    }

    class Stimulation {
        +NodeId m_nodeId
        +CName m_nodepoint
        +InputSocketStamp m_isockStamp
    }

    ExecutionStream *-- ActionChannel
    ExecutionStream *-- ControlChannel
    ExecutionStream *-- StimulationChannel

    ActionChannel o-- ActionRecord
    ControlChannel o-- ControlRequest
    StimulationChannel o-- Stimulation
```

### SceneGraph节点结构

```mermaid
classDiagram
    class SceneGraph {
        -DynArray~NodeDefinition~ nodes
        -DynArray~Connection~ connections
        -HashMap~NodeId,Node*~ nodeMap
        +GetNode(NodeId) Node*
        +TraverseFromNode(NodeId)
    }

    class NodeDefinition {
        <<abstract>>
        +NodeId id
        +CName nodeName
        +InputSocket[] inputSockets
        +OutputSocket[] outputSockets
    }

    class InputSocket {
        +InputSocketStamp stamp
        +SocketName name
    }

    class OutputSocket {
        +OutputSocketStamp stamp
        +SocketName name
        +InputSocketId[] targets
    }

    class Connection {
        +OutputSocketId source
        +InputSocketId target
    }

    class DialogLineNode {
        +LocalizedString dialogLine
        +PerformerId speaker
        +InterruptSettings interruptSettings
    }

    class ChoiceNode {
        +Choice[] choices
        +Float timeout
    }

    class SectionNode {
        +CName sectionName
        +NodeId entryPoint
    }

    SceneGraph *-- NodeDefinition
    SceneGraph *-- Connection

    NodeDefinition *-- InputSocket
    NodeDefinition *-- OutputSocket

    NodeDefinition <|-- DialogLineNode
    NodeDefinition <|-- ChoiceNode
    NodeDefinition <|-- SectionNode
```

---

## 性能分析流程图

### 帧预算分配

```mermaid
pie title CPU帧预算分配 (16.67ms @ 60FPS)
    "场景逻辑更新" : 2.0
    "ExecutionStream处理" : 1.5
    "动作执行" : 3.0
    "物理模拟" : 4.0
    "AI更新" : 2.5
    "渲染准备" : 3.67
```

### 场景系统性能优化决策树

```mermaid
flowchart TD
    Start{帧率下降?}

    Start -->|是| CheckProf[启用Profiler]
    Start -->|否| End1([保持监控])

    CheckProf --> Hotspot{定位热点}

    Hotspot -->|ExecutionStream| OptStream[优化Stream]
    Hotspot -->|中断检查| OptInterrupt[优化中断]
    Hotspot -->|动作执行| OptActions[优化动作]
    Hotspot -->|其他| OptOther[其他优化]

    OptStream --> IndexCheck{已索引?}
    IndexCheck -->|否| DoIndex[强制索引]
    IndexCheck -->|是| PreAlloc[预分配容量]

    OptInterrupt --> ReduceFreq[降低检查频率]
    ReduceFreq --> Cache[缓存结果]

    OptActions --> Batch[批处理动作]
    Batch --> Prioritize[优先级排序]

    DoIndex --> Measure1{测量改善}
    PreAlloc --> Measure1
    Cache --> Measure1
    Prioritize --> Measure1
    OptOther --> Measure1

    Measure1 -->|显著改善| End2([完成优化])
    Measure1 -->|改善不足| Deeper[更深层分析]

    Deeper --> End3([联系引擎团队])

    style CheckProf fill:#ffe66d
    style DoIndex fill:#4ecdc4
    style End2 fill:#a8e6cf
```

---

## 总结

这些架构图和流程图展示了InteractiveScene系统的：

1. **层次化设计** - 清晰的职责分离
2. **数据流** - 从编辑器到运行时的完整路径
3. **状态管理** - 场景生命周期与Tier切换
4. **信号机制** - 节点间的通信协议
5. **系统集成** - 与其他游戏系统的协作关系
6. **类层次** - 扩展点与继承结构

使用这些图表可以：
- 快速理解系统架构
- 定位特定功能的实现位置
- 设计新功能的集成方案
- 调试问题时追踪数据流

---

*本文档使用Mermaid语法绘制*
*可在支持Mermaid的Markdown查看器中渲染*
*推荐工具: Typora, VS Code (Markdown Preview Enhanced), GitHub*
