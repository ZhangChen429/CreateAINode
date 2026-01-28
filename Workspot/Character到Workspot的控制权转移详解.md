# Character决策到Workspot接管：控制权转移完整流程

## 核心问题：什么情况下发生Character决策到Workspot的转变？

这是一个**分层系统的控制权交接**过程，涉及三个阶段：

1. **Character AI决策**："我要不要使用这个Workspot？"
2. **控制权移交**："WorkspotSystem接管Character"
3. **Workspot执行**："按照Workspot脚本播放动画"

---

## 一、转变触发的三种情况

### 情况1：AI驱动（主动使用）

**Character的行为树决策使用Workspot**

```cpp
// Character的行为树（伪代码）
BehaviorTree: NPC_Civilian
├─ Selector: "我应该做什么？"
│  ├─ Sequence: 战斗
│  │  ├─ BTTask_CheckEnemy → 检查是否有敌人
│  │  └─ BTTask_Combat → 战斗
│  │
│  ├─ BTTask_UseWorkspot ← ⚠️ 这个任务触发转变
│  │  ├─ 决策：我需要休息/吃饭/工作
│  │  ├─ 查询：附近有哪些Workspot？
│  │  ├─ 评估：哪个最合适？
│  │  └─ 执行：调用WorkspotSystem::SetupWorkspot()
│  │     ↓
│  │     【控制权移交到WorkspotSystem】
│  │
│  └─ Sequence: 巡逻
│     └─ ...
```

### 情况2：脚本驱动（任务/剧情）

**Quest/Dialogue脚本强制NPC使用Workspot**

```redscript
// RedScript示例
let npc = GameInstance.FindEntityByID(npcID);
let workspotNode = this.GetNode("restaurant_chair_01");

// ⚠️ 脚本直接命令NPC使用Workspot
WorkspotSystem.PlayInWorkspot(
    npc,
    workspotNode,
    entryAnimName = "walk_to_sit",
    workspotCategory = AISpotCategory.Restaurant
);
// → 立即触发控制权移交
```

### 情况3：玩家交互（玩家使用Workspot）

**玩家按E键坐椅子/使用ATM等**

```cpp
// 玩家交互系统伪代码
void PlayerInteractionSystem::OnUseButtonPressed()
{
    InteractableObject* obj = GetTargetInteractable();
    if (obj->HasWorkspot())
    {
        WorkspotComponent* wsComp = obj->GetWorkspotComponent();
        // ⚠️ 玩家使用Workspot
        WorkspotSystem.SetupWorkspot(
            playerEntity,
            wsComp->GetWorkspotResource()
        );
        // → 玩家控制被WorkspotSystem接管
    }
}
```

---

## 二、控制权转移的完整流程（基于代码分析）

### 阶段1：Character AI决策"使用Workspot"

#### 1.1 AI发现Workspot

```cpp
// 伪代码：BTTask_FindWorkspot
class BTTask_FindWorkspot : public BTTaskNode
{
    EBTNodeResult ExecuteTask(UBehaviorTreeComponent& OwnerComp)
    {
        // 1. 查询附近的Workspot
        TArray<WorkspotComponent*> nearbyWorkspots =
            QueryNearbyWorkspots(
                ownerLocation,
                searchRadius = 50.0f,
                filter = AISpotCategory::Rest  // 筛选类型
            );

        // 2. 评估每个Workspot
        WorkspotComponent* bestWorkspot = nullptr;
        float bestScore = -1.0f;

        for (auto ws : nearbyWorkspots)
        {
            // 检查是否可用
            if (!ws->IsAvailable())
                continue;

            // 评分：距离、舒适度、朝向等
            float score = EvaluateWorkspot(ws, ownerLocation);
            if (score > bestScore)
            {
                bestScore = score;
                bestWorkspot = ws;
            }
        }

        if (bestWorkspot)
        {
            // 3. 保存到Blackboard
            Blackboard->SetValue("TargetWorkspot", bestWorkspot);
            return EBTNodeResult::Succeeded;
        }

        return EBTNodeResult::Failed;
    }
};
```

