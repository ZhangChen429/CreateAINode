# InteractiveScene技术参考手册
> 开发者实战指南 - 代码示例与API详解

---

## 📖 目录

1. [快速开始](#快速开始)
2. [核心API参考](#核心api参考)
3. [常见模式](#常见模式)
4. [调试技巧](#调试技巧)
5. [性能优化](#性能优化)
6. [常见陷阱](#常见陷阱)

---

## 快速开始

### 创建基础场景实例

```cpp
// 1. 准备场景ID
scn::SceneId sceneId = scn::SceneId::FromPath("base\\quest\\main_quests\\q110\\scenes\\q110_02_placide.scene");

// 2. 设置所有者（通常是Quest Phase）
scn::SceneInstanceOwnerId ownerId(questPhaseId);

// 3. 配置实例参数
scn::SceneInstanceParams params;
params.initialState = scn::SceneInstanceInitialState::Playing;

// 4. 配置外围系统
scn::SceneInstancePeripherals peripherals;
peripherals.tierSystem = &game->GetTierSystem();
peripherals.audioSystem = &game->GetAudioDirectorSystem();

// 5. 创建实例
scn::SceneOperationStatus status;
scn::SceneInstanceId instanceId = sceneSystem->CreateSceneInstance(
    sceneId,
    ownerId,
    params,
    peripherals,
    scn::SceneInstanceDebug::None,
    status
);

if (status == scn::SceneOperationStatus::Success) {
    RED_LOG(SceneSystem, "Scene instance created: %llu", instanceId.GetValue());
}
```

---

## 核心API参考

### A. SceneSystem接口

#### 实例管理

```cpp
// 创建新实例
SceneInstanceId CreateSceneInstance(
    SceneId sceneId,
    SceneInstanceOwnerId ownerId,
    const SceneInstanceParams& params,
    const SceneInstancePeripherals& peripherals,
    SceneInstanceDebug debugMode,
    SceneOperationStatus& outStatus
);

// 查找活跃实例
SceneInstanceId FindActiveSceneInstanceId(
    SceneId sceneId,
    SceneInstanceOwnerId ownerId
) const;

// 分叉实例（创建副本）
SceneInstanceId ForkSceneInstance(
    SceneInstanceOwnerId ownerId,
    const SceneForkedInstanceParams& params,
    const SceneInstancePeripherals& peripherals,
    SceneInstanceDebug debugMode,
    SceneOperationStatus& outStatus
);
```

#### 控制请求

```cpp
// 队列控制请求
ControlRequestId QueueRequest(
    SceneInstanceId sciId,
    const Control::Request& request,
    ControlRequestTarget target = ControlRequestTarget()
);

// 控制请求示例
namespace scn::Control {
    struct Request {
        enum class Type {
            Play,
            Pause,
            Stop,
            JumpToSection,
            SetPlaybackSpeed
        };

        Type type;
        CName sectionName;  // 用于JumpToSection
        Float playbackSpeed; // 用于SetPlaybackSpeed
    };
}
```

---

### B. SceneListener接口

#### 实现监听器

```cpp
class MySceneListener : public scn::SceneListener {
public:
    void OnNotificationsPublished(
        const scn::SceneNotificationQueue& notifications
    ) override {
        for (const auto& notif : notifications) {
            switch (notif.m_notificationType) {
                case scn::SceneNotificationType::instanceStarted:
                    OnSceneStarted(notif.m_sceneInstanceId);
                    break;

                case scn::SceneNotificationType::instanceFinished:
                    OnSceneFinished(notif.m_sceneInstanceId);
                    break;

                case scn::SceneNotificationType::instanceEntryPoint: {
                    auto data = scn::SceneNotificationData<
                        scn::SceneNotificationType::instanceEntryPoint
                    >(notif);
                    OnEntryPoint(notif.m_sceneInstanceId, data.m_entryPoint);
                    break;
                }

                case scn::SceneNotificationType::instanceInterrupted:
                    OnSceneInterrupted(notif.m_sceneInstanceId);
                    break;
            }
        }
    }

private:
    void OnSceneStarted(scn::SceneInstanceId sciId) {
        RED_LOG(Quest, "Scene started: %llu", sciId.GetValue());
    }

    void OnEntryPoint(scn::SceneInstanceId sciId, CName entryPoint) {
        RED_LOG(Quest, "Entry point reached: %s", entryPoint.AsChar());
        // 触发Quest目标更新等
    }
};

// 注册监听器
MySceneListener listener;
sceneSystem->RegisterListener(instanceId, listener);

// 使用完毕后注销
sceneSystem->DeregisterListener(instanceId, listener);
```

---

### C. Tier系统API

#### 请求Tier切换

```cpp
// 准备Tier数据
THandle<scn::SceneTier3Data> tier3Data = THandle<scn::SceneTier3Data>::Create();
tier3Data->tier = GameplayTier::Tier3_LimitedGameplay;
tier3Data->emptyHands = true;

// 设置摄像头限制
tier3Data->cameraSettings.yawLeftLimit = -60.0f;    // 左转60度
tier3Data->cameraSettings.yawRightLimit = 60.0f;    // 右转60度
tier3Data->cameraSettings.pitchTopLimit = 30.0f;    // 抬头30度
tier3Data->cameraSettings.pitchBottomLimit = -30.0f; // 低头30度

// 设置速度衰减（接近边界时减速）
tier3Data->cameraSettings.speedMultiplier = 0.3f;

// 请求Tier
scn::TierQuery query;
query.entityId = playerEntityId;

tierSystem->RequestTier(query, tier3Data);

// 清除Tier（恢复到更低层级）
tierSystem->ClearTier(query, GameplayTier::Tier3_LimitedGameplay);
```

#### 查询当前Tier

```cpp
GameplayTier currentTier = tierSystem->GetTier(playerEntityId);

switch (currentTier) {
    case GameplayTier::Tier1_FullGameplay:
        // 玩家完全自由
        break;
    case GameplayTier::Tier3_LimitedGameplay:
        // 玩家在Tier3限制中
        break;
}
```

---

### D. 中断系统使用

#### 定义中断条件

```cpp
// 距离中断条件
class MyDistanceInterrupt : public scn::IInterruptCondition {
    RTTI_DECLARE_TYPE(MyDistanceInterrupt);

private:
    ent::EntityID m_targetEntity;
    Float m_maxDistance;

public:
    scn::InterruptConditionType GetType() const override {
        return scn::InterruptConditionType::DistancePlayerEntity;
    }

    Bool CheckCondition(const scn::ISceneConditionContext& context) const override {
        // 获取玩家位置
        WorldPosition playerPos = context.GetPlayerPosition();

        // 获取目标实体位置
        ent::Entity* target = context.GetEntity(m_targetEntity);
        if (!target) return false;

        WorldPosition targetPos = target->GetWorldPosition();

        // 计算距离
        Float distance = WorldPosition::Distance(playerPos, targetPos);

        return distance > m_maxDistance;
    }
};
```

#### 在场景图中配置中断

```json
{
  "nodes": [
    {
      "id": 42,
      "type": "scnDialogLineNode",
      "interruptSettings": {
        "conditions": [
          {
            "$type": "scnsInterruptConditionDistancePlayerEntity",
            "distance": 10.0,
            "entityReference": "npc_placide"
          },
          {
            "$type": "scnsInterruptConditionPlayerCombat"
          }
        ],
        "interruptionBranch": {
          "targetNodeId": 100,
          "transitionType": "immediate"
        }
      }
    }
  ]
}
```

---

## 常见模式

### 模式1：等待场景完成

```cpp
class SceneWaiter : public scn::SceneListener {
    std::atomic<bool> m_finished{false};
    scn::SceneInstanceId m_watchedInstance;

public:
    void WatchScene(scn::SceneInstanceId sciId) {
        m_watchedInstance = sciId;
        m_finished = false;
    }

    void OnNotificationsPublished(
        const scn::SceneNotificationQueue& notifications
    ) override {
        for (const auto& notif : notifications) {
            if (notif.m_sceneInstanceId == m_watchedInstance) {
                if (notif.m_notificationType ==
                    scn::SceneNotificationType::instanceFinished) {
                    m_finished = true;
                }
            }
        }
    }

    bool IsFinished() const { return m_finished; }
};

// 使用
SceneWaiter waiter;
sceneSystem->RegisterListener(instanceId, waiter);
waiter.WatchScene(instanceId);

// 在游戏循环中
while (!waiter.IsFinished()) {
    // 等待场景完成
    Red::Sleep(16); // 16ms
}
```

### 模式2：条件分支场景

```cpp
// 场景图结构
/*
[StartNode]
    ↓
[CheckFactNode]
    ├─ FactA=true → [PathA_Node]
    └─ FactA=false → [PathB_Node]
*/

// 在代码中设置Fact
game::FactsDB* factsDB = game->GetFactsDB();
factsDB->SetFact("q110_path_choice", 1); // 1=PathA, 2=PathB

// 场景图会自动根据Fact值选择分支
```

### 模式3：动态演员绑定

```cpp
// 场景定义时使用抽象ActorId
// 运行时绑定具体实体

scn::SceneInstanceParams params;

// 绑定演员
params.actorBindings.Insert(
    scn::ActorId("npc_companion"),
    ent::EntityID(companionEntity->GetId())
);

params.actorBindings.Insert(
    scn::ActorId("npc_enemy"),
    ent::EntityID(enemyEntity->GetId())
);

// 创建场景实例时这些绑定会生效
scn::SceneInstanceId instanceId = sceneSystem->CreateSceneInstance(
    sceneId, ownerId, params, peripherals, debug, status
);
```

### 模式4：Braindance控制

```cpp
// 检查是否在可回放区域
if (sceneSystem->IsRewindableSectionActive()) {
    // 获取当前进度（0.0 - 1.0）
    Float progress = sceneSystem->GetRewindableSectionProgress();

    // 暂停
    sceneSystem->SetRewindableSectionPlaySpeed(scn::PlaySpeed::Pause);

    // 快进2倍速
    sceneSystem->SetRewindableSectionPlaySpeed(scn::PlaySpeed::Fast);

    // 跳转到50%位置
    Float totalDuration = sceneSystem->GetRewindableSectionDurationInSec();
    scn::SceneTime targetTime = static_cast<scn::SceneTime>(totalDuration * 0.5f * 1000);

    sceneSystem->RewindableSectionJumpToTime(
        2.0f,  // 跳转速度
        targetTime,
        scn::PlayDirection::Forward,
        scn::PlaySpeed::Normal
    );
}
```

---

## 调试技巧

### 1. 启用场景调试器

```cpp
#ifndef RED_CONFIGURATION_FINAL
// 创建场景实例时启用调试
scn::SceneInstanceId instanceId = sceneSystem->CreateSceneInstance(
    sceneId,
    ownerId,
    params,
    peripherals,
    scn::SceneInstanceDebug::Enabled,  // 启用调试
    status
);

// 获取调试器接口
::interop::ObjectPtr debugger = sceneSystem->CreateDebuggerInterop();

// 跳转到特定时间点（编辑器功能）
scn::JumpContext jumpCtx(
    instanceId,
    scn::NodeId(42),  // 目标节点
    scn::SceneTime(5000)  // 5秒位置
);
sceneSystem->Editor_JumpToTimepos(jumpCtx);
#endif
```

### 2. 日志记录

```cpp
// 监听所有场景事件
class DebugSceneListener : public scn::SceneListener {
public:
    void OnNotificationsPublished(
        const scn::SceneNotificationQueue& notifications
    ) override {
        for (const auto& notif : notifications) {
            RED_LOG(SceneDebug, "Scene %llu: Event %d",
                notif.m_sceneInstanceId.GetValue(),
                static_cast<int>(notif.m_notificationType));

            // 记录Entry/Exit点
            if (notif.m_notificationType ==
                scn::SceneNotificationType::instanceEntryPoint) {
                auto data = scn::SceneNotificationData<
                    scn::SceneNotificationType::instanceEntryPoint
                >(notif);
                RED_LOG(SceneDebug, "  EntryPoint: %s",
                    data.m_entryPoint.AsChar());
            }
        }
    }
};
```

### 3. 检查ExecutionStream

```cpp
#ifndef RED_CONFIGURATION_FINAL
// 获取场景的ExecutionStream（内部API）
const scn::ExecutionStream& exestream =
    sceneInstance->GetExecutionStream();

// 检查通道内容
const scn::ActionChannel& actionChannel = exestream.GetActionChannel();
RED_LOG(SceneDebug, "Action count: %u", actionChannel.GetRecordCount());

const scn::ControlChannel& controlChannel = exestream.GetControlChannel();
RED_LOG(SceneDebug, "Control count: %u", controlChannel.GetRecordCount());

const scn::StimulationChannel& stimChannel = exestream.GetStimulationChannel();
RED_LOG(SceneDebug, "Stimulation count: %u", stimChannel.GetStimulationCount());

// 检查索引状态
if (!exestream.IsIndexed()) {
    RED_LOG_WARNING(SceneDebug, "ExecutionStream not indexed - performance may suffer");
}
#endif
```

---

## 性能优化

### 1. 预分配ExecutionStream容量

```cpp
// 对于已知复杂度的场景，预分配容量避免动态扩容
scn::ExecutionStream exestream(scn::PoolSceneSystem{});

// 预估动作数量并预分配
exestream.Reserve(
    1000,  // 动作通道容量
    100,   // 控制通道容量
    200    // 刺激通道容量
);
```

### 2. 及时清理完成的场景

```cpp
class SceneCleanupListener : public scn::SceneListener {
    ISceneSystem* m_sceneSystem;

public:
    void OnNotificationsPublished(
        const scn::SceneNotificationQueue& notifications
    ) override {
        for (const auto& notif : notifications) {
            if (notif.m_notificationType ==
                scn::SceneNotificationType::instanceFinished) {
                // 场景完成后自动注销监听器
                m_sceneSystem->DeregisterListener(
                    notif.m_sceneInstanceId, *this);
            }
        }
    }
};
```

### 3. 优化中断条件检查频率

```cpp
// 不要每帧都检查所有中断条件
class OptimizedInterruptCondition : public scn::IInterruptCondition {
    mutable Uint32 m_lastCheckFrame;
    mutable Bool m_cachedResult;

public:
    Bool CheckCondition(const scn::ISceneConditionContext& context) const override {
        Uint32 currentFrame = context.GetCurrentFrame();

        // 只每5帧检查一次
        if (currentFrame - m_lastCheckFrame < 5) {
            return m_cachedResult;
        }

        m_lastCheckFrame = currentFrame;
        m_cachedResult = DoActualCheck(context);
        return m_cachedResult;
    }

private:
    Bool DoActualCheck(const scn::ISceneConditionContext& context) const;
};
```

### 4. 使用Workspot池

```cpp
// 复用Workspot实例而不是每次创建
class WorkspotPool {
    red::DynArray<scn::SceneWorkspotInstanceId> m_availableWorkspots;

public:
    scn::SceneWorkspotInstanceId AcquireWorkspot(
        scn::ISceneSystem* sceneSystem,
        const scn::WorkspotDataRequest& request
    ) {
        if (m_availableWorkspots.Empty()) {
            // 创建新Workspot
            scn::WorkspotDataRequestOutput output;
            sceneSystem->GetWorkspotData(request, output);
            return output.instanceId;
        }

        // 复用现有Workspot
        auto id = m_availableWorkspots.Back();
        m_availableWorkspots.PopBack();
        return id;
    }

    void ReleaseWorkspot(scn::SceneWorkspotInstanceId id) {
        m_availableWorkspots.PushBack(id);
    }
};
```

---

## 常见陷阱

### 陷阱1：忘记注销监听器

```cpp
// ❌ 错误：监听器离开作用域但未注销
void PlaySceneBad(scn::ISceneSystem* sceneSystem) {
    MySceneListener listener;
    sceneSystem->RegisterListener(instanceId, listener);
    // listener析构，但sceneSystem仍持有悬空指针！
}

// ✅ 正确：使用RAII
class ScopedSceneListener {
    scn::ISceneSystem* m_system;
    scn::SceneInstanceId m_instanceId;
    scn::SceneListener* m_listener;

public:
    ScopedSceneListener(
        scn::ISceneSystem* system,
        scn::SceneInstanceId instanceId,
        scn::SceneListener& listener
    ) : m_system(system), m_instanceId(instanceId), m_listener(&listener) {
        m_system->RegisterListener(m_instanceId, *m_listener);
    }

    ~ScopedSceneListener() {
        m_system->DeregisterListener(m_instanceId, *m_listener);
    }
};
```

### 陷阱2：在通知回调中修改场景

```cpp
// ❌ 危险：在通知处理中停止场景可能导致迭代器失效
class DangerousListener : public scn::SceneListener {
    ISceneSystem* m_sceneSystem;

    void OnNotificationsPublished(
        const scn::SceneNotificationQueue& notifications
    ) override {
        for (const auto& notif : notifications) {
            if (ShouldStop(notif)) {
                // 可能在遍历通知队列时修改场景状态！
                scn::Control::Request req;
                req.type = scn::Control::Request::Type::Stop;
                m_sceneSystem->QueueRequest(notif.m_sceneInstanceId, req);
            }
        }
    }
};

// ✅ 安全：延迟操作
class SafeListener : public scn::SceneListener {
    red::DynArray<scn::SceneInstanceId> m_pendingStops;

    void OnNotificationsPublished(
        const scn::SceneNotificationQueue& notifications
    ) override {
        for (const auto& notif : notifications) {
            if (ShouldStop(notif)) {
                m_pendingStops.PushBack(notif.m_sceneInstanceId);
            }
        }
    }

    void ProcessPendingStops(ISceneSystem* sceneSystem) {
        for (auto sciId : m_pendingStops) {
            scn::Control::Request req;
            req.type = scn::Control::Request::Type::Stop;
            sceneSystem->QueueRequest(sciId, req);
        }
        m_pendingStops.Clear();
    }
};
```

### 陷阱3：Tier冲突

```cpp
// ❌ 多个系统同时请求不同Tier
void ConflictingTiers(scn::TierSystem* tierSystem, ent::EntityID playerId) {
    // 系统A请求Tier3
    THandle<scn::SceneTier3Data> tier3 = CreateTier3Data();
    scn::TierQuery query{playerId};
    tierSystem->RequestTier(query, tier3);

    // 系统B立即请求Tier1（覆盖Tier3）
    THandle<scn::SceneTier1Data> tier1 = CreateTier1Data();
    tierSystem->RequestTier(query, tier1);  // Tier3被覆盖！
}

// ✅ 使用Tier堆栈管理
class TierManager {
    struct TierRequest {
        scn::TierQuery query;
        THandle<scn::SceneTierData> data;
        Uint32 priority;
    };

    red::DynArray<TierRequest> m_activeRequests;

    void RequestTier(
        scn::TierSystem* tierSystem,
        const TierRequest& request
    ) {
        m_activeRequests.PushBack(request);

        // 按优先级排序
        std::sort(m_activeRequests.begin(), m_activeRequests.end(),
            [](const TierRequest& a, const TierRequest& b) {
                return a.priority > b.priority;
            });

        // 应用最高优先级Tier
        if (!m_activeRequests.Empty()) {
            const auto& topRequest = m_activeRequests.Front();
            tierSystem->RequestTier(topRequest.query, topRequest.data);
        }
    }
};
```

### 陷阱4：ExecutionStream未索引

```cpp
// ❌ 大量时间查询未索引的Stream
void SlowTimeQuery(const scn::ExecutionStream& stream) {
    for (Uint32 i = 0; i < 1000; ++i) {
        scn::SceneTime time = static_cast<scn::SceneTime>(i * 100);
        // 未索引时这是O(n)操作，重复1000次非常慢！
        auto actions = stream.GetActionChannel().GetActionsAtTime(time);
    }
}

// ✅ 使用索引
void FastTimeQuery(scn::ExecutionStream& stream) {
    // 一次性索引
    if (!stream.IsIndexed()) {
        stream.Reindex();
    }

    // 现在查询是O(log n)
    for (Uint32 i = 0; i < 1000; ++i) {
        scn::SceneTime time = static_cast<scn::SceneTime>(i * 100);
        auto actions = stream.GetActionChannel().GetActionsAtTime(time);
    }
}
```

---

## RedScript集成

### 从脚本控制场景

```swift
// RedScript示例
class MyQuestPhase extends ScriptedQuestPhase {
    private let sceneSystem: ref<SceneSystem>;

    protected cb func OnActivate(questPhase: CName) -> Bool {
        this.sceneSystem = GameInstance.GetSceneSystem();

        // 检查实体是否在场景中
        let npcEntity = this.GetNPCEntity();
        if this.sceneSystem.IsEntityInScene(npcEntity.GetEntityID()) {
            LogChannel(n"Quest", "NPC is in scene");
        }

        // 检查是否在对话中
        if this.sceneSystem.IsEntityInDialogue(npcEntity.GetEntityID()) {
            LogChannel(n"Quest", "NPC is in dialogue");
        }
    }

    // 等待场景完成
    protected cb func WaitForSceneEnd() -> Bool {
        let player = GameInstance.GetPlayerSystem().GetLocalPlayerMainGameObject();

        while this.sceneSystem.IsEntityInScene(player.GetEntityID()) {
            // 等待
            GameInstance.GetDelaySystem().DelayCallback(
                this.OnCheckScene, 0.5
            );
        }

        this.OnSceneFinished();
    }
}
```

### 访问Braindance功能

```swift
// Braindance控制脚本
class BraindanceController extends IScriptable {
    private let sceneSystem: ref<SceneSystem>;

    public func Initialize() {
        this.sceneSystem = GameInstance.GetSceneSystem();
    }

    public func IsActive() -> Bool {
        return this.sceneSystem.IsRewindableSectionActive();
    }

    public func GetProgress() -> Float {
        return this.sceneSystem.GetRewindableSectionProgress();
    }

    public func Pause() {
        this.sceneSystem.SetRewindableSectionPlaySpeed(
            scnPlaySpeed.Pause
        );
    }

    public func FastForward() {
        this.sceneSystem.SetRewindableSectionPlaySpeed(
            scnPlaySpeed.Fast
        );
    }

    public func GetTimeInfo() -> String {
        let current = this.sceneSystem.GetRewindableSectionTimeInSec();
        let total = this.sceneSystem.GetRewindableSectionDurationInSec();
        return s"\\(current) / \\(total)";
    }
}
```

---

## 高级技巧

### 自定义Action Definition

```cpp
// 创建自定义场景动作
class ActionDefinitionCustomTeleport : public scn::ActionDefinition {
    RTTI_DECLARE_TYPE(ActionDefinitionCustomTeleport);

public:
    struct Params {
        WorldPosition targetPosition;
        Bool useFadeEffect;
        Float fadeDuration;
    };

    Params m_params;
    scn::Msec m_duration;

    // 创建模拟动作实例
    virtual scn::ActionInstancePtr<scn::sim::SimActionInstance>
    DoCreateSimActionInstance(
        scn::ActionId id,
        const red::memory::Pool& pool
    ) const override {
        return scn::MakeActionInstance<SimActionInstanceCustomTeleport>(
            id, pool, m_params
        );
    }

    virtual scn::Msec DoGetDuration() const override {
        return m_duration;
    }
};

// 注册到系统
RTTI_IMPL_TYPE_CLASS(ActionDefinitionCustomTeleport);
```

### 动态场景合成

```cpp
// 运行时组合多个场景片段
class SceneComposer {
public:
    scn::ExecutionStream ComposeScenes(
        const red::DynArray<scn::ExecutionStream>& scenes,
        const red::DynArray<scn::Msec>& delays
    ) {
        scn::ExecutionStream result(scn::PoolSceneSystem{});

        scn::Msec currentOffset = 0;
        for (Uint32 i = 0; i < scenes.Size(); ++i) {
            if (i < delays.Size()) {
                currentOffset += delays[i];
            }

            // 合并子场景并偏移时间
            result.CombineSubstream(scenes[i], currentOffset);

            // 计算子场景时长
            currentOffset += GetStreamDuration(scenes[i]);
        }

        // 重新索引优化查询
        result.Reindex(true);

        return result;
    }

private:
    scn::Msec GetStreamDuration(const scn::ExecutionStream& stream) {
        // 查找最后一个动作的结束时间
        const auto& actionChannel = stream.GetActionChannel();
        // 实现细节...
        return 0;
    }
};
```

---

## 总结

InteractiveScene系统的使用要点：

1. **正确管理生命周期** - 始终注销监听器，避免内存泄漏
2. **理解异步特性** - 通知是批量发布的，不是即时的
3. **善用Tier系统** - 在叙事需求与玩家自由间找平衡
4. **优化ExecutionStream** - 预分配容量，及时索引
5. **小心中断条件** - 避免过于频繁的检查
6. **利用调试工具** - 在开发环境充分使用调试功能

---

*本手册基于Cyberpunk 2077源代码编写*
*适用版本：REDengine 4*
*最后更新：2026-02-09*
