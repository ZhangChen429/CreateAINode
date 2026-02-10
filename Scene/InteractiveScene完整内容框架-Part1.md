# InteractiveScene系统完整内容框架
> 7大核心元素的完整子元素拆解与层级结构

---

## 📋 文档说明

本文档将InteractiveScene系统的7大核心元素**完全拆解到子元素层级**，构建一个完整的内容框架，帮助你：
1. 理解每个元素的完整组成
2. 掌握各子元素的职责
3. 设计实现时有明确的检查清单

---

## 🎯 框架总览

```
InteractiveScene System (交互式场景系统)
├─ 1. Scene Graph (场景图)
│  ├─ 1.1 节点系统
│  ├─ 1.2 连接系统
│  ├─ 1.3 插座系统
│  ├─ 1.4 变量系统
│  └─ 1.5 版本控制
│
├─ 2. Execution Stream (执行流)
│  ├─ 2.1 动作通道 (Action Channel)
│  ├─ 2.2 控制通道 (Control Channel)
│  ├─ 2.3 激活通道 (Activation Channel)
│  ├─ 2.4 时间管理
│  └─ 2.5 索引系统
│
├─ 3. Tier System (Tier系统)
│  ├─ 3.1 Tier定义
│  ├─ 3.2 Tier数据
│  ├─ 3.3 Tier堆栈
│  ├─ 3.4 约束应用
│  └─ 3.5 过渡管理
│
├─ 4. Signal System (信号系统)
│  ├─ 4.1 信号类型
│  ├─ 4.2 插座机制
│  ├─ 4.3 信号队列
│  ├─ 4.4 分发器
│  └─ 4.5 调试追踪
│
├─ 5. Actor System (演员系统)
│  ├─ 5.1 演员标识
│  ├─ 5.2 绑定管理
│  ├─ 5.3 状态管理
│  ├─ 5.4 控制接管
│  └─ 5.5 同步系统
│
├─ 6. Interrupt System (中断系统)
│  ├─ 6.1 中断条件
│  ├─ 6.2 返回条件
│  ├─ 6.3 上下文管理
│  ├─ 6.4 中断场景
│  └─ 6.5 恢复机制
│
└─ 7. Resource Manager (资源管理)
   ├─ 7.1 资源引用
   ├─ 7.2 预加载系统
   ├─ 7.3 流式加载
   ├─ 7.4 内存管理
   └─ 7.5 优先级调度
```

---

## 📦 1. Scene Graph (场景图) - 完整拆解

### 1.1 节点系统 (Node System)

```
节点系统
├─ 节点基类
│  ├─ NodeId (节点唯一标识)
│  ├─ NodeName (节点名称)
│  ├─ NodeType (节点类型枚举)
│  ├─ NodeState (节点状态: Active/Inactive/Completed)
│  └─ NodeMetadata (元数据: 作者、版本、注释)
│
├─ 叙事节点
│  ├─ StartNode (场景入口)
│  │  ├─ EntryPointName (入口点名称)
│  │  └─ InitialConditions (初始条件)
│  │
│  ├─ DialogueNode (对话节点)
│  │  ├─ Speaker (说话者ID)
│  │  ├─ DialogueLine (对话内容)
│  │  │  ├─ TextContent (文本内容)
│  │  │  ├─ LocalizationKey (本地化键)
│  │  │  ├─ EmotionTag (情绪标签)
│  │  │  └─ LipsyncData (口型数据引用)
│  │  ├─ Duration (持续时间)
│  │  ├─ Skippable (是否可跳过)
│  │  └─ SubtitleSettings (字幕设置)
│  │
│  ├─ ChoiceNode (选择节点)
│  │  ├─ Choices[] (选项数组)
│  │  │  ├─ ChoiceText (选项文本)
│  │  │  ├─ ConditionCheck (显示条件)
│  │  │  ├─ ConsequenceAction (后果动作)
│  │  │  └─ TimeoutValue (超时时间)
│  │  ├─ DefaultChoice (默认选项)
│  │  └─ UILayoutHint (UI布局提示)
│  │
│  └─ SectionNode (区段节点)
│     ├─ SectionName (区段名称)
│     ├─ EntryPoint (进入点)
│     ├─ ExitPoint (退出点)
│     └─ SubGraphReference (子图引用)
│
├─ 逻辑节点
│  ├─ BranchNode (分支节点)
│  │  ├─ ConditionType (条件类型)
│  │  │  ├─ FactCheck (事实检查)
│  │  │  ├─ VariableCompare (变量比较)
│  │  │  ├─ RandomChance (随机概率)
│  │  │  └─ CustomScript (自定义脚本)
│  │  ├─ TrueOutput (真输出)
│  │  └─ FalseOutput (假输出)
│  │
│  ├─ WaitNode (等待节点)
│  │  ├─ WaitType (等待类型)
│  │  │  ├─ WaitTime (等待时间)
│  │  │  ├─ WaitEvent (等待事件)
│  │  │  └─ WaitCondition (等待条件)
│  │  └─ TimeoutBehavior (超时行为)
│  │
│  └─ LoopNode (循环节点)
│     ├─ LoopType (循环类型: Count/Condition)
│     ├─ LoopCount (循环次数)
│     ├─ BreakCondition (中断条件)
│     └─ LoopBody (循环体节点)
│
└─ 动作节点
   ├─ ActionNode (通用动作)
   │  ├─ ActionType (动作类型枚举)
   │  ├─ TargetActor (目标演员)
   │  ├─ ActionParameters (动作参数)
   │  └─ ExecutionTime (执行时刻)
   │
   ├─ TimelineNode (时间轴节点)
   │  ├─ TimelineAsset (时间轴资源)
   │  ├─ PlaybackSpeed (播放速度)
   │  └─ Bindings[] (演员绑定)
   │
   └─ ScriptNode (脚本节点)
      ├─ ScriptFunction (脚本函数)
      ├─ Parameters[] (参数列表)
      └─ ReturnHandling (返回值处理)
```

---