#### 1.2 AI决策使用Workspot

```cpp
// 伪代码：BTTask_UseWorkspot
class BTTask_UseWorkspot : public BTTaskNode
{
    EBTNodeResult ExecuteTask(UBehaviorTreeComponent& OwnerComp)
    {
        // 1. 从Blackboard获取目标Workspot
        WorkspotComponent* targetWS = Blackboard->GetValue("TargetWorkspot");

        // 2. 检查前置条件
        if (!targetWS || !targetWS->IsAvailable())
            return EBTNodeResult::Failed;

        // 3. ⚠️ 关键：请求使用Workspot
        bool success = RequestWorkspotUsage(
            ownerEntity,
            targetWS
        );

        if (success)
        {
            // ✓ 控制权将被移交
            // AI行为树会被"暂停"，直到Workspot结束
            return EBTNodeResult::InProgress;
        }

        return EBTNodeResult::Failed;
    }

    void OnTaskFinished(UBehaviorTreeComponent& OwnerComp, EBTNodeResult Result)
    {
        // Workspot结束后，控制权返回
        // AI继续执行下一个节点
    }
};
```

### 阶段2：调用WorkspotSystem::SetupWorkspot()

**这是控制权转移的核心入口**

```cpp
// workspotSystem.cpp: 231-273
void WorkspotSystem::SetupWorkspot(
    const THandle<ent::Entity>& ent,          // NPC/Player实体
    const WorkspotSetupContext& setup,        // Workspot配置
    SyncedWorkspotInfo* syncInfo              // 同步信息（可选）
)
{
    // 1. 验证参数
    if (!setup.m_workspot.IsValid() || !ent)
    {
        return;  // 无效，不转移控制权
    }

    RED_SCOPE_LOCK(m_instancesLock);

    ent::EntityID ownerId = ent->GetEntityID();

    // 2. 查找是否已经有该实体的WorkspotInstance
    WorkspotInstance* instance = FindInstance_NoLock(ownerId);

    // 3. 如果已经在使用Workspot，先停止
    if (instance && instance->IsActive())
    {
        // 发送Stop命令，清理旧的Workspot
        auto dummyData = red::UniquePtr<IWorkspotCommandData>();
        SendCommandInternal_NoLock(ownerId, OriginId::Invalid(), CMD_Stop, dummyData);
        StopFromClearing_NoLock(instance);
    }
    else
    {
        // 4. 创建新的WorkspotInstance
        instance = &CreateInstance_NoLock(ent);
        // ⚠️ 此时为该NPC创建了专属的Workspot运行时实例
    }

    // 5. ⚠️ 关键：设置Workspot，加载WorkspotTree
    instance->SetupWorkspot(*this, setup, m_callbackManager);
    // 从这一刻起，NPC的动画控制权移交给WorkspotSystem

    // 6. 通知同步系统（多人同步Workspot）
    m_synchronizer.OnWorkspotSetup(ent->GetEntityID(), setup.m_workspot.m_locId, syncInfo);

    // 7. 处理缓存的命令（如果有提前发送的命令）
    for (Int32 i = (Int32)m_cachedCmd.Size() - 1; i >= 0; --i)
    {
        CachedCommand& cmd = m_cachedCmd[i];
        if (cmd.m_targetEnt == ownerId &&
            (setup.m_workspot.m_locId == cmd.m_fromSource ||
             cmd.m_fromSource == work::OriginId::Invalid()))
        {
            // 应用缓存的命令（如JumpToEntry等）
            SendCommandInternal_NoLock(ownerId, cmd.m_fromSource, cmd.m_cmd, cmd.m_data);
            m_cachedCmd.RemoveAt(i);
        }
    }
}
```

### 阶段3：WorkspotInstance::SetupWorkspot()

**实际加载WorkspotTree并创建迭代器**

