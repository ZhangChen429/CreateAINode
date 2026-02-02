# SmartObject 实战教程：AI 坐椅子完整示例

> **目标**：从零搭建一个完整的 SmartObject 系统，让 AI 自动找到椅子并坐下
>
> **难度**：⭐⭐☆☆☆（初级-中级）
>
> **预计时间**：30-45 分钟

---

## 📋 前提条件

### 必需插件（Project Settings → Plugins）

- ✅ **SmartObjects** - 核心插件
- ✅ **GameplayBehaviorSmartObjects** - 行为框架集成
- ✅ **GameplayBehaviors** - 行为系统基础

### 必需资源

- 椅子模型：`SM_Chair`
- AI 角色蓝图：`BP_AICharacter`（带 AI Controller）
- 动画资源：
  - `AM_SitDown` - 坐下动画蒙太奇
  - `AM_StandUp` - 起身动画蒙太奇
  - `Anim_Sitting_Idle` - 坐姿待机动画

---

## 第一步：创建 GameplayBehavior Config（坐下行为）

### 1.1 创建行为蓝图

**路径**：`Content/SmartObjects/Behaviors/`

**操作步骤**：

1. 右键 Content Browser → **Blueprint Class**
2. 搜索并选择父类：**GameplayBehavior**
3. 命名：`BP_SitBehavior`

---

### 1.2 实现坐下逻辑（BP_SitBehavior）

打开 `BP_SitBehavior`，在 **事件图表** 中实现以下逻辑：

#### 节点连接图

```
┌────────────────────────────────────────────────────────────────┐
│ Event OnTriggered (from GameplayBehavior)                      │
└────┬───────────────────────────────────────────────────────────┘
     │
     ├─ 1. 获取插槽信息
     │   └─ Get Smartobject Subsystem (World Context Object: Self)
     │        └─ Get Slot View (Claim Handle: Avatar Claim Handle)
     │             └─ Get Slot Definition
     │                  └─ Get Slot World Transform (Owner Transform)
     │                       └─ Break Transform (Location, Rotation)
     │
     ├─ 2. 移动到坐姿位置
     │   └─ Set Actor Location and Rotation
     │        ├─ Target: Avatar (from OnTriggered)
     │        ├─ New Location: Location (from Transform)
     │        └─ New Rotation: Rotation (from Transform)
     │
     ├─ 3. 播放坐下动画
     │   └─ Play Anim Montage
     │        ├─ Skeletal Mesh Component: Avatar → Get Mesh
     │        ├─ Montage: AM_SitDown
     │        └─ Play Rate: 1.0
     │
     ├─ 4. 等待坐下完成
     │   └─ Delay (5.0 seconds)
     │
     ├─ 5. 播放起身动画
     │   └─ Play Anim Montage (AM_StandUp)
     │
     └─ 6. 结束行为
         └─ End Behavior
              ├─ Avatar: Avatar
              └─ Result: Success
```

---

#### 详细参数配置

**节点 1：获取插槽变换**

```cpp
// Get Smartobject Subsystem
World Context Object: Self

// Get Slot View
Claim Handle: 从 OnTriggered 的 "Smartobject Claim Handle" 引脚获取

// Get Slot World Transform
Owner Transform: 从 Slot View → Get Smart Object Component → Get Owner → Get Actor Transform
```

**节点 2：移动角色**

```cpp
// Set Actor Location and Rotation
Target: Avatar (从 OnTriggered Event 引脚)
New Location: Slot Transform.Location
New Rotation: Slot Transform.Rotation
Sweep: false  // 不进行碰撞检测
Teleport: true  // 瞬移到位置
```

**节点 3：播放动画**

```cpp
// Play Anim Montage
In Skeletal Mesh Component: Avatar → Get Mesh
Montage to Play: AM_SitDown
Play Rate: 1.0
Starting Position: 0.0
Starting Section: None
```

**节点 4-6：等待和结束**

```cpp
// Delay
Duration: 5.0  // 坐着持续 5 秒

// Play Anim Montage (StandUp)
Montage to Play: AM_StandUp

// End Behavior
Avatar: Avatar
Result: Success
```

---

