# Patrol系统自动使用Entity中Workspot的完整流程

## 一、整体架构

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                           Patrol系统架构                                        │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ┌─────────────────┐                                                           │
│  │ PatrolSplineNode │  ← 编辑器中配置的巡逻路径                                  │
│  │  + patrolPoints  │     包含多个PatrolSplinePointDefinition                   │
│  └────────┬────────┘                                                           │
│           │                                                                    │
│           ▼                                                                    │
│  ┌──────────────────────┐                                                      │
│  │ PatrolSplineProgress │  ← 运行时的巡逻进度管理                                │
│  │  Initialize()        │     解析所有巡逻点的Workspot数据                       │
│  │  ComputePatrolPath() │     计算实际的控制点序列                               │
│  └────────┬─────────────┘                                                      │
│           │                                                                    │
│           ▼                                                                    │
│  ┌─────────────────┐       ┌───────────────────────────┐                       │
│  │  ActionPatrol   │──────▶│ ProcessControlPoint()     │                       │
│  │  (巡逻Action)    │       │ 到达控制点时处理Workspot   │                       │
│  └─────────────────┘       └───────────┬───────────────┘                       │
│                                        │                                       │
│                                        ▼                                       │
│                            ┌───────────────────────────┐                       │
│                            │ SetupWorkspotActionEvent  │  ← 输出给行为树       │
│                            │ DependentWorkspotData     │    触发Workspot动画   │
│                            └───────────────────────────┘                       │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、关键数据结构

### 2.1 PatrolSplinePointDefinition (巡逻点定义)

```cpp
// patrolSplinePointDefinition.h
enum class PatrolSplinePointTypes
{
    Workspot,   // Workspot点位
    LookAt,     // 看向目标点
    ClearLookAt // 清除看向
};

class PatrolSplinePointDefinition : public ISerializable
{
    PatrolSplinePointTypes m_pointType;  // 点类型
    NodeRef m_node;                       // 节点引用（可以是AISpot或Entity）
    game::EntityReference m_target;       // LookAt目标
};
```

### 2.2 PatrolSplineControlPoint (运行时控制点)

```cpp
// patrolProgressDefinitions.h:29-60
struct PatrolSplineControlPoint
{
    // 点类型判断
    Bool IsWorkspotPoint() const { return m_pointType == PatrolSplinePointTypes::Workspot; }
    Bool IsLookAtPoint() const { return m_pointType == PatrolSplinePointTypes::LookAt; }

    world::PatrolSplinePointTypes m_pointType;

    // ⚠️ 关键：ExtendedWorkspotSetup 包含从Entity提取的Workspot参数
    world::NodeRef m_workspotNodeRef;
    ExtendedWorkspotSetup m_workspot;        // ← 这里存储从Entity提取的数据
    WorldTransform m_workspotTransform;
    work::WorkEntryId m_infiniteSequenceWorkspotId;
    Bool m_isWorkspotStatic;

    // Entry/Exit点信息
    work::EntryPoint m_entryPoint;
    work::EntryPoint m_exitPoint;
    Vector4 m_workspotEntryPosition;
    Vector4 m_workspotExitPosition;

    // Spline上的位置参数
    Float m_splineExitPositionParam;
    Float m_splineReturnPositionParam;
    SplinePointIndex m_splineExitSectionIndex;
    SplinePointIndex m_splineReturnSectionIndex;

    // 其他
    game::EntityReference m_lookAtTarget;
    red::SharedPtr<work::WorkspotGlobalItemManager> m_globalItemManager;
};
```

### 2.3 ExtendedWorkspotSetup (扩展Workspot设置)

```cpp
// workspotSetup.h:103-115
struct ExtendedWorkspotSetup
{
    Bool IsSynced() const;  // 检查是否需要同步
    Bool ExtractWorkspotParameters( const THandle< world::INodeInstance >& selectedSpot );

    work::WorkspotParams m_mainWorkspotParams;    // NPC的Workspot参数
    work::WorkspotParams m_syncedWorkspotParams;  // 设备的同步Workspot参数
    THandle< ent::Entity > m_dependentEntity;     // 同步的依赖Entity
    CName m_syncSlotName;                         // 同步槽名称
};
```

---

## 三、完整流程详解

### 阶段1：初始化 - PatrolSplineProgress::Initialize()

