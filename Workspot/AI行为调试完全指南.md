# AI行为调试完全指南

## 一、核心概念

Cyberpunk 2077的AI调试系统提供了多个ImGui面板，用于实时查看和调试NPC的行为、状态、命令和信号。

---

## 二、选择调试目标NPC

### 2.1 使用相机射线选择

**操作步骤**：
```
1. 在游戏中进入调试模式
2. 将相机对准目标NPC
3. 按小键盘【4】键（Numpad 4）
4. 系统会通过射线检测选中NPC
```

**代码实现**：
```cpp
// debugSystem.cpp: 302-327
THandle<game::Object> CDebugSystem::SelectObjectFromCameraLook()
{
    THandle<game::Object> selectedObject = nullptr;

    if ( const auto* physicsSystem = GetGameInstance()->GetScene().GetSystem< world::RuntimeSystemPhysics >())
    {
        Transform xCam;
        if (GetGameInstance()->GetSystem< ICameraSystem >().GetRenderCameraWorldTransform(xCam))
        {
            static float CAST_RANGE = 100.0f;
            const Vector4 castDir = xCam.GetForward();
            const Vector4 castStart = xCam.GetPosition();
            const Vector3 castEnd = castStart + castDir * CAST_RANGE;

            physics::TraceResult traceResult;
            static const auto filter = physics::QueryFilter::Make( RED_NAME_CONSTEXPR( "AI" ) );
            auto params = physics::RaycastParams::Create(castStart, castEnd, filter)
                                                .SetUseCase(physics::QueryUseCase::Debug);

            if (physicsSystem->SyncRaycast(params, traceResult) == physics::SyncResult::Hit)
            {
                selectedObject = traceResult.FindHitObject< Object >();
            }
        }
    }
    return selectedObject;
}
```

**说明**：
- 射线长度：100米
- 过滤器：只检测AI实体
- 选中后，NPC会被标记为 `m_lastSelected`
- 所有调试面板都会显示这个NPC的信息

### 2.2 验证选中状态

选中NPC后，打开任何AI调试面板（见下文），会显示该NPC的信息。如果未选中，会显示：
```
"No NPC selected."
```

---

## 三、调试面板概览

### 3.1 可用的调试面板

```
调试系统提供以下面板：
┌─────────────────────────────────────────────────┐
│ 1. NPC QA Debug           - 综合信息面板          │
│ 2. AI Debugger            - AI行为树调试          │
│ 3. AI Commands            - AI命令队列            │
│ 4. Signals                - 信号调试              │
│ 5. AI Log                 - AI日志（特定NPC）     │
│ 6. Move Controllers       - 移动控制器            │
│ 7. Failure Table          - 失败记录表            │
└─────────────────────────────────────────────────┘
```

### 3.2 访问方式

**通过F12菜单**（推测，基于RED Engine惯例）：
```
按F12 → Debug → AI
├─ NPC QA Debug
├─ AI Debugger
├─ AI Commands
├─ Signals
└─ AI Log
```

**或通过Console命令**（需要开发者模式）：
```
显示NPC QA面板：
Game.GetDebugSystem():ShowNPCQADebugger()

显示AI调试器：
Game.GetDebugSystem():ShowAIDebugger()
```

---

## 四、NPC QA Debug面板（综合面板）⭐⭐⭐⭐⭐

### 4.1 面板概述

**用途**：**最核心的调试面板**，为QA团队设计，用于bug报告截图。包含NPC的所有关键信息。

**窗口名称**：`"NPC QA Debug##Game"`

**代码入口**：
```cpp
// debugSystem.cpp: 1474-1477
void CDebugSystem::ShowNPCQADebugger()
{
    debug::NPCQAPanel::Show( m_lastSelected );
}
```

### 4.2 显示的信息类别

#### 1. 基本信息（Info Panel）

```cpp
// debugNPCQAPanel.cpp: 208-289
void NPCQAPanel::ShowInfoPanel( const THandle< game::Object >& object )
```

**显示内容**：
```
┌──────────────────────────────────────────────────┐
│ ENT. ID      : 0x00000ABC123456                  │  ← Entity ID
│ ENT. CLS.    : Puppet                            │  ← Entity类
│ RES. PATH    : base\characters\npcs\citizen.ent  │  ← 资源路径
│ DISP. NAME   : Random Citizen                    │  ← 显示名称
│ CH. REC      : Character.Citizen_Male_01         │  ← TweakDB角色记录
│ POS          : {100.5, 250.3, 10.0}              │  ← 世界坐标
│                 [TP 2 NPC] [TP 2 Player]         │  ← 传送按钮
│ SAVABLE      : YES                               │  ← 是否保存到存档
│ VISUAL TAGS  : Male, Average, Casual             │  ← 外观标签
└──────────────────────────────────────────────────┘
```

