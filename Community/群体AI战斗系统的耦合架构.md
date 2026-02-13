# 群体AI战斗系统的耦合架构分析

## 一、核心问题解答

### 1.1 CommunitySystem在群体AI混战中耦合了哪些系统？

答案：**至少耦合了10+个核心系统**，形成复杂的协作网络。

```
CommunitySystem在群体AI战斗中的系统耦合架构：

┌────────────────────────────────────────────────────────────┐
│                    CommunitySystem                         │
│                 (场景级NPC管理系统)                        │
└────────────────────────────────────────────────────────────┘
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
       ▼                  ▼                  ▼
  生成与生命周期      AI行为与感知      战斗与战术协调
       │                  │                  │
       │                  │                  │
┌──────┴──────────┐ ┌─────┴─────────┐ ┌─────┴──────────┐
│ 1. PopulationSystem  │ 4. AI系统层    │ 7. SquadManager   │
│ 2. EntityStubSystem  │ 5. SenseManager│ 8. AttitudeManager│
│ 3. PersistencySystem │ 6. TargetTracker│ 9. InfluenceMap   │
│ 4. WorkspotManager   │               │10. CombatSpace     │
└──────────────────┘ └───────────────┘ └────────────────┘
```

## 二、Squad系统是独立系统吗？

### 2.1 答案：**是！SquadManager是完全独立的IGameSystem**

**文件位置**: `common/game/include/squadManager.h`

```cpp
class SquadManager final : public ISquadManager
{
    RTTI_DECLARE_TYPE(SquadManager);
    RED_BASE_CLASS(ISquadManager);

    // ISquadManager继承自IGameSystem
    // SquadManager是完全独立的游戏系统
};

// 基类定义
class ISquadManager : public IGameSystem
{
    // 独立的系统接口
    virtual void WorldAttached(world::RuntimeScene& scene) override;
    virtual void WorldPendingDetach(world::RuntimeScene& scene) override;
    virtual void RegisterUpdateFunctions(UpdatableSystemRegistrar&) override;
};
```

### 2.2 Squad System的架构层次

```
SquadManager (AI::SquadManager)
├── 管理所有Squad实例
│   ├── HashMap<CName, SquadEntry> m_squads
│   └── HashMap<CName, SharedPtr<CombatSquad>> m_combatSquads
│
├── 生命周期管理
│   ├── WorldAttached() - 系统初始化
│   ├── Update(Float deltaTime) - 每帧更新
│   └── WorldPendingDetach() - 系统清理
│
├── Squad类型工厂
│   ├── SquadBase - 基础小队（Community, Global, Security, Attitude）
│   ├── CombatSquad - 战斗小队（Combat）
│   └── FollowerSquad - 跟随小队（Follower）
│
└── 与其他系统的接口
    ├── IInfluenceMapSystem - 影响力地图
    ├── INavigationSystem - 导航系统
    └── IPersistencySystem - 存档系统
```

## 三、如何定义AI群组？

### 3.1 三层群组定义架构

2077使用**三层架构**定义AI群组，每层负责不同的职责：

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: AttitudeGroup（派系层）- 全局敌友关系            │
│  ├── 定义位置：TweakDB Attitude Records                   │
│  ├── 作用范围：全游戏世界                                 │
│  ├── 管理系统：AttitudeManager (IGameSystem)              │
│  └── 示例：Arasaka, NCPD, Maelstrom, Player                │
└─────────────────────────────────────────────────────────────┘
                          ↓ (派系关系矩阵)
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: Squad（小队层）- 战术协同单位                    │
│  ├── 定义位置：.community文件 + TweakDB Character_Record   │
│  ├── 作用范围：场景/区域                                  │
│  ├── 管理系统：SquadManager (IGameSystem)                 │
│  ├── 类型：Community/Global/Security/Combat/Follower       │
│  └── 示例：checkpoint_alpha_squad, base_defense_squad      │
└─────────────────────────────────────────────────────────────┘
                          ↓ (战术命令与信息共享)
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: Community（场景层）- 生命周期管理                │
│  ├── 定义位置：.community文件                             │
│  ├── 作用范围：特定场景区域                               │
│  ├── 管理系统：CommunitySystem (IGameSystem)              │
│  └── 示例：militech_checkpoint.community                   │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 实际定义方式详解