```cpp
// patrolProgressDefinitions.cpp:525-585
void PatrolSplineProgress::Initialize( const world::RuntimeScene& scene,
                                       const Spline& spline,
                                       const red::DynArray< THandle< PatrolSplinePointDefinition > >& patrolPoints,
                                       ... )
{
    for ( const auto& point : patrolPoints )
    {
        ResolvedWorkspot resolvedWorkspot;

        // 1. 解析节点引用
        auto nodeInstance = AI::runtimeUtils::ResolveNode( scene, point->GetNodeRef() );

        switch ( point->GetPointType() )
        {
        case PointTypes::Workspot:
            // ⚠️ 关键调用：GetWorkspotData 提取Workspot数据
            if ( !AI::runtimeUtils::GetWorkspotData( nodeInstance,
                                                     resolvedWorkspot.m_parameters,  // ExtendedWorkspotSetup
                                                     resolvedWorkspot.m_transform,
                                                     isWorkspotInfinite,
                                                     resolvedWorkspot.m_isWorkspotStatic,
                                                     resolvedWorkspot.m_clippingSpace,
                                                     resolvedWorkspot.m_globalItemManager ) )
            {
                continue;  // 提取失败则跳过
            }
            break;
        }

        m_resolvedWorkspots.PushBack( resolvedWorkspot );
    }
}
```

### 阶段2：提取Workspot数据 - GetWorkspotData() + ExtractWorkspotParameters()

```cpp
// aiRuntimeUtils.cpp:93-123
Bool GetWorkspotData( const world::NodeInstancePtr& nodeInstance,
                      game::ExtendedWorkspotSetup& outWorkspotParams,
                      ... )
{
    // ⚠️ 核心：调用 ExtractWorkspotParameters
    if ( !nodeInstance || !outWorkspotParams.ExtractWorkspotParameters( nodeInstance ) )
    {
        return false;
    }

    outTransform = WorldTransform( nodeInstance->GetInitialTransform() );

    // 额外从AISpotNode获取属性
    if ( auto aiSpotInstance = Cast< world::AISpotNodeInstance >( nodeInstance ) )
    {
        if ( auto aiSpotNode = aiSpotInstance->GetSourceNodeAs< world::AISpotNode >() )
        {
            outIsWorkspotInfinite = aiSpotNode->GetIsWorkspotInfinite();
            outIsWorkspotStatic = aiSpotNode->GetIsWorkspotStatic();
            // ...
        }
    }

    return true;
}
```

```cpp
// workspotSetup.cpp:48-87
Bool ExtendedWorkspotSetup::ExtractWorkspotParameters( const THandle< world::INodeInstance >& selectedSpot )
{
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 路径1：AISpotNodeInstance（标准AISpot节点）
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    if ( THandle< const world::AISpotNodeInstance > spotInstance = Cast< ... >( selectedSpot ) )
    {
        if ( const AI::ActionSpotInstance* spot = Cast< ... >( spotInstance->GetSpotInstance() ) )
        {
            if ( THandle< work::WorkspotResource > workspot = spot->GetResource() )
            {
                m_mainWorkspotParams = work::WorkspotParams( workspot, ... );
                return true;
            }
        }
    }

    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 路径2：EntityNodeInstance（.ent实体）
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    else if ( THandle< const world::EntityNodeInstance > entityInstance = Cast< ... >( selectedSpot ) )
    {
        if ( const auto entity = entityInstance->GetCreatedEntity() )
        {
            if ( THandle< ent::GameEntity > workspotEntity = Cast< ent::GameEntity >( entity ) )
            {
                // 查找名为 "default" 的 WorkspotResourceComponent
                auto pred = []( const THandle< ent::IComponent >& component )
                {
                    return component->IsA< work::WorkspotResourceComponent >()
                        && ( component->GetName() == RED_NAME_CONSTEXPR_NOREG( "default" ) );
                };

                THandle< work::WorkspotResourceComponent > workspotComponent = ...;
                if ( workspotComponent )
                {
                    // ⚠️ 关键：从Component提取三个资源
                    m_mainWorkspotParams = work::WorkspotParams{
                        workspotComponent->m_npcResource.Get(),  // NPC用的Workspot
                        work::GenerateOriginId( selectedSpot->GetGlobalNodeID() )
                    };

                    m_syncedWorkspotParams = work::WorkspotParams{
                        workspotComponent->m_deviceResource.Get(),  // 设备用的Workspot
                        work::OriginId::Invalid()
                    };

                    m_dependentEntity = workspotEntity;  // 记录依赖的Entity
                    m_syncSlotName = workspotComponent->m_syncSlotName;  // 同步槽名
                    return true;
                }
            }
        }
    }
    return false;
}
```

