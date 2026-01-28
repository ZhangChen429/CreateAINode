# Cyberpunk 2077 Workspot系统调试完全指南

## 核心问题：如何追踪NPC什么时候开始执行哪个Workspot？

---

## 一、调试系统架构概览

### 1.1 调试系统组件

```cpp
// 调试系统核心组件
work::WorkspotSystem
├─ work::Debugger (主调试器)
│  ├─ m_activeTools (活跃的调试工具列表)
│  ├─ m_drawer (可视化绘制器)
│  └─ m_tickCounter (帧计数器)
├─ work::CallbackManager (回调管理器)
└─ work::Synchronizer (同步管理器)

// 调试工具基类
work::DebuggingTool
├─ WorkspotFunctionalTestsDebuggingTool (功能测试)
├─ workspotAnimObjectDebugger (动画对象调试)
└─ 其他自定义调试工具
```

### 1.2 调试开关

```cpp
// workspotDebugger.h: 10-22
#if !defined( RED_CONFIGURATION_FINAL ) || defined(USE_PROFILER)
#define WORKSPOT_DEBUG_ENABLED
#endif

#ifdef WORKSPOT_DEBUG_ENABLED
#define WORKSPOT_DEBUG_MACRO( x ) x  // 非Final版本启用
#else
#define WORKSPOT_DEBUG_MACRO( x )    // Final版本禁用
#endif
```

**关键**：调试功能仅在 **非Final配置** 或 **USE_PROFILER** 定义时启用。

---

## 二、关键调试时机与回调

### 2.1 Workspot生命周期的调试回调

#### ① Instance创建时
```cpp
// workspotSystem.cpp: 185
void WorkspotSystem::CreateInstance_NoLock(...)
{
    auto instance = CreateHandle<WorkspotInstance>( ent, m_scene->GetGameInstance() );

    WORKSPOT_DEBUG_MACRO( m_debugger.OnInstanceCreated( ownerId ) );
    // ↑ 第一个调试点：WorkspotInstance创建时触发
}
```

**触发时机**：NPC首次调用 `WorkspotSystem::SetupWorkspot()` 时
**获取信息**：NPC的EntityID

---

#### ② Workspot设置时（Setup）
```cpp
// workspotSystem.cpp: 242-243
void WorkspotSystem::SetupWorkspot(...)
{
    ent::EntityID ownerId = ent->GetEntityID();

    WORKSPOT_DEBUG_MACRO(
        String path = setup.m_workspot.m_tree->GetPath().ToDebugString();
        m_debugger.OnWorkspotSetup( ownerId, setup.m_workspot.m_tree, path );
    );
    // ↑ 第二个调试点：Workspot资源加载时触发
}
```

**触发时机**：WorkspotTree资源被设置到Instance时
**获取信息**：
- `ownerId` - NPC的EntityID
- `resource` - WorkspotTree资源句柄
- `path` - Workspot资源路径（如 `"workspots/restaurant_chair_01.workspot"`）

**这是追踪"NPC使用了哪个Workspot"的关键点！**

---

#### ③ Workspot开始播放时（Started）
```cpp
// gameWorkspotsInstance.cpp: 322
void WorkspotInstance::Play(...)
{
    context.m_synchronizer.OnWorkspotPlay( GetOwnerId() );
    m_workspotStartTime = context.m_currTime;

    WORKSPOT_DEBUG_MACRO( context.m_debugger.OnWorkspotStarted( GetOwnerId() ) );
    // ↑ 第三个调试点：动画序列开始播放时触发

    context.m_callbackManager.OnWorkspotStarted( m_ownerID, m_locId );
}
```

**触发时机**：收到 `CMD_Play` 命令后，迭代器开始推进时
**获取信息**：
- `ownerId` - NPC的EntityID
- `m_workspotStartTime` - 开始播放的游戏时间

---

#### ④ 动画切换时（AnimationChanged）
```cpp
// gameWorkspotsInstance.cpp: 1243-1253
GlobalWorkspotTime WorkspotInstance::UpdateRecord(...)
{
    // 提取当前Entry的数据
    m_iterator->GetData( data );

    if( data.m_animationName != m_currentData.m_animationName )
    {
        WORKSPOT_DEBUG_MACRO(
            context.m_debugger.OnAnimationChanged(
                GetOwnerId(),
                data.m_animationName,
                data.m_entryId,
                data.m_entryFlags
            );
        );
    }
}
```