#### 方式1：通过Community配置Squad

**文件**: `.community`文件

```json
{
    "entryName": "guard_01",
    "characterRecordId": "Character.militech_soldier",
    "phases": [
        {
            "phaseName": "default",
            "timePeriods": [
                {
                    "hour": "Day",
                    "quantity": 5
                }
            ]
        }
    ],
    "initializers": [
        {
            "type": "SquadInitializer",
            "entries": [
                {
                    "type": "Community",  // Squad类型
                    "value": "base_defense_squad"  // Squad名称
                }
            ]
        }
    ]
}
```

**运行时效果**：
```cpp
// common/gamePopulation/src/communityEntryInitializer.cpp:64

void SquadInitializer::InitializeStub(game::EntityStub* stub) const {
    // 1. 获取NPC的SquadMemberComponent
    const THandle<game::SquadMemberComponentPS>& squadPS =
        stub->FindComponentPS<game::SquadMemberComponentPS>(...);

    // 2. 将NPC加入指定的Squad
    squadPS->HandleCommunityInitializerEntries(m_entries);
    //   → SquadManager.FindOrCreateSquad("base_defense_squad")
    //   → Squad.AddMember(npc)
    //   → NPC现在属于"base_defense_squad"小队
}
```

#### 方式2：通过TweakDB Character_Record配置

**文件位置**: `common/gameTweakDB/include/records/tweakDBCharacter.h`

```cpp
class Character_Record : public SpawnableObject_Record {
    // Squad相关字段
    TweakDB::VCName globalSquad;      // 全局小队名
    TweakDB::VCName communitySquad;   // 社区小队名
    TweakDB::VCName securitySquad;    // 安全小队名

    // AttitudeGroup字段
    TweakDB::VCName baseAttitudeGroup;  // 派系组
};
```

**TweakDB数据示例**:
```
Character.militech_soldier {
    baseAttitudeGroup: "militech"              // 属于Militech派系
    communitySquad: "militech_patrol_squad"    // 自动加入社区小队
    globalSquad: "militech_global"             // 自动加入全局小队
}
```

#### 方式3：通过SquadMemberComponent动态加入

**文件位置**: `common/game/src/squadMemberComponent.cpp`

```cpp
void SquadMemberComponent::OnInitialize(const ent::ComponentInitializeContext& info) {
    // 1. 获取系统引用
    m_squadManager = info.entityOwner->GetGameSystem<AI::ISquadManager>();
    m_attitudeManager = info.entityOwner->GetGameSystem<game::IAttitudeManager>();

    // 2. 监听AttitudeGroup变化
    m_attitudeManager->RegisterAgentAttitudeGroupChange(
        entityId,
        *this,
        [this](const CName& fromGrp, const CName& toGrp) {
            // AttitudeGroup变化时，自动加入对应的Attitude Squad
            LoadSquad(m_squadManager->CreateDefinition(toGrp, AI::SquadType::Attitude));
        }
    );
}
```

## 四、10+个耦合系统的详细说明

### 4.1 生成与生命周期层

#### 1️⃣ PopulationSystem（人口系统）

**作用**: 负责将EntityStub实例化为实际的Entity
```cpp
// CommunitySystem → PopulationSystem
IPopulationSystem& populationSystem = GetSystem<IPopulationSystem>();
populationSystem.SpawnEntity(stub);
```

#### 2️⃣ EntityStubSystem（实体存根系统）

**作用**: 轻量级Entity占位符，用于流式加载优化
```cpp
// CommunityEntrySpawner创建Stub
EntityStubCreationTokenHandle tokenHandle =
    stubSystem.RequestStubCreation(
        characterRecordId,
        worldPosition,
        spawnInView
    );
```

#### 3️⃣ PersistencySystem（存档系统）

**作用**: 保存和加载NPC状态
```cpp
// 存档Community状态
struct CommunityEntrySavedState {
    DynArray<EntityID> m_entityIds;           // 存活的NPC
    Uint16 m_totalDeadEntitiesCount;          // 死亡计数
    CName m_currentPhaseName;                 // 当前Phase
};
```

#### 4️⃣ WorkspotManager（工作点管理系统）