### 阶段3：计算巡逻路径 - ComputePatrolPath()

```cpp
// patrolProgressDefinitions.cpp:587-640
void PatrolSplineProgress::ComputePatrolPath( const Object* puppet,
                                              move::MovementType movementType,
                                              Bool startFromClosestPoint,
                                              Bool reverseSpline )
{
    m_currentControlPoints.Clear();

    for ( Uint32 i = 0; i < m_resolvedWorkspots.Size(); ++i )
    {
        const auto& workspot = m_resolvedWorkspots[ ... ];
        PatrolSplineControlPoint splineWaypoint;

        if ( workspot.m_pointType == PointTypes::Workspot )
        {
            // 复制Workspot参数到控制点
            splineWaypoint.m_workspotNodeRef = workspot.m_nodeRef;
            splineWaypoint.m_workspot = workspot.m_parameters;  // ⚠️ ExtendedWorkspotSetup
            splineWaypoint.m_workspotTransform = workspot.m_transform;
            splineWaypoint.m_infiniteSequenceWorkspotId = workspot.m_infiniteSequenceWorkspotId;
            splineWaypoint.m_isWorkspotStatic = workspot.m_isWorkspotStatic;
            splineWaypoint.m_globalItemManager = workspot.m_globalItemManager;

            // 计算Entry/Exit点在Spline上的最佳位置
            InitializePatrolControlPoint( workspot.m_parameters.m_mainWorkspotParams,
                                          workspot.m_transform,
                                          workspot.m_clippingSpace,
                                          movementType,
                                          puppet,
                                          splineWaypoint );
        }

        m_currentControlPoints.EmplaceBack( splineWaypoint );
    }

    // 按Spline参数排序控制点
    if ( m_sortPatrolPoints )
    {
        std::sort( m_currentControlPoints.Begin(), m_currentControlPoints.End(), ... );
    }
}
```

### 阶段4：处理控制点 - ActionPatrol::ProcessControlPoint()

```cpp
// actionPatrol.cpp:405-470
void ActionPatrol::ProcessControlPoint( const PatrolSplineControlPoint& controlPoint )
{
    if ( controlPoint.IsLookAtPoint() && m_outLookAtTarget )
    {
        *m_outLookAtTarget = controlPoint.m_lookAtTarget;
    }
    else if ( controlPoint.IsWorkspotPoint() && m_outWorkspotData )
    {
        const auto& owner = m_movingAgent->GetOwner();

        // 检查Workspot是否启用
        AI::IWorkspotManager* workspotMgr = owner.GetGameSystem< AI::IWorkspotManager >();
        const world::GlobalNodeRef globalNodeRef = controlPoint.m_workspotNodeRef.Resolve( ... );

        if ( !workspotMgr->IsSpotEnabled( globalNodeId ) )
            return;

        // 预留Spot（防止冲突）
        AI::SpotUsageToken token = workspotMgr->SetSpotUser( globalNodeId, owner.GetEntityID(), ... );
        if ( !token.IsValid() )
            return;

        ps->SetUsedAISpot( std::move( token ) );

        // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        // 创建 SetupWorkspotActionEvent 输出
        // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        *m_outWorkspotData = CreateHandle< SetupWorkspotActionEvent >();

        // 使用 m_mainWorkspotParams (从Entity的m_npcResource提取)
        (*m_outWorkspotData)->m_parameters.m_workspot = controlPoint.m_workspot.m_mainWorkspotParams;
        (*m_outWorkspotData)->m_parameters.m_entryAnimation = controlPoint.m_entryPoint.m_animName;
        (*m_outWorkspotData)->m_parameters.m_exitAnimation = controlPoint.m_exitPoint.m_animName;
        (*m_outWorkspotData)->m_parameters.m_entryPositionLS = controlPoint.m_entryPoint.m_transform;
        (*m_outWorkspotData)->m_parameters.m_infiniteSequenceWorkspotId = controlPoint.m_infiniteSequenceWorkspotId;
        (*m_outWorkspotData)->m_parameters.m_isWorkspotStatic = controlPoint.m_isWorkspotStatic;
        (*m_outWorkspotData)->m_parameters.m_globalItemManager = controlPoint.m_globalItemManager;

        // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        // 处理同步Workspot (Entity场景)
        // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        if ( m_outDependentWorkspotData && controlPoint.m_workspot.IsSynced() )
        {
            *m_outDependentWorkspotData = CreateHandle< DependentWorkspotData >();

            // 设备Entity
            (*m_outDependentWorkspotData)->m_entity = controlPoint.m_workspot.m_dependentEntity;
            // 设备的Workspot参数 (m_deviceResource)
            (*m_outDependentWorkspotData)->m_workspotParams = controlPoint.m_workspot.m_syncedWorkspotParams;

            // 计算同步位置
            Transform syncTransformDeviceSpace;
            if ( owner.GetGame()->GetSystem< WorkspotGameSystem >().GetSyncWorkspotTransform(
                    controlPoint.m_workspot.m_syncedWorkspotParams,   // 设备Workspot
                    controlPoint.m_workspot.m_mainWorkspotParams,     // NPC Workspot
                    syncTransformDeviceSpace,
                    controlPoint.m_workspot.m_syncSlotName ) )        // 同步槽名
            {
                // 调整Workspot位置到设备空间
                const WorldTransform& deviceTransform = controlPoint.m_workspot.m_dependentEntity->GetWorldTransform();
                spotInPuppetLS = deviceTransform.TransformXForm( syncTransformDeviceSpace );
            }
        }

        (*m_outWorkspotData)->m_parameters.m_workspotTransform = spotInPuppetLS;
    }
}
```

