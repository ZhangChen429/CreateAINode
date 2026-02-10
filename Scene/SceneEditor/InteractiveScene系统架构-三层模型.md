# InteractiveScene 系统架构 - 三层设计模型

> 从程序设计角度构建的抽象架构模型

---

## 📐 架构总览

InteractiveScene 系统采用**三层分离架构**，从抽象到具体逐层细化：

```
┌─────────────────────────────────────────┐
│   第一层：概念模型层 (Conceptual)        │  ← 做什么 (WHAT)
│   定义核心概念、职责边界、交互协议        │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│   第二层：逻辑架构层 (Architectural)     │  ← 怎么做 (HOW)
│   组件接口、数据流、生命周期、扩展点      │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│   第三层：实现策略层 (Implementation)    │  ← 落地细节 (DETAIL)
│   设计模式、解决方案、优化策略、集成指南  │
└─────────────────────────────────────────┘
```

---

# 第一层：概念模型层 (Conceptual Model Layer)

> **目标**：定义"是什么"，建立统一的心智模型

## 1.1 核心概念定义

### 概念A：叙事容器 (Narrative Container)
```
定义：封装交互式叙事的最小可组合单元

核心属性：
├─ 独立性：可以单独加载、执行、卸载
├─ 可组合：支持嵌套、串联、并行
└─ 确定性：给定输入产生可预测的输出

职责边界：
✓ 管理叙事逻辑的组织结构
✓ 定义参与者和资源的引用关系
✗ 不负责具体渲染和物理模拟
✗ 不直接操作游戏实体
```

### 概念B：时间控制器 (Temporal Controller)
```
定义：将离散事件映射到连续时间轴的调度器

核心属性：
├─ 精确性：时间精度可配置（通常毫秒级）
├─ 可控性：支持播放、暂停、快进、回退
└─ 并发性：支持多轨道并行执行

职责边界：
✓ 管理事件的时序关系
✓ 协调异步操作的同步点
✗ 不负责事件的语义解释
✗ 不处理业务逻辑
```

### 概念C：自由度调节器 (Freedom Regulator)
```
定义：动态控制玩家操作空间的约束管理器

核心属性：
├─ 渐进性：约束级别连续可调
├─ 可逆性：约束可以被添加和移除
└─ 透明性：约束变化对玩家可感知

职责边界：
✓ 管理输入映射和摄像机约束
✓ 定义不同约束级别的切换规则
✗ 不直接读取硬件输入
✗ 不执行具体的摄像机运算
```

### 概念D：通信协议 (Communication Protocol)
```
定义：组件间消息传递的标准化机制

核心属性：
├─ 解耦性：发送者和接收者互不感知
├─ 可靠性：消息保证送达或失败通知
└─ 可追溯：消息路径可记录和回放

职责边界：
✓ 定义消息格式和路由规则
✓ 管理订阅和发布关系
✗ 不包含具体的业务数据
✗ 不负责消息内容的解析
```

### 概念E：状态托管器 (State Custodian)
```
定义：代理管理外部实体状态的中介层

核心属性：
├─ 非侵入：不修改原始实体结构
├─ 可恢复：支持状态快照和回滚
└─ 局部性：只影响注册的实体

职责边界：
✓ 保存和恢复实体状态
✓ 临时接管实体控制权
✗ 不拥有实体的生命周期
✗ 不处理实体的创建和销毁
```

### 概念F：异常处理器 (Exception Handler)
```
定义：处理叙事流程偏离预期路径的容错机制

核心属性：
├─ 预测性：提前定义可能的偏离场景
├─ 自动化：满足条件自动触发
└─ 可返回：支持返回原流程或永久分支

职责边界：
✓ 监控触发条件
✓ 保存和恢复执行上下文
✗ 不负责条件的具体判定逻辑
✗ 不处理致命错误（Crash）
```

### 概念G：资源预测器 (Resource Predictor)
```
定义：基于执行流预测资源需求的智能加载器

核心属性：
├─ 前瞻性：提前分析未来N秒的需求
├─ 优先级：根据重要性排序加载
└─ 自适应：根据硬件能力调整策略

职责边界：
✓ 分析资源依赖关系
✓ 调度异步加载任务
✗ 不负责资源的具体解码
✗ 不处理资源文件的查找
```

---

## 1.2 职责边界矩阵

| 概念 | 核心职责 | 不负责 | 依赖关系 |
|------|---------|--------|---------|
| 叙事容器 | 组织结构 | 执行细节 | 被所有组件引用 |
| 时间控制器 | 时序调度 | 业务逻辑 | 依赖叙事容器 |
| 自由度调节器 | 约束管理 | 输入处理 | 被时间控制器调用 |
| 通信协议 | 消息路由 | 数据解析 | 被容器内部使用 |
| 状态托管器 | 状态代理 | 实体创建 | 依赖通信协议 |
| 异常处理器 | 流程分支 | 条件判定 | 依赖所有组件 |
| 资源预测器 | 加载调度 | 资源解码 | 依赖叙事容器 |

---

## 1.3 交互协议定义

### 协议类型一：命令模式 (Command Pattern)
```
用途：组件向外部系统发送操作指令

消息流向：
InteractiveScene → 游戏系统

消息结构：
{
  type: "command",
  target: <目标系统标识>,
  operation: <操作名称>,
  parameters: <操作参数>,
  priority: <优先级>,
  timestamp: <时间戳>
}

保证：
- 幂等性：重复发送不产生副作用
- 可撤销：支持Undo操作
```

### 协议类型二：事件模式 (Event Pattern)
```
用途：外部系统向 InteractiveScene 通知状态变化

消息流向：
游戏系统 → InteractiveScene

消息结构：
{
  type: "event",
  source: <事件源标识>,
  eventName: <事件名称>,
  payload: <事件数据>,
  timestamp: <时间戳>
}

保证：
- 异步性：不阻塞发送者
- 顺序性：同源事件按发送顺序处理
```

### 协议类型三：查询模式 (Query Pattern)
```
用途：组件间的状态查询

消息流向：
双向

消息结构：
Request: {
  type: "query",
  target: <目标组件>,
  queryType: <查询类型>,
  parameters: <查询参数>
}

Response: {
  success: <是否成功>,
  data: <查询结果>,
  error: <错误信息>
}

保证：
- 同步性：立即返回结果或超时
- 不可变性：查询不改变状态
```

---

## 1.4 设计原则

### 原则1：单一职责原则 (Single Responsibility)
```
每个概念只解决一类问题：
- 叙事容器 → 只管结构
- 时间控制器 → 只管时序
- 自由度调节器 → 只管约束

反例：
❌ 让叙事容器直接管理资源加载
❌ 让时间控制器处理输入事件
```

