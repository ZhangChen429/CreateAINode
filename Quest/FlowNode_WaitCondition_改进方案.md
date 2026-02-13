# FlowNode Wait Condition - 改进方案（事件注册管理）

## 问题诊断

### ❌ 当前实现的问题

```cpp
// 当前错误的做法：Condition自己管理事件
void UFlowCondition_Timer::InitializeCondition()
{
    // ❌ 在Condition内部直接设置Timer
    GetWorld()->GetTimerManager().SetTimer(TimerHandle, ...);
}
```

**核心缺陷：**
1. ❌ 条件自己管理生命周期，节点无法统一控制
2. ❌ SaveGame时需要每个条件单独处理
3. ❌ Cleanup时可能遗漏清理
4. ❌ 无法追踪哪些事件被注册了

---

## ✅ 正确的设计模式

### Cyberpunk 2077的模式

```cpp
// 2077: 通过ExecutionContext中心化管理事件
class SignalStoppingNodeDefinition {
    void RegisterEvent(NodeExecutionContext& executionContext) const {
        // 统一注册到ExecutionContext
        executionContext.RegisterEvent(
            GetExtendedId(),
            GetEventName(),
            CreateEventListener(executionContext)  // 创建监听器
        );
    }

    void UnregisterEvent(NodeExecutionContext& executionContext) const {
        // 统一注销
        executionContext.UnregisterEvent(GetExtendedId());
    }

    Bool IsEventRegistered(const NodeExecutionContext& executionContext) const {
        return FindEventListener(executionContext) != nullptr;
    }
};
```

### FlowGraph的模式

```cpp
// FlowGraph: 通过节点统一管理Delegate订阅
class UFlowNode_ComponentObserver {
    void StartObserving() {
        // 订阅Subsystem的事件
        FlowSubsystem->OnComponentRegistered.AddUniqueDynamic(...);
        FlowSubsystem->OnComponentTagAdded.AddUniqueDynamic(...);

        // 对每个Actor订阅Component事件
        Component->OnNotifyFromComponent.AddUObject(...);

        // 记录已订阅的Actors
        RegisteredActors.Add(Actor, Component);
    }

    void StopObserving() {
        // 统一取消订阅
        FlowSubsystem->OnComponentRegistered.RemoveAll(this);

        // 清理每个Actor的订阅
        for (auto& Pair : RegisteredActors) {
            Component->OnNotifyFromComponent.RemoveAll(this);
        }
        RegisteredActors.Empty();
    }
};
```

---

## 🎯 改进后的架构

### 核心原则

1. **节点统一管理** - 所有事件注册/注销由WaitCondition节点控制
2. **条件只提供接口** - Condition只告诉节点"需要监听什么"
3. **节点持有句柄** - Timer、Delegate句柄都保存在节点层
4. **统一生命周期** - 通过StartWaiting/StopWaiting统一管理

### 新的类职责划分

```
┌─────────────────────────────────────────────────┐
│          UFlowNode_WaitCondition                │
│  (负责：事件注册、生命周期、SaveGame)            │
├─────────────────────────────────────────────────┤
│  - FTimerHandle TimeoutTimer                    │
│  - TArray<FDelegateHandle> EventHandles  ★NEW  │
│  - TArray<TWeakObjectPtr> ObservedActors ★NEW  │
│                                                  │
│  + StartWaiting()  ★核心                        │
│    └─ RegisterEventListeners()  ★NEW           │
│  + StopWaiting()   ★核心                        │
│    └─ UnregisterEventListeners() ★NEW          │
│  + OnConditionEvent()  ★NEW                     │
└─────────────────────────────────────────────────┘
                    ▼ 使用
┌─────────────────────────────────────────────────┐
│          UFlowConditionBase                      │
│  (只负责：提供监听接口、评估逻辑)                │
├─────────────────────────────────────────────────┤
│  + GetRequiredEventTypes() ★NEW                 │
│    返回：需要监听的事件类型                       │
│  + GetObservedActors() ★NEW                     │
│    返回：需要监听的Actors                         │
│  + EvaluateCondition()                          │
│    纯逻辑：检查条件是否满足                       │
└─────────────────────────────────────────────────┘
```

---

## 📝 新的代码实现

### 1. 条件事件类型枚举

