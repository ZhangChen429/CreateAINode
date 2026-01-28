# QuestNode中AssignAiRole与Workspot的关系

## 概述

在Quest系统中，`AssignAiRole`节点用于为NPC分配AI角色行为。其中的`Enter Closest`和`AlertedSpot`参数与Workspot系统有密切关系。

---

## 一、核心概念

### 1.1 PatrolPathParameters - 巡逻路径参数

```cpp
// patrolPathParameters.h:23-60
class PatrolPathParameters : public IScriptable
{
    world::NodeRef m_path;                    // 巡逻路径引用
    move::MovementType m_movementType;        // 移动类型
    Bool m_patrolWithWeapon = false;          // 是否持武器巡逻
    Bool m_enterClosest = true;               // ⚠️ Enter Closest
    Bool m_isBackAndForth = false;            // 往返巡逻
    Bool m_isInfinite = true;                 // 无限循环
    Uint32 m_numberOfLoops = 1;               // 循环次数
    Bool m_sortPatrolPoints = true;           // 排序巡逻点
};
```

### 1.2 PatrolCommand - 巡逻命令

```cpp
// aiMoveCommand.h:230-242
class PatrolCommand : public MoveCommand
{
    AI_COMMAND_PARAM( PathParameters, pathParams, THandle< AI::PatrolPathParameters > );
    AI_COMMAND_PARAM( AlertedPathParameters, alertedPathParams, THandle< AI::PatrolPathParameters > );
    AI_COMMAND_PARAM( AlertedRadius, alertedRadius, Float );
    AI_COMMAND_PARAM( AlertedSpots, alertedSpots, red::DynArray< world::NodeRef > );  // ⚠️ AlertedSpots
};
```

---

## 二、Enter Closest 详解

### 2.1 功能说明

`Enter Closest`（`m_enterClosest`）决定NPC如何进入巡逻路径：

| 值 | 行为 |
|---|------|
| `true` | NPC从**最近的巡逻点**开始巡逻 |
| `false` | NPC从**巡逻路径的起点**开始巡逻 |

### 2.2 代码实现

```cpp
// aiPatrolActionNode.cpp:115-116
Bool startFromClosestPoint = pathParameters->EnterClosest();  // 获取EnterClosest设置
m_definition.GetStartFromClosestPoint()->GetValue( context, startFromClosestPoint );
```

### 2.3 与Workspot的关系

当NPC到达巡逻点时，如果该点配置了Workspot，则：

1. **获取Workspot数据**：
```cpp
// aiPatrolActionNode.cpp:139-148
const game::ActionPatrol::SetupParams params(
    owner->GetMovingAgent(),
    *pathParameters,
    *patrolProgress,
    startFromClosestPoint,      // ⚠️ Enter Closest参数
    playStartAnimation,
    playStopAnimation,
    &context[ i_workspotData ],          // Workspot数据输出
    &context[ i_workspotExtraData ],     // 依赖的Workspot数据
    &context[ i_lookAtTarget ]
);
```

2. **Workspot数据结构**：
```cpp
// 输出参数定义
m_workspotData( CreateHandle< ArgumentMapping >( ArgumentType::Serializable,
    RED_NAME_CONSTEXPR( "gameSetupWorkspotActionEvent" ) ) )
m_dependentWorkspotData( CreateHandle< ArgumentMapping >( ArgumentType::Serializable,
    RED_NAME_CONSTEXPR( "gameDependentWorkspotData" ) ) )
```

---

## 三、AlertedSpot 详解

### 3.1 功能说明

`AlertedSpot`是NPC处于**警戒状态**时使用的特殊Workspot点位。

**使用场景**：
- NPC正常巡逻时使用普通巡逻路径
- 当NPC进入警戒状态（Alerted）时，切换到AlertedSpot

### 3.2 核心实现

#### 3.2.1 FindAlertedWorkspotTask

```cpp
// aiFindAlertedWorkspotTask.h:48-82
class FindAlertedWorkspotTask final : public Task
{
    // 输入参数
    THandle< ArgumentMapping > m_usedTokens;      // 已使用的Token列表
    THandle< ArgumentMapping > m_spots;           // AlertedSpot列表
    THandle< ArgumentMapping > m_radius;          // 搜索半径
    THandle< ArgumentMapping > m_outWorkspotData; // 输出的Workspot数据
};
```

