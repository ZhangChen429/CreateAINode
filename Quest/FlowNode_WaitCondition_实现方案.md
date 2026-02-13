# FlowGraph - Wait Until Condition 节点实现方案

## 一、设计概览

基于FlowGraph插件的异步架构，实现类似Cyberpunk 2077的条件等待系统。

### 1.1 核心优势

利用FlowGraph的原生异步设计：
- ✅ **天生异步** - FlowNode本身就是异步设计
- ✅ **事件驱动** - 通过Delegate系统实现
- ✅ **状态管理** - EFlowNodeState已有完整状态机
- ✅ **SaveGame** - 内置序列化支持
- ✅ **网络同步** - 支持Replicated
- ✅ **编辑器集成** - 自动获得可视化调试

### 1.2 与2077系统对比

| 2077概念 | FlowGraph实现 |
|----------|---------------|
| PauseConditionNode | FlowNode_WaitCondition |
| IBaseCondition | UFlowConditionBase |
| EventListener | FConditionDelegate + Timer |
| NodeResult::StayInNode | 保持Active状态 |
| NodeResult::LeaveNode | TriggerOutput() + Finish() |
| TokensState | EFlowNodeState |
| SectionNode | FlowNode_PlayLevelSequence |

---

## 二、文件结构

```
D:\Data\UEProject\Workspot\Plugins\FlowNode\
├── FlowNode.uplugin
├── Source/
│   └── FlowNode/
│       ├── FlowNode.Build.cs
│       ├── Public/
│       │   ├── Conditions/
│       │   │   ├── FlowConditionBase.h              // 条件基类
│       │   │   ├── FlowCondition_Timer.h            // 定时器条件
│       │   │   ├── FlowCondition_ActorState.h       // Actor状态条件
│       │   │   ├── FlowCondition_GameplayTag.h      // GameplayTag条件
│       │   │   ├── FlowCondition_LevelSequence.h    // 序列条件
│       │   │   ├── FlowCondition_Custom.h           // 自定义条件
│       │   │   └── FlowCondition_Composite.h        // 组合条件(AND/OR)
│       │   ├── Nodes/
│       │   │   └── FlowNode_WaitCondition.h         // Wait条件节点
│       │   └── Types/
│       │       └── FlowConditionTypes.h             // 枚举和结构体
│       └── Private/
│           ├── Conditions/
│           │   ├── FlowConditionBase.cpp
│           │   ├── FlowCondition_Timer.cpp
│           │   ├── FlowCondition_ActorState.cpp
│           │   ├── FlowCondition_GameplayTag.cpp
│           │   ├── FlowCondition_LevelSequence.cpp
│           │   ├── FlowCondition_Custom.cpp
│           │   └── FlowCondition_Composite.cpp
│           └── Nodes/
│               └── FlowNode_WaitCondition.cpp
```

---

## 三、核心类设计

### 3.1 条件基类 - FlowConditionBase.h

