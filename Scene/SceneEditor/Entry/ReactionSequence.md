# ReactionSequence：反应序列系统详解
ReactionSequence 是 Workspot 系统中用于响应游戏事件（碰撞、威胁、情绪刺激等）的特殊容器节点。它允许 NPC 在 Workspot 中对外部刺激做出即时反应。

## 1. 支持的反应类型
从 tweakDBWorkspotReactionType.h 可以看到，系统定义了 7 种反应类型：
```c++
enum class WorkspotReactionType : Uint32
{
    Anger = 0u,          // 愤怒反应
    BumpLeftBack = 1u,   // 左后方碰撞
    BumpLeftFront = 2u,  // 左前方碰撞
    BumpRightBack = 3u,  // 右后方碰撞
    BumpRightFront = 4u, // 右前方碰撞
    Curious = 5u,        // 好奇反应
    Fear = 6u,           // 恐惧反应

    Count = 7u,
    Invalid = 8u,
};
```

### 反应类型详解
| 反应类型       | 英文名         | 触发场景             | 常见用途                                  |
|----------------|----------------|----------------------|-------------------------------------------|
| 愤怒           | Anger          | NPC 被激怒、受到威胁 | 坐着的 NPC 被玩家多次打扰后站起来愤怒反应 |
| 左后方碰撞     | BumpLeftBack   | NPC 左后方被撞击     | 坐在椅子上的 NPC 被从左后方撞到，转头查看 |
| 左前方碰撞     | BumpLeftFront  | NPC 左前方被撞击     | 用餐的 NPC 被从左前方碰撞，放下餐具       |
| 右后方碰撞     | BumpRightBack  | NPC 右后方被撞击     | 阅读的 NPC 被从右后方撞到，转身查看       |
| 右前方碰撞     | BumpRightFront | NPC 右前方被撞击     | 喝咖啡的 NPC 被从右前方撞到，躲避动作     |
| 好奇           | Curious        | NPC 注意到异常情况   | 工作的 NPC 听到枪声，抬头观察             |
| 恐惧           | Fear           | NPC 感到害怕、惊慌   | 平民 NPC 看到枪战，惊慌失措               |

## 2. ReactionSequence 的结构
```c++
class ReactionSequence : public Sequence
{
    // === 反应类型配置 ===
    red::DynArray<game::data::RecordID> m_reactionTypes;  // 支持的反应类型列表

    // === 动画混合配置 ===
    Float m_forcedBlendIn = 0.2f;  // 强制混入时间（快速切换到反应动画）

    // === 面部表情系统 ===
    CName m_mainEmotionalState;      // 主要情绪状态（如 "fear", "anger"）
    CName m_emotionalExpression;     // 情绪表达（如 "afraid", "angry"）
    Float m_facialKeyWeight = 1.0f;  // 面部表情权重

    // 自动生成的面部动画名称
    CName m_facialIdleMaleAnimation;        // 男性面部待机动画
    CName m_facialIdleKey_MaleAnimation;    // 男性面部关键帧动画
    CName m_facialIdleFemaleAnimation;      // 女性面部待机动画
    CName m_facialIdleKey_FemaleAnimation;  // 女性面部关键帧动画
};
```

### 关键属性说明
#### m_reactionTypes（反应类型列表）
- 类型：red::DynArray<game::data::RecordID>
- 用途：定义此 ReactionSequence 响应哪些反应类型
- 配置方式：在编辑器中通过 TweakDB 下拉菜单选择
- 示例：
  ```
  m_reactionTypes: array<2>
    [0]: WorkspotReactionType.Fear
    [1]: WorkspotReactionType.BumpRightBack
  ```
  这表示该反应序列会响应"恐惧"和"右后方碰撞"两种事件。