**功能**：
- **复制到剪贴板**：右键点击任何值 → "Copy to clipboard"
- **传送NPC**：
  - `TP 2 NPC`：将玩家传送到NPC位置
  - `TP 2 Player`：将NPC传送到玩家位置

#### 2. Workspot信息（Workspot Info）

```cpp
// debugNPCQAPanel.cpp: 604-653
void NPCQAPanel::ShowWorkspotInfo( const THandle< game::Object >& object )
```

**显示内容**：
```
┌──────────────────────────────────────────────────┐
│ IN WS        : YES                               │  ← 是否在Workspot中
│ WS RES       : workspots\animations\             │  ← Workspot资源路径
│                  sit_chair.workspot              │
│ LOC ID       : 12345                             │  ← Workspot位置ID
└──────────────────────────────────────────────────┘
```

**作用**：
- 快速确认NPC是否在Workspot中
- 查看当前使用的Workspot资源
- 定位Workspot实例（通过LOC ID）

#### 3. AI信息（AI Info Panel）

```cpp
// debugNPCQAPanel.cpp: 357-553
void NPCQAPanel::ShowAIInfoPanel( const THandle< game::Puppet >& object )
```

**显示内容**：
```
┌──────────────────────────────────────────────────┐
│ AI Component 信息：                              │
│ ├─ AI State       : Combat / Patrol / Idle      │
│ ├─ Combat Target  : Player (EntityID: 0x...)    │
│ ├─ Friendly Target: NPC_02 (EntityID: 0x...)    │
│ ├─ LOD Flags      : CRWD WRKS                   │
│ │   ↑ CRWD  = Crowd                             │
│ │   ↑ CNMT  = Cinematic                         │
│ │   ↑ WRKS  = Workspot Static                   │
│ │   ↑ VHCL  = Vehicle                           │
│ │   ↑ USEW  = Using Workspot Through UseWorkspot│
│ └─ Archetype      : Aggressive / Passive        │
└──────────────────────────────────────────────────┘
```

**LOD标志说明**：
```cpp
// debugNPCQAPanel.cpp: 321-349
void ShowLODFlags( AI::CAgent* ai )
{
    const auto lodContext = ai->GetBehaviorEnvironment().GetLODContext();

    LOD_FLAG_( lodContext.IsCrowd(), "CRWD", "Crowd" );
    LOD_FLAG_( lodContext.IsCinematic(), "CNMT", "Cinematic" );
    LOD_FLAG_( lodContext.IsWorkspotStatic(), "WRKS", "Workspot Static" );
    LOD_FLAG_( lodContext.IsVehicle(), "VHCL", "Vehicle" );
    LOD_FLAG_( lodContext.IsUsingWorkspotThroughUseWorkspotAction(), "USEW",
               "Using Workspot Through 'UseWorkspot' Action" );
}
```

**关键信息**：
- **WRKS**：NPC在Workspot中且使用静态LOD
- **USEW**：NPC正在通过UseWorkspot动作进入Workspot
- 这些标志影响AI行为的更新频率和复杂度

#### 4. 行为树列表（Behaviors）

```cpp
// debugNPCQAPanel.cpp: 555-594
void NPCQAPanel::ShowAIBehaviors( const THandle< game::Puppet >& object )
```

**显示内容**：
```
┌──────────────────────────────────────────────────┐
│ BEHAVIORS                                        │
│ ├─ behaviors\npc\citizen_idle.behavior (ACTIVE) │  ← 绿色=激活
│ ├─ behaviors\npc\citizen_patrol.behavior        │  ← 白色=未激活
│ └─ behaviors\npc\citizen_combat.behavior        │
└──────────────────────────────────────────────────┘
```

**颜色含义**：
- **绿色**（`c_activeColor`）：行为树当前激活
- **白色**（`c_valColor`）：行为树加载但未激活

**悬停提示**（Hover Tooltip）：
```
Resource Path: base\behaviors\npc\citizen_idle.behavior
Is Active? YES!
```

#### 5. 动作列表（Actions）

```cpp
// debugNPCQAPanel.cpp: 597-601
void NPCQAPanel::ShowActions( const THandle< game::Puppet >& object )
{
    object->GetActionsContainerForEdit().Debug_ShowQAInfo( *this );
    ImGui::Separator();
}
```

**显示内容**：
```
当前执行的动作队列（如：Walk, Attack, UseWorkspot等）
```

#### 6. 挂载信息（Mounting Info）

```cpp
// debugNPCQAPanel.cpp: 656-692
void NPCQAPanel::ShowMountingInfo( const THandle< game::Object >& object )
```