```cpp
// FlowConditionTypes.h

/**
 * 条件需要监听的事件类型
 */
UENUM(BlueprintType)
enum class EFlowConditionEventType : uint8
{
    None                UMETA(DisplayName = "None"),

    // Timer相关
    Timer               UMETA(DisplayName = "Timer"),

    // Actor相关
    ActorMoved          UMETA(DisplayName = "Actor Moved"),
    ActorDestroyed      UMETA(DisplayName = "Actor Destroyed"),
    ActorTagChanged     UMETA(DisplayName = "Actor Tag Changed"),

    // Component相关
    ComponentNotify     UMETA(DisplayName = "Component Notify"),

    // Gameplay相关
    GameplayTagAdded    UMETA(DisplayName = "Gameplay Tag Added"),
    GameplayTagRemoved  UMETA(DisplayName = "Gameplay Tag Removed"),

    // 序列相关
    SequenceFinished    UMETA(DisplayName = "Sequence Finished"),
    SequenceMarker      UMETA(DisplayName = "Sequence Marker"),

    // 自定义
    Custom              UMETA(DisplayName = "Custom")
};

/**
 * 条件事件注册请求
 */
USTRUCT(BlueprintType)
struct FFlowConditionEventRequest
{
    GENERATED_BODY()

    /** 事件类型 */
    UPROPERTY()
    EFlowConditionEventType EventType = EFlowConditionEventType::None;

    /** 需要观察的Actor（可选） */
    UPROPERTY()
    TWeakObjectPtr<AActor> ObservedActor;

    /** 关联的GameplayTag（可选） */
    UPROPERTY()
    FGameplayTag RelatedTag;

    /** 自定义参数 */
    UPROPERTY()
    TMap<FName, FString> Parameters;
};
```

### 2. 改进后的FlowConditionBase

```cpp
// FlowConditionBase.h

UCLASS(Abstract, Blueprintable, EditInlineNew, CollapseCategories)
class FLOWNODE_API UFlowConditionBase : public UObject
{
    GENERATED_BODY()

public:
    // ========================================================================
    // ★ 新增：事件注册接口
    // ========================================================================

    /**
     * 获取需要监听的事件类型
     * 节点会根据返回值注册相应的事件监听器
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Flow|Condition")
    TArray<FFlowConditionEventRequest> GetRequiredEvents(const FFlowConditionContext& Context);
    virtual TArray<FFlowConditionEventRequest> GetRequiredEvents_Implementation(const FFlowConditionContext& Context);

    /**
     * 事件触发回调
     * 当注册的事件发生时，节点会调用此方法
     * @param EventType 事件类型
     * @param EventData 事件数据（Actor、Tag等）
     * @return 是否需要重新评估条件
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Flow|Condition")
    bool OnConditionEvent(EFlowConditionEventType EventType, const FFlowConditionContext& EventContext);
    virtual bool OnConditionEvent_Implementation(EFlowConditionEventType EventType, const FFlowConditionContext& EventContext);

    // ========================================================================
    // 简化后的生命周期（不再需要自己管理Timer）
    // ========================================================================

    /**
     * 初始化条件（简化版）
     * 只需要设置内部状态，不需要注册事件
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Flow|Condition")
    void InitializeCondition(const FFlowConditionContext& Context, UFlowNode_WaitCondition* OwnerNode);
    virtual void InitializeCondition_Implementation(const FFlowConditionContext& Context, UFlowNode_WaitCondition* OwnerNode);

    /**
     * 清理条件（简化版）
     * 不需要取消订阅，节点会统一处理
     */
    UFUNCTION(BlueprintNativeEvent, Category = "Flow|Condition")
    void CleanupCondition();
    virtual void CleanupCondition_Implementation();

    // ========================================================================
    // 条件评估（不变）
    // ========================================================================

    UFUNCTION(BlueprintNativeEvent, Category = "Flow|Condition")
    EFlowConditionResult EvaluateCondition(const FFlowConditionContext& Context);
    virtual EFlowConditionResult EvaluateCondition_Implementation(const FFlowConditionContext& Context);

    UFUNCTION(BlueprintNativeEvent, Category = "Flow|Condition")
    float GetProgress() const;
    virtual float GetProgress_Implementation() const { return 0.0f; }

protected:
    // ❌ 移除：不再需要自己管理Timer
    // FTimerHandle PollingTimerHandle;

    // ❌ 移除：不再需要NotifyConditionChanged
    // void NotifyConditionChanged(EFlowConditionResult NewResult);
};
```

### 3. 改进后的FlowNode_WaitCondition