### 1.2 连接系统 (Connection System)

```
连接系统
├─ 连接基础
│  ├─ ConnectionId (连接唯一标识)
│  ├─ SourceNodeId (源节点)
│  ├─ SourceSocketId (源插座)
│  ├─ TargetNodeId (目标节点)
│  ├─ TargetSocketId (目标插座)
│  └─ ConnectionPriority (优先级)
│
├─ 连接类型
│  ├─ SequentialConnection (顺序连接)
│  │  └─ ExecutionOrder (执行顺序)
│  │
│  ├─ ConditionalConnection (条件连接)
│  │  ├─ Condition (条件表达式)
│  │  └─ EvaluationTiming (求值时机)
│  │
│  └─ ParallelConnection (并行连接)
│     └─ SyncPoint (同步点)
│
└─ 连接验证
   ├─ TypeCompatibility (类型兼容性检查)
   ├─ CyclicDetection (循环检测)
   └─ DeadEndDetection (死端检测)
```

---

### 1.3 插座系统 (Socket System)

```
插座系统
├─ 输入插座 (Input Socket)
│  ├─ InputSocketId
│  ├─ SocketName (插座名称)
│  ├─ SocketType (插座类型)
│  │  ├─ ExecutionSocket (执行插座)
│  │  ├─ DataSocket (数据插座)
│  │  └─ EventSocket (事件插座)
│  ├─ AllowMultipleConnections (允许多连接)
│  └─ DefaultValue (默认值)
│
├─ 输出插座 (Output Socket)
│  ├─ OutputSocketId
│  ├─ SocketName
│  ├─ SocketType
│  ├─ TargetSockets[] (目标插座数组)
│  └─ BroadcastMode (广播模式)
│
└─ 系统插座 (System Sockets)
   ├─ OnStart (开始插座)
   ├─ OnComplete (完成插座)
   ├─ OnCancel (取消插座)
   ├─ OnError (错误插座)
   └─ OnInterrupt (中断插座)
```

---

### 1.4 变量系统 (Variable System)

```
变量系统
├─ 变量定义
│  ├─ VariableId (变量标识)
│  ├─ VariableName (变量名称)
│  ├─ VariableType (变量类型)
│  │  ├─ Boolean (布尔)
│  │  ├─ Integer (整数)
│  │  ├─ Float (浮点)
│  │  ├─ String (字符串)
│  │  ├─ EntityReference (实体引用)
│  │  └─ Custom (自定义)
│  ├─ Scope (作用域)
│  │  ├─ Local (本场景)
│  │  ├─ Global (全局)
│  │  └─ Persistent (持久化)
│  └─ DefaultValue (默认值)
│
├─ 变量操作
│  ├─ Get (获取)
│  ├─ Set (设置)
│  ├─ Increment (递增)
│  ├─ Decrement (递减)
│  └─ Compare (比较)
│
└─ 变量绑定
   ├─ NodeVariableBinding (节点变量绑定)
   ├─ ActorVariableBinding (演员变量绑定)
   └─ ExternalVariableBinding (外部变量绑定)
```

---

### 1.5 版本控制 (Versioning)

```
版本控制
├─ 版本信息
│  ├─ VersionNumber (版本号)
│  ├─ CreationDate (创建日期)
│  ├─ LastModifiedDate (修改日期)
│  └─ Author (作者)
│
├─ 兼容性
│  ├─ MinimumEngineVersion (最低引擎版本)
│  ├─ DeprecatedNodes[] (废弃节点列表)
│  └─ MigrationScript (迁移脚本)
│
└─ 变更追踪
   ├─ ChangeLog[] (变更日志)
   └─ DiffData (差异数据)
```

---

## ⚡ 2. Execution Stream (执行流) - 完整拆解

### 2.1 动作通道 (Action Channel)

