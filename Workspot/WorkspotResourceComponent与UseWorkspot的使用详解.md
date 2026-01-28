# WorkspotResourceComponent 与 UseWorkspot 的使用详解

## 一、WorkspotResourceComponent 定义

```cpp
// workspotComponent.h:13-25
class WorkspotResourceComponent : public ent::IPlacedComponent
{
    TResRef< WorkspotResource > m_resource;       // Player用的Workspot资源
    TResRef< WorkspotResource > m_npcResource;    // NPC用的Workspot资源
    TResRef< WorkspotResource > m_deviceResource; // 设备(同步对象)用的Workspot资源
    CName m_syncSlotName;                         // 同步槽名称
};
```

### 1.1 三个Resource的用途

| 字段 | 用途 | 使用者 |
|------|------|--------|
| `m_resource` | 玩家的Workspot资源 | Player（注释说明为了向后兼容没有重命名） |
| `m_npcResource` | NPC的Workspot资源 | NPC走到该Entity时使用 |
| `m_deviceResource` | 设备/同步对象的Workspot资源 | 作为同步Workspot的从属资源 |

### 1.2 m_syncSlotName 的作用

`m_syncSlotName` 将该Entity标记为一个**同步Workspot点**。当NPC与该Entity交互时，系统会通过 `WorkspotSynchronizer` 使用这个slot name建立Master-Slave同步关系。

---

## 二、UseWorkspot 是否可以直接使用 .ent？

### 答案：不能直接使用，但间接支持

UseWorkspot的QuestNode要求 `m_workspotNode` 指向一个 **AISpotNode**，不能直接指向一个 .ent。

### 2.1 UseWorkspot的验证逻辑

```cpp
// useWorkspotNode.cpp:1539-1569
Bool UseWorkspotParamsV1::ValidateExecution( NodeExecutionContext& executionContext ) const
{
    if( m_function == UseWorkspotNodeFunctions::UseWorkspot )
    {
        const world::GlobalNodeRef resolvedNodeRef = m_workspotNode.Resolve(...);

        // ⚠️ 必须解析为 AISpotNodeInstance
        if ( auto nodeInstance = resolvedNodeRef.GetNodeInstance< const world::AISpotNodeInstance >( ... ) )
        {
            if ( THandle< const world::AISpotNodeInstance > aiSpot = Cast< const world::AISpotNodeInstance >( nodeInstance ) )
            {
                if ( const AI::ActionSpotInstance* actionSpotInstance = ... )
                {
                    if ( THandle< work::WorkspotResource > workspotResource = actionSpotInstance->GetResource() )
                    {
                        return static_cast< Bool >( workspotResource->m_workspotTree );
                    }
                }
            }
            return false;  // ⚠️ 不是AISpotNode则失败
        }
    }
    return true;
}
```

**结论**：UseWorkspot QuestNode **只接受 AISpotNode 的 NodeRef**。

---

## 三、.ent 上的 WorkspotResourceComponent 真正的使用场景

### 3.1 Patrol系统中的间接使用

`.ent` 上的 `WorkspotResourceComponent` 通过 `ExtendedWorkspotSetup` 被使用，这发生在 **巡逻（Patrol）系统** 中：

```cpp
// workspotSetup.cpp:48-87
Bool ExtendedWorkspotSetup::ExtractWorkspotParameters( const THandle< world::INodeInstance >& selectedSpot )
{
    // ━━━ 路径1：AISpotNode ━━━
    if ( THandle< const world::AISpotNodeInstance > spotInstance = Cast< const world::AISpotNodeInstance >( selectedSpot ) )
    {
        // 从AISpot获取WorkspotResource
        m_mainWorkspotParams = work::WorkspotParams( workspot, ... );
        return true;
    }

    // ━━━ 路径2：Entity节点（使用WorkspotResourceComponent）━━━
    else if ( THandle< const world::EntityNodeInstance > entityInstance = Cast< const world::EntityNodeInstance >( selectedSpot ) )
    {
        if ( const auto entity = entityInstance->GetCreatedEntity() )
        {
            // 查找名为"default"的WorkspotResourceComponent
            auto pred = []( const THandle< ent::IComponent >& component )
            {
                return component->IsA< work::WorkspotResourceComponent >()
                    && ( component->GetName() == RED_NAME_CONSTEXPR_NOREG( "default" ) );
            };

            if ( workspotComponent )
            {
                // ⚠️ 使用 m_npcResource 作为NPC的Workspot
                m_mainWorkspotParams = work::WorkspotParams{
                    workspotComponent->m_npcResource.Get(), ...
                };

                // ⚠️ 使用 m_deviceResource 作为同步Workspot
                m_syncedWorkspotParams = work::WorkspotParams{
                    workspotComponent->m_deviceResource.Get(), ...
                };

                // ⚠️ Entity自身成为同步依赖对象
                m_dependentEntity = workspotEntity;

                // ⚠️ 读取同步槽名
                m_syncSlotName = workspotComponent->m_syncSlotName;
                return true;
            }
        }
    }
    return false;
}
```

### 3.2 典型使用场景：NPC与设备交互

```
┌─────────────────────────────────────────────────────────────────┐
│                      .ent 资产（设备实体）                       │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │        WorkspotResourceComponent ("default")            │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ m_resource       = player_use_device.workspot           │    │
│  │ m_npcResource    = npc_use_device.workspot       ──────────▶ NPC的动画树  │
│  │ m_deviceResource = device_reaction.workspot      ──────────▶ 设备的动画树  │
│  │ m_syncSlotName   = "sync_slot_01"                ──────────▶ 同步槽       │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**举例**：NPC使用自动售货机

```
NPC（m_npcResource）            自动售货机（m_deviceResource）
    │                                  │
    ▼                                  ▼