```cpp
// gameWorkspotsInstance.cpp: 120-130+
void WorkspotInstance::SetupWorkspot(
    WorkspotSystem& workspotSystem,
    const WorkspotSetupContext& setup,
    WorkspotCallbackManager& callbackManager
)
{
    // 1. 加载Workspot资源
    m_workResource = setup.m_workspot.m_tree;        // WorkspotTree资源
    m_animResource = setup.m_workspot.m_animGraph;   // AnimGraph资源
    m_locId = setup.m_workspot.m_locId;              // 点位ID

    // 2. 设置通信接口
    m_commFunctions = setup.m_commFun;  // 回调函数，用于通知Character

    // 3. 查找进入动画
    m_entryAnimToUse = FindEntryAnim(setup.m_entryAnim);
    // 如果指定了entryAnimName，查找对应的EntryAnim节点ID

    // 4. 查找退出动画
    m_exitAnimToUse = FindExitAnim(setup.m_exitAnim);

    // 5. ⚠️ 创建WorkspotTree的根迭代器
    // 这是关键！从这里开始，Workspot脚本开始执行
    if (m_workResource && m_workResource->GetRootEntry())
    {
        IteratorCreationContext context = {
            m_randGen,                          // 随机数生成器
            setup.m_autoTransitionBlendTime     // 过渡混合时间
        };

        // 创建根Entry的迭代器
        m_iterator.Reset(
            m_workResource->GetRootEntry()->CreateIterator(context)
        );
        // 从此刻起，m_iterator指向WorkspotTree的开头
    }

    // 6. 初始化物品管理
    m_globalItemManager = setup.m_globalItemManager;
    m_itemOverride = setup.m_itemOverride;

    // 7. 设置裁剪空间（用于EntryPoint筛选）
    m_clippingSpace = setup.m_clippingSpace;

    // 8. 初始化状态机
    m_currentState = State::Initialized;

    // 9. 通知回调管理器
    callbackManager.OnWorkspotStarted(m_ownerID, m_locId);
}
```

### 阶段4：发送CMD_Play命令，开始执行

```cpp
// AI系统或脚本调用
WorkspotSystem::SendCommand(entityID, CMD_Play);

// ↓ WorkspotSystem处理命令

void WorkspotSystem::ProcessCommand_NoLock(CommandEntry& cmd)
{
    if (cmd.m_commands & CMD_Play)
    {
        // ⚠️ 开始播放Workspot
        cmd.m_target->Play(context);
        // 从此刻起，Workspot迭代器开始推进
    }
}

// ↓ WorkspotInstance::Play()

void WorkspotInstance::Play(CommandContext& context)
{
    if (m_currentState == State::Initialized)
    {
        // ⚠️ 状态切换到PlaybackRequest
        m_currentState = State::PlaybackRequest;

        // 如果有EntryAnim，先播放进入动画
        if (m_entryAnimToUse.IsValid())
        {
            m_iterator->GoTo(m_entryAnimToUse, iterContext);
            // 迭代器跳转到EntryAnim
        }
        else
        {
            // 否则直接开始执行主序列
            m_iterator->Next(iterContext);
        }

        // ⚠️ 通知Character系统：我要开始移动了
        m_commFunctions->MovementRequest(
            entryTransform,      // 进入点位的位置
            startLocation,       // 当前位置
            duration,            // 移动时长
            entryAnimName,       // 进入动画名
            forceTime,           // 强制播放时间
            recordFlags,         // 标记
            forceSnapToTerrain   // 是否贴地
        );
    }
}
```

---

## 三、控制权转移后的状态变化

### 3.1 Character的状态

```cpp
// Character AI行为树
BTTask_UseWorkspot::ExecuteTask()
{
    RequestWorkspotUsage(entity, workspot);

    // ⚠️ 返回InProgress，行为树暂停在此节点
    return EBTNodeResult::InProgress;

    // Character的AI决策被"冻结"
    // - 不再执行其他行为
    // - 不再响应战斗/巡逻等决策
    // - 完全由WorkspotSystem控制
}

// 等待Workspot结束...

void BTTask_UseWorkspot::OnTaskFinished(EBTNodeResult Result)
{
    // WorkspotSystem调用 m_commFunctions->OnCompleted()
    // → 通知行为树任务完成
    // → 行为树恢复，继续执行下一个节点
}
```

