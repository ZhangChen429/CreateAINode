我直接帮你把这份 **2077 Workspot Entry 执行条件完整梳理** 整理成**干净、标准、可直接保存的 Markdown 文档**，结构不变、内容完整、格式清爽：

---

# 2077 Workspot Entry 执行条件完整梳理

## 核心架构：两层筛选机制

### 第1层：静态筛选（编辑时定义）
- ConditionalSequence 的条件检查
- WorkspotTree 的白名单/黑名单

### 第2层：运行时选择（执行时计算）
- EntryAnim：位置+角度最优匹配
- Selector/RandomList：权重+概率+历史
- IdleGuard：自动姿态过渡
- ReactionSequence：事件触发

---

## 1️⃣ EntryAnim - 位置和角度匹配系统

### 核心代码
```cpp
// workspotResource.cpp:954
work::EntryPoint WorkspotTree::GetClosestEntryAnim(
    const Transform& posLS,  // 角色当前位置（Workspot局部空间）
    const AnimSearchContext& context
) const {
    work::EntryPoint res;
    Float currMaxDistance = std::numeric_limits<Float>::max();

    // 遍历所有 EntryAnim
    helper::ForEachNodeConditional<EntryAnim>( m_rootEntry, cont,
        [&]( EntryAnim* entryAnim ) {
            helper::EvaluateDistance<EntryAnim, true>(
                entryAnim, context, posLS, res, currMaxDistance, this
            );
        }
    );

    return res;  // 返回距离最近的 EntryAnim
}
```

### 距离计算公式（核心算法）
```cpp
// workspotResource.cpp:928
void EvaluateDistance( const EntryAnim* entryAnim, ... ) {
    // 1. 获取动画的运动轨迹变换
    Transform movementTransform;
    FindAnimTransform( entryAnim->m_animName, movementTransform, context );

    // 2. 计算参考变换（进入点是动画起点的逆）
    Transform referenceTransform = movementTransform.GetInverse();

    // 3. 计算加权距离：位置距离平方 + 角度差（弧度）
    Float distance =
        (referenceTransform.GetPosition() - posLS.GetPosition()).SquareMag3() +
        (referenceTransform.GetOrientation() * posLS.GetOrientation().Inverse()).GetAngle();

    // 4. 选择距离最小的
    if ( distance < currMaxDistance && 满足其他条件 ) {
        res = entryAnim;
        currMaxDistance = distance;
    }
}
```

### 执行条件完整列表

| 条件类型     | 检查内容                                           | 代码位置           | 说明                                       |
| ------------ | -------------------------------------------------- | ------------------ | ------------------------------------------ |
| 位置距离     | `(pos1 - pos2).SquareMag3()`                       | EvaluateDistance   | NPC 与 Entry 点的 3D 距离平方              |
| 角度差       | `(quat1 * quat2.Inverse()).GetAngle()`            | EvaluateDistance   | NPC 朝向与 Entry 点朝向的夹角（弧度）|
| MovementType | `entryAnim->m_movementType == context.m_movementType` | EvaluateDistance:943 | Walk/Stand/Sprint 等移动类型匹配           |
| SlotName     | `entryAnim->m_slotName == context.m_slotName`     | EvaluateDistance:931 | 同步槽位名称（多人同步用）|
| 动画存在性   | `FindAnimTransform(...) 返回 true`                | EvaluateDistance:932 | 动画在 AnimSet 中必须存在                  |
| 条件序列     | `ConditionalSequence::CheckConditions()`           | ForEachNodeConditional:584 | 父容器的条件必须满足 |

### 实际使用场景
```cpp
// AI 选择 Entry 的代码
// aiSelectWorkspotEntryTask.cpp:139
const Transform deltaInWorkspotSpace =
    workspotTransform.TransformInv( currActorTransform );

work::EntryPoint entryPoint =
    workspotData->m_parameters.m_workspot.GetClosestEntryAnim(
        deltaInWorkspotSpace,  // NPC 在 Workspot 局部空间的位置
        searchCont             // 包含 MovementType 等上下文
    );

if ( !entryPoint.m_entryId ) {
    // 如果 Walk 模式找不到，尝试 Stand 模式
    searchCont.m_movementType = move::MovementType::Stand;
    entryPoint = workspotData->m_parameters.m_workspot.GetClosestEntryAnim(
        deltaInWorkspotSpace, searchCont
    );
}
```