**显示内容**：
```
┌──────────────────────────────────────────────────┐
│ MOUNTED      : YES                               │
│ SLOT         : driver                            │  ← 挂载槽ID
│ PARENT       : 0x00000DEF456789                  │  ← 父实体ID（如车辆）
└──────────────────────────────────────────────────┘
```

**用途**：检查NPC是否在载具中、坐在椅子上等

#### 7. 布娃娃系统（Ragdoll Info）

```cpp
// debugNPCQAPanel.cpp: 695-761
void NPCQAPanel::ShowRagdollInfo( const THandle< game::Object >& object )
```

**显示内容**：
```
布娃娃物理状态（激活/未激活）
```

### 4.3 使用场景

**场景1：调试Workspot未触发**
```
问题：NPC没有进入Workspot

步骤：
1. 选中NPC（Numpad 4）
2. 打开 NPC QA Debug 面板
3. 查看 "IN WS" 字段
   - 如果显示 "NO" → NPC未进入Workspot
   - 检查 LOD 标志中是否有 "USEW"
     - 有 "USEW" → UseWorkspot动作正在执行
     - 无 "USEW" → UseWorkspot动作未触发
4. 查看 "BEHAVIORS" 部分
   - 确认是否有Workspot相关的行为树激活
5. 查看 "POS" 坐标
   - 与Workspot位置对比，确认距离是否合理
```

**场景2：调试NPC同步问题**
```
问题：Slave NPC没有同步到Master

步骤：
1. 选中 Master NPC（Numpad 4）
2. 打开 NPC QA Debug 面板
3. 记录 "WS RES" 和 "LOC ID"
4. 查看 "IN WS" → 应该是 "YES"
5. 切换选中 Slave NPC
6. 对比 "WS RES" 是否不同
7. 检查 "IN WS" 是否为 "NO"
   - 如果是 NO → Slave未进入Workspot，同步失败
```

---

## 五、AI Debugger面板（行为树调试）⭐⭐⭐⭐

### 5.1 面板概述

**用途**：查看和调试AI行为树的详细执行状态

**窗口名称**：`"AI Debugger##AI"`

**代码实现**：
```cpp
// debugSystem.cpp: 1358-1409
void CDebugSystem::ShowAIDebugger()
{
#ifdef RED_USE_IMGUI
    static const auto c_debuggerWindowName = RED_NAME_CONSTEXPR( "AI Debugger##AI" );

    if( !red::ImGui::IsWindowRequested( c_debuggerWindowName ) )
    {
        return;
    }

    if( !ImGui::Begin( c_debuggerWindowName.AsChar() ) )
    {
        ImGui::End();
        return;
    }

    auto lastSelected = m_lastSelected.ToHandle();

    if ( !lastSelected || !lastSelected->IsAttached() )
    {
        ImGui::Text( "No NPC selected." );
        ImGui::End();
        return;
    }

    const auto debuggable = lastSelected->GetDebuggableAgent();
    if ( debuggable )
    {
        CDebuggableAgent::TBehaviorDebuggerList debuggers;
        debuggable->GetSortedDebuggers( debuggers );

        // 动作追踪输入框
        red::String& tracedAction = debuggable->GetTracedActionRef();
        const Uint32 capacity = 64;
        char name[ capacity ] = "";
        red::Strcpy( name, tracedAction.AsChar(), capacity, tracedAction.Length() );

        ImGui::InputText( "Debug action name", name, capacity );
        debuggable->SetTracedAction( red::String( name ) );

        // 显示所有行为树调试器
        for ( const auto& debugger : debuggers )
        {
            debugger->OnDrawGui( false );
        }
    }
    else
    {
        ImGui::Text( "Selected object doesn't have CDebuggableAgent." );
    }

    ImGui::End();
#endif
}
```

### 5.2 功能详解

#### 1. 动作追踪（Debug Action Name）

**输入框**：`"Debug action name"`

**用途**：输入特定动作名称，系统会追踪并高亮显示该动作的执行

**示例**：
```
输入：  "MoveToWorkspot"
结果：  行为树中所有涉及"MoveToWorkspot"的节点会被高亮
```

#### 2. 行为树可视化

**显示内容**：
```
每个激活的行为树会展开显示其节点结构：

Behavior Tree: citizen_idle.behavior
├─ Selector
│  ├─ Sequence: Check_Workspot
│  │  ├─ Condition: Is_Workspot_Available  [RUNNING]
│  │  ├─ Action: Move_To_Workspot          [SUCCESS]
│  │  └─ Action: Enter_Workspot            [RUNNING] ← 当前节点
│  └─ Sequence: Idle_Animation
│     └─ Action: Play_Idle_Anim            [IDLE]
```

