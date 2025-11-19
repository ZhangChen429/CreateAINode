
## 三、所有AI命令参数类型完整列表

### 3.1 基础类型

| 类名 | 层级 | 用途 |
|------|------|------|
| **AICommandParams** | 抽象基类 | 所有命令参数的基类 |
| **NotImplementedAICommandParams** | 占位类 | 未实现的命令占位符 |

### 3.2 SendAICommandNode 的策略

**文件：** `sendAICommandNode.h`

| 参数类 | 用途 | 主要属性 |
|--------|------|---------|
| **ConstAICommandParams** | 包装现有命令 | `m_command: AI::CommandPtr` |
| **UseWorkspotCommandParams** | 使用工作点 | `m_workspotNode`, `m_moveToWorkspot`, `m_forceEntryAnimName` |

### 3.3 CombatNode 的策略（9种）

**文件：** `combatNode.h`

| 参数类 | 友好名称 | 用途 | 主要属性 |
|--------|---------|------|---------|
| **CombatNodeParams_CombatTarget** | Combat Target | 设置战斗目标 | `m_targetNode`, `m_targetPuppet`, `m_duration`, `m_immediately` |
| **CombatNodeParams_ShootAt** | Shoot At | 射击目标 | `m_targetOverrideNode`, `m_targetOverridePuppet`, `m_duration`, `m_once` |
| **CombatNodeParams_LookAtTarget** | Look At | 看向目标 | `m_targetNode`, `m_targetPuppet`, `m_duration`, `m_immediately` |
| **CombatNodeParams_ThrowGrenade** | Throw Grenade | 投掷手雷 | `m_targetOverrideNode`, `m_targetOverridePuppet`, `m_duration`, `m_once` |
| **CombatNodeParams_UseCover** | Use Cover | 使用掩体 | `m_cover`, `m_oneTimeSelection`, `m_forcedEntryAnimation`, `m_forceStance` |
| **CombatNodeParams_SwitchWeapon** | Switch Weapon | 切换武器 | `m_mode` (PrimaryWeapon/SecondaryWeapon) |
| **CombatNodeParams_PrimaryWeapon** | Primary Weapon | 主武器操作 | `m_unEquip` (装备/卸下) |
| **CombatNodeParams_SecondaryWeapon** | Secondary Weapon | 副武器操作 | `m_unEquip` (装备/卸下) |
| **CombatNodeParams_RestrictMovementToArea** | **Restrict Movement** ✅ | **限制移动区域** | **`m_area`** (区域节点引用) |

### 3.4 MovePuppetNode 的策略（5种）

**文件：** `movePuppetNode.h`

#### 3.4.1 MoveOnSplineParams - 沿样条曲线移动

**主要属性**：
```cpp
world::NodeRef m_splineNodeRef;               // 样条曲线引用
Bool m_startFromClosestPoint;                 // 从最近点开始
Bool m_useStart;                              // 使用起始动画
Bool m_useStop;                               // 使用停止动画
Bool m_reverse;                               // 反向移动
Bool m_useAlertedState;                       // 使用警戒状态
Bool m_useCombatState;                        // 使用战斗状态
Bool m_alwaysUseStealth;                      // 总是使用潜行
THandle<MoveOnSplineAdditionalParams> m_additionalParams;  // 额外参数
```

**额外参数类型**：
- **SimpleMoveOnSplineParams**: 简单移动
  - `m_movementType`: 移动类型
  - `m_facingTargetRef`: 面向目标
  - `m_snapToTerrain`: 贴地

- **AnimMoveOnSplineParams**: 动画移动
  - `m_controllersSetupName`: 控制器设置
  - `m_customStartAnimationName`: 自定义起始动画
  - `m_customMainAnimationName`: 自定义主动画
  - `m_customStopAnimationName`: 自定义停止动画

- **WithCompanionMoveOnSplineParams**: 伴随移动
  - `m_companionRef`: 伴随者引用
  - `m_companionDistancePreset`: 伴随距离预设
  - `m_companionPosition`: 伴随位置 (Behind/InFront)
  - `m_shootingTargetRef`: 射击目标

#### 3.4.2 MoveToParams - 移动到目标