```
动作通道
├─ 动作记录 (Action Record) ⭐核心Entry
│  ├─ 基础信息
│  │  ├─ ActionId (动作唯一标识)
│  │  ├─ ActionDefinitionId (动作定义ID)
│  │  ├─ ActionType (动作类型)
│  │  ├─ StartTime (开始时间/毫秒)
│  │  ├─ Duration (持续时间/毫秒)
│  │  └─ Priority (优先级)
│  │
│  ├─ 目标信息
│  │  ├─ PerformerId (执行者ID)
│  │  ├─ TargetId (目标ID/可选)
│  │  └─ AffectedActors[] (影响的演员列表)
│  │
│  ├─ 参数数据
│  │  ├─ Parameters (参数字典)
│  │  ├─ BlendSettings (混合设置)
│  │  └─ InterruptSettings (中断设置)
│  │
│  └─ 执行状态
│     ├─ ExecutionState (执行状态枚举)
│     │  ├─ Pending (待执行)
│     │  ├─ Executing (执行中)
│     │  ├─ Completed (已完成)
│     │  └─ Cancelled (已取消)
│     └─ CompletionCallback (完成回调)
│
├─ 动作类型定义
│  ├─ 动画动作
│  │  ├─ PlayAnimation (播放动画)
│  │  │  ├─ AnimationAsset (动画资源)
│  │  │  ├─ BlendInTime (淡入时间)
│  │  │  ├─ BlendOutTime (淡出时间)
│  │  │  └─ LoopCount (循环次数)
│  │  │
│  │  ├─ StopAnimation (停止动画)
│  │  └─ BlendAnimation (混合动画)
│  │
│  ├─ Tier动作
│  │  ├─ SetSceneTier (设置Tier)
│  │  │  ├─ TargetTier (目标Tier)
│  │  │  ├─ TierData (Tier数据)
│  │  │  ├─ TransitionDuration (过渡时长)
│  │  │  └─ UsePlayerWorkspot (是否使用工作点)
│  │  │
│  │  ├─ ClearSceneTier (清除Tier)
│  │  └─ PushTier/PopTier (堆栈操作)
│  │
│  ├─ 摄像机动作
│  │  ├─ SetCameraParams (设置摄像机参数)
│  │  │  ├─ FOV (视野角度)
│  │  │  ├─ NearClip/FarClip (裁剪面)
│  │  │  ├─ MotionBlur (运动模糊)
│  │  │  └─ DOF (景深)
│  │  │
│  │  ├─ SetCameraConstraints (设置约束)
│  │  └─ BlendCamera (摄像机混合)
│  │
│  ├─ 音频动作
│  │  ├─ PlaySound (播放音效)
│  │  │  ├─ SoundAsset (音效资源)
│  │  │  ├─ Volume (音量)
│  │  │  ├─ Pitch (音调)
│  │  │  ├─ 3DSettings (3D音效设置)
│  │  │  └─ FadeIn/FadeOut (淡入淡出)
│  │  │
│  │  ├─ PlayDialogue (播放对话)
│  │  │  ├─ VoiceAsset (语音资源)
│  │  │  ├─ SubtitleData (字幕数据)
│  │  │  ├─ LipsyncData (口型数据)
│  │  │  └─ FacialAnimData (面部动画)
│  │  │
│  │  └─ StopSound (停止音效)
│  │
│  ├─ VFX动作
│  │  ├─ SpawnVFX (生成特效)
│  │  │  ├─ VFXAsset (特效资源)
│  │  │  ├─ SpawnLocation (生成位置)
│  │  │  ├─ AttachTo (附加到)
│  │  │  └─ AutoDestroy (自动销毁)
│  │  │
│  │  ├─ StopVFX (停止特效)
│  │  └─ ModifyVFX (修改特效)
│  │
│  ├─ 实体动作
│  │  ├─ TeleportEntity (传送实体)
│  │  │  ├─ TargetPosition (目标位置)
│  │  │  ├─ TargetRotation (目标旋转)
│  │  │  └─ UseFadeEffect (使用淡入淡出)
│  │  │
│  │  ├─ SpawnEntity (生成实体)
│  │  ├─ DestroyEntity (销毁实体)
│  │  ├─ AttachEntity (附加实体)
│  │  └─ DetachEntity (分离实体)
│  │
│  ├─ 工作点动作
│  │  ├─ UseWorkspot (使用工作点)
│  │  │  ├─ WorkspotId (工作点ID)
│  │  │  ├─ ForceBlendIn (强制混合)
│  │  │  ├─ SnapToStart (对齐到起点)
│  │  │  └─ IdleAnimation (待机动画)
│  │  │
│  │  └─ ExitWorkspot (退出工作点)
│  │
│  ├─ AI动作
│  │  ├─ AICommand (AI指令)
│  │  │  ├─ CommandType (指令类型)
│  │  │  ├─ TargetLocation (目标位置)
│  │  │  ├─ Priority (优先级)
│  │  │  └─ Timeout (超时)
│  │  │
│  │  └─ SetAIBehavior (设置AI行为)
│  │
│  ├─ Look-At动作
│  │  ├─ LookAtTarget (注视目标)
│  │  │  ├─ Target (目标)
│  │  │  ├─ Weight (权重)
│  │  │  ├─ Limits (限制角度)
│  │  │  └─ Speed (速度)
│  │  │
│  │  ├─ EnablePlayerLookAt (启用玩家注视)
│  │  └─ DisableLookAt (禁用注视)
│  │
│  ├─ 装备动作
│  │  ├─ EquipItem (装备物品)
│  │  │  ├─ ItemId (物品ID)
│  │  │  ├─ SlotType (槽位类型)
│  │  │  └─ AnimationType (动画类型)
│  │  │
│  │  ├─ UnequipItem (卸下物品)
│  │  └─ SwapItem (交换物品)
│  │
│  └─ 环境动作
│     ├─ ToggleDoor (切换门状态)
│     ├─ ToggleLight (切换灯光)
│     ├─ TriggerDevice (触发设备)
│     └─ SetEnvironmentState (设置环境状态)
│
└─ 通道管理
   ├─ RecordArray[] (记录数组)
   ├─ TimeIndex (时间索引)
   ├─ Sort() (排序方法)
   ├─ GetRecordsInRange() (范围查询)
   └─ Clear() (清空)
```

---

### 2.2 控制通道 (Control Channel)

```
控制通道
├─ 控制请求 (Control Request) ⭐核心Entry
│  ├─ 基础信息
│  │  ├─ RequestId (请求ID)
│  │  ├─ RequestType (请求类型)
│  │  ├─ Timestamp (时间戳/毫秒)
│  │  └─ Priority (优先级)
│  │
│  ├─ 目标信息
│  │  ├─ TargetNodeId (目标节点)
│  │  ├─ TargetType (目标类型)
│  │  └─ Scope (作用域)
│  │
│  └─ 操作参数
│     ├─ Operation (操作枚举)
│     │  ├─ Enable (启用)
│     │  ├─ Disable (禁用)
│     │  ├─ Activate (激活)
│     │  ├─ Deactivate (停用)
│     │  └─ Reset (重置)
│     └─ OperationData (操作数据)
│
├─ 请求类型
│  ├─ 节点控制
│  │  ├─ EnableNode (启用节点)
│  │  ├─ DisableNode (禁用节点)
│  │  └─ ResetNode (重置节点)
│  │
│  ├─ 流程控制
│  │  ├─ Play (播放)
│  │  ├─ Pause (暂停)
│  │  ├─ Stop (停止)
│  │  ├─ Resume (恢复)
│  │  └─ Restart (重新开始)
│  │
│  ├─ 跳转控制
│  │  ├─ JumpToNode (跳转到节点)
│  │  ├─ JumpToTime (跳转到时间点)
│  │  ├─ JumpToSection (跳转到区段)
│  │  └─ JumpToMarker (跳转到标记)
│  │
│  └─ 速度控制
│     ├─ SetPlaybackSpeed (设置播放速度)
│     │  ├─ SpeedMultiplier (速度倍数)
│     │  └─ BlendTime (混合时间)
│     │
│     └─ TimeScale (时间缩放)
│
└─ 通道管理
   ├─ RequestQueue[] (请求队列)
   ├─ ProcessRequest() (处理请求)
   └─ ValidateRequest() (验证请求)
```