```cpp
#pragma once

#include "CoreMinimal.h"
#include "UObject/Object.h"
#include "FlowConditionTypes.h"
#include "FlowConditionBase.generated.h"

class UFlowNode_WaitCondition;

/**
 * 条件评估结果
 */
UENUM(BlueprintType)
enum class EFlowConditionResult : uint8
{
    NotFulfilled    UMETA(DisplayName = "Not Fulfilled"),      // 未满足
    Fulfilled       UMETA(DisplayName = "Fulfilled"),          // 已满足
    Failed          UMETA(DisplayName = "Failed"),             // 失败（永远无法满足）
    Cancelled       UMETA(DisplayName = "Cancelled")           // 取消
};

/**
 * 条件检查模式
 */
UENUM(BlueprintType)
enum class EFlowConditionCheckMode : uint8
{
    EventDriven     UMETA(DisplayName = "Event Driven"),       // 事件驱动（推荐）
    PollingEveryTick UMETA(DisplayName = "Polling Every Tick"), // 每帧轮询
    PollingInterval UMETA(DisplayName = "Polling Interval")    // 间隔轮询
};

/**
 * 条件上下文
 */
USTRUCT(BlueprintType)
struct FFlowConditionContext
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    TObjectPtr<UWorld> World = nullptr;

    UPROPERTY(BlueprintReadWrite)
    TObjectPtr<AActor> Instigator = nullptr;

    UPROPERTY(BlueprintReadWrite)
    TMap<FName, FString> Parameters;
};

/**
 * 条件基类 - 所有FlowCondition的抽象基类
 *
 * 设计理念：
 * - 支持事件驱动和轮询两种模式
 * - 可序列化，支持SaveGame
 * - 提供调试信息
 */
UCLASS(Abstract, Blueprintable, EditInlineNew)
class FLOWNODE_API UFlowConditionBase : public UObject
{
    GENERATED_BODY()

public:
    UFlowConditionBase();

    // ========================================================================
    // 生命周期
    // ========================================================================

    /**
     * 初始化条件
     * @param Context 条件上下文
     * @param OwnerNode 拥有此条件的节点
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Flow|Condition")
    void InitializeCondition(const FFlowConditionContext& Context, UFlowNode_WaitCondition* OwnerNode);
    virtual void InitializeCondition_Implementation(const FFlowConditionContext& Context, UFlowNode_WaitCondition* OwnerNode);

    /**
     * 清理条件资源（取消订阅、清理Timer等）
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Flow|Condition")
    void CleanupCondition();
    virtual void CleanupCondition_Implementation();

    // ========================================================================
    // 条件评估
    // ========================================================================

    /**
     * 评估条件是否满足
     * @return 条件结果
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Flow|Condition")
    EFlowConditionResult EvaluateCondition(const FFlowConditionContext& Context);
    virtual EFlowConditionResult EvaluateCondition_Implementation(const FFlowConditionContext& Context);

    /**
     * 获取条件进度（0.0 - 1.0）
     * 可用于UI显示或调试
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Flow|Condition")
    float GetProgress() const;
    virtual float GetProgress_Implementation() const { return 0.0f; }

    // ========================================================================
    // 检查模式
    // ========================================================================

    /** 条件检查模式 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Condition")
    EFlowConditionCheckMode CheckMode = EFlowConditionCheckMode::EventDriven;

    /** 轮询间隔（仅当CheckMode为PollingInterval时有效）*/
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Condition",
              meta = (EditCondition = "CheckMode == EFlowConditionCheckMode::PollingInterval", ClampMin = "0.1"))
    float PollingInterval = 0.5f;

    // ========================================================================
    // 调试信息
    // ========================================================================

    /**
     * 获取条件的友好描述（用于编辑器显示）
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Flow|Condition")
    FString GetConditionDescription() const;
    virtual FString GetConditionDescription_Implementation() const;

    /**
     * 获取条件的调试字符串
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Flow|Condition")
    FString GetDebugString() const;
    virtual FString GetDebugString_Implementation() const;

    // ========================================================================
    // SaveGame支持
    // ========================================================================

    /**
     * 保存条件状态
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Flow|Condition")
    void SaveConditionState(FFlowConditionSaveData& OutSaveData);
    virtual void SaveConditionState_Implementation(FFlowConditionSaveData& OutSaveData);

    /**
     * 加载条件状态
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Flow|Condition")
    void LoadConditionState(const FFlowConditionSaveData& SaveData);
    virtual void LoadConditionState_Implementation(const FFlowConditionSaveData& SaveData);

protected:
    // ========================================================================
    // 内部状态
    // ========================================================================

    /** 当前条件结果（缓存） */
    UPROPERTY(Transient)
    EFlowConditionResult CachedResult = EFlowConditionResult::NotFulfilled;

    /** 拥有此条件的节点 */
    UPROPERTY(Transient)
    TObjectPtr<UFlowNode_WaitCondition> OwnerNode = nullptr;

    /** 条件上下文 */
    UPROPERTY(Transient)
    FFlowConditionContext ConditionContext;

    /** 轮询Timer句柄 */
    FTimerHandle PollingTimerHandle;

    // ========================================================================
    // 辅助方法
    // ========================================================================

    /**
     * 通知节点条件已改变
     * 子类在检测到条件变化时应调用此方法
     */
    void NotifyConditionChanged(EFlowConditionResult NewResult);

    /**
     * 启动轮询检查
     */
    void StartPolling();

    /**
     * 停止轮询检查
     */
    void StopPolling();

    /**
     * 轮询回调
     */
    void OnPollingTick();
};

/**
 * 条件保存数据
 */
USTRUCT(BlueprintType)
struct FFlowConditionSaveData
{
    GENERATED_BODY()

    UPROPERTY(SaveGame)
    TSubclassOf<UFlowConditionBase> ConditionClass;

    UPROPERTY(SaveGame)
    TArray<uint8> ConditionData;

    UPROPERTY(SaveGame)
    float ElapsedTime = 0.0f;
};
```

