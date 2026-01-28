# SyncEntry（同步Entry）使用完全指南

## 核心概念

**SyncEntry** 是WorkspotTree中用于**同步多个NPC动作**的机制，让多个Actor能够协同执行相同的动画序列。

---

## 一、Entry类型概览

### 1.1 普通Entry（支持同步）

```cpp
// workspotTreeItems.h: 268-303
class EntryAnim : public IEntry
{
    CName  m_animName;        // 动画名称
    CName  m_idleAnim;        // 底层idle动画
    CName  m_slotName;        // ⚠️ 同步槽名称（关键）
    Float  m_blendOutTime;
    Bool   m_isSynchronized;  // ⚠️ 是否启用同步
    Transform m_syncOffset;   // ⚠️ 同步偏移量

    // 其他参数...
};
```

**关键属性**：
- `m_isSynchronized` = true：启用同步
- `m_slotName`：同步槽的唯一标识符
- `m_syncOffset`：从Master到Slave的位置偏移

### 1.2 SyncMasterEntryAnim（强制Master）

```cpp
// workspotTreeItems.h: 305-322
class SyncMasterEntryAnim : public EntryAnim
{
    SyncMasterEntryAnim()
    {
        m_isSynchronized = true;  // 强制启用同步
    }

    // 只允许作为Master
    virtual Bool AllowSync(Bool asMaster) const override
    {
        return asMaster == true;
    }
};
```

**用途**：
- 显式标记为Master Entry
- 强制启用同步，不能关闭
- 只能作为Master，不能作为Slave

---

## 二、同步机制工作原理

### 2.1 Master-Slave架构

```
┌─────────────────────────────────────────┐
│ Master NPC (主控NPC)                     │
│ ├─ WorkspotInstance                     │
│ │  └─ EntryAnim (m_isSynchronized=true) │
│ │     ├─ m_slotName = "sync_slot_01"    │
│ │     └─ 播放动画：dance_01             │
│ └─ SyncEntry (在Synchronizer中)         │
│    ├─ m_masterOf = [Slave1, Slave2]     │
│    ├─ m_lastSyncAnimTime = 123.45       │
│    └─ m_currentSlotAnim = "sync_slot_01"│
└─────────────────────────────────────────┘
         │
         │ 同步信号
         ↓
┌─────────────────────────────────────────┐
│ Slave NPC 1                              │
│ ├─ WorkspotInstance                     │
│ │  └─ EntryAnim (m_slotName相同)        │
│ │     └─ 被强制同步到Master的时间点      │
│ └─ SyncEntry                             │
│    └─ m_slaveOf = Master                 │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Slave NPC 2                              │
│ ├─ WorkspotInstance                     │
│ └─ SyncEntry                             │
│    └─ m_slaveOf = Master                 │
└─────────────────────────────────────────┘
```

### 2.2 同步流程时序

```
时间轴：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

T0: Master和Slave都启动Workspot
    Master: OnWorkspotSetup() → 创建SyncEntry
    Slave:  OnWorkspotSetup() → 创建SyncEntry，标记m_slaveOf = Master

T1: Master收到CMD_Play
    Master: OnWorkspotPlay() → 开始执行Workspot序列

T2: Master进入SyncMasterEntryAnim
    ├─ UpdateRecord() 检测到 IEntry::Synchronized 标志
    ├─ 调用 WorkspotSynchronizer::PlaySyncAnim()
    │  ├─ 记录当前时间：m_lastSyncAnimTime = T2
    │  ├─ 记录槽名称：m_currentSlotAnim = "sync_slot_01"
    │  └─ 遍历所有Slave
    │     └─ 对每个Slave发送 CMD_JumpToEntry
    │        ├─ entryId = 查找Slave中相同slotName的Entry
    │        ├─ immediate = true (立即跳转)
    │        └─ forceTime = 当前时间 (强制同步时间点)
    └─ Master继续播放自己的动画

T2+δ: Slave收到JumpToEntry命令
    ├─ 立即跳转到slotName="sync_slot_01"的Entry
    ├─ 强制时间点对齐到Master的forceTime
    └─ 开始播放相同的动画（与Master同步）

T3: Master播放dance_01动画
    Slave也在播放dance_01动画
    → 两者完全同步！

T4: Master完成SyncEntry
    ├─ OnWorkspotFinished()
    └─ 清理Synchronizer中的链接关系
```