```cpp
// FlowNode_WaitCondition.h

UCLASS(NotBlueprintable, meta = (DisplayName = "Wait Until Condition"))
class FLOWNODE_API UFlowNode_WaitCondition : public UFlowNode
{
    GENERATED_BODY()

public:
    // ========================================================================
    // ★ 新增：事件管理
    // ========================================================================

    /**
     * 条件事件回调（由各种Delegate调用）
     */
    UFUNCTION()
    void OnConditionEventTriggered(EFlowConditionEventType EventType, AActor* RelatedActor = nullptr, const FGameplayTag& RelatedTag = FGameplayTag());

protected:
    // ========================================================================
    // ★ 核心：事件注册管理
    // ========================================================================

    /** 注册所有需要的事件监听器 */
    void RegisterEventListeners();

    /** 注销所有事件监听器 */
    void UnregisterEventListeners();

    /** 注册单个事件 */
    void RegisterEvent(const FFlowConditionEventRequest& Request);

    /** 注销单个事件 */
    void UnregisterEvent(EFlowConditionEventType EventType);

    // ========================================================================
    // ★ 事件句柄存储（统一管理）
    // ========================================================================

    /** Timer句柄（超时、Timer条件等） */
    UPROPERTY(Transient)
    TMap<FName, FTimerHandle> TimerHandles;

    /** Delegate句柄（用于取消订阅） */
    UPROPERTY(Transient)
    TArray<FDelegateHandle> DelegateHandles;

    /** 观察的Actors（用于清理订阅） */
    UPROPERTY(Transient)
    TMap<TWeakObjectPtr<AActor>, TWeakObjectPtr<UActorComponent>> ObservedActors;

    /** 观察的Components */
    UPROPERTY(Transient)
    TArray<TWeakObjectPtr<UFlowComponent>> ObservedComponents;

    // ========================================================================
    // 辅助方法
    // ========================================================================

    /** 注册Timer事件 */
    void RegisterTimerEvent(const FFlowConditionEventRequest& Request);

    /** 注册Actor事件 */
    void RegisterActorEvent(const FFlowConditionEventRequest& Request);

    /** 注册Component事件 */
    void RegisterComponentEvent(const FFlowConditionEventRequest& Request);

    /** 注册Sequence事件 */
    void RegisterSequenceEvent(const FFlowConditionEventRequest& Request);

    /** Timer完成回调 */
    UFUNCTION()
    void OnTimerComplete(FName TimerName);

    /** Actor移动回调 */
    UFUNCTION()
    void OnActorMoved(AActor* Actor);

    /** Actor销毁回调 */
    UFUNCTION()
    void OnActorDestroyed(AActor* Actor);

    /** Component通知回调 */
    UFUNCTION()
    void OnComponentNotify(UFlowComponent* Component, const FGameplayTag& Tag);

    /** Sequence完成回调 */
    UFUNCTION()
    void OnSequenceFinished();
};
```

### 4. 改进后的FlowNode_WaitCondition实现