#### 3.2.2 选择最近的AlertedSpot

```cpp
// aiFindAlertedWorkspotTask.cpp:64-123
Bool PickSpot( ent::Entity& owner, IWorkspotManager& workspotMgr,
               const red::DynArray< world::NodeRef >& spots,
               Float radius, ReservedWorkpotData& outReservedWorkspot )
{
    auto position = owner.GetWorldPosition().AsVector3();

    Float bestDistanceSq = pow( radius, 2 );  // 搜索半径平方

    for( auto spot : spots )
    {
        // 1. 解析节点引用
        world::GlobalNodeRef resolvedNodeRef = spot.Resolve( world::GlobalNodeID::GetRoot() );

        if( auto selectedSpotNode = Cast< world::AISpotNodeInstance >( node ) )
        {
            if( auto spot = Cast< const ActionSpotInstance >( selectedSpotNode->GetSpotInstance() ) )
            {
                // 2. 检查Workspot是否可用
                if( workspotMgr.IsSpotEnabled( selectedSpotNode->GetGlobalNodeID() ) )
                {
                    // 3. 计算距离
                    Float distanceSq = ( selectedSpotNode->GetInitialTransform().GetPosition()
                                        - position ).SquareMag3();

                    // 4. 选择最近的点
                    if ( distanceSq < bestDistanceSq )
                    {
                        // 5. 预留该Spot
                        SpotUsageToken token = workspotMgr.SetSpotUser(
                            selectedSpotNode->GetGlobalNodeID(),
                            owner.GetEntityID(),
                            IWorkspotManager::SpotUsageState::Reserved
                        );

                        if ( token.IsValid() )
                        {
                            bestNode = selectedSpotNode;
                            bestSpot = spot;
                            bestDistanceSq = distanceSq;
                            outReservedWorkspot.m_token = std::move( token );
                        }
                    }
                }
            }
        }
    }

    // 6. 获取Workspot资源
    if( bestNode != nullptr )
    {
        outReservedWorkspot.m_nodeId = bestNode->GetGlobalNodeID();
        outReservedWorkspot.m_workspot = work::WorkspotParams(
            bestSpot->GetResource(),
            work::GenerateOriginId( outReservedWorkspot.m_nodeId )
        );
        return true;
    }

    return false;
}
```

### 3.3 AlertedSpot数据结构

```cpp
// aiFindAlertedWorkspotTask.cpp:55-62
struct ReservedWorkpotData
{
    work::WorkspotParams m_workspot;    // Workspot参数（资源+OriginId）
    world::GlobalNodeID m_nodeId;       // 全局节点ID
    WorldTransform m_wsPosition;        // 世界空间位置
    SpotUsageToken m_token;             // 使用Token（防止冲突）
};
```

### 3.4 清理AlertedSpot

```cpp
// aiFindAlertedWorkspotTask.cpp:280-299
void ClearUsedAlertedSpotsTask::OnDeactivate( ExecutionContext& context ) const
{
    IWorkspotManager* workspotMgr = owner->GetGameSystem< IWorkspotManager >();

    // 遍历所有已使用的Token
    for( auto& token : tokensArray )
    {
        // 释放Spot占用
        workspotMgr->ClearSpotUser( std::move( token ) );
    }
}
```

---

## 四、整体流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     QuestNode: AssignAiRole                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────┐     ┌──────────────────────┐              │
│  │  PatrolPathParameters │     │    PatrolCommand     │              │
│  ├──────────────────────┤     ├──────────────────────┤              │
│  │ m_enterClosest=true  │────▶│ pathParams           │              │
│  │ m_path               │     │ alertedPathParams    │              │
│  │ m_movementType       │     │ alertedRadius        │              │
│  └──────────────────────┘     │ alertedSpots ────────┼──────┐       │
│                               └──────────────────────┘      │       │
│                                                             │       │
└─────────────────────────────────────────────────────────────│───────┘
                                                              │
                                                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         AI Behavior Tree                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│    正常状态                              警戒状态                    │
