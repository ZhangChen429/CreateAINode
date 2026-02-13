# Community生命周期管理与帮派群体系统深度解析

## 一、Community的生命周期管理

### 1.1 完整的生命周期状态机

```cpp
Community生命周期（7个关键阶段）

1. [Creation] 创建
   ├── WorldAttached()
   │   └── CommunitySystem初始化
   ├── CreateCommunities(registryData)
   │   ├── CreateCommunitiesInternal()
   │   │   ├── 验证数据 (VerifyCommunityRegistryItem)
   │   │   ├── 创建Community实例
   │   │   ├── 创建所有CommunityEntry
   │   │   └── 设置初始Phase状态
   │   └── 加入m_communities映射表
   └── 状态：Inactive（创建但未激活）

2. [Activation] 激活
   ├── ActivateCommunityEntry(id, entryName)
   │   └── 加入m_pendingRequests队列
   ├── ProcessPendingRequests()
   │   └── ActivateCommunityInternal()
   │       └── entry->Activate()
   │           ├── m_spawner->Activate()
   │           │   └── StartStubsCreation()
   │           │       ├── 恢复存档的EntityStub
   │           │       └── 创建新的EntityStub
   │           └── m_timePeriodScheduler->Activate()
   └── 状态：Active（激活，等待Area）

3. [Area Stream In] 区域流入
   ├── OnCommunityAreaAttached(id, area)
   │   └── 加入m_pendingStreamInOutRequests队列
   ├── ProcessStreamInOutRequests()
   │   └── OnCommunityAreaStreamedInInternal()
   │       └── entry->OnCommunityAreaStreamedIn(area)
   │           ├── m_spawner->OnAreaStreamedIn(area)
   │           │   ├── 获取workspot位置数据
   │           │   └── StartStubsCreation()
   │           │       └── 实际创建EntityStub
   │           └── 分配workspots给stubs
   └── 状态：Active + StreamedIn（运行中）

4. [Runtime Operation] 运行时操作
   ├── Update() - 每帧更新
   │   ├── ProcessPendingRequests()
   │   ├── ProcessStreamInOutRequests()
   │   └── 更新所有active entries
   ├── 时间段切换
   │   ├── OnTimePeriodChanged()
   │   │   └── entry->OnCurrentTimePeriodChanged()
   │   │       ├── 删除多余的NPC
   │   │       └── 创建新的NPC
   ├── Phase切换
   │   └── SetCurrentCommunityEntryPhase(id, entry, phase)
   │       ├── entry->SetCurrentPhase(phase)
   │       │   ├── 停止当前Phase的spawner
   │       │   ├── 切换到新Phase配置
   │       │   └── 重新启动spawner
   │       └── 可能改变：数量、workspot、AI参数
   └── 状态：Active + StreamedIn（持续运行）

5. [Area Stream Out] 区域流出
   ├── OnCommunityAreaDetached(id, area)
   │   └── 加入m_pendingStreamInOutRequests队列
   ├── ProcessStreamInOutRequests()
   │   └── OnCommunityAreaStreamedOutInternal()
   │       └── entry->OnCommunityAreaStreamedOut(area)
   │           └── m_spawner->OnAreaStreamedOut(area)
   │               ├── StopStubsCreation()
   │               │   ├── 删除所有EntityStub
   │               │   └── 清除workspot标记
   │               └── 保存状态（如果可存档）
   └── 状态：Active（激活但无Area）

6. [Deactivation] 停用
   ├── DeactivateCommunityEntry(id, entryName)
   │   └── 加入m_pendingRequests队列
   ├── ProcessPendingRequests()
   │   └── DeactivateCommunityInternal()
   │       └── entry->Deactivate()
   │           ├── m_spawner->Deactivate()
   │           │   ├── StopStubsCreation()
   │           │   └── 保存死亡计数器状态
   │           └── m_timePeriodScheduler->Deactivate()
   └── 状态：Inactive

7. [Deletion] 删除
   ├── WorldPendingDetach()
   │   ├── DeactivateAllCommunities()
   │   └── DeleteAllCommunities()
   │       └── m_communities.Clear()
   ├── DeleteCommunities(ids)
   │   └── DeleteCommunitiesInternal()
   │       └── m_communities.Remove(id)
   └── 状态：Destroyed
```

