# InteractiveScene系统完整内容框架 - Part 2
> 中断系统与资源管理的完整子元素拆解

---

## 🛡️ 6. Interrupt System (中断系统) - 完整拆解

### 6.1 中断条件 (Interrupt Conditions)

```
中断条件
├─ 条件基类 (Base Condition)
│  ├─ ConditionId (条件ID)
│  ├─ ConditionType (条件类型)
│  ├─ ConditionName (条件名称)
│  ├─ Priority (优先级)
│  ├─ Enabled (启用状态)
│  └─ CheckFrequency (检查频率)
│     ├─ EveryFrame (每帧)
│     ├─ EveryNFrames (每N帧)
│     └─ OnEvent (事件触发时)
│
├─ 距离条件 (Distance Conditions)
│  ├─ DistancePlayerEntity (玩家与实体距离)
│  │  ├─ TargetEntityId (目标实体)
│  │  ├─ DistanceThreshold (距离阈值)
│  │  ├─ ComparisonMode (比较模式)
│  │  │  ├─ GreaterThan (大于)
│  │  │  ├─ LessThan (小于)
│  │  │  └─ InRange (在范围内)
│  │  ├─ Use2DDistance (使用2D距离)
│  │  └─ ContinuousCheck (连续检查)
│  │
│  ├─ DistancePlayerNode (玩家与节点距离)
│  │  ├─ NodeReference (节点引用)
│  │  ├─ DistanceThreshold
│  │  └─ (其他同上)
│  │
│  ├─ DistanceSpeaker (说话者距离)
│  │  ├─ SpeakerId (说话者ID)
│  │  ├─ DistanceThreshold
│  │  └─ (其他同上)
│  │
│  └─ DistanceBetweenActors (演员间距离)
│     ├─ Actor1Id
│     ├─ Actor2Id
│     └─ DistanceThreshold
│
├─ 状态条件 (State Conditions)
│  ├─ PlayerCombat (玩家战斗状态)
│  │  ├─ TriggerOnEnter (进入战斗触发)
│  │  ├─ TriggerOnExit (退出战斗触发)
│  │  └─ CombatIntensity (战斗强度阈值)
│  │
│  ├─ ActorDistracted (演员分心)
│  │  ├─ ActorId
│  │  ├─ DistractionType (分心类型)
│  │  │  ├─ LookAway (视线离开)
│  │  │  ├─ InCombat (进入战斗)
│  │  │  └─ Stunned (被眩晕)
│  │  └─ DistractionDuration (分心持续时间)
│  │
│  ├─ AnyoneDistracted (任何人分心)
│  │  ├─ ParticipantList[] (参与者列表)
│  │  └─ ThresholdCount (阈值数量)
│  │
│  └─ ActorDead (演员死亡)
│     ├─ ActorId
│     └─ CriticalActor (是否关键角色)
│
├─ 事件条件 (Event Conditions)
│  ├─ TriggerActivated (触发器激活)
│  │  ├─ TriggerReference (触发器引用)
│  │  ├─ TriggerMode (触发模式)
│  │  │  ├─ OnEnter (进入时)
│  │  │  ├─ OnExit (退出时)
│  │  │  └─ WhileInside (在内部时)
│  │  └─ RequiredEntity (必需实体)
│  │
│  ├─ FactChanged (事实变化)
│  │  ├─ FactName (事实名称)
│  │  ├─ ExpectedValue (期望值)
│  │  ├─ ComparisonOperator (比较运算符)
│  │  │  ├─ Equal (等于)
│  │  │  ├─ NotEqual (不等于)
│  │  │  ├─ Greater (大于)
│  │  │  └─ Less (小于)
│  │  └─ FactType (事实类型)
│  │     ├─ Integer
│  │     ├─ Float
│  │     ├─ Boolean
│  │     └─ String
│  │
│  ├─ VehicleImpact (载具撞击)
│  │  ├─ ImpactForce (撞击力度)
│  │  ├─ MinimumThreshold (最小阈值)
│  │  └─ PlayerMounted (玩家必须在载具中)
│  │
│  └─ CustomEvent (自定义事件)
│     ├─ EventName (事件名称)
│     ├─ EventData (事件数据)
│     └─ EventSource (事件源)
│
├─ 时间条件 (Time Conditions)
│  ├─ TimeoutCondition (超时条件)
│  │  ├─ TimeoutDuration (超时时长)
│  │  ├─ StartTime (开始时间)
│  │  └─ PauseOnInactive (不活跃时暂停)
│  │
│  └─ TimeRangeCondition (时间范围条件)
│     ├─ StartTime
│     ├─ EndTime
│     └─ RepeatDaily (每日重复)
│
└─ 复合条件 (Composite Conditions)
   ├─ AndCondition (与条件)
   │  ├─ SubConditions[] (子条件数组)
   │  └─ AllMustBeTrue (全部必须为真)
   │
   ├─ OrCondition (或条件)
   │  └─ AnyCanBeTrue (任一为真即可)
   │
   └─ NotCondition (非条件)
      └─ InvertCondition (反转条件)
```