### 3.2 WorkspotSystem的状态

```cpp
// WorkspotSystem::Update(deltaTime)
void WorkspotSystem::Update(Float dt)
{
    for (auto& instance : m_instances)
    {
        // ⚠️ 每帧更新所有活跃的WorkspotInstance
        UpdateInstance_NoLock(instance, timeToProcess);
    }
}

// WorkspotInstance::UpdateRecord()
GlobalWorkspotTime WorkspotInstance::UpdateRecord(
    GlobalWorkspotTime timeToProcess,
    UpdateContext& context
)
{
    // 1. 检查当前迭代器是否有效
    if (!m_iterator->IsValid(condContext))
    {
        // 当前Entry已完成，推进迭代器
        m_iterator->Next(iterContext);
    }

    // 2. 提取当前Entry的数据
    WorkspotEntryData entryData;
    m_iterator->GetData(entryData);

    // 3. 应用到动画系统
    if (entryData.m_idleAnimName)
    {
        // 播放底层idle动画
        PlayIdleAnimation(entryData.m_idleAnimName);
    }
    if (entryData.m_animationName)
    {
        // 播放上层动作动画
        PlayAnimation(entryData.m_animationName);
    }

    // 4. 检查动画是否播放完毕
    if (IsAnimationFinished())
    {
        // 推进到下一个Entry
        m_iterator->Next(iterContext);
    }

    // 5. 返回下次更新时间
    return nextUpdateTime;
}
```

### 3.3 动画系统的状态

```cpp
// 动画系统接收来自WorkspotSystem的指令
AnimationControllerComponent::OnWorkspotData(WorkspotEntryData& data)
{
    // 1. 设置底层通道
    if (data.m_idleAnimName)
    {
        SetLayerAnimation(
            layerIndex = 0,           // 底层
            animName = data.m_idleAnimName,
            looping = true,           // 循环播放
            blendTime = 0.5f
        );
    }

    // 2. 设置上层通道
    if (data.m_animationName)
    {
        SetLayerAnimation(
            layerIndex = 1,           // 上层
            animName = data.m_animationName,
            looping = false,          // 播放一次
            blendTime = data.m_blendTime
        );
    }

    // 3. 处理暂停
    if (data.m_pauseTime > 0)
    {
        // PauseClip：只保持底层idle，暂停时间
        SetPauseTimer(data.m_pauseTime);
    }
}
```

---

## 四、控制权转移的关键数据结构

### 4.1 WorkspotSetupContext（转移参数）

```cpp
// workspotSystem.h: 116-132
struct WorkspotSetupContext
{
    work::WorkspotParams m_workspot;     // Workspot参数
    // ├─ WorkspotTree资源
    // ├─ AnimGraph资源
    // └─ OriginId（点位ID）

    const world::GlobalNodeID& m_nodeId; // 世界节点ID

    red::SharedPtr<IWorkspotInstanceCommFunc> m_commFun;  // ⚠️ 回调接口
    // 用于Workspot通知Character的关键接口：
    // - OnCompleted(): Workspot结束
    // - OnMovementStateChanged(): 移动状态变化
    // - TeleportRequest(): 请求传送
    // - MovementRequest(): 请求移动

    CName m_entryAnim;                   // 指定的进入动画
    CName m_exitAnim;                    // 指定的退出动画
    Bool m_isWorkspotStatic = false;     // 是否是静态Workspot

    // 可选参数
    red::SharedPtr<work::WorkspotGlobalItemManager> m_globalItemManager;
    work::WorkspotItemOverride m_itemOverride;
    WorkspotClippingSpace m_clippingSpace;
    Bool m_disableInertiaBlend = false;
    Bool m_snapToGround = false;
};
```