### 1.2 关键代码实现

**创建流程**:
```cpp
// common/gamePopulation/src/communitySystem.cpp:561

void CommunitySystem::CreateCommunitiesInternal(
    const DynArray<world::CommunityRegistryItem>& newCommunitiesData
) {
    RED_SCOPE_LOCK(m_communitiesMapLock);

    for (const auto& communityRegistryItem : newCommunitiesData) {
        const CommunityID& newCommunityId = communityRegistryItem.m_communityId;

        if (GetCommunityNoLock(newCommunityId) == nullptr) {
            if (VerifyCommunityRegistryItem(communityRegistryItem)) {
                // 创建Community实例
                Community::CreationContext ctx = {
                    communityRegistryItem,
                    *m_communityEntryUpdateScheduler,
                    *m_timePeriodCallbacksHandler,
                    *m_workspotsMarker,
                    gameInstance,
                    *m_tweakDBDataContainer,
                    FindSavedState(newCommunityId)  // 存档恢复
                };

                m_communities.Emplace(
                    newCommunityId,
                    CreateSharedPtr<Community>(ctx)
                );
            }
        }
    }
}
```

**激活流程**:
```cpp
// common/gamePopulation/src/community.cpp:78

void Community::ActivateEntry(const CName& entryName) {
    if (entryName.Empty()) {
        // 激活所有Entry
        for (BaseCommunityEntry* entry : m_allEntries) {
            entry->Activate();
        }
    } else {
        // 激活指定Entry
        BaseCommunityEntry* entry = GetEntry(entryName);
        if (entry != nullptr) {
            entry->Activate();
            //   ↓
            //   CommunityEntrySpawner::Activate()
            //   ↓
            //   StartStubsCreation() - 开始生成NPC
        }
    }
}
```

**Area流式加载**:
```cpp
// common/gamePopulation/src/communitySystem.cpp:269

void CommunitySystem::OnCommunityAreaAttached(
    const CommunityID& communityId,
    const THandle<community::Area>& communityArea
) {
    if (const SharedPtr<const Community> community = GetCommunity(communityId).Lock()) {
        DynArray<CName> entriesNames;
        community->GetEntriesNames(entriesNames);

        RED_SCOPE_LOCK(m_pendingStreamInOutRequestsLock);
        for (const CName& entryName : entriesNames) {
            // 加入待处理队列（异步处理，避免卡顿）
            m_pendingStreamInOutRequests.EmplaceBack(
                communityId,
                entryName,
                communityArea,
                StreamInOutRequestType::StreamIn
            );
        }
    }
}
```

### 1.3 性能优化的生命周期管理

```cpp
关键性能优化策略：

1. 异步请求队列（避免卡顿）
   ├── m_pendingRequests
   │   └── Activate/Deactivate/SetPhase请求
   └── m_pendingStreamInOutRequests
       └── StreamIn/StreamOut请求

   每帧处理：
   ProcessPendingRequests() {
       while (有请求 && 未超时间限制) {
           处理一个请求;
       }
   }

2. 分帧EntityStub创建
   UpdateStubsCreation() {
       创建少量stubs;
       if (还有未创建的) {
           ScheduleStubsCreationUpdate(0.8~1.0秒后);
       }
   }

3. 流式加载时间限制
   static constexpr Double s_streamInOutRequestsTimeLimitPerFrameSec = 0.001;
   // 每帧最多1ms用于处理Area流入/流出

4. 懒加载机制
   - Community创建时不加载EntityStub
   - Area流入时才真正创建Entity
   - Area流出时立即删除Entity

5. 对象池重用
   m_reusableEntityIds - 存储可重用的EntityID
   避免频繁分配/释放EntityID
```

## 二、管理群体AI的核心价值

### 2.1 核心价值分析

