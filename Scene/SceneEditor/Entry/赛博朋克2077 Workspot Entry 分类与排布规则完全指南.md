# 赛博朋克 2077 Workspot Entry 分类与排布规则完全指南

基于 workspot 编辑器截图和源码解析，以下是 Entry 的详细分类、技术特性及排布规范（源码引用源自 `workspotResource.h` `workspotTreeItems.h` 等核心文件）。

## 一、Entry 类型分类（基于标志位）

在 `workspotResource.h:187-216` 中定义了所有 Entry 的标志位枚举：



```
enum EntryFlags : Uint32

{

&#x20;   // === 节点类型标志 ===

&#x20;   Animation        = RED\_FLAG(1),   // 0x002 - 普通动画

&#x20;   FastExit         = RED\_FLAG(2),   // 0x004 - 快速退出

&#x20;   SlowExit         = RED\_FLAG(3),   // 0x008 - 慢速退出

&#x20;   SlowEnter        = RED\_FLAG(4),   // 0x010 - 慢速进入

&#x20;   Pause            = RED\_FLAG(5),   // 0x020 - 暂停

&#x20;   Synchronized     = RED\_FLAG(6),   // 0x040 - 同步动画

&#x20;   TagNode          = RED\_FLAG(7),   // 0x080 - 标签节点

&#x20;   Reaction         = RED\_FLAG(8),   // 0x100 - 反应

&#x20;   LookAtDrivenTurn = RED\_FLAG(9),   // 0x200 - 视线驱动转身

&#x20;   // === 组合标志 ===

&#x20;   EnterNode  = SlowEnter,

&#x20;   ExitNode   = FastExit | SlowExit,

&#x20;   MotionNode = ExitNode | SlowEnter | Synchronized | MotionAnim | MoveToMotionAnim | LookAtDrivenTurn,

&#x20;   LeafNode   = Animation | FastExit | SlowExit | SlowEnter | Pause,

};
```

## 二、从截图中识别的 Entry 类型

### 1. 容器类型（紫色图标）

#### A. Reaction Sequence（反应序列）



* **图标**：紫色序列图标，带有 bump 方向标签

* **代码**：`work::ReactionSequence` (workspotTreeItems.h:345)

* **用途**：处理 bump 等外部事件的反应动画

* **核心特点**：


  * 必须位于 Root 直接子级（根级别，否则无法被系统识别）

  * 包含 `m_reactionTypes` 数组（存储触发条件）

  * 通过 `FindReactionEntry()` 函数查找触发

  * 支持 `m_forcedBlendIn` 强制混合时间（动画过渡更自然）

#### B. Sequence（普通序列）



* **图标**：紫色序列图标

* **代码**：`work::Sequence` (workspotTreeItems.h:324)

* **用途**：按顺序执行一组子节点动画（支持循环）

* **核心特点**：


  * 可嵌套在其他容器中（如 Reaction Sequence、Root）

  * 包含 `m_loopInfinitely` 布尔值（控制是否无限循环）

  * 支持 `m_category` 分类（用于多序列的概率选择）

#### C. Random List（随机列表）



* **图标**：列表容器图标

* **代码**：`work::RandomList` (workspotTreeItems.h:406)

* **用途**：从子节点中随机选择指定数量的动画播放

* **核心特点**：


  * `m_minClips` / `m_maxClips`：单次播放的动画数量范围

  * `m_weights`：子节点权重数组（控制随机概率）

  * `m_dontRepeatLastAnims`：避免连续播放同一动画

  * `m_pauseBetweenLength`：动画间的间隔暂停时长（可选）

### 2. 入口类型（绿色向上箭头）

#### Entry Anim（入口动画）



* **图标**：绿色带向上箭头

* **代码**：`work::EntryAnim` (workspotTreeItems.h:268)

* **标志位**：`SlowEnter | MoveToMotionAnim`

* **用途**：角色进入 workspot 时的过渡动画（含移动逻辑）

