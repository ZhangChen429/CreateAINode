# 动画节点/片段 完整说明文档
---

## 一、基础动画类
### 1. AnimClip（普通动画）
| 属性名 | 类型 | 说明 |
| ---- | ---- | ---- |
| m_animName | CName | 要播放的动画文件名（不含扩展名） |
| m_blendOutTime | Float | 动画结束时的淡出时间，用于平滑过渡到下一个动画（默认 0.5秒） |

**用途**：普通手势动画、idle 动画等纯姿态类无位移动画

---

### 2. MotionAnimClip（带位移的动画）
> ✅ 继承 AnimClip 的所有参数
> ✅ 额外标志：`IEntry::MotionAnim`

**用途**：
- 包含根骨骼位移的动画（如走路、转身、跑步）
- 系统会自动将动画中的位移数据应用到角色实际位置

**使用场景**：移动动画、需要改变角色世界坐标位置的动画

---

### 3. SyncAnimClip（同步动画）
| 属性名 | 类型 | 说明 |
| ---- | ---- | ---- |
| m_animName | CName | 动画名称 |
| m_blendOutTime | Float | 混出时间 |
| m_slotName | CName | 同步槽位名称（如 "chair_slot_01"） |
| m_syncOffset | Transform | 相对于同步点的偏移（位置+旋转双维度） |

**用途**：多人同步动画，确保多个角色在交互时精准对齐位置与姿态
**使用场景**：双人动画、多人交互（握手、搬箱子、协作动作）

---

### 4. AnimClipWithItem（带道具的动画）
| 属性名 | 类型 | 说明 |
| ---- | ---- | ---- |
| m_animName | CName | 动画名称 |
| m_blendOutTime | Float | 混出时间 |
| m_itemActions | red::DynArray<THandle<IWorkspotItemAction>> | 道具动作列表 |

**用途**：`m_itemActions` 定义动画过程中道具的生命周期行为，包含：
- SpawnItem：生成道具
- DespawnItem：销毁道具
- AttachItem：将道具附加到指定骨骼

**使用场景**：拿起/放下物品、使用武器/工具、拾取道具的动画

---

### 5. LookAtDrivenTurn（视线驱动转身）
| 属性名 | 类型 | 说明 |
| ---- | ---- | ---- |
| m_turnAnimName | CName | 转身动画名称 |
| m_turnAngle | Int32 | 转身角度（度，可从动画名自动提取/手动设置） |
| m_blendTime | Float | 混合时间（默认 0.5秒） |

**用途**：
- 系统根据角色当前视线方向，自动匹配最贴合需求的转身动画
- 示例：需要转35°时，自动选择turn45动画而非turn90动画，过渡更自然

**使用场景**：NPC 看向目标对象时的智能自动转身、视角切换转身

---

## 二、进入/退出类
### 6. EntryAnim（入口动画）
| 属性名 | 类型 | 说明 |
| ---- | ---- | ---- |
| m_animName | CName | 从外部进入 workspot 的动画（如 walk_to_sit） |
| m_idleAnim | CName | 进入后的 idle 姿态标识（transition核心参数） |
| m_slotName | CName | 同步槽位（可选） |
| m_blendOutTime | Float | 混出时间 |
| m_isSynchronized | Bool | 是否同步 |
| m_syncOffset | Transform | 同步偏移 |
| m_movementType | move::MovementType | 移动类型（Walk/Jog/Sprint） |
| m_orientationType | move::MovementOrientationType | 朝向类型（Forward/TowardsObject） |

**核心用途**：角色从自由状态进入指定交互位姿的过渡动画
**使用场景**：角色走向椅子并坐下、走到桌子旁俯身、走到操作台站立

---

### 7. ExitAnim（慢速退出）
| 属性名 | 类型 | 说明 |
| ---- | ---- | ---- |
| m_animName | CName | 退出动画名称 |
| m_slotName | CName | 同步槽位（可选） |
| m_idleAnim | CName | 退出前需要的 idle 姿态（系统自动transition） |
| m_isSynchronized | Bool | 是否同步 |
| m_stayOnNavmesh | Bool | 保持在导航网格上，确保退出后在可行走区域 |
| m_snapZToNavmesh | Bool | Z轴吸附到导航网格，防止浮空/穿地 |
| m_disableRandomExit | Bool | 禁用随机选择，设为true时仅能手动指定调用 |
| m_syncOffset | Transform | 同步偏移 |
| m_movementType | move::MovementType | 移动类型 |