**主要属性**：
```cpp
THandle<UniversalRef> m_movementTargetRef;    // 移动目标
THandle<UniversalRef> m_facingTargetRef;      // 面向目标
Bool m_rotateEntityTowardsFacingTarget;       // 旋转朝向
move::MovementType m_movementType;            // 移动类型
Bool m_ignoreNavigation;                      // 忽略导航
Float m_desiredDistanceFromTarget;            // 目标距离
Bool m_finishWhenDestinationReached;          // 到达时完成
Bool m_alwaysUseStealth;                      // 总是潜行
```

#### 3.4.3 PatrolParams - 巡逻

**主要属性**：
```cpp
THandle<AI::PatrolPathParameters> m_pathParams;  // 巡逻路径参数
Bool m_repeatCommandOnInterrupt;                 // 中断后重复
```

#### 3.4.4 FollowParams - 跟随

**主要属性**：
```cpp
THandle<UniversalRef> m_companionRef;         // 跟随目标
Float m_companionDistance = 5.0f;             // 跟随距离
Float m_destinationPointTolerance = 2.0f;     // 目标点容差
Bool m_stopWhenDestinationReached;            // 到达后停止
move::MovementType m_movementType;            // 移动类型
Bool m_matchSpeed;                            // 匹配速度
Bool m_useTeleport;                           // 使用传送
```

#### 3.4.5 MovePuppetNodeParams - 移动参数容器

**用途**：统一管理所有移动类型的参数

```cpp
MoveType m_moveType;  // 枚举类型：MoveOnSpline, MoveTo, RotateTo, Patrol, Follow
THandle<MoveOnSplineParams> m_moveOnSplineParams;
THandle<MoveToParams> m_moveToParams;
THandle<AICommandParams> m_otherParams;
```

### 3.5 MiscAICommandNode 的策略

**文件：** `miscAICommandNode.h`

| 参数类 | 用途 | 备注 |
|--------|------|------|
| **MiscAICommandNodeParams** | 杂项命令基类 | 抽象基类 |
| **ScriptedAICommandParams** | 脚本自定义命令 | 通过反射调用脚本函数 |
| **AIClearRoleCommandParams** | 清除AI角色 | 默认功能，恢复默认行为 |
| **AIAssignRoleCommandParams** | 分配AI角色 | 设置NPC的角色和行为模式 |

### 3.6 EquipItemNode 的策略

**文件：** `equipItemNode.h`

| 参数类 | 用途 | 说明 |
|--------|------|------|
| **EquipItemParams** | 装备/卸下物品 | NPC装备管理 |

### 3.7 VehicleCommandNode 的策略

**文件：** `vehicleCommandNode.h`

| 参数类 | 用途 | 说明 |
|--------|------|------|
| **VehicleCommandParams** | 车辆控制命令 | 控制车辆AI行为 |

### 3.8 TeleportPuppetNode 的策略

**文件：** `teleportTypes.h`

| 参数类 | 用途 | 说明 |
|--------|------|------|
| **TeleportPuppetParamsV1** | NPC传送 | 瞬移NPC到指定位置 |

---



  🔹 其他节点的策略 (共享 AICommandParams 基类)

  1️⃣ 战斗策略组 (CombatNode使用)
  - CombatNodeParams_CombatTarget - 设置战斗目标
  - CombatNodeParams_ShootAt - 射击目标
  - CombatNodeParams_LookAtTarget - 注视目标
  - CombatNodeParams_ThrowGrenade - 投掷手榴弹
  - CombatNodeParams_UseCover - 使用掩体
  - CombatNodeParams_SwitchWeapon - 切换武器
  - CombatNodeParams_PrimaryWeapon - 主武器
  - CombatNodeParams_SecondaryWeapon - 副武器
  - CombatNodeParams_RestrictMovementToArea - 限制移动区域

  2️⃣ 移动策略组 (MovePuppetNode使用)
  - MoveOnSplineParams - 沿样条线移动
  - MoveToParams - 移动到目标点
  - PatrolParams - 巡逻
  - FollowParams - 跟随
  - MovePuppetNodeParams - 通用移动

  3️⃣ 场景交互策略
  - UseWorkspotCommandParams - 使用工作点
  - ConstAICommandParams - 常量AI命令
  - EquipItemParams - 装备物品
  - TeleportPuppetParamsV1 - 传送

  4️⃣ 载具策略
  - VehicleCommandParams - 载具命令

  5️⃣ 工具策略
  - NotImplementedAICommandParams - 未实现占位符