---

### 6.2 返回条件 (Return Conditions)

```
返回条件
├─ 条件基类
│  ├─ ReturnConditionId
│  ├─ ReturnConditionType
│  ├─ EvaluationFrequency (求值频率)
│  └─ Timeout (超时设置)
│
├─ 距离返回条件
│  ├─ ReturnDistancePlayerEntity (玩家返回实体附近)
│  │  ├─ TargetEntityId
│  │  ├─ ReturnDistance (返回距离)
│  │  ├─ StayDuration (停留时长要求)
│  │  └─ IndicatorDisplay (指示器显示)
│  │
│  ├─ ReturnDistancePlayerNode (玩家返回节点)
│  └─ ReturnDistanceSpeaker (玩家返回说话者)
│
├─ 状态返回条件
│  ├─ ReturnPlayerCombat (玩家退出战斗)
│  │  ├─ RequiredNoCombat (要求无战斗)
│  │  ├─ GracePeriod (宽限期)
│  │  └─ AllEnemiesDead (所有敌人死亡)
│  │
│  ├─ ReturnDistracted (分心结束)
│  │  └─ ActorFocused (演员重新聚焦)
│  │
│  └─ ReturnConditionCleared (条件清除)
│     └─ OriginalConditionResolved (原条件解决)
│
├─ 事件返回条件
│  ├─ ReturnFact (事实返回)
│  │  ├─ FactName
│  │  ├─ RequiredValue
│  │  └─ ComparisonOperator
│  │
│  ├─ ReturnTrigger (触发器返回)
│  │  └─ TriggerReference
│  │
│  └─ ReturnCustomEvent (自定义事件返回)
│
├─ 时间返回条件
│  ├─ ReturnTimeout (超时返回)
│  │  └─ MaxWaitTime (最大等待时间)
│  │
│  └─ ReturnImmediate (立即返回)
│     └─ 测试用途
│
└─ 特殊返回条件
   ├─ NeverReturn (永不返回)
   │  └─ 用于不可逆中断
   │
   ├─ AlwaysReturn (总是返回)
   │  └─ DummyCondition (虚拟条件，测试用)
   │
   └─ ConditionalReturn (条件返回)
      ├─ MultipleConditions[]
      └─ ReturnMode (返回模式)
         ├─ FirstSatisfied (第一个满足)
         └─ AllSatisfied (全部满足)
```

---

### 6.3 上下文管理 (Context Management)

```
上下文管理
├─ 上下文快照 (Context Snapshot)
│  ├─ 场景状态
│  │  ├─ CurrentNodeId (当前节点)
│  │  ├─ PlaybackTime (播放时间)
│  │  ├─ ExecutionStreamState (执行流状态)
│  │  │  ├─ ActionChannelSnapshot (动作通道快照)
│  │  │  ├─ ControlChannelSnapshot (控制通道快照)
│  │  │  └─ ActivationChannelSnapshot (激活通道快照)
│  │  ├─ ActiveTier (激活Tier)
│  │  └─ SceneVariables (场景变量)
│  │
│  ├─ 演员状态
│  │  ├─ ActorStates[] (演员状态数组)
│  │  │  ├─ ActorId
│  │  │  ├─ Position
│  │  │  ├─ Rotation
│  │  │  ├─ AnimationState
│  │  │  ├─ PhysicsState
│  │  │  └─ GameplayState
│  │  │
│  │  ├─ ActorBindings[] (演员绑定)
│  │  └─ ActorRelationships[] (演员关系)
│  │
│  ├─ 玩家状态
│  │  ├─ PlayerPosition
│  │  ├─ PlayerRotation
│  │  ├─ CameraTransform (摄像机变换)
│  │  ├─ PlayerHealth
│  │  ├─ PlayerEquipment
│  │  ├─ PlayerBuffs
│  │  └─ PlayerInventory (可选)
│  │
│  ├─ 环境状态
│  │  ├─ TimeOfDay (一天中的时间)
│  │  ├─ Weather (天气)
│  │  ├─ DynamicObjects[] (动态物体)
│  │  └─ TriggerStates[] (触发器状态)
│  │
│  └─ 元数据
│     ├─ SnapshotId
│     ├─ Timestamp
│     ├─ InterruptReason (中断原因)
│     └─ SnapshotSize (快照大小)
│
├─ 上下文操作
│  ├─ CaptureContext() (捕获上下文)
│  │  ├─ 选择性捕获 (性能优化)
│  │  ├─ 压缩数据
│  │  └─ 验证完整性
│  │
│  ├─ RestoreContext() (恢复上下文)
│  │  ├─ 解压数据
│  │  ├─ 验证兼容性
│  │  ├─ 插值恢复 (平滑过渡)
│  │  └─ 冲突解决
│  │
│  ├─ CompareContexts() (比较上下文)
│  │  └─ 生成差异报告
│  │
│  └─ MergeContexts() (合并上下文)
│     └─ 部分恢复场景
│
└─ 存储管理
   ├─ ContextStack[] (上下文堆栈)
   │  └─ 支持嵌套中断
   │
   ├─ ContextCache (上下文缓存)
   │  ├─ MaxCacheSize
   │  └─ EvictionPolicy (驱逐策略)
   │
   └─ PersistentStorage (持久化存储)
      └─ 保存/加载支持
```