* **核心特点**：


  * 包含移动到目标位置的逻辑（自动适配 workspot 坐标）

  * `m_movementType`：移动类型（Walk/Jog/Sprint）

  * `m_orientationType`：朝向类型（Forward/TowardsObject）

  * `m_isSynchronized`：是否同步（多人协作场景）

### 3. 动画类型（绿色圆点）

#### Anim / AnimClip（普通动画）



* **图标**：绿色圆点

* **代码**：`work::AnimClip` (workspotTreeItems.h:14)

* **标志位**：`Animation`

* **用途**：播放单个独立动画片段

* **常见变体**：


  * `MotionAnimClip`：包含位移的动画（如行走时的手部动作）

  * `SyncAnimClip`：多人同步动画（如对话时的姿态配合）

  * `AnimClipWithItem`：带道具的动画（如吸烟、持武器）

### 4. 特殊动画类型

#### A. Look-at Driven Turn（视线驱动转身）



* **图标**：紫色转向图标

* **代码**：`work::LookAtDrivenTurn` (workspotTreeItems.h:57)

* **标志位**：`LookAtDrivenTurn`

* **用途**：根据角色视线方向自动匹配转身动画

* **核心特点**：


  * `m_turnAngle`：转身角度（从动画文件名提取，如 `turn_90_left`）

  * 系统自动选择最接近当前视线需求的动画片段

#### B. Pause（暂停）



* **图标**：蓝色暂停图标

* **代码**：`work::PauseClip` (workspotTreeItems.h:225)

* **标志位**：`Pause`

* **用途**：播放 idle 动画并暂停指定时长（序列中添加间隔）

* **核心特点**：


  * `m_timeMin` / `m_timeMax`：暂停时长范围（随机取值）

  * 本质是 "idle + 时间控制"，避免动画衔接生硬

#### C. Tag Node（标签节点）



* **图标**：标签图标

* **代码**：`work::TagNode` (workspotTreeItems.h:248)

* **标志位**：`TagNode`

* **用途**：作为动画跳转的目标标记点

* **核心特点**：


  * 无实际动画，仅用于逻辑标记

  * 通过 `JumpToEntry(entryTag)` 函数跳转至此节点

### 5. 退出类型（红色向下箭头）

#### A. Exit Anim（慢速退出）



* **图标**：红色带向下箭头

* **代码**：`work::ExitAnim` (workspotTreeItems.h:164)

* **标志位**：`SlowExit`

* **用途**：正常场景下退出 workspot 的动画（含离开移动）

* **核心特点**：


  * 包含从 workspot 位置离开的移动逻辑

  * 支持同步模式（多人同时退出）

  * 可随机选择（除非设置 `m_disableRandomExit`）

#### B. Fast Exit（快速退出）



* **图标**：红色快速退出图标

* **代码**：`work::FastExit` (workspotTreeItems.h:133)

* **标志位**：`FastExit | MotionAnim`

* **用途**：紧急场景退出（如战斗触发、恐惧反应）

* **核心特点**：


  * `m_forcedBlendIn`：强制混合时间（快速过渡无卡顿）

  * 立即中断当前所有动画，优先级最高

## 三、Entry 排布规则（核心！）

### 1. 层级结构规则（官方标准）



```
Root (IContainerEntry)

├── Reaction Sequence \[必须在根级别]

│   ├── Entry Anim（可选）

│   ├── Sequence / Random List（反应动画组）

│   │   ├── Anim（反应动作）

│   │   ├── Look-at Driven Turn（转身适配）

│   │   └── Pause（间隔）

│   └── Exit Anim / Fast Exit（反应后退出，可选）

├── Sequence \[主循环序列，支持多个]

│   ├── Entry Anim \[可选，单独入口]

│   ├── Anim / Random List \[循环核心内容]

│   ├── Pause \[可选，添加间隔]

│   └── Look-at Driven Turn \[可选，视线适配]

├── Exit Anim \[必须有至少一个]

└── Fast Exit \[可选但推荐，紧急退出]
```