### 原则2：依赖倒置原则 (Dependency Inversion)
```
高层模块不依赖低层模块，都依赖抽象：
- 叙事容器不直接依赖具体游戏引擎
- 使用接口/协议进行通信
- 具体实现通过注入提供

实现方式：
interface IGameSystemBridge {
  SendCommand(command)
  QueryState(query)
  RegisterEventHandler(handler)
}
```

### 原则3：开放封闭原则 (Open-Closed)
```
对扩展开放，对修改封闭：
- 新增节点类型不修改核心系统
- 新增动作类型通过注册机制
- 新增约束类型通过配置文件

扩展点：
- 节点类型注册表
- 动作工厂
- 条件评估器插件
```

### 原则4：接口隔离原则 (Interface Segregation)
```
不强迫组件依赖不需要的接口：
- 只读组件只暴露查询接口
- 不同游戏系统使用不同的Adapter
- 按需组合接口

示例：
interface IReadableScene {
  GetCurrentNode()
  GetVariable(name)
}

interface IControllableScene extends IReadableScene {
  SetVariable(name, value)
  Jump(nodeId)
}
```

### 原则5：最小惊讶原则 (Least Astonishment)
```
系统行为符合直觉预期：
- 约束级别数字越大 = 限制越多
- 时间流逝是线性的（除非明确暂停）
- 中断后默认可以返回

反例：
❌ Tier1是最严格的约束（违反直觉）
❌ 跳转节点后不重置时间（惊讶）
```

---

# 第二层：逻辑架构层 (Architectural Design Layer)

> **目标**：定义"如何组织"，建立可实现的架构

## 2.1 组件接口设计

### 接口组A：核心容器接口

```
// 场景容器接口
interface ISceneContainer {
  // 生命周期管理
  Initialize(config: SceneConfig): Result
  Activate(): void
  Deactivate(): void
  Dispose(): void

  // 运行时控制
  Start(): void
  Pause(): void
  Resume(): void
  Stop(): void

  // 状态查询
  GetState(): SceneState
  GetCurrentNode(): NodeReference
  GetVariable(name: string): Variant
  SetVariable(name: string, value: Variant): void

  // 导航控制
  Jump(targetNodeId: NodeId): void
  Evaluate(condition: Expression): bool

  // 事件监听
  OnNodeEnter: Event<NodeId>
  OnNodeExit: Event<NodeId>
  OnVariableChanged: Event<string, Variant>
  OnCompleted: Event<CompletionReason>
}
```

```
// 时间控制接口
interface ITemporalController {
  // 播放控制
  Play(): void
  Pause(): void
  Seek(time: float): void
  SetSpeed(multiplier: float): void

  // 时间查询
  GetCurrentTime(): float
  GetDuration(): float
  IsPlaying(): bool

  // 事件调度
  ScheduleAction(action: Action, time: float): ScheduleHandle
  CancelAction(handle: ScheduleHandle): void

  // 同步点管理
  WaitFor(condition: Func<bool>): Promise
  Synchronize(handles: ScheduleHandle[]): void

  // 事件回调
  OnTimeUpdate: Event<float>
  OnActionTriggered: Event<Action>
}
```

```
// 自由度调节接口
interface IFreedomRegulator {
  // 层级管理
  PushTier(tier: TierDefinition): void
  PopTier(): void
  GetActiveTier(): TierDefinition

  // 约束应用
  ApplyConstraints(tier: TierDefinition): void
  RemoveConstraints(tier: TierDefinition): void

  // 查询接口
  IsInputAllowed(inputAction: string): bool
  GetCameraConstraints(): CameraConstraints
  GetMovementConstraints(): MovementConstraints

  // 事件
  OnTierChanged: Event<TierDefinition, TierDefinition>
}
```

### 接口组B：通信与状态接口

```
// 信号系统接口
interface ISignalSystem {
  // 信号发送
  Emit(signal: Signal): void
  EmitTo(target: NodeId, signal: Signal): void
  Broadcast(signal: Signal): void

  // 信号订阅
  Subscribe(signalType: string, handler: SignalHandler): SubscriptionHandle
  Unsubscribe(handle: SubscriptionHandle): void

  // 信号查询
  GetPendingSignals(): Signal[]
  HasPendingSignal(type: string): bool

  // 调试支持
  EnableTracing(filter: string): void
  GetSignalHistory(): Signal[]
}
```

```
// 状态托管接口
interface IStateCustodian {
  // 实体注册
  RegisterActor(actorId: ActorId, entity: IGameEntity): void
  UnregisterActor(actorId: ActorId): void

  // 状态快照
  CaptureState(actorId: ActorId): StateSnapshot
  RestoreState(actorId: ActorId, snapshot: StateSnapshot): void

  // 控制接管
  TakeControl(actorId: ActorId, controllerId: string): void
  ReleaseControl(actorId: ActorId): void

  // 查询
  IsRegistered(actorId: ActorId): bool
  GetControlOwner(actorId: ActorId): string
}
```

### 接口组C：扩展与资源接口

```
// 异常处理接口
interface IExceptionHandler {
  // 中断注册
  RegisterInterrupt(interrupt: InterruptDefinition): InterruptHandle
  UnregisterInterrupt(handle: InterruptHandle): void

  // 条件监控
  EvaluateConditions(): void
  IsInterruptActive(): bool

  // 上下文管理
  SaveContext(): ExecutionContext
  RestoreContext(context: ExecutionContext): void

  // 事件
  OnInterruptTriggered: Event<InterruptDefinition>
  OnInterruptResolved: Event<ResolutionType>
}
```

```
// 资源预测接口
interface IResourcePredictor {
  // 资源分析
  AnalyzeDependencies(scene: ISceneContainer): ResourceList
  PredictUpcoming(lookaheadTime: float): ResourceList

  // 加载控制
  RequestPreload(resources: ResourceList, priority: Priority): void
  CancelPreload(resources: ResourceList): void

  // 查询
  IsLoaded(resourceId: ResourceId): bool
  GetLoadProgress(): float

  // 内存管理
  GetMemoryUsage(): MemoryStats
  ReleaseUnused(): void
}
```

---

## 2.2 数据流设计

### 数据流模式一：Pipeline（管道）
```
用途：场景启动流程

数据流向：
原始场景定义
  → 解析器 (Parser)
  → 验证器 (Validator)
  → 优化器 (Optimizer)
  → 运行时容器 (Runtime Container)

每个阶段：
- 输入：上一阶段的输出
- 输出：标准化数据结构
- 错误处理：链式传播或中断

代码结构：
class ScenePipeline {
  stages: IPipelineStage[]

  Execute(input: SceneAsset): Result<ISceneContainer> {
    data = input
    for stage in stages {
      result = stage.Process(data)
      if result.IsError() {
        return result
      }
      data = result.Value()
    }
    return Success(data)
  }
}
```