```
为什么需要Community System来管理群体AI？

传统方法的问题：
❌ 直接放置Entity → 内存占用大，加载慢
❌ 手动管理生命周期 → 复杂，易出错
❌ 无统一控制 → 难以实现动态场景
❌ 难以扩展 → 添加新NPC需要修改多处

Community System的核心价值：

┌────────────────────────────────────────────────────────────┐
│ 1. 统一的生命周期管理（Unified Lifecycle）                │
├────────────────────────────────────────────────────────────┤
│ 价值：                                                     │
│ • 一个API控制整个场景（一行代码激活/停用所有NPC）         │
│ • 自动处理创建、激活、流式加载、销毁                      │
│ • 减少Bug（不会遗忘清理、不会重复创建）                   │
│                                                            │
│ 示例：                                                     │
│ SetCurrentPhase("combat") - 瞬间切换整个场景              │
│ • 平民消失                                                 │
│ • 守卫增加                                                 │
│ • 增援到达                                                 │
│ • AI参数改变                                               │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ 2. 内存和性能优化（Memory & Performance）                 │
├────────────────────────────────────────────────────────────┤
│ 价值：                                                     │
│ • 按需加载：玩家靠近时才创建Entity                        │
│ • 流式卸载：玩家离开时立即释放内存                        │
│ • 分帧创建：避免卡顿                                       │
│ • 对象池复用：减少内存分配                                │
│                                                            │
│ 效果：                                                     │
│ 开放世界有1000+个Community                                │
│ 但同时只加载玩家附近的20-30个                             │
│ 内存占用降低97%！                                         │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ 3. 动态场景支持（Dynamic Scenarios）                      │
├────────────────────────────────────────────────────────────┤
│ 价值：                                                     │
│ • Phase切换实现多阶段剧情                                  │
│ • TimePeriod自动昼夜变化                                   │
│ • 任务系统深度集成                                         │
│ • 支持复杂的叙事设计                                       │
│                                                            │
│ 示例：                                                     │
│ 任务："潜入帮派基地"                                       │
│ Phase 1: "normal" - 日常巡逻（5个守卫）                   │
│ Phase 2: "alert" - 警报触发（10个守卫+增援）              │
│ Phase 3: "lockdown" - 封锁（20个守卫+重装单位）           │
│ Phase 4: "cleanup" - 任务后（尸体+清理人员）              │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ 4. 持久化世界状态（Persistent World）                     │
├────────────────────────────────────────────────────────────┤
│ 价值：                                                     │
│ • 保存NPC死亡状态                                          │
│ • 记录玩家的影响                                           │
│ • 真实的世界变化                                           │
│ • 避免"不死"守卫                                           │
│                                                            │
│ 实现：                                                     │
│ CommunitySavedState {                                      │
│     createdEntityStubIds,    // 存活的NPC                  │
│     totalDeadEntitiesCount,  // 死亡数量                   │
│     extractedStubs,          // 被提取的NPC（任务用）      │
│     currentPhaseName         // 当前阶段                   │
│ }                                                          │
│                                                            │
│ 玩家杀死3个守卫 → 存档 → 加载后只剩2个！                  │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ 5. 设计师友好（Designer-Friendly）                        │
├────────────────────────────────────────────────────────────┤
│ 价值：                                                     │
│ • 可视化编辑（.community文件）                            │
│ • 无需编程知识                                             │
│ • 模板化复用                                               │
│ • 快速迭代                                                 │
│                                                            │
│ 工作流程：                                                 │
│ 1. 设计师在编辑器中创建.community                         │
│ 2. 配置Entry、Phase、TimePeriod                           │
│ 3. 放置Area到场景                                          │
│ 4. 测试 → 调整 → 完成                                     │
│                                                            │
│ 不需要：                                                   │
│ ✗ 写代码                                                   │
│ ✗ 手动管理Entity                                           │
│ ✗ 处理流式加载                                             │
│ ✗ 写存档逻辑                                               │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ 6. 群体行为协调（Group Coordination）                     │
├────────────────────────────────────────────────────────────┤
│ 价值：                                                     │
│ • 统一时间驱动（所有NPC同步切换状态）                     │
│ • Squad系统集成                                            │
│ • 共享配置（同一个.community的NPC共享设置）               │
│ • 一致性保证                                               │
│                                                            │
│ 效果：                                                     │
│ 所有守卫在同一时刻：                                       │
│ • 从日班切换到夜班                                         │
│ • 从巡逻切换到战斗                                         │
│ • 从正常切换到警戒                                         │
│                                                            │
│ 而不是各自为政，时间不同步！                              │
└────────────────────────────────────────────────────────────┘
```