### 1.3 创建 GameplayBehavior Config

**路径**：`Content/SmartObjects/Behaviors/`

**操作步骤**：

1. 右键 Content Browser → **Blueprint Class**
2. 父类：**GameplayBehaviorConfig**
3. 命名：`GBC_SitOnChair`

**配置 GBC_SitOnChair**（Class Defaults）：

```
Behavior Class: BP_SitBehavior
```

---

## 第二步：创建 SmartObject Definition（椅子定义）

### 2.1 创建 Definition 资产

**路径**：`Content/SmartObjects/Definitions/`

**操作步骤**：

1. 右键 Content Browser → **Smart Object → Smart Object Definition**
2. 命名：`DA_Chair`

---

### 2.2 配置 DA_Chair

打开 `DA_Chair`，进行以下配置：

#### 基础信息

```
┌─────────────────────────────────────────────────────────┐
│ Smart Object Definition - DA_Chair                      │
├─────────────────────────────────────────────────────────┤
│ ▼ Smart Object                                          │
│   ├─ Activity Tags:                                     │
│   │    └─ + SmartObject.Sit                             │
│   │                                                     │
│   ├─ User Tag Filter: (Empty)                           │
│   │                                                     │
│   ├─ Activity Tags Merging Policy: Combine              │
│   │                                                     │
│   └─ User Tags Filtering Policy: Combine                │
└─────────────────────────────────────────────────────────┘
```

**设置步骤**：

1. **Activity Tags** - 点击 `+` 添加标签
   - 输入：`SmartObject.Sit`
   - 回车确认

2. **User Tag Filter** - 留空（允许所有 AI 使用）

3. **策略保持默认**：
   - Activity Tags Merging Policy: `Combine`
   - User Tags Filtering Policy: `Combine`

---

#### 配置插槽（Slots）

点击 **Slots** 数组的 `+` 按钮，添加 **Slot[0]**

```
┌─────────────────────────────────────────────────────────┐
│ ▼ Slots                                                 │
│   └─ ▼ [0]                                              │
│       ├─ Name: "Seat"                                   │
│       ├─ Enabled: ✅ true                               │
│       │                                                 │
│       ├─ Offset: X=50.0, Y=0.0, Z=45.0                  │
│       ├─ Rotation: Pitch=0, Yaw=180, Roll=0             │
│       │                                                 │
│       ├─ Activity Tags: (Empty - 继承父对象)             │
│       ├─ User Tag Filter: (Empty)                       │
│       │                                                 │
│       ├─ ▼ Behavior Definitions                         │
│       │   └─ ▼ [0] GameplayBehaviorSmartObjectBehavior  │
│       │       └─ Gameplay Behavior Config: GBC_SitOnChair│
│       │                                                 │
│       └─ ▼ Definition Data                              │
│           └─ (暂时为空)                                  │
└─────────────────────────────────────────────────────────┘
```

**详细配置步骤**：

**2.2.1 基础属性**

```
Name: Seat  // 插槽名称（方便识别）
Enabled: ✅  // 初始启用状态
```

**2.2.2 位置和旋转**

```
Offset:
  X: 50.0   // 向前偏移 50cm（椅子坐垫位置）
  Y: 0.0    // 左右居中
  Z: 45.0   // 向上 45cm（坐姿高度）

Rotation:
  Pitch: 0    // 不俯仰
  Yaw: 180    // 转身 180 度（背对椅背）
  Roll: 0     // 不翻滚
```

**💡 坐标系说明**：
- X 轴：椅子前方（红色箭头）
- Y 轴：椅子右侧（绿色箭头）
- Z 轴：椅子上方（蓝色箭头）

**2.2.3 添加行为定义**

1. 找到 **Behavior Definitions** 数组
2. 点击 `+` 添加元素
3. 类型选择：`GameplayBehaviorSmartObjectBehaviorDefinition`
4. 展开 **[0]**
5. **Gameplay Behavior Config** 选择：`GBC_SitOnChair`

---

### 2.3 添加入口注解（Entrance Annotation）

为了让 AI 知道从哪里接近椅子，需要添加入口注解。

**步骤**：