---

## 三、代码实现详解

### 3.1 创建同步链接

```cpp
// workspotSynchronizer.cpp: 106-149
void WorkspotSynchronizer::OnWorkspotSetup(
    const ent::EntityID& ownerId,
    const work::OriginId& originID,
    const SyncedWorkspotInfo* syncInfo
)
{
    // 方式1：通过PersistentLink（场景中预定义的链接）
    auto perIter = m_persistentLinks.Find( originID );
    if( perIter != m_persistentLinks.End() )
    {
        PersistentLink& linkInfo = perIter.Value();
        linkInfo.m_currentOccupant = ownerId;

        // 建立Master-Slave关系
        for( OriginId master : linkInfo.m_masterOf )
        {
            SyncEntry& entry = m_lookupMap[ownerId];
            entry.m_masterOf.PushBack( slaveId );

            SyncEntry& slaveEntry = m_lookupMap[slaveId];
            slaveEntry.m_slaveOf = ownerId;
        }
    }

    // 方式2：通过运行时传入的syncInfo
    if( syncInfo && syncInfo->m_masterId )
    {
        const ent::EntityID& masterID = *syncInfo->m_masterId;

        SyncEntry& entry = m_lookupMap[ownerId];
        entry.m_slaveOf = masterID;  // 标记Slave

        SyncEntry& masterEntry = m_lookupMap[masterID];
        masterEntry.m_masterOf.PushBack( ownerId );  // Master记录Slave
    }
}
```

### 3.2 同步动画播放

```cpp
// workspotSynchronizer.cpp: 222-249
Bool WorkspotSynchronizer::PlaySyncAnim(
    const ent::EntityID& actorId,    // Master的EntityID
    Float timeToSync,                // 当前动画的剩余时间
    CName slotName                   // 同步槽名称
)
{
    auto iter = m_lookupMap.Find( actorId );
    if( iter != m_lookupMap.End() )
    {
        SyncEntry& entry = iter.Value();

        // 记录Master的状态
        entry.m_lastSyncAnimTime = m_system.GetCurrentTime();
        entry.m_currentSlotAnim = slotName;

        // 同步所有Slave
        for( ent::EntityID& syncedWs : entry.m_masterOf )
        {
            // 在Slave的WorkspotTree中查找相同slotName的Entry
            work::WorkEntryId entryId =
                m_system.GetSyncEntryIdForSlotName( syncedWs, false, slotName );

            if( entryId )
            {
                // 创建跳转命令
                auto data = CreateUniquePtr<JumpToCommandData>();
                data->m_entryId = entryId;      // 目标Entry
                data->m_immediate = true;       // 立即跳转
                data->m_forceTime = timeToSync; // 强制时间对齐

                // 发送命令到Slave
                m_system.SendCommandImmediate( syncedWs, CMD_JumpToEntry, data );
            }
        }

        return true;
    }

    return false;
}
```

### 3.3 Slave加入同步（延迟启动）