**触发时机**：每次动画变化时
**获取信息**：
- `animName` - 新动画的名称（如 `"sit_eat"`）
- `entryId` - Entry的ID
- `flags` - Entry标志（AnimClip/PauseClip等）

---

#### ⑤ Workspot结束时（Finished）
```cpp
// gameWorkspotsInstance.cpp: 1103
void WorkspotInstance::OnCompleted(...)
{
    WORKSPOT_DEBUG_MACRO( debugger.OnWorkspotFinished( GetOwnerId() ) );

    // 通知回调管理器
    m_callbackManager.OnWorkspotFinished( m_ownerID, m_locId );
}
```

**触发时机**：Workspot序列播放完毕或被强制退出时
**获取信息**：NPC的EntityID

---

#### ⑥ Instance销毁时
```cpp
// workspotSystem.cpp: 200
void WorkspotSystem::RemoveInstance_NoLock(...)
{
    ent::EntityID ownerId = instance.GetOwnerId();

    WORKSPOT_DEBUG_MACRO( m_debugger.OnInstanceRemoved( ownerId ) );
}
```

**触发时机**：WorkspotInstance被销毁时
**获取信息**：NPC的EntityID

---

### 2.2 命令调试回调

```cpp
// workspotSystem.cpp: 354
Bool WorkspotSystem::SendCommandInternal_NoLock(...)
{
    WORKSPOT_DEBUG_MACRO(
        m_debugger.OnCommandReceived( ownerId, oriId, false, cmd, data, instance );
    );

    RED_AI_LOGF( ownerId, "ws", "Workspot command sent: %hs",
                 GetWorkspotCommandFriendlyName( cmd ).AsChar() );
}
```

**触发时机**：每次发送Workspot命令时（Play、SlowExit、FastExit等）
**获取信息**：
- `ownerId` - NPC的EntityID
- `cmd` - 命令类型（CMD_Play、CMD_SlowExit等）
- `data` - 命令附带数据

---

## 三、如何实现自定义调试工具

### 3.1 创建自定义DebuggingTool

```cpp
// 示例：自定义Workspot日志记录工具
class MyWorkspotLogger : public work::DebuggingTool
{
    RTTI_DECLARE_POLYMORPHIC_TYPE( MyWorkspotLogger );

public:
    // 支持的调试模式
    virtual Bool SupportsCommand( DebuggerCommandData* data ) const override
    {
        return data->m_mode == work::WorkspotDebugMode::VisualLogOn;
    }

    // ① Workspot设置时记录
    virtual void OnWorkspotSetup(
        const ent::EntityID& actorId,
        const THandle< work::WorkspotTree >& resource,
        const String& path
    ) override
    {
        // ⚠️ 关键：记录NPC使用了哪个Workspot
        LogToFile(
            "NPC [%s] Setup Workspot: %hs at Time: %f",
            actorId.ToDebugString(),
            path.AsChar(),
            GetCurrentGameTime()
        );
    }

    // ② Workspot开始播放时记录
    virtual void OnWorkspotStarted( const ent::EntityID& actorId ) override
    {
        LogToFile(
            "NPC [%s] Started Workspot at Time: %f",
            actorId.ToDebugString(),
            GetCurrentGameTime()
        );
    }

    // ③ 动画切换时记录
    virtual void OnAnimationChanged(
        const ent::EntityID& actorId,
        CName animName,
        WorkEntryId entryId,
        Uint32 flags
    ) override
    {
        LogToFile(
            "NPC [%s] Animation Changed: %s (Entry: %d, Flags: 0x%X)",
            actorId.ToDebugString(),
            animName.AsChar(),
            entryId.m_id,
            flags
        );
    }

    // ④ Workspot结束时记录
    virtual void OnWorkspotFinished( const ent::EntityID& actorId ) override
    {
        Float duration = GetCurrentGameTime() - m_startTimes[actorId];
        LogToFile(
            "NPC [%s] Finished Workspot, Duration: %.2f seconds",
            actorId.ToDebugString(),
            duration
        );
    }

    // 更新逻辑（可选）
    virtual Bool Update( Float dt, WorkspotSystem* sysPtr ) override
    {
        return true; // 返回true保持工具活跃
    }

private:
    void LogToFile( const char* format, ... )
    {
        // 实现日志写入逻辑
        // 可以使用RED_LOG_DEBUG或写入自定义文件
    }

    red::HashMap<ent::EntityID, Float> m_startTimes;
};

RTTI_BEGIN_TYPE_IN_NAMESPACE( MyWorkspotLogger, work )
RTTI_PARENT_TYPE( work::DebuggingTool )
RTTI_END_TYPE()
```