1. 在 **Slot[0]** 的 **Definition Data** 数组点击 `+`
2. 类型选择：`SmartObjectSlotEntranceAnnotation`
3. 展开配置：

```
┌─────────────────────────────────────────────────────────┐
│ ▼ Definition Data                                       │
│   └─ ▼ [0] SmartObjectSlotEntranceAnnotation            │
│       ├─ ▼ Entries                                      │
│       │   └─ ▼ [0]                                      │
│       │       ├─ Offset: X=-100.0, Y=0.0, Z=0.0         │
│       │       ├─ Rotation: Pitch=0, Yaw=0, Roll=0       │
│       │       ├─ Tags: SmartObject.Entrance.Front       │
│       │       └─ Selection Priority: 1.0                │
│       │                                                 │
│       ├─ bTrackSlotTransform: ✅ true                   │
│       └─ bCheckSlotLocationOverlap: ✅ true             │
└─────────────────────────────────────────────────────────┘
```

**参数说明**：

```
Offset: X=-100.0, Y=0.0, Z=0.0
  → 入口在椅子前方 100cm（负 X 方向）

Rotation: Pitch=0, Yaw=0, Roll=0
  → 面向椅子（不旋转）

Tags: SmartObject.Entrance.Front
  → 标记为"前方入口"

bTrackSlotTransform: true
  → 入口位置跟随椅子移动

bCheckSlotLocationOverlap: true
  → 检查插槽位置是否被占用
```

---

### 2.4 （可选）添加碰撞检测注解

为了防止 AI 坐在已有人的椅子上，可以添加碰撞注解。

**步骤**：

1. 在 **Definition Data** 数组点击 `+`
2. 类型选择：`SmartObjectAnnotation_SlotUserCollision`
3. 配置：

```
┌─────────────────────────────────────────────────────────┐
│ [1] SmartObjectAnnotation_SlotUserCollision             │
│   ├─ Offset: X=0.0, Y=0.0, Z=0.0                        │
│   ├─ Rotation: Pitch=0, Yaw=0, Roll=0                   │
│   │                                                     │
│   ├─ ▼ Capsule                                          │
│   │   ├─ Radius: 40.0                                   │
│   │   └─ Half Height: 50.0                              │
│   │                                                     │
│   └─ bTrackSlotTransform: ✅ true                       │
└─────────────────────────────────────────────────────────┘
```

这会在插槽位置创建一个检测胶囊体，防止多个 AI 同时坐。

---

### 2.5 预览配置（可选但推荐）

在 **Preview Data** 中设置预览：

```
┌─────────────────────────────────────────────────────────┐
│ ▼ Preview Data                                          │
│   ├─ Object Mesh Path: SM_Chair                         │
│   ├─ User Actor Class: BP_AICharacter                   │
│   └─ User Validation Filter Class: (None)               │
└─────────────────────────────────────────────────────────┘
```

这样可以在 Definition 编辑器中直接预览效果。

---

## 第三步：在关卡中放置椅子

### 3.1 创建椅子 Blueprint Actor

**路径**：`Content/SmartObjects/Actors/`

**操作步骤**：

1. 右键 Content Browser → **Blueprint Class**
2. 父类：**Actor**
3. 命名：`BP_Chair`

---

### 3.2 配置 BP_Chair 组件

打开 `BP_Chair`，添加以下组件：

```
BP_Chair (Actor)
├─ DefaultSceneRoot
│   ├─ StaticMeshComponent (SM_Chair)  ← 添加
│   │    └─ Static Mesh: SM_Chair
│   │    └─ Collision: BlockAll
│   │
│   └─ SmartObjectComponent            ← 添加
│        └─ Definition: DA_Chair
```

**详细步骤**：

**3.2.1 添加 Static Mesh**

1. 点击 **Add Component** → **Static Mesh**
2. 重命名为 `ChairMesh`
3. 在 Details 面板：
   - **Static Mesh**: 选择 `SM_Chair`
   - **Collision Presets**: `BlockAll`

**3.2.2 添加 SmartObject Component**

1. 点击 **Add Component** → **Smart Object Component**
2. 在 Details 面板：
   - **Definition → Definition Asset**: 选择 `DA_Chair`