### 2. 必须遵守的硬性规则

#### A. Reaction Sequence 位置强制要求

源码 `workspotResource.cpp:1499-1509` 明确：



```
work::WorkEntryId WorkspotTree::FindReactionEntry(CName reactionName) const

{

&#x20;   IContainerEntry\* cont = Cast\<IContainerEntry>(m\_rootEntry.Get());

&#x20;   // 只在根级别查找 Reaction Sequence！

&#x20;   for (THandleEntry>& record : cont->m\_list)

&#x20;   {

&#x20;       if (THandle\<ReactionSequence> reaction = Cast>(record))

&#x20;       {

&#x20;           if (reaction->ContainsReaction(reactionName))

&#x20;               return reaction->m\_id;

&#x20;       }

&#x20;   }

}
```

> **结论**
>
> ：Reaction Sequence 必须放在 Root 直接子级，否则 
>
> `FindReactionEntry()`
>
>  无法找到，反应动画失效！

#### B. Entry/Exit 节点配对规则



* 至少包含一个 Exit 节点（ExitAnim 或 FastExit），否则角色无法退出 workspot

* Entry Anim 是可选的（可直接从外部位置进入循环动画）

* 推荐组合：SlowExit（正常退出）+ FastExit（紧急退出），覆盖所有场景

#### C. 容器非空规则



* Sequence 必须包含至少一个子节点（否则无动画可执行）

* Random List 必须包含 ≥ `m_minClips` 个子节点（否则随机逻辑失效）

### 3. 推荐的排布顺序（CDPR 官方规范）

从编辑器截图可看出，CDPR 遵循 "功能分组" 原则，推荐顺序：



1. **Reaction Sequences（所有反应）**

* 按 bump 方向分组（如 BumpLeft、BumpRightFront）

* 按事件类型分组（如 Fear、Combat）

1. **Entry Anims（所有入口）**

* 按进入方向分组（如 Front、Left、Back）

* 按移动类型分组（如 Walk、Jog）

1. **Main Sequences（主循环序列）**

* idle 动画序列（基础姿态）

* 交互序列（如使用道具、查看环境）

1. **Exit Anims（所有退出）**

* 按离开方向分组（如 Front、Left）

1. **Fast Exits（紧急退出）**

* 按触发场景分组（如 Combat、Fear）

### 4. 嵌套规则（允许 / 禁止）

#### 允许嵌套



* Sequence 可包含：Anim、Pause、Look-at Driven Turn、Tag Node、子 Sequence

* Random List 可包含：Anim、Pause（不建议包含 Entry/Exit）

* Reaction Sequence 可包含任何类型（Entry、Anim、Exit 等）

#### 禁止嵌套



* Entry/Exit 节点应在容器顶层（避免深度嵌套，逻辑混乱）

* Reaction Sequence 禁止嵌套在其他容器中（必须根级别）

## 四、实际使用示例

### 示例 1：简单坐椅子 Workspot（基础场景）



```
Root

├── Entry Anim: walk\_to\_sit（走向椅子并坐下）

├── Sequence \[m\_loopInfinitely=true 无限循环]

│   ├── Anim: sit\_idle\_01（基础坐姿 idle）

│   ├── Pause (2-5 seconds)（随机暂停 2-5 秒）

│   ├── Random List（随机执行小动作）

│   │   ├── Anim: sit\_adjust\_01（调整坐姿）

│   │   ├── Anim: sit\_scratch\_01（挠痒）

│   │   └── Anim: sit\_look\_around\_01（环顾四周）

│   └── Pause (2-5 seconds)（再次暂停）

├── Exit Anim: sit\_to\_stand\_front（从前方起身离开）

├── Exit Anim: sit\_to\_stand\_left（从左侧起身离开）

└── Fast Exit: sit\_fast\_exit（紧急起身逃离）
```