---

### 6.4 中断场景 (Interrupt Scenarios)

```
中断场景
├─ 场景定义 (Scenario Definition)
│  ├─ ScenarioId (场景ID)
│  ├─ ScenarioName (场景名称)
│  ├─ ScenarioType (场景类型)
│  │  ├─ Warning (警告)
│  │  ├─ MinorInterrupt (轻度中断)
│  │  ├─ MajorInterrupt (重度中断)
│  │  └─ CriticalFailure (严重失败)
│  │
│  ├─ TriggerConditions[] (触发条件数组)
│  │  └─ 可配置多个条件
│  │
│  ├─ ReturnConditions[] (返回条件数组)
│  │  └─ 可配置多个返回路径
│  │
│  └─ Priority (优先级)
│
├─ 中断响应 (Interrupt Response)
│  ├─ 警告响应 (Warning Response)
│  │  ├─ PlayWarningVO (播放警告语音)
│  │  │  ├─ VoiceLineId
│  │  │  ├─ Speaker
│  │  │  └─ Urgency (紧急程度)
│  │  │
│  │  ├─ ShowWarningUI (显示警告UI)
│  │  │  ├─ MessageText
│  │  │  ├─ IconType
│  │  │  └─ Duration
│  │  │
│  │  └─ ContinueMainLine (继续主线)
│  │
│  ├─ 软中断响应 (Soft Interrupt)
│  │  ├─ PauseMainLine (暂停主线)
│  │  ├─ ExecuteInterruptContent (执行中断内容)
│  │  │  ├─ InterruptDialogue (中断对话)
│  │  │  ├─ InterruptAnimation (中断动画)
│  │  │  └─ InterruptCamera (中断摄像机)
│  │  │
│  │  ├─ WaitForReturn (等待返回)
│  │  └─ ResumeMainLine (恢复主线)
│  │
│  └─ 硬中断响应 (Hard Interrupt)
│     ├─ StopMainLine (停止主线)
│     ├─ SaveContext (保存上下文)
│     ├─ JumpToBranch (跳转到分支)
│     │  ├─ FailureBranch (失败分支)
│     │  ├─ AlternativeBranch (替代分支)
│     │  └─ RecoveryBranch (恢复分支)
│     │
│     └─ NotifyQuest (通知任务系统)
│
├─ 中断内容 (Interrupt Content)
│  ├─ 提示内容 (Reminder Content)
│  │  ├─ "嘿，回来这里"
│  │  ├─ "你去哪了？"
│  │  └─ "我们还没聊完呢"
│  │
│  ├─ 警告内容 (Warning Content)
│  │  ├─ "别走太远"
│  │  ├─ "注意，你正在离开"
│  │  └─ UI图标+方向指示
│  │
│  ├─ 失败内容 (Failure Content)
│  │  ├─ "任务失败"
│  │  ├─ "你离开了场景"
│  │  └─ 任务日志更新
│  │
│  └─ 替代内容 (Alternative Content)
│     ├─ "好吧，既然你不想听"
│     ├─ "我们换个方式"
│     └─ 进入替代分支
│
└─ 场景优先级
   ├─ LowPriority (低优先级)
   │  └─ 可被覆盖
   │
   ├─ MediumPriority (中优先级)
   │  └─ 正常中断
   │
   └─ HighPriority (高优先级)
      └─ 覆盖其他中断
```

---

### 6.5 恢复机制 (Recovery Mechanism)