**3.2.3 调整相对位置（如果需要）**

确保 Static Mesh 的原点在椅子底部中心，这样 Slot Offset 才准确。

---

### 3.3 放置到关卡

1. 编译并保存 `BP_Chair`
2. 拖拽 `BP_Chair` 到关卡场景中
3. 在场景中摆放几把椅子（测试用）

---

## 第四步：配置 AI 角色

### 4.1 确保 AI 角色有 AI Controller

打开 `BP_AICharacter`，检查：

```
┌─────────────────────────────────────────────────────────┐
│ BP_AICharacter (Class Defaults)                         │
├─────────────────────────────────────────────────────────┤
│ ▼ Pawn                                                  │
│   ├─ Auto Possess AI: Placed in World or Spawned        │
│   └─ AI Controller Class: BP_AIController               │
└─────────────────────────────────────────────────────────┘
```

如果没有 AI Controller，创建一个：

1. 右键 → **Blueprint Class** → **AIController**
2. 命名：`BP_AIController`
3. 在 `BP_AICharacter` 中设置 `AI Controller Class = BP_AIController`

---

### 4.2 （可选）添加用户标签

如果椅子的 `User Tag Filter` 有要求，需要给 AI 添加标签：

```
BP_AICharacter (Components)
├─ GameplayTagComponent (添加此组件)
│   └─ Gameplay Tags:
│        └─ Character.Humanoid  // 示例标签
```

---

## 第五步：创建行为树

### 5.1 创建行为树资产

**路径**：`Content/AI/BehaviorTrees/`

**操作步骤**：

1. 右键 Content Browser → **Artificial Intelligence → Behavior Tree**
2. 命名：`BT_FindAndSitOnChair`

---

### 5.2 创建黑板（Blackboard）

**路径**：`Content/AI/BehaviorTrees/`

**操作步骤**：

1. 右键 → **Artificial Intelligence → Blackboard**
2. 命名：`BB_AICharacter`

**添加黑板键**：

打开 `BB_AICharacter`，添加以下键：

```
┌─────────────────────────────────────────────────────────┐
│ Blackboard: BB_AICharacter                              │
├─────────────────────────────────────────────────────────┤
│ Keys:                                                   │
│   ├─ [0] SmartObjectClaimHandle                         │
│   │      Type: SmartObjectClaimHandle                   │
│   │      Instance Synced: ✅                            │
│   │                                                     │
│   └─ [1] TargetLocation (可选 - 用于调试)                │
│          Type: Vector                                   │
└─────────────────────────────────────────────────────────┘
```

---

### 5.3 配置行为树节点

打开 `BT_FindAndSitOnChair`，构建以下节点结构：

```
Root
└─ Sequence
    ├─ BTTask_FindAndUseGameplayBehaviorSmartObject
    │   ├─ Activity Requirements: SmartObject.Sit
    │   ├─ Claim Priority: Normal
    │   ├─ Radius: 1000.0
    │   └─ Blackboard Key: SmartObjectClaimHandle
    │
    └─ Wait (可选 - 重复使用)
         └─ Wait Time: 2.0
```

**详细节点配置**：

#### 节点 1：BTTask_FindAndUseGameplayBehaviorSmartObject

**Details 面板配置**：

```
┌─────────────────────────────────────────────────────────┐
│ Find And Use Gameplay Behavior Smart Object             │
├─────────────────────────────────────────────────────────┤
│ ▼ Smart Objects                                         │
│   ├─ Activity Requirements:                             │
│   │    └─ Query: Match Tag                              │
│   │         └─ Tag: SmartObject.Sit                     │
│   │                                                     │
│   ├─ Claim Priority: Normal                             │
│   │                                                     │
│   ├─ ▼ EQS Request:                                     │
│   │   └─ Query Template: (None - 使用 Fallback Radius)  │
│   │                                                     │
│   └─ Fallback Radius: 1000.0                            │
│                                                         │
│ ▼ Node                                                  │
│   └─ Node Name: "Find and Sit on Chair"                 │
└─────────────────────────────────────────────────────────┘
```

