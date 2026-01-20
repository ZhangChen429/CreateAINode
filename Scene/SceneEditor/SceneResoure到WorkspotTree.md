
## SceneSolution：：Workspot

  | 特性     | External Workspot      | Internal Workspot        |
  |----------|------------------------|--------------------------|
  | 存储位置 | 独立的 .workspot 文件  | 嵌入在 .scenesolution 中 |
  | 引用方式 | 通过资源路径引用       | 直接包含数据             |
  | 复用性   | ✅ 可在多个场景复用    | ❌ 仅限当前场景          |
  | 编辑方式 | 修改 .workspot 文件    | 在场景编辑器中直接编辑   |
  | 版本控制 | 独立文件，便于管理     | 与场景绑定               |
  | 适用场景 | 通用动作（站立、坐下） | 场景特定动作             |
  | 数据来源 | m_modelWorkspot        | m_workspotData           |

#  SceneEditorResource到Workspot数据结构
```plantuml
@startuml SceneEditorResource 结构类图
skinparam class {
    BackgroundColor #f9f9f9
    BorderColor #222222
    ArrowColor #333333
}
skinparam rectangle {
    BackgroundColor #f0f8ff
    BorderColor #222222
}

class SceneEditorResource {
  + 文件名: .scenesolution
  - m_workspots: List<SceneWorkspot> 「核心数组」
}

class SceneWorkspot {
  - m_name: String 「工作点位名称」
  - m_workspotData: AbstractWorkspotData 「多态数据体」
  + DowngradeToInternalTree() : void 「核心方法：降级为内置树」
  + UpgradeToExternalReference() : void 「反向方法：升级为外部引用」
}

abstract class AbstractWorkspotData

class WorkspotData_ExternalWorkspotResource extends AbstractWorkspotData {
  - m_workspotResource: String 「外部文件路径」
  > 示例值: base\workspots\...\guard.workspot
  → 指向外部 .workspot 物理文件 📄
}

class WorkspotData_EmbeddedWorkspotTree extends AbstractWorkspotData {
  - m_workspotTree: WorkspotTree 「内嵌完整数据树」
  → 数据直接存储在当前 .scenesolution 文件内
}

class WorkspotTree {
  - AnimSets: 动画集配置
  - Sequences: 动作序列配置
  - Transitions: 状态过渡规则
  - State Machine: 状态机逻辑
}

SceneEditorResource "1" *-- "*" SceneWorkspot : 包含多个
SceneWorkspot "1" --> "1" AbstractWorkspotData : 持有 → 二选一实现
AbstractWorkspotData <|-- WorkspotData_ExternalWorkspotResource : 实现：外部引用
AbstractWorkspotData <|-- WorkspotData_EmbeddedWorkspotTree : 实现：内部嵌入 ⭐
WorkspotData_EmbeddedWorkspotTree "1" *-- "1" WorkspotTree : 强聚合(内嵌)
SceneWorkspot .-> WorkspotData_EmbeddedWorkspotTree : DowngradeToInternalTree() → 转换目标

rectangle "SceneWorkspot #1 (External 示例)" {
  note right: m_name = "bodyguard_stand"
  SceneWorkspot --> WorkspotData_ExternalWorkspotResource
}

rectangle "SceneWorkspot #2 (Internal 示例 ⭐)" {
  note right: m_name = "hanako_special_movement"
  SceneWorkspot --> WorkspotData_EmbeddedWorkspotTree
}
@enduml
```

上图可知，Internal Workspot 节点会直接嵌入 .scenesolution 文件内，而 External Workspot 节点则需要通过 m_workspotResource 属性引用外部 .workspot 文件。

同时无论内外部都有相应的转化方式，外部转为内部，内部转为外部