┌────────────┐               ┌────────────────┐
│ npc_vending │  ◀─同步─▶    │ vending_react  │
│ _machine    │  syncSlot     │ _animation     │
│ .workspot   │  ="slot_buy"  │ .workspot      │
├────────────┤               ├────────────────┤
│ Entry: 走向│               │ Entry: 待机    │
│ Idle: 投币 │               │ Idle: 亮灯     │
│ Exit: 取货 │               │ Exit: 出货动画 │
└────────────┘               └────────────────┘
```

---

## 四、两种Workspot入口对比

```
                    ┌──────────────────────────────────────┐
                    │          Workspot使用入口              │
                    └──────────┬───────────────────────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                 │
    ┌─────────▼──────────┐           ┌──────────▼──────────┐
    │   UseWorkspot       │           │  Patrol / AI系统     │
    │   QuestNode         │           │  (ExtendedSetup)     │
    ├────────────────────┤           ├─────────────────────┤
    │ 输入: NodeRef       │           │ 输入: NodeRef        │
    │       ↓             │           │       ↓              │
    │ 必须→ AISpotNode    │           │ 可以→ AISpotNode     │
    │ ❌ 不接受 Entity    │           │ 可以→ Entity (.ent)  │
    │                    │           │                     │
    │ 读取:              │           │ 读取:               │
    │ ActionSpotInstance │           │ 路径1: AISpot资源    │
    │ → GetResource()    │           │ 路径2: Component的   │
    │                    │           │   m_npcResource     │
    │                    │           │   m_deviceResource  │
    │                    │           │   m_syncSlotName    │
    └────────────────────┘           └─────────────────────┘
```

---

## 五、如何让UseWorkspot与.ent间接配合

虽然UseWorkspot不能直接指向.ent，但可以通过以下方式配合：

### 5.1 在.ent旁放置AISpotNode

```
World中的布局:
    ├── VendingMachine.ent          ← 带有WorkspotResourceComponent
    │       └── WorkspotResourceComponent
    │             m_npcResource = npc_vending.workspot
    │             m_deviceResource = device_vending.workspot
    │             m_syncSlotName = "slot_buy"
    │
    └── VendingMachine_AISpot       ← AISpotNode（放在.ent旁边）
            └── ActionSpotInstance
                  resource = npc_vending.workspot   ← 同一个资源
```

UseWorkspot QuestNode指向 `VendingMachine_AISpot`，而Patrol系统可以直接通过 `VendingMachine.ent` 的 `WorkspotResourceComponent` 获取同样的资源。

### 5.2 Patrol系统的自动发现

Patrol系统中，当NPC到达巡逻点时，`ExtendedWorkspotSetup::ExtractWorkspotParameters` 会自动检测：
1. 先尝试Cast为 `AISpotNodeInstance`
2. 如果不是，再尝试Cast为 `EntityNodeInstance`
3. 如果是Entity，查找其 `WorkspotResourceComponent`

---

## 六、m_syncSlotName 的同步机制详解

### 6.1 在ExtendedWorkspotSetup中的使用

当Patrol系统检测到Entity上的WorkspotResourceComponent时：

```cpp
m_mainWorkspotParams = WorkspotParams{ workspotComponent->m_npcResource.Get(), ... };   // NPC的Workspot
m_syncedWorkspotParams = WorkspotParams{ workspotComponent->m_deviceResource.Get(), ... }; // 设备的Workspot
m_dependentEntity = workspotEntity;   // 设备Entity
m_syncSlotName = workspotComponent->m_syncSlotName;  // 同步槽名
```

### 6.2 同步流程

```
1. NPC到达巡逻点（Entity）
       │
2. ExtractWorkspotParameters 提取参数
       │
3. NPC使用 m_npcResource 的WorkspotTree
   设备Entity使用 m_deviceResource 的WorkspotTree
       │
4. 通过 m_syncSlotName 建立 Master-Slave 关系
       │
5. WorkspotSynchronizer.PlaySyncAnim()
   Master播放动画时，通过slotName找到Slave的对应SyncAnimClip
   向Slave发送 CMD_JumpToEntry 实现同步
```

### 6.3 与SyncAnimClip的关联

WorkspotTree中的 `SyncAnimClip` 节点通过 `m_slotName` 与 `WorkspotResourceComponent.m_syncSlotName` 对应：

```cpp
// workspotTreeItems.h
class SyncAnimClip : public AnimClip
{
    CName m_slotName;       // ⚠️ 必须与Component的m_syncSlotName匹配
    Transform m_syncOffset; // 同步偏移
};
```

---

## 七、总结

| 问题 | 答案 |
|------|------|
| **WorkspotResourceComponent的用途** | 将Workspot资源附加到Entity上，支持Player/NPC/Device三种资源 |
| **UseWorkspot能直接使用.ent吗？** | **不能**。UseWorkspot的`m_workspotNode`必须指向AISpotNode |
| **谁能使用.ent上的WorkspotResourceComponent？** | Patrol/AI系统通过`ExtendedWorkspotSetup`间接使用 |
| **m_syncSlotName的作用** | 建立NPC与设备Entity之间的动画同步关系 |
| **Component名称要求** | 必须命名为 `"default"` 才会被系统识别 |
| **m_resource vs m_npcResource** | `m_resource`是Player用的（历史遗留命名），`m_npcResource`是NPC用的 |