### 数据流模式二：Event Bus（事件总线）
```
用途：组件间松耦合通信

拓扑结构：
           EventBus (中心)
              ↓   ↑
    ┌─────────┼───┼─────────┐
    ↓         ↓   ↑         ↑
时间控制器  状态托管  外部系统

消息类型：
- 时间事件：TimeUpdated, ActionTriggered
- 状态事件：VariableChanged, NodeEntered
- 系统事件：TierChanged, ResourceLoaded

订阅规则：
- 按类型过滤
- 支持通配符
- 可设置优先级

代码结构：
class EventBus {
  subscribers: Map<EventType, List<Handler>>

  Publish(event: Event) {
    handlers = subscribers[event.Type]
    for handler in handlers.SortedByPriority() {
      handler.Handle(event)
    }
  }

  Subscribe(type: EventType, handler: Handler, priority: int) {
    subscribers[type].Add(handler, priority)
  }
}
```

### 数据流模式三：Reactive Stream（响应式流）
```
用途：时间驱动的连续数据流

流式处理：
时间源 (TimeSource)
  → 过滤器 (Filter: 只处理激活的)
  → 映射器 (Map: 转换为动作)
  → 执行器 (Executor: 应用到系统)

特性：
- 背压支持：下游处理慢时暂停上游
- 错误恢复：单个事件失败不中断流
- 可组合：多个流可以合并、分支

代码结构：
class TimeStream {
  operators: IStreamOperator[]

  Flow() {
    while (isActive) {
      event = timeSource.Next()

      for op in operators {
        event = op.Transform(event)
        if event == null { break }
      }

      if event != null {
        executor.Execute(event)
      }
    }
  }
}
```

---

## 2.3 生命周期管理

### 阶段划分

```
[未初始化] → Initialize() → [已初始化]
                ↓
            Activate() → [激活状态]
                ↓
             Start() → [运行中]
                ↓
  ┌─────── Pause() → [暂停]
  │          ↓
  │       Resume() ──┘
  │
  └──────→ Stop() → [已停止]
                ↓
          Deactivate() → [已去激活]
                ↓
            Dispose() → [已释放]
```

### 状态转换约束

```
合法转换：
✓ [已初始化] → Activate() → [激活状态]
✓ [运行中] → Pause() → [暂停]
✓ [暂停] → Resume() → [运行中]
✓ [运行中] → Stop() → [已停止]

非法转换：
✗ [未初始化] → Start()  // 必须先初始化
✗ [已释放] → Activate()  // 无法复活
✗ [暂停] → Stop()  // 必须先Resume再Stop

边界条件：
- Initialize() 可重复调用（幂等）
- Dispose() 后对象不可再用
- Stop() 后可再次Start()（重播）
```

### 资源管理策略

```
阶段 → 资源操作

[Initialize]
  - 分配内存缓冲区
  - 创建线程池
  - 注册全局回调

[Activate]
  - 加载场景定义
  - 绑定演员实体
  - 预加载关键资源

[Start]
  - 启动时间控制器
  - 应用初始Tier
  - 触发开始事件

[Pause]
  - 暂停时间流
  - 保存当前状态（可选）

[Stop]
  - 停止所有调度任务
  - 重置变量状态

[Deactivate]
  - 解绑演员实体
  - 卸载非共享资源
  - 恢复游戏系统状态

[Dispose]
  - 释放所有内存
  - 注销全局回调
  - 清空缓存
```

---

## 2.4 扩展点设计

### 扩展点1：节点类型注册

```
机制：工厂模式 + 注册表

接口定义：
interface INodeType {
  GetTypeName(): string
  CreateInstance(config: NodeConfig): INode
  GetInputSockets(): SocketDefinition[]
  GetOutputSockets(): SocketDefinition[]
}

注册流程：
NodeTypeRegistry.Register("CustomDialogue", new CustomDialogueType())

运行时使用：
nodeType = NodeTypeRegistry.Get(config.TypeName)
node = nodeType.CreateInstance(config)

优势：
- 无需修改核心代码
- 支持插件化
- 可动态加载
```

### 扩展点2：动作类型注册

```
机制：命令模式 + 策略模式

接口定义：
interface IActionExecutor {
  GetActionType(): string
  CanExecute(context: ExecutionContext): bool
  Execute(params: ActionParams, context: ExecutionContext): void
  Undo(context: ExecutionContext): void  // 可选
}

注册流程：
ActionRegistry.Register("CustomEffect", new CustomEffectExecutor())

运行时使用：
executor = ActionRegistry.Get(action.Type)
if executor.CanExecute(context) {
  executor.Execute(action.Params, context)
}

优势：
- 动作逻辑可复用
- 支持撤销操作
- 易于测试
```

### 扩展点3：条件评估器插件

```
机制：表达式解释器 + 插件架构

接口定义：
interface IConditionEvaluator {
  GetConditionType(): string
  Evaluate(condition: Condition, context: EvaluationContext): bool
}

注册流程：
ConditionRegistry.Register("proximity", new ProximityEvaluator())

运行时使用：
evaluator = ConditionRegistry.Get(condition.Type)
result = evaluator.Evaluate(condition, context)

扩展示例：
class ProximityEvaluator : IConditionEvaluator {
  Evaluate(condition, context) {
    distance = context.GetDistance(condition.actorA, condition.actorB)
    return distance < condition.threshold
  }
}

优势：
- 无需硬编码条件类型
- 支持领域特定语言（DSL）
- 可视化编辑器友好
```

### 扩展点4：游戏系统适配器

```
机制：适配器模式 + 依赖注入

接口定义：
interface IGameSystemAdapter {
  GetSystemName(): string
  SendCommand(command: Command): void
  QueryState(query: Query): Variant
  RegisterEventSource(eventType: string, source: IEventSource): void
}

注入流程：
sceneContainer.InjectAdapter("Input", new UnityInputAdapter())
sceneContainer.InjectAdapter("Camera", new UnrealCameraAdapter())

运行时使用：
adapter = adapters["Input"]
adapter.SendCommand(new DisableMovementCommand())

不同引擎实现：
class UnityInputAdapter : IGameSystemAdapter {
  SendCommand(cmd) {
    if cmd is DisableMovementCommand {
      PlayerInput.enabled = false
    }
  }
}

class UnrealCameraAdapter : IGameSystemAdapter {
  SendCommand(cmd) {
    if cmd is SetCameraConstraintCommand {
      CameraManager.SetViewTargetWithBlend(cmd.target)
    }
  }
}

优势：
- 引擎无关
- 易于移植
- 可模拟测试
```

---

# 第三层：实现策略层 (Implementation Strategy Layer)

> **目标**：定义"落地方案"，提供可操作的指南

## 3.1 设计模式应用

### 模式1：状态机模式 (State Machine)