**作用**: 管理NPC占用的Workspot
```cpp
// 尝试使用Workspot
AI::SpotUsageToken spotToken =
    workspotMgr.TryUseSpot(spotNodeId);
```

---

### 4.2 AI行为与感知层

#### 5️⃣ SenseManager（感知系统）

**作用**: 管理NPC的视觉、听觉等感知
```cpp
// 文件位置: common/game/src/senseManager.cpp

class SenseManager : public ISenseManager {
    // NPC感知到敌人
    void OnEnemyDetected(EntityID sensor, EntityID target);

    // 共享感知信息给Squad
    void BroadcastToSquad(EntityID sensor, SenseEvent event);
};
```

**与Squad的集成**:
```
NPC_A感知到敌人
    → SenseManager.OnEnemyDetected()
    → Squad.BroadcastEvent(EnemySpotted, enemyId)
    → 所有Squad成员收到通知
    → NPC_B/C/D进入警戒状态
```

#### 6️⃣ TargetTrackerManager（目标追踪系统）

**作用**: 追踪敌人位置和状态
```cpp
// 文件位置: common/game/src/targetTrackerComponent.cpp

// CombatSquad使用TargetTracker共享目标信息
void Enemy::AddTracker(const squads::MemberId& tracker,
                       const TrackerUpdateCall& updateCall) {
    m_trackers.Insert(tracker, EnemyTracker(tracker, updateCall, true));
}

// 更新敌人位置
void Enemy::UpdatePosition() {
    for (auto& tracker : m_trackers) {
        tracker.UpdatePos();  // 从TargetTrackerComponent获取位置
    }
}
```

---

### 4.3 战斗与战术协调层

#### 7️⃣ SquadManager（小队管理系统）⭐核心

**文件位置**: `common/game/src/squadManager.cpp`

**作用**: 管理所有Squad，协调战术

```cpp
class SquadManager : public ISquadManager {
    // Squad集合
    HashMap<CName, SquadEntry> m_squads;
    HashMap<CName, SharedPtr<CombatSquad>> m_combatSquads;

    // 依赖的其他系统
    game::influence::ISystem* m_influenceMapSystem;
    game::IPersistencySystem* m_persistencySystem;
    AI::INavigationSystem* m_navigationSystem;

    // 每帧更新所有Squad
    void Update(Float deltaTime) {
        for (auto& [name, entry] : m_squads) {
            entry.m_squad->Update(deltaTime);
        }
    }
};
```

**SquadManager初始化时获取的系统**:
```cpp
void SquadManager::WorldAttached(world::RuntimeScene& scene) {
    // 获取影响力地图系统
    m_influenceMapSystem = game->GetSystemPtr<game::influence::ISystem>().Get();

    // 获取持久化系统
    m_persistencySystem = game->GetSystemPtr<game::IPersistencySystem>().Get();

    // 获取导航系统
    m_navigationSystem = game->GetSystemPtr<AI::INavigationSystem>().Get();
}
```

#### 8️⃣ AttitudeManager（态度管理系统）⭐核心

**文件位置**: `common/gameCore/include/attitudeManagerInterface.h`

**作用**: 管理派系关系矩阵

```cpp
class IAttitudeManager : public IGameSystem {
    // 获取派系间态度
    virtual Bool GetGlobalAttitude(CName srcGroup, CName dstGroup,
                                   EAIAttitude& attitude) const = 0;

    // 设置临时态度覆盖
    virtual Bool SetTemporaryGlobalAttitude(CName srcGroup, CName dstGroup,
                                            EAIAttitude attitude) = 0;

    // 获取实体的AttitudeGroup
    virtual CName GetAttitudeGroup(const ent::EntityID& entityID) const = 0;

    // 派系变化通知
    virtual void NotifyGroupChange(const ent::EntityID& aid,
                                   const CName& fromGroup,
                                   const CName& toGroup) = 0;
};
```

**AttitudeGroup定义示例**:
```cpp
// TweakDB中定义的派系关系矩阵

AttitudeGroup.arasaka ← → AttitudeGroup.militech = Hostile  // 敌对
AttitudeGroup.arasaka ← → AttitudeGroup.ncpd = Friendly     // 友好
AttitudeGroup.player ← → AttitudeGroup.maelstrom = Hostile  // 敌对
```