### 关键发现
Entry 点的位置是从动画数据中提取的：
```cpp
// 每个 EntryAnim 的 m_animName 动画会被采样
// 动画的最后一帧（或第一帧）的根骨骼变换 = Entry 点位置
Transform movementTransform;
FindAnimTransform( "walk_to_chair_front", movementTransform, context );

// walk_to_chair_front 动画结束位置 = 椅子前方进入点
// walk_to_chair_left 动画结束位置 = 椅子左侧进入点
```

---

## 2️⃣ Selector / RandomList - 加权随机选择系统

### 核心代码
```cpp
// workspotTreeItems.cpp:1072
class SelectorIterator {
    virtual Float GetFilteredWeights(
        const RandomList* pointedEntry,
        const EntryIterationContext& context,
        red::DynArray<Float>& outWeights
    ) const {
        Float totalWeight = 0.f;

        for ( Uint32 i = 0; i < m_contEntry->m_list.Size(); ++i ) {
            const THandle<IEntry>& entry = m_contEntry->m_list[i];

            // 1. 检查历史记录（避免重复播放）
            if ( m_playbackHistory.Exist( entry->m_id ) )
                continue;

            // 2. 检查 ConditionalSequence 的条件
            if ( ConditionalSequence* cs = Cast<ConditionalSequence>( entry.Get() ) ) {
                if ( !cs->CheckConditions( context.m_condContext ) )
                    continue;
            }

            // 3. 检查 Category 概率（时间段概率控制）
            Sequence* seq = Cast<Sequence>( entry.Get() );
            const Float entryCategoryWeight =
                context.m_categoryProbabilities->GetProbabilityForCategory( seq->m_category );

            if ( entryCategoryWeight > 0.f ) {
                outWeights[i] = entryCategoryWeight * m_contEntry->m_weights[i];
                totalWeight += outWeights[i];
            }
        }

        return totalWeight;
    }
};
```

### 执行条件完整列表

| 条件类型      | 检查内容                              | 配置位置                         | 说明                                   |
| ------------- | ------------------------------------- | -------------------------------- | -------------------------------------- |
| 权重          | `m_weights[i]`                        | RandomList.m_weights             | 每个子 Entry 的静态权重                |
| Category 概率  | `GetProbabilityForCategory()`         | Sequence.m_category              | 时间段动态概率（早餐/午餐/晚餐）|
| 历史记录      | `m_playbackHistory.Exist()`           | RandomList.m_dontRepeatLastAnims | 避免重复最近 N 个动画                  |
| 条件检查      | `CheckConditions()`                   | ConditionalSequence.m_conditionList | Player/Bodytype/Tag 等条件         |
| 最小/最大数量 | `m_minClips, m_maxClips`              | RandomList                       | 随机播放 N-M 个动画                    |
| 暂停时间      | `m_pauseBetweenLength`                | RandomList                       | 动画间隔时间                           |

### 实际案例
```cpp
// 餐厅 NPC 的时间段行为
Selector {
    m_idleAnim: "stand",
    m_weights: [0.5, 0.3, 0.2],  // 静态权重
    m_list: [
        // 早餐行为（7-9am）
        Sequence {
            m_category: "breakfast",
            m_list: [
                AnimClip { name: "eat_toast" },
                AnimClip { name: "drink_coffee" }
            ]
        },

        // 午餐行为（12-2pm）
        Sequence {
            m_category: "lunch",
            m_list: [
                AnimClip { name: "eat_sandwich" },
                AnimClip { name: "drink_soda" }
            ]
        },

        // 休息行为（任何时间）
        Sequence {
            m_category: "any",
            m_list: [
                AnimClip { name: "check_phone" }
            ]
        }
    ]
}
```

**运行时概率计算示例：**
- 早上 8 点：
  - breakfast 概率 = 1.0
  - lunch 概率 = 0.0
  - any 概率 = 1.0
- 最终权重：
  - breakfast = 0.5 * 1.0 = 0.5
  - lunch = 0.3 * 0.0 = 0.0
  - any = 0.2 * 1.0 = 0.2
- 选中概率：breakfast=71.4%, any=28.6%

---

## 3️⃣ ConditionalSequence - 条件组合系统

### 可用条件类型
```cpp
// workspotConditions.h

1. IsPlayerCondition
   // 检查是否是玩家
   context.m_parentEntity->IsPlayer()

2. BodytypeCondition
   // 检查骨骼类型（体型）
   context.m_bodyType == m_rig.GetPath()

3. ActorTagCondition
   // 检查 Actor 标签
   context.m_parentEntity->HasTag( m_tag )

4. CoverTypeCondition
   // 检查掩体类型
   context.m_coverType == (m_isHighCover ? HighCover : LowCover)

5. TimeOfDayCondition
   // 检查游戏时间
   currentTime >= m_activeAfter && currentTime <= m_activeUntil

6. ScriptedCondition
   // 自定义脚本条件
   m_script->Check( context )
```

