# WorkSpotTree 通过 Entry 组织 NPC 点位完整行为的宏观总结

## 一、WorkspotTree 的整体结构



```
WorkspotTree (工作点资源)

├── m\_rootEntry (根Entry，必须是IContainerEntry类型)

│   └── m\_list (子Entry列表)

│       ├── EntryAnim节点 (进入动画组)

│       ├── Sequence/Selector节点 (循环行为组)

│       ├── ReactionSequence节点 (反应动画组) ⚠️必须在根层级

│       └── ExitAnim节点 (退出动画组)

└── m\_customTransitionAnims (自定义过渡动画列表)
```

## 二、NPC 在点位的完整生命周期

### 阶段 1: 进入阶段 (Entry Phase)

#### 根层级结构



```
Root Sequence/RandomList (m\_idleAnim = "stand")

├── EntryAnim (进入动画)

│   ├── m\_animName = "walk\_to\_stand"  // 从行走到站立的动画

│   ├── m\_idleAnim = "stand"          // 目标idle状态

│   ├── m\_movementType = Walk         // 移动类型

│   └── 内部包裹的Sequence

│       ├── m\_idleAnim = "stand"      // 到达后的idle状态

│       └── m\_list → \[后续行为节点]
```

#### 运作机制:



1. NPC 从外部移动到 workspot

2. 系统调用`GetClosestEntryAnim()`查找最合适的 EntryAnim

3. 播放 EntryAnim 的`m_animName`动画（带运动数据）

4. 动画结束后，NPC 进入 EntryAnim 内部的 Sequence，处于`m_idleAnim`指定的 idle 状态

### 阶段 2: 循环行为阶段 (Loop Phase)

#### 2.1 简单循环 - Sequence



```
Sequence (m\_idleAnim = "sit", m\_loopInfinitely = true)

├── AnimClip (m\_animName = "sit\_fidget\_01", m\_blendOutTime = 0.5)

├── PauseClip (m\_timeMin = 2.0, m\_timeMax = 5.0)

├── AnimClip (m\_animName = "sit\_look\_around")

└── \[循环回到开始]
```

**行为表现**: NPC 坐在椅子上 → 播放坐姿小动作 → 暂停 2-5 秒 → 环顾四周 → 循环

#### 2.2 随机行为 - RandomList



```
RandomList (m\_idleAnim = "stand")

├── m\_minClips = 2, m\_maxClips = 4

├── m\_weights = \[0.4, 0.3, 0.3]

├── m\_pauseBetweenLength = 3.0

└── m\_list:

  ├── AnimClip (m\_animName = "stand\_check\_phone")

  ├── AnimClip (m\_animName = "stand\_stretch")

  └── AnimClip (m\_animName = "stand\_yawn")
```

**行为表现**: NPC 随机播放 2-4 个动画，每个动画间隔 3 秒，权重影响选择概率

#### 2.3 多姿态切换 - Selector（核心机制）



```
Selector (m\_idleAnim = "stand")  // 基础idle状态

├── m\_minClips = 1, m\_maxClips = 1

├── m\_weights = \[0.5, 0.5]

└── m\_list:

  ├── Sequence (m\_idleAnim = "sit")  // 坐姿行为组

  │   ├── AnimClip (m\_animName = "sit\_work\_typing")

  │   └── AnimClip (m\_animName = "sit\_drink\_coffee")

  └── Sequence (m\_idleAnim = "stand")  // 站姿行为组

      ├── AnimClip (m\_animName = "stand\_talk\_phone")

      └── AnimClip (m\_animName = "stand\_walk\_around")
```

##### 行为表现与过渡机制:



1. **初始状态**: NPC 处于 "stand" 状态（Selector 的 m\_idleAnim）

2. **选择第一个行为**: Selector 随机选中 "sit" 的 Sequence

* 检测到 idle 变化: fromIdle="stand" → toIdle="sit"

* `SelectorIterator::GetNextElement()`触发:



```
DetermineTransitionAnim(customAnims, "stand", "sit", transitionAnim);

// 查找: stand\_\_2\_\_sit 或自定义过渡动画

if (!transitionAnim || !AnimExists(transitionAnim)) {

  // Selector独有的fallback机制

  m\_idleSequence->m\_idleAnim = "stand";  // 保持Selector的基础idle

  m\_idleSequence->m\_list = \[sitSequence];

  return m\_idleSequence;  // 用临时Sequence包裹，避免穿模

}
```



* 如果找到过渡动画：播放`stand__2__sit` → 进入 sit 行为组