### 3.2 Wait条件节点 - FlowNode_WaitCondition.h

```cpp
#pragma once

#include "CoreMinimal.h"
#include "Nodes/FlowNode.h"
#include "Conditions/FlowConditionBase.h"
#include "FlowNode_WaitCondition.generated.h"

/**
 * Wait Until Condition 节点
 *
 * 功能：
 * - 暂停执行流，直到指定条件满足
 * - 支持超时机制
 * - 支持取消操作
 * - 完整的SaveGame支持
 * - 可视化调试支持
 *
 * 输入Pin：
 * - In: 开始等待条件
 * - Cancel: 取消等待
 *
 * 输出Pin：
 * - Fulfilled: 条件满足
 * - Timeout: 超时
 * - Cancelled: 被取消
 * - Failed: 条件失败
 */
UCLASS(NotBlueprintable, meta = (DisplayName = "Wait Until Condition"))
class FLOWNODE_API UFlowNode_WaitCondition : public UFlowNode
{
    GENERATED_BODY()

public:
    UFlowNode_WaitCondition();

    // ========================================================================
    // FlowNode Override
    // ========================================================================

    virtual void ExecuteInput(const FName& PinName) override;
    virtual void Cleanup() override;
    virtual void OnSave_Implementation() override;
    virtual void OnLoad_Implementation() override;

#if WITH_EDITOR
    virtual FString GetNodeDescription() const override;
    virtual EDataValidationResult ValidateNode() override;
#endif

    // ========================================================================
    // 条件配置
    // ========================================================================

    /**
     * 要等待的条件
     * 使用Instanced支持多种条件类型
     */
    UPROPERTY(EditAnywhere, Instanced, Category = "Condition",
              meta = (ShowOnlyInnerProperties))
    TObjectPtr<UFlowConditionBase> Condition;

    /**
     * 超时时间（秒）
     * -1表示无超时
     */
    UPROPERTY(EditAnywhere, Category = "Condition", meta = (ClampMin = "-1.0"))
    float TimeoutDuration = -1.0f;

    /**
     * 是否在超时后自动完成节点
     */
    UPROPERTY(EditAnywhere, Category = "Condition",
              meta = (EditCondition = "TimeoutDuration > 0"))
    bool bFinishOnTimeout = true;

    /**
     * 是否显示调试信息
     */
    UPROPERTY(EditAnywhere, Category = "Debug")
    bool bShowDebugInfo = false;

    // ========================================================================
    // 回调接口
    // ========================================================================

    /**
     * 条件状态变化回调
     * 由UFlowConditionBase调用
     */
    void OnConditionChanged(EFlowConditionResult NewResult);

protected:
    // ========================================================================
    // 内部方法
    // ========================================================================

    /** 启动条件等待 */
    void StartWaiting();

    /** 停止条件等待 */
    void StopWaiting();

    /** 超时回调 */
    void OnTimeout();

    /** 处理条件满足 */
    void HandleConditionFulfilled();

    /** 处理条件失败 */
    void HandleConditionFailed();

    /** 处理超时 */
    void HandleTimeout();

    /** 处理取消 */
    void HandleCancelled();

    // ========================================================================
    // SaveGame数据
    // ========================================================================

    /** 保存的剩余超时时间 */
    UPROPERTY(SaveGame)
    float RemainingTimeoutTime = 0.0f;

    /** 保存的条件状态 */
    UPROPERTY(SaveGame)
    FFlowConditionSaveData SavedConditionState;

    /** 是否正在等待 */
    UPROPERTY(SaveGame)
    bool bIsWaiting = false;

private:
    // ========================================================================
    // 运行时状态
    // ========================================================================

    /** 超时Timer句柄 */
    FTimerHandle TimeoutTimerHandle;

    /** 等待开始时间 */
    float WaitStartTime = 0.0f;
};
```