---

## 四、流程图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            Patrol系统使用Entity Workspot流程                          │
└─────────────────────────────────────────────────────────────────────────────────────┘

                                    编辑器配置阶段
                                    ═════════════
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  PatrolSplineNode                                                                   │
│  ├── Spline (巡逻路径)                                                               │
│  └── patrolPoints[]                                                                 │
│        ├── [0] PatrolSplinePointDefinition                                          │
│        │         m_pointType = Workspot                                             │
│        │         m_node = "my_vending_machine"  ← 可以是Entity的NodeRef             │
│        │                                                                            │
│        └── [1] PatrolSplinePointDefinition                                          │
│                  m_pointType = LookAt                                               │
│                  m_node = "look_at_point"                                           │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          │ 游戏运行时
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  PatrolSplineProgress::Initialize()                                                 │
│                                                                                     │
│  for each patrolPoint:                                                              │
│      │                                                                              │
│      ├── ResolveNode( scene, point->GetNodeRef() )                                  │
│      │       │                                                                      │
│      │       └── 返回 EntityNodeInstance (如果指向.ent)                              │
│      │                                                                              │
│      └── GetWorkspotData( nodeInstance, ... )                                       │
│              │                                                                      │
│              └── ExtractWorkspotParameters( nodeInstance )                          │
│                      │                                                              │
│                      ├── Cast< EntityNodeInstance >  ✓                              │
│                      │                                                              │
│                      ├── 查找名为 "default" 的 WorkspotResourceComponent            │
│                      │                                                              │
│                      └── 提取:                                                      │
│                            m_mainWorkspotParams   = m_npcResource                   │
│                            m_syncedWorkspotParams = m_deviceResource                │
│                            m_dependentEntity      = Entity本身                      │
│                            m_syncSlotName         = Component的syncSlotName         │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  PatrolSplineProgress::ComputePatrolPath()                                          │
│                                                                                     │
│  构建 m_currentControlPoints[]                                                       │
│       │                                                                             │
│       └── PatrolSplineControlPoint                                                  │
│             m_workspot = ExtendedWorkspotSetup (包含NPC和Device的Workspot)           │
│             m_workspotTransform = Entity的Transform                                 │
│             m_entryPoint = 计算的Entry点                                            │
│             m_exitPoint = 计算的Exit点                                              │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  ActionPatrol (运行中)                                                               │
│                                                                                     │
│  NPC沿Spline移动...                                                                  │
│       │                                                                             │
│       ▼                                                                             │
│  到达控制点 → CheckForCompletion() → ProcessControlPoint()                           │
│       │                                                                             │
│       ├── 检查 IsSpotEnabled()                                                      │
│       │                                                                             │
│       ├── 预留 SetSpotUser()                                                        │
│       │                                                                             │
│       ├── 创建 SetupWorkspotActionEvent                                             │
│       │       m_parameters.m_workspot = m_mainWorkspotParams (NPC的Workspot)        │
│       │                                                                             │
│       └── 如果 IsSynced():                                                          │
│               创建 DependentWorkspotData                                            │
│                   m_entity = m_dependentEntity (设备Entity)                         │
│                   m_workspotParams = m_syncedWorkspotParams (设备Workspot)          │
│                                                                                     │
│               计算同步Transform通过 GetSyncWorkspotTransform()                       │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  行为树接收 SetupWorkspotActionEvent                                                 │
│                                                                                     │
│  → WorkspotSystem.SetupWorkspot()                                                   │
│  → NPC开始播放Workspot动画                                                           │
│                                                                                     │
│  如果有 DependentWorkspotData:                                                       │
│  → 设备Entity也开始播放对应的同步动画                                                 │
│  → 通过 syncSlotName 建立 Master-Slave 关系                                          │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 五、配置Entity的要求

