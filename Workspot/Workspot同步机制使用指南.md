# Cyberpunk 2077 Workspot 同步机制完整使用指南

> 作者：基于CDPR2077源代码分析
> 版本：1.0
> 日期：2025年

---

## 目录

- [1. 核心概念](#1-核心概念)
- [2. 可用的同步节点类型](#2-可用的同步节点类型)
- [3. 创建同步Workspot的步骤](#3-创建同步workspot的步骤)
- [4. 核心参数配置详解](#4-核心参数配置详解)
- [5. 代码查询流程](#5-代码查询流程)
- [6. 运行时同步流程](#6-运行时同步流程)
- [7. 实际示例：双人握手](#7-实际示例双人握手)
- [8. 常见错误排查](#8-常见错误排查)
- [9. 检查清单](#9-检查清单)
- [10. 源代码参考](#10-源代码参考)

---

## 1. 核心概念

**同步系统(Sync System)** 用于实现多个角色之间的协同动画，例如：
- 握手、拥抱等双人交互
- 多人对话场景
- 协同操作物体
- 车辆多人乘坐

### 角色分类

| 角色类型 | 说明 | 职责 |
|---------|------|------|
| **Master (主控)** | 主导动画播放 | 控制动画时间轴，其他角色跟随 |
| **Slave (从属)** | 跟随主控播放 | 通过槽位名称和偏移量与主控同步 |

### 关键机制

```
Master Workspot (主控方)
    ↓ 通过 slotName 连接
Slave Workspot (从属方)
    ↓ 通过 syncOffset 定位
Synchronized Animation (同步动画播放)
```

---

## 2. 可用的同步节点类型

### 2.1 SyncAnimClip - 同步动画片段

```cpp
class SyncAnimClip : public AnimClip {
    CName m_animName;        // 动画名称
    CName m_slotName;        // 槽位名称(连接标识) ⭐
    Transform m_syncOffset;  // 同步偏移(相对位置) ⭐
    Float m_blendOutTime;    // 混合退出时间
};
```

**使用场景：** 循环播放的主体动画（如坐着聊天、握手持续等）

**文件位置：** `workspotTreeItems.h:89-108`

---

### 2.2 EntryAnim - 进入动画（可设为同步）

```cpp
class EntryAnim : public IEntry {
    CName m_animName;           // 进入动画名称
    CName m_slotName;           // 槽位名称 ⭐
    Bool m_isSynchronized;      // ✓勾选启用同步 ⭐
    Transform m_syncOffset;     // 同步偏移 ⭐
    CName m_idleAnim;           // 空闲姿态
    Float m_blendOutTime;       // 混合时间
    move::MovementType m_movementType;          // 移动类型
    move::MovementOrientationType m_orientationType;  // 朝向类型
};
```

**使用场景：** 进入workspot的同步动画（如走向椅子坐下）

**重要属性：**
- `m_isSynchronized` 必须设为 `true` 才能启用同步
- `m_slotName` 必须与配对的workspot相同

**文件位置：** `workspotTreeItems.h:268-303`

---

### 2.3 SyncMasterEntryAnim - 仅限主控的进入动画

```cpp
class SyncMasterEntryAnim : public EntryAnim {
    // 继承所有EntryAnim属性
    // m_isSynchronized 自动为 true，无法修改

    virtual Bool AllowSync(Bool asMaster) const override {
        return asMaster == true;  // ⭐仅允许作为Master使用
    }
};
```

**使用场景：** 明确标记为主控方的进入动画

**限制：**
- 只能被Master实体使用
- Slave实体使用会导致同步失败

**文件位置：** `workspotTreeItems.h:305-322`

---

### 2.4 ExitAnim - 退出动画（可设为同步）

```cpp
class ExitAnim : public IEntry {
    CName m_animName;           // 退出动画名称
    CName m_slotName;           // 槽位名称 ⭐
    Bool m_isSynchronized;      // ✓勾选启用同步 ⭐
    Transform m_syncOffset;     // 同步偏移 ⭐
    CName m_idleAnim;           // 空闲姿态
    Bool m_stayOnNavmesh;       // 保持在导航网格上
    Bool m_snapZToNavmesh;      // Z轴吸附到导航网格
    Bool m_disableRandomExit;   // 禁用随机退出
    move::MovementType m_movementType;  // 移动类型
};
```

**使用场景：** 退出workspot的同步动画（如起身离开）

**文件位置：** `workspotTreeItems.h:164-223`

---

## 3. 创建同步Workspot的步骤

### 3.1 Master Workspot（主控workspot）结构

```
WorkspotTree (主控)
└── Sequence (Root)
    ├── SyncMasterEntryAnim                    // 方案1: 使用专用主控节点
    │   ├── m_animName: "sit_down_chair_master"
    │   ├── m_slotName: "sync_slot_A"          // ⭐关键:槽位名称
    │   ├── m_isSynchronized: true (自动)
    │   └── m_syncOffset: Transform::IDENTITY  // 主控通常为原点
    │
    │   或者
    │
    ├── EntryAnim                              // 方案2: 使用普通进入节点
    │   ├── m_animName: "sit_down_chair_master"
    │   ├── m_slotName: "sync_slot_A"
    │   ├── m_isSynchronized: true             // ⭐必须手动勾选
    │   └── m_syncOffset: Transform::IDENTITY
    │
    ├── SyncAnimClip                           // 主体循环动画
    │   ├── m_animName: "sit_idle_master"
    │   ├── m_slotName: "sync_slot_A"
    │   └── m_syncOffset: Transform::IDENTITY
    │
    └── ExitAnim                               // 退出动画
        ├── m_animName: "sit_exit_master"
        ├── m_slotName: "sync_slot_A"
        ├── m_isSynchronized: true             // ⭐必须手动勾选
        └── m_syncOffset: Transform::IDENTITY
```

---

### 3.2 Slave Workspot（从属workspot）结构

```
WorkspotTree (从属)
└── Sequence (Root)
    ├── EntryAnim                              // ⚠️不能使用SyncMasterEntryAnim
    │   ├── m_animName: "sit_down_chair_slave"
    │   ├── m_slotName: "sync_slot_A"          // ⭐必须与Master相同
    │   ├── m_isSynchronized: true             // ⭐必须勾选
    │   └── m_syncOffset: Transform(
    │           Position: Vector4(1.0, 0.0, 0.0),    // 在主控右侧1米
    │           Rotation: Quaternion(0, 0, 0)
    │       )
    │
    ├── SyncAnimClip
    │   ├── m_animName: "sit_idle_slave"
    │   ├── m_slotName: "sync_slot_A"
    │   └── m_syncOffset: Transform(
    │           Position: Vector4(1.0, 0.0, 0.0)
    │       )
    │
    └── ExitAnim
        ├── m_animName: "sit_exit_slave"
        ├── m_slotName: "sync_slot_A"
        ├── m_isSynchronized: true
        └── m_syncOffset: Transform(
                Position: Vector4(1.0, 0.0, 0.0)
            )
```

---

## 4. 核心参数配置详解

### 4.1 m_slotName (槽位名称) - 最关键 ❗❗❗

**作用：** 用于匹配Master和Slave，是同步系统的"连接桥梁"

**规则：**
```cpp
// Master workspot 中
m_slotName = "handshake_left"

// Slave workspot 中
m_slotName = "handshake_left"  // ⭐必须完全一致，区分大小写
```

**命名建议：**
- 使用有意义的描述性名称，如：
  - `"conversation_seat_A"` / `"conversation_seat_B"`
  - `"handshake_sync"`
  - `"vehicle_driver"` / `"vehicle_passenger_front"`
- 避免使用通用名称如 `"sync"` 或 `"slot1"`

**错误示例：**
```cpp
// ❌ 错误：大小写不一致
Master: m_slotName = "SyncSlot"
Slave:  m_slotName = "syncslot"

// ❌ 错误：拼写差异
Master: m_slotName = "sync_slotA"
Slave:  m_slotName = "sync_slot_A"
```

---

### 4.2 m_syncOffset (同步偏移)

**作用：** 定义Slave相对于Master的空间位置和旋转

**坐标系：** Master的本地空间（Local Space）

#### Master设置（通常）
```cpp
m_syncOffset = Transform::IDENTITY();
// Position: (0, 0, 0)
// Rotation: (0, 0, 0)
```

#### Slave设置示例

**示例1：面对面站立（握手）**
```cpp
m_syncOffset = Transform(
    Position: Vector4(0.0, 1.2, 0.0),    // Y轴前1.2米
    Rotation: Quaternion(0, 0, 180)      // 旋转180度面对面
);
```

**示例2：并排坐（长椅）**
```cpp
m_syncOffset = Transform(
    Position: Vector4(1.0, 0.0, 0.0),    // X轴右侧1米
    Rotation: Quaternion(0, 0, 0)        // 同方向
);
```

**示例3：对面坐（桌子）**
```cpp
m_syncOffset = Transform(
    Position: Vector4(0.0, 1.5, 0.0),    // Y轴对面1.5米
    Rotation: Quaternion(0, 0, 180)      // 面对面
);
```

**坐标轴说明：**
```
    +Y (前)
     ↑
     |
     |
-X ←─┼─→ +X (右)
     |
     |
     ↓
    -Y (后)

+Z 为上，-Z 为下
```

---

### 4.3 m_isSynchronized (是否同步)

**适用节点：** `EntryAnim` 和 `ExitAnim`

**设置方法：**
```cpp
// 在编辑器中勾选 "isSynchronized" 复选框
// 或在代码中设置
m_isSynchronized = true;  // 启用同步
```

**自动启用的节点：**
- `SyncAnimClip` - 构造函数中自动添加 `Synchronized` 标志
- `SyncMasterEntryAnim` - 构造函数中强制设置 `m_isSynchronized = true`

**检查方法：**
```cpp
// workspotTreeItems.h:290
virtual Uint32 GetFlags() const override {
    return m_isSynchronized ? (m_flags | IEntry::Synchronized) : m_flags;
}
```

---

## 5. 代码查询流程

### 5.1 通过slotName查找同步Transform

**源文件：** `workspotResource.cpp:1041-1077`

```cpp
Bool WorkspotTree::GetSyncWorkspotTransform(
    CName slotName,
    Transform& outPosition,
    IContainerEntry* cont = nullptr
) const {
    // 从根节点开始递归查找
    if (cont == nullptr) {
        cont = Cast<IContainerEntry>(m_rootEntry.Get());
    }

    for (THandle<IEntry>& record : cont->m_list) {
        // 查找 EntryAnim 节点
        if (EntryAnim* entry = IsEntry<EntryAnim>(
            record.Get(),
            IEntry::SlowEnter | IEntry::Synchronized,  // ⭐必须有这两个标志
            slotName
        )) {
            outPosition = entry->m_syncOffset;  // ✓ 返回偏移量
            return true;
        }

        // 查找 ExitAnim 节点
        else if (ExitAnim* entry = IsEntry<ExitAnim>(
            record.Get(),
            IEntry::SlowExit | IEntry::Synchronized,
            slotName
        )) {
            outPosition = entry->m_syncOffset;
            return true;
        }

        // 查找 SyncAnimClip 节点
        else if (SyncAnimClip* entry = IsEntry<SyncAnimClip>(
            record.Get(),
            IEntry::Synchronized,
            slotName
        )) {
            outPosition = entry->m_syncOffset;
            return true;
        }

        // 递归查找容器节点
        if (IContainerEntry* container = Cast<IContainerEntry>(record.Get())) {
            if (GetSyncWorkspotTransform(slotName, outPosition, container)) {
                return true;
            }
        }
    }

    return false;
}
```

---

### 5.2 获取同步Entry的ID

**源文件：** `workspotResource.cpp:1079-1114`

```cpp
WorkEntryId WorkspotTree::GetSyncEntryIdForSlotName(
    CName slotName,
    Bool asMaster,      // ⭐true=作为Master查找, false=作为Slave查找
    IContainerEntry* cont = nullptr
) const {
    if (cont == nullptr) {
        cont = Cast<IContainerEntry>(m_rootEntry.Get());
    }

    for (THandle<IEntry>& record : cont->m_list) {
        if (EntryAnim* entry = IsEntry<EntryAnim>(
            cont, record.Get(),
            IEntry::SlowEnter | IEntry::Synchronized,
            slotName
        )) {
            // ⭐检查是否允许作为Master/Slave
            if (entry->AllowSync(asMaster))
                return entry->m_id;
        }
        else if (ExitAnim* entry = IsEntry<ExitAnim>(...)) {
            return entry->m_id;
        }
        else if (SyncAnimClip* entry = IsEntry<SyncAnimClip>(...)) {
            return entry->m_id;
        }

        // 递归搜索
        if (IContainerEntry* container = Cast<IContainerEntry>(record.Get())) {
            WorkEntryId id = GetSyncEntryIdForSlotName(slotName, asMaster, container);
            if (id) return id;
        }
    }

    return WorkEntryId::invalid;
}
```

---

### 5.3 AllowSync权限检查

**SyncMasterEntryAnim的限制**（`workspotTreeItems.h:321`）：

```cpp
class SyncMasterEntryAnim : public EntryAnim {
    virtual Bool AllowSync(Bool asMaster) const override {
        return asMaster == true;  // ⭐仅允许作为Master
    }
};
```

**EntryAnim的默认行为**（`workspotTreeItems.h:302`）：

```cpp
class EntryAnim : public IEntry {
    virtual Bool AllowSync(Bool asMaster) const {
        return true;  // 可以作为Master或Slave
    }
};
```

**查询逻辑：**
```
查询asMaster=true时:
  ├─ SyncMasterEntryAnim → ✓ 返回true
  └─ EntryAnim           → ✓ 返回true

查询asMaster=false时:
  ├─ SyncMasterEntryAnim → ✗ 返回false (拒绝)
  └─ EntryAnim           → ✓ 返回true
```

---

## 6. 运行时同步流程

**源文件：** `workspotSynchronizer.cpp:168-235`

### 6.1 同步器执行步骤

```cpp
// 步骤1: Master开始播放动画
// Master的当前槽位动画名称存储在 m_currentSlotAnim

// 步骤2: 查找Master的Entry ID
work::WorkEntryId masterEntryId = m_system.GetSyncEntryIdForSlotName(
    masterEntityId,     // Master实体ID
    true,               // asMaster = true
    masterEntry.m_currentSlotAnim  // 当前槽位动画
);

if (masterEntryId == invalid) {
    // ⚠️ 未找到Master同步点，同步失败
    return;
}

// 步骤3: 查找匹配的Slave workspot
work::WorkEntryId slaveEntryId = m_system.GetSyncEntryIdForSlotName(
    slaveEntityId,      // Slave实体ID
    false,              // asMaster = false
    slotName            // 相同的槽位名称
);

if (slaveEntryId == invalid) {
    // ⚠️ 未找到Slave同步点，同步失败
    return;
}

// 步骤4: 获取Slave的偏移位置
Transform syncOffset;
if (!m_system.GetSyncWorkspotTransform(slaveEntityId, slotName, syncOffset)) {
    // ⚠️ 未找到偏移量，使用默认Identity
    syncOffset = Transform::IDENTITY();
}

// 步骤5: 计算Slave在世界空间的位置
Transform masterWorldTransform = GetWorldTransform(masterEntityId);
Transform slaveWorldPosition;

slaveWorldPosition.SetPosition(
    masterWorldTransform.GetPosition() +
    masterWorldTransform.GetOrientation().TransformVector(syncOffset.GetPosition())
);
slaveWorldPosition.SetOrientation(
    masterWorldTransform.GetOrientation() * syncOffset.GetOrientation()
);

// 步骤6: 跳转到同步Entry
GoToEntry(slaveEntityId, slaveEntryId);

// 步骤7: 同步播放
// Slave的动画时间轴 = Master的动画时间轴
syncedAnimationTime = masterAnimationTime;
```

---

### 6.2 时间同步机制

```cpp
// 在 WorkspotInstance 中
void UpdateAnimation(Float deltaTime) {
    if (isSynchronized) {
        // Slave不自行更新时间，从Master获取
        m_currentAnimTime = GetMasterAnimTime();
    } else {
        // Master正常更新时间
        m_currentAnimTime += deltaTime;
    }
}
```

---

## 7. 实际示例：双人握手

### 7.1 Master Workspot配置

**文件：** `handshake_master.workspot`

```json
{
  "WorkspotTree": {
    "m_rootEntry": {
      "Type": "Sequence",
      "m_list": [
        {
          "Type": "SyncMasterEntryAnim",
          "m_id": 1,
          "m_animName": "man_greeting_handshake_approach_01",
          "m_slotName": "handshake_sync",
          "m_isSynchronized": true,
          "m_syncOffset": {
            "Position": [0.0, 0.0, 0.0],
            "Rotation": [0.0, 0.0, 0.0, 1.0]
          },
          "m_movementType": "Walk",
          "m_orientationType": "Forward"
        },
        {
          "Type": "SyncAnimClip",
          "m_id": 2,
          "m_animName": "man_greeting_handshake_shake_01",
          "m_slotName": "handshake_sync",
          "m_syncOffset": {
            "Position": [0.0, 0.0, 0.0],
            "Rotation": [0.0, 0.0, 0.0, 1.0]
          },
          "m_blendOutTime": 0.5
        },
        {
          "Type": "ExitAnim",
          "m_id": 3,
          "m_animName": "man_greeting_handshake_exit_01",
          "m_slotName": "handshake_sync",
          "m_isSynchronized": true,
          "m_syncOffset": {
            "Position": [0.0, 0.0, 0.0],
            "Rotation": [0.0, 0.0, 0.0, 1.0]
          },
          "m_movementType": "Walk"
        }
      ]
    }
  }
}
```

---

### 7.2 Slave Workspot配置

**文件：** `handshake_slave.workspot`

```json
{
  "WorkspotTree": {
    "m_rootEntry": {
      "Type": "Sequence",
      "m_list": [
        {
          "Type": "EntryAnim",
          "m_id": 1,
          "m_animName": "woman_greeting_handshake_approach_01",
          "m_slotName": "handshake_sync",
          "m_isSynchronized": true,
          "m_syncOffset": {
            "Position": [0.0, 1.2, 0.0],
            "Rotation": [0.0, 0.0, 1.0, 0.0]
          },
          "m_movementType": "Walk",
          "m_orientationType": "Forward"
        },
        {
          "Type": "SyncAnimClip",
          "m_id": 2,
          "m_animName": "woman_greeting_handshake_shake_01",
          "m_slotName": "handshake_sync",
          "m_syncOffset": {
            "Position": [0.0, 1.2, 0.0],
            "Rotation": [0.0, 0.0, 1.0, 0.0]
          },
          "m_blendOutTime": 0.5
        },
        {
          "Type": "ExitAnim",
          "m_id": 3,
          "m_animName": "woman_greeting_handshake_exit_01",
          "m_slotName": "handshake_sync",
          "m_isSynchronized": true,
          "m_syncOffset": {
            "Position": [0.0, 1.2, 0.0],
            "Rotation": [0.0, 0.0, 1.0, 0.0]
          },
          "m_movementType": "Walk"
        }
      ]
    }
  }
}
```

**关键解释：**
- `Position: [0.0, 1.2, 0.0]` - Slave在Master前方1.2米
- `Rotation: [0.0, 0.0, 1.0, 0.0]` - 旋转180度面对面（四元数格式）

---

### 7.3 动画时长要求

```cpp
// ⚠️ 重要：Master和Slave的对应动画必须时长一致

Master动画时长:
  - man_greeting_handshake_approach_01: 2.5秒
  - man_greeting_handshake_shake_01:    3.0秒
  - man_greeting_handshake_exit_01:     1.8秒

Slave动画时长（必须匹配）:
  - woman_greeting_handshake_approach_01: 2.5秒 ✓
  - woman_greeting_handshake_shake_01:    3.0秒 ✓
  - woman_greeting_handshake_exit_01:     1.8秒 ✓
```

---

## 8. 常见错误排查

### 错误1：同步未触发

**症状：** Slave角色不跟随Master播放动画

**可能原因：**
```cpp
// ❌ 原因1: slotName不匹配
Master: m_slotName = "sync_handshake"
Slave:  m_slotName = "handshake_sync"  // 不同！

// ❌ 原因2: Slave使用了SyncMasterEntryAnim
{
  "Type": "SyncMasterEntryAnim"  // 错误！Slave应使用EntryAnim
}

// ❌ 原因3: 忘记勾选m_isSynchronized
{
  "Type": "EntryAnim",
  "m_isSynchronized": false  // 错误！应为true
}
```

**解决方案：**
1. 检查Master和Slave的`m_slotName`是否完全一致（区分大小写）
2. Slave使用`EntryAnim`而非`SyncMasterEntryAnim`
3. 确保`m_isSynchronized = true`

---

### 错误2：位置偏移错误

**症状：** Slave出现在错误的位置或旋转不正确

**可能原因：**
```cpp
// ❌ 原因1: 偏移量基于错误的坐标系
m_syncOffset.Position = [1.0, 0.0, 0.0];  // 本应在前方，但出现在右侧

// ❌ 原因2: 旋转角度计算错误
m_syncOffset.Rotation = Euler(0, 0, 90);  // 应为180度

// ❌ 原因3: Master的syncOffset不是Identity
Master: m_syncOffset.Position = [0.5, 0.0, 0.0];  // 错误！Master应为原点
```

**解决方案：**
1. **Master的偏移必须为Identity：**
   ```cpp
   m_syncOffset = Transform::IDENTITY();
   ```

2. **验证Slave偏移的坐标轴：**
   ```
   +X = Master的右侧
   +Y = Master的前方
   +Z = Master的上方
   ```

3. **使用四元数或欧拉角：**
   ```cpp
   // 面对面（旋转180度）
   Quaternion: [0.0, 0.0, 1.0, 0.0]
   Euler:      [0.0, 0.0, 180.0]
   ```

---

### 错误3：动画不对齐

**症状：** 两个角色动画播放速度不同步或位置漂移

**可能原因：**
```cpp
// ❌ 原因1: 动画时长不一致
Master: "handshake_shake_m" - 3.0秒
Slave:  "handshake_shake_s" - 2.5秒  // 时长不同！

// ❌ 原因2: 动画的Motion Extraction不匹配
Master动画有位移: 从(0,0,0)移动到(0,0.5,0)
Slave动画无位移:  保持在(0,0,0)
```

**解决方案：**
1. **确保动画时长完全一致**
2. **检查Motion Extraction设置：**
   ```cpp
   // 在动画编辑器中
   Master动画: ExtractedMotion = (0.0, 0.5, 0.0)
   Slave动画:  ExtractedMotion = (0.0, 0.5, 0.0)  // 必须相同
   ```

---

### 错误4：Slave无法启动

**症状：** Slave workspot根本不播放

**可能原因：**
```cpp
// ❌ 原因1: Slave的AnimSet未加载
m_finalAnimsets = [];  // 空数组！

// ❌ 原因2: Slave的Rig类型不匹配
Master Rig: "base/characters/common/player_base_bodies/player_man_average.rig"
Slave Rig:  "base/characters/common/crowd/crowd_woman.rig"
// 如果动画不支持这个Rig，会失败

// ❌ 原因3: WorkspotSystem未注册Slave
RegisterSyncedWorkspot(masterEntityId, slaveEntityId, slotName);  // 忘记调用
```

**解决方案：**
1. 检查`m_finalAnimsets`包含正确的AnimSet
2. 确保动画支持目标Rig
3. 正确调用同步注册函数

---

### 错误5：多个Slave冲突

**症状：** 有多个Slave时，只有一个能正常同步

**可能原因：**
```cpp
// ❌ 原因1: 使用了相同的slotName
Slave1: m_slotName = "sync_slot"
Slave2: m_slotName = "sync_slot"  // 冲突！

// ❌ 原因2: 偏移量相同导致位置重叠
Slave1: m_syncOffset.Position = [1.0, 0.0, 0.0]
Slave2: m_syncOffset.Position = [1.0, 0.0, 0.0]  // 重叠！
```

**解决方案：**
```cpp
// ✓ 为每个Slave使用不同的slotName
Slave1: m_slotName = "passenger_front"
Slave2: m_slotName = "passenger_rear_left"
Slave3: m_slotName = "passenger_rear_right"

// ✓ 确保偏移量不重叠
Slave1: m_syncOffset.Position = [1.0, 0.0, 0.0]  // 右侧
Slave2: m_syncOffset.Position = [-1.0, 0.0, 0.0] // 左侧
Slave3: m_syncOffset.Position = [0.0, 1.5, 0.0]  // 前方
```

---

## 9. 检查清单

在创建同步Workspot前，请确认以下所有项目：

### 🔲 Master Workspot

- [ ] 使用 `SyncMasterEntryAnim` 或设置 `EntryAnim.m_isSynchronized = true`
- [ ] 所有同步节点的 `m_slotName` 已设置且一致
- [ ] 所有同步节点的 `m_syncOffset = Transform::IDENTITY()`
- [ ] SyncAnimClip 的动画名称正确
- [ ] ExitAnim 的 `m_isSynchronized = true`（如需同步退出）

---

### 🔲 Slave Workspot

- [ ] **不使用** `SyncMasterEntryAnim`，仅使用 `EntryAnim`
- [ ] 所有同步节点的 `m_slotName` 与Master完全一致
- [ ] 所有同步节点的 `m_isSynchronized = true`
- [ ] `m_syncOffset` 正确定义了相对于Master的位置
- [ ] 动画名称对应Slave版本的动画
- [ ] SyncAnimClip 和 ExitAnim 的 `m_syncOffset` 与EntryAnim一致

---

### 🔲 动画资源

- [ ] Master和Slave的对应动画时长**完全相同**
- [ ] 动画的Motion Extraction设置一致
- [ ] 动画已添加到对应的AnimSet中
- [ ] AnimSet已分配给正确的Rig

---

### 🔲 运行时设置

- [ ] 调用了 `RegisterSyncedWorkspot()` 注册同步关系
- [ ] Master实体先启动workspot
- [ ] Slave实体在Master启动后触发同步
- [ ] 检查日志无 `GetSyncEntryIdForSlotName` 返回invalid的警告

---

### 🔲 测试验证

- [ ] 在编辑器中预览Master和Slave的动画
- [ ] 测试不同的 `m_syncOffset` 值
- [ ] 验证旋转角度正确（面对面/并排/背对背）
- [ ] 检查多帧时间对齐
- [ ] 测试Enter、Loop、Exit的完整流程

---

## 10. 源代码参考

### 关键文件位置

```
源码根目录: D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev\

核心文件:
├── src/common/gameWorkspots/include/
│   ├── workspotTreeItems.h          # 节点类型定义 (89-322行)
│   ├── workspotResource.h           # WorkspotTree主体 (323-528行)
│   └── workspotSystem.h             # 同步系统接口 (423-424行)
│
└── src/common/gameWorkspots/src/
    ├── workspotTreeItems.cpp        # 节点迭代器实现 (477-491行)
    ├── workspotResource.cpp         # 同步查询实现 (1041-1114行)
    ├── workspotSynchronizer.cpp     # 运行时同步逻辑 (168-235行)
    └── gameWorkspotsInstance.cpp    # 实例管理 (1846-1853行)
```

---

### 核心函数调用链

```
播放同步Workspot
    ↓
WorkspotSystem::PlayWorkspot(masterEntityId)
    ↓
WorkspotSystem::RegisterSyncedWorkspot(masterEntityId, slaveEntityId, slotName)
    ↓
WorkspotSynchronizer::OnMasterAnimationChanged()
    ↓
    ├─→ GetSyncEntryIdForSlotName(masterEntityId, true, slotName)
    │       ↓
    │   WorkspotTree::GetSyncEntryIdForSlotName()
    │       ↓
    │   遍历查找匹配 slotName + Synchronized flag 的节点
    │       ↓
    │   检查 AllowSync(asMaster=true)
    │       ↓
    │   返回 EntryId
    │
    └─→ GetSyncEntryIdForSlotName(slaveEntityId, false, slotName)
            ↓
        查找Slave的同步Entry
            ↓
        GetSyncWorkspotTransform(slaveEntityId, slotName, syncOffset)
            ↓
        计算Slave世界位置 = Master位置 + (Master旋转 * syncOffset)
            ↓
        GoToEntry(slaveEntityId, slaveEntryId)
            ↓
        同步动画时间轴
```

---

### 调试技巧

#### 1. 启用Workspot调试绘制

```cpp
// 在游戏控制台输入:
StreamingAnimation.ShowStreamedInWorkspotAnims 1

// 会在屏幕上显示:
// [F] - Found (在AnimSet中找到)
// [P] - Preload collected (已预加载)
// [C] - Cinematic (过场动画)
// [L] - Loaded (已加载到内存)
```

#### 2. 断点调试位置

```cpp
// workspotResource.cpp:1041
Bool WorkspotTree::GetSyncWorkspotTransform(...)
// 在此设断点，检查slotName匹配情况

// workspotResource.cpp:1079
WorkEntryId WorkspotTree::GetSyncEntryIdForSlotName(...)
// 检查是否找到正确的EntryId

// workspotSynchronizer.cpp:168
if (work::WorkEntryId entryId = m_system.GetSyncEntryIdForSlotName(...))
// 检查Master和Slave的EntryId查询结果
```

#### 3. 日志输出

```cpp
// 添加调试日志:
RED_LOG(WorkspotSync, "Master SlotName: %hs, EntryId: %u",
    slotName.AsChar(),
    entryId.value);

RED_LOG(WorkspotSync, "Slave SyncOffset: (%.2f, %.2f, %.2f)",
    syncOffset.GetPosition().X,
    syncOffset.GetPosition().Y,
    syncOffset.GetPosition().Z);
```

---

## 附录A: 快速参考表

| 节点类型 | 用途 | m_slotName | m_isSynchronized | m_syncOffset |
|---------|------|------------|------------------|--------------|
| **SyncMasterEntryAnim** | Master进入 | 必填 | 自动true | Identity |
| **EntryAnim** | 通用进入 | 必填 | 手动勾选 | Master=Identity<br>Slave=自定义 |
| **SyncAnimClip** | 循环动画 | 必填 | 自动true | 同上 |
| **ExitAnim** | 退出动画 | 必填 | 手动勾选 | 同上 |

---

## 附录B: 常用槽位名称建议

```cpp
// 车辆相关
"vehicle_driver"
"vehicle_passenger_front"
"vehicle_passenger_rear_left"
"vehicle_passenger_rear_right"

// 对话场景
"conversation_seat_main"
"conversation_seat_A"
"conversation_seat_B"

// 社交互动
"handshake_sync"
"hug_initiator"
"hug_receiver"
"dance_leader"
"dance_follower"

// 物体操作
"lift_heavy_object_left"
"lift_heavy_object_right"
"door_opener"
"door_holder"
```

---

## 附录C: Transform计算公式

### 世界坐标转换

```cpp
// Slave在世界空间的最终位置
Vector4 slaveWorldPosition =
    masterWorldPosition +
    masterWorldRotation.TransformVector(slaveSyncOffset.Position);

Quaternion slaveWorldRotation =
    masterWorldRotation * slaveSyncOffset.Rotation;
```

### 常用旋转值

| 目标方向 | 欧拉角 (Pitch, Yaw, Roll) | 四元数 (X, Y, Z, W) |
|---------|--------------------------|-------------------|
| 同向 | (0, 0, 0) | (0, 0, 0, 1) |
| 面对面 | (0, 0, 180) | (0, 0, 1, 0) |
| 左转90度 | (0, 0, 90) | (0, 0, 0.707, 0.707) |
| 右转90度 | (0, 0, -90) | (0, 0, -0.707, 0.707) |
| 背对背 | (0, 0, 180) | (0, 0, 1, 0) |

---

**文档版本：** 1.0
**最后更新：** 2025年
**作者：** 基于CDPR2077引擎源代码分析