---

### 2.3 激活通道 (Activation Channel)

```
激活通道
├─ 激活记录 (Stimulation/Activation) ⭐核心Entry
│  ├─ 基础信息
│  │  ├─ StimulationId (激活ID)
│  │  ├─ Timestamp (时间戳/毫秒)
│  │  └─ Priority (优先级)
│  │
│  ├─ 目标信息
│  │  ├─ TargetNodeId (目标节点)
│  │  ├─ NodePoint (节点点)
│  │  │  ├─ Start (开始)
│  │  │  ├─ Cancel (取消)
│  │  │  ├─ Disable (禁用)
│  │  │  ├─ Concluded (完成)
│  │  │  └─ Custom (自定义)
│  │  └─ InputSocketId (输入插座)
│  │
│  ├─ 源信息
│  │  ├─ SourceNodeId (源节点)
│  │  ├─ OutputSocketId (输出插座)
│  │  └─ TriggerReason (触发原因)
│  │
│  └─ 传播数据
│     ├─ PayloadData (负载数据)
│     ├─ PropagationDelay (传播延迟)
│     └─ BroadcastMode (广播模式)
│
├─ 激活类型
│  ├─ 直接激活 (Direct Activation)
│  │  └─ 单一节点触发
│  │
│  ├─ 广播激活 (Broadcast Activation)
│  │  └─ 多节点同时触发
│  │
│  ├─ 延迟激活 (Delayed Activation)
│  │  └─ 延迟后触发
│  │
│  └─ 条件激活 (Conditional Activation)
│     └─ 满足条件时触发
│
└─ 通道管理
   ├─ StimulationQueue[] (激活队列)
   ├─ PropagateSignal() (传播信号)
   ├─ FilterStimulations() (过滤激活)
   └─ TraceStimulation() (追踪激活路径)
```

---

### 2.4 时间管理 (Time Management)

```
时间管理
├─ 时间表示
│  ├─ SceneTime (场景时间/毫秒)
│  ├─ GameTime (游戏时间)
│  └─ RealTime (现实时间)
│
├─ 播放控制
│  ├─ CurrentTime (当前时间)
│  ├─ PlaybackSpeed (播放速度)
│  │  ├─ Pause (0x)
│  │  ├─ Slow (0.5x)
│  │  ├─ Normal (1x)
│  │  ├─ Fast (2x)
│  │  └─ Custom (自定义)
│  ├─ PlayDirection (播放方向)
│  │  ├─ Forward (正向)
│  │  └─ Backward (反向/Braindance)
│  └─ LoopMode (循环模式)
│
├─ 时间操作
│  ├─ TranslatePos() (正向平移)
│  ├─ TranslateNeg() (负向平移)
│  ├─ SetTime() (设置时间)
│  ├─ GetDuration() (获取总时长)
│  └─ Normalize() (归一化时间)
│
└─ 时间事件
   ├─ OnTimeUpdate (时间更新事件)
   ├─ OnTimeReached (到达时间点事件)
   └─ OnPlaybackEnd (播放结束事件)
```

---

### 2.5 索引系统 (Index System)

```
索引系统
├─ 索引类型
│  ├─ TimeIndex (时间索引)
│  │  ├─ BucketSize (桶大小)
│  │  ├─ Buckets[] (时间桶数组)
│  │  └─ QuickLookup (快速查找)
│  │
│  ├─ ActorIndex (演员索引)
│  │  ├─ ActorId → Records映射
│  │  └─ RangeQuery (范围查询)
│  │
│  └─ TypeIndex (类型索引)
│     └─ ActionType → Records映射
│
├─ 索引操作
│  ├─ BuildIndex() (构建索引)
│  ├─ RebuildIndex() (重建索引)
│  ├─ UpdateIndex() (更新索引)
│  └─ IsIndexed() (检查索引状态)
│
└─ 查询优化
   ├─ GetRecordsAtTime() (O(log n))
   ├─ GetRecordsInRange() (O(log n + k))
   ├─ GetRecordsByActor() (O(1) + k)
   └─ GetRecordsByType() (O(1) + k)
```

---

## 🎮 3. Tier System (Tier系统) - 完整拆解

### 3.1 Tier定义 (Tier Definitions)

```
Tier定义
├─ Tier枚举
│  ├─ Tier 0: Undefined (未定义)
│  ├─ Tier 1: Full Gameplay (完全自由)
│  ├─ Tier 2: Staged Gameplay (受限移动)
│  ├─ Tier 3: Limited Gameplay (限制移动+视角)
│  ├─ Tier 4: FPP Cinematic (极度限制)
│  └─ Tier 5: Cinematic (完全控制)
│
├─ Tier特性定义
│  ├─ AllowedActions[] (允许的动作列表)
│  │  ├─ Movement (移动)
│  │  ├─ Sprint (冲刺)
│  │  ├─ Jump (跳跃)
│  │  ├─ Crouch (蹲伏)
│  │  ├─ Combat (战斗)
│  │  ├─ Interaction (交互)
│  │  ├─ Menu (菜单)
│  │  └─ CameraControl (摄像机控制)
│  │
│  ├─ InputRestrictions (输入限制)
│  │  ├─ DisabledInputs[] (禁用输入列表)
│  │  ├─ ModifiedInputs[] (修改输入列表)
│  │  └─ InputSensitivity (输入灵敏度)
│  │
│  └─ VisualFeedback (视觉反馈)
│     ├─ UIChanges (UI变化)
│     ├─ VignetteEffect (晕影效果)
│     └─ IndicatorDisplay (指示器显示)
│
└─ Tier描述
   ├─ Name (名称)
   ├─ Description (描述)
   └─ UsageScenarios[] (使用场景列表)
```

---

### 3.2 Tier数据 (Tier Data)