# ChangeWorkEvent对Workspot调用机制
```plantuml

@startuml ChangeWorkEvent 流程-极简纯排版版
skinparam default {
  BackgroundColor transparent
  BorderColor #2d3436
  FontColor #2d3436
  FontSize 12
}
skinparam rectangle {
  RoundCorner 5
  BackgroundColor #f8f9fa
}

' 标题
rectangle "ChangeWorkEvent 使用流程 (两者完全相同)" as Step0

' 流程节点
rectangle "WorkEntry { workspotId, entryId }" as Step1

rectangle "External Workspot" as Step2_1
rectangle "Internal Workspot" as Step2_2

rectangle "WorkspotData_ExternalWorkspotResource\n└─ m_workspotResource (TResRef)" as Step3_1
rectangle "WorkspotData_EmbeddedWorkspotTree\n└─ m_workspotTree (THandle)" as Step3_2

rectangle "加载 .workspot 文件" as Step4_1
rectangle "直接使用嵌入的树" as Step4_2

rectangle "WorkspotTree ✅\n├─ m_rootEntry (节点树)\n├─ m_finalAnimsets (动画集)\n├─ m_globalProps (道具)\n└─ 混合参数、转换动画等" as Step5

' 严格的流程指向
Step0 --> Step1
Step1 --> Step2_1
Step1 --> Step2_2
Step2_1 --> Step3_1
Step2_2 --> Step3_2
Step3_1 --> Step4_1
Step3_2 --> Step4_2
Step4_1 --> Step5
Step4_2 --> Step5
@enduml
```

  | 特性       | External Workspot                 | Internal Workspot                 |
  |------------|-----------------------------------|-----------------------------------|
  | 使用方式   | ✅ ChangeWorkEvent                | ✅ ChangeWorkEvent                |
  | 引用方式   | WorkEntry { workspotId, entryId } | WorkEntry { workspotId, entryId } |
  | 获取接口   | GetWorkspotTree()                 | GetWorkspotTree()                 |
  | 数据来源   | 加载 .workspot 文件               | 使用嵌入的 m_workspotTree         |
  | 运行时行为 | ✅ 完全相同                       | ✅ 完全相同                       |
  | 编译结果   | ✅ 完全相同                       | ✅ 完全相同                       |
  | 代码差异   | ❌ 无差异                         | ❌ 无差异                         |





$env:GOOGLE_GEMINI_BASE_URL="https://jeniya.cn"
$env:GEMINI_API_KEY="sk-5P0JVMtxkRn1SvjgByj2cdeVj16T8ryVut2H8pG7jI1AX4mM"