---

## 四、具体条件实现示例

### 4.1 定时器条件 - FlowCondition_Timer.h

```cpp
#pragma once

#include "Conditions/FlowConditionBase.h"
#include "FlowCondition_Timer.generated.h"

/**
 * 定时器条件 - 等待指定时间
 */
UCLASS(meta = (DisplayName = "Timer Condition"))
class FLOWNODE_API UFlowCondition_Timer : public UFlowConditionBase
{
    GENERATED_BODY()

public:
    UFlowCondition_Timer();

    // ========================================================================
    // 配置
    // ========================================================================

    /** 等待时长（秒） */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Timer", meta = (ClampMin = "0.0"))
    float Duration = 1.0f;

    /** 是否使用真实时间（不受时间膨胀影响） */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Timer")
    bool bUseRealTime = false;

    // ========================================================================
    // Override
    // ========================================================================

    virtual void InitializeCondition_Implementation(const FFlowConditionContext& Context, UFlowNode_WaitCondition* OwnerNode) override;
    virtual void CleanupCondition_Implementation() override;
    virtual EFlowConditionResult EvaluateCondition_Implementation(const FFlowConditionContext& Context) override;
    virtual float GetProgress_Implementation() const override;
    virtual FString GetConditionDescription_Implementation() const override;
    virtual void SaveConditionState_Implementation(FFlowConditionSaveData& OutSaveData) override;
    virtual void LoadConditionState_Implementation(const FFlowConditionSaveData& SaveData) override;

protected:
    /** Timer句柄 */
    FTimerHandle TimerHandle;

    /** 开始时间 */
    float StartTime = 0.0f;

    /** 剩余时间（用于SaveGame） */
    float RemainingTime = 0.0f;

    /** Timer完成回调 */
    void OnTimerComplete();
};
```

### 4.2 Actor状态条件 - FlowCondition_ActorState.h