```
应用场景：管理场景生命周期

实现策略：
┌──────────────┐
│ SceneContext │ ← 持有当前状态
└──────────────┘
       ↓ 委托
┌──────────────┐
│   IState     │ ← 状态接口
└──────────────┘
       ↑
       ├── InitializedState
       ├── ActiveState
       ├── RunningState
       ├── PausedState
       └── StoppedState

代码模板：
interface IState {
  OnEnter(context: SceneContext): void
  OnExit(context: SceneContext): void
  Start(): IState?
  Pause(): IState?
  Resume(): IState?
  Stop(): IState?
}

class RunningState : IState {
  Pause() {
    return new PausedState()
  }

  Stop() {
    context.StopTimeline()
    return new StoppedState()
  }

  // 其他操作返回 null（表示不允许）
}

优势：
- 状态转换逻辑清晰
- 避免if-else嵌套
- 易于扩展新状态

陷阱：
⚠ 避免状态爆炸（>10个状态需重新设计）
⚠ 避免状态间共享可变数据
```

### 模式2：观察者模式 (Observer)

```
应用场景：事件通知机制

实现策略：
┌─────────┐
│ Subject │ ← 被观察对象（如SceneContainer）
└─────────┘
     │ 通知
     ↓
┌──────────┐
│ Observer │ ← 观察者接口
└──────────┘
     ↑
     ├── UIUpdater
     ├── AudioTrigger
     └── AnalyticsLogger

代码模板：
interface IObserver<T> {
  OnNotify(event: T): void
}

class Subject<T> {
  observers: List<IObserver<T>>

  Attach(observer: IObserver<T>) {
    observers.Add(observer)
  }

  Notify(event: T) {
    foreach (observer in observers) {
      observer.OnNotify(event)
    }
  }
}

变体：事件聚合器（Event Aggregator）
class EventAggregator {
  handlers: Map<Type, List<Handler>>

  Subscribe<T>(handler: Action<T>) {
    handlers[typeof(T)].Add(handler)
  }

  Publish<T>(event: T) {
    foreach (handler in handlers[typeof(T)]) {
      handler(event)
    }
  }
}

优势：
- 解耦发送者和接收者
- 支持一对多通知
- 动态添加/移除观察者

陷阱：
⚠ 避免观察者内部再触发事件（循环通知）
⚠ 记得在Dispose时取消订阅（内存泄漏）
```

### 模式3：命令模式 (Command)

```
应用场景：动作执行和撤销

实现策略：
┌─────────┐
│ Command │ ← 命令接口
└─────────┘
     ↑
     ├── PlayAnimationCommand
     ├── SetTierCommand
     └── JumpNodeCommand

每个命令封装：
- 接收者（Receiver）
- 参数（Parameters）
- 执行逻辑（Execute）
- 撤销逻辑（Undo）

代码模板：
interface ICommand {
  Execute(): void
  Undo(): void
  CanUndo(): bool
}

class SetVariableCommand : ICommand {
  container: ISceneContainer
  varName: string
  newValue: Variant
  oldValue: Variant  // 保存用于Undo

  Execute() {
    oldValue = container.GetVariable(varName)
    container.SetVariable(varName, newValue)
  }

  Undo() {
    container.SetVariable(varName, oldValue)
  }
}

命令队列：
class CommandQueue {
  history: Stack<ICommand>

  Execute(cmd: ICommand) {
    cmd.Execute()
    if cmd.CanUndo() {
      history.Push(cmd)
    }
  }

  Undo() {
    if history.Count > 0 {
      cmd = history.Pop()
      cmd.Undo()
    }
  }
}

优势：
- 动作可序列化（用于网络同步）
- 支持撤销/重做
- 易于记录和回放

陷阱：
⚠ 注意命令的内存占用（历史记录）
⚠ 并发执行需考虑顺序依赖
```

### 模式4：策略模式 (Strategy)

```
应用场景：Tier约束应用策略

实现策略：
┌──────────────────┐
│ ConstraintPolicy │ ← 策略接口
└──────────────────┘
       ↑
       ├── InputConstraintPolicy
       ├── CameraConstraintPolicy
       └── MovementConstraintPolicy

代码模板：
interface IConstraintPolicy {
  Apply(tier: TierDefinition, context: GameContext): void
  Remove(context: GameContext): void
}

class InputConstraintPolicy : IConstraintPolicy {
  Apply(tier, context) {
    allowedActions = tier.GetAllowedInputs()
    context.InputSystem.SetAllowedActions(allowedActions)
  }

  Remove(context) {
    context.InputSystem.RestoreDefaultActions()
  }
}

策略组合：
class TierManager {
  policies: List<IConstraintPolicy>

  ApplyTier(tier: TierDefinition) {
    foreach (policy in policies) {
      policy.Apply(tier, gameContext)
    }
  }
}

优势：
- 约束逻辑可插拔
- 易于A/B测试不同策略
- 运行时切换策略

陷阱：
⚠ 策略间可能有依赖（需定义应用顺序）
⚠ 避免策略过多（>5个考虑重新抽象）
```

### 模式5：对象池模式 (Object Pool)

```
应用场景：信号和事件对象复用

实现策略：
┌────────────┐
│ ObjectPool │
└────────────┘
  ↓ 获取     ↑ 归还
┌────────────┐
│   Object   │

代码模板：
class ObjectPool<T> where T : new() {
  available: Queue<T>
  inUse: Set<T>
  factory: Func<T>
  reset: Action<T>

  Acquire(): T {
    if available.Count > 0 {
      obj = available.Dequeue()
    } else {
      obj = factory()
    }
    inUse.Add(obj)
    return obj
  }

  Release(obj: T) {
    if inUse.Contains(obj) {
      reset(obj)  // 重置状态
      inUse.Remove(obj)
      available.Enqueue(obj)
    }
  }
}

使用示例：
signalPool = new ObjectPool<Signal>(
  factory: () => new Signal(),
  reset: (s) => s.Clear()
)

signal = signalPool.Acquire()
signal.Type = "node_entered"
// ... 使用signal
signalPool.Release(signal)

优势：
- 减少GC压力
- 提高性能（避免频繁分配）
- 预测内存占用

陷阱：
⚠ 必须正确Reset对象状态
⚠ 注意多线程安全（加锁或无锁队列）
⚠ 池大小需根据实际负载调整
```

---

## 3.2 常见问题解决方案

### 问题1：时间漂移 (Time Drift)

```
现象：
执行流时间与实际游戏时间不同步，导致动作延迟或提前

原因：
- 帧率波动导致累积误差
- 暂停/恢复时时间计算错误
- 异步操作完成时间不可预测

解决方案：
class TimeController {
  realTime: float  // 真实游戏时间
  virtualTime: float  // 场景虚拟时间
  timeScale: float  // 时间缩放

  Update(deltaTime: float) {
    realTime += deltaTime
    virtualTime += deltaTime * timeScale

    // 关键：使用虚拟时间触发事件
    ProcessEvents(virtualTime)
  }

  Pause() {
    timeScale = 0
  }

  Resume() {
    timeScale = 1
    // 关键：不补偿暂停期间的时间
  }

  Seek(targetTime: float) {
    virtualTime = targetTime
    // 关键：跳过中间事件或快速执行
  }
}

最佳实践：
✓ 使用虚拟时间作为基准
✓ 暂停时不累积时间
✓ Seek时提供"跳过"或"快速播放"选项
✗ 不要用DateTime.Now计算差值（受系统时间影响）
```

