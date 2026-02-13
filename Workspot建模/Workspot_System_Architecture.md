# Workspot 系统完整架构文档

> 基于虚幻引擎(Unreal Engine)的 Workspot 交互系统框架设计文档

---

## 目录

1. [完整类继承体系](#1-完整类继承体系)
2. [系统架构分层](#2-系统架构分层)
3. [运行时执行流程](#3-运行时执行流程)
4. [数据流和依赖关系](#4-数据流和依赖关系)
5. [模块依赖关系](#5-模块依赖关系)
6. [状态机详细设计](#6-状态机详细设计)
7. [内存布局和对象生命周期](#7-内存布局和对象生命周期)
8. [线程和异步处理](#8-线程和异步处理)

---

## 1. 完整类继承体系

### 1.1 数据资产层

#### UWorkspotResource
**继承**: `UPrimaryDataAsset`

主要的 Workspot 数据资产类,包含:
- `UWorkspotTree* WorkspotTree` - Workspot 节点树
- `TArray<TSoftObjectPtr<AnimSequence>> AnimSet` - 动画资源集合
- `TArray<FWorkspotPropDefinition> GlobalProps` - 全局道具定义
- `TArray<FWorkspotCharacterFilter> SupportedTypes` - 支持的角色类型过滤器

**主要方法**:
- `bool ValidateResource()` - 验证资源有效性
- `UAnimSequence* GetAnimationByName(FName)` - 根据名称获取动画

#### UWorkspotTree
**继承**: `UObject`

Workspot 节点树结构:
- `UWorkspotEntry* RootEntry` - 根节点
- `FName TreeId` - 树ID
- `TMap<FName, UWorkspotEntry*> EntryCache` - 节点缓存

**主要方法**:
- `UWorkspotEntry* FindEntryById(FName)` - 根据ID查找节点
- `void ValidateAnimations()` - 验证动画
- `void BuildEntryCache()` - 构建节点缓存

---

### 1.2 节点基类层

#### UWorkspotEntry (抽象基类)
**继承**: `UObject`

所有 Workspot 节点的基类:
- `FName EntryId` - 节点ID
- `uint32 Flags` - 标志位
- `FLinearColor EditorColor` - 编辑器颜色

**主要方法**:
- `UWorkspotEntryIterator* CreateIterator(Context)` - 创建迭代器
- `bool ContainsEntry(FName)` - 是否包含指定节点
- `FString GetFriendlyName()` - 获取友好名称
- `void ForEachAnimation(Function)` - 遍历所有动画
- `bool SupportsRig(Skeleton)` - 是否支持指定骨骼

#### UWorkspotContainerEntry (抽象容器类)
**继承**: `UWorkspotEntry`

容器节点基类:
- `TArray<UWorkspotEntry*> Children` - 子节点列表
- `FName IdleAnimName` - 待机动画名称

**主要方法**:
- `void AddChild(UWorkspotEntry*)` - 添加子节点
- `void RemoveChild(int32)` - 移除子节点
- `int32 GetChildCount()` - 获取子节点数量
- `void ForEachChild(Function)` - 遍历子节点

---

### 1.3 叶子节点类

#### UWorkspotAnimClip
**继承**: `UWorkspotEntry`

基础动画片段:
- `FName AnimName` - 动画名称
- `float BlendInTime` - 淡入时间
- `float BlendOutTime` - 淡出时间
- `bool bLoopAnimation` - 是否循环
- `float PlayRate` - 播放速率
- `FName SlotName` - 动画槽名称

#### UWorkspotMotionAnimClip
**继承**: `UWorkspotAnimClip`

带位移的动画片段:
- `FVector TargetOffset` - 目标偏移
- `bool bUseRootMotion` - 使用根运动

#### UWorkspotSyncAnimClip
**继承**: `UWorkspotAnimClip`

同步动画片段:
- `FName SyncSlotName` - 同步槽名称
- `FTransform SyncOffset` - 同步偏移
- `bool bIsMaster` - 是否为主控

#### UWorkspotAnimClipWithItem
**继承**: `UWorkspotAnimClip`

带道具操作的动画片段:
- `TArray<FWorkspotItemAction> ItemActions` - 道具行为列表

#### UWorkspotEntryAnim
**继承**: `UWorkspotEntry`

进入动画节点:
- `FName AnimName` - 动画名称
- `FName IdleAnimName` - 待机动画
- `EWorkspotMovementType MovementType` - 移动类型
- `EWorkspotOrientationType OrientationType` - 朝向类型
- `bool bIsSynchronized` - 是否同步
- `float MovementSpeed` - 移动速度

#### UWorkspotExitAnim
**继承**: `UWorkspotEntry`

退出动画节点:
- `FName AnimName` - 动画名称
- `EWorkspotMovementType MovementType` - 移动类型
- `bool bStayOnNavmesh` - 保持在导航网格上
- `bool bSnapZToNavmesh` - Z轴对齐导航网格
- `FVector ExitOffset` - 退出偏移

#### UWorkspotFastExit
**继承**: `UWorkspotEntry`

快速退出节点:
- `FName AnimName` - 动画名称
- `float ForcedBlendIn` - 强制淡入时间
- `EWorkspotMovementType MovementType` - 移动类型

#### UWorkspotPauseClip
**继承**: `UWorkspotEntry`

暂停片段:
- `float TimeMin` - 最小时间
- `float TimeMax` - 最大时间
- `float BlendOutTime` - 淡出时间

#### UWorkspotTagNode
**继承**: `UWorkspotEntry`

标签节点:
- `FName Tag` - 标签名称

---

### 1.4 容器节点类

#### UWorkspotSequence
**继承**: `UWorkspotContainerEntry`

序列容器:
- `bool bLoopInfinitely` - 无限循环
- `int32 LoopCount` - 循环次数
- `EWorkspotCategory Category` - 分类

#### UWorkspotRandomList
**继承**: `UWorkspotContainerEntry`

随机列表容器:
- `int32 MinClips` - 最小片段数
- `int32 MaxClips` - 最大片段数
- `int32 DontRepeatLastAnims` - 不重复最近N个动画
- `TArray<float> Weights` - 权重列表
- `bool bNormalizeWeights` - 归一化权重

**私有方法**:
- `void NormalizeWeights()` - 归一化权重值

#### UWorkspotSelector
**继承**: `UWorkspotRandomList`

选择器容器:
- `EWorkspotSelectionMode SelectionMode` - 选择模式

#### UWorkspotReactionSequence
**继承**: `UWorkspotSequence`

反应序列容器:
- `TArray<FName> ReactionTypes` - 反应类型列表
- `FName MainEmotionalState` - 主要情绪状态
- `FName EmotionalExpression` - 情绪表达
- `float FacialKeyWeight` - 面部关键帧权重
- `float ReactionRadius` - 反应半径

**方法**:
- `bool ContainsReaction(FName)` - 是否包含指定反应

#### UWorkspotConditionalSequence
**继承**: `UWorkspotSequence`

条件序列容器:
- `ELogicalOperation MultipleConditionOperator` - 多条件逻辑运算符
- `TArray<UWorkspotCondition*> ConditionList` - 条件列表
- `UWorkspotEntry* DefaultEntry` - 默认节点

**方法**:
- `bool CheckConditions(Context)` - 检查条件

---

### 1.5 迭代器层

#### UWorkspotEntryIterator (抽象基类)
**继承**: `UObject`

节点迭代器基类:
- `UWorkspotEntry* PointedEntry` - 当前指向的节点
- `FWorkspotContext Context` - 上下文

**主要方法**:
- `bool IsReady(Context)` - 是否就绪
- `bool IsValid(Context)` - 是否有效
- `void Next(Context)` - 移动到下一个
- `bool GoTo(FName, Context)` - 跳转到指定节点
- `void GetData(OutData)` - 获取数据
- `void Reset()` - 重置

#### UWorkspotAnimClipIterator
**继承**: `UWorkspotEntryIterator`

动画片段迭代器:
- `FName CurrentAnim` - 当前动画
- `float ElapsedTime` - 已过时间

#### UWorkspotSequenceIterator
**继承**: `UWorkspotEntryIterator`

序列迭代器:
- `int32 CurrentIndex` - 当前索引
- `int32 LoopCounter` - 循环计数器
- `UWorkspotEntryIterator* CurrentChildIterator` - 当前子迭代器

#### UWorkspotRandomListIterator
**继承**: `UWorkspotEntryIterator`

随机列表迭代器:
- `int32 ClipsPlayed` - 已播放片段数
- `TArray<int32> PlayHistory` - 播放历史
- `int32 CurrentSelection` - 当前选择

**私有方法**:
- `int32 SelectRandomChild()` - 选择随机子节点
- `int32 CalculateWeightedRandom()` - 计算加权随机

#### UWorkspotReactionIterator
**继承**: `UWorkspotEntryIterator`

反应迭代器:
- `FName ActiveReaction` - 活跃反应
- `bool bInReaction` - 是否在反应中
- `UWorkspotEntryIterator* SavedIterator` - 保存的迭代器

**方法**:
- `bool CheckReactionTrigger()` - 检查反应触发

#### UWorkspotConditionalIterator
**继承**: `UWorkspotEntryIterator`

条件迭代器:
- `UWorkspotEntryIterator* SelectedBranch` - 选择的分支

**私有方法**:
- `int32 EvaluateConditions()` - 评估条件

---

### 1.6 运行时系统层

#### UWorkspotSubsystem
**继承**: `UGameInstanceSubsystem`

Workspot 全局管理子系统:
- `TArray<UWorkspotInstance*> ActiveInstances` - 活跃实例列表
- `TMap<FGuid, UWorkspotInstance*> InstanceLookup` - 实例查找表
- `TArray<FWorkspotCommandEntry> CommandQueue` - 命令队列
- `UWorkspotSynchronizer* Synchronizer` - 同步器
- `UWorkspotCallbackManager* CallbackManager` - 回调管理器

**主要方法**:
- `UWorkspotInstance* SetupWorkspot(Actor, Resource, Transform)` - 设置 Workspot
- `bool SendCommand(EntityId, Command, Data)` - 发送命令
- `void Tick(DeltaTime)` - 更新
- `bool IsActorInWorkspot(Actor)` - Actor 是否在 Workspot 中
- `UWorkspotInstance* GetWorkspotInstance(Actor)` - 获取实例

**私有方法**:
- `void ProcessCommandQueue()` - 处理命令队列
- `void UpdateInstances(DeltaTime)` - 更新实例
- `void PruneInvalidInstances()` - 清理无效实例

#### UWorkspotInstance
**继承**: `UObject`

Workspot 运行时实例:
- `UWorkspotResource* WorkspotResource` - 资源引用
- `UWorkspotEntryIterator* CurrentIterator` - 当前迭代器
- `AActor* OwnerActor` - 所有者 Actor
- `EWorkspotState CurrentState` - 当前状态
- `FTransform WorkspotTransform` - Workspot 变换
- `float StateTimer` - 状态计时器
- `TArray<FWorkspotStateSnapshot> StateStack` - 状态栈

**主要方法**:
- `bool Update(DeltaTime)` - 更新
- `void ProcessCommand(Command, Data)` - 处理命令
- `FName GetCurrentEntryId()` - 获取当前节点ID
- `bool IsInState(State)` - 是否在指定状态

**私有方法**:
- `void TransitionToState(NewState)` - 状态转换
- `void PlayCurrentAnimation()` - 播放当前动画
- `void HandleMovement(DeltaTime)` - 处理移动

#### UWorkspotSynchronizer
**继承**: `UObject`

Workspot 同步管理器:
- `TMap<FName, FWorkspotSyncBinding> ActiveBindings` - 活跃绑定
- `TArray<FWorkspotSyncSlot> RegisteredSlots` - 注册的槽位

**主要方法**:
- `bool BindMasterSlave(Master, Slave, SlotName)` - 绑定主从
- `void UnbindSlot(EntityId, SlotName)` - 解绑槽位
- `void SyncUpdate(DeltaTime)` - 同步更新
- `FTransform GetSyncTransform(Master, SlotName)` - 获取同步变换

**私有方法**:
- `void UpdateSlaveTransform(Binding)` - 更新从属变换
- `void SynchronizeAnimTime(Binding)` - 同步动画时间

#### UWorkspotCallbackManager
**继承**: `UObject`

回调管理器:
- `TArray<TScriptInterface<IWorkspotListener>> Listeners` - 监听器列表

**主要方法**:
- `void RegisterListener(Listener)` - 注册监听器
- `void UnregisterListener(Listener)` - 注销监听器
- `void NotifyWorkspotStarted(EntityId, Resource)` - 通知 Workspot 开始
- `void NotifyWorkspotEnded(EntityId)` - 通知 Workspot 结束
- `void NotifyAnimationChanged(EntityId, EntryId)` - 通知动画改变
- `void NotifyReactionTriggered(EntityId, ReactionType)` - 通知反应触发

---

### 1.7 组件层

#### UWorkspotComponent
**继承**: `USceneComponent`

场景组件:
- `UWorkspotResource* WorkspotResource` - Workspot 资源
- `FName SyncSlotName` - 同步槽名称
- `bool bAutoActivate` - 自动激活
- `bool bShowDebugInfo` - 显示调试信息
- `FLinearColor DebugColor` - 调试颜色

**主要方法**:
- `bool RequestUse(Actor)` - 请求使用
- `void ReleaseWorkspot(Actor)` - 释放 Workspot
- `FTransform GetWorkspotTransform()` - 获取变换
- `void OnComponentCreated()` - 组件创建时

**保护方法**:
- `void DrawDebugInfo()` - 绘制调试信息

---

### 1.8 条件系统

#### UWorkspotCondition (抽象基类)
**继承**: `UObject`

条件基类:
- `EWorkspotConditionTest TestMode` - 测试模式

**主要方法**:
- `bool Evaluate(Context)` - 评估条件

#### UWorkspotRigCondition
**继承**: `UWorkspotCondition`

骨骼条件:
- `TSoftObjectPtr<USkeleton> RequiredSkeleton` - 需要的骨骼

#### UWorkspotGenderCondition
**继承**: `UWorkspotCondition`

性别条件:
- `EWorkspotGender RequiredGender` - 需要的性别

#### UWorkspotBodyTypeCondition
**继承**: `UWorkspotCondition`

体型条件:
- `EWorkspotBodyType RequiredBodyType` - 需要的体型

#### UWorkspotTagCondition
**继承**: `UWorkspotCondition`

标签条件:
- `FGameplayTag RequiredTag` - 需要的标签

---

## 2. 系统架构分层

### 2.1 编辑器层 (Editor Layer)

- **Workspot Asset Editor** - 资产可视化编辑器
- **Node Graph Editor** - 节点图编辑器
- **Animation Preview** - 动画预览器
- **Validation Tools** - 验证工具
- **Debug Visualizer** - 调试可视化

### 2.2 资源层 (Resource Layer)

- **UWorkspotResource** - 主数据资产
- **UWorkspotTree** - 节点树结构
- **Animation Assets** - 动画资源引用
- **Prop Definitions** - 道具定义
- **Character Filters** - 角色过滤器

### 2.3 定义层 (Definition Layer)

- **Node Types** - 节点类型定义
- **Container Nodes** - 容器节点
- **Leaf Nodes** - 叶子节点
- **Condition System** - 条件系统
- **Item Actions** - 道具行为

### 2.4 系统层 (System Layer)

- **UWorkspotSubsystem** - 全局管理子系统
- **UWorkspotSynchronizer** - 同步管理器
- **UWorkspotCallbackManager** - 回调管理器
- **Command Queue** - 命令队列
- **Instance Pool** - 实例池

### 2.5 实例层 (Instance Layer)

- **UWorkspotInstance** - 运行时实例
- **State Machine** - 状态机
- **Iterator Stack** - 迭代器栈
- **Animation Controller** - 动画控制器
- **Movement Controller** - 移动控制器

### 2.6 迭代器层 (Iterator Layer)

- **Entry Iterator** - 节点迭代器基类
- **Sequence Iterator** - 序列迭代器
- **Random Iterator** - 随机迭代器
- **Reaction Iterator** - 反应迭代器
- **Conditional Iterator** - 条件迭代器

### 2.7 组件层 (Component Layer)

- **UWorkspotComponent** - 场景组件
- **Sync Slot** - 同步槽
- **Debug Draw** - 调试绘制

### 2.8 执行层 (Execution Layer)

- **Animation System** - 动画系统
- **Movement System** - 移动系统
- **Item System** - 道具系统
- **AI System** - AI系统
- **Quest System** - 任务系统

### 2.9 引擎层 (Engine Layer)

- **Anim Instance** - 动画实例
- **Skeletal Mesh** - 骨骼网格
- **Character Movement** - 角色移动
- **Navigation System** - 导航系统

---

## 3. 运行时执行流程

### 3.1 阶段1: 初始化 Workspot

1. **AI Controller** 调用 `WorkspotSubsystem::SetupWorkspot(Actor, Resource, Transform)`
2. **WorkspotSubsystem** 获取 Actor 的 WorkspotComponent
3. **WorkspotSubsystem** 创建 WorkspotInstance
4. **WorkspotInstance** 初始化资源、Actor、Transform
5. **WorkspotInstance** 获取根节点 (RootEntry)
6. **WorkspotEntry** 创建迭代器 `CreateIterator(Context)`
7. **EntryIterator** 检查条件 (Rig/Gender/Body Type)
8. 返回 Instance ID 给 AI Controller

### 3.2 阶段2: 开始播放

1. **AI Controller** 发送 `CMD_Play` 命令
2. **WorkspotSubsystem** 添加命令到队列
3. **WorkspotInstance** 处理 `CMD_Play` 命令
4. **WorkspotInstance** 转换到 `MovingToEntry` 状态

**如果有 EntryAnim:**
5. **EntryIterator** 跳转到 EntryAnimId
6. **EntryIterator** 获取节点数据 (AnimName, Transform 等)
7. **Movement System** 移动到 Workspot 位置
8. **Animation System** 播放 EntryAnim
9. 循环更新位置直到到达目标
10. 动画完成,转换到 `InWorkspot` 状态

### 3.3 阶段3: 主循环执行

**循环更新流程:**

1. **WorkspotSubsystem** 调用 `Instance::Update(DeltaTime)`
2. **WorkspotInstance** 更新状态计时器

**序列节点 (Sequence Node):**
- 当前索引递增
- 获取下一个子节点

**随机列表节点 (RandomList Node):**
- 基于权重和历史选择随机节点
- 更新播放历史

**反应节点 (Reaction Node):**
- 检查反应触发条件
- 如果触发,保存当前状态并跳转到反应

**条件节点 (Conditional Node):**
- 评估条件
- 根据结果选择分支

3. 创建子迭代器
4. 获取动画数据
5. 播放下一个动画

**同步模式 (Synchronized):**

**主控 (Master):**
- 更新从属位置
- 同步从属动画时间

**从属 (Slave):**
- 获取主控变换
- 应用同步偏移
- 设置世界变换

### 3.4 阶段4: 退出 Workspot

1. **AI Controller** 发送 `CMD_SlowExit` 命令
2. **WorkspotInstance** 处理退出命令
3. **EntryIterator** 跳转到 ExitAnimId

**如果找到 ExitAnim:**
4. 获取退出动画数据
5. 播放退出动画
6. 移动离开 Workspot

**如果没有 ExitAnim:**
4. 执行快速退出

7. 动画完成后转换到 `Completed` 状态
8. 通知 Subsystem 完成
9. **WorkspotSubsystem** 移除实例
10. **CallbackManager** 触发 `OnWorkspotEnded` 回调

---

## 4. 数据流和依赖关系

### 4.1 创作阶段 (Authoring)

- **Asset Editor** - 资产编辑器
- **Node Graph** - 节点图编辑
- **Anim Config** - 动画配置
- **Conditions** - 条件设置

### 4.2 编译阶段 (Compilation)

- **Serialization** - 资源序列化
- **Condition Compile** - 条件编译
- **Anim References** - 动画引用解析
- **Validation** - 验证检查

### 4.3 打包阶段 (Packaging)

- **WorkspotResource** (.uasset)
- **Anim Assets** - 动画资源
- **Dependencies** - 依赖打包

### 4.4 加载阶段 (Loading)

- **Async Loader** - 异步加载器
- **Resource Cache** - 资源缓存
- **Anim Streaming** - 动画流式加载

### 4.5 初始化阶段 (Initialization)

- **WorkspotSubsystem** 初始化
- **Register Components** - 注册组件
- **Preload Resources** - 预加载资源

### 4.6 运行时阶段 (Runtime)

- **Create Instance** - 创建实例
- **Build Iterator** - 构建迭代器
- **Check Conditions** - 检查条件
- **Execution Loop** - 执行循环

### 4.7 执行细节 (Execution Details)

- **Play Animation** - 播放动画
- **Move Character** - 角色移动
- **Manage Props** - 道具管理
- **Sync Control** - 同步控制
- **Trigger Reactions** - 触发反应

### 4.8 引擎交互 (Engine Interaction)

- **AnimInstance** - 动画实例
- **CharacterMovement** - 移动组件
- **SkeletalMesh** - 骨骼网格
- **NavSystem** - 导航系统

---

## 5. 模块依赖关系

### 5.1 WorkspotCore Module

**核心数据和定义模块**

- **Data Assets**
  - UWorkspotResource
  - UWorkspotTree
  
- **Node Definitions**
  - Entry Classes
  
- **Iterator System**
  - Iterator Classes
  
- **Condition System**
  - Condition Classes

**依赖**: UE CoreUObject

---

### 5.2 WorkspotRuntime Module

**运行时系统模块**

- **Subsystem**
  - UWorkspotSubsystem
  
- **Instance Manager**
  - UWorkspotInstance
  
- **Synchronizer**
  - UWorkspotSynchronizer
  
- **Callback System**
  - UWorkspotCallbackManager
  
- **Command Queue**
  - Command Processing

**依赖**: WorkspotCore, UE Engine

---

### 5.3 WorkspotComponent Module

**组件模块**

- **Scene Component**
  - UWorkspotComponent
  
- **Debug Visualizer**
  - Debug Drawing
  
- **Sync Slot**
  - Sync Definitions

**依赖**: WorkspotCore, WorkspotRuntime, UE Engine

---

### 5.4 WorkspotEditor Module

**编辑器模块** (仅编辑器)

- **Asset Editor**
  - FWorkspotAssetEditor
  
- **Graph Editor**
  - Node Graph UI
  
- **Details Customization**
  - Property Panels
  
- **Validation Tools**
  - Asset Validation
  
- **Preview Viewport**
  - Animation Preview

**依赖**: WorkspotCore, UE UnrealEd

---

### 5.5 WorkspotAnimation Module

**动画控制模块**

- **Animation Controller**
  - Anim Playback
  
- **Blend Manager**
  - Blend Control
  
- **Sync Manager**
  - Anim Sync

**依赖**: WorkspotCore, UE AnimGraphRuntime

---

### 5.6 WorkspotAI Module

**AI 集成模块**

- **BTTask_UseWorkspot**
  - Behavior Tree Task
  
- **EQS Integration**
  - EQS Generator/Test
  
- **AI Controller Ext**
  - AI Extensions

**依赖**: WorkspotRuntime, UE AIModule

---

### 5.7 UE Core Modules

**虚幻引擎核心模块**

- Engine
- CoreUObject
- AnimGraphRuntime
- AIModule
- NavigationSystem
- UnrealEd (编辑器)

---

## 6. 状态机详细设计

### 6.1 状态枚举

```cpp
enum class EWorkspotState
{
    Uninitialized,           // 未初始化
    Initializing,            // 初始化中
    LoadingResources,        // 加载资源
    ValidatingConditions,    // 验证条件
    Ready,                   // 就绪
    Failed,                  // 失败
    WaitingForCommand,       // 等待命令
    MovingToEntry,           // 移动到进入点
    PlayingEntryAnim,        // 播放进入动画
    InWorkspot,              // 在 Workspot 中
    PlayingSequence,         // 播放序列
    Paused,                  // 暂停
    CheckingExit,            // 检查退出
    PlayingSlowExit,         // 播放慢速退出
    PlayingFastExit,         // 播放快速退出
    TeleportExit,            // 瞬移退出
    MovingAway,              // 移动离开
    Completed,               // 完成
    Stopped,                 // 停止
    Disposed,                // 已释放
    Synchronized             // 同步状态
};
```

### 6.2 状态转换表

| 从状态 | 到状态 | 触发条件 |
|--------|--------|----------|
| Uninitialized | Initializing | SetupWorkspot() |
| Initializing | LoadingResources | 开始加载 |
| LoadingResources | ValidatingConditions | 资源就绪 |
| ValidatingConditions | Ready | 验证通过 |
| ValidatingConditions | Failed | 验证失败 |
| Ready | WaitingForCommand | 初始化完成 |
| WaitingForCommand | MovingToEntry | CMD_Play + 有 EntryAnim |
| WaitingForCommand | InWorkspot | CMD_Play + 无 EntryAnim |
| MovingToEntry | PlayingEntryAnim | 到达位置 |
| PlayingEntryAnim | InWorkspot | 进入完成 |
| InWorkspot | PlayingSequence | 开始主循环 |
| PlayingSequence | PlayingSequence | 无限循环 |
| PlayingSequence | CheckingExit | 停止命令 |
| InWorkspot | Paused | CMD_Pause |
| Paused | InWorkspot | CMD_Unpause |
| CheckingExit | PlayingSlowExit | CMD_SlowExit + 有 ExitAnim |
| CheckingExit | PlayingFastExit | CMD_FastExit |
| CheckingExit | TeleportExit | 无退出定义 |
| PlayingSlowExit | MovingAway | 退出动画完成 |
| PlayingFastExit | MovingAway | 快速退出完成 |
| MovingAway | Completed | 离开 Workspot |
| TeleportExit | Completed | 瞬间退出 |
| Paused | Stopped | CMD_Stop |
| PlayingSequence | Stopped | CMD_Stop (强制) |
| Completed | Disposed | 清理 |
| Stopped | Disposed | 清理 |
| Failed | Disposed | 清理 |
| InWorkspot | Synchronized | 同步绑定 |
| Synchronized | InWorkspot | 同步解绑 |

### 6.3 PlayingSequence 子状态

在 `PlayingSequence` 状态下,有以下子状态:

1. **PlayingAnimation** - 播放动画
2. **CheckingNext** - 检查下一个
3. **SelectingSequence** - 选择序列节点
4. **SelectingRandom** - 选择随机节点
5. **CheckingReaction** - 检查反应
6. **HandlingReaction** - 处理反应
7. **EvaluatingCondition** - 评估条件

### 6.4 Synchronized 子状态

在 `Synchronized` 状态下,分为:

1. **Master** - 主控
   - SyncUpdate - 同步更新
   - UpdateSlavePosition - 更新从属位置
   - SyncAnimTime - 同步动画时间

2. **Slave** - 从属
   - WaitingForMaster - 等待主控
   - 应用同步变换

### 6.5 状态注释

- **MovingToEntry**: 使用 NavSystem 寻路,播放 EntryAnim
- **PlayingSequence**: 主要执行循环,支持中断和反应
- **Synchronized**: Master 控制动画时间,Slave 跟随位置和时间
- **Completed**: 触发 OnWorkspotEnded,清理所有资源

---

## 7. 内存布局和对象生命周期

### 7.1 持久内存 (Persistent Memory)

**UWorkspotResource**
- 类型: Data Asset
- 生命周期: 游戏运行期间
- 说明: 主数据资产,常驻内存

**UWorkspotTree**
- 类型: Node Tree
- 生命周期: 与 Resource 相同
- 说明: 节点树结构

**Animation Assets**
- 类型: Soft References
- 生命周期: 按需加载
- 说明: 软引用,流式加载

---

### 7.2 子系统内存 (Subsystem Memory)

**UWorkspotSubsystem**
- 类型: Singleton
- 生命周期: GameInstance
- 说明: 全局单例

**Instance Pool**
- 类型: TArray
- 生命周期: 动态增长
- 说明: 实例池

**Command Queue**
- 类型: TArray
- 生命周期: 每帧清空
- 说明: 命令队列

**Synchronizer**
- 类型: UObject
- 生命周期: GameInstance
- 说明: 同步管理器

---

### 7.3 实例内存 (Instance Memory)

**UWorkspotInstance**
- 类型: UObject
- 生命周期: Workspot 活跃期间
- 说明: 运行时实例

**Iterator Stack**
- 类型: 指针数组
- 生命周期: 与 Instance 相同
- 说明: 迭代器栈

**State Data**
- 类型: Struct
- 生命周期: 与 Instance 相同
- 说明: 状态数据

**Animation Data**
- 类型: Cached
- 生命周期: 与 Instance 相同
- 说明: 缓存的动画数据

---

### 7.4 临时内存 (Temporary Memory)

**Command Data**
- 类型: UniquePtr
- 生命周期: 单帧
- 说明: 命令数据

**Iterator Results**
- 类型: Stack Allocated
- 生命周期: 函数作用域
- 说明: 迭代器结果

**Condition Context**
- 类型: Stack Allocated
- 生命周期: 检查期间
- 说明: 条件上下文

**Debug Data**
- 类型: Transient
- 生命周期: Debug 模式
- 说明: 调试数据

---

### 7.5 组件内存 (Component Memory)

**UWorkspotComponent**
- 类型: SceneComponent
- 生命周期: Actor 生命周期
- 说明: 场景组件

**Debug Geometry**
- 类型: 临时
- 生命周期: 每帧重建
- 说明: 调试几何体

---

## 8. 线程和异步处理

### 8.1 游戏线程 (Game Thread)

- **Tick Update** - 主循环
- **Command Processing** - 命令处理
- **State Updates** - 状态更新
- **Callback Dispatch** - 回调分发

### 8.2 动画线程 (Animation Thread)

- **Anim Evaluation** - 动画求值
- **Blend Processing** - 混合处理
- **Bone Updates** - 骨骼更新

### 8.3 加载线程 (Loading Thread)

- **Asset Streaming** - 资源流式加载
- **Animation Loading** - 动画加载
- **Dependency Resolution** - 依赖解析

### 8.4 AI 线程 (AI Thread)

- **Behavior Tree** - 行为树更新
- **EQS Query** - EQS 查询
- **Pathfinding** - 寻路

### 8.5 同步点 (Sync Points)

- **Frame Barrier** - 帧同步
- **Resource Ready** - 资源就绪
- **Animation Complete** - 动画完成

### 8.6 线程交互流程

1. **游戏线程** → 异步请求 → **动画线程**
   - 动画线程处理动画求值、混合、骨骼更新
   - 完成后通知 → **Animation Complete 同步点**

2. **游戏线程** → 资源请求 → **加载线程**
   - 加载线程处理资源流式加载、动画加载、依赖解析
   - 加载完成 → **Resource Ready 同步点**

3. **AI 线程** → Workspot 请求 → **游戏线程**
   - 游戏线程处理命令
   - AI 响应 ← **游戏线程**
   - 寻路结果 → **游戏线程**

4. **Frame Barrier** → **游戏线程** (新帧开始)

---

## 附录

### A. 枚举类型定义

```cpp
// 工作点类别
enum class EWorkspotCategory
{
    Idle,           // 待机
    Sitting,        // 坐
    Standing,       // 站
    Leaning,        // 靠
    Working,        // 工作
    Entertainment,  // 娱乐
    Custom          // 自定义
};

// 移动类型
enum class EWorkspotMovementType
{
    Teleport,       // 瞬移
    Walk,           // 走
    Run,            // 跑
    Slide,          // 滑动
    Custom          // 自定义
};

// 朝向类型
enum class EWorkspotOrientationType
{
    TowardsObject,  // 朝向对象
    MatchObject,    // 匹配对象
    KeepCurrent,    // 保持当前
    Custom          // 自定义
};

// 选择模式
enum class EWorkspotSelectionMode
{
    Random,         // 随机
    Sequential,     // 顺序
    Weighted,       // 加权
    Conditional     // 条件
};

// 逻辑运算符
enum class ELogicalOperation
{
    And,            // 与
    Or,             // 或
    Xor,            // 异或
    Not             // 非
};

// 条件测试模式
enum class EWorkspotConditionTest
{
    Equal,          // 等于
    NotEqual,       // 不等于
    GreaterThan,    // 大于
    LessThan,       // 小于
    Contains,       // 包含
    NotContains     // 不包含
};

// 性别
enum class EWorkspotGender
{
    Male,           // 男性
    Female,         // 女性
    Any             // 任意
};

// 体型
enum class EWorkspotBodyType
{
    Small,          // 小型
    Medium,         // 中型
    Large,          // 大型
    Any             // 任意
};
```

### B. 结构体定义

```cpp
// 工作点上下文
struct FWorkspotContext
{
    AActor* Actor;                      // 执行 Actor
    USkeletalMeshComponent* Mesh;       // 骨骼网格组件
    FTransform Transform;               // 变换
    TMap<FName, UObject*> Properties;   // 属性
};

// 工作点道具定义
struct FWorkspotPropDefinition
{
    FName PropId;                       // 道具ID
    TSubclassOf<AActor> PropClass;      // 道具类
    FTransform RelativeTransform;       // 相对变换
    FName AttachSocket;                 // 附加插槽
};

// 工作点角色过滤器
struct FWorkspotCharacterFilter
{
    TSoftObjectPtr<USkeleton> Skeleton; // 骨骼
    EWorkspotGender Gender;             // 性别
    EWorkspotBodyType BodyType;         // 体型
    FGameplayTagContainer RequiredTags; // 需要的标签
};

// 工作点道具行为
struct FWorkspotItemAction
{
    FName ActionId;                     // 行为ID
    float TriggerTime;                  // 触发时间
    FName ItemId;                       // 道具ID
    FName ActionType;                   // 行为类型
};

// 工作点同步绑定
struct FWorkspotSyncBinding
{
    FGuid MasterId;                     // 主控ID
    FGuid SlaveId;                      // 从属ID
    FName SlotName;                     // 槽位名称
    FTransform Offset;                  // 偏移
};

// 工作点同步槽位
struct FWorkspotSyncSlot
{
    FName SlotName;                     // 槽位名称
    FTransform RelativeTransform;       // 相对变换
};

// 工作点状态快照
struct FWorkspotStateSnapshot
{
    EWorkspotState State;               // 状态
    float TimeInState;                  // 状态时间
    FName CurrentEntryId;               // 当前节点ID
};

// 工作点命令条目
struct FWorkspotCommandEntry
{
    FGuid EntityId;                     // 实体ID
    FName Command;                      // 命令
    TSharedPtr<void> Data;              // 数据
};
```

### C. 接口定义

```cpp
// 工作点监听器接口
class IWorkspotListener
{
public:
    virtual void OnWorkspotStarted(FGuid EntityId, UWorkspotResource* Resource) = 0;
    virtual void OnWorkspotEnded(FGuid EntityId) = 0;
    virtual void OnAnimationChanged(FGuid EntityId, FName EntryId) = 0;
    virtual void OnReactionTriggered(FGuid EntityId, FName ReactionType) = 0;
    virtual void OnStateChanged(FGuid EntityId, EWorkspotState NewState) = 0;
};
```

---

## 总结

Workspot 系统是一个完整的、模块化的交互动画系统框架,具有以下特点:

1. **层次清晰**: 从数据资产到运行时实例,层次分明
2. **扩展性强**: 基于继承的节点系统,易于扩展新节点类型
3. **灵活配置**: 支持序列、随机、条件、反应等多种组合方式
4. **性能优化**: 迭代器模式、对象池、异步加载等优化手段
5. **编辑器友好**: 完善的可视化编辑和调试工具
6. **多线程支持**: 合理的线程分工和同步机制
7. **AI 集成**: 与虚幻引擎 AI 系统深度集成

这个框架适用于各类需要复杂交互动画的游戏场景,如 NPC 行为、物品使用、环境交互等。