```plantuml

@startuml Workspot_Full_ClassDiagram
skinparam class {
    BackgroundColor #f9f9f9
    BorderColor #2c3e50
    ArrowColor #2c3e50
    FontName Microsoft YaHei
}
skinparam abstract {
    BackgroundColor #e8f4fd
    BorderColor #3498db
}

class SceneEditorResource {
    +red::DynArray~SceneWorkspot~ m_workspots
    +THandle~SceneDescriptor~ m_sceneDescriptor
    +red::DynArray~SceneActor~ m_actors
    +GetWorkspots() SceneWorkspot[]
    +CreateExternalWorkspot() SceneWorkspot
    +CreateInternalWorkspot() SceneWorkspot
}

class SceneWorkspot {
    +SceneWorkspotId m_id
    +CName m_name
    +THandle~WorkspotData~ m_workspotData
    +Bool IsExternal()
    +Bool IsBasedOnModel()
    +THandle~WorkspotTree~ GetWorkspotTree()
    +res::ResourcePath GetExternalResourcePath()
}

abstract class WorkspotData {
    +GetWorkspotParams() WorkspotParams*
    +GetWorkspotTree() WorkspotTree*
    +IsExternal() Bool*
}

class WorkspotData_ExternalWorkspotResource {
    +TResRef~WorkspotResource~ m_workspotResource
    +GetWorkspotParams() WorkspotParams
    +GetWorkspotTree() WorkspotTree
    +IsExternal() true
}

class WorkspotData_EmbeddedWorkspotTree {
    +THandle~WorkspotTree~ m_workspotTree
    +GetWorkspotParams() WorkspotParams
    +GetWorkspotTree() WorkspotTree
    +IsExternal() false
}

class WorkspotResource {
    +THandle~WorkspotTree~ m_workspotTree
    +OnPostLoad()
    +PreloadEntryAnimationsForPlayer()
}

class WorkspotTree {
    +THandle~IEntry~ m_rootEntry
    +Uint32 m_idCounter
    +red::TagList m_tags
    +red::DynArray~WorkspotAnimsetEntry~ m_finalAnimsets
    +red::DynArray~TransitionAnim~ m_customTransitionAnims
    +red::DynArray~WorkspotGlobalProp~ m_globalProps
    +TResAsyncRef~anim::Rig~ m_workspotRig
    +Float m_blendOutTime
    +CName m_animGraphSlotName
    +GetEntryVectors()
    +GetExitVectors()
    +FindEntryAnim()
    +FindExitAnim()
}

abstract class IEntry {
    +WorkEntryId m_id
    +Uint32 m_flags
    +CreateIterator() EntryIterator*
    +ContainEntry() Bool*
    +ForEachAnimation()*
}

class IContainerEntry {
    +CName m_idleAnim
    +red::DynArray~IEntry~ m_list
}

class WorkspotAnimsetEntry {
    +TResAsyncRef~anim::Rig~ m_rig
    +anim::AnimSetup m_animations
    +red::DynArray~TResRef~anim::AnimSet~~ m_loadingHandles
}

class WorkspotParams {
    +THandle~WorkspotTree~ m_tree
    +THandle~anim::AnimGraph~ m_animGraph
    +OriginId m_locId
    +GetEntryVectors()
    +GetExitVectors()
    +FindEntryAnim()
}

class ChangeWorkEvent {
    +WorkEntry m_work
    +Transform m_placementOffset
    +AnimationPivotPosition m_pivotPosition
    +scnb::events::AnimationInfo m_transitionAnimInfo
    +scnb::events::AnimationInfo m_gameplayAnimInfo
}

class WorkEntry {
    +SceneWorkspotId m_workspotId
    +WorkEntryId m_sequenceEntryId
}

' 关系定义
SceneEditorResource "1" *-- "0..*" SceneWorkspot : contains
SceneWorkspot "1" *-- "1" WorkspotData : has
WorkspotData <|-- WorkspotData_ExternalWorkspotResource : inherits
WorkspotData <|-- WorkspotData_EmbeddedWorkspotTree : inherits

WorkspotData_ExternalWorkspotResource "1" --> "1" WorkspotResource : references
WorkspotData_EmbeddedWorkspotTree "1" *-- "1" WorkspotTree : embeds
WorkspotResource "1" *-- "1" WorkspotTree : contains

WorkspotTree "1" *-- "1" IEntry : root
WorkspotTree "1" *-- "0..*" WorkspotAnimsetEntry : animsets
IEntry <|-- IContainerEntry : inherits
IContainerEntry "1" *-- "0..*" IEntry : children

WorkspotParams "1" --> "1" WorkspotTree : wraps
WorkspotParams "1" --> "0..1" WorkspotResource : optional graph

ChangeWorkEvent "1" *-- "1" WorkEntry : uses
WorkEntry "1" --> "1" SceneWorkspot : references by id
@enduml

```
# 类关系图