```
恢复机制
├─ 恢复策略 (Recovery Strategy)
│  ├─ DirectResume (直接恢复)
│  │  ├─ 适用：短暂中断
│  │  ├─ 方法：
│  │  │  └─ 直接继续执行
│  │  └─ 过渡：无过渡或淡入
│  │
│  ├─ SmooothResume (平滑恢复)
│  │  ├─ 适用：位置/状态有变化
│  │  ├─ 方法：
│  │  │  ├─ 插值位置
│  │  │  ├─ 混合动画
│  │  │  └─ 平滑摄像机
│  │  └─ 过渡：1-2秒混合
│  │
│  ├─ PartialResume (部分恢复)
│  │  ├─ 适用：某些状态不可恢复
│  │  ├─ 方法：
│  │  │  ├─ 恢复关键状态
│  │  │  ├─ 重新初始化其他状态
│  │  │  └─ 修正不一致
│  │  └─ 策略：最小影响原则
│  │
│  └─ RecontextualizedResume (重新语境化恢复)
│     ├─ 适用：世界状态显著改变
│     ├─ 方法：
│     │  ├─ 播放过渡对话
│     │  ├─ 调整剧情逻辑
│     │  └─ 更新目标
│     └─ 示例："刚才发生了什么？让我们继续..."
│
├─ 恢复检查 (Recovery Validation)
│  ├─ PreRecoveryCheck (恢复前检查)
│  │  ├─ ValidateActors (验证演员)
│  │  │  ├─ ActorExists (演员存在)
│  │  │  ├─ ActorAlive (演员存活)
│  │  │  └─ ActorAvailable (演员可用)
│  │  │
│  │  ├─ ValidateState (验证状态)
│  │  │  ├─ WorldStateConsistent (世界状态一致)
│  │  │  ├─ ResourcesAvailable (资源可用)
│  │  │  └─ NoBlockingConditions (无阻塞条件)
│  │  │
│  │  └─ ValidateContext (验证上下文)
│  │     ├─ ContextNotCorrupted (上下文未损坏)
│  │     └─ ContextCompatible (上下文兼容)
│  │
│  ├─ RecoveryConflicts (恢复冲突)
│  │  ├─ ActorMissing (演员丢失)
│  │  │  └─ 解决：跳过或替换演员
│  │  │
│  │  ├─ ActorDead (演员死亡)
│  │  │  └─ 解决：进入失败分支
│  │  │
│  │  ├─ StateInvalid (状态无效)
│  │  │  └─ 解决：重新初始化
│  │  │
│  │  └─ LocationChanged (位置改变)
│  │     └─ 解决：传送或重新语境化
│  │
│  └─ PostRecoveryValidation (恢复后验证)
│     ├─ VerifyState (验证状态)
│     ├─ VerifySync (验证同步)
│     └─ VerifyFlow (验证流程)
│
├─ 恢复过渡 (Recovery Transition)
│  ├─ VisualTransition (视觉过渡)
│  │  ├─ FadeInOut (淡入淡出)
│  │  ├─ BlurTransition (模糊过渡)
│  │  └─ CutTransition (切换过渡)
│  │
│  ├─ AudioTransition (音频过渡)
│  │  ├─ CrossFade (交叉淡化)
│  │  └─ VolumeRamp (音量渐变)
│  │
│  └─ NarrativeTransition (叙事过渡)
│     ├─ TransitionDialogue (过渡对话)
│     └─ ExplanationVO (解释语音)
│
└─ 失败处理 (Failure Handling)
   ├─ RecoveryFailed (恢复失败)
   │  ├─ Reason (失败原因)
   │  ├─ FallbackAction (回退动作)
   │  │  ├─ AbortScene (放弃场景)
   │  │  ├─ JumpToSafeState (跳转到安全状态)
   │  │  └─ RestartScene (重启场景)
   │  │
   │  └─ NotifyUser (通知用户)
   │
   └─ ErrorLog (错误日志)
      ├─ LogLevel
      ├─ ErrorMessage
      └─ StackTrace
```

---

## 🔄 7. Resource Manager (资源管理) - 完整拆解

### 7.1 资源引用 (Resource References)