### 2.2 与传统方法的对比

```cpp
// 传统方法（直接放置Entity）

场景中：
├── Guard_Entity_01
├── Guard_Entity_02
├── Guard_Entity_03
├── Guard_Entity_04
└── Guard_Entity_05

问题：
1. 所有Entity同时加载 → 内存浪费
2. 需要手动切换状态 → 脚本复杂
3. 无法动态调整数量 → 灵活性差
4. 死亡后立即respawn → 不真实
5. 无统一管理 → 难以维护

---

// Community System方法

.community文件：
guards.community {
    Entry: "patrol_guards"
    CharacterRecord: "Character.gang_guard"
    Phase: "normal" {
        TimePeriod: Day, quantity: 3
        TimePeriod: Night, quantity: 5
    }
    Phase: "combat" {
        quantity: 10
    }
}

优势：
✓ 按需加载（玩家靠近时才创建）
✓ 一行代码切换（SetPhase("combat")）
✓ 动态数量（白天3个，夜晚5个）
✓ 死亡持久化（死了就不会再生）
✓ 集中管理（一个文件控制所有）

性能对比：
传统方法：5个Entity * 200KB = 1MB（常驻内存）
Community：  0KB（未加载）或 1MB（加载时）
           流式卸载后回到0KB！
```

## 三、帮派能否共享同一个行为树？

### 3.1 答案：是！帮派成员共享行为树实例

```cpp
关键理解：

行为树（Behavior Tree）= "程序代码"
行为树实例（Instance）= "运行中的程序"

类比：
.behavior文件      →  .exe可执行文件
BehaviorTreeInstance  →  运行中的进程

共享机制：
同一个帮派的所有成员：
✓ 共享相同的.behavior文件（代码）
✓ 各自有独立的BehaviorTreeInstance（进程）
✓ 共享BehaviorTree资源（节省内存）
✗ 不共享运行状态（各自独立决策）
```

**代码实现**:

```cpp
// 行为树的加载和实例化

1. Character_Record定义
Character.maelstrom_grunt {
    archetypeData: "ArchetypeData.GenericMeleeT2"
    // ↓ 指向同一个Archetype
}

2. AI::Archetype资源（共享）
ArchetypeData.GenericMeleeT2 {
    archetype: "base/gameplay/ai/archetypes/humanoid.aiarch"
    // ↓ 所有使用这个Archetype的NPC共享这个资源
}

3. AI::Archetype实例（共享）
humanoid.aiarch {
    behaviorDefinition: {
        treeDefinition: "base/gameplay/ai/behaviors/root.behavior"
        // ↓ 所有humanoid NPC共享这个行为树文件
        CombatBehavior: "combat/humanoid/humanoid_melee.behavior"
    }
}

4. BehaviorTreeInstance（独立）
每个NPC创建时：
NPC_A.CreateAIComponent() {
    // 加载共享的Behavior Tree资源
    BehaviorResource* sharedResource = LoadBehavior("root.behavior");

    // 创建独立的实例
    myBehaviorInstance = CreateInstance(sharedResource);
    // ↑ 独立的Blackboard、独立的状态
}

内存占用：
.behavior文件（root.behavior）: 500KB（共享，所有NPC只加载一次）
BehaviorInstance每个：           10KB（独立，每个NPC一个）

10个Maelstrom成员：
传统方法：500KB * 10 = 5MB
共享方法：500KB + 10KB * 10 = 600KB
节省内存：4.4MB (88%!)
```

### 3.2 帮派的差异化配置

虽然共享行为树，但帮派成员仍可以有差异：