```
Tier数据
├─ 基础Tier数据
│  ├─ TierDataBase
│  │  ├─ TierLevel (Tier等级)
│  │  ├─ EmptyHands (是否空手)
│  │  └─ TransitionSettings (过渡设置)
│  │     ├─ TransitionDuration (过渡时长)
│  │     ├─ EasingCurve (缓动曲线)
│  │     └─ TransitionAnimation (过渡动画)
│
├─ Tier1数据
│  └─ Tier1Data
│     └─ (无额外数据，继承基类)
│
├─ Tier2数据
│  └─ Tier2Data
│     ├─ WalkType (行走类型)
│     │  ├─ Slow (慢速)
│     │  ├─ Normal (正常)
│     │  └─ Fast (快速)
│     ├─ MaxSpeed (最大速度)
│     └─ AccelerationCurve (加速曲线)
│
├─ Tier3数据
│  └─ Tier3Data (运动受限基类)
│     ├─ UsePlayerWorkspot (使用玩家工作点)
│     ├─ CameraSettings (摄像机设置)
│     │  ├─ YawLeftLimit (左旋转限制)
│     │  ├─ YawRightLimit (右旋转限制)
│     │  ├─ PitchTopLimit (上仰角限制)
│     │  ├─ PitchBottomLimit (下俯角限制)
│     │  ├─ SpeedMultiplier (速度倍数)
│     │  └─ SoftConstraints (软约束设置)
│     │     ├─ EdgeSoftness (边缘柔和度)
│     │     └─ ResistanceCurve (阻力曲线)
│     └─ LookAtSettings (注视设置)
│        ├─ EnableLookAt (启用注视)
│        └─ LookAtTarget (注视目标)
│
├─ Tier4数据
│  └─ Tier4Data (运动受限+路径)
│     ├─ (继承Tier3所有设置)
│     ├─ SplineReference (路径引用)
│     ├─ LockToSpline (锁定到路径)
│     ├─ AllowManualControl (允许手动控制)
│     └─ CameraShakeSettings (摄像机晃动)
│
└─ Tier5数据
   └─ Tier5Data (完全控制)
      ├─ (继承Tier4所有设置)
      ├─ OverrideFOV (覆盖视野)
      ├─ CustomFOV (自定义FOV)
      ├─ ForceTPP (强制第三人称)
      └─ CinematicCamera (电影摄像机设置)
```

---

### 3.3 Tier堆栈 (Tier Stack)

```
Tier堆栈
├─ 堆栈条目 (Stack Entry)
│  ├─ TierData (Tier数据)
│  ├─ Priority (优先级)
│  ├─ OwnerId (所有者ID)
│  ├─ Timestamp (时间戳)
│  └─ Metadata (元数据)
│
├─ 堆栈管理
│  ├─ PushTier() (压入Tier)
│  │  ├─ 参数：TierData, Priority
│  │  └─ 返回：StackEntryId
│  │
│  ├─ PopTier() (弹出Tier)
│  │  ├─ 参数：StackEntryId
│  │  └─ 触发：重新计算激活Tier
│  │
│  ├─ ClearTier() (清除特定Tier)
│  │  └─ 清除指定层级的所有条目
│  │
│  └─ ClearAllTiers() (清除所有Tier)
│
├─ 激活计算
│  ├─ GetActiveTier() (获取激活Tier)
│  │  └─ 返回栈顶最高优先级Tier
│  │
│  ├─ RecalculateActiveTier() (重算激活Tier)
│  │  └─ 在堆栈变化时调用
│  │
│  └─ TierChangeEvent (Tier变化事件)
│     ├─ OldTier (旧Tier)
│     ├─ NewTier (新Tier)
│     └─ Reason (变化原因)
│
└─ 冲突解决
   ├─ PriorityComparison (优先级比较)
   ├─ TimestampTieBreak (时间戳打破平局)
   └─ ConflictResolution (冲突解决策略)
```

---

### 3.4 约束应用 (Constraint Application)

```
约束应用
├─ 输入约束 (Input Constraints)
│  ├─ InputContextSwitching (输入上下文切换)
│  │  ├─ DisableActions[] (禁用动作)
│  │  ├─ ModifyActions[] (修改动作)
│  │  └─ EnableActions[] (启用动作)
│  │
│  ├─ InputMappingOverride (输入映射覆盖)
│  │  ├─ OriginalMapping (原始映射)
│  │  ├─ OverrideMapping (覆盖映射)
│  │  └─ RestoreMapping (恢复映射)
│  │
│  └─ InputSensitivityModifier (输入灵敏度修改)
│     ├─ MouseSensitivity (鼠标灵敏度)
│     └─ GamepadSensitivity (手柄灵敏度)
│
├─ 摄像机约束 (Camera Constraints)
│  ├─ CameraClamp (摄像机钳制)
│  │  ├─ YawClamp (偏航钳制)
│  │  │  ├─ MinYaw (最小偏航)
│  │  │  ├─ MaxYaw (最大偏航)
│  │  │  ├─ SoftEdge (软边缘)
│  │  │  └─ ClampMode (钳制模式)
│  │  │     ├─ Hard (硬钳制)
│  │  │     ├─ Soft (软钳制)
│  │  │     └─ Asymptotic (渐近)
│  │  │
│  │  └─ PitchClamp (俯仰钳制)
│  │     └─ (结构同YawClamp)
│  │
│  ├─ CameraSpeedModifier (摄像机速度修改)
│  │  ├─ BaseSpeed (基础速度)
│  │  ├─ SpeedCurve (速度曲线)
│  │  └─ AccelerationFactor (加速因子)
│  │
│  └─ CameraEffects (摄像机效果)
│     ├─ FOVOverride (FOV覆盖)
│     ├─ DOFSettings (景深设置)
│     └─ MotionBlur (运动模糊)
│
├─ 移动约束 (Movement Constraints)
│  ├─ MovementSpeedLimit (移动速度限制)
│  │  ├─ MaxWalkSpeed (最大行走速度)
│  │  ├─ SprintDisabled (禁用冲刺)
│  │  └─ JumpDisabled (禁用跳跃)
│  │
│  ├─ MovementAreaRestriction (移动区域限制)
│  │  ├─ AllowedArea (允许区域)
│  │  ├─ SoftBoundary (软边界)
│  │  └─ HardBoundary (硬边界)
│  │
│  └─ MovementModeOverride (移动模式覆盖)
│     ├─ ForcedMovementMode (强制移动模式)
│     └─ MovementBlending (移动混合)
│
├─ UI约束 (UI Constraints)
│  ├─ HUDVisibility (HUD可见性)
│  │  ├─ ShowCrosshair (显示准星)
│  │  ├─ ShowHealthBar (显示血条)
│  │  ├─ ShowMinimap (显示小地图)
│  │  └─ ShowObjectives (显示目标)
│  │
│  ├─ MenuAccess (菜单访问)
│  │  ├─ AllowPauseMenu (允许暂停菜单)
│  │  ├─ AllowInventory (允许物品栏)
│  │  └─ AllowMap (允许地图)
│  │
│  └─ ContextualUI (情境UI)
│     ├─ ShowInteractionPrompts (显示交互提示)
│     └─ ShowDialogueUI (显示对话UI)
│
└─ 战斗约束 (Combat Constraints)
   ├─ WeaponRestrictions (武器限制)
   │  ├─ DisallowWeaponDraw (禁止拔武器)
   │  ├─ DisallowFiring (禁止开火)
   │  └─ DisallowWeaponSwitch (禁止切换武器)
   │
   └─ CombatModeDisable (禁用战斗模式)
```