#### m_forcedBlendIn（强制混入时间）
- 默认值：0.2 秒
- 用途：当反应被触发时，快速从当前动画切换到反应动画的混合时间
- 效果：值越小，反应越突然；值越大，过渡越平滑
- 推荐值：
  - 惊吓反应：0.1 秒（快速）
  - 恐惧反应：0.2 秒（中速）
  - 好奇反应：0.3 秒（缓慢）

#### 面部表情系统
ReactionSequence 支持自动配置面部表情，增强情绪表达：
```c++
// 设置主要情绪状态为 "fear"（恐惧）
m_mainEmotionalState = "fear"

// 自动生成面部动画名称：
m_facialIdleMaleAnimation = "idle__fear__male"
m_facialIdleFemaleAnimation = "idle__fear__female"
```

## 3. 触发机制
### 3.1 触发流程
游戏事件发生
↓
触发 BumpEvent / ReactionEvent
↓
WorkspotSystem::GetReactionEntry(actorId, reactionName)
↓
WorkspotInstance::FindReactionEntry(reactionName)
↓
WorkspotTree::FindReactionEntry(reactionName)
↓
遍历 m_rootEntry 的直接子节点
↓
找到匹配的 ReactionSequence
↓
检查 ReactionSequence::ContainsReaction(reactionName)
↓
返回 ReactionSequence 的 EntryId
↓
中断当前动画，播放反应序列

### 3.2 查找逻辑（源代码）
```c++
// workspotResource.cpp: 1499-1512
work::WorkEntryId WorkspotTree::FindReactionEntry(CName reactionName) const
{
    IContainerEntry* cont = Cast<IContainerEntry>(m_rootEntry.Get());

    // 只在 m_rootEntry 的直接子节点中查找
    for (THandle<IEntry>& record : cont->m_list)
    {
        if (THandle<ReactionSequence> reaction = Cast<ReactionSequence>(record))
        {
            if (reaction->ContainsReaction(reactionName))
            {
                return reaction->m_id;  // 返回找到的反应序列ID
            }
        }
    }
    return work::WorkEntryId::invalid;
}
```
**重要**：ReactionSequence 必须是 m_rootEntry 的直接子节点才能被检测到！

### 3.3 反应匹配逻辑
```c++
// workspotTreeItems.cpp: 1383-1398
Bool ReactionSequence::ContainsReaction(const CName reactionName) const
{
    // 遍历配置的反应类型列表
    for (const auto record : m_reactionTypes)
    {
        if (THandle<game::data::WorkspotReactionType_Record> reaction =
            game::data::GetRecordHandle<game::data::WorkspotReactionType_Record>(record))
        {
            if (reaction->enumName == reactionName)
            {
                return true;  // 找到匹配的反应类型
            }
        }
    }
    return false;
}
```

## 4. 典型配置示例
### 示例 1：餐厅碰撞反应
```
RootEntry (Sequence)
├── ReactionSequence: "bump_reactions"
│   ├── m_reactionTypes:
│   │   ├── [0] WorkspotReactionType.BumpLeftBack
│   │   ├── [1] WorkspotReactionType.BumpRightBack
│   │   ├── [2] WorkspotReactionType.BumpLeftFront
│   │   └── [3] WorkspotReactionType.BumpRightFront
│   ├── m_forcedBlendIn: 0.15
│   └── m_list:
│       ├── [0] AnimClip: "sit_react_bump_look_around"
│       └── [1] AnimClip: "sit_back_to_idle"
│
├── Sequence: "eating_loop" (主循环)
│   └── m_list:
│       ├── AnimClip: "sit_eat_fork"
│       └── AnimClip: "sit_drink_cup"
│
└── ExitAnim: "stand_up"
```
**工作原理**：
1. NPC 正常执行 "eating_loop" 吃饭动画
2. 玩家从左后方撞到 NPC
3. 触发 BumpLeftBack 事件
4. 查找到 "bump_reactions" ReactionSequence
5. 中断吃饭动画，快速混入（0.15秒）到 "sit_react_bump_look_around"
6. 播放反应序列后，返回待机状态