**节点状态**：
- **RUNNING**：正在执行
- **SUCCESS**：执行成功
- **FAILURE**：执行失败
- **IDLE**：未激活

#### 3. 行为树参数

**显示内容**：
```
Behavior Arguments:
├─ CombatTarget     : EntityID(0x...)
├─ FriendlyTarget   : EntityID(0x...)
├─ TargetPosition   : {100.5, 200.3, 10.0}
└─ MovementSpeed    : 3.5
```

### 5.3 使用场景

**场景：调试NPC未进入Workspot**
```
问题：NPC停在Workspot附近，但不进入

步骤：
1. 选中NPC（Numpad 4）
2. 打开 AI Debugger 面板
3. 在 "Debug action name" 输入：UseWorkspot
4. 查看行为树节点状态
   - 找到 UseWorkspot 相关的节点
   - 查看状态：
     - RUNNING → 正在执行
     - FAILURE → 执行失败（检查失败原因）
     - IDLE → 未触发（检查前置条件）
5. 展开父节点，查看条件检查：
   - Is_Workspot_Available？
   - Is_Within_Distance？
   - Is_Not_In_Combat？
6. 找到失败的条件节点，定位问题
```

---

## 六、AI Commands面板（命令队列）⭐⭐⭐⭐

### 6.1 面板概述

**用途**：查看AI命令队列，显示当前和等待执行的命令

**窗口名称**：`"AI Commands##AI_CAgent"`

**代码实现**：
```cpp
// ai.cpp: 1193-1228
void CAgent::DrawAICommandDebugger( game::Puppet* puppet )
{
#ifdef RED_USE_IMGUI
    static const auto windowName = RED_NAME_CONSTEXPR( "AI Commands##AI_CAgent" );

    if ( !red::ImGui::IsWindowRequested( windowName ) )
    {
        return;
    }

    if ( !ImGui::Begin( windowName.AsChar() ) )
    {
        ImGui::End();
        return;
    }

    if ( puppet == nullptr )
    {
        ImGui::Text( "No NPC selected." );
    }
    else if ( !puppet->IsAttached() )
    {
        ImGui::Text( "Selected NPC is not attached." );
    }
    else if ( !puppet->GetAI() )
    {
        ImGui::Text( "Selected puppet doesn't have AI." );
    }
    else
    {
        puppet->GetAI()->GetCommandQueue().ShowGUI();  // ← 显示命令队列
    }

    ImGui::End();
#endif
}
```

### 6.2 命令队列内容

**显示格式**：
```
┌──────────────────────────────────────────────────┐
│ AI Commands                                      │
├──────────────────────────────────────────────────┤
│ [0] CMD_Play         (Priority: HIGH)  [ACTIVE]  │
│     └─ Entry ID: 123                             │
│                                                  │
│ [1] CMD_JumpToEntry  (Priority: IMMEDIATE)      │
│     ├─ Entry ID: 456                             │
│     ├─ Immediate: true                           │
│     └─ ForceTime: 5.0                            │
│                                                  │
│ [2] CMD_Stop         (Priority: NORMAL)          │
└──────────────────────────────────────────────────┘
```

**命令类型**（与Workspot相关）：
```cpp
// Workspot系统常见命令：
- CMD_Play          : 开始播放Workspot
- CMD_Stop          : 停止Workspot
- CMD_JumpToEntry   : 跳转到特定Entry（同步机制使用）
- CMD_Pause         : 暂停Workspot
- CMD_Resume        : 恢复Workspot
```

### 6.3 使用场景

**场景：调试同步命令未发送**
```
问题：Slave没有收到Master的同步命令

步骤：
1. 触发Master的同步Entry
2. 选中 Slave NPC（Numpad 4）
3. 打开 AI Commands 面板
4. 查看命令队列：
   - 应该看到 CMD_JumpToEntry 命令
   - 检查参数：
     - Entry ID：是否是正确的同步Entry
     - Immediate：应该是 true
     - ForceTime：同步时间点
5. 如果没有看到命令：
   - 问题在Master端，同步命令未发送
   - 检查Master的m_masterOf列表
6. 如果看到命令但未执行：
   - 问题在Slave端，命令被阻塞
   - 检查优先级和前置条件
```

---

## 七、Signals面板（信号调试）⭐⭐⭐

### 7.1 面板概述

**用途**：查看AI信号表（BoolSignalTable）的状态

**窗口名称**：`"Signals##AI_CAgent"`

