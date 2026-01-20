# CDPR Workspot 与动画系统设计规范
## 一、命名规范体系
### 1. Workspot 文件命名模式
**命名格式**
```
{角色}__{姿势}_{位置}_{道具}__{行为}__{变体编号}.workspot
```

**示例拆解**
```
hanako__sit_barstool_bar__sit_around__01
└─────┘  └──────────────────┘ └────────┘ └┘
  角色      姿势+位置+道具        行为    变体
```

```
johnny__stand_ground_cigarette__idle__01
└─────┘ └──────────────────────┘ └──┘ └┘
  角色    姿势+位置+道具           行为 变体
```

```
generic__stand_ground__stand_around__04
└─────┘ └───────────┘ └───────────┘ └┘
  通用NPC   姿势+位置      行为        变体
```

**命名元素枚举**
| 元素类型 | 可选值 |
|----------|--------|
| 角色类型 | hanako, johnny, player, generic, bodyguard, judy |
| 姿势 | sit, stand, walk |
| 位置 | ground（地面）, barstool（吧台椅）, chair（椅子）, stage（舞台） |
| 道具 | cigarette（香烟）, piano（钢琴）, pistol（手枪） |
| 行为 | idle, sit_around, stand_around, play, sing, alerted, aggressive |

### 2. Idle 动画命名规范
**命名格式**
```
{角色}_{姿势}_{具体位置}_{倾斜角度}__{手部位置}_{腿部状态}_{变体}
```

**示例拆解**
```
hanako_sit_barstool_bar_lean90__rh_on_bar_legs_crossed_01
└─────┘└─┘└───────────┘└─────┘ └────────┘└───────────┘└┘
  角色  姿势   位置       倾斜    手部位置   腿部状态   变体
```

```
johnny_stand__lh_cigarette__aggressive_01
└─────┘└───┘ └───────────┘ └──────────┘└┘
  角色  姿势   手部+道具      情绪状态   变体
```

```
sit_barstool_bar_lean270__lh_on_bar_01
└─┘└───────────┘└──────┘ └─────────┘└┘
  姿势   位置       倾斜     手部位置  变体
```

**核心枚举配置**
- **倾斜角度系统**
  - lean0 - 正前方
  - lean90 - 向右倾斜 90°
  - lean180 - 向后
  - lean270 - 向左倾斜 90°

- **手部位置**
  - rh_on_bar - 右手放在吧台上
  - lh_on_bar - 左手放在吧台上
  - 2h_on_bar - 双手放在吧台上
  - lh_cigarette - 左手拿烟
  - arms_legs_crossed - 双臂双腿交叉

### 3. Entry/Exit 动画命名规范
**Entry 动画格式**
```
walk_{direction}__to__{target_pose}__turn{angle}_{variant}
```

**示例拆解**
```
walk_0__to_stand_2h_on_sides_01__turn315_01
└────┘  └──────────────────────┘ └──────┘└┘
  方向        目标姿势               旋转角度 变体
```

```
walk_0__to_sit_barstool_bar_lean0__2h_on_bar_01__turn45_01
└────┘  └──────────────────────────────────────┘ └─────┘└┘
  方向              目标姿势                        旋转   变体
```

**核心枚举配置**
- **支持的进入方向**
  - walk_0 - 从正前方走来（0°）

- **支持的旋转角度**（全方位 8 方向覆盖）
  - turn0_01、turn45_01、turn90_01、turn135_01
  - turn180_01、turn225_01、turn270_01、turn315_01

## 二、IEntry 树的组织模式
### 模式 1：简单循环（Simple Idle Loop）
**用途**：静态 NPC 的循环待机动作
**示例**：`hanako__sit_barstool_bar__sit_around__01`
**结构**
```
Root sequence
└─ Sequence (idle: hanako_sit_barstool_bar_lean90__rh_on_bar_legs_crossed_01)
   └─ Random list
      ├─ Anim: hanako__sit_barstool_bar_lean90__rh_on_bar_legs_crossed_01__sip_drink_01
      ├─ Anim: hanako__sit_barstool_bar_lean90__rh_on_bar_legs_crossed_01__sip_drink_02
      └─ Anim: hanako__sit_barstool_bar_lean90__rh_on_bar_legs_crossed_01__nervous_tap_01
```
**特点**
- 结构简单，仅包含基础 idle 姿势 + RandomList 变体
- 适合背景 NPC 无交互待机