---

### 3.5 过渡管理 (Transition Management)

```
过渡管理
├─ 过渡定义
│  ├─ TransitionId (过渡ID)
│  ├─ FromTier (源Tier)
│  ├─ ToTier (目标Tier)
│  ├─ Duration (持续时间)
│  └─ Priority (优先级)
│
├─ 过渡阶段
│  ├─ PreTransition (过渡前)
│  │  ├─ ValidateTransition (验证过渡)
│  │  ├─ PrepareResources (准备资源)
│  │  └─ NotifyListeners (通知监听器)
│  │
│  ├─ TransitionExecution (过渡执行)
│  │  ├─ BlendPhase (混合阶段)
│  │  │  ├─ BlendInputs (混合输入)
│  │  │  ├─ BlendCamera (混合摄像机)
│  │  │  └─ BlendUI (混合UI)
│  │  │
│  │  ├─ AnimationPhase (动画阶段)
│  │  │  ├─ EnterAnimation (进入动画)
│  │  │  ├─ ExitAnimation (退出动画)
│  │  │  └─ WorkspotTransition (工作点过渡)
│  │  │
│  │  └─ MaskingPhase (遮罩阶段)
│  │     ├─ VisualMask (视觉遮罩)
│  │     └─ AudioMask (音频遮罩)
│  │
│  └─ PostTransition (过渡后)
│     ├─ ApplyConstraints (应用约束)
│     ├─ UpdateState (更新状态)
│     └─ TriggerCompletionEvent (触发完成事件)
│
├─ 过渡曲线
│  ├─ EasingFunction (缓动函数)
│  │  ├─ Linear (线性)
│  │  ├─ EaseIn (缓入)
│  │  ├─ EaseOut (缓出)
│  │  ├─ EaseInOut (缓入缓出)
│  │  └─ Custom (自定义)
│  │
│  └─ InterpolationMethod (插值方法)
│     ├─ Lerp (线性插值)
│     ├─ Slerp (球面插值)
│     └─ Bezier (贝塞尔插值)
│
└─ 过渡中断
   ├─ InterruptTransition (中断过渡)
   ├─ RollbackTransition (回滚过渡)
   └─ ForceCompleteTransition (强制完成过渡)
```

---

## 📡 4. Signal System (信号系统) - 完整拆解

### 4.1 信号类型 (Signal Types)

```
信号类型
├─ 执行信号 (Execution Signal)
│  ├─ Start (开始)
│  ├─ Complete (完成)
│  ├─ Cancel (取消)
│  └─ Error (错误)
│
├─ 数据信号 (Data Signal)
│  ├─ VariableChanged (变量改变)
│  ├─ StateUpdated (状态更新)
│  └─ DataTransfer (数据传输)
│
├─ 事件信号 (Event Signal)
│  ├─ TriggerActivated (触发器激活)
│  ├─ ConditionMet (条件满足)
│  └─ CustomEvent (自定义事件)
│
└─ 系统信号 (System Signal)
   ├─ NodeEnabled (节点启用)
   ├─ NodeDisabled (节点禁用)
   ├─ GraphStateChanged (图状态改变)
   └─ ErrorOccurred (错误发生)
```

---

### 4.2 插座机制 (Socket Mechanism)

```
插座机制
├─ 输入插座 (Input Socket)
│  ├─ SocketId (插座ID)
│  ├─ SocketName (插座名称)
│  ├─ SocketType (插座类型)
│  ├─ ConnectionPolicy (连接策略)
│  │  ├─ SingleConnection (单连接)
│  │  ├─ MultipleConnections (多连接)
│  │  └─ RequiredConnection (必需连接)
│  ├─ DataType (数据类型)
│  └─ DefaultBehavior (默认行为)
│
├─ 输出插座 (Output Socket)
│  ├─ SocketId
│  ├─ SocketName
│  ├─ SocketType
│  ├─ TargetSockets[] (目标插座数组)
│  ├─ BroadcastMode (广播模式)
│  │  ├─ Sequential (顺序)
│  │  ├─ Parallel (并行)
│  │  └─ Conditional (条件)
│  └─ SignalTransform (信号变换)
│
└─ 系统插座 (System Sockets)
   ├─ 标准插座
   │  ├─ OnStart (ID: 1024)
   │  ├─ OnComplete (ID: 1025)
   │  ├─ OnDisable (ID: 1026)
   │  └─ OnTrigger (ID: 1027)
   │
   └─ 插座操作
      ├─ Connect() (连接)
      ├─ Disconnect() (断开)
      └─ Validate() (验证)
```