```cpp
#pragma once

#include "Conditions/FlowConditionBase.h"
#include "GameplayTagContainer.h"
#include "FlowCondition_ActorState.generated.h"

/**
 * Actor状态条件类型
 */
UENUM(BlueprintType)
enum class EActorStateConditionType : uint8
{
    ReachLocation       UMETA(DisplayName = "Reach Location"),      // 到达位置
    InRange             UMETA(DisplayName = "In Range"),            // 在范围内
    HasTag              UMETA(DisplayName = "Has Tag"),             // 拥有Tag
    IsDead              UMETA(DisplayName = "Is Dead"),             // 死亡
    HealthBelow         UMETA(DisplayName = "Health Below"),        // 生命值低于
    Custom              UMETA(DisplayName = "Custom")               // 自定义
};

/**
 * Actor状态条件
 */
UCLASS(meta = (DisplayName = "Actor State Condition"))
class FLOWNODE_API UFlowCondition_ActorState : public UFlowConditionBase
{
    GENERATED_BODY()

public:
    UFlowCondition_ActorState();

    // ========================================================================
    // 配置
    // ========================================================================

    /** 条件类型 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Actor State")
    EActorStateConditionType ConditionType = EActorStateConditionType::ReachLocation;

    /** 目标Actor（通过FlowComponent的IdentityTag查找） */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Actor State")
    FGameplayTag ActorIdentityTag;

    /** 或直接引用Actor */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Actor State")
    TSoftObjectPtr<AActor> DirectActorReference;

    // ReachLocation参数
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Location",
              meta = (EditCondition = "ConditionType == EActorStateConditionType::ReachLocation"))
    FVector TargetLocation = FVector::ZeroVector;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Location",
              meta = (EditCondition = "ConditionType == EActorStateConditionType::ReachLocation", ClampMin = "0.0"))
    float AcceptanceRadius = 100.0f;

    // InRange参数
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Range",
              meta = (EditCondition = "ConditionType == EActorStateConditionType::InRange"))
    TSoftObjectPtr<AActor> RangeTargetActor;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Range",
              meta = (EditCondition = "ConditionType == EActorStateConditionType::InRange", ClampMin = "0.0"))
    float RequiredRange = 500.0f;

    // HasTag参数
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Tags",
              meta = (EditCondition = "ConditionType == EActorStateConditionType::HasTag"))
    FGameplayTag RequiredTag;

    // HealthBelow参数
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Health",
              meta = (EditCondition = "ConditionType == EActorStateConditionType::HealthBelow", ClampMin = "0.0"))
    float HealthThreshold = 50.0f;

    // ========================================================================
    // Override
    // ========================================================================

    virtual void InitializeCondition_Implementation(const FFlowConditionContext& Context, UFlowNode_WaitCondition* OwnerNode) override;
    virtual void CleanupCondition_Implementation() override;
    virtual EFlowConditionResult EvaluateCondition_Implementation(const FFlowConditionContext& Context) override;
    virtual float GetProgress_Implementation() const override;
    virtual FString GetConditionDescription_Implementation() const override;

protected:
    /** 缓存的目标Actor */
    UPROPERTY(Transient)
    TObjectPtr<AActor> CachedTargetActor = nullptr;

    /** 解析目标Actor */
    AActor* ResolveTargetActor(const FFlowConditionContext& Context);

    /** Actor移动回调 */
    UFUNCTION()
    void OnActorMoved(AActor* Actor);

    /** Actor标签改变回调 */
    UFUNCTION()
    void OnActorTagsChanged();
};
```

### 4.3 关卡序列条件 - FlowCondition_LevelSequence.h

```cpp
#pragma once

#include "Conditions/FlowConditionBase.h"
#include "LevelSequence.h"
#include "LevelSequencePlayer.h"
#include "FlowCondition_LevelSequence.generated.h"

/**
 * 序列条件类型
 */
UENUM(BlueprintType)
enum class ESequenceConditionType : uint8
{
    IsPlaying       UMETA(DisplayName = "Is Playing"),          // 正在播放
    HasFinished     UMETA(DisplayName = "Has Finished"),        // 已完成
    ReachedTime     UMETA(DisplayName = "Reached Time"),        // 到达指定时间
    ReachedMarker   UMETA(DisplayName = "Reached Marker")       // 到达标记
};

/**
 * 关卡序列条件
 */
UCLASS(meta = (DisplayName = "Level Sequence Condition"))
class FLOWNODE_API UFlowCondition_LevelSequence : public UFlowConditionBase
{
    GENERATED_BODY()

public:
    UFlowCondition_LevelSequence();

    // ========================================================================
    // 配置
    // ========================================================================

    /** 条件类型 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Sequence")
    ESequenceConditionType ConditionType = ESequenceConditionType::HasFinished;

    /** 关卡序列资产 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Sequence")
    TSoftObjectPtr<ULevelSequence> LevelSequence;

    /** 目标时间（秒） */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Sequence",
              meta = (EditCondition = "ConditionType == ESequenceConditionType::ReachedTime"))
    float TargetTime = 0.0f;

    /** 目标标记名称 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Sequence",
              meta = (EditCondition = "ConditionType == ESequenceConditionType::ReachedMarker"))
    FName MarkerName;

    /** 是否反转条件（例如：不在播放） */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Sequence")
    bool bInvertCondition = false;

    // ========================================================================
    // Override
    // ========================================================================

    virtual void InitializeCondition_Implementation(const FFlowConditionContext& Context, UFlowNode_WaitCondition* OwnerNode) override;
    virtual void CleanupCondition_Implementation() override;
    virtual EFlowConditionResult EvaluateCondition_Implementation(const FFlowConditionContext& Context) override;
    virtual FString GetConditionDescription_Implementation() const override;

protected:
    /** 缓存的序列播放器 */
    UPROPERTY(Transient)
    TObjectPtr<ULevelSequencePlayer> CachedPlayer = nullptr;

    /** 查找序列播放器 */
    ULevelSequencePlayer* FindSequencePlayer(const FFlowConditionContext& Context);

    /** 序列完成回调 */
    UFUNCTION()
    void OnSequenceFinished();
};
```