### 模式 2：分段循环（Segmented Loop with Pauses）
**用途**：需要停顿感的角色行为
**示例**：`johnny__stand_ground_cigarette__idle__01`
**结构**
```
Root sequence
└─ Sequence (idle: johnny_stand__lh_cigarette__aggressive_01)
   ├─ Random list
   │  └─ Sequence
   │     ├─ Random list
   │     │  ├─ Anim: johnny__stand__lh_cigarette__aggressive_01__puff_01
   │     │  ├─ Anim: johnny__stand__lh_cigarette__aggressive_01__puff_02
   │     │  ├─ Anim: johnny__stand__lh_cigarette__aggressive_01__puff_03
   │     │  └─ ...
   │     └─ Pause (1.000000 - 2.000000)  ← 随机停顿 1-2 秒
   │
   ├─ Sequence
   │  ├─ Random list
   │  │  ├─ Anim: shuffle_01
   │  │  ├─ Anim: shuffle_02
   │  │  └─ Anim: think_01
   │  └─ Pause (1.000000 - 2.000000)
```
**特点**
- 嵌套 Sequence 实现分段动作
- 每段动作后添加随机 Pause 节点，创造自然节奏感
- 适合主要角色的 idle 动作表现

### 模式 3：反应系统（Reaction System）
**用途**：支持物理碰撞反应的 NPC
**示例**：`generic__stand_ground__stand_around__04`
**结构**
```
Root sequence
├─ Reaction Sequence: idle: <no_auto_transition> [BumpRightBack]
│  └─ Sequence (idle: stand_2h_on_sides_01)
│     └─ Anim: stand_2h_on_sides_01_bump_back_right
│
├─ Reaction Sequence: idle: <no_auto_transition> [BumpLeftBack]
│  └─ Sequence (idle: stand_2h_on_sides_01)
│     └─ Anim: stand_2h_on_sides_01_bump_back_left
│
├─ Reaction Sequence: idle: <no_auto_transition> [BumpRightFront]
│  └─ ...
│
├─ Reaction Sequence: idle: <no_auto_transition> [BumpLeftFront]
│  └─ ...
│
└─ Sequence (idle: stand_2h_on_sides_01)  ← 主循环
   └─ Random list
      └─ ...
```
**特点**
- 顶层包含多个 ReactionSequence 节点，对应不同碰撞方向
- `idle: <no_auto_transition>` 表示禁用自动过渡
- 游戏逻辑检测到碰撞后，触发对应方向的 Reaction 动画

### 模式 4：多方向入场系统（Multi-Directional Entry）
**用途**：需要从多个方向进入的 Workspot
**示例**：`bodyguard__stand_ground_pistol__alerted__01`
**结构**
```
Root sequence
├─ Entry anim: walk_0__to_sit_barstool_bar_lean0__2h_on_bar_01__turn315_01
├─ Entry anim: stand_2h_on_sides_01__to_sit_barstool_bar_lean0__2h_on_bar_01__turn45_01
├─ Entry anim: walk_0__to_sit_barstool_bar_lean270__lh_on_bar_01__turn180_01
├─ Entry anim: walk_0__to_sit_barstool_bar_lean270__lh_on_bar_01__turn90_01
├─ ... (大量 Entry anim，覆盖所有角度和姿势组合)
│
├─ Reaction Sequence [Fear]
│  └─ Sequence (idle: stand_2h_up_01)
│     └─ Anim: sit_barstool_2h_up_01
│
├─ Fast exit: sit_barstool__2h_up_01__to_stand_2h_up_01
│
└─ Sequence (idle: sit_barstool_bar_lean270__lh_on_bar_01)
   ├─ Pause (6.000000 - 10.000000)
   └─ Exit anims...
```
**特点**
- 包含大量 Entry anim，覆盖 **8 方向 × 多起始姿势 × 多目标姿势** 组合
- 运行时根据 Actor 当前位置和朝向，自动选择最合适的 Entry anim
- 配置 Fast exit 节点，支持紧急退出 Workspot