### 问题2：信号循环 (Signal Loop)

```
现象：
节点A发送信号激活节点B，节点B又激活节点A，形成无限循环

原因：
- 设计错误：逻辑环路
- 条件判断失效
- 状态未正确更新

解决方案：
class SignalDispatcher {
  propagationDepth: int = 0
  maxDepth: int = 100
  visitedNodes: Set<NodeId>

  Dispatch(signal: Signal) {
    propagationDepth++

    if propagationDepth > maxDepth {
      throw SignalLoopException("信号传播深度超过限制")
    }

    if visitedNodes.Contains(signal.TargetNodeId) {
      LogWarning("检测到潜在循环，跳过节点: " + signal.TargetNodeId)
      propagationDepth--
      return
    }

    visitedNodes.Add(signal.TargetNodeId)
    targetNode.OnSignalReceived(signal)
    visitedNodes.Remove(signal.TargetNodeId)

    propagationDepth--
  }
}

预防措施：
✓ 设计时工具检测环路（拓扑排序）
✓ 运行时限制传播深度
✓ 节点设置"冷却时间"（短时间内不重复激活）
✗ 不要在信号处理函数中直接发送新信号（改用队列）
```

### 问题3：状态不一致 (State Inconsistency)

```
现象：
中断后恢复，演员状态与场景状态不匹配

原因：
- 快照保存不完整
- 恢复顺序错误
- 外部系统状态未保存

解决方案：
class StateCustodian {
  Capture(): Snapshot {
    snapshot = new Snapshot()

    // 按依赖顺序保存
    snapshot.sceneState = CaptureSceneState()  // 先保存场景
    snapshot.actorStates = CaptureActorStates()  // 再保存演员
    snapshot.systemStates = CaptureSystemStates()  // 最后保存系统

    snapshot.timestamp = GetCurrentTime()
    snapshot.checksum = CalculateChecksum(snapshot)

    return snapshot
  }

  Restore(snapshot: Snapshot) {
    // 验证完整性
    if CalculateChecksum(snapshot) != snapshot.checksum {
      throw CorruptedSnapshotException()
    }

    // 按依赖顺序恢复（与保存相反）
    RestoreSystemStates(snapshot.systemStates)
    RestoreActorStates(snapshot.actorStates)
    RestoreSceneState(snapshot.sceneState)

    // 验证恢复结果
    VerifyRestoration(snapshot)
  }

  CaptureActorStates(): ActorState[] {
    states = []
    foreach (actor in registeredActors) {
      state = new ActorState()
      state.position = actor.transform.position
      state.rotation = actor.transform.rotation
      state.animationState = actor.animator.GetCurrentState()
      state.aiState = actor.aiController.SaveState()
      state.customData = actor.SaveCustomData()  // 扩展点
      states.Add(state)
    }
    return states
  }
}

最佳实践：
✓ 保存完整状态（不要只保存diff）
✓ 使用校验和验证完整性
✓ 明确定义依赖顺序
✓ 提供状态迁移机制（版本升级）
✗ 不要依赖外部系统的隐式状态
```

### 问题4：资源加载卡顿 (Loading Stutter)

```
现象：
场景播放时突然卡顿，因为资源未提前加载

原因：
- 预测范围太小
- 加载优先级错误
- 同步加载阻塞主线程

解决方案：
class ResourcePredictor {
  lookaheadWindow: float = 5.0  // 预测未来5秒
  preloadDistance: float = 3.0  // 提前3秒加载

  Update(currentTime: float) {
    predictedResources = AnalyzeTimeWindow(
      currentTime + preloadDistance,
      currentTime + lookaheadWindow
    )

    foreach (resource in predictedResources) {
      if !IsLoaded(resource) && !IsLoading(resource) {
        priority = CalculatePriority(resource, currentTime)
        LoadAsync(resource, priority)
      }
    }
  }

  CalculatePriority(resource: Resource, currentTime: float): int {
    // 距离触发时间越近，优先级越高
    timeUntilNeeded = resource.triggerTime - currentTime
    basePriority = (int)(100 / timeUntilNeeded)

    // 关键资源加权
    if resource.isCritical {
      basePriority *= 2
    }

    return basePriority
  }

  LoadAsync(resource: Resource, priority: int) {
    request = new LoadRequest(resource, priority)
    loadQueue.Enqueue(request)

    // 后台线程处理
    ThreadPool.QueueUserWorkItem(() => {
      data = FileSystem.Load(resource.path)
      MainThread.InvokeAsync(() => {
        resource.SetData(data)
        OnResourceLoaded(resource)
      })
    })
  }
}

内存管理：
class MemoryManager {
  maxMemory: long = 500 * 1024 * 1024  // 500MB限制

  OnResourceLoaded(resource: Resource) {
    if GetCurrentMemoryUsage() > maxMemory {
      UnloadLeastRecentlyUsed()
    }
  }

  UnloadLeastRecentlyUsed() {
    candidates = loadedResources
      .Where(r => !r.isPermanent)
      .OrderBy(r => r.lastAccessTime)

    foreach (resource in candidates) {
      if GetCurrentMemoryUsage() < maxMemory * 0.8 {
        break
      }
      Unload(resource)
    }
  }
}

最佳实践：
✓ 异步加载所有非关键资源
✓ 使用优先级队列
✓ 动态调整预测窗口（根据帧率）
✓ 实现内存配额管理
✗ 不要在主线程中阻塞等待资源
```

### 问题5：Tier切换不平滑 (Jarring Tier Transition)

```
现象：
约束突然应用，玩家感觉被"夺走控制权"

原因：
- 瞬间切换约束
- 缺少视觉/听觉反馈
- 玩家未预期到约束

解决方案：
class TierTransition {
  transitionDuration: float = 0.3  // 300ms过渡

  ApplyTierSmooth(newTier: TierDefinition) {
    oldConstraints = currentTier.GetConstraints()
    newConstraints = newTier.GetConstraints()

    // 计算差异
    diff = CalculateDiff(oldConstraints, newConstraints)

    // 渐进应用
    StartCoroutine(SmoothTransition(diff, transitionDuration))

    // 提供反馈
    PlayTransitionFeedback(newTier)
  }

  SmoothTransition(diff: ConstraintDiff, duration: float) {
    elapsed = 0

    while elapsed < duration {
      t = elapsed / duration
      t = EaseInOutCubic(t)  // 缓动函数

      // 插值应用约束
      currentConstraints = Lerp(oldConstraints, newConstraints, t)
      ApplyConstraints(currentConstraints)

      elapsed += deltaTime
      yield
    }

    // 确保最终值精确
    ApplyConstraints(newConstraints)
  }

  PlayTransitionFeedback(tier: TierDefinition) {
    // 视觉反馈
    if tier.restrictionLevel > currentTier.restrictionLevel {
      VignetteFX.FadeIn(0.2, transitionDuration)
    }

    // 听觉反馈
    AudioSystem.Play("tier_change_sfx", volume: 0.5)

    // UI反馈
    if tier.showHint {
      UI.ShowHint(tier.hintText, duration: 2.0)
    }
  }
}

设计原则：
✓ 使用缓动函数（Easing）
✓ 提供多通道反馈（视觉+听觉+UI）
✓ 给玩家"预警"（提前0.5秒提示）
✓ 保留"核心自由"（如视角，即使在Tier4也能看）
✗ 不要瞬间禁用所有输入
```