**与Squad的集成**:
```cpp
// SquadMemberComponent监听AttitudeGroup变化
void AddAttitudeHandler(const ent::EntityID& eid) {
    m_attitudeManager->RegisterAgentAttitudeGroupChange(
        eid,
        *this,
        [this](const CName& fromGrp, const CName& toGrp) {
            // AttitudeGroup变化时，自动切换Attitude Squad
            LoadSquad(m_squadManager->CreateDefinition(
                toGrp,
                AI::SquadType::Attitude
            ));
        }
    );
}
```

#### 9️⃣ InfluenceMapSystem（影响力地图系统）

**文件位置**: `common/gameCore/include/influenceMapSystemInterface.h`

**作用**: 为CombatSquad提供战术地图

```cpp
// CombatSquad使用InfluenceMap管理战术层
class CombatSquad : public SquadBase {
    // 战术层：标记哪些区域被某个战术占用
    using TacticalLayer = game::influence::Layer<TacticCell, CanBeCachedOut>;

    // 可达性层：计算从敌人位置到我方成员的可达性
    struct Reachability {
        game::influence::Layer<RCell, CanBeRemoved>* m_layer;
        game::influence::GridMask m_horizont;
        Float m_distanceLimit = 30.f;
    };

    // 注册战术
    void RegisterTactic(const CName& tacticName,
                       const DynArray<CombatSector::Type>& sectors,
                       const THandle<CombatAlley>& referenceAlley,
                       Float timeout);
};
```

**战术层工作原理**:
```
场景：小队成员执行"侧翼包抄"战术

1. CombatSquad注册战术：
   RegisterTactic("FlankLeft", [LeftFlank], offensiveCombatAlley, 10.0f)

2. InfluenceMap创建TacticalLayer：
   在影响力地图上标记"左侧区域"为"被FlankLeft战术占用"

3. 其他Squad成员查询：
   CheckTacticInEffect(position, memberId)
   → 如果position在FlankLeft占用的区域内
   → 返回true，避免重复执行相同战术

4. 超时清理：
   10秒后，如果没有更新，TacticalLayer自动清除标记
```

#### 🔟 CombatSpaceManager（战斗空间管理系统）

**作用**: 管理掩体、射击位置等战斗空间资源

```cpp
// CombatSquad查询可用掩体
ICombatSpaceManager& combatSpace = GetSystem<ICombatSpaceManager>();
Vector3 coverPosition = combatSpace.FindBestCover(squadMemberPos, enemyPos);
```

---

### 4.4 其他耦合系统

#### 1️⃣1️⃣ NavigationSystem（导航系统）

**作用**: 路径规划，巡逻路线
```cpp
AI::INavigationSystem* navSys = m_squadManager->GetNavigationSystem();
navSys.FindPath(from, to);
```

#### 1️⃣2️⃣ EntitySpawnerEventsBroadcaster（实体生成事件广播）

**作用**: 监听NPC生成/死亡事件
```cpp
void SquadManager::WorldAttached(world::RuntimeScene& scene) {
    // 监听NPC死亡事件
    m_broadcastCallbackId = GetGameInstance()
        ->GetSystem<game::IEntitySpawnerEventsBroadcaster>()
        .RegisterCallback([this](game::EntitySpawnerEventType eventType,
                                const ent::EntityID& spawnedEntityId,
                                const game::CommunityID& spawnerId,
                                game::EntityStub* stub) {
            if (eventType != game::EntitySpawnerEventType::Spawn) {
                // NPC死亡，从所有CombatSquad中移除
                for (auto sq : m_combatSquads) {
                    sq.Value()->DropEnemy(AI::Enemy::ConvertId(spawnedEntityId));
                }
            }
        }, "SpawnerCallback_SquadManager");
}
```

#### 1️⃣3️⃣ StatPoolsSystem（状态池系统）

**作用**: 管理NPC的生命值、体力等
```cpp
// SquadMemberComponent检查NPC是否死亡
if (statPoolSystem->HasStatPoolValueReachedMin(
        entityId,
        game::data::StatPoolType::Health)) {
    // NPC已死亡，不加入Squad
    return;
}
```

## 五、群体AI战斗的完整数据流