### 4.4 组合条件 - FlowCondition_Composite.h

```cpp
#pragma once

#include "Conditions/FlowConditionBase.h"
#include "FlowCondition_Composite.generated.h"

/**
 * 组合类型
 */
UENUM(BlueprintType)
enum class ECompositeConditionType : uint8
{
    And     UMETA(DisplayName = "AND - All must be fulfilled"),
    Or      UMETA(DisplayName = "OR - Any must be fulfilled"),
    Not     UMETA(DisplayName = "NOT - Must not be fulfilled"),
    Xor     UMETA(DisplayName = "XOR - Exactly one must be fulfilled")
};

/**
 * 组合条件 - 组合多个子条件
 */
UCLASS(meta = (DisplayName = "Composite Condition (AND/OR/NOT)"))
class FLOWNODE_API UFlowCondition_Composite : public UFlowConditionBase
{
    GENERATED_BODY()

public:
    UFlowCondition_Composite();

    // ========================================================================
    // 配置
    // ========================================================================

    /** 组合类型 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Composite")
    ECompositeConditionType CompositeType = ECompositeConditionType::And;

    /** 子条件列表 */
    UPROPERTY(EditAnywhere, Instanced, BlueprintReadWrite, Category = "Composite")
    TArray<TObjectPtr<UFlowConditionBase>> ChildConditions;

    // ========================================================================
    // Override
    // ========================================================================

    virtual void InitializeCondition_Implementation(const FFlowConditionContext& Context, UFlowNode_WaitCondition* OwnerNode) override;
    virtual void CleanupCondition_Implementation() override;
    virtual EFlowConditionResult EvaluateCondition_Implementation(const FFlowConditionContext& Context) override;
    virtual float GetProgress_Implementation() const override;
    virtual FString GetConditionDescription_Implementation() const override;

protected:
    /** 子条件状态变化回调 */
    void OnChildConditionChanged(EFlowConditionResult NewResult);
};
```

---

## 五、实现文件示例

### 5.1 FlowNode_WaitCondition.cpp (核心实现)

