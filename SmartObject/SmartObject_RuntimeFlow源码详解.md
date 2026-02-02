# SmartObject RuntimeFlow 源码详解

> **基于 UE 5.7 引擎源码分析**
>
> **源码路径**: `D:\AppSoft\UEItem\UE_5.7\Engine\Plugins\Runtime\SmartObjects`
>
> **日期**: 2026-01-30

---

## 目录

- [核心概念](#核心概念)
- [状态机模型](#状态机模型)
- [核心数据结构](#核心数据结构)
- [RuntimeFlow 完整流程](#runtimeflow-完整流程)
- [关键 API 源码分析](#关键-api-源码分析)
- [事件系统](#事件系统)
- [线程安全说明](#线程安全说明)
- [最佳实践](#最佳实践)

---

## 核心概念

### RuntimeFlow 是什么？

**RuntimeFlow** 是指 SmartObject 在运行时的完整生命周期管理流程，包括：

```
注册 → 发现 → 声明(Claim) → 占用(Occupy) → 释放(Release) → 循环
```

### 关键文件

| 文件 | 路径 | 作用 |
|------|------|------|
| **SmartObjectRuntime.h** | Public/ | 定义运行时数据结构和状态枚举 |
| **SmartObjectRuntime.cpp** | Private/ | 实现状态转换逻辑 |
| **SmartObjectSubsystem.h** | Public/ | 定义 Subsystem API |
| **SmartObjectSubsystem.cpp** | Private/ | 实现完整的 RuntimeFlow |
| **SmartObjectTypes.h** | Public/ | 定义 Handle、Event 等基础类型 |

---

## 状态机模型

### 状态定义

```cpp
// 源码：SmartObjectRuntime.h:22
enum class ESmartObjectSlotState : uint8
{
    Invalid,    // 无效状态
    Free,       // 空闲（可用）
    Claimed,    // 已声明（预定但未使用）
    Occupied,   // 占用中（正在使用）
};
```

### 状态转换图

```
┌────────────────────────────────────────────────────┐
│ SmartObject Slot 状态机                             │
├────────────────────────────────────────────────────┤
│                                                    │
│         ┌───────────────────────┐                 │
│         │                       │                 │
│         │    1. Free (空闲)      │ ◄────┐         │
│         │                       │      │         │
│         └───────────┬───────────┘      │         │
│                     │                  │         │
│            MarkSlotAsClaimed()         │         │
│                     │                  │         │
│                     ↓                  │         │
│         ┌───────────────────────┐      │         │
│         │                       │      │         │
│         │  2. Claimed (已声明)   │      │         │
│         │   (预定但未使用)        │      │         │
│         │                       │      │         │
│         └───────────┬───────────┘      │         │
│                     │                  │         │
│            MarkSlotAsOccupied()   MarkSlotAsFree()│
│                     │                  │         │
│                     ↓                  │         │
│         ┌───────────────────────┐      │         │
│         │                       │      │         │
│         │  3. Occupied (占用中)  │ ─────┘         │
│         │   (正在使用)           │                 │
│         │                       │                 │
│         └───────────────────────┘                 │
│                                                    │
└────────────────────────────────────────────────────┘
```

### 状态转换规则

#### 1. Free → Claimed（声明槽位）

**条件**：
- Slot 和 Object 都处于 Enabled 状态
- Slot 当前状态为 `Free`
- 或者 Slot 状态为 `Claimed` 但新的声明优先级更高

**源码位置**: `SmartObjectRuntime.cpp:164`

```cpp
bool FSmartObjectRuntimeSlot::CanBeClaimed(ESmartObjectClaimPriority ClaimPriority) const
{
    return IsEnabled()
        && (State == ESmartObjectSlotState::Free
            || (State == ESmartObjectSlotState::Claimed
                && ClaimedPriority < ClaimPriority));
}
```

**执行结果**：
- 状态变为 `Claimed`
- 记录 User Handle
- 记录声明优先级
- 触发 `OnClaimed` 事件

---

#### 2. Claimed → Occupied（开始使用）

**条件**：
- Slot 状态必须是 `Claimed`
- User Handle 必须匹配（声明者和使用者必须是同一个）
- SmartObject 必须处于 Enabled 状态

**源码位置**: `SmartObjectSubsystem.cpp:1515`

```cpp
if (Slot.GetState() == ESmartObjectSlotState::Claimed)
{
    if (Slot.User == ClaimHandle.UserHandle)
    {
        Slot.State = ESmartObjectSlotState::Occupied;
        OnSlotChangedInternal(SmartObjectRuntime, Slot, ClaimHandle.SlotHandle,
                              ESmartObjectChangeReason::OnOccupied, Slot.UserData);
        return BehaviorDefinition;
    }
}
```

**执行结果**：
- 状态变为 `Occupied`
- 返回 BehaviorDefinition（用于执行行为）
- 触发 `OnOccupied` 事件

---

#### 3. Claimed/Occupied → Free（释放槽位）

**条件**：
- Slot 状态必须是 `Claimed` 或 `Occupied`
- User Handle 必须匹配

**源码位置**: `SmartObjectRuntime.cpp:146`

```cpp
bool FSmartObjectRuntimeSlot::Release(const FSmartObjectClaimHandle& ClaimHandle, const bool bAborted)
{
    if (State != ESmartObjectSlotState::Claimed && State != ESmartObjectSlotState::Occupied)
    {
        UE_LOG(LogSmartObject, Error, TEXT("Expected state is 'Claimed' or 'Occupied'"));
        return false;
    }

    if (ClaimHandle.UserHandle != User)
    {
        UE_LOG(LogSmartObject, Error, TEXT("User mismatch"));
        return false;
    }

    if (bAborted)
    {
        // 如果是中止（而非正常完成），触发回调
        OnSlotInvalidatedDelegate.ExecuteIfBound(ClaimHandle, State);
    }

    State = ESmartObjectSlotState::Free;
    User.Invalidate();
    UserData.Reset();
    ClaimedPriority = ESmartObjectClaimPriority::None;
    return true;
}
```

**执行结果**：
- 状态变为 `Free`
- 清空 User Handle
- 清空 UserData
- 重置优先级
- 触发 `OnReleased` 事件

---

## 核心数据结构

### 1. FSmartObjectRuntime（对象运行时）

**源码**: `SmartObjectRuntime.h:252`

```cpp
USTRUCT()
struct FSmartObjectRuntime
{
    GENERATED_BODY()

    // ============================================
    // 核心数据
    // ============================================

    /** 关联的 SmartObject Definition（资产） */
    UPROPERTY()
    TObjectPtr<const USmartObjectDefinition> Definition = nullptr;

    /** 运行时槽位数组 */
    UPROPERTY(Transient)
    TArray<FSmartObjectRuntimeSlot> Slots;

    /** 实例的世界变换 */
    UPROPERTY(Transient, VisibleAnywhere, Category=SmartObjects)
    FTransform Transform;

    /** 运行时标签（可动态修改） */
    UPROPERTY(Transient, VisibleAnywhere, Category=SmartObjects)
    FGameplayTagContainer Tags;

    /** 注册句柄（在 Subsystem 中的唯一标识） */
    UPROPERTY(Transient, VisibleAnywhere, Category=SmartObjects)
    FSmartObjectHandle RegisteredHandle;

    /** 关联的组件（可能为空，如果 Actor 未加载） */
    UPROPERTY()
    TWeakObjectPtr<USmartObjectComponent> OwnerComponent;

    /** 禁用标志（位掩码，支持多个禁用原因） */
    UPROPERTY(Transient, VisibleAnywhere, Category=SmartObjects)
    uint16 DisableFlags = 0;

    /** 事件委托 */
    FOnSmartObjectEvent OnEvent;

    // ============================================
    // 关键方法
    // ============================================

    /** 获取 Definition */
    const USmartObjectDefinition& GetDefinition() const
    {
        checkf(Definition != nullptr, TEXT("Must be valid"));
        return *Definition;
    }

    /** 检查是否启用 */
    bool IsEnabled() const
    {
        return DisableFlags == 0;
    }

    /** 设置启用/禁用状态（支持多个原因） */
    void SetEnabled(FGameplayTag ReasonTag, bool bEnabled);
};
```

**关键理解**：
- **Definition 是只读的**：运行时不会修改 Definition，只是引用
- **Slots 是可变的**：每个 Slot 都有自己的运行时状态
- **DisableFlags 是位掩码**：支持多个系统同时禁用/启用对象

---

### 2. FSmartObjectRuntimeSlot（槽位运行时）

**源码**: `SmartObjectRuntime.h:134`

```cpp
USTRUCT()
struct FSmartObjectRuntimeSlot
{
    GENERATED_BODY()

    // ============================================
    // 槽位几何信息
    // ============================================

    /** 槽位相对于 SmartObject 的偏移 */
    UPROPERTY(Transient, VisibleAnywhere, Category=SmartObjects)
    FVector3f Offset = FVector3f::ZeroVector;

    /** 槽位相对于 SmartObject 的旋转 */
    UPROPERTY(Transient, VisibleAnywhere, Category=SmartObjects)
    FRotator3f Rotation = FRotator3f::ZeroRotator;

    // ============================================
    // 状态数据
    // ============================================

    /** 当前状态（Free/Claimed/Occupied） */
    UPROPERTY(Transient, VisibleAnywhere, Category=SmartObjects)
    ESmartObjectSlotState State = ESmartObjectSlotState::Free;

    /** 使用者句柄 */
    UPROPERTY(Transient, VisibleAnywhere, Category=SmartObjects)
    FSmartObjectUserHandle User;

    /** 声明优先级 */
    UPROPERTY(Transient, VisibleAnywhere, Category=SmartObjects)
    ESmartObjectClaimPriority ClaimedPriority = ESmartObjectClaimPriority::None;

    /** 槽位启用状态 */
    UPROPERTY(Transient, VisibleAnywhere, Category=SmartObjects)
    uint8 bSlotEnabled : 1;

    /** 父对象启用状态 */
    UPROPERTY(Transient, VisibleAnywhere, Category=SmartObjects)
    uint8 bObjectEnabled : 1;

    /** 运行时标签 */
    UPROPERTY(Transient, VisibleAnywhere, Category=SmartObjects)
    FGameplayTagContainer Tags;

    /** 用户数据（声明/使用时传入的上下文数据） */
    FInstancedStruct UserData;

    /** 槽位状态数据（可在运行时添加） */
    FInstancedStructContainer StateData;

    /** 槽位失效回调 */
    FOnSlotInvalidated OnSlotInvalidatedDelegate;

    // ============================================
    // 关键方法
    // ============================================

    /** 获取槽位世界变换 */
    FTransform GetSlotWorldTransform(const FTransform& OwnerTransform) const
    {
        return FTransform(FRotator(Rotation), FVector(Offset)) * OwnerTransform;
    }

    /** 检查是否可以被声明 */
    bool CanBeClaimed(ESmartObjectClaimPriority ClaimPriority) const;

    /** 声明槽位 */
    bool Claim(const FSmartObjectUserHandle& InUser, ESmartObjectClaimPriority ClaimPriority);

    /** 释放槽位 */
    bool Release(const FSmartObjectClaimHandle& ClaimHandle, const bool bAborted);

    /** 检查是否启用 */
    bool IsEnabled() const { return bSlotEnabled && bObjectEnabled; }
};
```

**关键理解**：
- **bSlotEnabled 和 bObjectEnabled 都为 true 才可用**
- **UserData 存储使用者的上下文信息**（例如 AI 的引用）
- **StateData 允许运行时添加额外数据**

---

### 3. Handle 体系

#### FSmartObjectHandle（对象句柄）

**源码**: `SmartObjectTypes.h:152`

```cpp
USTRUCT(BlueprintType)
struct FSmartObjectHandle
{
    GENERATED_BODY()

    bool IsValid() const
    {
        return *this != Invalid;
    }

    friend FString LexToString(const FSmartObjectHandle Handle)
    {
        return *Handle.Guid.ToString(EGuidFormats::DigitsWithHyphensInBraces);
    }

private:
    /** 唯一标识符（GUID） */
    FGuid Guid;

    /** 只能由 Factory 创建 */
    friend struct FSmartObjectHandleFactory;
    explicit FSmartObjectHandle(const FGuid InID) : Guid(InID) {}

public:
    static const FSmartObjectHandle Invalid;
};
```

**用途**：唯一标识一个注册到 Subsystem 的 SmartObject 实例

---

#### FSmartObjectSlotHandle（槽位句柄）

**用途**：标识某个 SmartObject 的特定槽位

```cpp
struct FSmartObjectSlotHandle
{
    FSmartObjectHandle SmartObjectHandle;  // 所属的 SmartObject
    int32 SlotIndex;                       // 槽位索引
};
```

---

#### FSmartObjectUserHandle（用户句柄）

**源码**: `SmartObjectTypes.h:100`

```cpp
USTRUCT()
struct FSmartObjectUserHandle
{
    GENERATED_BODY()

    bool IsValid() const
    {
        return *this != Invalid;
    }

private:
    /** 由 Subsystem 分配的唯一 ID */
    uint32 ID = INDEX_NONE;

    friend class USmartObjectSubsystem;
    explicit FSmartObjectUserHandle(const uint32 InID) : ID(InID) {}

public:
    static const FSmartObjectUserHandle Invalid;
};
```

**用途**：唯一标识使用 SmartObject 的用户（通常是 AI）

---

#### FSmartObjectClaimHandle（声明句柄）

**源码**: `SmartObjectRuntime.h:50`

```cpp
USTRUCT(BlueprintType)
struct FSmartObjectClaimHandle
{
    GENERATED_BODY()

    /** 所属的 SmartObject */
    UPROPERTY(BlueprintReadOnly, Category="Default")
    FSmartObjectHandle SmartObjectHandle;

    /** 声明的槽位 */
    UPROPERTY(BlueprintReadOnly, Category="Default")
    FSmartObjectSlotHandle SlotHandle;

    /** 声明的用户 */
    UPROPERTY(Category="Default")
    FSmartObjectUserHandle UserHandle;

    bool IsValid() const
    {
        return SmartObjectHandle.IsValid()
            && SlotHandle.IsValid()
            && UserHandle.IsValid();
    }
};
```

**用途**：绑定 SmartObject、Slot 和 User 的三元组

**完整的 Handle 关系图**：

```
┌───────────────────────────────────────────────┐
│ Handle 体系                                    │
├───────────────────────────────────────────────┤
│                                               │
│  FSmartObjectHandle                           │
│  └─> 标识一个 SmartObject 实例                │
│       (例如: {12345678-1234-1234-1234...})    │
│                                               │
│         ↓ 包含                                │
│                                               │
│  FSmartObjectSlotHandle                       │
│  ├─ SmartObjectHandle (父对象)                │
│  └─ SlotIndex (槽位索引: 0, 1, 2...)          │
│                                               │
│         ↓ 声明后生成                          │
│                                               │
│  FSmartObjectClaimHandle                      │
│  ├─ SmartObjectHandle                         │
│  ├─ SlotHandle                                │
│  └─ UserHandle (谁声明的)                     │
│                                               │
│         ↓ 用于                                │
│                                               │
│  MarkSlotAsOccupied()                         │
│  MarkSlotAsFree()                             │
│  等运行时操作                                  │
│                                               │
└───────────────────────────────────────────────┘
```

---

## RuntimeFlow 完整流程

### 流程图

```
┌────────────────────────────────────────────────────────────┐
│ RuntimeFlow 完整流程                                         │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  阶段 1: 初始化与注册                                       │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━    │
│                                                            │
│  1. Actor BeginPlay                                        │
│      └─> USmartObjectComponent::BeginPlay()               │
│           └─> RegisterToSubsystem()                       │
│                └─> USmartObjectSubsystem::RegisterSmartObject()│
│                     ├─ 创建 FSmartObjectRuntime            │
│                     ├─ 生成 FSmartObjectHandle             │
│                     ├─ 初始化所有 Slots                    │
│                     └─ 添加到空间索引                      │
│                                                            │
│  ↓                                                         │
│                                                            │
│  阶段 2: AI 查询与选择                                      │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━    │
│                                                            │
│  2. AI 发起查询                                            │
│      └─> USmartObjectSubsystem::FindSmartObjects()        │
│           ├─ 空间查询（Octree/HashGrid）                  │
│           ├─ 标签过滤（ActivityTags）                     │
│           ├─ 条件验证（Preconditions）                    │
│           └─ 返回 FSmartObjectRequestResult[]             │
│                                                            │
│  3. AI 选择目标                                            │
│      └─ 从结果中选择一个 Slot                             │
│         (基于距离、优先级等)                               │
│                                                            │
│  ↓                                                         │
│                                                            │
│  阶段 3: 声明槽位 (Claim)                                   │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━    │
│                                                            │
│  4. AI 声明槽位                                            │
│      └─> USmartObjectSubsystem::MarkSlotAsClaimed()       │
│           ├─ 验证槽位状态（CanBeClaimed）                 │
│           ├─ 生成 FSmartObjectUserHandle                  │
│           ├─ FSmartObjectRuntimeSlot::Claim()             │
│           │   ├─ State = Claimed                          │
│           │   ├─ User = UserHandle                        │
│           │   └─ ClaimedPriority = Priority               │
│           ├─ 触发 OnClaimed 事件                          │
│           └─ 返回 FSmartObjectClaimHandle                 │
│                                                            │
│  ↓                                                         │
│                                                            │
│  阶段 4: 导航到槽位                                        │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━    │
│                                                            │
│  5. AI 移动到槽位                                          │
│      ├─ FindEntranceLocationForSlot()                     │
│      │   └─ 获取入口位置（Entry Location）                │
│      └─ AIMoveTo(EntranceLocation)                        │
│                                                            │
│  ↓                                                         │
│                                                            │
│  阶段 5: 开始使用 (Occupy)                                  │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━    │
│                                                            │
│  6. AI 到达槽位后开始使用                                   │
│      └─> USmartObjectSubsystem::MarkSlotAsOccupied()      │
│           ├─ 验证 ClaimHandle                             │
│           ├─ 验证 User 匹配                               │
│           ├─ State: Claimed → Occupied                    │
│           ├─ 触发 OnOccupied 事件                         │
│           └─ 返回 BehaviorDefinition                      │
│                                                            │
│  7. 执行行为                                               │
│      └─ 根据 BehaviorDefinition 执行                      │
│         ├─ GameplayBehavior (播放动画等)                  │
│         └─ GameplayInteraction (StateTree)                │
│                                                            │
│  ↓                                                         │
│                                                            │
│  阶段 6: 释放槽位 (Release)                                 │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━    │
│                                                            │
│  8. 行为完成或中止                                         │
│      └─> USmartObjectSubsystem::MarkSlotAsFree()          │
│           ├─ FSmartObjectRuntimeSlot::Release()           │
│           │   ├─ State: Occupied/Claimed → Free           │
│           │   ├─ User.Invalidate()                        │
│           │   ├─ UserData.Reset()                         │
│           │   └─ ClaimedPriority = None                   │
│           └─ 触发 OnReleased 事件                         │
│                                                            │
│  ↓                                                         │
│                                                            │
│  循环：槽位重新变为 Free，可被其他 AI 使用                  │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## 关键 API 源码分析

### 1. MarkSlotAsClaimed（声明槽位）

**源码位置**: `SmartObjectSubsystem.cpp:1324`

```cpp
FSmartObjectClaimHandle USmartObjectSubsystem::MarkSlotAsClaimed(
    const FSmartObjectSlotHandle& SlotHandle,
    ESmartObjectClaimPriority ClaimPriority,
    const FConstStructView UserData)
{
    if (!SlotHandle.IsValid())
    {
        UE_LOG(LogSmartObject, Log, TEXT("Invalid slot handle"));
        return FSmartObjectClaimHandle::InvalidHandle;
    }

    FSmartObjectClaimHandle OutClaimHandle(FSmartObjectClaimHandle::InvalidHandle);

    // 执行在有效的 Runtime 和 Slot 上
    ExecuteOnValidatedMutableRuntimeAndSlot(SlotHandle,
        [&OutClaimHandle, this, SlotHandle, ClaimPriority, UserData]
        (FSmartObjectRuntime& SmartObjectRuntime, FSmartObjectRuntimeSlot& Slot)
        {
            // 快速检查：槽位是否可以被声明
            if (!Slot.CanBeClaimed(ClaimPriority))
            {
                UE_LOG(LogSmartObject, Log,
                    TEXT("Can't claim slot '%s' - disabled or not free"),
                    *LexToString(SlotHandle));
                return;
            }

            // 如果当前已被声明（但优先级更低），先释放原有声明
            bool bIsClaimOverridden = false;
            if (Slot.GetState() == ESmartObjectSlotState::Claimed)
            {
                const FInstancedStruct Payload(MoveTemp(Slot.UserData));
                const FSmartObjectClaimHandle ExistingClaim(
                    SlotHandle.SmartObjectHandle, SlotHandle, Slot.User);

                // 释放原有声明（标记为中止）
                Slot.Release(ExistingClaim, /*bAborted*/ true);

                UE_LOG(LogSmartObject, Log,
                    TEXT("Released '%s' due to claim override"),
                    *LexToString(ExistingClaim));

                // 触发 OnReleased 事件
                OnSlotChangedInternal(SmartObjectRuntime, Slot, SlotHandle,
                    ESmartObjectChangeReason::OnReleased, Payload);

                bIsClaimOverridden = true;
            }

            // 生成新的 UserHandle
            const FSmartObjectUserHandle UserHandle =
                FSmartObjectUserHandle(NextFreeUserID++);

            // 执行 Claim
            const bool bSuccess = Slot.Claim(UserHandle, ClaimPriority);
            if (bSuccess)
            {
                // 存储 UserData
                Slot.UserData = FInstancedStruct(UserData);

                // 生成 ClaimHandle
                OutClaimHandle = FSmartObjectClaimHandle(
                    SlotHandle.SmartObjectHandle, SlotHandle, UserHandle);

                UE_LOG(LogSmartObject, Log, TEXT("Claimed using handle '%s'"),
                    *LexToString(OutClaimHandle));

                // 触发 OnClaimed 事件
                OnSlotChangedInternal(SmartObjectRuntime, Slot, SlotHandle,
                    ESmartObjectChangeReason::OnClaimed, Slot.UserData);
            }
        }, __FUNCTION__);

    return OutClaimHandle;
}
```

**关键步骤**：

1. **验证 SlotHandle**：检查句柄是否有效
2. **检查可声明性**：调用 `Slot.CanBeClaimed()`
3. **处理声明覆盖**：如果已被低优先级声明，先释放
4. **生成 UserHandle**：分配唯一的用户 ID
5. **执行 Claim**：调用 `Slot.Claim()`
6. **存储 UserData**：保存用户上下文
7. **触发事件**：通知监听者槽位已被声明
8. **返回 ClaimHandle**：用于后续操作

---

### 2. MarkSlotAsOccupied（开始使用）

**源码位置**: `SmartObjectSubsystem.cpp:1487`

```cpp
const USmartObjectBehaviorDefinition* USmartObjectSubsystem::MarkSlotAsOccupiedInternal(
    FSmartObjectRuntime& SmartObjectRuntime,
    const FSmartObjectClaimHandle& ClaimHandle,
    TSubclassOf<USmartObjectBehaviorDefinition> DefinitionClass)
{
    checkf(ClaimHandle.IsValid(), TEXT("Must be valid ClaimHandle"));

    // 检查 SmartObject 是否启用
    if (!SmartObjectRuntime.IsEnabled())
    {
        UE_LOG(LogSmartObject, Log,
            TEXT("Can't use handle '%s' - object is disabled"),
            *LexToString(ClaimHandle));
        return nullptr;
    }

    // 获取 BehaviorDefinition
    const USmartObjectBehaviorDefinition* BehaviorDefinition =
        GetBehaviorDefinitionInternal(SmartObjectRuntime,
            ClaimHandle.SlotHandle, DefinitionClass);

    if (BehaviorDefinition == nullptr)
    {
        UE_LOG(LogSmartObject, Warning,
            TEXT("Unable to find behavior definition of type '%s'"),
            *DefinitionClass.Get()->GetName());
        return nullptr;
    }

    UE_LOG(LogSmartObject, Log, TEXT("Start using handle '%s'"),
        *LexToString(ClaimHandle));

    FSmartObjectRuntimeSlot& Slot =
        SmartObjectRuntime.Slots[ClaimHandle.SlotHandle.GetSlotIndex()];

    // 验证状态和用户
    if (Slot.GetState() == ESmartObjectSlotState::Claimed)
    {
        if (Slot.User == ClaimHandle.UserHandle)
        {
            // ✅ 状态转换：Claimed → Occupied
            Slot.State = ESmartObjectSlotState::Occupied;

            // 触发 OnOccupied 事件
            OnSlotChangedInternal(SmartObjectRuntime, Slot,
                ClaimHandle.SlotHandle,
                ESmartObjectChangeReason::OnOccupied,
                Slot.UserData);

            return BehaviorDefinition;
        }

        // ❌ User 不匹配
        UE_LOG(LogSmartObject, Error,
            TEXT("Slot is already assigned to '%s', but trying to occupy with '%s'"),
            *LexToString(Slot.User), *LexToString(ClaimHandle.UserHandle));
    }
    else
    {
        // ❌ 状态不是 Claimed
        UE_LOG(LogSmartObject, Error,
            TEXT("State is expected to be 'Claimed', but it is '%s'"),
            *UEnum::GetValueAsString(Slot.GetState()));
    }

    return nullptr;
}
```

**关键步骤**：

1. **验证 ClaimHandle**：必须有效
2. **检查 Object 启用**：SmartObject 必须启用
3. **获取 BehaviorDefinition**：从 Definition 中查找指定类型的 Behavior
4. **验证 Slot 状态**：必须是 `Claimed`
5. **验证 User 匹配**：声明者和使用者必须一致
6. **状态转换**：`Claimed → Occupied`
7. **触发事件**：通知监听者槽位开始被使用
8. **返回 BehaviorDefinition**：供 AI 执行行为

---

### 3. MarkSlotAsFree（释放槽位）

**源码位置**: `SmartObjectSubsystem.cpp:1536`

```cpp
bool USmartObjectSubsystem::MarkSlotAsFree(const FSmartObjectClaimHandle& ClaimHandle)
{
    bool bOutReleased = false;

    ExecuteOnValidatedMutableRuntimeAndSlot(ClaimHandle.SlotHandle,
        [&bOutReleased, this, &ClaimHandle]
        (FSmartObjectRuntime& SmartObjectRuntime, FSmartObjectRuntimeSlot& Slot)
        {
            // 保存 UserData（用于事件通知）
            const FInstancedStruct Payload(MoveTemp(Slot.UserData));

            // 执行释放
            bOutReleased = Slot.Release(ClaimHandle, /*bAborted*/ false);

            if (bOutReleased)
            {
                UE_LOG(LogSmartObject, Log, TEXT("Released using handle '%s'"),
                    *LexToString(ClaimHandle));

                // 触发 OnReleased 事件
                OnSlotChangedInternal(SmartObjectRuntime, Slot,
                    ClaimHandle.SlotHandle,
                    ESmartObjectChangeReason::OnReleased,
                    Payload);
            }
        }, __FUNCTION__);

    return bOutReleased;
}
```

**对应的 Slot.Release() 实现**:

**源码位置**: `SmartObjectRuntime.cpp:146`

```cpp
bool FSmartObjectRuntimeSlot::Release(
    const FSmartObjectClaimHandle& ClaimHandle,
    const bool bAborted)
{
    if (!ensureMsgf(ClaimHandle.IsValid(),
        TEXT("Attempting to release with invalid handle")))
    {
        return false;
    }

    bool bReleased = false;

    // 验证状态
    if (State != ESmartObjectSlotState::Claimed &&
        State != ESmartObjectSlotState::Occupied)
    {
        UE_LOG(LogSmartObject, Error,
            TEXT("Expected state is 'Claimed' or 'Occupied', but it is '%s'"),
            *UEnum::GetValueAsString(State));
    }
    // 验证 User
    else if (ClaimHandle.UserHandle != User)
    {
        UE_LOG(LogSmartObject, Error,
            TEXT("User '%s' is trying to release slot claimed by '%s'"),
            *LexToString(ClaimHandle.UserHandle), *LexToString(User));
    }
    else
    {
        // 如果是中止（而非正常完成），触发回调
        if (bAborted)
        {
            const bool bExecuted =
                OnSlotInvalidatedDelegate.ExecuteIfBound(ClaimHandle, State);
            UE_LOG(LogSmartObject, Verbose,
                TEXT("Slot invalidated callback was%scalled"),
                bExecuted ? TEXT(" ") : TEXT(" not "));
        }

        // ✅ 状态转换：Claimed/Occupied → Free
        State = ESmartObjectSlotState::Free;
        User.Invalidate();
        UserData.Reset();
        ClaimedPriority = ESmartObjectClaimPriority::None;
        bReleased = true;
    }

    return bReleased;
}
```

**关键步骤**：

1. **验证 ClaimHandle**：必须有效
2. **验证状态**：必须是 `Claimed` 或 `Occupied`
3. **验证 User**：只有声明者才能释放
4. **处理中止情况**：如果 `bAborted = true`，触发回调
5. **状态转换**：`→ Free`
6. **清理数据**：User、UserData、Priority
7. **触发事件**：通知监听者槽位已释放

---

## 事件系统

### ESmartObjectChangeReason（变更原因）

**源码位置**: `SmartObjectTypes.h:712`

```cpp
enum class ESmartObjectChangeReason : uint8
{
    None,               // 无变更
    OnEvent,            // 外部事件
    OnTagAdded,         // 添加标签
    OnTagRemoved,       // 移除标签
    OnClaimed,          // 槽位被声明
    OnOccupied,         // 槽位开始被使用
    OnReleased,         // 槽位被释放
    OnSlotEnabled,      // 槽位启用
    OnSlotDisabled,     // 槽位禁用
    OnEnabled,          // 对象启用
    OnDisabled,         // 对象禁用
};
```

### FSmartObjectEventData（事件数据）

**源码位置**: `SmartObjectTypes.h:746`

```cpp
USTRUCT(BlueprintType)
struct FSmartObjectEventData
{
    GENERATED_BODY()

    /** 变更的 SmartObject */
    UPROPERTY(Transient, BlueprintReadOnly, Category = "SmartObject")
    FSmartObjectHandle SmartObjectHandle;

    /** 变更的 Slot（如果为空，表示是 Object 级别的事件） */
    UPROPERTY(Transient, BlueprintReadOnly, Category = "SmartObject")
    FSmartObjectSlotHandle SlotHandle;

    /** 变更原因 */
    UPROPERTY(Transient, BlueprintReadOnly, Category = "SmartObject")
    ESmartObjectChangeReason Reason = ESmartObjectChangeReason::None;

    /** 相关标签（根据 Reason 不同而不同） */
    UPROPERTY(Transient, BlueprintReadOnly, Category = "SmartObject")
    FGameplayTag Tag;

    /** 有效负载（例如 UserData） */
    FConstStructView Payload;
};
```

### 事件监听示例

```cpp
// 在 Runtime 上注册监听器
void MyClass::ListenToSmartObjectEvents()
{
    USmartObjectSubsystem* Subsystem =
        GetWorld()->GetSubsystem<USmartObjectSubsystem>();

    FSmartObjectHandle ObjectHandle = /* ... */;

    // 获取 Runtime
    FSmartObjectRuntime* Runtime = Subsystem->GetRuntime(ObjectHandle);
    if (Runtime)
    {
        // 绑定事件委托
        Runtime->GetMutableEventDelegate().AddLambda(
            [](const FSmartObjectEventData& Event)
            {
                switch (Event.Reason)
                {
                case ESmartObjectChangeReason::OnClaimed:
                    UE_LOG(LogTemp, Log, TEXT("Slot %s was claimed"),
                        *LexToString(Event.SlotHandle));
                    break;

                case ESmartObjectChangeReason::OnOccupied:
                    UE_LOG(LogTemp, Log, TEXT("Slot %s is now occupied"),
                        *LexToString(Event.SlotHandle));
                    break;

                case ESmartObjectChangeReason::OnReleased:
                    UE_LOG(LogTemp, Log, TEXT("Slot %s was released"),
                        *LexToString(Event.SlotHandle));
                    break;

                default:
                    break;
                }
            });
    }
}
```

---

## 线程安全说明

**源码位置**: `SmartObjectSubsystem.h:279`

### 官方说明

```cpp
/**
 * [Notes regarding thread safety]
 * The subsystem is not thread-safe, but a first pass has been made to make it
 * possible to perform a set of operations from multiple threads.
 * To use this mode the following compiler switch is required:
 * #define WITH_SMARTOBJECT_MT_INSTANCE_LOCK 1
 *
 * Not safe:
 *  - runtime instance lifetime controlled from Registration/Unregistration
 *    (i.e., CreateSmartObject, RegisterCollection, UnregisterCollection, etc.)
 *  - queries: to prevent locking for a long time it is still required to send
 *    queries from a single thread (e.g., async request pattern like MassSmartObject)
 *
 * Safe operation on a smart object instance or slot from an object or slot handle:
 *  - query and set Enable state
 *  - query and set Transform/Location
 *  - query and set Tags
 *  - update slot state (e.g., MarkSlotAsClaimed, MarkSlotAsReleased, etc.)
 *  - use a slot view using ReadSlotData/MutateSlotData
 */
```

### 多线程模式（实验性）

**启用条件**：`WITH_SMARTOBJECT_MT_INSTANCE_LOCK = 1`

**线程安全操作**：
- ✅ 查询/设置启用状态
- ✅ 查询/设置 Transform
- ✅ 查询/设置 Tags
- ✅ 更新 Slot 状态（Claim、Release 等）
- ✅ 使用 SlotView（ReadSlotData、MutateSlotData）

**非线程安全操作**：
- ❌ 注册/注销（RegisterCollection、UnregisterCollection）
- ❌ 创建/销毁（CreateSmartObject、DestroySmartObject）
- ❌ 空间查询（FindSmartObjects）

**建议**：
- 大多数项目使用**单线程模式**即可
- 如果需要多线程，参考 **MassSmartObject** 的异步请求模式

---

## 最佳实践

### 1. 正确的状态转换流程

```cpp
// ✅ 正确的流程
void UseSmartObject(FSmartObjectSlotHandle SlotHandle)
{
    USmartObjectSubsystem* Subsystem = GetWorld()->GetSubsystem<USmartObjectSubsystem>();

    // 1. 声明槽位
    FSmartObjectClaimHandle ClaimHandle =
        Subsystem->MarkSlotAsClaimed(SlotHandle, ESmartObjectClaimPriority::Normal);

    if (!ClaimHandle.IsValid())
    {
        UE_LOG(LogTemp, Warning, TEXT("Failed to claim slot"));
        return;
    }

    // 2. 移动到槽位
    // ... AI 移动逻辑 ...

    // 3. 开始使用
    const UGameplayBehaviorSmartObjectBehaviorDefinition* BehaviorDef =
        Subsystem->MarkSlotAsOccupied<UGameplayBehaviorSmartObjectBehaviorDefinition>(ClaimHandle);

    if (BehaviorDef == nullptr)
    {
        UE_LOG(LogTemp, Warning, TEXT("Failed to occupy slot"));
        Subsystem->MarkSlotAsFree(ClaimHandle);  // 释放声明
        return;
    }

    // 4. 执行行为
    // ... 执行 BehaviorDef ...

    // 5. 完成后释放
    Subsystem->MarkSlotAsFree(ClaimHandle);
}
```

---

### 2. 处理中断情况

```cpp
// ✅ 正确处理中断
void OnAIInterrupted(FSmartObjectClaimHandle ClaimHandle)
{
    if (!ClaimHandle.IsValid())
        return;

    USmartObjectSubsystem* Subsystem = GetWorld()->GetSubsystem<USmartObjectSubsystem>();

    // 检查槽位是否仍然有效
    if (Subsystem->IsClaimedSmartObjectValid(ClaimHandle))
    {
        // 释放槽位
        bool bReleased = Subsystem->MarkSlotAsFree(ClaimHandle);

        if (bReleased)
        {
            UE_LOG(LogTemp, Log, TEXT("Successfully released slot after interruption"));
        }
    }
}
```

---

### 3. 优先级覆盖

```cpp
// ✅ 高优先级可以抢占低优先级
void ClaimWithPriority(FSmartObjectSlotHandle SlotHandle)
{
    USmartObjectSubsystem* Subsystem = GetWorld()->GetSubsystem<USmartObjectSubsystem>();

    // 高优先级声明（会自动释放低优先级的声明）
    FSmartObjectClaimHandle ClaimHandle =
        Subsystem->MarkSlotAsClaimed(
            SlotHandle,
            ESmartObjectClaimPriority::High  // 高优先级
        );

    // 如果槽位已被低优先级声明，会触发：
    // 1. 原有声明的 OnSlotInvalidatedDelegate（如果有注册）
    // 2. OnReleased 事件
    // 3. 新的 OnClaimed 事件
}
```

---

### 4. 监听槽位失效

```cpp
// ✅ 注册槽位失效回调
void RegisterSlotInvalidationCallback(FSmartObjectClaimHandle ClaimHandle)
{
    USmartObjectSubsystem* Subsystem = GetWorld()->GetSubsystem<USmartObjectSubsystem>();

    Subsystem->MutateSlotData(ClaimHandle.SlotHandle,
        [this, ClaimHandle](FSmartObjectSlotView SlotView)
        {
            // 获取 Slot（通过 const_cast，因为 Delegate 不在公开 API 中）
            FSmartObjectRuntimeSlot* Slot =
                const_cast<FSmartObjectRuntimeSlot*>(SlotView.Slot);

            if (Slot)
            {
                // 注册回调
                Slot->OnSlotInvalidatedDelegate.BindLambda(
                    [this](const FSmartObjectClaimHandle& InvalidatedClaim,
                           ESmartObjectSlotState CurrentState)
                    {
                        UE_LOG(LogTemp, Warning,
                            TEXT("Slot was invalidated! State: %s"),
                            *UEnum::GetValueAsString(CurrentState));

                        // 处理失效（例如：中止当前行为）
                        HandleSlotInvalidated(InvalidatedClaim);
                    });
            }
        });
}
```

---

### 5. 验证槽位状态

```cpp
// ✅ 在关键操作前验证
void SafeUseSmartObject(FSmartObjectClaimHandle ClaimHandle)
{
    USmartObjectSubsystem* Subsystem = GetWorld()->GetSubsystem<USmartObjectSubsystem>();

    // 1. 验证 ClaimHandle 仍然有效
    if (!Subsystem->IsClaimedSmartObjectValid(ClaimHandle))
    {
        UE_LOG(LogTemp, Warning, TEXT("ClaimHandle is no longer valid"));
        return;
    }

    // 2. 验证槽位状态
    ESmartObjectSlotState State = Subsystem->GetSlotState(ClaimHandle.SlotHandle);
    if (State != ESmartObjectSlotState::Claimed)
    {
        UE_LOG(LogTemp, Warning, TEXT("Slot state is not Claimed: %s"),
            *UEnum::GetValueAsString(State));
        return;
    }

    // 3. 安全地标记为 Occupied
    const USmartObjectBehaviorDefinition* BehaviorDef =
        Subsystem->MarkSlotAsOccupied(ClaimHandle,
            UGameplayBehaviorSmartObjectBehaviorDefinition::StaticClass());

    if (BehaviorDef)
    {
        // 成功，继续执行
    }
}
```

---

### 6. 使用 SlotView 读取数据

```cpp
// ✅ 线程安全的 Slot 数据读取
void ReadSlotData(FSmartObjectSlotHandle SlotHandle)
{
    USmartObjectSubsystem* Subsystem = GetWorld()->GetSubsystem<USmartObjectSubsystem>();

    bool bSuccess = Subsystem->ReadSlotData(SlotHandle,
        [](FConstSmartObjectSlotView SlotView)
        {
            // 在这个 Lambda 内，Slot 数据被锁定（多线程模式下）

            // 读取槽位状态
            ESmartObjectSlotState State = SlotView.GetState();

            // 读取槽位标签
            const FGameplayTagContainer& Tags = SlotView.GetTags();

            // 读取 Definition 数据
            const USmartObjectDefinition& Def = SlotView.GetSmartObjectDefinition();

            UE_LOG(LogTemp, Log, TEXT("Slot State: %s, Tags: %s"),
                *UEnum::GetValueAsString(State),
                *Tags.ToString());
        });

    if (!bSuccess)
    {
        UE_LOG(LogTemp, Warning, TEXT("Failed to read slot data"));
    }
}
```

---

## 总结

### RuntimeFlow 核心要点

```
1. 状态机
   Free → Claimed → Occupied → Free

2. 关键 API
   ├─ MarkSlotAsClaimed()   (声明)
   ├─ MarkSlotAsOccupied()  (开始使用)
   └─ MarkSlotAsFree()      (释放)

3. Handle 体系
   ├─ FSmartObjectHandle        (对象)
   ├─ FSmartObjectSlotHandle    (槽位)
   ├─ FSmartObjectUserHandle    (用户)
   └─ FSmartObjectClaimHandle   (声明绑定)

4. 事件系统
   ├─ OnClaimed   (槽位被声明)
   ├─ OnOccupied  (开始使用)
   └─ OnReleased  (释放)

5. 数据结构
   ├─ FSmartObjectRuntime       (对象运行时)
   └─ FSmartObjectRuntimeSlot   (槽位运行时)
```

### 架构优势

| 优势 | 说明 |
|------|------|
| **解耦** | Definition（资产）和 Runtime（运行时）分离 |
| **可扩展** | 支持多种 BehaviorDefinition 类型 |
| **事件驱动** | 状态变更自动通知监听者 |
| **优先级管理** | 支持高优先级抢占低优先级 |
| **线程安全（可选）** | 支持多线程访问（实验性） |

---

## 附录

### 相关文档

- [SmartObject 完整调用流程](./SmartObject完整调用流程.md)
- [SmartObject 实战教程](./SmartObject实战教程_AI坐椅子.md)
- [BehaviorDefinition 关系与注册机制](../../../E:/Data/screenshot/WorkSpot截图/View/SmartObject_BehaviorDefinition关系与注册机制.md)

### 调试命令

```cpp
// 显示所有已注册的 SmartObject
showdebug SmartObject

// 颜色说明：
// - 绿色 = Free
// - 黄色 = Claimed
// - 红色 = Occupied
// - 灰色 = Disabled
```

---

**文档结束** 📄