### 模式 5：Look-at Driven Turn（视线驱动转身）
**用途**：角色需要注视目标并转身
**示例**：`hanako__stand_stage__sing__01`
**结构**
```
Root sequence
└─ Sequence (idle: hanako_stand_2h_front_01)
   ├─ Look-at driven turn: hanako_stand_2h_front_01__turn240_01
   ├─ Look-at driven turn: None
   ├─ Look-at driven turn: hanako_stand_2h_front_01__turn120_01
   ├─ Pause (2.000000 - 5.000000)
   ├─ Anim: hanako_stand_2h_front_01__shuffle_01
   └─ Pause (2.000000 - 5.000000)
```
**特点**
- `Look-at driven turn` 节点根据目标位置，自动选择转身动画
- `None` 表示目标在正前方，无需转身
- 支持任意角度转身（如 turn120_01、turn240_01）

### 模式 6：Motion Anim（运动动画）
**用途**：带根运动的对话动作
**示例**：`johnny__walk_ground_cigarette__aggressive__01`
**结构**
```
Root sequence
└─ Sequence (idle: johnny_stand__lh_cigarette__aggressive_01)
   └─ Random list
      ├─ Motion anim: johnny__stand__lh_cigarette__aggressive_01__conversation_step_01
      └─ Motion anim: johnny__stand__lh_cigarette__aggressive_01__conversation_step_02
```
**特点**
- Motion anim 会改变角色在世界空间中的位置
- 用于对话中的小幅度移动（向前一步、后退、侧步等）
- 图标为橙色圆点，区别于普通 Anim 的绿色圆点

### 模式 7：特殊行为（Special Behavior）
#### 示例 A：钢琴演奏
**示例**：`hanako__stand_ground_piano__play__01`
**结构**
```
Root sequence
└─ Sequence (idle: hanako_sit_chair_lean0__play_piano_01)
   ├─ Pause (5.000000 - 10.000000)  ← 演奏时长
   └─ Exit anim: hanako_sit_chair_lean0__play_piano_01__to__hanako_stand_2h_front_01
```

#### 示例 B：简单坐姿
**示例**：`johnny__sit_chair_lean_back__sit_around__01`
**结构**
```
Root sequence
└─ Sequence (idle: johnny_sit_chair_lean180__arms_legs_crossed_01)
   └─ Random list
      ├─ Anim: johnny__sit_chair_lean180__arms_legs_crossed_01__shuffle_01
      ├─ Anim: johnny__sit_chair_lean180__arms_legs_crossed_01__scratch_neck_01
      ├─ Anim: johnny__sit_chair_lean180__arms_legs_crossed_01__deep_breath_01
      └─ Anim: johnny__sit_chair_lean180__arms_legs_crossed_01__look_around_01
```

## 三、GlobalWorkspotProperties 配置模式
### 属性面板结构
```
Global workspot properties:
├─ Dont Inject Workspot Graph: [false]
├─ Anim Graph Slot Name: WORKSPOT
├─ Auto Transition Blend Time: 1.000000
├─ Initial Actions: [initialActions] array<0>
├─ Blend Out Time: 0.000000
├─ Entities Paths: [entitiesPaths] array<1> 或 array<2>
├─ Animsets: [animsets] array<0>
└─ Final Animsets: [finalAnimsets] array<1> 或 array<2>
```

### 关键配置观察
1. **Animsets vs Final Animsets**
   - 几乎所有案例中 `animsets: array<0>`（空）
   - `finalAnimsets` 均配置有效值（array<1> 或 array<2>）
   - 证实 AnimSet 自动生成机制

2. **Entities Paths 配置**
   - 案例 `q115__player__sit_barstool_lean_left__sit_around__01` 配置 `array<2>`
   - 候选 Entity 通常为 `man_average.ent` 和 `woman_average.ent`
   - 用于自动生成 AnimSets 的角色资源