```cpp
差异化的4个层次：

1. Archetype参数覆盖（Override）
Character.maelstrom_grunt {
    archetypeData: "ArchetypeData.GenericMeleeT2" {
        // 覆盖参数
        arguments: {
            aggressiveness: 0.8,    // 激进度
            reactionTime: 0.3,      // 反应时间
            coverPreference: 0.2    // 掩体偏好（低=更激进）
        }
    }
}

Character.maelstrom_veteran {
    archetypeData: "ArchetypeData.GenericMeleeT3" {
        arguments: {
            aggressiveness: 0.9,    // 老兵更激进
            reactionTime: 0.15,     // 反应更快
            health: 200             // 血量更高
        }
    }
}

2. 装备差异（Equipment）
Character.maelstrom_grunt {
    primaryEquipment: "EquipmentGroup.MaelstromMelee"
    // ↑ 棍棒、拳套
}

Character.maelstrom_veteran {
    primaryEquipment: "EquipmentGroup.MaelstromElite"
    // ↑ 赛博武器、重型武器
}

3. Initializers（运行时配置）
.community文件中：
Entry: "maelstrom_leader" {
    characterRecordId: "Character.maelstrom_sergeant"
    initializers: [
        SquadInitializer {
            entries: [{type: Community, value: "maelstrom_squad_alpha"}]
        },
        PatrolInitializer {
            patrolRole: PatrolRole.Leader  // 领导巡逻
        }
    ]
}

Entry: "maelstrom_grunt_01" {
    characterRecordId: "Character.maelstrom_grunt"
    initializers: [
        SquadInitializer {
            entries: [{type: Community, value: "maelstrom_squad_alpha"}]
        },
        PatrolInitializer {
            patrolRole: PatrolRole.Follower  // 跟随巡逻
        }
    ]
}

4. TweakDB动态参数
运行时可以修改：
TweakDB.SetFlat("Character.maelstrom_grunt.health", 150);
// 动态调整所有grunt的血量
```

### 3.3 实际内存共享示例

```cpp
场景：10个Maelstrom帮派成员

共享的资源（只加载一次）：
├── root.behavior (500KB)
├── combat_root.behavior (300KB)
├── humanoid_melee.behavior (200KB)
├── AI::Archetype (50KB)
├── Character_Record (10KB)
└── 总计：1.06MB（共享）

独立的资源（每个NPC一份）：
├── BehaviorTreeInstance (10KB)
├── Blackboard数据 (5KB)
├── AIComponent (8KB)
├── Entity数据 (50KB)
└── 每个NPC：73KB

10个NPC总内存：
1.06MB（共享） + 73KB * 10 = 1.79MB

如果不共享：
(500KB + 300KB + 200KB + 50KB + 10KB + 73KB) * 10 = 11.33MB

节省内存：11.33MB - 1.79MB = 9.54MB (84%!)

这就是Community System管理群体AI的核心价值！
```

## 四、如何实现2077中复杂的帮派群体系统

### 4.1 帮派系统的完整架构