```
资源引用
├─ 引用类型 (Reference Types)
│  ├─ DirectReference (直接引用)
│  │  ├─ ResourcePath (资源路径)
│  │  ├─ ResourceType (资源类型)
│  │  └─ LoadPriority (加载优先级)
│  │
│  ├─ IndirectReference (间接引用)
│  │  ├─ ReferenceId (引用ID)
│  │  └─ ResolutionRule (解析规则)
│  │
│  └─ ConditionalReference (条件引用)
│     ├─ Condition (条件)
│     └─ AlternativeReferences[] (备选引用)
│
├─ 资源类型 (Resource Types)
│  ├─ 视觉资源
│  │  ├─ Meshes (网格)
│  │  ├─ Textures (纹理)
│  │  ├─ Materials (材质)
│  │  ├─ VFX (特效)
│  │  └─ Lightmaps (光照贴图)
│  │
│  ├─ 动画资源
│  │  ├─ AnimationClips (动画片段)
│  │  ├─ AnimationControllers (动画控制器)
│  │  ├─ BlendTrees (混合树)
│  │  └─ IKRigs (IK装备)
│  │
│  ├─ 音频资源
│  │  ├─ VoiceLines (语音)
│  │  ├─ SoundEffects (音效)
│  │  ├─ Music (音乐)
│  │  └─ AmbientSounds (环境音)
│  │
│  ├─ 实体资源
│  │  ├─ CharacterPrefabs (角色预制件)
│  │  ├─ PropPrefabs (道具预制件)
│  │  ├─ VehiclePrefabs (载具预制件)
│  │  └─ EnvironmentPrefabs (环境预制件)
│  │
│  └─ 数据资源
│     ├─ Dialogues (对话数据)
│     ├─ Localization (本地化数据)
│     ├─ ConfigTables (配置表)
│     └─ ScriptAssets (脚本资源)
│
├─ 引用扫描 (Reference Scanning)
│  ├─ ScanSceneGraph() (扫描场景图)
│  │  ├─ 遍历所有节点
│  │  ├─ 提取资源引用
│  │  └─ 构建依赖图
│  │
│  ├─ ScanExecutionStream() (扫描执行流)
│  │  ├─ 扫描动作记录
│  │  ├─ 提取资源ID
│  │  └─ 标记使用时间
│  │
│  └─ ScanActorBindings() (扫描演员绑定)
│     ├─ 扫描演员资源
│     └─ 提取依赖资源
│
└─ 依赖管理 (Dependency Management)
   ├─ DependencyGraph (依赖图)
   │  ├─ Nodes[] (资源节点)
   │  ├─ Edges[] (依赖边)
   │  └─ ResolutionOrder[] (解析顺序)
   │
   ├─ CircularDependency (循环依赖)
   │  ├─ Detection (检测)
   │  └─ Resolution (解决)
   │
   └─ MissingReference (缺失引用)
      ├─ Detection (检测)
      └─ FallbackStrategy (回退策略)
```

---

### 7.2 预加载系统 (Preloading System)

```
预加载系统
├─ 预加载策略 (Preload Strategy)
│  ├─ StaticPreload (静态预加载)
│  │  ├─ SceneStartup (场景启动时)
│  │  ├─ LoadAllResources (加载所有资源)
│  │  └─ 适用：小型场景
│  │
│  ├─ DynamicPreload (动态预加载)
│  │  ├─ PredictiveLoading (预测性加载)
│  │  ├─ LoadAhead (提前加载)
│  │  └─ 适用：大型场景
│  │
│  └─ HybridPreload (混合预加载)
│     ├─ CriticalStatic (关键资源静态加载)
│     ├─ NonCriticalDynamic (非关键资源动态加载)
│     └─ 适用：中大型场景
│
├─ 预加载分析 (Preload Analysis)
│  ├─ GraphAnalysis (图分析)
│  │  ├─ AnalyzeSceneGraph() (分析场景图)
│  │  ├─ PredictExecution() (预测执行路径)
│  │  │  ├─ MainPath (主路径)
│  │  │  ├─ LikelyBranches (可能分支)
│  │  │  └─ UnlikelyBranches (不太可能分支)
│  │  │
│  │  └─ GenerateLoadList() (生成加载列表)
│  │     ├─ Priority1: 必需资源
│  │     ├─ Priority2: 高概率资源
│  │     └─ Priority3: 低概率资源
│  │
│  ├─ TimelineAnalysis (时间轴分析)
│  │  ├─ AnalyzeExecutionStream() (分析执行流)
│  │  ├─ CalculateLoadTiming() (计算加载时机)
│  │  │  ├─ ResourceUsageTime (资源使用时间)
│  │  │  ├─ LoadDuration (加载耗时)
│  │  │  └─ SafetyBuffer (安全缓冲)
│  │  │
│  │  └─ OptimizeLoadOrder() (优化加载顺序)
│  │
│  └─ PlayerBehaviorPrediction (玩家行为预测)
│     ├─ HistoricalData (历史数据)
│     ├─ PredictChoice (预测选择)
│     └─ AdaptiveLoading (自适应加载)
│
├─ 预加载执行 (Preload Execution)
│  ├─ LoadQueue (加载队列)
│  │  ├─ QueueEntry (队列条目)
│  │  │  ├─ ResourceId
│  │  │  ├─ Priority
│  │  │  ├─ LoadBy (必须在X时间前加载)
│  │  │  └─ LoadStatus
│  │  │
│  │  ├─ Enqueue() (入队)
│  │  ├─ Dequeue() (出队)
│  │  └─ Reorder() (重排序)
│  │
│  ├─ AsyncLoading (异步加载)
│  │  ├─ LoadWorkerThreads[] (加载工作线程)
│  │  ├─ LoadCallback (加载回调)
│  │  └─ ProgressTracking (进度追踪)
│  │
│  └─ LoadBatching (批量加载)
│     ├─ BatchSize (批次大小)
│     ├─ BatchInterval (批次间隔)
│     └─ BatchOptimization (批次优化)
│
└─ 预加载监控 (Preload Monitoring)
   ├─ LoadProgress (加载进度)
   │  ├─ TotalResources (总资源数)
   │  ├─ LoadedResources (已加载数)
   │  ├─ LoadingResources (加载中)
   │  └─ FailedResources (失败数)
   │
   ├─ PerformanceMetrics (性能指标)
   │  ├─ LoadTime (加载时间)
   │  ├─ MemoryUsage (内存使用)
   │  └─ IOBandwidth (IO带宽)
   │
   └─ Alerts (警报)
      ├─ SlowLoading (加载过慢)
      ├─ MemoryWarning (内存警告)
      └─ ResourceMissing (资源缺失)
```