```cpp
// workspotSynchronizer.cpp: 152-180
void WorkspotSynchronizer::OnWorkspotPlay( const ent::EntityID& ownerId )
{
    auto iter = m_lookupMap.Find( ownerId );
    if( iter != m_lookupMap.End() )
    {
        SyncEntry& entry = iter.Value();

        // 如果这个NPC是Slave
        auto masterIter = m_lookupMap.Find( entry.m_slaveOf );
        if( masterIter != m_lookupMap.End() )
        {
            SyncEntry& masterEntry = masterIter.Value();

            // 如果Master已经在播放同步动画
            if( masterEntry.m_lastSyncAnimTime > 0.f )
            {
                // 查找Slave中对应的Entry
                work::WorkEntryId entryId =
                    m_system.GetSyncEntryIdForSlotName(
                        ownerId,
                        true,  // 作为Slave
                        masterEntry.m_currentSlotAnim
                    );

                if( entryId )
                {
                    // 计算时间偏移
                    Float forceTime =
                        m_system.GetCurrentTime() - masterEntry.m_lastSyncAnimTime;

                    // 跳转到同步点
                    auto data = CreateUniquePtr<JumpToCommandData>();
                    data->m_entryId = entryId;
                    data->m_immediate = false;  // 非立即（允许过渡）
                    data->m_forceTime = forceTime;

                    m_system.SendCommandImmediate( ownerId, CMD_JumpToEntry, data );
                }
            }
        }
    }
}
```

---

## 四、实际使用场景

### 场景1：两人共舞

**需求**：两个NPC跳双人舞，动作完全同步

**WorkspotTree配置**：

**Master的WorkspotTree**：
```
Sequence
├─ EntryAnim: walk_to_dance_position
└─ SyncMasterEntryAnim  ← ⚠️ 使用SyncMasterEntryAnim
   ├─ m_animName = "dance_couple_01"
   ├─ m_slotName = "couple_dance_sync"  ← 关键：同步槽名称
   ├─ m_isSynchronized = true (自动设置)
   └─ m_syncOffset = (0, 0, 0)
```

**Slave的WorkspotTree**：
```
Sequence
├─ EntryAnim: walk_to_dance_position
└─ EntryAnim  ← 普通EntryAnim，但启用同步
   ├─ m_animName = "dance_couple_01"  ← 同名动画
   ├─ m_slotName = "couple_dance_sync"  ← ⚠️ 相同的槽名称
   ├─ m_isSynchronized = true  ← 启用同步
   └─ m_syncOffset = (1.5, 0, 0)  ← 相对Master的位置偏移
```

**代码启动**：
```cpp
// 启动Master
WorkspotSetupContext masterSetup;
masterSetup.m_workspot.m_tree = LoadResource("dance_master.workspot");
WorkspotSystem::SetupWorkspot( masterNPC, masterSetup, nullptr );
WorkspotSystem::SendCommand( masterNPC, CMD_Play );

// 启动Slave（指定Master）
SyncedWorkspotInfo syncInfo;
syncInfo.m_masterId = &masterNPC;  // ⚠️ 关键：指定Master

WorkspotSetupContext slaveSetup;
slaveSetup.m_workspot.m_tree = LoadResource("dance_slave.workspot");
WorkspotSystem::SetupWorkspot( slaveNPC, slaveSetup, &syncInfo );
WorkspotSystem::SendCommand( slaveNPC, CMD_Play );
```

**执行流程**：
```
Master: walk_to_dance_position → 到达位置
Master: 进入SyncMasterEntryAnim "couple_dance_sync"
    ↓
    PlaySyncAnim() 触发
    ↓
Slave: 收到CMD_JumpToEntry
    ↓
    立即跳转到 m_slotName="couple_dance_sync" 的Entry
    ↓
Master和Slave: 同时播放 "dance_couple_01" 动画
    ↓
完美同步！
```

---

### 场景2：多人合唱

**需求**：3个NPC围成一圈合唱，动作同步

**Master Workspot**：
```
Sequence
└─ SyncMasterEntryAnim
   ├─ m_slotName = "choir_sync_01"
   └─ m_animName = "sing_together"
```

**Slave1 Workspot**：
```
Sequence
└─ EntryAnim
   ├─ m_slotName = "choir_sync_01"  ← 相同槽名
   ├─ m_isSynchronized = true
   ├─ m_syncOffset = (2.0, 0, 0)    ← 位置1
   └─ m_animName = "sing_together"
```