### 示例 2：带 Reaction 的 Workspot（复杂交互场景，对应截图）



```
Root

├── Reaction Sequence \[BumpLeftBack]（被从左后方碰撞）

│   ├── Anim: bump\_left\_back\_reaction（左后碰撞反应动画）

│   └── （自动返回主循环序列）

├── Reaction Sequence \[BumpRightFront]（被从右前方碰撞）

│   └── Anim: bump\_right\_front\_reaction（右前碰撞反应动画）

├── Reaction Sequence \[Fear]（恐惧触发）

│   ├── Anim: stand\_fear\_start（恐惧起始姿态）

│   └── Fast Exit: fear\_run\_away（恐惧逃离）

├── Entry Anim: walk\_to\_stand\_2h（双手持物站立入口）

├── Sequence: idle\_stand\_2h\_front（正面双手持物 idle）

│   ├── Anim: stand\_2h\_check\_shoes（低头看鞋）

│   ├── Look-at Driven Turn（多个角度转身，适配视线）

│   └── Pause（随机间隔）

├── Sequence: idle\_stand\_2h\_phone（持物看手机序列）

│   └── Random List（随机手机交互动作）

│       ├── Anim: stand\_phone\_step\_forward（向前迈步看手机）

│       ├── Anim: stand\_phone\_shuffle（小碎步调整位置）

│       └── Anim: stand\_phone\_look（专注看手机）

├── Exit Anim: stand\_to\_walk（多个方向步行退出）

└── Fast Exit: None（无紧急退出，根据场景需求设置）
```

## 五、常见错误和注意事项

### ❌ 错误示例（必避坑）



1. **Reaction Sequence 嵌套错误**



```
Root

└── Sequence

&#x20;   └── Reaction Sequence \[BumpLeft]  ❌ 嵌套在 Sequence 中，无法被 FindReactionEntry() 找到！
```



1. **缺少 Exit 节点**



```
Root

├── Entry Anim

└── Sequence

&#x20;   └── Anim  ❌ 无 Exit 节点，角色无法退出 workspot！
```



1. **Entry/Exit 深度嵌套**



```
Sequence

└── Random List

&#x20;   └── Entry Anim  ❌ 入口动画嵌套过深，逻辑混乱且无法正常触发！
```

### ✅ 正确示例（参考标准）



```
Root

├── Reaction Sequence (根级别) ✓

├── Entry Anim (顶层容器) ✓

├── Sequence

│   ├── Anim ✓

│   └── Pause ✓

├── Exit Anim (顶层容器) ✓

└── Fast Exit (顶层容器) ✓
```

## 六、总结

### 核心逻辑闭环

Workspot 的 Entry 分类本质是 **动画状态机的完整生命周期**：



1. 入口（Entry Anim）→ 如何进入 workspot

2. 循环（Sequence/Random List + Anim）→ 在 workspot 中执行的动作

3. 反应（Reaction Sequence）→ 对外部事件的响应

4. 退出（Exit Anim / Fast Exit）→ 如何离开 workspot

### 最重要的排布规则（熟记！）



* ✅ Reaction Sequence 必须在根级别（系统查找依赖）

* ✅ Entry/Exit 节点应放在容器顶层（逻辑清晰，触发稳定）

* ✅ 至少包含一个 Exit 节点（避免角色 "卡" 在 workspot）

* ✅ 按功能分组排布（反应→入口→循环→退出，符合官方规范）

### 结构优势

遵循以上规则的 Workspot 具有：



1. 兼容性：适配游戏原生 WorkspotSystem，无触发 bug

2. 可维护性：层级清晰，便于后续修改和扩展

3. 灵活性：支持循环、随机、反应等复杂动画逻辑

4. 稳定性：避免因结构错误导致的动画失效或角色异常

> （注：文档部分内容可能由 AI 生成）