**Activity Requirements 配置步骤**：

1. 展开 **Activity Requirements**
2. 点击 **Query** 下拉菜单
3. 选择：**Match Tag**（或 **Any Tags Match**）
4. 在 **Tag** 字段输入：`SmartObject.Sit`

**其他参数**：

```
Claim Priority: Normal
  → AI 声明插槽的优先级

Fallback Radius: 1000.0
  → 搜索半径（1000cm = 10m）
  → 如果没有配置 EQS，使用这个简单半径查询

EQS Request: None
  → 可选的环境查询系统（高级用法）
  → 初学者留空，使用 Fallback Radius 即可
```

---

#### 节点 2：Wait（可选 - 重复循环）

```
Wait Time: 2.0
  → 等待 2 秒后重新查找椅子（测试用）

Random Deviation: 0.5
  → 随机偏差 ±0.5 秒
```

如果只想坐一次，可以删除 Wait 节点，或者使用：

```
Root
└─ Sequence
    └─ BTTask_FindAndUseGameplayBehaviorSmartObject
```

---

### 5.4 绑定黑板到行为树

1. 选择 **Root** 节点
2. **Details** 面板 → **Blackboard Asset**: `BB_AICharacter`

---

## 第六步：启动行为树

### 6.1 在 AI Controller 中运行行为树

打开 `BP_AIController`，在 **Event Graph** 中添加：

```
Event On Possess
  └─ Run Behavior Tree
       └─ BTAsset: BT_FindAndSitOnChair
```

**详细步骤**：

1. 右键 → **Event On Possess**
2. 拖出 → 搜索 **Run Behavior Tree**
3. **BTAsset** 选择：`BT_FindAndSitOnChair`

---

### 6.2 （可选）调试日志

添加日志节点方便调试：

```
Event On Possess
  ├─ Print String ("AI Possessed, Starting Behavior Tree")
  └─ Run Behavior Tree (BT_FindAndSitOnChair)
```

---

## 第七步：测试和调试

### 7.1 放置 AI 到场景

1. 拖拽 `BP_AICharacter` 到关卡
2. 确保椅子在 AI 的搜索半径内（10m）

---

### 7.2 启用调试可视化

**方法 1：控制台命令**

按 **~** 键打开控制台，输入：

```
showdebug SmartObject
```

这会显示：
- 所有 SmartObject 的位置（球体）
- 插槽状态（绿色=Free，红色=Claimed/Occupied）
- 入口位置（箭头）

**方法 2：编辑器可视化**

在场景中选择 `BP_Chair`，你应该看到：
- 插槽位置（小球体）
- 入口位置（箭头）
- 调试线条

---

### 7.3 运行测试

1. 点击 **Play** 或 **Simulate**
2. 观察 AI 行为：

**预期流程**：

```
Step 1: AI 生成
  └─ AI Controller 自动运行行为树

Step 2: 查找椅子
  └─ BTTask 执行空间查询
  └─ 找到最近的可用椅子

Step 3: 声明插槽
  └─ Subsystem 将插槽标记为 Claimed

Step 4: 导航到入口
  └─ AI 移动到椅子前方 100cm

Step 5: 使用插槽
  └─ 标记为 Occupied
  └─ 执行 BP_SitBehavior
       ├─ 瞬移到坐姿位置
       ├─ 播放坐下动画
       ├─ 等待 5 秒
       └─ 播放起身动画

Step 6: 释放插槽
  └─ 插槽恢复为 Free
  └─ 行为树返回 Success

Step 7: (如果有 Wait 节点) 重复
```

---

### 7.4 常见问题排查

#### 问题 1：AI 不移动

**检查清单**：

- ✅ AI 有 AI Controller？
- ✅ 行为树正在运行？（Event On Possess 中调用了 Run Behavior Tree）
- ✅ 黑板绑定正确？
- ✅ 场景中有 Nav Mesh？（P 键显示）
- ✅ 椅子在搜索半径内（1000cm = 10m）？

**调试方法**：

```
在 AI Controller 中添加日志：

Event On Possess
  ├─ Print String ("AI Possessed")
  └─ Run Behavior Tree
       └─ 后续添加 Print String ("Behavior Tree Started")
```