### 示例 2：恐惧反应（带面部表情）
```
RootEntry (Sequence)
├── ReactionSequence: "fear_reaction"
│   ├── m_reactionTypes:
│   │   └── [0] WorkspotReactionType.Fear
│   ├── m_forcedBlendIn: 0.1  // 惊吓反应，快速切换
│   ├── m_mainEmotionalState: "fear"
│   ├── m_emotionalExpression: "afraid"
│   ├── m_facialKeyWeight: 1.0
│   └── m_list:
│       ├── [0] AnimClip: "sit_scared_hands_up"
│       ├── [1] AnimClip: "sit_scared_look_around"
│       └── [2] FastExit: "stand_panic_run"
│
└── Sequence: "reading_loop"
    └── m_list:
        └── AnimClip: "sit_read_book"
```
**触发场景**：
- NPC 正在看书
- 附近发生枪战
- 游戏逻辑发送 Fear 反应事件
- NPC 立即切换到恐惧反应序列
- 播放惊吓、环顾四周动画
- 最后快速退出 Workspot 并逃跑

### 示例 3：多层反应系统
```
RootEntry (Sequence)
├── ReactionSequence: "light_bump"
│   ├── m_reactionTypes:
│   │   ├── [0] WorkspotReactionType.BumpLeftFront
│   │   └── [1] WorkspotReactionType.BumpRightFront
│   ├── m_forcedBlendIn: 0.2
│   └── m_list:
│       └── [0] AnimClip: "sit_minor_annoyance"  // 轻微不满
│
├── ReactionSequence: "heavy_bump"
│   ├── m_reactionTypes:
│   │   ├── [0] WorkspotReactionType.BumpLeftBack
│   │   └── [1] WorkspotReactionType.BumpRightBack
│   ├── m_forcedBlendIn: 0.1
│   └── m_list:
│       ├── [0] AnimClip: "sit_startled_turn"  // 惊吓转身
│       └── [1] AnimClip: "sit_angry_gesture"  // 愤怒手势
│
├── ReactionSequence: "anger_escalation"
│   ├── m_reactionTypes:
│   │   └── [0] WorkspotReactionType.Anger
│   ├── m_forcedBlendIn: 0.15
│   └── m_list:
│       ├── [0] AnimClip: "sit_stand_up_angry"
│       └── [2] ExitAnim: "stand_confront"  // 站起来对峙
│
└── Sequence: "working_loop"
    └── ...
```
**场景设计**：
- 前方轻微碰撞：NPC 稍微不满，继续工作
- 后方重度碰撞：NPC 惊吓转身，显示愤怒
- 持续骚扰：游戏逻辑发送 Anger 事件，NPC 站起来对峙

## 5. 实际应用场景
### 场景 1：餐厅服务生反应
**需求**：服务生在工作时对玩家碰撞做出自然反应
```
m_reactionTypes:
  - BumpLeftBack
  - BumpRightBack
  - BumpLeftFront
  - BumpRightFront
```
**反应动画**：
- "stand_work_bump_slight_turn"  // 轻微转身
- "stand_work_stabilize"         // 稳定身体
- 返回工作待机

### 场景 2：图书馆读者反应
**需求**：读者在安静环境被打扰时的分层反应
- ReactionSequence 1: "curious_reaction"
  ```
  m_reactionTypes: [Curious]
  反应: "sit_read_look_up" → "sit_read_continue"
  ```
- ReactionSequence 2: "fear_reaction"
  ```
  m_reactionTypes: [Fear]
  反应: "sit_scared_drop_book" → FastExit
  ```

### 场景 3：酒吧醉汉反应
**需求**：醉汉对碰撞反应迟钝，但会逐渐愤怒
- ReactionSequence: "drunk_bump"
  ```
  m_reactionTypes: [BumpLeftBack, BumpRightBack]
  m_forcedBlendIn: 0.5  // 反应慢
  反应: "sit_drunk_sway" → "sit_drunk_mumble"
  ```