```cpp
// FlowNode_WaitCondition.cpp

void UFlowNode_WaitCondition::StartWaiting()
{
    if (bIsWaiting)
    {
        return;
    }

    bIsWaiting = true;
    WaitStartTime = GetWorld()->GetTimeSeconds();

    // 构建上下文
    FFlowConditionContext Context;
    Context.World = GetWorld();
    Context.Instigator = TryGetRootFlowActorOwner();

    // 初始化条件
    Condition->InitializeCondition(Context, this);

    // ★ 立即评估一次
    EFlowConditionResult InitialResult = Condition->EvaluateCondition(Context);
    if (InitialResult == EFlowConditionResult::Fulfilled)
    {
        HandleConditionFulfilled();
        return;
    }
    else if (InitialResult == EFlowConditionResult::Failed)
    {
        HandleConditionFailed();
        return;
    }

    // ★ 关键：注册事件监听器
    RegisterEventListeners();

    // 设置超时（如果需要）
    if (TimeoutDuration > 0.0f)
    {
        FTimerHandle TimeoutHandle;
        GetWorld()->GetTimerManager().SetTimer(
            TimeoutHandle,
            this,
            &UFlowNode_WaitCondition::OnTimeout,
            TimeoutDuration,
            false
        );
        TimerHandles.Add(TEXT("Timeout"), TimeoutHandle);
    }
}

void UFlowNode_WaitCondition::StopWaiting()
{
    if (!bIsWaiting)
    {
        return;
    }

    bIsWaiting = false;

    // ★ 关键：注销所有事件监听器
    UnregisterEventListeners();

    // 清理Timer
    for (auto& Pair : TimerHandles)
    {
        GetWorld()->GetTimerManager().ClearTimer(Pair.Value);
    }
    TimerHandles.Empty();

    // 清理条件
    if (Condition)
    {
        Condition->CleanupCondition();
    }
}

void UFlowNode_WaitCondition::RegisterEventListeners()
{
    if (!Condition)
    {
        return;
    }

    // 获取需要监听的事件
    FFlowConditionContext Context;
    Context.World = GetWorld();
    Context.Instigator = TryGetRootFlowActorOwner();

    TArray<FFlowConditionEventRequest> Events = Condition->GetRequiredEvents(Context);

    // 注册每个事件
    for (const FFlowConditionEventRequest& Request : Events)
    {
        RegisterEvent(Request);
    }
}

void UFlowNode_WaitCondition::UnregisterEventListeners()
{
    // 取消所有Delegate订阅
    for (const FDelegateHandle& Handle : DelegateHandles)
    {
        // 根据Handle类型取消订阅
        // 注意：实际实现需要记录Delegate的来源
    }
    DelegateHandles.Empty();

    // 清理观察的Actors
    for (auto& Pair : ObservedActors)
    {
        if (AActor* Actor = Pair.Key.Get())
        {
            // 取消Actor相关订阅
            // Actor->OnDestroyed.Remove(...)
        }
    }
    ObservedActors.Empty();

    // 清理观察的Components
    for (TWeakObjectPtr<UFlowComponent> Comp : ObservedComponents)
    {
        if (UFlowComponent* Component = Comp.Get())
        {
            Component->OnNotifyFromComponent.RemoveAll(this);
        }
    }
    ObservedComponents.Empty();
}

void UFlowNode_WaitCondition::RegisterEvent(const FFlowConditionEventRequest& Request)
{
    switch (Request.EventType)
    {
    case EFlowConditionEventType::Timer:
        RegisterTimerEvent(Request);
        break;

    case EFlowConditionEventType::ActorMoved:
    case EFlowConditionEventType::ActorDestroyed:
        RegisterActorEvent(Request);
        break;

    case EFlowConditionEventType::ComponentNotify:
        RegisterComponentEvent(Request);
        break;

    case EFlowConditionEventType::SequenceFinished:
        RegisterSequenceEvent(Request);
        break;

    default:
        break;
    }
}

void UFlowNode_WaitCondition::RegisterTimerEvent(const FFlowConditionEventRequest& Request)
{
    // 从参数中获取时长
    float Duration = 1.0f;
    if (Request.Parameters.Contains(TEXT("Duration")))
    {
        Duration = FCString::Atof(*Request.Parameters[TEXT("Duration")]);
    }

    FName TimerName = Request.Parameters.Contains(TEXT("Name"))
        ? FName(*Request.Parameters[TEXT("Name")])
        : TEXT("ConditionTimer");

    FTimerHandle TimerHandle;
    FTimerDelegate TimerDelegate;
    TimerDelegate.BindUFunction(this, FName("OnTimerComplete"), TimerName);

    GetWorld()->GetTimerManager().SetTimer(
        TimerHandle,
        TimerDelegate,
        Duration,
        false
    );

    TimerHandles.Add(TimerName, TimerHandle);
}

void UFlowNode_WaitCondition::RegisterActorEvent(const FFlowConditionEventRequest& Request)
{
    if (AActor* Actor = Request.ObservedActor.Get())
    {
        if (Request.EventType == EFlowConditionEventType::ActorDestroyed)
        {
            // 订阅Actor销毁事件
            Actor->OnDestroyed.AddDynamic(this, &UFlowNode_WaitCondition::OnActorDestroyed);
        }

        ObservedActors.Add(Actor, nullptr);
    }
}

void UFlowNode_WaitCondition::RegisterComponentEvent(const FFlowConditionEventRequest& Request)
{
    // 从FlowSubsystem查找匹配的Components
    if (UFlowSubsystem* FlowSubsystem = GetFlowSubsystem())
    {
        FGameplayTagContainer Tags;
        Tags.AddTag(Request.RelatedTag);

        for (TWeakObjectPtr<UFlowComponent> Comp : FlowSubsystem->GetComponents<UFlowComponent>(Tags))
        {
            if (UFlowComponent* Component = Comp.Get())
            {
                Component->OnNotifyFromComponent.AddDynamic(this, &UFlowNode_WaitCondition::OnComponentNotify);
                ObservedComponents.Add(Component);
            }
        }
    }
}

void UFlowNode_WaitCondition::OnConditionEventTriggered(
    EFlowConditionEventType EventType,
    AActor* RelatedActor,
    const FGameplayTag& RelatedTag)
{
    if (!bIsWaiting || !Condition)
    {
        return;
    }

    // 构建事件上下文
    FFlowConditionContext EventContext;
    EventContext.World = GetWorld();
    EventContext.Instigator = RelatedActor;

    // ★ 通知条件有事件发生
    bool bShouldReEvaluate = Condition->OnConditionEvent(EventType, EventContext);

    if (bShouldReEvaluate)
    {
        // 重新评估条件
        EFlowConditionResult Result = Condition->EvaluateCondition(EventContext);

        if (Result == EFlowConditionResult::Fulfilled)
        {
            HandleConditionFulfilled();
        }
        else if (Result == EFlowConditionResult::Failed)
        {
            HandleConditionFailed();
        }
    }
}

void UFlowNode_WaitCondition::OnTimerComplete(FName TimerName)
{
    OnConditionEventTriggered(EFlowConditionEventType::Timer);
}

void UFlowNode_WaitCondition::OnActorDestroyed(AActor* Actor)
{
    OnConditionEventTriggered(EFlowConditionEventType::ActorDestroyed, Actor);
}

void UFlowNode_WaitCondition::OnComponentNotify(UFlowComponent* Component, const FGameplayTag& Tag)
{
    OnConditionEventTriggered(EFlowConditionEventType::ComponentNotify, Component->GetOwner(), Tag);
}
```