```cpp
#include "Nodes/FlowNode_WaitCondition.h"
#include "Conditions/FlowConditionBase.h"

#define LOCTEXT_NAMESPACE "FlowNode_WaitCondition"

UFlowNode_WaitCondition::UFlowNode_WaitCondition()
{
    Category = TEXT("Condition");

#if WITH_EDITOR
    NodeStyle = TEXT("Condition.WaitCondition");
#endif

    // 定义输入Pin
    InputPins.Add(FFlowPin(TEXT("In")));
    InputPins.Add(FFlowPin(TEXT("Cancel")));

    // 定义输出Pin
    OutputPins.Add(FFlowPin(TEXT("Fulfilled")));
    OutputPins.Add(FFlowPin(TEXT("Timeout")));
    OutputPins.Add(FFlowPin(TEXT("Cancelled")));
    OutputPins.Add(FFlowPin(TEXT("Failed")));
}

void UFlowNode_WaitCondition::ExecuteInput(const FName& PinName)
{
    if (PinName == TEXT("In"))
    {
        // 验证条件是否存在
        if (!Condition)
        {
            LogError(TEXT("No condition specified!"));
            HandleConditionFailed();
            return;
        }

        // 开始等待
        StartWaiting();
    }
    else if (PinName == TEXT("Cancel"))
    {
        HandleCancelled();
    }
}

void UFlowNode_WaitCondition::StartWaiting()
{
    bIsWaiting = true;
    WaitStartTime = GetWorld()->GetTimeSeconds();

    // 初始化条件上下文
    FFlowConditionContext Context;
    Context.World = GetWorld();
    Context.Instigator = TryGetRootFlowActorOwner();

    // 初始化条件
    Condition->InitializeCondition(Context, this);

    // 立即检查一次条件
    EFlowConditionResult Result = Condition->EvaluateCondition(Context);
    if (Result == EFlowConditionResult::Fulfilled)
    {
        HandleConditionFulfilled();
        return;
    }
    else if (Result == EFlowConditionResult::Failed)
    {
        HandleConditionFailed();
        return;
    }

    // 设置超时Timer
    if (TimeoutDuration > 0.0f)
    {
        GetWorld()->GetTimerManager().SetTimer(
            TimeoutTimerHandle,
            this,
            &UFlowNode_WaitCondition::OnTimeout,
            TimeoutDuration,
            false
        );
    }

    // 节点保持Active状态，等待条件满足
}

void UFlowNode_WaitCondition::StopWaiting()
{
    bIsWaiting = false;

    // 清理超时Timer
    if (TimeoutTimerHandle.IsValid())
    {
        GetWorld()->GetTimerManager().ClearTimer(TimeoutTimerHandle);
        TimeoutTimerHandle.Invalidate();
    }

    // 清理条件
    if (Condition)
    {
        Condition->CleanupCondition();
    }
}

void UFlowNode_WaitCondition::OnConditionChanged(EFlowConditionResult NewResult)
{
    if (!bIsWaiting)
    {
        return;
    }

    switch (NewResult)
    {
    case EFlowConditionResult::Fulfilled:
        HandleConditionFulfilled();
        break;

    case EFlowConditionResult::Failed:
        HandleConditionFailed();
        break;

    case EFlowConditionResult::Cancelled:
        HandleCancelled();
        break;

    default:
        break;
    }
}

void UFlowNode_WaitCondition::HandleConditionFulfilled()
{
    if (bShowDebugInfo)
    {
        float ElapsedTime = GetWorld()->GetTimeSeconds() - WaitStartTime;
        LogNote(FString::Printf(TEXT("Condition fulfilled after %.2f seconds"), ElapsedTime));
    }

    StopWaiting();
    TriggerOutput(TEXT("Fulfilled"), true);  // true = 完成节点
}

void UFlowNode_WaitCondition::HandleConditionFailed()
{
    LogError(TEXT("Condition evaluation failed!"));
    StopWaiting();
    TriggerOutput(TEXT("Failed"), true);
}

void UFlowNode_WaitCondition::HandleTimeout()
{
    if (bShowDebugInfo)
    {
        LogNote(FString::Printf(TEXT("Condition timed out after %.2f seconds"), TimeoutDuration));
    }

    StopWaiting();
    TriggerOutput(TEXT("Timeout"), bFinishOnTimeout);
}

void UFlowNode_WaitCondition::HandleCancelled()
{
    LogNote(TEXT("Condition waiting cancelled"));
    StopWaiting();
    TriggerOutput(TEXT("Cancelled"), true);
}

void UFlowNode_WaitCondition::OnTimeout()
{
    HandleTimeout();
}

void UFlowNode_WaitCondition::Cleanup()
{
    StopWaiting();
    Super::Cleanup();
}

void UFlowNode_WaitCondition::OnSave_Implementation()
{
    Super::OnSave_Implementation();

    // 保存剩余超时时间
    if (TimeoutTimerHandle.IsValid())
    {
        RemainingTimeoutTime = GetWorld()->GetTimerManager().GetTimerRemaining(TimeoutTimerHandle);
    }

    // 保存条件状态
    if (Condition)
    {
        Condition->SaveConditionState(SavedConditionState);
    }
}

void UFlowNode_WaitCondition::OnLoad_Implementation()
{
    Super::OnLoad_Implementation();

    if (!bIsWaiting)
    {
        return;
    }

    // 恢复条件状态
    if (Condition && SavedConditionState.ConditionClass)
    {
        FFlowConditionContext Context;
        Context.World = GetWorld();
        Context.Instigator = TryGetRootFlowActorOwner();

        Condition->InitializeCondition(Context, this);
        Condition->LoadConditionState(SavedConditionState);
    }

    // 恢复超时Timer
    if (RemainingTimeoutTime > 0.0f)
    {
        GetWorld()->GetTimerManager().SetTimer(
            TimeoutTimerHandle,
            this,
            &UFlowNode_WaitCondition::OnTimeout,
            RemainingTimeoutTime,
            false
        );
    }
}

#if WITH_EDITOR
FString UFlowNode_WaitCondition::GetNodeDescription() const
{
    if (Condition)
    {
        return Condition->GetConditionDescription();
    }
    return TEXT("Wait Until Condition");
}

EDataValidationResult UFlowNode_WaitCondition::ValidateNode()
{
    EDataValidationResult Result = Super::ValidateNode();

    if (!Condition)
    {
        ValidationLog.Error<UFlowNode>(TEXT("No condition specified!"), this);
        Result = EDataValidationResult::Invalid;
    }

    return Result;
}
#endif

#undef LOCTEXT_NAMESPACE
```