---

### 7.3 流式加载 (Streaming Loading)

```
流式加载
├─ 流式策略 (Streaming Strategy)
│  ├─ ProximityStreaming (距离流式)
│  │  ├─ LoadRadius (加载半径)
│  │  ├─ UnloadRadius (卸载半径)
│  │  └─ PlayerPosition (玩家位置)
│  │
│  ├─ ProgressionStreaming (进度流式)
│  │  ├─ CurrentProgress (当前进度)
│  │  ├─ UpcomingSegments[] (即将到来的片段)
│  │  └─ LookAhead (前瞻距离)
│  │
│  └─ EventDrivenStreaming (事件驱动流式)
│     ├─ TriggerEvents[] (触发事件)
│     ├─ LoadOnEvent (事件时加载)
│     └─ UnloadOnEvent (事件时卸载)
│
├─ LOD管理 (LOD Management)
│  ├─ LODLevel (LOD层级)
│  │  ├─ LOD0 (最高质量)
│  │  ├─ LOD1 (高质量)
│  │  ├─ LOD2 (中质量)
│  │  ├─ LOD3 (低质量)
│  │  └─ LOD4 (极低质量)
│  │
│  ├─ LODSelection (LOD选择)
│  │  ├─ DistanceBasedLOD (基于距离)
│  │  ├─ ImportanceBasedLOD (基于重要性)
│  │  └─ PerformanceBasedLOD (基于性能)
│  │
│  └─ LODTransition (LOD过渡)
│     ├─ CrossFade (交叉淡化)
│     ├─ PopSuppression (Pop抑制)
│     └─ HysteresisZone (迟滞区)
│
├─ 流式缓冲 (Streaming Buffer)
│  ├─ RingBuffer (环形缓冲)
│  │  ├─ BufferSize (缓冲大小)
│  │  ├─ ReadPointer (读指针)
│  │  └─ WritePointer (写指针)
│  │
│  ├─ DoubleBuffering (双缓冲)
│  │  ├─ FrontBuffer (前缓冲)
│  │  ├─ BackBuffer (后缓冲)
│  │  └─ BufferSwap (缓冲交换)
│  │
│  └─ BufferManagement (缓冲管理)
│     ├─ BufferAllocation (缓冲分配)
│     ├─ BufferRelease (缓冲释放)
│     └─ BufferCompaction (缓冲压缩)
│
└─ 流式调度 (Streaming Scheduling)
   ├─ Scheduler (调度器)
   │  ├─ ScheduleLoad (调度加载)
   │  ├─ ScheduleUnload (调度卸载)
   │  └─ PriorityAdjustment (优先级调整)
   │
   ├─ BandwidthManagement (带宽管理)
   │  ├─ IOBudget (IO预算)
   │  ├─ ThrottleLoading (节流加载)
   │  └─ BurstLoading (突发加载)
   │
   └─ FrameBudget (帧预算)
      ├─ MaxLoadTimePerFrame (每帧最大加载时间)
      ├─ LoadDistribution (加载分布)
      └─ SpikeSmoothing (峰值平滑)
```

---

### 7.4 内存管理 (Memory Management)