---

### 4.3 信号队列 (Signal Queue)

```
信号队列
├─ 队列条目 (Queue Entry)
│  ├─ SignalId (信号ID)
│  ├─ Priority (优先级)
│  ├─ Timestamp (时间戳)
│  ├─ SourceInfo (源信息)
│  │  ├─ SourceNodeId
│  │  ├─ OutputSocketId
│  │  └─ TriggerContext
│  ├─ TargetInfo (目标信息)
│  │  ├─ TargetNodeId
│  │  ├─ InputSocketId
│  │  └─ NodePoint
│  └─ PayloadData (负载数据)
│
├─ 队列管理
│  ├─ Enqueue() (入队)
│  │  ├─ 参数：Signal
│  │  └─ 排序：按优先级+时间戳
│  │
│  ├─ Dequeue() (出队)
│  │  └─ 返回：下一个待处理信号
│  │
│  ├─ Peek() (查看队首)
│  └─ Clear() (清空队列)
│
├─ 队列策略
│  ├─ FIFO (先进先出)
│  ├─ PriorityQueue (优先队列)
│  └─ CustomOrdering (自定义排序)
│
└─ 队列监控
   ├─ QueueSize (队列大小)
   ├─ AverageWaitTime (平均等待时间)
   └─ DroppedSignals (丢弃信号数)
```

---

### 4.4 分发器 (Dispatcher)

```
分发器
├─ 分发核心
│  ├─ DispatchSignal() (分发信号)
│  │  ├─ 查找目标节点
│  │  ├─ 验证目标插座
│  │  ├─ 传递负载数据
│  │  └─ 触发节点激活
│  │
│  ├─ BroadcastSignal() (广播信号)
│  │  └─ 多目标并行分发
│  │
│  └─ ConditionalDispatch() (条件分发)
│     └─ 根据条件选择目标
│
├─ 路由规则
│  ├─ DirectRouting (直接路由)
│  │  └─ 一对一映射
│  │
│  ├─ BroadcastRouting (广播路由)
│  │  └─ 一对多映射
│  │
│  └─ ConditionalRouting (条件路由)
│     └─ 基于条件分支
│
├─ 传播控制
│  ├─ PropagationDelay (传播延迟)
│  ├─ PropagationDepthLimit (传播深度限制)
│  └─ CycleDetection (循环检测)
│
└─ 错误处理
   ├─ TargetNotFound (目标未找到)
   ├─ InvalidSocket (无效插座)
   ├─ PropagationTimeout (传播超时)
   └─ CyclicReference (循环引用)
```

---

### 4.5 调试追踪 (Debug Tracing)

```
调试追踪
├─ 信号追踪
│  ├─ SignalLog[] (信号日志)
│  │  ├─ SignalId
│  │  ├─ Timestamp
│  │  ├─ SourceNode
│  │  ├─ TargetNode
│  │  ├─ PayloadData
│  │  └─ ProcessingTime
│  │
│  ├─ TracePath() (追踪路径)
│  │  └─ 显示完整信号传播链
│  │
│  └─ TraceHistory() (追踪历史)
│     └─ 查看历史信号记录
│
├─ 可视化
│  ├─ GraphVisualization (图可视化)
│  │  ├─ HighlightActiveNodes (高亮激活节点)
│  │  ├─ ShowSignalFlow (显示信号流)
│  │  └─ AnimateSignalPropagation (动画信号传播)
│  │
│  └─ Timeline V可视化 (时间轴可视化)
│     └─ 在时间轴上显示信号事件
│
└─ 统计信息
   ├─ SignalCount (信号总数)
   ├─ AveragePropagationTime (平均传播时间)
   ├─ FailedSignals (失败信号数)
   └─ MostActiveNode (最活跃节点)
```

---

## 🎭 5. Actor System (演员系统) - 完整拆解

### 5.1 演员标识 (Actor Identification)

```
演员标识
├─ 标识类型
│  ├─ ActorId (演员ID)
│  │  └─ 标识NPC和玩家
│  │
│  ├─ PropId (道具ID)
│  │  └─ 标识可交互物体
│  │
│  ├─ PerformerId (执行者ID)
│  │  └─ ActorId或PropId的联合
│  │
│  └─ EntityId (实体ID)
│     └─ 游戏世界中的实体引用
│
├─ 演员类型
│  ├─ Player (玩家)
│  │  ├─ PlayerCharacter
│  │  └─ PlayerVehicle
│  │
│  ├─ NPC (非玩家角色)
│  │  ├─ MainCharacter (主要角色)
│  │  ├─ SupportingCharacter (配角)
│  │  └─ BackgroundCharacter (背景角色)
│  │
│  ├─ Prop (道具)
│  │  ├─ InteractiveProp
│  │  └─ StaticProp
│  │
│  └─ Vehicle (载具)
│     ├─ PlayerVehicle
│     └─ NPCVehicle
│
└─ 标识管理
   ├─ IdRegistry (ID注册表)
   ├─ IdResolver (ID解析器)
   └─ IdValidator (ID验证器)
```

---

### 5.2 绑定管理 (Binding Management)

```
绑定管理
├─ 演员绑定 (Actor Binding)
│  ├─ BindingEntry (绑定条目)
│  │  ├─ LogicalActorId (逻辑演员ID)
│  │  ├─ PhysicalEntityId (物理实体ID)
│  │  ├─ BindingType (绑定类型)
│  │  │  ├─ Direct (直接绑定)
│  │  │  ├─ Reference (引用绑定)
│  │  │  └─ Dynamic (动态绑定)
│  │  ├─ BindingState (绑定状态)
│  │  │  ├─ Unbound (未绑定)
│  │  │  ├─ Binding (绑定中)
│  │  │  ├─ Bound (已绑定)
│  │  │  └─ Failed (失败)
│  │  └─ BindingPriority (绑定优先级)
│  │
│  ├─ BindActor() (绑定演员)
│  │  ├─ 查找可用实体
│  │  ├─ 验证兼容性
│  │  ├─ 建立绑定关系
│  │  └─ 通知监听器
│  │
│  └─ UnbindActor() (解绑演员)
│     ├─ 断开绑定关系
│     ├─ 释放实体
│     └─ 清理状态
│
├─ 绑定策略
│  ├─ StaticBinding (静态绑定)
│  │  └─ 设计时指定实体
│  │
│  ├─ DynamicBinding (动态绑定)
│  │  └─ 运行时查找实体
│  │
│  ├─ ConditionalBinding (条件绑定)
│  │  └─ 根据条件选择实体
│  │
│  └─ FallbackBinding (回退绑定)
│     └─ 主目标不可用时的备选
│
└─ 绑定验证
   ├─ CompatibilityCheck (兼容性检查)
   ├─ AvailabilityCheck (可用性检查)
   └─ ConflictResolution (冲突解决)
```