```plantuml
@startuml SceneEditor-Workspot 类关系图
' 全局样式配置
skinparam class {
    BackgroundColor #f9f9f9
    BorderColor #222222
    ArrowColor #333333
    FontName 微软雅黑
    FontSize 10
}
skinparam association {
    LineColor #333333
    FontSize 9
}
skinparam arrow {
    FontColor #666666
}

' ========== 顶层核心类 ==========
class SceneEditorResource {
    - m_workspots: red::DynArray<THandle<SceneWorkspot>>
    - m_sceneDescriptor: THandle<SceneDescriptor>
    - m_actors: red::DynArray<SceneActor>
    - m_ridAssocs: red::DynArray<RidAssoc>
}

class SceneWorkspot {
    - m_id: SceneWorkspotId
    - m_name: CName
    - m_workspotData: THandle<scn::WorkspotData>
    + IsExternal(): bool → m_workspotData->IsExternal()
    + GetWorkspotTree() → m_workspotData->GetWorkspotTree()
}

' ========== 抽象基类 ==========
abstract class WorkspotData {
    + {abstract} GetWorkspotParams()
    + {abstract} GetWorkspotTree()
    + {abstract} IsExternal(): bool
}

abstract class IEntry {
    - m_id: WorkEntryId
    - m_flags: Uint32
    + {abstract} CreateIterator()
    + {abstract} ContainEntry()
}

' ========== WorkspotData 子类 - 内部/外部实现 ==========
class WorkspotData_External {
    - m_workspotResource: TResRef<WorkspotRes>
    + IsExternal(): bool → true
}
class WorkspotData_Embedded {
    - m_workspotTree: THandle<WorkspotTree>
    + IsExternal(): bool → false
}

' ========== 资源/数据载体类 ==========
class WorkspotResource {
    note right: .workspot 外部文件
    - m_workspotTree: THandle<WorkspotTree>
}
class WorkspotTree {
    - m_rootEntry: THandle<IEntry>
    - m_finalAnimsets: DynArray<>
    - m_customTransitionAnims
    - m_globalProps
    - m_tags: TagList
    - m_workspotRig
    - m_blendOutTime
    - m_animGraphSlotName
}

' ========== IEntry 子类 - 树节点实现 ==========
class IdleSequence {
    note right: Container 容器节点
}
class AnimationEntry {
    note right: Leaf 叶子节点
}
class ReactionNode {
    note right: Tagged 带标签节点
}

' ========== 关联关系 + 多重度 + 说明 ==========
' 一对多 聚合关系
SceneEditorResource "0..*" --* "contains 包含" SceneWorkspot
' 一对一 关联关系
SceneWorkspot "1" -- "has 持有" WorkspotData
' 继承关系 (抽象类实现)
WorkspotData <|-- WorkspotData_External : External Type
WorkspotData <|-- WorkspotData_Embedded : Internal Type
' 引用/嵌入关系
WorkspotData_External -.-> WorkspotResource : references 引用
WorkspotData_Embedded -- WorkspotTree : embeds 直接嵌入
WorkspotResource --> WorkspotTree : 关联
' 树形结构关系
WorkspotTree "1" --* "contains 包含" WorkspotTree
WorkspotTree "1" -- "root 根节点" IEntry
' 继承关系 (IEntry子类)
IEntry <|-- IdleSequence
IEntry <|-- AnimationEntry
IEntry <|-- ReactionNode

@enduml
```
### ChangeWorkEvent的WorkspotLibrary


  | 类型              | 在 Outline 中     | 在 WorkspotLibraries 中           | 判断条件                                                  |
  |-------------------|-------------------|-----------------------------------|-----------------------------------------------------------|
  | Library Workspots | External Workspot | 按文件夹层级显示（如 chairs\sit\) | IsExternal() == true 且路径以 base\workspot_library\ 开头 |
  | Scene External    | External Workspot | _scene_not_in_workspot_library\   | IsExternal() == true 但路径不在 library 中                |
  | Scene Embedded    | Internal Workspot | _scene_embedded\                  | IsExternal() == false                                     |



# 深挖WorkspotTree
IEntry : public ISerializable 是一个纯抽象基类（接口类），也是所有「工作台 / 交互位 (Workspot)」动画节点的顶级父类；
## IEntry 是 WorkspotTree 动画树的「通用节点抽象」，是整个工作台 (Workspot) 动画逻辑体系的「最小执行单元 & 数据载体」