```
2077帮派系统 = 4层架构

┌─────────────────────────────────────────────────────────────┐
│ Layer 1: Attitude System（态度系统）- 全局派系关系        │
├─────────────────────────────────────────────────────────────┤
│ AttitudeGroup（态度组）层次结构：                          │
│                                                             │
│ gangs (根)                                                  │
│ ├── maelstrom                                               │
│ │   ├── maelstrom_base                                      │
│ │   ├── maelstrom_leadership                                │
│ │   └── maelstrom_recruits                                  │
│ ├── tyger_claws                                             │
│ │   ├── tyger_claws_base                                    │
│ │   └── tyger_claws_elite                                   │
│ ├── valentinos                                              │
│ │   ├── valentinos_base                                     │
│ │   └── valentinos_leadership                               │
│ ├── 6th_street                                              │
│ └── scavengers                                              │
│                                                             │
│ 态度矩阵（Attitude Matrix）：                              │
│             player  ncpd  maelstrom  tyger  valentinos      │
│ player        -      友好    敌对     中立     中立        │
│ ncpd         友好     -      敌对     敌对     敌对        │
│ maelstrom    敌对    敌对     -       敌对     敌对        │
│ tyger_claws  中立    敌对    敌对      -       敌对        │
│ valentinos   中立    敌对    敌对     敌对      -          │
│                                                             │
│ 实现：                                                      │
│ Character.maelstrom_grunt {                                 │
│     baseAttitudeGroup: "maelstrom"  // 自动敌对其他帮派   │
│     affiliation: "Factions.Maelstrom"                       │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Layer 2: Squad System（小队系统）- 战术协同                │
├─────────────────────────────────────────────────────────────┤
│ 帮派小队类型：                                              │
│                                                             │
│ 1. Combat Squad（战斗小队）                                 │
│    ├── 共享敌人信息                                         │
│    ├── 协同战术（掩护、包抄）                              │
│    └── 战术角色分配                                         │
│                                                             │
│ 2. Community Squad（社区小队）                              │
│    ├── 同一Community的帮派成员                             │
│    ├── 共享警戒状态                                         │
│    └── 统一反应                                             │
│                                                             │
│ 3. Audio Squad（音频小队）                                  │
│    ├── 协调语音对话                                         │
│    └── 避免同时说话                                         │
│                                                             │
│ 实现：                                                      │
│ .community文件中所有帮派成员：                             │
│ initializers: [                                             │
│     SquadInitializer {                                      │
│         entries: [                                          │
│             {type: Community, value: "maelstrom_base_alpha"}│
│         ]                                                   │
│     }                                                       │
│ ]                                                           │
│                                                             │
│ 效果：一人发现玩家 → 全队进入战斗                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Layer 3: Community System（社区系统）- 场景管理            │
├─────────────────────────────────────────────────────────────┤
│ 帮派据点Community配置：                                     │
│                                                             │
│ maelstrom_hideout.community {                               │
│     entries: [                                              │
│         // 1. 入口守卫                                      │
│         {                                                   │
│             entryName: "gate_guards"                        │
│             characterRecordId: "Character.maelstrom_grunt" │
│             phases: [                                       │
│                 {                                           │
│                     phaseName: "normal"                     │
│                     timePeriods: [                          │
│                         {hour: Day, quantity: 2},           │
│                         {hour: Night, quantity: 4}          │
│                     ]                                       │
│                 },                                          │
│                 {                                           │
│                     phaseName: "alert"                      │
│                     quantity: 6  // 警报时增加              │
│                 }                                           │
│             ],                                              │
│             initializers: [SquadInitializer...]             │
│         },                                                  │
│                                                             │
│         // 2. 巡逻队                                        │
│         {                                                   │
│             entryName: "patrol_leader"                      │
│             characterRecordId: "Character.maelstrom_sergeant"│
│             initializers: [                                 │
│                 SquadInitializer,                           │
│                 PatrolInitializer                           │
│             ]                                               │
│         },                                                  │
│                                                             │
│         // 3. 内部成员                                      │
│         {                                                   │
│             entryName: "hideout_members"                    │
│             characterRecordId: "Character.maelstrom_thug"  │
│             phases: [                                       │
│                 {                                           │
│                     phaseName: "normal"                     │
│                     quantity: 8,                            │
│                     workspots: [...]  // 坐着、站着         │
│                 }                                           │
│             ]                                               │
│         },                                                  │
│                                                             │
│         // 4. 首领                                          │
│         {                                                   │
│             entryName: "boss"                               │
│             characterRecordId: "Character.maelstrom_boss"  │
│             phases: [                                       │
│                 {phaseName: "normal", quantity: 1}          │
│             ]                                               │
│         }                                                   │
│     ]                                                       │
│ }                                                           │
│                                                             │
│ 总计：2-4个入口守卫 + 1个巡逻队 + 8个内部成员 + 1个首领   │
│       = 12-14个帮派成员动态管理                            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Layer 4: Behavior Tree（行为树）- AI决策                   │
├─────────────────────────────────────────────────────────────┤
│ 帮派成员共享的行为树层次：                                 │
│                                                             │
│ root.behavior                                               │
│ ├── Idle（闲置）                                            │
│ │   ├── Gang Ambient（帮派环境行为）                       │
│ │   │   ├── Smoke（抽烟）                                   │
│ │   │   ├── Talk to Gang Members（与帮派成员交谈）         │
│ │   │   ├── Check Weapons（检查武器）                      │
│ │   │   └── Intimidate Passerby（恐吓路人）                │
│ │   └── Workspot Actions（工作点动作）                     │
│ │                                                           │
│ ├── Alerted（警戒）                                         │
│ │   ├── Gang Alert Behavior（帮派警戒）                    │
│ │   │   ├── Call for Backup（呼叫增援）                    │
│ │   │   ├── Coordinate with Squad（与小队协调）            │
│ │   │   └── Search Area（搜索区域）                        │
│ │   └── Report to Boss（向首领报告）                       │
│ │                                                           │
│ ├── Combat（战斗）                                          │
│ │   ├── Gang Combat Behavior（帮派战斗）                   │
│ │   │   ├── Aggressive Melee（激进近战）                   │
│ │   │   ├── Coordinated Fire（协调射击）                   │
│ │   │   ├── Use Cyberware（使用赛博装备）                  │
│ │   │   └── Protect Leader（保护首领）                     │
│ │   └── Squad Tactics（小队战术）                          │
│ │                                                           │
│ ├── Retreat（撤退）                                         │
│ │   ├── Fall Back to Hideout（撤回据点）                   │
│ │   └── Regroup with Squad（重新集结）                     │
│ │                                                           │
│ └── Special Behaviors（特殊行为）                          │
│     ├── Execute Hostage（处决人质）                        │
│     ├── Loot Bodies（搜刮尸体）                            │
│     └── Set Traps（设置陷阱）                              │
│                                                             │
│ 帮派差异化：                                                │
│ Maelstrom: 更激进，使用赛博武器，喜欢近战                 │
│ Tyger Claws: 更有纪律，使用刀剑，注重荣誉                 │
│ Valentinos: 重视领地，使用手枪，保护平民                  │
│ 6th Street: 军事化，使用突击步枪，守护社区                │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 实际案例：Maelstrom据点攻防战

```cpp
场景：玩家潜入Maelstrom据点