**Slave2 Workspot**：
```
Sequence
└─ EntryAnim
   ├─ m_slotName = "choir_sync_01"  ← 相同槽名
   ├─ m_isSynchronized = true
   ├─ m_syncOffset = (1.0, 1.7, 0)  ← 位置2
   └─ m_animName = "sing_together"
```

**启动代码**：
```cpp
// 1. 启动Master
SetupWorkspot( masterNPC, masterWorkspot, nullptr );
SendCommand( masterNPC, CMD_Play );

// 2. 启动Slave1
SyncedWorkspotInfo sync1;
sync1.m_masterId = &masterNPC;
SetupWorkspot( slave1NPC, slave1Workspot, &sync1 );
SendCommand( slave1NPC, CMD_Play );

// 3. 启动Slave2
SyncedWorkspotInfo sync2;
sync2.m_masterId = &masterNPC;
SetupWorkspot( slave2NPC, slave2Workspot, &sync2 );
SendCommand( slave2NPC, CMD_Play );
```

**结果**：3个NPC完全同步地播放"sing_together"动画

---

### 场景3：延迟加入的同步

**需求**：Slave晚于Master启动，但仍要同步

**时间轴**：
```
T0: Master启动，进入SyncEntry，开始播放动画
T5: Slave启动（晚了5秒）
```

**系统处理**：
```cpp
// T5: Slave调用OnWorkspotPlay()
void OnWorkspotPlay( slaveNPC )
{
    // 检测到Slave的Master已经在播放
    SyncEntry& masterEntry = m_lookupMap[masterNPC];

    if( masterEntry.m_lastSyncAnimTime > 0 )  // Master已开始
    {
        // 计算时间差
        Float elapsed = CurrentTime - masterEntry.m_lastSyncAnimTime;
        // elapsed = 5秒

        // 跳转到Master的当前位置
        JumpToEntry( slaveNPC, syncEntryId, forceTime = 5.0 );
        // → Slave从动画的第5秒开始播放
    }
}
```

**结果**：Slave"追上"Master的进度，实现同步

---

## 五、关键参数说明

### 5.1 m_slotName（同步槽名称）

```cpp
CName m_slotName;  // 例如："dance_sync_01", "sit_sync", "fight_combo_01"
```

**作用**：
- Master和Slave通过**相同的slotName**建立同步关系
- 系统通过slotName查找Slave中对应的Entry
- **必须唯一**且**Master/Slave一致**

**命名建议**：
```
sync_<场景>_<动作类型>_<编号>

示例：
- sync_dance_couple_01      # 双人舞
- sync_fight_combo_punch    # 战斗连击
- sync_ritual_group_01      # 群体仪式
```

### 5.2 m_syncOffset（同步偏移）

```cpp
Transform m_syncOffset;  // 从Master到Slave的相对位置
```

**用途**：
- 定义Slave相对于Master的空间位置
- 保证多个NPC在正确的位置上执行同步动作

**示例**：
```cpp
// 双人舞：两人相距1.5米
masterOffset = Transform::IDENTITY;
slaveOffset  = Transform( Vector3(1.5, 0, 0) );

// 圆圈合唱：3人围成120度圆圈
slave1Offset = Transform( Vector3(2.0, 0, 0), Rotation(0, 0, 0) );
slave2Offset = Transform( Vector3(1.0, 1.7, 0), Rotation(0, 0, 120) );
slave3Offset = Transform( Vector3(-1.0, 1.7, 0), Rotation(0, 0, 240) );
```

### 5.3 m_isSynchronized（同步标志）

```cpp
Bool m_isSynchronized;  // true = 启用同步
```

**影响**：
- true → Entry标记为 `IEntry::Synchronized`
- UpdateRecord() 检测到此标志 → 调用 `PlaySyncAnim()`
- 触发同步机制