```plantuml
@startuml IEntry 继承层级完整类图
' 全局样式
skinparam class {
    BackgroundColor #f9f9f9
    BorderColor #222222
    ArrowColor #333333
    FontName 微软雅黑
    FontSize 10
}
skinparam groupInheritance 2
' ========== 抽象基类 ==========
abstract class IEntry <<Abstract>> {
    - m_id: WorkEntryId
    - m_flags: Uint32
    + {abstract} CreateIterator(): EntryIterator*
    + {abstract} ContainEntry(id): Bool
    + {abstract} GetFriendlyName(): String
    + {abstract} CreateCopy(): THandle<IEntry>
    + ForEachAnimation(fun)
    + ForEachNode(preFun, postFun)
    --
    **Flags 常量**
    Animation = 0x02
    FastExit = 0x04
    SlowExit = 0x08
    SlowEnter = 0x10
    Pause = 0x20
    Synchronized = 0x40
    TagNode = 0x80
    Reaction = 0x100
    LookAtDrivenTurn = 0x200
    HasItem = 0x2000
    MotionAnim = 0x8000
}
abstract class IContainerEntry <<Abstract>> {
    - m_idleAnim: CName
    - m_list: red::DynArray<THandle<IEntry>>
    + ContainEntry(id): Bool {override}
    + ForEachAnimation(fun) {override}
    + ForEachNode(preFun, postFun) {override}
}
' ========== 叶子节点（Leaf Entries）==========
package "Leaf Entries (叶子节点 - 直接执行动画)" #DDFFDD {
    
    class AnimClip {
        - m_animName: CName
        - m_blendOutTime: Float
        + CreateIterator(): EntryIterator*
        + ForEachAnimation(fun)
        + GetFriendlyName(): String
        --
        flags: Animation
    }
    class MotionAnimClip {
        + GetFriendlyName(): String
        --
        flags: Animation | MotionAnim
        用途: 带根运动的动画（行走/跑步）
    }
    class AnimClipWithItem {
        - m_itemActions: DynArray<IWorkspotItemAction>
        --
        flags: Animation | HasItem
        用途: 拿枪、吸烟等带道具动画
    }
    class SyncAnimClip {
        - m_slotName: CName
        - m_syncOffset: Transform
        --
        flags: Animation | Synchronized
        用途: 多角色同步动画（握手、对话）
    }
    class EntryAnim {
        - m_animName: CName
        - m_idleAnim: CName
        - m_slotName: CName
        - m_blendOutTime: Float
        - m_isSynchronized: Bool
        - m_syncOffset: Transform
        - m_movementType: move::MovementType
        - m_orientationType: move::MovementOrientationType
        --
        flags: SlowEnter | MoveToMotionAnim
        用途: 从外部进入 workspot
    }
    class SyncMasterEntryAnim {
        + AllowSync(asMaster): Bool {override}
        --
        强制 m_isSynchronized = true
        用途: 多人同步进入时的主控角色
    }
    class ExitAnim {
        - m_animName: CName
        - m_slotName: CName
        - m_idleAnim: CName
        - m_isSynchronized: Bool
        - m_stayOnNavmesh: Bool
        - m_snapZToNavmesh: Bool
        - m_disableRandomExit: Bool
        - m_syncOffset: Transform
        - m_movementType: move::MovementType
        --
        flags: SlowExit
        用途: 正常退出 workspot（带动画）
    }
    class FastExit {
        - m_animName: CName
        - m_forcedBlendIn: Float
        - m_movementType: move::MovementType
        --
        flags: FastExit | MotionAnim
        用途: 被打断时的紧急退出
    }
    class LookAtDrivenTurn {
        - m_turnAnimName: CName
        - m_turnAngle: Int32
        - m_blendTime: Float
        --
        flags: LookAtDrivenTurn
        用途: 根据视线方向自动转身
    }
    class PauseClip {
        - m_timeMin: Float
        - m_timeMax: Float
        - m_blendOutTime: Float
        --
        flags: Pause
        用途: 序列中插入随机长度暂停
    }
    class TagNode {
        - m_tag: CName
        --
        flags: TagNode
        用途: 命名跳转点，可通过 tag 查找
    }
}
' ========== 容器节点（Container Entries）==========
package "Container Entries (容器节点 - 包含子节点)" #DDDDFF {
    class Sequence {
        - m_previousLoopInfinitely: Bool
        - m_loopInfinitely: Bool
        - m_category: WorkspotCategory
        --
        用途: 按顺序播放子节点
        例如: idle → look_around → idle → scratch_head
    }
    class ReactionSequence {
        - m_reactionTypes: DynArray<RecordID>
        - m_forcedBlendIn: Float
        - m_facialKeyWeight: Float
        - m_mainEmotionalState: CName
        - m_emotionalExpression: CName
        - m_facialIdleMaleAnimation: CName
        - m_facialIdleKey_MaleAnimation: CName
        - m_facialIdleFemaleAnimation: CName
        - m_facialIdleKey_FemaleAnimation: CName
        --
        flags: Reaction
        用途: 响应外部事件（被攻击、惊吓）
        **不能直接选择，仅通过 Reaction 系统触发**
    }
    class ConditionalSequence {
        - m_multipleConditionOperator: LogicalOperation
        - m_conditionList: DynArray<IWorkspotCondition>
        --
        用途: 满足条件才播放序列
        例如: 只在夜间播放、只对特定实体播放
    }
    class RandomList {
        - m_minClips: Int8
        - m_maxClips: Int8
        - m_dontRepeatLastAnims: Int8
        - m_pauseBetweenLength: Float
        - m_pauseLengthDeviation: Float
        - m_pauseBlendOutTime: Float
        - m_weights: DynArray<Float>
        --
        MAX_REPEAT_HISTORY = 5
        用途: 随机选择子节点播放
        例如: 从 [scratch_head, look_around, yawn] 随机选 3-5 次
    }
    class Selector {
        --
        用途: 单次随机选择（不循环）
        权重选择，只播放一次
    }
}
' ========== 继承关系 ==========
IEntry <|-- IContainerEntry : 继承
IEntry <|-- AnimClip
IEntry <|-- EntryAnim
IEntry <|-- ExitAnim
IEntry <|-- FastExit
IEntry <|-- LookAtDrivenTurn
IEntry <|-- PauseClip
IEntry <|-- TagNode
AnimClip <|-- MotionAnimClip
AnimClip <|-- AnimClipWithItem
AnimClip <|-- SyncAnimClip
EntryAnim <|-- SyncMasterEntryAnim
IContainerEntry <|-- Sequence
IContainerEntry <|-- RandomList
Sequence <|-- ReactionSequence
Sequence <|-- ConditionalSequence
RandomList <|-- Selector
' ========== 组合关系 ==========
IContainerEntry "1" *-- "0..*" IEntry : m_list\n包含子节点
' ========== 说明注释 ==========
note right of IEntry
  **IEntry 是所有 Workspot 节点的基类**
  
  所有节点都有：
  • m_id: 唯一标识符
  • m_flags: 节点类型标志
  
  节点分为两大类：
  1. Leaf Entries (叶子) - 直接执行动画
  2. Container Entries (容器) - 组织子节点
end note
note right of IContainerEntry
  **容器节点的共同特性**
  
  • m_idleAnim: 默认 idle 动画
  • m_list: 子节点列表
  
  可以包含任意类型的 IEntry 子节点
  （包括其他容器节点，形成树结构）
end note
note bottom of AnimClip
  **AnimClip 是最基础的动画节点**
  
  只播放一个动画片段
  可以被扩展为：
  • MotionAnimClip (带位移)
  • AnimClipWithItem (带道具)
  • SyncAnimClip (多角色同步)
end note
note bottom of Sequence
  **Sequence 是最常用的容器节点**
  
  顺序播放子节点，可以循环
  
  扩展版本：
  • ReactionSequence: 响应外部事件
  • ConditionalSequence: 条件判断
end note
@enduml

```
~~~
  IEntry (抽象基类)
  ├── 直接子类（叶子节点）
  │   ├── AnimClip                    ← 基础动画片段
  │   │   ├── MotionAnimClip          ← 带位移动画
  │   │   ├── AnimClipWithItem        ← 带道具动画
  │   │   └── SyncAnimClip            ← 同步动画
  │   ├── EntryAnim                   ← 进入动画
  │   │   └── SyncMasterEntryAnim     ← 同步主入口
  │   ├── ExitAnim                    ← 退出动画
  │   ├── FastExit                    ← 快速退出
  │   ├── LookAtDrivenTurn            ← 视线转向
  │   ├── PauseClip                   ← 暂停节点
  │   └── TagNode                     ← 标签节点
  │
  └── IContainerEntry (抽象容器基类)
      ├── Sequence                    ← 序列（顺序播放）
      │   ├── ReactionSequence        ← 反应序列
      │   └── ConditionalSequence     ← 条件序列
      └── RandomList                  ← 随机列表
          └── Selector                ← 选择器
~~~
  3. 类与用途对照表

  | 你原图中的名称 | 实际类名         | 继承自          | 用途                               |
  |----------------|------------------|-----------------|------------------------------------|
  | IdleSequence   | Sequence         | IContainerEntry | 顺序播放子节点，可以包含 idle 动画 |
  | AnimationEntry | AnimClip         | IEntry          | 播放单个动画片段                   |
  | ReactionNode   | ReactionSequence | Sequence        | 响应外部事件（攻击、惊吓等）       |
  | ReactionNode   | TagNode          | IEntry          | 标记跳转点，可通过 tag 查找        |