初始状态（Phase: "normal"）：
├── Community: maelstrom_hideout
│   ├── gate_guards (2人，站岗)
│   ├── patrol_team (1队长 + 2成员，巡逻)
│   ├── hideout_members (8人，闲聊、检查武器)
│   └── boss (1人，在办公室)
│
│   所有成员：
│   └── AttitudeGroup: "maelstrom"
│   └── Squad: "maelstrom_hideout_alpha"
│   └── BehaviorTree: root.behavior (共享)

═══════════════════════════════════════════════════════════

触发事件1：玩家被gate_guard_01发现

gate_guard_01.BehaviorTree {
    OnSpotEnemy(player) {
        // 1. 自己进入警戒
        SwitchState(Alerted);

        // 2. 通知小队
        MySquad->BroadcastEvent(EnemySpotted, player);

        // 3. 呼叫增援（特殊行为）
        CallForBackup();
    }
}

═══════════════════════════════════════════════════════════

Squad响应（所有14个成员）:

Squad.BroadcastEvent(EnemySpotted, player) {
    for (member in AllMembers) {
        member->OnSquadEnemySpotted(player);
        // ↓
        // 每个成员的BehaviorTree切换到Alerted状态
    }
}

同时发生：
├── gate_guards: 进入战斗位置
├── patrol_team: 放弃巡逻，快速接近玩家
├── hideout_members: 放下手中物品，准备战斗
└── boss: 保持在办公室，等待报告

═══════════════════════════════════════════════════════════

触发事件2：玩家开火（Community Phase切换）

任务系统：
SetCurrentCommunityEntryPhase("maelstrom_hideout", CName::NONE(), "combat");

Phase切换效果：
├── gate_guards: 数量从2增加到6（增援到达）
├── patrol_team: 不变，但AI参数改变（更激进）
├── hideout_members: 全部切换到战斗姿态
├── boss: 可能撤退或加入战斗
└── 可选：生成新的reinforcement Entry（10+增援）