### 5.1 从.community到实际战斗的完整流程

```
1. Community定义阶段（.community文件）
   ├── Entry: "squad_leader"
   │   ├── characterRecordId: "Character.militech_sergeant"
   │   └── initializers: [SquadInitializer("base_defense")]
   ├── Entry: "squad_rifleman_01"
   │   ├── characterRecordId: "Character.militech_soldier"
   │   └── initializers: [SquadInitializer("base_defense")]
   └── Entry: "squad_rifleman_02"
       ├── characterRecordId: "Character.militech_soldier"
       └── initializers: [SquadInitializer("base_defense")]

                    ↓ (CommunitySystem激活)

2. 实例化阶段
   ├── CommunitySystem.ActivateCommunityEntry()
   ├── CommunityEntrySpawner.CreateEntityStub()
   │   ├── EntityStubSystem.RequestStubCreation()
   │   └── WorkspotManager.TryUseSpot()
   └── PopulationSystem.SpawnEntity(stub)

                    ↓ (Entity实例化完成)

3. Squad初始化阶段
   ├── SquadMemberComponent.OnAttach()
   ├── SquadInitializer.InitializeStub()
   │   ├── SquadManager.FindOrCreateSquad("base_defense")
   │   └── Squad.AddMember(npc)
   └── AttitudeManager.RegisterAgent(npc)
       └── 根据baseAttitudeGroup加入Attitude Squad

                    ↓ (NPC已加入Squad)

4. 战斗触发阶段
   ├── SenseManager.OnEnemyDetected(npc_01, player)
   │   └── npc_01感知到玩家
   └── npc_01.BehaviorTree.OnSpotEnemy()
       └── Squad.BroadcastEvent(EnemySpotted, player)

                    ↓ (Squad广播事件)

5. 小队响应阶段
   ├── squad_leader收到EnemySpotted事件
   │   ├── BehaviorTree切换到"CombatLeader"
   │   └── 发布Squad战术命令：
   │       ├── rifleman_01: "SuppressiveFire"
   │       └── rifleman_02: "FlankRight"
   │
   ├── squad_rifleman_01收到Order
   │   ├── SquadMemberBase.OnGiveOrder()
   │   ├── BehaviorTree切换到"SuppressiveFire"
   │   └── CombatSquad.RegisterTactic("Suppress", sectors, alley)
   │       └── InfluenceMapSystem.CreateLayer("Suppress_Layer")
   │
   └── squad_rifleman_02收到Order
       ├── SquadMemberBase.OnGiveOrder()
       ├── BehaviorTree切换到"FlankRight"
       ├── NavigationSystem.FindPath(current, flankPos)
       └── CombatSquad.CheckTacticInEffect(pos, memberId)
           └── 检查是否有其他成员已在执行侧翼

                    ↓ (战术执行中)

6. 战斗协同阶段
   ├── CombatSquad.AddEnemy(player)
   │   ├── Enemy.AddTracker(rifleman_01, updateCall)
   │   ├── Enemy.AddTracker(rifleman_02, updateCall)
   │   └── 所有成员共享敌人位置信息
   │
   ├── CombatSquad.Update()
   │   ├── UpdateTactics() - 更新战术层
   │   ├── UpdateReachability() - 更新可达性
   │   └── 评估战术效果
   │
   └── 持续战术调整
       ├── 如果玩家移动，重新计算CombatAlley
       ├── 如果成员死亡，选举新队长
       └── 如果玩家逃离，切换到Search阶段

                    ↓ (战斗结束)

7. 清理阶段
   ├── 如果所有敌人死亡/逃离
   │   └── CombatSquad.DropEnemy(player)
   ├── 如果Squad成员全部死亡
   │   └── SquadManager.DropSquad(squad)
   │       └── 10秒后自动清理
   └── Community可能切换Phase
       └── SetCurrentPhase("aftermath")
           └── 重新生成不同的Entry
```

## 六、系统耦合的设计优势

### 6.1 为什么需要这么多耦合系统？