3. **Workspot Rig 绑定**
   - 案例 `q115__hanako__sit_barstool_bar__sit_around__01` 绑定 `generic_barstool_bar_item_skeleton`
   - 实现 Workspot 与道具骨架的对齐

## 四、实际应用的设计模式总结
### 1. 角色定位分类
| 角色类型 | Workspot 特点 | 示例 |
|----------|---------------|------|
| 主要角色（Johnny, Hanako） | 复杂嵌套结构，多段 Pause，丰富变体动作 | `johnny__stand_ground_cigarette__idle__01` |
| 玩家 | 支持多方向入场，大量 Entry/Exit anim | `q115__player__sit_barstool_lean_left__sit_around__01` |
| 通用 NPC | 完整反应系统，支持 8 方向入场 | `generic__stand_ground__stand_around__04` |
| 特殊职业（Bodyguard） | 武器姿势变体，警戒反应逻辑 | `bodyguard__stand_ground_pistol__alerted__01` |

### 2. 行为复杂度分级
| 复杂度 | IEntry 层级 | 节点类型 | 使用场景 |
|--------|-------------|----------|----------|
| 简单 | 2-3 层 | Sequence + RandomList | 背景 NPC idle |
| 中等 | 3-5 层 | Sequence + RandomList + Pause | 前景 NPC idle |
| 复杂 | 5-7 层 | 多 Sequence 嵌套 + Pause + Entry/Exit | 互动 NPC |
| 超复杂 | 7+ 层 | Reaction + Entry(多方向) + Exit + FastExit | 战斗/警戒 NPC |

### 3. 过渡动画覆盖策略
| 策略类型 | 特点 | 适用场景 |
|----------|------|----------|
| 完整覆盖型 | 覆盖所有 walk→sit/stand 转换、8 旋转角度，含 50+ Entry anim | 战斗/警戒 NPC、玩家角色 |
| 最小覆盖型 | 仅保留核心 idle 循环，无 Entry/Exit anim | 背景静态 NPC、简单互动角色 |

### 4. Pause 节点的节奏设计
| Pause 时长范围 | 使用场景 | 效果 |
|----------------|----------|------|
| 1.0 - 2.0 秒 | 快速动作间隙 | 紧凑节奏（抽烟、小幅度移动） |
| 2.0 - 5.0 秒 | 标准 idle 间隙 | 自然节奏（等待、观察、呼吸） |
| 5.0 - 10.0 秒 | 特殊行为持续 | 慢节奏（演奏、沉思、对话） |
| 6.0 - 10.0 秒 | 长时间静止 | 静态场景（坐着发呆、站岗） |

## 五、核心设计哲学
1. **组合优于继承**
   - 通过 `Sequence + RandomList + Pause` 的节点组合，实现无限行为变化
   - 避免为每种行为创建独立节点类型，降低系统复杂度

2. **自动化优于手动配置**
   - AnimSets 完全由系统自动生成，无需手动维护
   - 仅需配置 `Entities Paths` 候选角色，即可完成资源绑定
   - 系统自动扫描 IEntry 树，提取所需动画资源

3. **覆盖所有可能性**
   - Entry anim 覆盖 **8 方向 × 多起始姿势 × 多目标姿势** 全组合
   - Reaction 系统覆盖 4 个碰撞方向，支持物理交互
   - Look-at 转身支持任意角度，实现自然的目标注视

4. **语义化命名**
   - 动画名称包含**姿势+角度+身体部位**完整信息
   - 设计师可通过名称直接理解动画内容，无需打开预览
   - 便于自动化工具解析、分类、管理动画资源

5. **分层设计**
   - **Workspot 层**：定义角色行为逻辑（状态、过渡、反应）
   - **AnimSet 层**：提供动画资源（idle、entry、exit、reaction）
   - 两层通过 `CName` 延迟绑定，实现逻辑与资源的解耦

通过这套设计，CDPR 用有限的代码框架，为《赛博朋克2077》创造出了无限的角色行为组合，兼顾了表现力与开发效率。

---

是否需要我帮你把这份文档里的**命名规范和设计模式**整理成一份可直接导入的**配置模板表格**？