**核心用途**：角色从交互位姿平稳退出到自由状态
**使用场景**：角色从椅子站起并走开、从桌子旁直身离开

---

### 8. FastExit（快速退出）
| 属性名 | 类型 | 说明 |
| ---- | ---- | ---- |
| m_animName | CName | 快速退出动画 |
| m_forcedBlendIn | Float | 强制混入时间（默认 0.2秒，极短） |
| m_movementType | move::MovementType | 移动类型 |

**用途**：`m_forcedBlendIn` 为极短的混入时间，可立即中断当前所有动作，无平滑过渡强制切出
**使用场景**：NPC 受惊吓、战斗触发、紧急躲避等突发/紧急情况（如从椅子上快速跳起）

---

## 三、容器类
### 9. Sequence（顺序序列）
| 属性名 | 类型 | 说明 |
| ---- | ---- | ---- |
| m_idleAnim | CName | 该序列的 idle 姿态标识（核心参数） |
| m_loopInfinitely | Bool | 是否无限循环子节点动画 |
| m_category | game::data::WorkspotCategory | 分类标签（如 "phone", "relax"） |
| m_list | red::DynArray<THandle<IEntry>> | 子节点列表 |

**用途**：按顺序依次播放列表内的所有子节点动画/容器
**核心特性**：
- true = 循环播放子节点至退出/中断
- false = 播放一遍子节点后进入序列的idle姿态
- m_category 用于Selector的概率筛选匹配

**使用场景**：组织一组有固定播放顺序的动画（如：坐下→拿手机→看手机→放下手机）

---

### 10. RandomList（随机列表）
| 属性名 | 类型 | 说明 |
| ---- | ---- | ---- |
| m_minClips | Int8 | 最少随机播放次数 |
| m_maxClips | Int8 | 最多随机播放次数 |
| m_dontRepeatLastAnims | Int8 | 避免重复最近N个播放过的动画 |
| m_pauseBetweenLength | Float | 动画之间的固定间隔暂停时长（秒） |
| m_pauseLengthDeviation | Float | 暂停时长随机偏差值 |
| m_pauseBlendOutTime | Float | 暂停阶段的混出时间 |
| m_weights | red::DynArray<Float> | 每个子节点的选中权重（如 [1.0,2.0,0.5]） |

**用途**：从子节点中随机抽取动画播放，实现动作多样化，避免机械重复
**核心特性**：
- 实际播放数量在 [m_minClips, m_maxClips] 区间内随机
- 权重越高的子节点被选中的概率越大
- 可规避最近播放的N个动画，防止连续重复

**使用场景**：随机手势、多样化的idle小动作、无固定顺序的休闲动画

---

### 11. Selector（智能选择器）
> ✅ 继承 RandomList 的所有参数
> ✅ 新增核心参数：m_idleAnim（基础idle，回退用）

**核心用途**：
- 自动处理不同idle姿态之间的transition过渡逻辑
- 当找不到子节点的直接过渡动画时，自动切到m_idleAnim作为中间过渡状态
- 结合权重+分类标签实现精准的随机选择

**使用场景**：统一管理多个不同姿态的Sequence（如：坐姿看手机、坐姿放松、坐姿喝水三类序列）

---

### 12. ReactionSequence（反应序列）
| 属性名                               | 类型                                | 说明                                                    |
| ------------------------------------ | ----------------------------------- | ------------------------------------------------------- |
| m_reactionTypes                      | red::DynArray<game::data::RecordID> | 支持的反应类型列表（TweakDB 中的 WorkspotReactionType） |
| m_forcedBlendIn                      | Float                               | 被外部事件触发时的强制混入时间                          |
| m_facialKeyWeight                    | Float                               | 面部动画权重                                            |
| m_mainEmotionalState                 | CName                               | 主要情绪状态                                            |
| m_emotionalExpression                | CName                               | 情绪表达细节                                            |
| m_facialIdleMale/FemaleAnimation     | CName                               | 男性/女性基础面部idle动画                               |
| m_facialIdleKey_Male/FemaleAnimation | CName                               | 男性/女性核心面部表情动画                               |

**用途**：处理外部事件触发的角色反应动画+面部表情联动
**⚠️ 重要强制规则**：必须放在 Root 节点的直接子级！
**使用场景**：处理 bump（碰撞）、fear（恐惧）、alert（警觉）等外部触发的反应动作

---

