# Cyberpunk 2077 - PauseNode、Token机制与Condition系统架构总览

## 文档说明
本文档详细记录了Cyberpunk 2077游戏引擎中PauseNode节点的Token机制和Condition条件等待系统的完整架构。

---

## 一、系统架构概览

### 1.1 核心组件关系图

```
┌─────────────────────────────────────────────────────────────┐
│                  Quest System (任务系统)                      │
│                                                               │
│  ┌─────────────────┐      ┌──────────────────┐              │
│  │ PauseNode       │◄────►│ Condition System │              │
│  │ (暂停节点)       │      │ (条件系统)        │              │
│  └────────┬────────┘      └──────────────────┘              │
│           │                                                   │
│           │ Token 传递                                        │
│           ▼                                                   │
│  ┌─────────────────┐                                         │
│  │ Token System    │                                         │
│  │ (Token机制)     │                                         │
│  └────────┬────────┘                                         │
│           │                                                   │
│           │ 状态控制                                          │
│           ▼                                                   │
│  ┌─────────────────┐                                         │
│  │ SectionNode     │                                         │
│  │ (场景节点)       │                                         │
│  └─────────────────┘                                         │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 工作流程

```
1. Quest信号流到达PauseNode
   ↓
2. PauseNode检查Condition (IsFulfilled)
   ↓
   条件未满足分支:
   ├─ 注册EventListener监听条件变化
   ├─ 返回 NodeResult::StayInNode
   └─ Token保持在PauseNode，不向下传递
   ↓
3. 系统持续检查Condition
   ↓
   条件满足时:
   ├─ 触发EventListener回调
   ├─ 调用PauseNode::OnExecute
   └─ 返回 NodeResult::LeaveNode
   ↓
4. 激活outputSockets.AddOutput("Out")
   ↓
5. Token继续传递到下一个节点 (SectionNode)
   ↓
6. SectionNode接收Token:
   ├─ 检查TokensState (activeStartToken, activePauseToken等)
   ├─ 根据Token状态决定操作:
   │  ├─ startProcessing
   │  ├─ continueProcessing
   │  └─ sustainProcessing
   └─ 处理PlaySpeed (Pause / Normal / Fast等)
```

---

## 二、文件结构索引

### 2.1 核心源文件路径

| 组件 | 头文件 | 实现文件 |
|------|--------|----------|
| **PauseNode** | `common/gameQuest/include/pauseConditionNode.h` | `common/gameQuest/src/pauseConditionNode.cpp` |
| **Token系统** | `common/gameSceneSystem/src/scnTokens.h` | - |
| **Token数据** | `common/gameSceneSystem/src/scnsTokenData.h` | - |
| **Section通用** | `common/gameSceneSystem/src/scnsSectionCommon.h` | `common/gameSceneSystem/src/scnsSectionCommon.cpp` |
| **Condition定义** | `common/gameSceneSystem/src/scnsSceneCondition.h` | - |
| **Condition类型** | `common/gameSceneSystem/src/scnsSceneConditionType.h` | - |
| **编译器优化** | `backend/backendScenes/src/scnbCompilerFunctions.cpp` | - |

### 2.2 文档结构

- `1_PauseNode架构.md` - PauseNode节点实现详解
- `2_Token机制.md` - Token系统完整实现
- `3_Condition系统.md` - Condition条件系统
- `4_SectionNode处理.md` - Section节点Token处理
- `5_编译器自动生成.md` - 自动生成PauseNode的编译器机制
- `6_代码示例.md` - 实际使用示例

---

## 三、核心设计模式

### 3.1 Token传递模式
使用Token在节点图中传递控制流和数据，实现异步状态管理。

### 3.2 Condition-Event模式
- 条件未满足时：注册事件监听器
- 条件满足时：触发回调，释放Token

### 3.3 State模式
每个节点维护自己的状态 (TokenData)，支持保存和恢复。

### 3.4 变体模式
TokenVariantData使用union存储不同类型的数据，实现类型安全的多态存储。

---

## 四、关键类层次结构

### 4.1 PauseNode类继承

```cpp
DisableableNodeDefinition (基类)
    ↓