* 如果没找到：直接混合到 sit 的第一个动画（通过临时 Sequence 保持平滑）

1. **sit 行为组循环**: 播放打字、喝咖啡等坐姿动画

2. **切换回 stand**: 下一轮随机选中 "stand" 的 Sequence

* 检测 idle 变化: fromIdle="sit" → toIdle="stand"

* 查找过渡: `sit__2__stand`

* 播放过渡 → 进入站立行为组

**核心作用**: 防止姿态突变引起的穿模（坐姿骨骼直接混合到站姿会导致角色穿过椅子）

### 阶段 3: 反应系统 (Reaction System)



```
Root Sequence

├── ... 其他Entry ...

└── ReactionSequence (⚠️必须在根层级)

  ├── m\_reactionTypes = \[TweakDB::WorkspotReactionType.Bump]

  ├── m\_forcedBlendIn = 0.2  // 快速混合进入反应

  ├── m\_mainEmotionalState = "surprise"  // 面部表情

  └── m\_list:

      ├── AnimClip (m\_animName = "reaction\_bump\_left")

      └── AnimClip (m\_animName = "reaction\_bump\_right")
```

#### 触发机制:



```
// 方式1: Bump系统自动触发

BumpAgent::TriggerNPCBump(BumpLocation::Front, BumpSide::Left, BumpIntensity::Medium);

↓

WorkspotSystem::SendReactionSignal(reactionName = "Bump")

↓

WorkspotTree::FindReactionEntry(reactionName)  // ⚠️仅搜索根层级

↓

ReactionSequenceIterator::GoTo(reactionEntryId)

↓

播放reaction\_bump\_left动画 + 应用面部表情

// 方式2: 脚本手动触发

workspotSystem.SendReactionSignal(npc, "Bump");
```

**关键注意点**: 为什么必须在根层级？`FindReactionEntry()`只遍历`m_rootEntry->m_list`，嵌套层级无法被找到。

### 阶段 4: 退出阶段 (Exit Phase)

#### 4.1 普通退出 - ExitAnim (SlowExit)



```
ExitAnim

├── m\_animName = "sit\_to\_walk"       // 退出动画

├── m\_idleAnim = "sit"               // ⚠️要求当前必须是sit状态

├── m\_movementType = Walk            // 退出后的移动类型

├── m\_isSynchronized = false

└── m\_disableRandomExit = false      // 允许随机选择退出动画
```

##### 触发方式:



```
// 方式1: 自然流程退出（如果Sequence不是无限循环）

context.m\_stepOut = false;  // 正常流程，会检查ExitAnim

// 方式2: 脚本命令退出

workspotSystem.SendJumpToAnimExit(npc, "sit\_to\_walk");  // 跳转到特定退出动画
```

##### IdleGuard 自动过渡:



```
// 当前idle = "sit\_work" (工作时的坐姿变体)

// ExitAnim要求idle = "sit" (基础坐姿)

SlowExitIterator::GoTo() 触发

↓

ProcessTransition() 检测到: fromIdle="sit\_work" → toIdle="sit"

↓

DetermineTransitionAnim("sit\_work", "sit", transitionAnim)

↓

播放 sit\_work\_\_2\_\_sit 过渡动画

↓

播放 ExitAnim的sit\_to\_walk

↓

NPC离开workspot，恢复Walk移动
```

#### 4.2 快速退出 - FastExit



```
FastExit

├── m\_animName = "stand\_fast\_exit"

├── m\_forcedBlendIn = 0.2           // 强制混合时间

└── m\_movementType = Run            // 快速移动类型
```

##### 触发方式 (⚠️仅命令触发):



```
// AI决策或战斗状态下

workspotSystem.SendCommand(npc, CMD\_FastExit);

↓

WorkspotInstance::FindBestFastExitAnim(context, gradingFunctor)

↓

根据NPC当前位置和方向，选择最合适的FastExit

↓

强制中断当前动画，0.2秒混合到fast\_exit

↓

NPC快速离开workspot
```

**关键注意点**: FastExit 不能在 Sequence 中作为顺序节点，它是外部命令触发，不参与正常的 Iterator 遍历。

## 三、完整的实际案例：餐厅 NPC