### 条件组合逻辑
```cpp
// workspotTreeItems.cpp:1197
const Bool ConditionalSequence::CheckConditions(
    const CheckConditionContext& context
) {
    for ( const THandle<IWorkspotCondition> cond : m_conditionList ) {
        const Bool result = cond->Check( context );

        if ( m_multipleConditionOperator == LogicalOperation::OR ) {
            if ( result ) return true;  // 任意一个满足即可
        }
        else if ( m_multipleConditionOperator == LogicalOperation::AND ) {
            if ( !result ) return false;  // 全部满足才行
        }
    }

    return m_multipleConditionOperator == LogicalOperation::AND;
}
```

### 实际案例
```cpp
// 只允许玩家且是女性体型的 Workspot
ConditionalSequence {
    m_multipleConditionOperator: "AND",
    m_conditionList: [
        IsPlayerCondition { },
        BodytypeCondition {
            m_rig: "base\\characters\\common\\player_base_bodies\\player_female_average.rig"
        }
    ],
    m_list: [
        AnimClip { name: "sit_female_specific_anim" }
    ]
}

// 早上或晚上的 NPC 行为
ConditionalSequence {
    m_multipleConditionOperator: "OR",
    m_conditionList: [
        TimeOfDayCondition {
            m_activeAfter: "06:00",
            m_activeUntil: "09:00"
        },
        TimeOfDayCondition {
            m_activeAfter: "18:00",
            m_activeUntil: "22:00"
        }
    ],
    m_list: [
        AnimClip { name: "morning_or_evening_behavior" }
    ]
}
```

---

## 4️⃣ IdleGuard - 自动姿态过渡系统

### 核心机制
```cpp
// workspotTreeItems.cpp:272
template< class EntryClass, class BaseIterator >
class IdleGuard : public BaseIterator {
    virtual void Next( const EntryIterationContext& context ) override {
        // 1. 检测 idle 变化
        CName fromIdle = context.m_currentIdleAnim;  // 当前姿态
        CName toIdle = GetTargetIdle();              // 目标姿态

        // 2. 如果姿态不同，需要过渡
        if ( fromIdle && toIdle && fromIdle != toIdle ) {
            // 查找过渡动画
            DetermineTransitionAnim(
                context.m_customTransitionAnims,
                fromIdle,
                toIdle,
                m_transiionAnimName  // 输出："stand__2__sit"
            );

            // 3. 检查过渡动画是否存在
            if ( context.m_animExistFunctor( m_transiionAnimName ) ) {
                m_state = State::PerformTransition;  // 插入过渡
            } else {
                // 降级方案：保持基础 idle，让动画系统平滑混合
                m_state = State::UseBaseIdle;
            }
        }
    }
};
```

### 过渡动画命名规则
```cpp
// workspotResource.cpp:300
CName GenerateTransitionAnimName( CName fromIdle, CName toIdle ) {
    // 规则：fromIdle__2__toIdle
    return String::Printf( "%s__2__%s", fromIdle.AsChar(), toIdle.AsChar() );
}

// 示例：
// stand → sit  =  "stand__2__sit"
// sit → crouch  =  "sit__2__crouch"
// crouch → stand  =  "crouch__2__stand"
```

### 执行条件

| 条件类型       | 检查内容                               | 触发时机                           |
| -------------- | -------------------------------------- | ---------------------------------- |
| Idle 变化      | `fromIdle != toIdle`                   | 切换到新的 Sequence/Selector 分支  |
| 过渡动画存在    | `m_animExistFunctor(transitionAnim)`   | IdleGuard.Next()                   |
| 自定义过渡优先  | 查找 `m_customTransitionAnims`         | 优先使用配置的自定义过渡           |
| 降级策略       | Selector 的 fallback 机制              | 过渡动画缺失时                     |

---

## 5️⃣ ReactionSequence - 事件触发系统

### 触发条件
```cpp
// workspotTreeItems.h:361
Bool ContainsReaction( const CName reactionName ) const {
    return std::find_if( m_reactionTypes.Begin(), m_reactionTypes.End(),
        [reactionName]( const game::data::RecordID& id ) {
            return id.GetRecordName() == reactionName;
        }
    ) != m_reactionTypes.End();
}
```