**代码实现**：
```cpp
// ai.cpp: 1231-1266
void CAgent::DrawSignalDebugger( game::Puppet* puppet )
{
#ifdef RED_USE_IMGUI
    static const auto windowName = RED_NAME_CONSTEXPR( "Signals##AI_CAgent" );

    if ( !red::ImGui::IsWindowRequested( windowName ) )
    {
        return;
    }

    if ( !ImGui::Begin( windowName.AsChar() ) )
    {
        ImGui::End();
        return;
    }

    if ( puppet == nullptr )
    {
        ImGui::Text( "No NPC selected." );
    }
    else if ( !puppet->GetAI() )
    {
        ImGui::Text( "Selected puppet doesn't have AI." );
    }
    else if ( !puppet->GetAI()->m_signals )
    {
        ImGui::Text( "Selected puppet has no signal table." );
    }
    else
    {
        puppet->GetAI()->m_signals->ShowGui();  // ← 显示信号表
    }

    ImGui::End();
#endif
}
```

### 7.2 信号表内容

**显示格式**：
```
┌──────────────────────────────────────────────────┐
│ Bool Signals                                     │
├──────────────────────────────────────────────────┤
│ WorkspotEntered           : TRUE                 │
│ WorkspotExited            : FALSE                │
│ InCombat                  : FALSE                │
│ TargetLost                : FALSE                │
│ AnimationFinished         : TRUE                 │
│ SyncReceived              : TRUE  ← 同步信号     │
└──────────────────────────────────────────────────┘
```

**常见信号**（与Workspot相关）：
```
Workspot相关信号：
- WorkspotEntered       : NPC进入Workspot时设置
- WorkspotExited        : NPC离开Workspot时设置
- SyncReceived          : 收到同步信号时设置
- AnimationFinished     : 动画播放完成时设置
- ReadyForSync          : Slave准备好接收同步时设置
```

### 7.3 使用场景

**场景：调试同步信号未传递**
```
问题：Slave没有响应Master的同步

步骤：
1. 触发Master的同步
2. 选中 Slave NPC（Numpad 4）
3. 打开 Signals 面板
4. 查找 "SyncReceived" 信号
   - TRUE → Slave收到了同步信号
   - FALSE → Slave未收到信号，问题在Master端
5. 查找 "WorkspotEntered" 信号
   - TRUE → Slave已进入Workspot
   - FALSE → Slave未进入，可能被其他条件阻塞
6. 查找 "ReadyForSync" 信号（如果存在）
   - TRUE → Slave准备好同步
   - FALSE → Slave还未就绪
```

---

## 八、AI Log面板（AI日志）⭐⭐⭐⭐

### 8.1 面板概述

**用途**：显示特定NPC的AI日志，只显示选中NPC的日志消息

**前提条件**：需要编译时启用 `RED_AI_DEBUG_LOG_ENABLED` 宏

**代码实现**：
```cpp
// debugSystem.cpp: 1417-1427
#if RED_CTSW( RED_AI_DEBUG_LOG_ENABLED )
void CDebugSystem::ShowAILog()
{
#ifdef RED_USE_IMGUI
    auto lastSelected = m_lastSelected.ToHandle();
    auto entityId = lastSelected ? lastSelected->GetEntityID() : ent::EntityID();

    AI::DebugLog::ShowGUI( entityId );  // ← 传入EntityID过滤日志
#endif
}
#endif
```

### 8.2 日志内容

**显示格式**：
```
┌──────────────────────────────────────────────────┐
│ AI Log (EntityID: 0x00000ABC123456)              │
├──────────────────────────────────────────────────┤
│ [12:34:56.123] [WorkspotSystem] Setup workspot  │
│     Origin: {100.5, 200.3, 10.0}                 │
│     Tree: workspots\sit_chair.workspot           │
│                                                  │
│ [12:34:56.456] [WorkspotSync] PlaySyncAnim      │
│     SlotName: sync_dance_couple_01               │
│     TimeToSync: 3.45                             │
│                                                  │
│ [12:34:56.789] [WorkspotSync] Sync slave        │
│     Slave EntityID: 0x00000DEF456789             │
│     Entry ID: 456                                │
│     ForceTime: 3.45                              │
│                                                  │
│ [12:34:57.012] [AI] Command executed            │
│     Command: CMD_JumpToEntry                     │
│     Result: Success                              │
└──────────────────────────────────────────────────┘
```

### 8.3 日志类别

**Workspot相关日志**：
```
[WorkspotSystem]    : Workspot系统消息
[WorkspotSync]      : 同步相关消息
[WorkspotInstance]  : 实例管理消息
[WorkspotEntry]     : Entry执行消息
```

**AI相关日志**：
```
[AI]                : AI通用消息
[AICommand]         : 命令执行消息
[AIBehavior]        : 行为树执行消息
[AISignal]          : 信号处理消息
```