### 13. ConditionalSequence（条件序列）
| 属性名 | 类型 | 说明 |
| ---- | ---- | ---- |
| m_multipleConditionOperator | LogicalOperation | 多条件逻辑运算规则（AND/OR） |
| m_conditionList | red::DynArray<THandle<IWorkspotCondition>> | 条件检测列表 |

**核心逻辑**：
- AND 规则：列表中**所有条件**都满足时，才会播放该序列的动画
- OR 规则：列表中**任一条件**满足时，即可播放该序列的动画

**用途**：根据角色属性动态选择匹配的动画变体
**使用场景**：按角色性别/体型/装备/职业选择不同的坐下/起身动画、按角色状态选择不同的交互动作

---

## 四、特殊类
### 14. PauseClip（暂停片段）
| 属性名 | 类型 | 说明 |
| ---- | ---- | ---- |
| m_timeMin | Float | 最短暂停时间（秒） |
| m_timeMax | Float | 最长暂停时间（秒） |
| m_blendOutTime | Float | 混出时间 |

**用途**：在动画序列中插入随机时长的停顿，提升角色动作的自然度
**核心特性**：
- 实际暂停时长在 [m_timeMin, m_timeMax] 区间内随机取值
- 暂停期间会持续播放当前绑定的idle动画，无姿态断层
- 避免角色机械性连续播放动作，贴近真实行为逻辑

**使用场景**：NPC休闲状态的动作间隔、交互过程中的停顿思考、无指令时的自然待机

---

### 15. TagNode（标签节点）
| 属性名 | 类型 | 说明 |
| ---- | ---- | ---- |
| m_tag | CName | 唯一标签名称 |

**用途**：作为动画序列中的**跳转标记点**，无实际动画播放逻辑
**调用方式**：通过脚本调用 `JumpToEntry(tagName)` 方法，可直接跳转到该标签对应的位置继续播放动画

**使用场景**：剧情流程的分支跳转、触发特定事件后切到指定动画段落、循环播放指定动画片段

---

## 五、参数使用优先级和注意事项
### ⚠️ 核心必设参数（错误设置=致命问题，优先级最高）
#### 1. m_idleAnim （容器类/EntryAnim/ExitAnim 均包含）
- ❌ 错误影响：transition过渡失败、角色穿模、动画卡顿、姿态断层、退出/进入逻辑失效
- ✅ 正确要求：必须与动画的**实际结束姿态**完全一致，做到姿态名与动画姿态一一匹配
- ✅ 核心作用：是所有动画过渡的锚点，系统通过该参数识别"当前角色姿态"以匹配下一个动画

#### 2. m_reactionTypes （仅 ReactionSequence）
- ❌ 错误影响：外部事件无法触发任何反应动画，该序列完全失效
- ✅ 正确要求：必须准确填写 TweakDB 中的标准 RecordID，无拼写/格式错误

#### 3. m_movementType （仅 EntryAnim/ExitAnim）
- ❌ 错误影响：AI寻路逻辑紊乱、动画速度与移动速度不匹配、角色滑步/瞬移
- ✅ 正确要求：Walk/Jog/Sprint 三种类型，必须与对应动画的实际移动速度严格匹配

---

### ✅ 经典常用参数组合示例
#### ✔️ 示例1：坐椅子完整动画配置（最常用）
```
Entry Anim 配置：
  m_animName = "walk_to_sit_front"  // 走向椅子并坐下的动画
  m_idleAnim = "sit_front"          // 核心必配：坐下后的最终姿态
  m_movementType = Walk             // 步行走向椅子

Sequence 配置：
  m_idleAnim = "sit_front"          // 必须与Entry的m_idleAnim完全匹配
  m_loopInfinitely = true           // 无限循环坐姿动画

Exit Anim 配置：
  m_animName = "sit_front_to_stand" // 从椅子站起的动画
  m_idleAnim = "sit_front"          // 核心必配：站起前的初始姿态
  m_movementType = Walk             // 步行离开椅子
```

#### ✔️ 示例2：多姿态管理（Selector+多Sequence 组合）
```
Selector 根配置：
  m_idleAnim = "sit_base"           // 基础坐姿，作为过渡中间态

├─ Sequence 子节点1：
│   m_idleAnim = "sit_phone"        // 看手机的坐姿姿态
│   m_category = "phone"            // 分类标签：手机相关
│
└─ Sequence 子节点2：
    m_idleAnim = "sit_relax"        // 放松的坐姿姿态
    m_category = "relax"            // 分类标签：休闲相关
```