---

#### 问题 2：找不到椅子

**检查清单**：

- ✅ `DA_Chair` 的 **Activity Tags** 包含 `SmartObject.Sit`？
- ✅ 行为树节点的 **Activity Requirements** 是 `SmartObject.Sit`？（拼写一致）
- ✅ `BP_Chair` 的 **SmartObjectComponent** 引用了 `DA_Chair`？
- ✅ 椅子在 AI 附近？

**调试方法**：

```
控制台输入：
showdebug SmartObject

查看：
- 椅子是否显示绿色球体（已注册）
- 插槽是否可见
```

---

#### 问题 3：找到了椅子但不坐

**检查清单**：

- ✅ `DA_Chair` 的 **Behavior Definitions** 配置了 `GBC_SitOnChair`？
- ✅ `GBC_SitOnChair` 的 **Behavior Class** 是 `BP_SitBehavior`？
- ✅ `BP_SitBehavior` 正确实现了逻辑？
- ✅ 动画蒙太奇资源存在？

**调试方法**：

在 `BP_SitBehavior` 的 **Event OnTriggered** 开头添加：

```
Print String ("Sit Behavior Triggered!")
```

---

#### 问题 4：坐下位置不对

**检查清单**：

- ✅ `DA_Chair` 的 **Slot Offset** 正确？
  - 正确示例：`X=50, Y=0, Z=45`
  - 错误示例：`X=500, Y=0, Z=0`（太远）
- ✅ `BP_Chair` 的坐标轴方向正确？
  - 红色（X）应该朝向椅子前方
  - 蓝色（Z）应该朝上

**调试方法**：

1. 在场景中选择 `BP_Chair`
2. 查看 Gizmo 箭头方向
3. 如果不对，旋转椅子或调整 Static Mesh 的相对变换

---

#### 问题 5：动画不播放

**检查清单**：

- ✅ 动画蒙太奇资源正确？（`AM_SitDown`, `AM_StandUp`）
- ✅ 动画蒙太奇与角色骨骼匹配？
- ✅ `BP_SitBehavior` 中获取了正确的 Skeletal Mesh Component？

**调试方法**：

在 **Play Anim Montage** 节点后添加：

```
Print String (Concat:
  "Playing Animation: "
  + Get Display Name(Montage)
)
```

---

## 第八步：进阶配置（可选）

### 8.1 添加前置条件（Preconditions）

限制只有特定距离内的 AI 才能使用：

**在 DA_Chair 中**：

1. 展开 **Preconditions**
2. 点击 **+** 添加条件
3. 类型选择：`WorldCondition_SmartObjectActorDistance`
4. 配置：

```
┌─────────────────────────────────────────────────────────┐
│ Preconditions                                           │
│   └─ [0] WorldCondition_SmartObjectActorDistance        │
│       ├─ Distance Check: Distance3D                     │
│       ├─ Operator: Less or Equal                        │
│       ├─ Distance: 500.0                                │
│       └─ bInvert: ✗ false                               │
└─────────────────────────────────────────────────────────┘
```

现在只有距离 ≤ 5m 的 AI 才能找到这把椅子。

---

### 8.2 添加多个入口

如果椅子可以从多个方向接近：

在 **SmartObjectSlotEntranceAnnotation** 的 **Entries** 数组中添加：

```
Entries:
  ├─ [0] 前方入口
  │   └─ Offset: X=-100, Y=0, Z=0
  │
  ├─ [1] 左侧入口
  │   └─ Offset: X=0, Y=-100, Z=0
  │
  └─ [2] 右侧入口
      └─ Offset: X=0, Y=100, Z=0
```

AI 会自动选择最近的入口。

---

### 8.3 使用 EQS 进行高级查询

创建 **Environment Query**：

**路径**：`Content/AI/EQS/`

1. 右键 → **Artificial Intelligence → Environment Query**
2. 命名：`EQS_FindChair`

**配置 EQS_FindChair**：