### 3.2 激活自定义调试工具

```cpp
// 在WorkspotSystem中激活调试工具
void ActivateMyLogger( WorkspotSystem& system, const ent::EntityID& actorId )
{
    work::DebuggerCommandData data;
    data.m_mode = work::WorkspotDebugMode::VisualLogOn;

    // 创建调试工具实例
    system.GetDebugger().CreateDebugTool(
        actorId,
        GetTypeObject<MyWorkspotLogger>(),
        &data,
        nullptr
    );
}
```

---

## 四、使用内置的日志系统

### 4.1 AI日志系统

```cpp
// workspotSystem.cpp: 356
RED_AI_LOGF( ownerId, "ws", "Workspot command sent: %hs",
             GetWorkspotCommandFriendlyName( cmd ).AsChar() );
```

**日志类别**: `"ws"` (Workspot)
**输出示例**:
```
[AI][EntityID:12345][ws] Workspot command sent: CMD_Play
[AI][EntityID:12345][ws] Workspot command sent: CMD_SlowExit
```

### 4.2 通用日志系统

```cpp
// workspotFunctionalTests.cpp: 50-57
void OnWorkspotSetup( const ent::EntityID& actorId, const String& path )
{
    RED_LOG_DEBUG( "%hs() path=%hs", __FUNCTION__, path.AsChar() );

    // 同时可以调用RedScript事件
    m_redscript->CallEvent( RED_NAME_CONSTEXPR( "OnWorkspotSetup" ), path );
}
```

**输出示例**:
```
[DEBUG] OnWorkspotSetup() path=workspots/restaurant_chair_01.workspot
```

---

## 五、实战调试场景

### 场景1：追踪特定NPC的Workspot使用

**目标**：找到场景中某个NPC（如 "NPC_Civilian_001"）什么时候开始执行哪个Workspot

#### 方法1：使用WorkspotFunctionalTestsDebuggingTool

```cpp
// 1. 在RedScript中注册监听器
class WorkspotListener extends IScriptable {
    private let m_targetNPCId: EntityID;

    public func OnWorkspotSetup(path: String) {
        // 记录Workspot路径
        LogChannel(n"Workspot", s"NPC started Workspot: \(path)");
    }

    public func OnWorkspotStarted() {
        LogChannel(n"Workspot", s"Workspot playback started");
    }
}

// 2. 激活调试工具
let data = new DebuggerCommandData();
data.m_mode = WorkspotDebugMode.FunctionalTests;

WorkspotSystem.SendCommand(npcEntityID, CMD_DebuggerCmd, data);
```

#### 方法2：直接查看日志

```cpp
// 在WorkspotSystem::SetupWorkspot()中添加断点或日志
// workspotSystem.cpp: 242-243

// 添加自定义日志
RED_LOG_DEBUG( "NPC [%s] Setup Workspot: %hs",
               ownerId.ToDebugString(),
               path.AsChar() );
```

**日志输出示例**:
```
[DEBUG] NPC [EntityID:0x00012345] Setup Workspot: workspots/restaurant_chair_01.workspot
[AI][EntityID:0x00012345][ws] Workspot command sent: CMD_Play
[DEBUG] NPC [EntityID:0x00012345] Started Workspot at Time: 125.43
```

---

### 场景2：追踪动画序列执行

**目标**：查看NPC在Workspot中执行了哪些动画

```cpp
// gameWorkspotsInstance.cpp中添加日志

void WorkspotInstance::UpdateRecord(...)
{
    m_iterator->GetData( data );

    // 添加日志
    if( data.m_animationName != m_currentData.m_animationName )
    {
        RED_LOG_DEBUG( "NPC [%s] Animation: %s (Idle: %s, Entry: %d)",
                       GetOwnerId().ToDebugString(),
                       data.m_animationName.AsChar(),
                       data.m_idleAnimName.AsChar(),
                       data.m_entryId.m_id );
    }
}
```

