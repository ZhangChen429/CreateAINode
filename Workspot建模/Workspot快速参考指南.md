# Workspot系统快速参考指南
> 常用操作、代码示例与配置模板

---

## 📋 目录

1. [快速开始](#快速开始)
2. [常用节点配置模板](#常用节点配置模板)
3. [代码示例](#代码示例)
4. [常见问题与解决方案](#常见问题与解决方案)
5. [性能优化清单](#性能优化清单)
6. [调试技巧](#调试技巧)

---

## 快速开始

### 创建简单的坐椅子Workspot

#### 步骤1：打开Workspot编辑器

```
1. 在REDengine编辑器中，创建新资源
2. 选择 WorkspotResource 类型
3. 命名：chair_sit_basic.workspot
```

#### 步骤2：设置基础配置

```json
{
  "workspotRig": "base\\characters\\entities\\citizen\\citizen_male_average.rig",
  "animGraphSlotName": "WORKSPOT",
  "autoTransitionBlendTime": 1.0,
  "blendOutTime": 0.5
}
```

#### 步骤3：构建节点树

```
RootEntry: Sequence
├─ Entry[0]: EntryAnim
│   ├─ animName: "walk_to_chair"
│   ├─ movementType: Walk
│   └─ orientationType: Forward
│
├─ Entry[1]: AnimClip
│   ├─ animName: "sit_down"
│   └─ blendOutTime: 0.3
│
├─ Entry[2]: Sequence (loopInfinitely=true)
│   ├─ idleAnim: "sit_idle"
│   └─ Entry[0]: PauseClip
│       ├─ timeMin: 2.0
│       └─ timeMax: 5.0
│
└─ Entry[3]: ExitAnim
    ├─ animName: "stand_up"
    └─ movementType: Walk
```

#### 步骤4：放置到场景

```
1. 在场景中创建Entity
2. 添加 WorkspotResourceComponent
3. 设置 m_resource = chair_sit_basic.workspot
4. 调整Transform（位置和朝向）
```

---

## 常用节点配置模板

### 模板1：基础Idle循环

```json
{
  "type": "Sequence",
  "id": 1,
  "loopInfinitely": true,
  "idleAnim": "base_idle",
  "list": [
    {
      "type": "RandomList",
      "id": 2,
      "minClips": 1,
      "maxClips": 1,
      "dontRepeatLastAnims": 0,
      "weights": [0.5, 0.3, 0.2],
      "list": [
        {
          "type": "AnimClip",
          "id": 3,
          "animName": "idle_variant_01"
        },
        {
          "type": "AnimClip",
          "id": 4,
          "animName": "idle_variant_02"
        },
        {
          "type": "PauseClip",
          "id": 5,
          "timeMin": 1.0,
          "timeMax": 3.0
        }
      ]
    }
  ]
}
```

### 模板2：带物品的动作序列

```json
{
  "type": "Sequence",
  "id": 10,
  "list": [
    {
      "type": "AnimClipWithItem",
      "id": 11,
      "animName": "pick_up_bottle",
      "itemActions": [
        {
          "type": "WorkspotItemAction_SpawnProp",
          "propId": "beer_bottle",
          "slotId": "hand_r"
        }
      ]
    },
    {
      "type": "RandomList",
      "id": 12,
      "minClips": 3,
      "maxClips": 5,
      "list": [
        {
          "type": "AnimClip",
          "id": 13,
          "animName": "drink_sip"
        },
        {
          "type": "AnimClip",
          "id": 14,
          "animName": "hold_bottle_idle"
        }
      ]
    },
    {
      "type": "AnimClipWithItem",
      "id": 15,
      "animName": "put_down_bottle",
      "itemActions": [
        {
          "type": "WorkspotItemAction_RemoveProp",
          "propId": "beer_bottle"
        }
      ]
    }
  ]
}
```

### 模板3：反应序列

```json
{
  "type": "ReactionSequence",
  "id": 20,
  "reactionTypes": [
    "Interactions.Greeting",
    "Interactions.Scared",
    "Interactions.Combat"
  ],
  "mainEmotionalState": "neutral",
  "emotionalExpression": "default",
  "facialKeyWeight": 1.0,
  "list": [
    {
      "type": "AnimClip",
      "id": 21,
      "animName": "greeting_wave"
    },
    {
      "type": "Sequence",
      "id": 22,
      "list": [
        {
          "type": "AnimClip",
          "id": 23,
          "animName": "scared_duck"
        },
        {
          "type": "FastExit",
          "id": 24,
          "animName": "run_away"
        }
      ]
    },
    {
      "type": "FastExit",
      "id": 25,
      "animName": "combat_exit"
    }
  ]
}
```

### 模板4：条件序列（多体型支持）

```json
{
  "type": "ConditionalSequence",
  "id": 30,
  "multipleConditionOperator": "OR",
  "conditionList": [
    {
      "type": "WorkspotCondition_RigPath",
      "rigPath": "base\\characters\\entities\\citizen\\citizen_male_big.rig"
    }
  ],
  "list": [
    {
      "type": "AnimClip",
      "id": 31,
      "animName": "sit_down_large_male"
    }
  ]
}
```

### 模板5：同步主从动画

**Master Workspot:**
```json
{
  "type": "SyncMasterEntryAnim",
  "id": 40,
  "animName": "handshake_master_entry",
  "isSynchronized": true,
  "slotName": "handshake_slave_slot",
  "syncOffset": {
    "position": {"x": 0.0, "y": 0.8, "z": 0.0},
    "orientation": {"r": 1.0, "i": 0.0, "j": 0.0, "k": 0.0}
  }
}
```

**Slave Workspot:**
```json
{
  "type": "EntryAnim",
  "id": 50,
  "animName": "handshake_slave_entry",
  "isSynchronized": true,
  "slotName": "handshake_slave_slot",
  "syncOffset": {
    "position": {"x": 0.0, "y": 0.0, "z": 0.0},
    "orientation": {"r": 1.0, "i": 0.0, "j": 0.0, "k": 0.0}
  }
}
```

---

## 代码示例

### 示例1：从C++代码设置Workspot

```cpp
#include "workspotSystem.h"
#include "workspotResource.h"

void SetupNPCWorkspot( ent::Entity* npc, const res::ResourcePath& workspotPath )
{
    // 获取WorkspotSystem
    work::WorkspotSystem* workspotSystem = GetGame()->GetWorkspotSystem();

    // 加载Workspot资源
    TResRef< work::WorkspotResource > workspotRes = LoadResource( workspotPath );
    if ( !workspotRes )
    {
        RED_LOG_ERROR( Workspot, "Failed to load workspot: %s", workspotPath.AsChar() );
        return;
    }

    // 创建WorkspotParams
    work::WorkspotParams params( workspotRes, work::OriginId::CreateUnique() );

    // 设置WorkspotSetupContext
    work::WorkspotSetupContext setupContext;
    setupContext.m_workspot = params;
    setupContext.m_nodeId = /* WorldNodeID */;
    setupContext.m_commFun = red::MakeShared< work::NullWorkspotInstanceCommFunc >();
    setupContext.m_entryAnim = CName::NONE(); // 自动选择
    setupContext.m_exitAnim = CName::NONE();  // 自动选择

    // 设置Workspot
    workspotSystem->SetupWorkspot( THandle< ent::Entity >( npc ), setupContext );

    // 发送播放命令
    workspotSystem->SendCommand( npc->GetEntityID(), work::CMD_Play );
}
```

### 示例2：监听Workspot事件

```cpp
class MyWorkspotListener : public work::IWorkspotListener
{
public:
    void OnWorkspotStarted( const ent::EntityID& entityId ) override
    {
        RED_LOG( Quest, "NPC %llu entered workspot", entityId.GetValue() );
        // 触发Quest目标
    }

    void OnWorkspotEnded( const ent::EntityID& entityId ) override
    {
        RED_LOG( Quest, "NPC %llu left workspot", entityId.GetValue() );
        // 完成Quest目标
    }

    void OnAnimationChanged( const ent::EntityID& entityId, work::WorkEntryId entryId ) override
    {
        RED_LOG( Workspot, "Animation changed to entry %d", entryId.GetValue() );
    }

    void OnReactionTriggered( const ent::EntityID& entityId, CName reactionType ) override
    {
        RED_LOG( Workspot, "Reaction triggered: %s", reactionType.AsChar() );
    }
};

// 注册监听器
void RegisterWorkspotListener()
{
    work::WorkspotSystem* workspotSystem = GetGame()->GetWorkspotSystem();
    red::SharedPtr< MyWorkspotListener > listener = red::MakeShared< MyWorkspotListener >();
    workspotSystem->RegisterCallback( listener );
}
```

### 示例3：从Quest控制Workspot

```cpp
class QuestWorkspotController
{
    work::WorkspotSystem* m_workspotSystem;
    ent::EntityID m_npcId;

public:
    void StartWorkspot()
    {
        m_workspotSystem->SendCommand( m_npcId, work::CMD_Play );
    }

    void MakeNPCLeave()
    {
        // 创建慢速退出命令
        red::UniquePtr< work::SlowExitCommandData > exitData =
            red::MakeUnique< work::SlowExitCommandData >();

        exitData->m_entryId = work::WorkEntryId::invalid; // 自动选择最佳退出
        exitData->m_immediate = false;
        exitData->m_doFastExitIfFails = true;

        m_workspotSystem->SendCommand(
            m_npcId,
            work::CMD_SlowExit,
            std::move( exitData )
        );
    }

    void ForceNPCExit()
    {
        // 创建快速退出命令
        red::UniquePtr< work::FastExitCommandData > exitData =
            red::MakeUnique< work::FastExitCommandData >();

        exitData->m_outDirection = Vector3( 1, 0, 0 ); // 向右逃跑
        exitData->m_exitType = work::FastExitType::Fear;
        exitData->m_immediate = true;

        m_workspotSystem->SendCommand(
            m_npcId,
            work::CMD_FastExit,
            std::move( exitData )
        );
    }

    void JumpToSpecificAnimation( work::WorkEntryId entryId )
    {
        red::UniquePtr< work::JumpToCommandData > jumpData =
            red::MakeUnique< work::JumpToCommandData >();

        jumpData->m_entryId = entryId;
        jumpData->m_immediate = true;

        m_workspotSystem->SendCommand(
            m_npcId,
            work::CMD_JumpToEntry,
            std::move( jumpData )
        );
    }
};
```

### 示例4：设置同步Workspot

```cpp
void SetupSynchronizedWorkspots(
    ent::Entity* masterNPC,
    ent::Entity* slaveNPC,
    const res::ResourcePath& masterWS,
    const res::ResourcePath& slaveWS
)
{
    work::WorkspotSystem* workspotSystem = GetGame()->GetWorkspotSystem();

    // 加载资源
    TResRef< work::WorkspotResource > masterRes = LoadResource( masterWS );
    TResRef< work::WorkspotResource > slaveRes = LoadResource( slaveWS );

    // 创建Master Workspot
    work::WorkspotParams masterParams( masterRes, work::OriginId::CreateUnique() );
    work::WorkspotSetupContext masterSetup;
    masterSetup.m_workspot = masterParams;
    masterSetup.m_commFun = red::MakeShared< work::NullWorkspotInstanceCommFunc >();

    workspotSystem->SetupWorkspot( THandle< ent::Entity >( masterNPC ), masterSetup );

    // 创建Slave Workspot
    work::WorkspotParams slaveParams( slaveRes, work::OriginId::CreateUnique() );
    work::WorkspotSetupContext slaveSetup;
    slaveSetup.m_workspot = slaveParams;
    slaveSetup.m_commFun = red::MakeShared< work::NullWorkspotInstanceCommFunc >();

    workspotSystem->SetupWorkspot( THandle< ent::Entity >( slaveNPC ), slaveSetup );

    // 绑定Master和Slave
    workspotSystem->AddPersistentLink(
        masterParams.m_locId,
        slaveParams.m_locId
    );

    // 同时开始播放
    workspotSystem->SendCommand( masterNPC->GetEntityID(), work::CMD_Play );
    workspotSystem->SendCommand( slaveNPC->GetEntityID(), work::CMD_Play );
}
```

### 示例5：RedScript集成

```swift
// RedScript代码示例

class WorkspotController extends ScriptableComponent
{
    private let workspotSystem: ref<WorkspotSystem>;
    private let npcEntity: wref<NPCPuppet>;

    private cb func OnInitialize()
    {
        this.workspotSystem = GameInstance.GetWorkspotSystem(this.GetGame());
    }

    public func StartWorkspot()
    {
        if !this.IsNPCInWorkspot()
        {
            // NPC不在Workspot中，可以开始
            // 实际的Workspot设置需要在C++层或通过Quest系统
        }
    }

    public func IsNPCInWorkspot() -> Bool
    {
        return this.workspotSystem.IsEntityInWorkspot(this.npcEntity.GetEntityID());
    }

    public func StopWorkspot()
    {
        if this.IsNPCInWorkspot()
        {
            // 发送停止命令
            // 需要通过C++接口或Quest节点
        }
    }

    public func OnWorkspotEnded()
    {
        // 回调处理
        LogChannel(n"Workspot", "NPC left workspot");
        this.OnNPCBecameFree();
    }

    private func OnNPCBecameFree()
    {
        // NPC完成Workspot后的逻辑
    }
}
```

---

## 常见问题与解决方案

### 问题1：NPC传送到Workspot而不是走过去

**症状：**
- NPC突然出现在Workspot位置
- 没有播放EntryAnim

**原因：**
- 缺少EntryAnim节点
- EntryAnim的movementType设置错误

**解决方案：**

```json
// 添加正确的EntryAnim节点
{
  "type": "EntryAnim",
  "animName": "walk_to_workspot",
  "movementType": "Walk",  // 确保不是Stand
  "orientationType": "Forward"
}
```

### 问题2：NPC动画一直重复同一个

**症状：**
- NPC只播放一个动画，没有变化

**原因：**
- 使用了单一AnimClip而不是RandomList
- RandomList的weights全部为0

**解决方案：**

```json
// 使用RandomList并设置正确的权重
{
  "type": "RandomList",
  "weights": [0.4, 0.3, 0.3],  // 确保权重>0
  "list": [
    {"type": "AnimClip", "animName": "idle_01"},
    {"type": "AnimClip", "animName": "idle_02"},
    {"type": "AnimClip", "animName": "idle_03"}
  ]
}
```

### 问题3：同步Workspot不同步

**症状：**
- Master和Slave动画不同步
- Slave位置不正确

**原因：**
- slotName不匹配
- syncOffset计算错误
- 动画时长不一致

**解决方案：**

```cpp
// 验证清单
1. Master和Slave的slotName完全一致
2. 测量并验证syncOffset
3. 确保动画时长相同（±0.1秒）

// 调试代码
#ifndef RED_CONFIGURATION_FINAL
void DebugSyncWorkspot( work::WorkspotInstance* master, work::WorkspotInstance* slave )
{
    Transform masterSlotTransform;
    if ( master->GetSyncWorkspotTransform( "sync_slot", masterSlotTransform ) )
    {
        RED_LOG( Debug, "Master slot position: (%f, %f, %f)",
            masterSlotTransform.GetPosition().X,
            masterSlotTransform.GetPosition().Y,
            masterSlotTransform.GetPosition().Z
        );
    }

    Float masterAnimTime = master->GetCurrentAnimTime();
    Float slaveAnimTime = slave->GetCurrentAnimTime();
    RED_LOG( Debug, "Anim time - Master: %f, Slave: %f, Diff: %f",
        masterAnimTime, slaveAnimTime, Abs(masterAnimTime - slaveAnimTime)
    );
}
#endif
```

### 问题4：物品在错误时间出现/消失

**症状：**
- NPC手里的物品突然出现
- 物品应该消失时还在

**原因：**
- ItemAction配置错误
- 缺少DespawnItem动作

**解决方案：**

```json
// 正确的物品管理流程
{
  "type": "Sequence",
  "list": [
    {
      "type": "AnimClipWithItem",
      "animName": "pick_up_item",
      "itemActions": [
        {
          "type": "WorkspotItemAction_SpawnProp",
          "propId": "item_id",
          "slotId": "hand_r",
          "onAnimEvent": "item_attach"  // 在特定事件时生成
        }
      ]
    },
    {
      "type": "RandomList",
      "list": [/* 使用物品的动作 */]
    },
    {
      "type": "AnimClipWithItem",
      "animName": "put_down_item",
      "itemActions": [
        {
          "type": "WorkspotItemAction_RemoveProp",
          "propId": "item_id",
          "onAnimEvent": "item_detach"  // 在特定事件时移除
        }
      ]
    }
  ]
}
```

### 问题5：Workspot卡住不动

**症状：**
- NPC进入Workspot后停止响应
- Update不再调用

**原因：**
- 迭代器返回无效数据
- 动画不存在导致系统等待
- 死循环（Sequence循环但没有退出条件）

**解决方案：**

```cpp
// 在编辑器中验证
void ValidateWorkspot( work::WorkspotTree* tree )
{
    // 1. 验证所有动画存在
    tree->ForEachAnimation( [this]( CName& animName, Bool isIdle ) {
        if ( !AnimationExists( animName ) )
        {
            RED_LOG_ERROR( Workspot, "Missing animation: %s", animName.AsChar() );
        }
    });

    // 2. 验证有退出节点
    Bool hasExit = false;
    tree->GetRootEntry()->ForEachNode(
        []( THandle<IEntry>& entry ) {},
        [&hasExit]( THandle<IEntry>& entry ) {
            if ( entry->GetFlags() & IEntry::ExitNode )
            {
                hasExit = true;
            }
        }
    );

    if ( !hasExit )
    {
        RED_LOG_WARNING( Workspot, "No exit node found - NPC may get stuck" );
    }
}
```

### 问题6：性能问题 - 太多活跃Workspot

**症状：**
- 帧率下降
- 内存占用高

**原因：**
- 太多NPC同时在Workspot中
- Workspot未正确清理

**解决方案：**

```cpp
// 限制活跃Workspot数量
class WorkspotManager
{
    constexpr Uint32 MAX_ACTIVE_WORKSPOTS = 50;

    void Update()
    {
        work::WorkspotSystem* ws = GetWorkspotSystem();

        if ( ws->NumOfActiveWorkspots() > MAX_ACTIVE_WORKSPOTS )
        {
            // 找到距离玩家最远的Workspot
            ent::EntityID playerID = GetPlayerID();
            Vector3 playerPos = GetPlayerPosition();

            red::DynArray< ent::EntityID > actorsInWorkspots;
            ws->CollectEntitiesInWorkspot( actorsInWorkspots );

            // 按距离排序
            std::sort( actorsInWorkspots.begin(), actorsInWorkspots.end(),
                [&playerPos, this]( ent::EntityID a, ent::EntityID b ) {
                    Float distA = Distance( GetEntityPos(a), playerPos );
                    Float distB = Distance( GetEntityPos(b), playerPos );
                    return distA > distB;
                }
            );

            // 移除最远的
            Uint32 toRemove = ws->NumOfActiveWorkspots() - MAX_ACTIVE_WORKSPOTS;
            for ( Uint32 i = 0; i < toRemove; ++i )
            {
                ws->SendCommand( actorsInWorkspots[i], work::CMD_Stop );
            }
        }
    }
};
```

---

## 性能优化清单

### ✅ Workspot设计优化

- [ ] 使用条件序列避免为每种情况创建单独的Workspot
- [ ] 限制RandomList的候选数量（建议≤10个）
- [ ] 避免过深的嵌套（建议≤5层）
- [ ] 合理设置PauseClip时间，避免过于频繁的动画切换
- [ ] 使用TagNode标记重要节点，便于跳转

### ✅ 运行时优化

- [ ] 预加载常用Workspot资源
- [ ] 使用资源池避免重复加载
- [ ] 限制同时活跃的Workspot实例数量
- [ ] 及时清理不再使用的Workspot实例
- [ ] 使用距离检查，远距离NPC不使用复杂Workspot

### ✅ 动画资源优化

- [ ] 共享AnimSet，避免重复资源
- [ ] 使用流式加载，按需加载动画
- [ ] 压缩动画数据
- [ ] 移除未使用的动画变体

### ✅ 同步优化

- [ ] 限制同时同步的Workspot数量（建议≤10对）
- [ ] 优化同步更新频率（不必每帧都同步位置）
- [ ] 使用LOD系统，远距离降低同步精度

---

## 调试技巧

### 1. 启用Workspot调试可视化

```cpp
#ifndef RED_CONFIGURATION_FINAL
// 控制台命令
void EnableWorkspotDebug()
{
    work::WorkspotSystem* ws = GetWorkspotSystem();
    ws->GetDebugger().SetMode( work::DebuggerMode::ShowAll );
}

// 显示内容：
// - 绿色框：Workspot位置
// - 蓝色箭头：EntryAnim路径
// - 红色箭头：ExitAnim路径
// - 黄色球：同步槽位置
// - 文字：当前动画名称和状态
#endif
```

### 2. 日志记录模板

```cpp
// 记录Workspot生命周期
class DebugWorkspotListener : public work::IWorkspotListener
{
    void OnWorkspotStarted( const ent::EntityID& entityId ) override
    {
        RED_LOG( WorkspotDebug, "[%llu] Workspot STARTED",
            entityId.GetValue() );
    }

    void OnWorkspotEnded( const ent::EntityID& entityId ) override
    {
        RED_LOG( WorkspotDebug, "[%llu] Workspot ENDED",
            entityId.GetValue() );
    }

    void OnAnimationChanged( const ent::EntityID& entityId, work::WorkEntryId entryId ) override
    {
        RED_LOG( WorkspotDebug, "[%llu] Animation changed to Entry[%d]",
            entityId.GetValue(), entryId.GetValue() );
    }
};
```

### 3. Workspot状态检查器

```cpp
#ifndef RED_CONFIGURATION_FINAL
void InspectWorkspotState( ent::EntityID entityId )
{
    work::WorkspotSystem* ws = GetWorkspotSystem();

    if ( !ws->IsActorInWorkspot( entityId ) )
    {
        RED_LOG( Debug, "Entity %llu is NOT in workspot", entityId.GetValue() );
        return;
    }

    work::WorkspotParams params;
    if ( ws->GetActorWorkspotParams( entityId, params ) )
    {
        RED_LOG( Debug, "Workspot Info:" );
        RED_LOG( Debug, "  - Tree: %s",
            params.m_tree ? params.m_tree->GetFriendlyName().AsChar() : "NULL" );
        RED_LOG( Debug, "  - OriginId: %llu", params.m_locId.GetValue() );
    }

    Uint32 flags = ws->GetRecordFlags( entityId );
    RED_LOG( Debug, "Current flags: 0x%X", flags );

    if ( flags & IEntry::Animation )
        RED_LOG( Debug, "  - Playing Animation" );
    if ( flags & IEntry::Synchronized )
        RED_LOG( Debug, "  - Synchronized" );
    if ( flags & IEntry::Reaction )
        RED_LOG( Debug, "  - In Reaction" );
}
#endif
```

### 4. 动画存在性验证工具

```cpp
// 编辑器工具：验证所有动画
void ValidateAllAnimations( work::WorkspotTree* tree )
{
    world::AnimationSystem* animSys = GetAnimationSystem();

    red::DynArray< CName > missingAnims;

    tree->ForEachAnimation( [&]( CName& animName, Bool isIdle ) {
        if ( animName == CName::NONE() ) return;

        // 检查动画是否存在于AnimSet中
        Bool exists = false;
        for ( auto& animset : tree->m_finalAnimsets )
        {
            if ( animSys->HasAnimation( animset.m_animations, animName ) )
            {
                exists = true;
                break;
            }
        }

        if ( !exists )
        {
            missingAnims.PushBack( animName );
        }
    });

    if ( !missingAnims.Empty() )
    {
        RED_LOG_ERROR( WorkspotValidation, "Missing animations:" );
        for ( auto& anim : missingAnims )
        {
            RED_LOG_ERROR( WorkspotValidation, "  - %s", anim.AsChar() );
        }
    }
    else
    {
        RED_LOG( WorkspotValidation, "All animations valid!" );
    }
}
```

### 5. 性能分析器

```cpp
#ifndef RED_CONFIGURATION_FINAL
class WorkspotPerformanceAnalyzer
{
    struct Stats
    {
        Uint32 totalWorkspots = 0;
        Uint32 activeWorkspots = 0;
        Uint32 syncedWorkspots = 0;
        Float averageUpdateTime = 0.0f;
        Float peakUpdateTime = 0.0f;
    };

    void AnalyzeFrame()
    {
        work::WorkspotSystem* ws = GetWorkspotSystem();

        Stats stats;
        stats.totalWorkspots = ws->NumOfActiveWorkspots();

        // 分析每个Workspot
        red::DynArray< ent::EntityID > actors;
        ws->CollectEntitiesInWorkspot( actors );

        for ( auto& actor : actors )
        {
            if ( ws->IsActorInWorkspot( actor ) )
            {
                stats.activeWorkspots++;
            }

            // 检查是否同步
            work::WorkspotClippingSpace clipSpace;
            if ( ws->GetWorkspotInstanceClippingSpace( actor, clipSpace ) )
            {
                // 有clipping space意味着可能在同步
                stats.syncedWorkspots++;
            }
        }

        RED_LOG( Performance, "Workspot Stats:" );
        RED_LOG( Performance, "  Total: %u", stats.totalWorkspots );
        RED_LOG( Performance, "  Active: %u", stats.activeWorkspots );
        RED_LOG( Performance, "  Synced: %u", stats.syncedWorkspots );
    }
};
#endif
```

---

## 总结

本快速参考指南提供了：

1. **快速开始模板** - 快速创建基础Workspot
2. **常用配置模板** - 常见场景的节点配置
3. **代码示例** - C++和RedScript集成示例
4. **问题解决方案** - 常见问题的诊断和修复
5. **性能优化清单** - 优化检查项
6. **调试技巧** - 实用的调试工具和方法

配合主文档和架构图集使用，可以快速上手Workspot系统开发。

---

*本文档基于Cyberpunk 2077源代码编写*
*包含实战经验和最佳实践*
*文档版本：1.0*
*生成日期：2026-02-13*