---

## 六、使用示例

### 6.1 等待玩家到达位置

```
[StartQuest] → [WaitCondition: PlayerAtLocation] → [OpenDoor]
                        ↓
               [ActorState Condition]
               - Type: ReachLocation
               - Target: PlayerCharacter
               - Location: (1000, 2000, 100)
               - Radius: 200
```

### 6.2 组合条件：位置+事件

```
[QuestStart] → [WaitCondition: CompleteObjective] → [QuestComplete]
                        ↓
               [Composite Condition: AND]
               ├─ ActorState: PlayerAtLocation
               └─ GameplayTag: Event.EnemiesDefeated
```

### 6.3 等待过场动画

```
[TriggerCutscene] → [PlayLevelSequence] → [WaitCondition: CutsceneFinished] → [Continue]
                                                   ↓
                                          [LevelSequence Condition]
                                          - Type: HasFinished
                                          - Sequence: MySequence
```

---

## 七、优势分析

### 7.1 相比Blueprint的优势

| 特性 | Blueprint | FlowNode WaitCondition |
|------|-----------|------------------------|
| 异步等待 | Delay节点（阻塞） | 事件驱动（非阻塞） |
| 条件组合 | 手动连接分支 | Composite自动处理 |
| SaveGame | 手动实现 | 内置支持 |
| 可视化调试 | 有限 | 完整的状态显示 |
| 网络同步 | 复杂 | 自动处理 |

### 7.2 相比2077系统的改进

1. **更好的UE集成**: 利用GameplayTag、LevelSequence等UE原生系统
2. **编辑器友好**: Instanced属性，所见即所得
3. **蓝图扩展**: 支持Blueprint创建自定义Condition
4. **性能优化**: 事件驱动优先，减少轮询开销

---

## 八、下一步实现计划

### Phase 1: 核心框架 (第1周)
- [x] FlowConditionBase基类
- [x] FlowNode_WaitCondition节点
- [x] FlowCondition_Timer示例

### Phase 2: 基础条件 (第2周)
- [ ] FlowCondition_ActorState
- [ ] FlowCondition_GameplayTag
- [ ] FlowCondition_Composite

### Phase 3: 高级条件 (第3周)
- [ ] FlowCondition_LevelSequence
- [ ] FlowCondition_Custom (Blueprint)
- [ ] FlowCondition_Quest

### Phase 4: 编辑器与调试 (第4周)
- [ ] 自定义节点样式
- [ ] 运行时可视化
- [ ] 单元测试

---

准备好开始实现了吗？我可以立即生成：
1. 完整的头文件和实现文件
2. Build.cs配置
3. 测试用例
4. 示例FlowGraph资产

需要我现在开始生成代码吗？