```
内存管理
├─ 内存池 (Memory Pool)
│  ├─ PoolDefinition (池定义)
│  │  ├─ ScenePool (场景池)
│  │  ├─ ActorPool (演员池)
│  │  ├─ ResourcePool (资源池)
│  │  └─ TemporaryPool (临时池)
│  │
│  ├─ PoolAllocation (池分配)
│  │  ├─ Allocate() (分配)
│  │  ├─ Deallocate() (释放)
│  │  └─ Reallocate() (重新分配)
│  │
│  └─ PoolStatistics (池统计)
│     ├─ TotalSize (总大小)
│     ├─ UsedSize (已用大小)
│     ├─ FreeSize (可用大小)
│     └─ Fragmentation (碎片率)
│
├─ 引用计数 (Reference Counting)
│  ├─ RefCount (引用计数)
│  │  ├─ AddRef() (增加引用)
│  │  ├─ Release() (释放引用)
│  │  └─ GetRefCount() (获取计数)
│  │
│  ├─ WeakReference (弱引用)
│  │  └─ 不增加引用计数
│  │
│  └─ AutoRelease (自动释放)
│     └─ RefCount = 0时自动释放
│
├─ 垃圾回收 (Garbage Collection)
│  ├─ GCStrategy (GC策略)
│  │  ├─ Immediate (立即)
│  │  ├─ Deferred (延迟)
│  │  └─ Incremental (增量)
│  │
│  ├─ GCTrigger (GC触发)
│  │  ├─ MemoryThreshold (内存阈值)
│  │  ├─ TimeInterval (时间间隔)
│  │  └─ Manual (手动)
│  │
│  └─ GCProcess (GC过程)
│     ├─ MarkPhase (标记阶段)
│     ├─ SweepPhase (清扫阶段)
│     └─ CompactPhase (压缩阶段)
│
├─ 内存监控 (Memory Monitoring)
│  ├─ MemoryUsage (内存使用)
│  │  ├─ CurrentUsage (当前使用)
│  │  ├─ PeakUsage (峰值使用)
│  │  └─ AverageUsage (平均使用)
│  │
│  ├─ MemoryAlerts (内存警报)
│  │  ├─ LowMemoryWarning (低内存警告)
│  │  ├─ CriticalMemory (内存临界)
│  │  └─ OutOfMemory (内存耗尽)
│  │
│  └─ MemoryProfiling (内存分析)
│     ├─ AllocationTracking (分配追踪)
│     ├─ LeakDetection (泄漏检测)
│     └─ MemoryMap (内存映射)
│
└─ 内存优化 (Memory Optimization)
   ├─ Compression (压缩)
   │  ├─ TextureCompression (纹理压缩)
   │  ├─ AudioCompression (音频压缩)
   │  └─ DataCompression (数据压缩)
   │
   ├─ Instancing (实例化)
   │  ├─ MeshInstancing (网格实例化)
   │  └─ MaterialInstancing (材质实例化)
   │
   └─ MemoryBudget (内存预算)
      ├─ SceneBudget (场景预算)
      ├─ ActorBudget (演员预算)
      └─ ResourceBudget (资源预算)
```

---

### 7.5 优先级调度 (Priority Scheduling)

```
优先级调度
├─ 优先级定义 (Priority Definition)
│  ├─ Critical (关键)
│  │  ├─ Priority: 100
│  │  ├─ 描述：必须立即加载
│  │  └─ 示例：当前节点资源
│  │
│  ├─ High (高)
│  │  ├─ Priority: 75
│  │  ├─ 描述：应尽快加载
│  │  └─ 示例：下一节点资源
│  │
│  ├─ Medium (中)
│  │  ├─ Priority: 50
│  │  ├─ 描述：可以稍后加载
│  │  └─ 示例：可能分支资源
│  │
│  ├─ Low (低)
│  │  ├─ Priority: 25
│  │  ├─ 描述：有空闲时加载
│  │  └─ 示例：远程分支资源
│  │
│  └─ Background (后台)
│     ├─ Priority: 10
│     ├─ 描述：不紧急
│     └─ 示例：预缓存资源
│
├─ 优先级计算 (Priority Calculation)
│  ├─ StaticPriority (静态优先级)
│  │  └─ 设计时指定
│  │
│  ├─ DynamicPriority (动态优先级)
│  │  ├─ DistanceFactor (距离因子)
│  │  ├─ TimeFactor (时间因子)
│  │  ├─ ProbabilityFactor (概率因子)
│  │  └─ Formula:
│  │     Priority = Base * Distance * Time * Probability
│  │
│  └─ AdjustedPriority (调整优先级)
│     ├─ BoostPriority() (提升优先级)
│     ├─ ReducePriority() (降低优先级)
│     └─ ResetPriority() (重置优先级)
│
├─ 调度策略 (Scheduling Strategy)
│  ├─ FIFO (先进先出)
│  │  └─ 同优先级按时间顺序
│  │
│  ├─ PriorityQueue (优先队列)
│  │  └─ 按优先级排序
│  │
│  ├─ WeightedRoundRobin (加权轮询)
│  │  └─ 根据优先级分配时间片
│  │
│  └─ EarliestDeadlineFirst (最早截止时间优先)
│     └─ 根据"必须在X时刻前完成"排序
│
├─ 资源抢占 (Resource Preemption)
│  ├─ PreemptLowPriority (抢占低优先级)
│  │  ├─ 暂停低优先级加载
│  │  └─ 让出资源给高优先级
│  │
│  ├─ PreemptPolicy (抢占策略)
│  │  ├─ CanPreempt (可抢占)
│  │  ├─ NonPreemptive (不可抢占)
│  │  └─ ConditionalPreempt (条件抢占)
│  │
│  └─ ResumePreempted (恢复被抢占)
│     └─ 高优先级完成后恢复
│
└─ 调度监控 (Scheduling Monitoring)
   ├─ QueueStatistics (队列统计)
   │  ├─ QueueLength (队列长度)
   │  ├─ AverageWaitTime (平均等待时间)
   │  └─ Throughput (吞吐量)
   │
   ├─ ResourceUtilization (资源利用率)
   │  ├─ CPUUtilization (CPU利用率)
   │  ├─ IOUtilization (IO利用率)
   │  └─ MemoryUtilization (内存利用率)
   │
   └─ PerformanceAnalysis (性能分析)
      ├─ LoadLatency (加载延迟)
      ├─ SchedulingOverhead (调度开销)
      └─ OptimizationSuggestions (优化建议)
```