### 8.4 使用场景

**场景：跟踪同步流程**
```
问题：不确定同步在哪一步失败

步骤：
1. 清空日志（如果有清空按钮）
2. 选中 Master NPC（Numpad 4）
3. 打开 AI Log 面板
4. 触发同步Entry
5. 观察日志输出：
   [WorkspotSync] PlaySyncAnim
     SlotName: sync_dance_couple_01  ← Master触发同步

6. 切换选中 Slave NPC
7. 观察Slave的日志：
   [WorkspotSync] Sync slave
     Entry ID: 456                    ← Slave收到同步命令

   [AI] Command executed
     Command: CMD_JumpToEntry         ← 命令执行
     Result: Success                  ← 成功

8. 如果某一步缺失，定位问题所在
```

---

## 九、Move Controllers面板（移动控制器）⭐⭐

### 9.1 面板概述

**用途**：显示NPC的移动控制器状态

**代码实现**：
```cpp
// debugSystem.cpp: 1447-1472
void CDebugSystem::ShowMoveControllers( const RenderDebugContext& context )
{
    red::DynArray< THandle< game::Puppet > > puppets{ red::PoolDebug() };
    puppets.Reserve( m_pinnedNPCs.Size() + 1 );

    // 可以显示所有"固定"的NPC的移动控制器
    // （代码中被注释掉，默认只显示选中的NPC）

    if( THandle< game::Puppet > puppet = Cast< game::Puppet >( m_lastSelected.ToHandle() ) )
    {
        red::alg::PushBackUnique( puppets, puppet );
    }

    for( THandle< game::Puppet > puppet : puppets )
    {
        move::Component& moveComponent = puppet->GetMovingAgent();
        moveComponent.RenderImGuiSeparateWindow( context.debugFilterMask );
    }
}
```

### 9.2 显示内容

```
┌──────────────────────────────────────────────────┐
│ Move Controller (EntityID: 0x...)                │
├──────────────────────────────────────────────────┤
│ Current Position  : {100.5, 200.3, 10.0}         │
│ Target Position   : {101.5, 200.3, 10.0}         │
│ Velocity          : {0.5, 0.0, 0.0}              │
│ Speed             : 0.5 m/s                      │
│ Movement Type     : Walk                         │
│ Is Moving         : YES                          │
│ Has Path          : YES                          │
│ Path Length       : 5.0 m                        │
└──────────────────────────────────────────────────┘
```

---

## 十、Failure Table面板（失败记录表）⭐⭐

### 10.1 面板概述

**用途**：显示AI行为树执行过程中的失败记录

**代码实现**：
```cpp
// debugSystem.cpp: 1435-1445
void CDebugSystem::ShowFailureTable()
{
    if ( auto lastSelected = m_lastSelected.ToHandle() )
    {
        AI::AIDebuggableAgent::ShowFailureTableGui( lastSelected->GetDebuggableAgent() );
    }
    else
    {
        AI::AIDebuggableAgent::ShowFailureTableGui( nullptr );
    }
}
```

### 10.2 显示内容

```
┌──────────────────────────────────────────────────┐
│ Failure Table                                    │
├──────────────────────────────────────────────────┤
│ [12:34:56] Condition_Is_Workspot_Available       │
│     Reason: Workspot already occupied            │
│                                                  │
│ [12:34:57] Action_Enter_Workspot                 │
│     Reason: Distance too far (5.0m > 3.0m)       │
│                                                  │
│ [12:34:58] Sequence_Use_Workspot                 │
│     Reason: Child node failed                    │
└──────────────────────────────────────────────────┘
```

**用途**：
- 快速查找行为树失败的原因
- 历史记录，可以回溯之前的失败

---

## 十一、调试工作流

### 11.1 典型调试流程

**问题：NPC没有进入Workspot**