### 4.2 IWorkspotInstanceCommFunc（通信接口）

```cpp
// workspotSystem.h: 77-99
class IWorkspotInstanceCommFunc
{
public:
    // ⚠️ Workspot完成时调用
    virtual void OnCompleted() = 0;
    // → 通知Character AI行为树：任务完成，恢复控制权

    // ⚠️ 当前Entry变化时调用
    virtual void OnActiveRecordChanged(work::WorkEntryId& id) = 0;
    // → 通知Character：当前播放的Entry ID

    // ⚠️ 移动状态变化时调用
    virtual void OnMovementStateChanged(move::MovementType newMovementType) = 0;
    // → 通知Character：移动类型变化（Walk/Run/Stand等）

    // ⚠️ 请求传送
    virtual void TeleportRequest(const Transform& posLS, Bool snapToZ) = 0;
    // → Character直接传送到目标位置

    // ⚠️ 请求移动（带动画）
    virtual void MovementRequest(
        Transform transform,        // 目标位置
        Transform* startLocation,   // 起始位置
        Float logicDuration,        // 逻辑时长
        CName animName,             // 动画名
        Float forceTime,            // 强制播放时间
        const Uint32 recordFlags,   // 标记
        Bool forceSnapToTerrain     // 是否贴地
    ) = 0;
    // → Character播放移动动画，移动到目标位置

    // ⚠️ 退出动画播放前调用
    virtual void RequestEnd() = 0;
    // → 通知Character：即将退出Workspot

    // ⚠️ 是否自动播放退出动画
    virtual Bool AutoPlayExit() const = 0;
    // → 返回false时，Workspot进入IdleOnlyMode，等待外部命令
};
```

### 4.3 WorkspotInstance状态机

```cpp
// gameWorkspotsInstance.h: 46-58
enum class State
{
    Invalid,                   // 无效状态
    Initialized,              // 已初始化，等待Play命令
    PlaybackRequest,          // 请求播放（进入动画阶段）
    EnterSlide,               // 滑动进入（平滑移动到点位）
    Playback,                 // 正在播放Workspot序列
    ExitRequest,              // 请求退出
    Exit,                     // 正在播放退出动画
    ExitSlide,                // 滑动退出
    PreExitPlaybackFinish,    // 退出前播放完成
    Frozen,                   // 冻结状态（调试用）
};

// 状态转换流程
Initialized
  ↓ CMD_Play
PlaybackRequest
  ↓ EntryAnim开始
EnterSlide
  ↓ EntryAnim完成
Playback ← ⚠️ 核心状态：Workspot序列播放
  ↓ CMD_SlowExit / Sequence结束
ExitRequest
  ↓ ExitAnim开始
Exit
  ↓ ExitAnim完成
ExitSlide
  ↓ 离开Workspot
OnCompleted() → Character恢复控制权
```

---

## 五、实际运行示例

### 示例：NPC在餐厅使用椅子