**日志输出示例**:
```
[DEBUG] NPC [EntityID:0x00012345] Animation: sit_look_menu (Idle: sit, Entry: 5)
[DEBUG] NPC [EntityID:0x00012345] Animation: sit_eat (Idle: sit, Entry: 6)
[DEBUG] NPC [EntityID:0x00012345] Animation: sit_drink (Idle: sit, Entry: 7)
```

---

### 场景3：使用可视化调试（DebugDrawer）

```cpp
// 在自定义DebuggingTool中实现OnRenderDebug

class MyWorkspotVisualizer : public work::DebuggingTool
{
    virtual void OnRenderDebug( const RenderDebugContext& context ) const override
    {
        // 在3D世界中绘制NPC的Workspot信息
        for (auto& entry : m_activeNPCs)
        {
            const Vector3& npcPos = entry.m_position;
            const String& workspotName = entry.m_workspotName;

            // 在NPC头顶显示Workspot名称
            context.DrawText3D(
                npcPos + Vector3(0, 0, 2.0f),
                workspotName,
                Color::Yellow
            );

            // 绘制从NPC到Workspot点位的连线
            context.DrawLine(
                npcPos,
                entry.m_workspotPosition,
                Color::Green
            );
        }
    }
};
```

---

### 场景4：性能分析

```cpp
// 使用Profiler追踪Workspot性能

#if defined(USE_PROFILER)
void WorkspotSystem::Update( Float dt )
{
    PC_SCOPE( "WorkspotSystem/Update" );

    for (auto& instance : m_instances)
    {
        PC_SCOPE_DYNAMIC( instance->GetWorkspotPath().AsChar() );

        // Profiler会显示每个Workspot的更新耗时
        UpdateInstance_NoLock( instance, timeToProcess );
    }
}
#endif
```

**Profiler输出**:
```
WorkspotSystem/Update: 2.3ms
  ├─ workspots/restaurant_chair_01.workspot: 0.8ms
  ├─ workspots/atm_machine.workspot: 0.5ms
  └─ workspots/bench_sit.workspot: 0.4ms
```

---

## 六、关键调试API总结

### 6.1 调试回调时序图

```
Character AI决定使用Workspot
  ↓
WorkspotSystem::SetupWorkspot()
  ├─ OnInstanceCreated(entityID)              ← 创建实例
  └─ OnWorkspotSetup(entityID, resource, path) ← ⚠️ 关键：记录Workspot路径
      ↓
WorkspotSystem::SendCommand(CMD_Play)
  └─ OnCommandReceived(entityID, CMD_Play)
      ↓
WorkspotInstance::Play()
  └─ OnWorkspotStarted(entityID)             ← 开始播放
      ↓
WorkspotInstance::UpdateRecord() [每帧调用]
  ├─ OnAnimationChanged(entityID, animName)  ← 动画切换
  ├─ OnAnimationSkipped(entityID, animName)  ← 动画跳过
  └─ OnAnimationMissing(entityID, animName)  ← 动画缺失
      ↓
WorkspotInstance::OnCompleted()
  └─ OnWorkspotFinished(entityID)            ← 结束播放
      ↓
WorkspotSystem::RemoveInstance()
  └─ OnInstanceRemoved(entityID)             ← 销毁实例
```

### 6.2 关键信息获取表

| 信息 | 回调时机 | 获取方式 |
|------|---------|----------|
| **NPC是谁** | 所有回调 | `ent::EntityID actorId` |
| **使用了哪个Workspot** | `OnWorkspotSetup` | `String path` (资源路径) |
| **什么时候开始** | `OnWorkspotStarted` | `m_workspotStartTime` |
| **当前播放的动画** | `OnAnimationChanged` | `CName animName` |
| **当前Entry ID** | `OnAnimationChanged` | `WorkEntryId entryId` |
| **Entry类型** | `OnAnimationChanged` | `Uint32 flags` (AnimClip/PauseClip) |
| **Workspot持续时长** | `OnWorkspotFinished` | `当前时间 - m_workspotStartTime` |

---

## 七、快速调试清单

### 想知道"NPC什么时候开始执行哪个Workspot"？