---

## 3.3 性能优化策略

### 策略1：时间事件的数据结构优化

```
问题：
执行流可能包含数千个事件，线性查询O(n)不可接受

解决方案：
使用优先级队列 (Priority Queue / Min-Heap)

class EventTimeline {
  events: MinHeap<TimedEvent>  // 按时间排序

  AddEvent(event: TimedEvent) {
    events.Insert(event, event.triggerTime)  // O(log n)
  }

  Update(currentTime: float) {
    while events.Count > 0 && events.Peek().triggerTime <= currentTime {
      event = events.Pop()  // O(log n)
      ExecuteEvent(event)
    }
  }
}

优化效果：
- 插入：O(n) → O(log n)
- 查询最早事件：O(n) → O(1)
- 弹出事件：O(n) → O(log n)

进一步优化：分桶策略
class BucketedTimeline {
  buckets: Map<int, List<TimedEvent>>  // 按秒分桶
  bucketSize: float = 1.0

  AddEvent(event: TimedEvent) {
    bucketIndex = (int)(event.triggerTime / bucketSize)
    buckets[bucketIndex].Add(event)
  }

  Update(currentTime: float) {
    bucketIndex = (int)(currentTime / bucketSize)

    // 只检查当前桶
    if buckets.ContainsKey(bucketIndex) {
      foreach (event in buckets[bucketIndex]) {
        if event.triggerTime <= currentTime {
          ExecuteEvent(event)
        }
      }
      buckets.Remove(bucketIndex)  // 处理完移除
    }
  }
}

适用场景：
- 优先级队列：事件分布稀疏
- 分桶策略：事件密集且分布均匀
```

### 策略2：信号传播的批处理

```
问题：
每帧可能发送数十个信号，单独处理开销大

解决方案：
批量处理 + 延迟执行

class SignalBatcher {
  pendingSignals: List<Signal>
  batchSize: int = 32

  Emit(signal: Signal) {
    pendingSignals.Add(signal)

    // 达到批次大小立即处理
    if pendingSignals.Count >= batchSize {
      Flush()
    }
  }

  Update() {
    // 每帧结束前处理剩余信号
    if pendingSignals.Count > 0 {
      Flush()
    }
  }

  Flush() {
    // 按目标分组（减少缓存未命中）
    grouped = pendingSignals.GroupBy(s => s.TargetNodeId)

    foreach (group in grouped) {
      targetNode = GetNode(group.Key)
      targetNode.ProcessSignalBatch(group.ToArray())
    }

    pendingSignals.Clear()
  }
}

优化效果：
- 减少函数调用开销
- 提高缓存命中率
- 支持并行处理
```

### 策略3：状态快照的增量保存

```
问题：
完整快照可能包含数MB数据，频繁保存影响性能

解决方案：
增量快照 (Delta Snapshot) + 定期全量

class IncrementalSnapshotter {
  baseSnapshot: Snapshot
  deltas: List<DeltaSnapshot>
  fullSnapshotInterval: int = 10  // 每10次增量后全量

  Capture(): SnapshotHandle {
    currentState = CollectCurrentState()

    if ShouldTakeFullSnapshot() {
      baseSnapshot = currentState
      deltas.Clear()
      return new SnapshotHandle(baseSnapshot)
    } else {
      delta = CalculateDelta(baseSnapshot, currentState)
      deltas.Add(delta)
      return new SnapshotHandle(baseSnapshot, deltas.ToArray())
    }
  }

  Restore(handle: SnapshotHandle) {
    state = handle.baseSnapshot.Clone()

    // 应用所有增量
    foreach (delta in handle.deltas) {
      ApplyDelta(state, delta)
    }

    RestoreState(state)
  }

  CalculateDelta(base: Snapshot, current: Snapshot): DeltaSnapshot {
    delta = new DeltaSnapshot()

    // 只记录变化的字段
    if base.position != current.position {
      delta.changedFields.Add("position", current.position)
    }
    if base.rotation != current.rotation {
      delta.changedFields.Add("rotation", current.rotation)
    }

    return delta
  }
}

优化效果：
- 内存占用减少 70-90%
- 保存速度提升 5-10倍
- 适合频繁保存的场景（如自动存档）
```

### 策略4：资源依赖的静态分析

```
问题：
运行时分析依赖关系开销大

解决方案：
设计时预计算 + 运行时查表

// 设计时工具
class DependencyAnalyzer {
  Analyze(sceneAsset: SceneAsset): DependencyGraph {
    graph = new DependencyGraph()

    // 遍历所有节点
    foreach (node in sceneAsset.nodes) {
      dependencies = ExtractDependencies(node)
      graph.AddNode(node.id, dependencies)
    }

    // 拓扑排序
    graph.Sort()

    // 序列化为资源清单
    manifest = new ResourceManifest()
    manifest.resources = graph.GetAllResources()
    manifest.loadOrder = graph.GetTopologicalOrder()

    SaveManifest(sceneAsset.path + ".manifest", manifest)

    return graph
  }

  ExtractDependencies(node: Node): Resource[] {
    deps = []

    if node is DialogueNode {
      deps.Add(node.voiceClip)
      deps.Add(node.subtitleFont)
    } else if node is AnimationNode {
      deps.Add(node.animationClip)
      deps.Add(node.characterModel)
    }

    return deps
  }
}

// 运行时加载
class RuntimeLoader {
  LoadScene(scenePath: string) {
    manifest = LoadManifest(scenePath + ".manifest")

    // 直接按预计算的顺序加载
    foreach (resourceId in manifest.loadOrder) {
      LoadResource(resourceId)
    }
  }
}

优化效果：
- 运行时分析时间：秒级 → 毫秒级
- 避免循环依赖（设计时检测）
- 支持并行加载（独立分支）
```

### 策略5：多线程执行流