要让Patrol系统能自动使用Entity中的Workspot，Entity必须满足：

### 5.1 Component命名要求

```
Entity (.ent)
└── Components
    └── WorkspotResourceComponent
          name = "default"  ← ⚠️ 必须命名为 "default"
```

代码验证:
```cpp
auto pred = []( const THandle< ent::IComponent >& component )
{
    return component->IsA< work::WorkspotResourceComponent >()
        && ( component->GetName() == RED_NAME_CONSTEXPR_NOREG( "default" ) );  // ⚠️
};
```

### 5.2 Component属性配置

```cpp
WorkspotResourceComponent:
    m_resource       = player_xxx.workspot    // Player使用
    m_npcResource    = npc_xxx.workspot       // ⚠️ NPC使用（Patrol系统读取这个）
    m_deviceResource = device_xxx.workspot    // 设备同步动画（可选）
    m_syncSlotName   = "sync_slot_name"       // 同步槽名（可选）
```

### 5.3 巡逻路径配置

```
PatrolSplineNode:
    └── patrolPoints[]
          └── PatrolSplinePointDefinition
                m_pointType = Workspot
                m_node = NodeRef 指向 Entity  ← 可以是Entity，不必是AISpot
```

---

## 六、同步机制触发条件

当以下条件都满足时，Patrol系统会触发同步Workspot：

```cpp
// actionPatrol.cpp:455
if ( m_outDependentWorkspotData && controlPoint.m_workspot.IsSynced() )
```

`IsSynced()` 的判断逻辑：
```cpp
// workspotSetup.cpp:43-46
Bool ExtendedWorkspotSetup::IsSynced() const
{
    return m_mainWorkspotParams.IsValid()      // NPC Workspot有效
        && m_syncedWorkspotParams.IsValid()    // 设备Workspot有效
        && m_dependentEntity                    // 依赖Entity存在
        && m_dependentEntity->IsInitialized(); // Entity已初始化
}
```

**结论**：只有当Entity的 `WorkspotResourceComponent` 同时配置了 `m_npcResource` 和 `m_deviceResource` 时，才会触发同步机制。

---

## 七、总结

| 步骤 | 函数 | 作用 |
|------|------|------|
| 1 | `PatrolSplineProgress::Initialize()` | 解析所有巡逻点，调用GetWorkspotData |
| 2 | `GetWorkspotData()` | 调用ExtractWorkspotParameters |
| 3 | `ExtractWorkspotParameters()` | **关键**：从AISpot或Entity提取Workspot参数 |
| 4 | `ComputePatrolPath()` | 构建运行时控制点列表 |
| 5 | `ActionPatrol::ProcessControlPoint()` | NPC到达时创建SetupWorkspotActionEvent |
| 6 | 行为树处理Event | 触发WorkspotSystem播放动画 |

**核心机制**：
- `ExtractWorkspotParameters()` 有两条路径：AISpotNode 和 EntityNode
- EntityNode路径会查找名为 `"default"` 的 `WorkspotResourceComponent`
- 从Component读取 `m_npcResource`（NPC用）和 `m_deviceResource`（设备同步用）
- 这使得Patrol系统能够自动使用Entity中内嵌的Workspot资源