✅ **最直接的方法**：
1. 在 `WorkspotSystem::SetupWorkspot()` 的 **第242-243行** 添加断点
2. 运行游戏，等待断点触发
3. 检查变量：
   - `ownerId` → NPC的EntityID
   - `path` → Workspot资源路径（如 `"workspots/restaurant_chair_01.workspot"`）
   - `setup.m_workspot.m_tree` → WorkspotTree资源句柄

✅ **使用日志的方法**：
```cpp
// workspotSystem.cpp: 242后添加
RED_LOG_DEBUG( "[WORKSPOT TRACE] NPC [%s] using Workspot: %hs",
               ownerId.ToDebugString(),
               path.AsChar() );
```

✅ **使用AI日志查看**：
- 启用AI日志：`Game.EnableAILog(true, "ws")`
- 查找包含 `"Workspot command sent"` 的日志行
- 关联EntityID即可追踪特定NPC

✅ **使用调试工具**：
1. 创建自定义 `DebuggingTool`
2. 重写 `OnWorkspotSetup()` 回调
3. 记录到文件或屏幕显示

---

## 八、常见调试场景示例

### 示例1：NPC使用错误的Workspot

**症状**：NPC坐在椅子上但播放的是站立动画

**调试步骤**：
1. 在 `OnWorkspotSetup()` 检查 `path` 是否正确
2. 在 `OnAnimationChanged()` 检查播放的动画名称
3. 验证WorkspotTree的EntryAnim是否正确

**输出示例**：
```
[WORKSPOT TRACE] NPC [0x12345] using Workspot: workspots/wrong_chair.workspot
[ANIMATION] sit_down → Expected: walk_to_sit ⚠️ 错误！
```

---

### 示例2：Workspot从未开始

**症状**：`OnWorkspotSetup()` 被调用，但 `OnWorkspotStarted()` 从未触发

**调试步骤**：
1. 检查 `CMD_Play` 命令是否被发送
2. 在 `OnCommandReceived()` 检查接收到的命令
3. 检查 `WorkspotInstance::Play()` 是否被调用

**可能原因**：
- AI行为树卡在 `BTTask_UseWorkspot::ExecuteTask()` 阶段
- Character未调用 `SendCommand(CMD_Play)`
- Workspot的Preconditions不满足

---

### 示例3：动画播放不连贯

**症状**：动画之间有明显的跳变

**调试步骤**：
1. 在 `OnAnimationChanged()` 记录每次动画切换
2. 检查 `m_blendTime` 是否合理
3. 检查是否有 `OnAnimationSkipped()` 被频繁调用

**输出示例**：
```
[ANIMATION] sit_idle → sit_eat (BlendTime: 0.5s) ✓ 正常
[ANIMATION] sit_eat → stand_up (BlendTime: 0.0s) ⚠️ 跳变！
```

---

## 九、总结

### 回答您的核心问题

**"如何Debug目前使用Workspot的角色和相应的AISpot等信息，找到场景中一个NPC什么时候开始执行的哪个Workspot？"**

**最佳实践**：

1. **在 `OnWorkspotSetup()` 回调中记录**：
   - 这是唯一能同时获取 **NPC ID** 和 **Workspot路径** 的地方
   - 路径示例：`"workspots/restaurant_chair_01.workspot"`

2. **在 `OnWorkspotStarted()` 回调中记录开始时间**：
   - 使用 `m_workspotStartTime` 获取精确的开始时间戳

3. **使用 `OnAnimationChanged()` 追踪播放进度**：
   - 查看Entry ID和动画名称
   - 验证动画序列是否符合预期

4. **启用AI日志查看命令流**：
   - 所有Workspot命令（Play、Exit等）都会记录到AI日志
   - 使用EntityID过滤特定NPC的日志

### 关键代码位置

| 需求 | 文件 | 行号 | 说明 |
|------|------|------|------|
| Workspot设置 | workspotSystem.cpp | 242-243 | **最关键的追踪点** |
| Workspot开始 | gameWorkspotsInstance.cpp | 322 | 播放开始时间点 |
| 动画切换 | gameWorkspotsInstance.cpp | 1243-1253 | 追踪动画序列 |
| 命令发送 | workspotSystem.cpp | 354 | 追踪所有命令 |

这就是完整的Workspot调试指南！您现在可以追踪任何NPC在任何时刻使用的任何Workspot了。