```
步骤1：基础信息收集
├─ 选中NPC（Numpad 4）
├─ 打开 NPC QA Debug 面板
├─ 检查 "IN WS" → NO
└─ 记录 "POS" 坐标和 "ENT. ID"

步骤2：检查AI状态
├─ 查看 "LOD Flags"
│  └─ 是否有 "USEW" 标志？
│     ├─ 有 → UseWorkspot动作正在执行
│     └─ 无 → UseWorkspot动作未触发
├─ 查看 "BEHAVIORS"
│  └─ 是否有Workspot相关行为树激活？
└─ 查看 "Combat Target"
   └─ 是否在战斗中？（战斗状态可能阻止Workspot）

步骤3：深入行为树调试
├─ 打开 AI Debugger 面板
├─ 输入 "UseWorkspot" 到 "Debug action name"
├─ 查看行为树节点状态
│  ├─ 找到 UseWorkspot 相关节点
│  └─ 查看状态（RUNNING / FAILURE / IDLE）
└─ 检查父节点条件
   ├─ Is_Workspot_Available？
   ├─ Is_Within_Distance？
   └─ Is_Not_In_Combat？

步骤4：检查命令队列
├─ 打开 AI Commands 面板
├─ 查看是否有 Workspot 相关命令
│  ├─ CMD_Play
│  ├─ CMD_JumpToEntry
│  └─ CMD_Stop
└─ 检查命令状态和参数

步骤5：查看失败记录
├─ 打开 Failure Table 面板
├─ 查找 Workspot 相关失败记录
└─ 分析失败原因

步骤6：查看详细日志
├─ 打开 AI Log 面板
├─ 过滤 [WorkspotSystem] 消息
└─ 跟踪执行流程
```

### 11.2 同步调试流程

**问题：Slave没有同步到Master**

```
步骤1：检查Master状态
├─ 选中 Master NPC（Numpad 4）
├─ 打开 NPC QA Debug 面板
├─ 确认 "IN WS" → YES
├─ 记录 "WS RES" 和 "LOC ID"
└─ 打开 AI Log 面板
   └─ 查找 [WorkspotSync] PlaySyncAnim 消息
      ├─ 如果有 → Master触发了同步
      ├─ 记录 SlotName
      └─ 记录 TimeToSync

步骤2：检查Slave状态
├─ 选中 Slave NPC（Numpad 4）
├─ 打开 NPC QA Debug 面板
├─ 检查 "IN WS" → 应该是 YES
│  └─ 如果是 NO → Slave未进入Workspot
├─ 对比 "WS RES"
│  └─ 应该与Master不同（或相同，取决于设计）
└─ 打开 AI Log 面板
   └─ 查找 [WorkspotSync] Sync slave 消息
      ├─ 如果有 → Slave收到了同步命令
      └─ 如果没有 → 同步命令未发送

步骤3：检查命令传递
├─ （保持选中Slave）
├─ 打开 AI Commands 面板
├─ 查找 CMD_JumpToEntry 命令
│  ├─ 如果有 → 命令已发送
│  │  └─ 检查参数：
│  │     ├─ Entry ID：是否正确
│  │     ├─ Immediate：应该是 true
│  │     └─ ForceTime：同步时间点
│  └─ 如果没有 → 命令未发送，问题在Master端

步骤4：检查信号状态
├─ （保持选中Slave）
├─ 打开 Signals 面板
├─ 查找同步相关信号
│  ├─ SyncReceived → 应该是 TRUE
│  ├─ WorkspotEntered → 应该是 TRUE
│  └─ ReadyForSync → 应该是 TRUE
└─ 如果有信号为 FALSE，定位问题

步骤5：检查行为树
├─ 打开 AI Debugger 面板
├─ 输入 SlotName 到 "Debug action name"
├─ 查看同步Entry的执行状态
└─ 检查是否有失败节点

步骤6：对比Master和Slave的同步时间
├─ 查看Master的 AI Log
│  └─ 记录同步触发时间 T1
├─ 查看Slave的 AI Log
│  └─ 记录命令接收时间 T2
└─ 计算延迟 ΔT = T2 - T1
   └─ 如果延迟过大，可能导致同步失败
```

---

## 十二、常见问题与解决

### 问题1：面板没有出现

**症状**：打开调试菜单后，面板不显示

**可能原因**：
1. ImGui功能未启用（编译时宏）
2. 面板被其他窗口遮挡
3. 面板在屏幕外

**解决**：
```
1. 确认编译选项包含：
   - RED_USE_IMGUI
   - RED_DEBUG_DRAW_AI_IMGUI

2. 检查F12菜单中面板是否勾选

3. 重置ImGui布局（如果有重置选项）
```

### 问题2："No NPC selected"

**症状**：所有面板都显示 "No NPC selected."

**原因**：未选中任何NPC

**解决**：
```
1. 将相机对准NPC
2. 按小键盘【4】键（Numpad 4）
3. 确认射线检测命中（可能需要靠近）
4. 如果仍然无法选中：
   - 检查NPC是否有AI组件
   - 检查NPC是否已Attach到场景
   - 检查物理过滤器是否正确
```

### 问题3："Selected object doesn't have CDebuggableAgent"

**症状**：AI Debugger 面板显示此消息

**原因**：选中的对象不是Puppet，或没有AI组件

**解决**：
```
1. 确认选中的是NPC，而不是其他物体
2. 检查NPC类型：
   - 应该是 Puppet 类或其子类
3. 检查AI组件是否初始化：
   - 打开 NPC QA Debug 面板
   - 查看 "ENT. CLS." 字段
   - 应该显示 "Puppet" 或 "NPC"
```