```
问题：
单线程处理执行流成为瓶颈

解决方案：
分离只读操作和写操作，只读操作并行化

class ParallelExecutionStream {
  readOnlyActions: List<Action>
  writeActions: List<Action>

  Update(currentTime: float) {
    // 分离读写
    (readOnly, write) = PartitionActions(currentTime)

    // 只读操作并行执行
    Parallel.ForEach(readOnly, action => {
      ExecuteReadOnly(action)
    })

    // 写操作串行执行
    foreach (action in write) {
      ExecuteWrite(action)
    }
  }

  PartitionActions(time: float): (Action[], Action[]) {
    pending = GetPendingActions(time)

    readOnly = pending.Where(a => a.IsReadOnly()).ToArray()
    write = pending.Where(a => !a.IsReadOnly()).ToArray()

    return (readOnly, write)
  }
}

线程安全的信号队列：
class LockFreeSignalQueue {
  queue: ConcurrentQueue<Signal>

  Enqueue(signal: Signal) {
    queue.Enqueue(signal)  // 无锁
  }

  DequeueAll(): Signal[] {
    results = []
    while queue.TryDequeue(out signal) {
      results.Add(signal)
    }
    return results
  }
}

注意事项：
⚠ 避免写操作并行（数据竞争）
⚠ 使用无锁数据结构（ConcurrentQueue）
⚠ 注意线程调度开销（任务粒度>1ms）
```

---

## 3.4 集成方案

### 方案A：Unity引擎集成

```
架构映射：
InteractiveScene → MonoBehaviour Component
├─ SceneContainer → SceneController.cs
├─ TimeController → Timeline Integration
├─ TierManager → InputSystem + Cinemachine
├─ SignalSystem → UnityEvent System
├─ ActorSystem → GameObject Registry
└─ ResourceManager → Addressables System

示例代码：
// SceneController.cs
public class SceneController : MonoBehaviour {
  public SceneAsset sceneAsset;

  private ISceneContainer container;
  private UnityGameSystemAdapter adapter;

  void Awake() {
    // 创建容器
    container = SceneContainerFactory.Create();

    // 注入Unity适配器
    adapter = new UnityGameSystemAdapter(this);
    container.InjectAdapter("Input", adapter);
    container.InjectAdapter("Camera", adapter);

    // 初始化
    container.Initialize(sceneAsset.config);
  }

  void Start() {
    container.Activate();
    container.Start();
  }

  void Update() {
    container.Update(Time.deltaTime);
  }

  void OnDestroy() {
    container.Stop();
    container.Deactivate();
    container.Dispose();
  }
}

// 适配器实现
public class UnityGameSystemAdapter : IGameSystemAdapter {
  private SceneController owner;

  public void SendCommand(Command cmd) {
    if (cmd is SetTierCommand tierCmd) {
      ApplyTierToUnity(tierCmd.Tier);
    } else if (cmd is PlayAnimationCommand animCmd) {
      PlayAnimation(animCmd);
    }
  }

  private void ApplyTierToUnity(TierDefinition tier) {
    // Input System
    var playerInput = owner.GetComponent<PlayerInput>();
    playerInput.SwitchCurrentActionMap(tier.InputMapName);

    // Cinemachine
    var vcam = CinemachineCore.Instance.GetActiveBrain(0).ActiveVirtualCamera;
    vcam.GetComponent<CinemachineFramingTransposer>().m_CameraDistance = tier.CameraDistance;
  }
}

资源集成：
- 使用Addressables预加载
- ScriptableObject存储场景配置
- Timeline同步动画和音频
```

### 方案B：Unreal Engine集成

```
架构映射：
InteractiveScene → UActorComponent
├─ SceneContainer → USceneContainerComponent
├─ TimeController → Sequencer Integration
├─ TierManager → Enhanced Input + Camera Rig
├─ SignalSystem → Delegate System
├─ ActorSystem → AActor Registry
└─ ResourceManager → Asset Manager

示例代码：
// SceneContainerComponent.h
UCLASS()
class USceneContainerComponent : public UActorComponent {
  GENERATED_BODY()

  UPROPERTY(EditAnywhere)
  USceneAsset* SceneAsset;

  private:
    TSharedPtr<ISceneContainer> Container;
    TSharedPtr<FUnrealGameSystemAdapter> Adapter;

  public:
    virtual void BeginPlay() override {
      // 创建容器
      Container = FSceneContainerFactory::Create();

      // 创建适配器
      Adapter = MakeShared<FUnrealGameSystemAdapter>(this);
      Container->InjectAdapter(TEXT("Input"), Adapter);
      Container->InjectAdapter(TEXT("Camera"), Adapter);

      // 初始化
      Container->Initialize(SceneAsset->Config);
      Container->Activate();
      Container->Start();
    }

    virtual void TickComponent(float DeltaTime, ...) override {
      Container->Update(DeltaTime);
    }

    virtual void EndPlay(...) override {
      Container->Stop();
      Container->Deactivate();
      Container->Dispose();
    }
};

// 适配器实现
class FUnrealGameSystemAdapter : public IGameSystemAdapter {
  void SendCommand(const FCommand& Command) override {
    if (Command.Type == ECommandType::SetTier) {
      ApplyTierToUnreal(Command.TierData);
    }
  }

  void ApplyTierToUnreal(const FTierDefinition& Tier) {
    // Enhanced Input
    auto* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(GetPlayerController());
    Subsystem->AddMappingContext(Tier.InputMappingContext, Tier.Priority);

    // Camera Rig
    auto* CameraManager = GetPlayerController()->PlayerCameraManager;
    CameraManager->ViewPitchMin = Tier.PitchMin;
    CameraManager->ViewPitchMax = Tier.PitchMax;
  }
};

蓝图集成：
- 暴露UFUNCTION供蓝图调用
- 使用Sequencer同步Cinematic
- DataAsset存储配置
```

### 方案C：自定义引擎集成

```
接口定义：
class IEngineAdapter {
  // 生命周期
  virtual void OnSceneStart() = 0;
  virtual void OnSceneEnd() = 0;

  // 输入
  virtual void SetInputMapping(InputMap map) = 0;
  virtual void RestoreInputMapping() = 0;

  // 摄像机
  virtual void SetCameraConstraints(CameraConstraints constraints) = 0;
  virtual void RemoveCameraConstraints() = 0;

  // 动画
  virtual void PlayAnimation(ActorId actor, AnimationClip clip) = 0;
  virtual void StopAnimation(ActorId actor) = 0;

  // 音频
  virtual void PlaySound(SoundClip clip, Vector3 position) = 0;

  // 资源
  virtual void PreloadResource(ResourceId id) = 0;
  virtual bool IsResourceLoaded(ResourceId id) = 0;
};

实现框架：
class CustomEngineAdapter : public IEngineAdapter {
  IInputSystem* inputSystem;
  ICameraSystem* cameraSystem;
  IAnimationSystem* animationSystem;
  IAudioSystem* audioSystem;
  IResourceSystem* resourceSystem;

  CustomEngineAdapter(GameEngine* engine) {
    inputSystem = engine->GetInputSystem();
    cameraSystem = engine->GetCameraSystem();
    animationSystem = engine->GetAnimationSystem();
    audioSystem = engine->GetAudioSystem();
    resourceSystem = engine->GetResourceSystem();
  }

  void PlayAnimation(ActorId actor, AnimationClip clip) override {
    IGameEntity* entity = GetEntity(actor);
    IAnimationComponent* animComp = entity->GetComponent<IAnimationComponent>();
    animComp->Play(clip.name, clip.blendTime);
  }

  // ... 实现其他接口
};

集成步骤：
1. 实现IEngineAdapter接口
2. 创建SceneContainer实例
3. 注入适配器：container->InjectAdapter(adapter)
4. 在游戏循环中调用：container->Update(deltaTime)
```