SignalStoppingNodeDefinition (信号停止节点)
    ↓
PauseConditionNodeDefinition (暂停条件节点)
```

### 4.2 Token类型体系

```cpp
// 三种核心Token类型
ExecutionToken    - 节点执行状态Token
SignalToken       - 节点间信号传递Token
StateToken        - 节点状态保存Token
```

### 4.3 Condition类型层次

```cpp
ISceneConditionType (接口)
    ├─ SceneNode_ConditionType        (场景节点条件)
    ├─ SectionNode_ConditionType      (Section节点条件) ★核心
    ├─ SceneInterrupt_ConditionType   (场景中断条件)
    ├─ SceneReturn_ConditionType      (场景返回条件)
    ├─ SceneTalking_ConditionType     (对话条件)
    ├─ SceneTier_ConditionType        (层级条件)
    └─ NewPlayerPuppetAttached_ConditionType (玩家附加条件)
```

---

## 五、核心枚举定义

### 5.1 TokenState (Token状态)

```cpp
enum class TokenState
{
    inactive,  // 不活跃状态
    active     // 活跃状态
};
```

### 5.2 TokenDataType (Token数据类型)

```cpp
enum class TokenDataType
{
    emptyData,                    // 空数据
    completionData,               // 完成数据
    andNodeState,                 // And节点状态
    xorNodeState,                 // Xor节点状态
    hubNodeState,                 // Hub节点状态
    randomizerNodeState,          // 随机器节点状态
    choiceNodeState,              // 选择节点状态
    sectionNodeState,             // Section节点状态 ★
    questNodeState,               // Quest节点状态
    rewindableSectionNodeState,   // 可回退Section状态
    produceTriggerData,           // 触发器数据
    interruptManagerNodeState,    // 中断管理器状态
};
```

### 5.3 NodeResult (节点结果)

```cpp
enum class NodeResult
{
    LeaveNode,   // 离开节点，继续执行
    StayInNode   // 停留在节点，等待条件
};
```

### 5.4 PlaySpeed (播放速度)

```cpp
enum class PlaySpeed
{
    Pause,      // 暂停
    Normal,     // 正常速度
    Slow,       // 慢速
    Fast,       // 快速
    VeryFast    // 超快速
};
```

---

## 六、关键数据结构

### 6.1 TokensState (Token状态集合)

```cpp
struct TokensState
{
    Bool activeStartToken : 1;           // 启动Token
    Bool activeCancelToken : 1;          // 取消Token
    Bool activeDisableToken : 1;         // 禁用Token

    Bool activePauseToken : 1;           // 暂停Token ★核心
    Bool activeForwardNormalToken : 1;   // 前进正常Token
    Bool activeForwardSlowToken : 1;     // 前进慢速Token
    Bool activeForwardFastToken : 1;     // 前进快速Token
    Bool activeForwardVeryFastToken : 1; // 前进超快Token
    Bool activeBackwardNormalToken : 1;  // 后退正常Token
    Bool activeBackwardSlowToken : 1;    // 后退慢速Token
    Bool activeBackwardFastToken : 1;    // 后退快速Token
    Bool activeBackwardVeryFastToken : 1;// 后退超快Token
    Bool activeControlTimeToken : 1;     // 时间控制Token

    Bool activeExeToken : 1;             // 活跃执行Token
    Bool inactiveExeToken : 1;           // 不活跃执行Token
    Bool exeToken : 1;                   // 执行Token标志
};
```

### 6.2 NodeProcessingState (节点处理状态)

```cpp
class NodeProcessingState
{
public:
    Uint32 m_eventCursor{ 0 };                              // 事件数组索引
    TimeWindow m_nodeTwActivity{ SceneTime(0), SceneTime(0) }; // 节点活动时间窗口
    TimeWindow m_realTwActivity{ SceneTime(0), SceneTime(0) }; // 实际活动时间窗口