---

### 5.3 状态管理 (State Management)

```
状态管理
├─ 演员状态 (Actor State)
│  ├─ 基础状态
│  │  ├─ Position (位置)
│  │  ├─ Rotation (旋转)
│  │  ├─ Scale (缩放)
│  │  └─ Visibility (可见性)
│  │
│  ├─ 动画状态
│  │  ├─ CurrentAnimation (当前动画)
│  │  ├─ AnimationTime (动画时间)
│  │  ├─ AnimationBlend (动画混合)
│  │  └─ AnimationSpeed (动画速度)
│  │
│  ├─ 物理状态
│  │  ├─ Velocity (速度)
│  │  ├─ AngularVelocity (角速度)
│  │  ├─ CollisionEnabled (碰撞启用)
│  │  └─ GravityEnabled (重力启用)
│  │
│  ├─ 游戏状态
│  │  ├─ Health (生命值)
│  │  ├─ Equipment (装备)
│  │  ├─ Buffs (增益)
│  │  ├─ Debuffs (减益)
│  │  └─ CombatState (战斗状态)
│  │
│  └─ 场景状态
│     ├─ InScene (在场景中)
│     ├─ SceneRole (场景角色)
│     ├─ CurrentAction (当前动作)
│     └─ ControlledBy (被控制)
│
├─ 状态操作
│  ├─ CaptureState() (捕获状态)
│  │  └─ 保存当前完整状态
│  │
│  ├─ RestoreState() (恢复状态)
│  │  └─ 恢复到保存的状态
│  │
│  ├─ CompareStates() (比较状态)
│  │  └─ 检测状态差异
│  │
│  └─ InterpolateState() (插值状态)
│     └─ 平滑过渡状态
│
└─ 状态同步
   ├─ SyncFrequency (同步频率)
   ├─ DeltaCompression (增量压缩)
   └─ ConflictResolution (冲突解决)
```

---

### 5.4 控制接管 (Control Takeover)

```
控制接管
├─ 接管类型
│  ├─ AIControl (AI控制)
│  │  ├─ BehaviorTreeOverride (行为树覆盖)
│  │  ├─ NavigationOverride (导航覆盖)
│  │  └─ DecisionOverride (决策覆盖)
│  │
│  ├─ AnimationControl (动画控制)
│  │  ├─ PlayAnimation (播放动画)
│  │  ├─ BlendAnimation (混合动画)
│  │  ├─ SetIK (设置IK)
│  │  └─ ApplyRagdoll (应用布娃娃)
│  │
│  ├─ MovementControl (移动控制)
│  │  ├─ SetVelocity (设置速度)
│  │  ├─ Teleport (传送)
│  │  ├─ FollowPath (跟随路径)
│  │  └─ ConstrainMovement (约束移动)
│  │
│  └─ LookAtControl (注视控制)
│     ├─ SetLookAtTarget (设置注视目标)
│     ├─ LookAtWeight (注视权重)
│     ├─ LookAtLimits (注视限制)
│     └─ LookAtSpeed (注视速度)
│
├─ 接管管理
│  ├─ TakeControl() (接管控制)
│  │  ├─ 保存原控制器状态
│  │  ├─ 禁用原控制器
│  │  ├─ 启用场景控制器
│  │  └─ 记录接管信息
│  │
│  ├─ ReleaseControl() (释放控制)
│  │  ├─ 禁用场景控制器
│  │  ├─ 恢复原控制器
│  │  ├─ 平滑过渡
│  │  └─ 清理接管记录
│  │
│  └─ ControlStatus (控制状态)
│     ├─ NormalControl (正常控制)
│     ├─ SceneControl (场景控制)
│     └─ SharedControl (共享控制)
│
└─ 冲突处理
   ├─ PriorityResolution (优先级解决)
   ├─ BlendingControl (混合控制)
   └─ ForceTakeover (强制接管)
```

---

### 5.5 同步系统 (Synchronization System)

```
同步系统
├─ 多演员同步
│  ├─ SyncGroup (同步组)
│  │  ├─ GroupId
│  │  ├─ MemberActors[]
│  │  ├─ SyncMode (同步模式)
│  │  │  ├─ Tight (紧密同步)
│  │  │  ├─ Loose (松散同步)
│  │  │  └─ LeaderFollower (主从同步)
│  │  └─ SyncPoint (同步点)
│  │
│  ├─ CreateSyncGroup() (创建同步组)
│  ├─ AddToGroup() (添加到组)
│  ├─ RemoveFromGroup() (从组移除)
│  └─ SyncGroupAction() (同步组动作)
│
├─ 时间同步
│  ├─ LocalTime (本地时间)
│  ├─ GlobalTime (全局时间)
│  ├─ TimeDelta (时间差)
│  └─ CompensateDelay() (补偿延迟)
│
└─ 事件同步
   ├─ SyncEvent (同步事件)
   ├─ WaitForSync() (等待同步)
   └─ TriggerSync() (触发同步)
```

---

*由于篇幅限制，中断系统和资源管理的详细拆解将在后续部分继续...*

继续阅读下一部分？我可以继续完成剩余的两个核心元素（中断系统和资源管理）的完整拆解。