**何时使用**：
- ✅ 普通EntryAnim + m_isSynchronized = true：灵活的同步Entry
- ✅ SyncMasterEntryAnim：显式Master，强制同步
- ✗ m_isSynchronized = false：普通Entry，不参与同步

---

## 六、高级用法

### 6.1 PersistentLink（场景预定义链接）

**用途**：在场景中预先定义Workspot之间的同步关系，无需运行时传参

**定义**：
```cpp
// 场景初始化时
WorkspotSynchronizer::AddPersistentLink(
    master = OriginId("dance_master_loc"),  // Master位置
    slave  = OriginId("dance_slave_loc")    // Slave位置
);
```

**优势**：
- NPC只需使用对应的Workspot位置
- 系统自动建立Master-Slave关系
- 适合固定场景（如酒吧舞池、仪式场地）

**示例**：
```cpp
// 酒吧舞池，预定义6个舞蹈位置的同步关系
AddPersistentLink( "dance_pos_01", "dance_pos_02" );  // 第1对
AddPersistentLink( "dance_pos_03", "dance_pos_04" );  // 第2对
AddPersistentLink( "dance_pos_05", "dance_pos_06" );  // 第3对
```

NPC使用时：
```cpp
// NPC1使用dance_pos_01 → 自动成为Master
// NPC2使用dance_pos_02 → 自动成为Slave，同步到NPC1
```

### 6.2 SendEventToConnectedSpots（事件广播）

**用途**：向所有同步关联的NPC发送事件

```cpp
// workspotSynchronizer.cpp: 82-95
Bool SendEventToConnectedSpots(
    const ent::EntityID entityId,  // 发送者
    CName evtName                  // 事件名称
)
{
    // 查找entityId的同步链接
    auto iter = m_lookupMap.Find( entityId );
    if( iter != m_lookupMap.End() )
    {
        // 创建事件
        THandle<ConnectedWorkspotNotificationEvent> event =
            CreateHandle<ConnectedWorkspotNotificationEvent>( evtName );

        // 递归发送到所有关联的NPC
        // （包括Master、所有Slave，以及它们的关联）
        ProcessNotificationInternal( entityId, evtName, ... );

        return true;
    }

    return false;
}
```

**使用场景**：
```cpp
// Master NPC触发事件
WorkspotSynchronizer::SendEventToConnectedSpots(
    masterNPC,
    CName("DanceFinished")
);

// 所有Slave NPC都会收到 "DanceFinished" 事件
// → 可以在RedScript中响应
class DanceSlaveNPC extends ScriptedPuppet
{
    @Event("DanceFinished")
    protected cb func OnDanceFinished(evt: ref<ConnectedWorkspotNotificationEvent>)
    {
        // Slave收到Master的事件
        // 可以执行后续逻辑（如：鼓掌、欢呼等）
    }
}
```

---

## 七、常见问题与调试

### 问题1：Slave没有同步

**症状**：Master播放动画，Slave没反应

**检查清单**：
- [ ] Slave的m_slotName与Master相同？
- [ ] Slave的m_isSynchronized = true？
- [ ] Slave的WorkspotTree中有对应slotName的Entry？
- [ ] Slave启动时传入了正确的SyncedWorkspotInfo？

**调试**：
```cpp
// 添加日志
void PlaySyncAnim(...)
{
    for( ent::EntityID& syncedWs : entry.m_masterOf )
    {
        work::WorkEntryId entryId = GetSyncEntryIdForSlotName( syncedWs, false, slotName );

        if( entryId )
        {
            RED_LOG_DEBUG("Syncing slave [%s] to slot [%s]",
                syncedWs.ToDebugString(), slotName.AsChar());
        }
        else
        {
            RED_LOG_ERROR("Slave [%s] has no entry with slotName [%s]!",
                syncedWs.ToDebugString(), slotName.AsChar());
            // ↑ 这里会告诉你Slave缺少对应的Entry
        }
    }
}
```

### 问题2：时间不同步