```
Root
└─ Actors Of Class
    ├─ Search Center: Querier
    ├─ Search Radius: 1000.0
    ├─ Actor Class: BP_Chair
    │
    └─ Tests:
        ├─ Distance (Weight: 1.0)
        │    └─ Test Purpose: Filter and Score
        │
        └─ Dot (Weight: 0.5)
             └─ Test Purpose: Score Only
```

在行为树中使用：

```
BTTask_FindAndUseGameplayBehaviorSmartObject
  └─ EQS Request:
       └─ Query Template: EQS_FindChair
```

---

### 8.4 添加运行时标签

在坐下时添加标签，方便其他系统查询：

**在 BP_SitBehavior 中**：

```
Event OnTriggered
  ├─ ... (移动、动画)
  │
  ├─ Add Tag to Slot
  │   ├─ Claim Handle: (from Event)
  │   └─ Tag: Character.State.Sitting
  │
  ├─ Delay (5.0)
  │
  ├─ Remove Tag from Slot
  │   └─ Tag: Character.State.Sitting
  │
  └─ End Behavior
```

---

## 第九步：优化和最佳实践

### 9.1 性能优化

**避免频繁查询**：

```
行为树中添加 Cooldown Decorator：

Root
└─ Sequence
    └─ Cooldown (3.0 seconds)
         └─ BTTask_FindAndUseSmartObject
```

**批量操作**：

如果场景中有很多 AI，使用 Mass AI 框架代替传统行为树。

---

### 9.2 代码复用

创建通用的 SmartObject 行为树：

```
BT_UseSmartObject_Generic
  └─ Sequence
      └─ BTTask_FindAndUseGameplayBehaviorSmartObject
           └─ Activity Requirements: (作为黑板键)
```

不同 AI 可以传入不同的 Activity Tag。

---

### 9.3 错误处理

在行为树中添加 Fallback：

```
Root
└─ Selector
    ├─ Sequence [尝试使用 SmartObject]
    │   └─ BTTask_FindAndUseSmartObject
    │
    └─ Sequence [失败后的备选行为]
        └─ Wait (5.0)
```

---

## 完整资产清单

### 必需资产

| 资产类型 | 资产名称 | 路径 | 说明 |
|---------|---------|------|------|
| **Behavior** | `BP_SitBehavior` | `Behaviors/` | 坐下行为蓝图 |
| **Config** | `GBC_SitOnChair` | `Behaviors/` | 行为配置 |
| **Definition** | `DA_Chair` | `Definitions/` | 椅子定义 |
| **Actor** | `BP_Chair` | `Actors/` | 椅子蓝图 |
| **AI** | `BP_AIController` | `AI/` | AI 控制器 |
| **AI** | `BP_AICharacter` | `AI/` | AI 角色 |
| **BehaviorTree** | `BT_FindAndSitOnChair` | `AI/BehaviorTrees/` | 行为树 |
| **Blackboard** | `BB_AICharacter` | `AI/BehaviorTrees/` | 黑板 |

### 动画资源

| 资产 | 说明 |
|------|------|
| `AM_SitDown` | 坐下动画蒙太奇 |
| `AM_StandUp` | 起身动画蒙太奇 |
| `Anim_Sitting_Idle` | 坐姿待机动画 |

---

## 总结

恭喜！你已经成功搭建了一个完整的 SmartObject 系统！

**核心流程回顾**：

```
1. 创建行为（BP_SitBehavior + GBC_SitOnChair）
   → 定义"怎么坐"

2. 创建定义（DA_Chair）
   → 定义"椅子提供什么"

3. 创建 Actor（BP_Chair）
   → 在场景中实例化

4. 创建行为树（BT_FindAndSitOnChair）
   → AI 决策"要坐下"

5. 运行测试
   → AI 自动找到椅子并坐下
```

**关键要点**：

- ✅ **数据驱动**：对象细节在 Definition 中配置
- ✅ **通用逻辑**：一个行为树节点处理所有对象
- ✅ **解耦设计**：AI、对象、行为分离
- ✅ **易于扩展**：添加新对象只需新建 Definition

**下一步**：

- 尝试添加更多类型的对象（床、工作台）
- 使用 EQS 优化查询
- 添加更复杂的条件和标签
- 集成到实际项目中

---

**文档结束** 🎉