### 问题4：日志面板为空

**症状**：AI Log 面板不显示任何日志

**可能原因**：
1. 编译时未启用 `RED_AI_DEBUG_LOG_ENABLED`
2. 日志级别设置过滤了AI日志
3. 选中的NPC没有产生日志

**解决**：
```
1. 确认编译选项包含：
   RED_AI_DEBUG_LOG_ENABLED

2. 检查日志级别设置

3. 触发NPC的AI行为，产生新日志
```

---

## 十三、高级技巧

### 13.1 固定NPC（Pinning NPCs）

**用途**：同时监控多个NPC

**方法**（推测，基于源码）：
```
1. 选中NPC（Numpad 4）
2. 按固定快捷键（可能是 Numpad 4 长按，或特定组合键）
3. NPC会被添加到 m_pinnedNPCs 列表
4. 某些面板会显示所有固定的NPC的信息
```

### 13.2 传送功能（Teleport）

**在 NPC QA Debug 面板中**：
```
[TP 2 NPC]      : 将玩家传送到NPC位置
                  用于快速接近调试目标

[TP 2 Player]   : 将NPC传送到玩家位置
                  用于测试特定场景
```

**代码实现**：
```cpp
// debugNPCQAPanel.cpp: 256-273
ImGui::SameLine();
if ( ImGui::Button( "TP 2 NPC" ) )
{
    NPCQAPanel::TeleportPlayerToNPC( object );
}

ImGui::SameLine();
if ( ImGui::Button( "TP 2 Player" ) )
{
    NPCQAPanel::TeleportNPCToPlayer( object );
}
```

### 13.3 复制信息到剪贴板

**方法**：
```
1. 在 NPC QA Debug 面板中
2. 右键点击任何文本值（EntityID, 坐标, 路径等）
3. 选择 "Copy to clipboard"
4. 信息会被复制到系统剪贴板
```

**支持的字段**：
```
- Entity ID
- Resource paths
- Display names
- Coordinates
- 任何可选中的文本
```

---

## 十四、总结

### 14.1 调试面板优先级

**按使用频率排序**：

1. **NPC QA Debug** ⭐⭐⭐⭐⭐
   - 最重要，包含所有基础信息
   - 应该首先打开的面板

2. **AI Log** ⭐⭐⭐⭐⭐
   - 实时跟踪执行流程
   - 详细记录所有事件

3. **AI Commands** ⭐⭐⭐⭐
   - 查看命令队列
   - 调试同步问题必用

4. **AI Debugger** ⭐⭐⭐⭐
   - 深入行为树调试
   - 查看执行状态

5. **Signals** ⭐⭐⭐
   - 检查信号传递
   - 调试事件触发

6. **Failure Table** ⭐⭐
   - 快速定位失败原因
   - 历史记录查询

7. **Move Controllers** ⭐⭐
   - 调试移动问题
   - 查看路径信息

### 14.2 快速参考

**选中NPC**：
```
Numpad 4（小键盘4）
```

**检查清单（Workspot问题）**：
```
□ NPC是否选中？           → "No NPC selected"
□ IN WS = YES？           → NPC是否在Workspot中
□ LOD有USEW标志？         → UseWorkspot动作是否执行
□ BEHAVIORS有激活？       → 行为树是否运行
□ AI Commands有命令？     → 命令是否发送
□ AI Log有消息？          → 执行流程是否正常
□ Signals正确？           → 信号传递是否成功
```

**同步调试检查清单**：
```
Master:
□ IN WS = YES
□ AI Log有 PlaySyncAnim
□ 记录 SlotName 和 TimeToSync

Slave:
□ IN WS = YES（或即将YES）
□ AI Commands有 CMD_JumpToEntry
□ AI Log有 Sync slave
□ Signals有 SyncReceived = TRUE
□ Entry ID 正确
□ ForceTime 正确
```

### 14.3 调试流程图

```
开始调试
    ↓
选中NPC（Numpad 4）
    ↓
打开 NPC QA Debug 面板 ← 必须首先打开
    ↓
检查基础信息（IN WS, POS, LOD等）
    ↓
    ├─→ 问题明确？
    │       ↓ YES
    │   打开对应的专项面板
    │   （AI Debugger / AI Commands / Signals）
    │
    └─→ NO
        ↓
    打开 AI Log 面板
        ↓
    跟踪执行流程
        ↓
    定位问题阶段
        ↓
    打开对应的专项面板深入调试
        ↓
    解决问题
```

---

现在您可以使用这些强大的调试工具来诊断和解决AI行为问题了！