**症状**：Master和Slave都在播放，但时间不对

**原因**：
- Slave的动画长度与Master不同
- forceTime设置不正确

**解决**：
```cpp
// 确保Master和Slave使用相同的动画
Master: m_animName = "dance_01"  // 时长: 5秒
Slave:  m_animName = "dance_01"  // 时长: 5秒 ← 必须相同

// 或者使用变体动画（但时长必须一致）
Master: m_animName = "dance_male_01"    // 5秒
Slave:  m_animName = "dance_female_01"  // 5秒 ← 时长相同即可
```

### 问题3：Slave位置不对

**症状**：Slave在错误的位置播放动画

**检查**：
```cpp
// 检查m_syncOffset
EntryAnim (Slave)
{
    m_syncOffset = Transform( Vector3(1.5, 0, 0) );
    // ↑ 这是相对于Master的偏移
    // 如果Master在 (100, 200, 0)
    // Slave应该在 (101.5, 200, 0)
}
```

**调试**：
```cpp
// 在UpdateRecord中添加日志
RED_LOG_DEBUG("Slave [%s] playing at position: %.2f, %.2f, %.2f",
    GetOwnerId().ToDebugString(),
    GetPosition().X, GetPosition().Y, GetPosition().Z);
```

---

## 八、最佳实践

### 8.1 命名规范

```
槽名称：sync_<用途>_<编号>
- sync_dance_couple_01
- sync_fight_combo_punch
- sync_ritual_circle_01

动画名称：保持Master和Slave一致（或使用对应变体）
- dance_couple_male_01   (Master)
- dance_couple_female_01 (Slave)
```

### 8.2 WorkspotTree结构建议

**Master Workspot**：
```
Sequence
├─ EntryAnim: approach
├─ SyncMasterEntryAnim: main_sync_action  ← 同步点
└─ ExitAnim: leave
```

**Slave Workspot**：
```
Sequence
├─ EntryAnim: approach
├─ EntryAnim (m_isSynchronized=true): main_sync_action  ← 同步点
└─ ExitAnim: leave
```

### 8.3 错误处理

```cpp
// 在设置Workspot时验证
Bool ValidateSyncSetup( WorkspotTree* masterTree, WorkspotTree* slaveTree, CName slotName )
{
    // 检查Master是否有对应的SyncEntry
    if( !masterTree->HasSyncEntry( slotName, true ) )
    {
        RED_LOG_ERROR("Master workspot missing sync entry: %s", slotName.AsChar());
        return false;
    }

    // 检查Slave是否有对应的Entry
    if( !slaveTree->HasSyncEntry( slotName, false ) )
    {
        RED_LOG_ERROR("Slave workspot missing sync entry: %s", slotName.AsChar());
        return false;
    }

    return true;
}
```

---

## 九、总结

### SyncEntry核心要点

1. **同步通过slotName建立**：Master和Slave必须有相同的slotName
2. **Master控制节奏**：Slave被动跟随Master的时间
3. **两种启动方式**：
   - PersistentLink（场景预定义）
   - SyncedWorkspotInfo（运行时指定）
4. **延迟加入支持**：Slave晚启动会自动追上进度
5. **事件广播**：Master可以向所有Slave发送事件

### 使用场景

- ✅ 双人舞蹈
- ✅ 群体仪式
- ✅ 战斗连击（多人配合）
- ✅ 合唱/演奏
- ✅ 任何需要多NPC协同的场景

### 关键API

```cpp
// 设置同步关系
WorkspotSynchronizer::AddPersistentLink( master, slave );

// 启动Workspot（指定Master）
SyncedWorkspotInfo syncInfo;
syncInfo.m_masterId = &masterNPC;
SetupWorkspot( slaveNPC, workspot, &syncInfo );

// 发送事件到同步组
SendEventToConnectedSpots( masterNPC, "EventName" );
```

现在您可以在Workspot中实现完美的多NPC同步动作了！