---

## 3.5 测试策略

### 单元测试

```
测试场景图逻辑：
class SceneGraphTests {
  Test_NodeConnection() {
    graph = new SceneGraph();
    nodeA = new DialogueNode("A");
    nodeB = new ChoiceNode("B");

    graph.AddNode(nodeA);
    graph.AddNode(nodeB);
    graph.Connect(nodeA.output, nodeB.input);

    Assert.AreEqual(1, nodeA.outputs[0].connections.Count);
    Assert.AreEqual(nodeB, nodeA.outputs[0].connections[0].targetNode);
  }

  Test_CycleDetection() {
    graph = new SceneGraph();
    // 创建环路：A → B → C → A

    Assert.Throws<CyclicGraphException>(() => {
      graph.Validate();
    });
  }
}

测试时间控制：
class TimeControllerTests {
  Test_EventScheduling() {
    controller = new TimeController();

    callCount = 0;
    controller.ScheduleAction(() => callCount++, 1.0);

    controller.Update(0.5);  // 0.5秒
    Assert.AreEqual(0, callCount);  // 未触发

    controller.Update(0.6);  // 1.1秒
    Assert.AreEqual(1, callCount);  // 已触发
  }

  Test_PauseResume() {
    controller = new TimeController();
    controller.Play();

    controller.Update(1.0);
    Assert.AreEqual(1.0, controller.GetCurrentTime());

    controller.Pause();
    controller.Update(1.0);  // 暂停期间
    Assert.AreEqual(1.0, controller.GetCurrentTime());  // 时间不变

    controller.Resume();
    controller.Update(1.0);
    Assert.AreEqual(2.0, controller.GetCurrentTime());
  }
}

Mock适配器：
class MockGameSystemAdapter : IGameSystemAdapter {
  public List<Command> commandHistory = [];

  public void SendCommand(Command cmd) {
    commandHistory.Add(cmd);
  }

  public void AssertCommandSent<T>() where T : Command {
    Assert.IsTrue(commandHistory.Any(c => c is T));
  }
}
```

### 集成测试

```
测试完整场景流程：
class SceneIntegrationTests {
  Test_SimpleDialogueFlow() {
    // 准备
    scene = LoadScene("test_dialogue.scene");
    mockAdapter = new MockGameSystemAdapter();
    scene.InjectAdapter(mockAdapter);

    // 执行
    scene.Start();
    scene.Update(2.0);  // 模拟2秒

    // 验证
    Assert.AreEqual(SceneState.Completed, scene.GetState());
    mockAdapter.AssertCommandSent<PlayAnimationCommand>();
    mockAdapter.AssertCommandSent<PlaySoundCommand>();
  }

  Test_InterruptAndReturn() {
    scene = LoadScene("test_interrupt.scene");
    scene.Start();

    // 正常执行到一半
    scene.Update(1.0);
    Assert.AreEqual("node_main", scene.GetCurrentNode().id);

    // 触发中断
    scene.TriggerInterrupt("distance_interrupt");
    Assert.AreEqual("node_interrupt", scene.GetCurrentNode().id);

    // 满足返回条件
    scene.SetVariable("player_returned", true);
    scene.Update(0.1);
    Assert.AreEqual("node_main", scene.GetCurrentNode().id);  // 返回
  }
}
```

### 性能测试

```
压力测试：
class PerformanceTests {
  Test_LargeScenePerformance() {
    scene = GenerateLargeScene(1000);  // 1000个节点

    stopwatch = Stopwatch.StartNew();
    scene.Initialize();
    stopwatch.Stop();

    Assert.Less(stopwatch.ElapsedMilliseconds, 100);  // <100ms初始化
  }

  Test_SignalThroughput() {
    signalSystem = new SignalSystem();

    // 发送10000个信号
    for (i = 0; i < 10000; i++) {
      signalSystem.Emit(new Signal("test"));
    }

    stopwatch = Stopwatch.StartNew();
    signalSystem.ProcessAll();
    stopwatch.Stop();

    Assert.Less(stopwatch.ElapsedMilliseconds, 16);  // <1帧
  }

  Test_MemoryUsage() {
    baseline = GC.GetTotalMemory(true);

    scene = LoadScene("test_scene.scene");
    scene.Start();

    for (i = 0; i < 1000; i++) {
      scene.Update(0.016);  // 1000帧
    }

    GC.Collect();
    finalMemory = GC.GetTotalMemory(true);

    memoryIncrease = finalMemory - baseline;
    Assert.Less(memoryIncrease, 10 * 1024 * 1024);  // <10MB增长
  }
}
```

---

## 📋 总结

### 三层模型的关键要点

```
第一层（概念模型层）：
- 定义7个核心概念及其职责边界
- 确立组件间的交互协议
- 遵循SOLID设计原则

第二层（逻辑架构层）：
- 设计清晰的组件接口
- 定义3种数据流模式（Pipeline, Event Bus, Reactive Stream）
- 明确生命周期管理和扩展点

第三层（实现策略层）：
- 应用5种核心设计模式
- 提供5类常见问题的解决方案
- 覆盖5个方向的性能优化
- 支持3种引擎的集成方案
```

### 使用指南

```
架构设计阶段：
→ 参考第一层理解核心概念
→ 使用职责边界矩阵避免耦合

详细设计阶段：
→ 参考第二层定义接口
→ 选择合适的数据流模式

编码实现阶段：
→ 参考第三层应用设计模式
→ 参考解决方案应对具体问题

优化阶段：
→ 参考性能优化策略
→ 使用测试策略验证

集成阶段：
→ 选择对应的引擎集成方案
→ 实现IEngineAdapter接口
```

### 扩展路径

```
最小实现（MVP）：
- 第一层的前3个概念（容器、时间控制器、自由度调节器）
- 第二层的核心接口
- 第三层的状态机模式

标准实现：
- 第一层全部7个概念
- 第二层全部接口和数据流
- 第三层的5种设计模式

完整实现：
- 三层全部内容
- 性能优化全部实施
- 引擎深度集成
```

---

**版本**: 2.0
**日期**: 2026-02-09
**架构模型**: 三层分离架构