```
WorkspotTree: "restaurant\_patron.workspot"

├── m\_workspotRig = "base\characters\common\player\_base\_bodies\player\_man\_average.rig"

├── m\_customTransitionAnims:

│   ├── {m\_fromIdle="stand", m\_toIdle="sit", m\_transitionAnim="stand\_\_2\_\_sit\_restaurant"}

│   └── {m\_fromIdle="sit", m\_toIdle="stand", m\_transitionAnim="sit\_\_2\_\_stand\_restaurant"}

└── m\_rootEntry: Sequence (m\_idleAnim="stand")

  ├── m\_loopInfinitely = false

  └── m\_list:

      ├─── EntryAnim (进入餐厅)

      │    ├── m\_animName = "walk\_0\_\_to\_\_stand\_restaurant\_01"

      │    ├── m\_idleAnim = "stand"

      │    ├── m\_movementType = Walk

      │    └── 内部Sequence (m\_idleAnim="stand")

      │        └── m\_list:

      │            └─── Selector (多阶段行为) ⭐核心

      │                 ├── m\_idleAnim = "stand"

      │                 ├── m\_minClips = 4, m\_maxClips = 6

      │                 └── m\_list:

      │                     ├─── Sequence "等待入座" (m\_idleAnim="stand")

      │                     │    ├── AnimClip (m\_animName="stand\_look\_around")

      │                     │    ├── PauseClip (m\_timeMin=1, m\_timeMax=3)

      │                     │    └── AnimClip (m\_animName="stand\_check\_phone")

      │                     │

      │                     ├─── Sequence "入座就餐" (m\_idleAnim="sit") ⚠️idle变化

      │                     │    ├── AnimClip (m\_animName="sit\_look\_menu")

      │                     │    ├── AnimClip (m\_animName="sit\_eat\_food")

      │                     │    ├── AnimClip (m\_animName="sit\_drink\_water")

      │                     │    └── AnimClip (m\_animName="sit\_wipe\_mouth")

      │                     │

      │                     ├─── Sequence "起身交谈" (m\_idleAnim="stand") ⚠️idle变化

      │                     │    ├── AnimClip (m\_animName="stand\_talk\_gesture\_01")

      │                     │    └── AnimClip (m\_animName="stand\_laugh")

      │                     │

      │                     └─── ConditionalSequence "VIP行为" (m\_idleAnim="sit\_vip")

      │                          ├── m\_conditionList:

      │                          │   └── BodyTypeCondition (requireRig="vip\_character.rig")

      │                          └── m\_list:

      │                              └── AnimClip (m\_animName="sit\_vip\_exclusive\_gesture")

      │

      ├─── ReactionSequence "碰撞反应" ⚠️根层级

      │    ├── m\_reactionTypes = \[TweakDB::Bump, TweakDB::Shove]

      │    ├── m\_forcedBlendIn = 0.2

      │    ├── m\_mainEmotionalState = "annoyed"

      │    └── m\_list:

      │        ├── AnimClip (m\_animName="reaction\_bump\_front\_light")

      │        └── AnimClip (m\_animName="reaction\_shove\_back")

      │

      ├─── ExitAnim "正常离开" (m\_idleAnim="stand")

      │    ├── m\_animName = "stand\_\_to\_\_walk\_0\_exit"

      │    ├── m\_movementType = Walk

      │    └── m\_disableRandomExit = false

      │

      ├─── ExitAnim "从座位离开" (m\_idleAnim="sit")

      │    ├── m\_animName = "sit\_\_to\_\_walk\_0\_stand\_up"

      │    ├── m\_movementType = Walk

      │    └── m\_isSynchronized = false

      │

      └─── FastExit "紧急撤离"

           ├── m\_animName = "stand\_combat\_exit\_fast"

           └── m\_movementType = Run
```

## 四、运行时行为流程

### 完整的 NPC 一生（从进入到离开）