```
【第1帧】Character AI决策
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NPC的行为树执行：
└─ BTTask_UseWorkspot::ExecuteTask()
   ├─ 查询附近Workspot
   ├─ 评估：restaurant_chair_01 得分最高
   └─ 调用：WorkspotSystem::SetupWorkspot(
        entity = NPC_Civilian_001,
        workspotTree = "restaurant_chair_01.workspot",
        entryAnim = CName::NONE()  // 自动选择最近的
      )

【第2帧】控制权移交
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WorkspotSystem::SetupWorkspot() 执行：
├─ 创建 WorkspotInstance
├─ 加载 "restaurant_chair_01.workspot"
│  └─ WorkspotTree结构：
│     ├─ EntryAnim: walk_to_sit
│     ├─ Selector: 坐下后行为
│     │  ├─ Sequence: 用餐
│     │  └─ Sequence: 等待
│     └─ ExitAnim: sit_to_walk
├─ 创建根迭代器
└─ 状态：Initialized

Character AI状态：
└─ BTTask_UseWorkspot 返回 InProgress
   → 行为树暂停，等待Workspot完成

【第3帧】发送Play命令
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WorkspotSystem::SendCommand(CMD_Play)
└─ WorkspotInstance::Play()
   ├─ 迭代器跳转到EntryAnim
   ├─ 状态：PlaybackRequest → EnterSlide
   └─ 通知Character：MovementRequest(
        transform = 椅子位置,
        animName = "walk_to_sit",
        duration = 2.0f
      )

NPC表现：
└─ 开始播放walk_to_sit动画，走向椅子

【第10帧】进入动画完成
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WorkspotInstance::UpdateRecord()
├─ EntryAnim播放完毕
├─ m_iterator->Next() 推进到Selector
├─ Selector随机选择：用餐Sequence
└─ m_iterator->GetData(entryData)
   → entryData.m_idleAnimName = "sit"         (底层)
   → entryData.m_animationName = "sit_look_menu" (上层)

动画系统：
├─ 底层：播放sit idle（循环）
└─ 上层：播放sit_look_menu（一次）

【第20帧】序列播放中
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
sit_look_menu播放完毕
└─ m_iterator->Next()
   → entryData.m_idleAnimName = "sit"
   → entryData.m_animationName = "sit_eat"

动画系统：
├─ 底层：sit idle 持续播放
└─ 上层：切换到sit_eat

【第30帧】用户触发事件
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
玩家撞到NPC
└─ BumpSystem::TriggerNPCBump(NPC_Civilian_001, Bump)
   └─ WorkspotSystem::SendCommand(
        ownerId = NPC_Civilian_001,
        cmd = CMD_SendReaction,
        data = {reactionName = "Bump"}
      )

WorkspotInstance处理：
├─ 查找ReactionSequence
├─ m_iterator->GoTo(reactionEntryId)
└─ 播放reaction_bump_left动画

动画系统：
├─ 中断当前动画
├─ 0.2秒强制混合到反应动画
└─ 播放完后恢复到中断前的位置

【第50帧】AI决定离开
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Character AI（外部系统）决定：该离开了
└─ WorkspotSystem::SendCommand(CMD_SlowExit)

WorkspotInstance::ReceiveCommands()
├─ 状态：Playback → ExitRequest
├─ 查找ExitAnim（idle="sit"）
├─ m_iterator->GoTo(exitAnimId)
└─ 播放sit_to_walk动画

【第60帧】退出完成，控制权返还
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ExitAnim播放完毕
└─ WorkspotInstance::OnCompleted()
   ├─ 调用：m_commFunctions->OnCompleted()
   │  └─ 通知Character AI：Workspot结束
   ├─ 清理WorkspotInstance
   └─ 从WorkspotSystem移除

Character AI状态：
└─ BTTask_UseWorkspot::OnTaskFinished(Succeeded)
   ├─ 行为树恢复
   └─ 继续执行下一个节点（巡逻/战斗/其他）

NPC恢复自主控制，继续正常AI行为
```

---

## 六、关键时刻总结

### 转变发生的关键时刻

| 时刻 | 发生的事 | 控制权状态 |
|------|----------|----------|
| **T0** | Character AI决策"使用Workspot" | Character完全控制 |
| **T1** | 调用`WorkspotSystem::SetupWorkspot()` | 开始移交 |
| **T2** | 创建`WorkspotInstance`，加载`WorkspotTree` | 50%移交 |
| **T3** | 创建根迭代器`m_iterator` | 70%移交 |
| **T4** | 发送`CMD_Play`命令 | 80%移交 |
| **T5** | `WorkspotInstance::Play()`开始执行 | **100%移交** ⚠️ |
| **T6~T100** | Workspot序列播放中 | Workspot完全控制 |
| **T101** | `OnCompleted()`通知Character | 开始返还 |
| **T102** | Character AI恢复 | Character完全控制 |

### 判断标准

**Character是否还有控制权？**