    utils::SDArray< SceneEventId, 32 > m_activeEvents;      // 活跃事件列表
};
```

---

## 七、关键代码位置索引

### 7.1 PauseNode核心逻辑

| 功能 | 文件 | 行号 |
|------|------|------|
| OnExecute实现 | `pauseConditionNode.cpp` | 131-163 |
| 条件检查 | `pauseConditionNode.cpp` | 144 |
| EventListener注册 | `pauseConditionNode.cpp` | 156 |
| EventListener注销 | `pauseConditionNode.cpp` | 148 |

### 7.2 Token处理逻辑

| 功能 | 文件 | 行号 |
|------|------|------|
| Token激活 | `scnTokens.h` | 201-215 |
| Token失效 | `scnTokens.h` | 180-186 |
| Token数据设置 | `scnTokens.h` | 299-302 |

### 7.3 Section Token处理

| 功能 | 文件 | 行号 |
|------|------|------|
| TokensState定义 | `scnsSectionCommon.h` | 21-43 |
| PauseToken处理 | `scnsSectionCommon.cpp` | 358-360 |
| 播放速度控制 | `scnsSectionCommon.cpp` | 362-393 |

### 7.4 编译器自动生成

| 功能 | 文件 | 行号 |
|------|------|------|
| 创建PauseNode | `scnbCompilerFunctions.cpp` | 96-117 |
| 图预处理 | `scnbCompilerFunctions.cpp` | 1544-1653 |

---

## 八、使用场景

### 8.1 典型应用场景

1. **等待Scene Section完成**
   - PauseNode检查Section是否已经播放完成
   - 使用 `SectionNode_ConditionType` 配置条件
   - 类型: `SceneConditionType::IsOutside`

2. **等待特定场景节点**
   - 等待某个Scene节点激活或完成
   - 使用 `SceneNode_ConditionType`

3. **等待角色对话**
   - 检查特定角色是否在对话中
   - 使用 `SceneTalking_ConditionType`

4. **等待场景层级变化**
   - 等待游戏进入特定Tier
   - 使用 `SceneTier_ConditionType`

### 8.2 编译时自动插入场景

```
编译前:
QuestNode → SectionNode

编译后 (自动插入):
QuestNode → PauseNode (等待Section完成) → SectionNode
```

---

## 九、性能考虑

### 9.1 Token机制优化
- Token使用位域 (bitfield) 压缩状态存储
- 使用union实现零开销的类型多态
- 预分配的活跃事件数组 (SDArray<SceneEventId, 32>)

### 9.2 Condition检查优化
- 支持立即检查 (IsImmediate)
- 事件驱动而非轮询
- 条件满足时自动注销监听器

---

## 十、调试提示

### 10.1 常见问题排查

**Quest卡住不继续执行:**
1. 检查PauseNode的Condition是否永远无法满足
2. 查看EventListener是否正确注册
3. 确认Section是否正常完成

**Token未正确传递:**
1. 检查TokensState的各个标志位
2. 确认Operation类型 (startProcessing/continueProcessing等)
3. 查看是否有DisableToken或CancelToken阻止

### 10.2 调试宏

```cpp
#define DEBUG_SCN_TOKENS  // 启用Token调试信息
```

---

## 十一、扩展阅读

详细实现请参考配套文档:
- `1_PauseNode架构.md` - PauseNode完整实现
- `2_Token机制.md` - Token系统深入解析
- `3_Condition系统.md` - Condition体系详解
- `4_SectionNode处理.md` - Section处理流程
- `5_编译器自动生成.md` - 自动生成机制
- `6_代码示例.md` - 实际使用案例

---

**文档版本:** 1.0
**最后更新:** 2026-02-13
**基于源码:** Cyberpunk 2077 Dev Build