---

## 🎯 总结：完整框架检查清单

### 7大核心元素完整性检查

```
✅ 1. Scene Graph (场景图)
   [✓] 1.1 节点系统 (10+ 节点类型)
   [✓] 1.2 连接系统 (连接类型、验证)
   [✓] 1.3 插座系统 (输入/输出/系统插座)
   [✓] 1.4 变量系统 (定义、操作、绑定)
   [✓] 1.5 版本控制 (版本信息、兼容性)

✅ 2. Execution Stream (执行流)
   [✓] 2.1 动作通道 (30+ 动作类型，Entry结构)
   [✓] 2.2 控制通道 (请求类型，Entry结构)
   [✓] 2.3 激活通道 (激活类型，Entry结构)
   [✓] 2.4 时间管理 (时间表示、播放控制)
   [✓] 2.5 索引系统 (索引类型、查询优化)

✅ 3. Tier System (Tier系统)
   [✓] 3.1 Tier定义 (5层Tier，特性定义)
   [✓] 3.2 Tier数据 (5种TierData结构)
   [✓] 3.3 Tier堆栈 (堆栈管理、冲突解决)
   [✓] 3.4 约束应用 (5类约束：输入/摄像机/移动/UI/战斗)
   [✓] 3.5 过渡管理 (过渡阶段、曲线、中断)

✅ 4. Signal System (信号系统)
   [✓] 4.1 信号类型 (4类信号)
   [✓] 4.2 插座机制 (输入/输出/系统插座)
   [✓] 4.3 信号队列 (队列管理、策略)
   [✓] 4.4 分发器 (分发核心、路由规则)
   [✓] 4.5 调试追踪 (信号追踪、可视化)

✅ 5. Actor System (演员系统)
   [✓] 5.1 演员标识 (4类ID)
   [✓] 5.2 绑定管理 (绑定策略、验证)
   [✓] 5.3 状态管理 (5类状态、同步)
   [✓] 5.4 控制接管 (4类控制、冲突处理)
   [✓] 5.5 同步系统 (多演员、时间、事件同步)

✅ 6. Interrupt System (中断系统)
   [✓] 6.1 中断条件 (6大类，20+具体条件)
   [✓] 6.2 返回条件 (4大类条件)
   [✓] 6.3 上下文管理 (快照、操作、存储)
   [✓] 6.4 中断场景 (定义、响应、内容)
   [✓] 6.5 恢复机制 (4种策略、验证、过渡)

✅ 7. Resource Manager (资源管理)
   [✓] 7.1 资源引用 (引用类型、资源类型、依赖)
   [✓] 7.2 预加载系统 (策略、分析、执行、监控)
   [✓] 7.3 流式加载 (策略、LOD、缓冲、调度)
   [✓] 7.4 内存管理 (池、引用计数、GC、监控)
   [✓] 7.5 优先级调度 (5级优先级、计算、策略)
```

---

## 📋 实现检查清单

当你开始实现InteractiveScene系统时，可以用这个清单逐项检查：

### 最小可行版本 (MVP)
```
核心三元素：
□ 场景图：节点+连接+插座
□ 执行流：动作通道+基础时间管理
□ Tier系统：Tier1-3+基础约束
```

### 标准版本
```
核心五元素：
□ +信号系统：基础信号+队列+分发
□ +演员系统：绑定+状态+控制
□ (可选)中断系统：距离+战斗条件
```

### 完整版本
```
全部七元素：
□ +中断系统：完整条件+恢复机制
□ +资源管理：预加载+流式+内存管理
```

---

*本文档提供InteractiveScene系统的完整内容框架*
*从宏观7大元素到微观子组件的完整拆解*
*可作为设计、实现、验收的参考蓝图*

**版本**: 1.0 - Part 2
**日期**: 2026-02-09
**状态**: ✅ 框架完整