### 5. 改进后的Timer条件

```cpp
// FlowCondition_Timer.cpp（大幅简化）

TArray<FFlowConditionEventRequest> UFlowCondition_Timer::GetRequiredEvents_Implementation(const FFlowConditionContext& Context)
{
    TArray<FFlowConditionEventRequest> Events;

    // ★ 只需要告诉节点"我需要一个Timer"
    FFlowConditionEventRequest TimerRequest;
    TimerRequest.EventType = EFlowConditionEventType::Timer;
    TimerRequest.Parameters.Add(TEXT("Duration"), FString::SanitizeFloat(Duration));
    TimerRequest.Parameters.Add(TEXT("Name"), TEXT("ConditionTimer"));

    Events.Add(TimerRequest);

    return Events;
}

bool UFlowCondition_Timer::OnConditionEvent_Implementation(
    EFlowConditionEventType EventType,
    const FFlowConditionContext& EventContext)
{
    if (EventType == EFlowConditionEventType::Timer)
    {
        bCompleted = true;
        return true;  // 需要重新评估条件
    }

    return false;
}

EFlowConditionResult UFlowCondition_Timer::EvaluateCondition_Implementation(const FFlowConditionContext& Context)
{
    return bCompleted ? EFlowConditionResult::Fulfilled : EFlowConditionResult::NotFulfilled;
}

float UFlowCondition_Timer::GetProgress_Implementation() const
{
    // 进度由节点追踪Timer剩余时间计算
    // 或者在Context中传递
    return bCompleted ? 1.0f : 0.5f;
}

// ✅ 不再需要自己管理Timer！
// ✅ 不再需要InitializeCondition设置Timer！
// ✅ 不再需要CleanupCondition清理Timer！
```

---

## 🎯 改进总结

### 关键改进点

1. **✅ 中心化事件管理**
   - 所有事件注册/注销由节点统一管理
   - 节点持有所有Timer和Delegate句柄

2. **✅ 条件只负责逻辑**
   - `GetRequiredEvents()` - 告诉节点需要监听什么
   - `OnConditionEvent()` - 处理事件并决定是否重新评估
   - `EvaluateCondition()` - 纯逻辑判断

3. **✅ 生命周期清晰**
   - `StartWaiting()` → `RegisterEventListeners()`
   - `StopWaiting()` → `UnregisterEventListeners()`
   - `Cleanup()` → 统一清理所有资源

4. **✅ SaveGame简化**
   - 节点统一保存Timer状态
   - 条件只保存业务数据

### 对比

| 维度 | ❌ 旧实现 | ✅ 新实现 |
|------|----------|----------|
| 事件管理 | 条件自己管理 | 节点统一管理 |
| Timer持有 | Condition | Node |
| Delegate持有 | Condition | Node |
| 生命周期 | 分散 | 集中 |
| SaveGame | 复杂 | 简单 |
| 清理 | 易遗漏 | 可靠 |

---

## 📝 下一步

1. 按新架构重写核心类
2. 实现ActorState条件（作为复杂示例）
3. 实现LevelSequence条件
4. 完善SaveGame逻辑
5. 添加调试工具

这个架构完全符合FlowGraph的设计理念，同时吸收了2077的精华！准备好重写代码了吗？