### 常见反应类型（TweakDB 定义）
- WorkspotReactionType.Bump         // 被撞
- WorkspotReactionType.Shove        // 被推
- WorkspotReactionType.PlayerApproach // 玩家靠近
- WorkspotReactionType.WeaponDrawn   // 拔枪
- WorkspotReactionType.Explosion     // 爆炸
- WorkspotReactionType.Combat        // 战斗

### 执行流程
1. 外部事件触发
   ```cpp
   GameSystem::TriggerWorkspotReaction( npcId, "Bump" );
   ```
2. Workspot 系统检查是否有匹配的 ReactionSequence
3. 中断当前动画
4. 播放反应动画
5. 反应动画结束后恢复原流程

---

## 6️⃣ 综合执行示例

### 场景：餐厅 NPC 完整流程
```cpp
WorkspotTree {
    m_rootEntry: Sequence {
        m_list: [
            // 1. EntryAnim 选择（位置+角度匹配）
            EntryAnim {
                name: "walk_to_chair_front",
                m_movementType: Walk,
                m_idleAnim: "stand"
            },
            EntryAnim {
                name: "walk_to_chair_left",
                m_movementType: Walk,
                m_idleAnim: "stand"
            },
            EntryAnim {
                name: "walk_to_chair_right",
                m_movementType: Walk,
                m_idleAnim: "stand"
            },

            // 2. Selector 加权随机选择
            Selector {
                m_idleAnim: "stand",
                m_weights: [0.6, 0.4],
                m_list: [
                    // 2.1 站立等待
                    Sequence {
                        m_idleAnim: "stand",
                        m_category: "waiting",
                        m_list: [
                            AnimClip { name: "stand_look_around" },
                            AnimClip { name: "stand_check_phone" }
                        ]
                    },

                    // 2.2 坐下就餐（条件：非玩家）
                    ConditionalSequence {
                        m_multipleConditionOperator: "AND",
                        m_conditionList: [
                            IsPlayerCondition {
                                m_expectedResult: Deny
                            }
                        ],
                        m_idleAnim: "sit",
                        m_list: [
                            AnimClip { name: "sit_eat_food" },
                            AnimClip { name: "sit_drink_water" }
                        ]
                    }
                ]
            },

            // 3. ReactionSequence
            ReactionSequence {
                m_reactionTypes: ["Bump", "Shove"],
                m_forcedBlendIn: 0.2,
                m_list: [
                    AnimClip { name: "sit_react_to_bump" }
                ]
            },

            // 4. ExitAnim
            ExitAnim {
                name: "stand_and_leave",
                m_idleAnim: "sit",
                m_movementType: Walk
            }
        ]
    }
}
```

### 执行时间线
- **T=0s**: AI 选择距离最近的 EntryAnim
- **T=2s**: 播放进入动画，姿态为 stand
- **T=4s**: 进入 Selector，按权重+时间段随机选中“坐下就餐”
- **T=4.1s**: IdleGuard 检测姿态变化，自动插入 `stand__2__sit`
- **T=5.6s**: 播放就餐动画
- **T=8s**: 玩家撞击触发 Reaction，中断并播放反应动画
- **T=10s**: 恢复原流程

---

## 🎯 关键设计模式总结

1. **策略模式**
   - SequenceIterator      → 顺序播放
   - RandomListIterator    → 加权随机
   - SelectorIterator      → 姿态选择 + 自动过渡
   - ReactionIterator      → 事件中断

2. **责任链模式**
   ForEachNodeConditional → ConditionalSequence → Bodytype → Tag → Category → Weights

3. **模板方法模式**
   ```
   BaseIterator::Next()
       ↓ 子类实现
   IdleGuard::Next()
       ↓ 检测 idle 变化
       ↓ 插入过渡动画
   ```

4. **工厂模式**
   ```cpp
   IEntry::CreateIterator( context ) {
       if ( type == Sequence ) return new SequenceIterator();
       if ( type == Selector ) return new SelectorIterator();
       ...
   }
   ```

---

## 📊 性能优化要点

1. **距离计算缓存**
   - EntryAnim 的 Transform 只在加载时采样一次
   - 运行时只做简单向量减法和点积
2. **条件惰性求值**
   - ForEachNodeConditional 按需遍历
   - ConditionalSequence 支持短路求值
3. **权重预计算**
   - Category 概率在创建时计算
   - 运行时仅乘法 + 归一化
4. **历史记录限制**
   - MAX_REPEAT_HISTORY = 5，控制内存开销

---

如果你需要，我还可以帮你：
- 再压缩成**一页速查版**
- 转成 **UE 可直接对照实现的架构图**
- 输出 **PDF / 导出文件** 格式