```
✅ 职责分离：
   ├── CommunitySystem - 负责"谁在哪里"
   ├── SquadManager - 负责"如何协作"
   ├── AttitudeManager - 负责"谁是敌友"
   ├── InfluenceMapSystem - 负责"战术空间"
   └── BehaviorTree - 负责"如何行动"

✅ 灵活性：
   ├── 可以独立调整Squad战术而不影响Community生成
   ├── 可以修改AttitudeGroup关系而不改变Squad行为
   └── 可以增加新的战术层而不修改核心AI

✅ 性能优化：
   ├── InfluenceMap使用空间索引，避免N²复杂度
   ├── Squad只在有成员时存在，空Squad自动销毁
   └── EntityStub延迟实例化，减少内存占用

✅ 可扩展性：
   ├── 新增SquadType只需扩展SquadManager
   ├── 新增战术只需注册到CombatSquad
   └── 新增派系只需在TweakDB添加AttitudeGroup
```

### 6.2 系统间通信机制

```cpp
// 1. 直接调用（同步）
CommunitySystem → PopulationSystem.SpawnEntity()
CommunitySystem → WorkspotManager.TryUseSpot()

// 2. 事件广播（异步）
Squad.BroadcastEvent(EnemySpotted, player)
  → CallbackCollection遍历所有成员
  → SquadMemberBase.OnSquadEnemySpotted()

// 3. 回调注册（观察者模式）
SquadManager.m_onMemberAddedNotification.Add(callback)
AttitudeManager.RegisterAgentAttitudeGroupChange(entityId, callback)

// 4. 共享数据结构（InfluenceMap）
CombatSquad写入TacticalLayer
  → InfluenceMapSystem管理Layer
  → 其他Squad读取TacticalLayer

// 5. Query接口（只读查询）
AttitudeManager.GetGlobalAttitude(srcGroup, dstGroup)
SquadManager.FindSquad(squadName)
```

## 七、实战案例：军事检查站被攻击

### 7.1 场景设置

```cpp
// militech_checkpoint.community

Entry: "squad_leader"
├── Character.militech_sergeant
├── Squad: "checkpoint_alpha" (Community)
└── AttitudeGroup: "militech"

Entry: "squad_rifleman_01"
├── Character.militech_soldier
├── Squad: "checkpoint_alpha" (Community)
└── AttitudeGroup: "militech"

Entry: "squad_sniper"
├── Character.militech_sniper
├── Squad: "checkpoint_alpha" (Community)
├── AttitudeGroup: "militech"
└── Workspot: "sniper_tower_01"
```

### 7.2 战斗流程系统调用链