│    ─────────                            ─────────                   │
│         │                                    │                      │
│         ▼                                    ▼                      │
│  ┌──────────────┐                   ┌─────────────────────┐         │
│  │PatrolAction  │                   │FindAlertedWorkspot  │         │
│  │Node          │                   │Task                 │         │
│  ├──────────────┤                   ├─────────────────────┤         │
│  │EnterClosest: │                   │m_spots: AlertedSpots│         │
│  │选择最近巡逻点│                   │m_radius: 搜索半径   │         │
│  └──────┬───────┘                   │PickSpot(): 选最近的│         │
│         │                           └──────────┬──────────┘         │
│         │                                      │                    │
│         ▼                                      ▼                    │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │                     WorkspotSystem                       │       │
│  ├──────────────────────────────────────────────────────────┤       │
│  │  SetupWorkspot( entity, workspotParams )                 │       │
│  │  └─ 加载WorkspotTree                                     │       │
│  │  └─ 创建WorkspotInstance                                 │       │
│  │  └─ 播放动画序列                                         │       │
│  └──────────────────────────────────────────────────────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 五、Enter Closest vs AlertedSpot 对比

| 特性 | Enter Closest | AlertedSpot |
|------|--------------|-------------|
| **作用对象** | 巡逻路径上的点 | 独立的警戒点位 |
| **触发条件** | NPC开始巡逻时 | NPC进入警戒状态时 |
| **选择逻辑** | 选择最近的巡逻点开始 | 在指定半径内选择最近的AlertedSpot |
| **Workspot关系** | 巡逻点可能关联Workspot | AlertedSpot必须是Workspot |
| **使用Token** | 否 | 是（预留占用，防止冲突） |

---

## 六、实际使用示例

### 6.1 配置Enter Closest

```cpp
// Quest脚本/编辑器中
PatrolPathParameters params;
params.m_path = "path_to_patrol_spline";
params.m_enterClosest = true;   // NPC从最近的点开始巡逻
params.m_movementType = MovementType::Walk;

// 如果巡逻点有Workspot，NPC会在到达时使用
```

### 6.2 配置AlertedSpot

```cpp
// Quest脚本/编辑器中
PatrolCommand cmd;
cmd.pathParams = normalPatrolParams;
cmd.alertedPathParams = alertedPatrolParams;
cmd.alertedRadius = 10.0f;      // 10米搜索半径
cmd.alertedSpots = {
    "workspot_alert_pos_01",    // 警戒位置1（必须是AISpotNode）
    "workspot_alert_pos_02",    // 警戒位置2
    "workspot_alert_pos_03"     // 警戒位置3
};

// 当NPC进入警戒状态时：
// 1. FindAlertedWorkspotTask在alertedSpots中搜索
// 2. 选择alertedRadius范围内最近的点
// 3. 预留该点（Token机制）
// 4. NPC移动到该点并使用Workspot
```

---

## 七、关键要点总结

### 7.1 Enter Closest

1. **本质**：巡逻路径的入口点选择策略
2. **与Workspot关系**：间接关联，巡逻点可能配有Workspot
3. **代码位置**：`PatrolPathParameters::m_enterClosest`

### 7.2 AlertedSpot

1. **本质**：警戒状态专用的Workspot点位列表
2. **与Workspot关系**：直接关联，AlertedSpot就是Workspot
3. **特殊机制**：
   - 半径范围搜索
   - Token预留机制（防止多NPC冲突）
   - 与WorkspotManager深度集成
4. **代码位置**：`PatrolCommand::m_alertedSpots` + `FindAlertedWorkspotTask`

### 7.3 核心区别

```
Enter Closest:
  "NPC从哪里开始巡逻？" → 选择最近的巡逻路径点

AlertedSpot:
  "NPC警戒时去哪里？" → 选择最近的警戒Workspot点
```

---

## 八、与WorkspotSystem的集成点

1. **WorkspotParams传递**
   - 通过`SetupWorkspotActionEvent`传递Workspot参数

2. **IWorkspotManager接口**
   - `IsSpotEnabled()` - 检查Spot是否可用
   - `SetSpotUser()` - 设置Spot使用者（预留）
   - `ClearSpotUser()` - 清除Spot使用者

3. **ActionSpotInstance**
   - `GetResource()` - 获取WorkspotResource
   - 包含完整的WorkspotTree定义

4. **SpotUsageToken**
   - RAII机制管理Spot占用
   - 自动释放，防止泄漏