═══════════════════════════════════════════════════════════

战斗过程（Squad战术协调）:

Squad.Update() {
    // 1. 分析战场
    AnalyzeBattlefield();
    // - 玩家位置：入口
    // - 玩家武器：远程
    // - 掩体情况：玩家有利

    // 2. 分配战术角色
    AssignSquadRoles();
    // - Suppressor (3人): 火力压制
    // - Flanker (4人): 侧翼包抄
    // - Support (2人): 后方支援
    // - Guard (2人): 保护首领
    // - Reserve (3人): 预备队

    // 3. 发布命令
    IssueSquadOrders();
}

每个成员根据角色执行战术：
├── Suppressor: 持续射击，限制玩家移动
├── Flanker: 绕到侧面和后方
├── Support: 投掷手雷，使用赛博武器
├── Guard: 守住首领房间
└── Reserve: 等待时机

═══════════════════════════════════════════════════════════

玩家击杀5个成员后：

Squad.OnMemberKilled(memberId) {
    m_casualties++;

    if (m_casualties >= 5 && m_totalMembers < 10) {
        // 损失过大，考虑撤退
        IssueRetreatOrder();
    }
}

撤退行为：
├── 剩余成员撤回内部
├── 设置陷阱
├── 准备最后防线
└── 首领可能逃跑

═══════════════════════════════════════════════════════════

任务完成后（Phase: "defeated"）:

SetCurrentCommunityEntryPhase("maelstrom_hideout", CName::NONE(), "defeated");

Phase效果：
├── 所有活着的成员: Deactivate()（撤离或投降）
├── dead_bodies Entry: Activate()（生成尸体）
├── 据点标记为：玩家控制
└── 保存状态：这个据点被清空了

存档数据：
CommunitySavedState {
    currentPhaseName: "defeated",
    totalDeadEntitiesCount: 9,  // 9个死亡
    createdEntityStubIds: [],   // 0个存活
    extractedStubs: [boss_id]   // 首领逃走了（任务用）
}

玩家再次访问：
└── 据点保持清空状态，不会respawn！
```

### 4.3 帮派系统的核心技术总结

```cpp
2077帮派系统的6大核心技术：

1. 态度继承（Attitude Hierarchy）
   ├── 层次化的AttitudeGroup
   ├── 子组自动继承父组的态度
   └── 动态修改态度关系

   示例：
   player攻击maelstrom_grunt
   → maelstrom组态度变为敌对
   → 所有maelstrom子组（base/leadership/recruits）自动敌对
   → 所有Maelstrom帮派成员攻击玩家

2. 小队战术系统（Squad Tactics）
   ├── 自动角色分配
   ├── 实时战术调整
   └── 协同攻击

3. 共享行为树（Shared Behavior Trees）
   ├── 节省内存（84%）
   ├── 统一AI逻辑
   └── 参数化差异

4. Phase动态切换（Dynamic Phase）
   ├── 多阶段剧情
   ├── 战斗强度调整
   └── 增援系统

5. 持久化世界（Persistent World）
   ├── 死亡不复活
   ├── 据点状态保存
   └── 玩家影响持久化

6. 流式性能优化（Streaming Optimization）
   ├── 按需加载
   ├── 分帧创建
   └── 自动卸载

这就是为什么2077的帮派战斗感觉如此真实和有深度！
```

## 五、总结：Community System的设计哲学

```
Community System = 游戏世界的"操作系统"

不是简单的NPC管理器
而是一个完整的世界模拟框架：

核心理念：
"让设计师能够像导演一样，
 精确控制每个场景的每个元素，
 同时保持高性能和真实感。"

三大支柱：
1. 统一管理（Unified Management）
   → 一个系统控制所有

2. 按需计算（On-Demand Computation）
   → 只计算玩家看得到的

3. 数据驱动（Data-Driven）
   → 设计师配置，程序员赋能

最终成果：
✓ 开放世界的规模感
✓ 任务场景的精确性
✓ 战斗系统的深度
✓ 性能的可控性
✓ 设计的灵活性

这就是2077 Community System的真正价值！
```