```
1. [T=0s] 玩家潜入检查站

   SenseManager: sniper感知到可疑目标
   ├── SensePreset.Relaxed → Alerted
   └── 未达到敌对阈值，继续观察

2. [T=5s] 玩家进入限制区域

   SenseManager: sniper确认敌对
   ├── OnEnemyDetected(sniper, player)
   └── sniper.BehaviorTree.OnSpotEnemy()
       └── Squad("checkpoint_alpha").BroadcastEvent(EnemySpotted, player)

3. [T=5.1s] 小队接收广播

   SquadManager协调：
   ├── squad_leader.OnSquadEnemySpotted(player)
   │   ├── BehaviorTree切换到"CombatLeader"
   │   └── IssueSquadOrder(Engage, player)
   │       ├── rifleman_01: "Move_And_Cover"
   │       └── sniper: "Overwatch"
   │
   └── AttitudeManager检查：
       └── GetGlobalAttitude("militech", "player") = Hostile
           └── 全部进入战斗状态

4. [T=5.2s] 创建CombatSquad

   SquadManager:
   ├── FindOrCreateCombatSquad("checkpoint_alpha_combat")
   ├── CombatSquad.Init()
   │   ├── m_influenceMapSystem获取
   │   ├── CreateDefensiveCombatAlley(from: player, to: squad)
   │   └── CreateOffensiveCombatAlley(from: squad, to: player)
   └── 将所有成员添加到CombatSquad

5. [T=5.3s] 添加敌人到CombatSquad

   CombatSquad.AddEnemy(player)
   ├── Enemy.AddTracker(leader, updateCall)
   ├── Enemy.AddTracker(rifleman_01, updateCall)
   ├── Enemy.AddTracker(sniper, updateCall)
   └── 所有成员共享玩家位置

6. [T=6s] 执行战术命令

   rifleman_01执行"Move_And_Cover":
   ├── SquadBase.GiveOrder(rifleman_01, "TakeCover")
   ├── rifleman_01.OnGiveOrder()
   │   └── BehaviorTree.SwitchTo("TakeCover")
   │       ├── CombatSpaceManager.FindBestCover(pos, enemyPos)
   │       ├── NavigationSystem.FindPath(current, coverPos)
   │       └── 移动到掩体
   └── CombatSquad.RegisterTactic("Defensive_Cover", ...)
       └── InfluenceMapSystem.CreateLayer("Defensive_Cover_Layer")
           └── 标记掩体区域为"已占用"

7. [T=7s] sniper执行"Overwatch"

   sniper执行"Overwatch":
   ├── 离开Workspot (WorkspotManager.ReleaseSpot())
   ├── SquadBase.GiveOrder(sniper, "Overwatch")
   ├── BehaviorTree.SwitchTo("Overwatch")
   │   ├── CombatSquad.GetOffensiveCombatAlley()
   │   │   └── 获取最佳射击角度
   │   └── 保持高位优势
   └── CombatSquad.RegisterTactic("Suppressive_Fire", sectors, alley)

8. [T=10s] 持续战斗更新

   SquadManager.Update():
   └── CombatSquad.Update()
       ├── UpdateTactics()
       │   ├── 检查战术超时
       │   ├── 更新TacticalLayer
       │   └── 评估战术覆盖率
       ├── UpdateReachability()
       │   ├── InfluenceMapSystem计算可达性
       │   └── 动态调整战术
       └── UpdateEnemies()
           ├── Enemy.Update() - 更新玩家位置
           └── 如果玩家逃离，BroadcastEvent(EnemyLost)

9. [T=15s] 队长死亡

   EntitySpawnerEventsBroadcaster:
   ├── OnEntityDeath(squad_leader)
   └── SquadBase.DropMember(squad_leader)
       ├── OnMemberRemoved(squad_leader)
       ├── 选举新队长（rifleman_01）
       └── 新队长继续指挥

10. [T=20s] 玩家撤离

    SenseManager.OnTargetLost():
    ├── CombatSquad.DropEnemy(player)
    ├── Squad.BroadcastEvent(SearchMode)
    └── BehaviorTree切换到"SearchBehavior"

11. [T=30s] 战斗结束

    SquadManager.Update():
    ├── Squad成员数量 = 1 (只剩sniper)
    ├── Enemy数量 = 0
    ├── Lifetime倒计时开始（10秒）
    └── [T=40s] Squad自动销毁
        ├── SquadManager.DropSquad("checkpoint_alpha_combat")
        └── CombatSquad.Deinit()
            └── InfluenceMapSystem.DestroyLayer("Defensive_Cover_Layer")
```

## 八、总结

### 8.1 核心要点

```
1. CommunitySystem耦合了10+个系统：
   ├── 生成层：Population, EntityStub, Persistency, Workspot
   ├── AI层：Sense, TargetTracker, Behavior Tree
   └── 战斗层：Squad, Attitude, InfluenceMap, CombatSpace

2. SquadManager是完全独立的IGameSystem：
   ├── 管理所有Squad实例
   ├── 提供7种SquadType
   └── 与InfluenceMap、Navigation、Persistency集成

3. AI群组定义采用三层架构：
   ├── Layer 1: AttitudeGroup（派系）- 全局敌友关系
   ├── Layer 2: Squad（小队）- 战术协同单位
   └── Layer 3: Community（场景）- 生命周期管理

4. 系统间通信机制：
   ├── 直接调用（同步）
   ├── 事件广播（异步）
   ├── 回调注册（观察者）
   ├── 共享数据（InfluenceMap）
   └── Query接口（只读）
```

### 8.2 设计哲学

```
"不是让AI自己形成群体，
 而是通过精心设计的系统协作，
 让每个系统专注于自己的职责，
 通过清晰的接口和事件机制，
 实现复杂的群体AI战斗行为。"

这就是为什么2077的群体AI能同时做到：
- 战术深度（Squad战术命令）
- 场景一致性（Community生命周期）
- 派系关系（AttitudeGroup矩阵）
- 性能优化（InfluenceMap空间索引）
```