```cpp
// 方式1：检查行为树状态
if (BTTask->IsInProgress())
{
    // 行为树暂停 → 控制权已移交给Workspot
}

// 方式2：查询WorkspotSystem
if (WorkspotSystem::IsActorInWorkspot(entityID))
{
    // 该NPC正在使用Workspot → Workspot控制中
}

// 方式3：检查WorkspotInstance状态
WorkspotInstance* inst = WorkspotSystem::FindInstance(entityID);
if (inst && inst->IsActive())
{
    // WorkspotInstance活跃 → Workspot控制中
}
```

---

## 七、为什么需要这样的转移机制？

### 7.1 分离关注点

```
Character AI关注：
- 战略决策："我要不要坐？"
- 目标选择："坐哪个椅子？"
- 高层行为："战斗？休息？巡逻？"

WorkspotSystem关注：
- 战术执行："如何坐椅子？"
- 动画播放："播放哪些动画？"
- 状态管理："当前在Entry的哪个位置？"
```

### 7.2 解耦依赖

```
传统方式：
Character AI必须知道：
- 如何播放坐下动画
- 坐下后做什么
- 如何站起来
→ 每种交互都要写代码

WorkspotTree方式：
Character AI只需知道：
- 有一个叫"restaurant_chair_01"的Workspot
- 调用SetupWorkspot(entity, workspot)
→ 其他的WorkspotTree负责
```

### 7.3 支持并行工作

```
程序员：
- 写WorkspotSystem（一次性工作）
- 写AI决策逻辑（通用逻辑）

关卡设计师：
- 配置Workspot资源（数据驱动）
- 不需要等程序实现

动画师：
- 按命名规范提交动画
- 自动被WorkspotTree引用
```

---

## 八、常见问题

### Q1：转移后Character还能做什么？

**A：几乎不能做任何决策**
- ✗ 不能战斗
- ✗ 不能巡逻
- ✗ 不能自主移动
- ✓ 可以接收外部命令（如SendReaction）
- ✓ 可以被强制退出（CMD_FastExit）

### Q2：如何中断Workspot？

**方式1：正常退出**
```cpp
WorkspotSystem::SendCommand(entityID, CMD_SlowExit);
// → 播放ExitAnim后退出
```

**方式2：快速退出**
```cpp
WorkspotSystem::SendCommand(entityID, CMD_FastExit);
// → 强制混合到FastExit动画，立即离开
```

**方式3：强制停止**
```cpp
WorkspotSystem::SendCommand(entityID, CMD_Stop);
// → 立即停止，不播放退出动画
```

### Q3：多个NPC能同时使用一个Workspot吗？

**A：取决于Workspot配置**
```cpp
WorkspotComponent::m_maxSimultaneousUsers = 2;
// → 允许2个NPC同时使用
// 例如：双人沙发、多人餐桌
```

### Q4：Workspot执行中AI能响应战斗吗？

**A：由外部系统决定**
```cpp
// 外部战斗系统检测
if (IsInCombat(npc))
{
    // 强制退出Workspot
    WorkspotSystem::SendCommand(npc, CMD_FastExit);
    // → NPC离开椅子，进入战斗
}
```

---

## 九、总结

### 控制权转移的本质

**不是"替代"，而是"委托"**

```
Character AI说：
"我决定使用这个椅子，但我不知道具体怎么坐。
WorkspotSystem，麻烦你替我执行，完成后通知我。"

WorkspotSystem回应：
"好的，我会按照WorkspotTree的脚本执行。
执行期间你暂停决策，完成后我调用OnCompleted()还给你。"
```

### 这是一种智能的分工

- **Character AI**：战略层（"做什么"）
- **WorkspotSystem**：战术层（"怎么做"）

通过`WorkspotSetupContext`和`IWorkspotInstanceCommFunc`实现双向通信，完成了优雅的控制权交接。

这正是开放世界游戏中，高效管理数万个NPC与数万个交互点位的关键设计！