- ReactionSequence: "drunk_anger"
  ```
  m_reactionTypes: [Anger]
  m_forcedBlendIn: 0.3
  反应: "sit_drunk_try_stand" → ExitAnim: "stand_drunk_aggressive"
  ```

## 6. 设计要点
### 6.1 ReactionSequence 必须是根节点的直接子节点
❌ 错误：
```
RootEntry (Sequence)
└── Sequence: "working"
    ├── ReactionSequence: "bump"  ← 不会被检测到！
    └── AnimClip: "work_idle"
```
✅ 正确：
```
RootEntry (Sequence)
├── ReactionSequence: "bump"  ← 正确位置
└── Sequence: "working"
    └── AnimClip: "work_idle"
```

### 6.2 反应类型不要重复
❌ 错误：
```
ReactionSequence 1:
  m_reactionTypes: [Fear, Curious]

ReactionSequence 2:
  m_reactionTypes: [Fear, Anger]  ← Fear 重复！
```
系统会返回第一个匹配的 ReactionSequence，导致第二个永远不会触发。

### 6.3 使用 m_forcedBlendIn 控制反应速度
- 惊吓反应：0.05 - 0.1 秒  // 非常快
- 恐惧反应：0.1 - 0.2 秒  // 快速
- 愤怒反应：0.15 - 0.25 秒  // 中速
- 好奇反应：0.2 - 0.3 秒  // 缓慢

### 6.4 面部表情自动配置
设置 m_mainEmotionalState 后，系统会自动生成面部动画名称：
```
m_mainEmotionalState = "fear"
↓ 自动生成
m_facialIdleMaleAnimation = "idle__fear__male"
m_facialIdleFemaleAnimation = "idle__fear__female"
```

## 7. 调试技巧
### 检查 ReactionSequence 是否被找到
在 C++ 代码中可以看到查找逻辑：
```c++
work::WorkEntryId reactionId = workspotTree->FindReactionEntry(RED_NAME("Fear"));
if (reactionId == work::WorkEntryId::invalid)
{
    // ReactionSequence 未找到，检查：
    // 1. 是否在 m_rootEntry 的直接子节点中
    // 2. m_reactionTypes 是否包含 "Fear"
}
```

### 常见问题
1. 反应不触发：
   - 检查 ReactionSequence 是否在根节点的直接子节点中
   - 检查 m_reactionTypes 是否包含正确的反应类型
2. 反应太慢/太快：
   - 调整 m_forcedBlendIn 值
3. 面部表情不显示：
   - 检查 Entity 的 AnimSet 是否包含面部动画
   - 确认 m_facialKeyWeight > 0

## 总结
### ReactionSequence 的核心特性
1. 支持 7 种反应类型：Anger, BumpLeftBack, BumpLeftFront, BumpRightBack, BumpRightFront, Curious, Fear
2. 必须是根节点的直接子节点：否则无法被检测到
3. 快速切换机制：通过 m_forcedBlendIn 实现即时反应
4. 面部表情支持：自动配置情绪相关的面部动画
5. 事件驱动：通过 BumpEvent, ReactionEvent 等游戏事件触发

### 设计建议
- 为同一个 Workspot 配置多个 ReactionSequence，覆盖不同反应类型
- 使用较短的 m_forcedBlendIn（0.1-0.2秒）确保反应足够快
- 利用面部表情系统增强情绪表达
- 反应序列结束后应返回适当的待机状态，或使用 ExitAnim 退出 Workspot

这套系统让 NPC 在 Workspot 中能够对玩家的行为和游戏事件做出自然、即时的反应，极大地提升了游戏的沉浸感和交互性。

我可以帮你把这份md文档里的**触发流程、配置结构**转换成**PlantUML可视化流程图**，方便你整合到技术文档中，需要吗？

WraithMale_ow_city
<no_auto_transition>