```
\[1] NPC生成，AI决定使用此workspot

  ↓

\[2] WorkspotSystem::GetClosestEntryAnim(npcPosition)

  → 找到: EntryAnim "walk\_0\_\_to\_\_stand\_restaurant\_01"

  ↓

\[3] 播放进入动画（带运动，NPC走到点位）

  → 当前idle状态变为: "stand"

  ↓

\[4] 进入EntryAnim内部的Sequence

  ↓

\[5] 进入Selector，开始随机行为

  ↓

\[6] 第1轮: 选中"等待入座"Sequence (idle="stand")

  → 无idle变化，直接播放

  → 播放: look\_around → 暂停1-3秒 → check\_phone

  ↓

\[7] 第2轮: 选中"入座就餐"Sequence (idle="sit")

  → 检测idle变化: stand → sit

  → SelectorIterator触发: DetermineTransitionAnim("stand", "sit")

  → 找到自定义动画: "stand\_\_2\_\_sit\_restaurant"

  → 播放过渡动画（NPC坐下）

  → 当前idle状态变为: "sit"

  → 播放: look\_menu → eat\_food → drink\_water → wipe\_mouth

  ↓

\[8] 第3轮: 选中"起身交谈"Sequence (idle="stand")

  → 检测idle变化: sit → stand

  → 播放过渡: "sit\_\_2\_\_stand\_restaurant" (NPC站起来)

  → 当前idle状态变为: "stand"

  → 播放: talk\_gesture → laugh

  ↓

\[9] 【突发事件】玩家撞到NPC

  → BumpComponent::TriggerNPCBump(Front, Left, Medium)

  → WorkspotSystem::SendReactionSignal("Bump")

  → FindReactionEntry("Bump") 找到ReactionSequence

  → 中断当前动画，0.2秒混合到reaction\_bump\_front\_light

  → 同时应用面部表情: annoyed

  → 反应动画播放完毕

  → 恢复到中断前的Sequence继续

  ↓

\[10] 第4-6轮: Selector继续随机播放行为...

  ↓

\[11] AI决定NPC应该离开 (或Sequence非无限循环自然结束)

  → context.m\_stepOut = false

  → ContainerIterator检测到m\_maxCount到达

  → 开始查找ExitAnim

  ↓

\[12] 当前idle="sit"，找到匹配的ExitAnim

  → ExitAnim要求idle="sit"，匹配成功

  → 播放: "sit\_\_to\_\_walk\_0\_stand\_up"

  → NPC站起来并开始行走

  ↓

\[13] NPC离开workspot，恢复AI自由移动
```

### 战斗状态的快速退出



```
\[场景] NPC正在餐厅座位上(idle="sit")，突然警报响起

\[1] 战斗AI接管: SendCommand(CMD\_FastExit)

  ↓

\[2] WorkspotInstance::FindBestFastExitAnim()

  → 评估所有FastExit的transform

  → 选择最近的: "stand\_combat\_exit\_fast"

  ↓

\[3] 强制中断当前动画

  → 忽略当前idle="sit"的要求

  → 0.2秒强制混合到FastExit动画

  ↓

\[4] NPC以Run速度快速离开workspot

  → 直接进入战斗移动状态
```

## 五、关键设计原则总结

### 1. Idle 驱动的状态机



* 每个容器必须有`m_idleAnim`

* IdleGuard 自动检测 idle 变化并插入过渡

* 过渡动画命名规则: `fromIdle__2__toIdle`

### 2. 层级职责分明

**根层级 (Root)**:

├── EntryAnim 组      → 处理外部进入

├── 主行为组         → Sequence/Selector/RandomList

├── ReactionSequence → 反应系统（必须根层级）

└── ExitAnim 组       → 处理各种退出情况

### 3. Selector 解决多姿态问题



* 问题：不同骨骼姿态直接混合会穿模

* 方案：通过 idle 状态 + 过渡动画实现平滑切换

* Fallback: 如果过渡不存在，用临时 Sequence 保持基础 idle

### 4. 命令与流程分离



* 流程节点: EntryAnim, Sequence 内的 AnimClip, ExitAnim

* 命令触发: FastExit, ReactionSequence, TagNode 跳转

* 不要混淆: FastExit 不能放在 Sequence 中作为顺序播放

### 5. Flag 驱动的 Iterator 过滤



```
context.m\_requestedFlags = IEntry::SlowEnter;

→ Iterator只激活匹配的Entry

context.m\_requestedFlags = IEntry::Animation;

→ 跳过Entry/Exit，只播放动画节点
```

## 六、NPC 在点位 "是什么样" 的完整答案

一个 WorkspotTree 定义的 NPC 在点位是:



1. 一个有进入仪式的 Actor (EntryAnim 定义如何到达点位)

2. 一个基于 idle 状态机的行为循环 (Sequence/Selector 定义循环做什么)

3. 一个能响应外部事件的反应系统 (ReactionSequence 定义如何应对 bump/shove)

4. 一个有多种退出策略的离开机制 (ExitAnim/FastExit 定义如何离开)

5. 一个自动处理姿态过渡的动画系统 (IdleGuard 确保动画平滑无穿模)

**实际表现**: 一个 NPC 走到餐桌旁坐下，随机进行等待、就餐、交谈等行为，如果被碰撞会播放反应动画，最后根据情况站起来正常离开或紧急撤离。整个过程中所有姿态变化 (站↔坐) 都有平滑的过渡动画，不会出现穿模或突变。

这就是 WorkspotTree Entry 组织系统的完整架构。

> （注：文档部分内容可能由 AI 生成）