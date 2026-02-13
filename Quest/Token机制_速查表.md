# Token机制 - 核心改进速查表

## ⚡ 快速对比

### 旧方式 ❌ vs 新方式 ✅

```cpp
// ❌ 旧方式：Condition自己管理Timer
class UFlowCondition_Timer {
    FTimerHandle TimerHandle;  // ❌ 条件持有Timer句柄

    void InitializeCondition() {
        // ❌ 条件自己设置Timer
        GetWorld()->GetTimerManager().SetTimer(
            TimerHandle, this, &UFlowCondition_Timer::OnComplete, Duration
        );
    }

    void CleanupCondition() {
        // ❌ 条件自己清理Timer
        GetWorld()->GetTimerManager().ClearTimer(TimerHandle);
    }

    void SaveConditionState() {
        // ❌ 条件保存Timer状态
        RemainingTime = GetWorld()->GetTimerManager().GetTimerRemaining(TimerHandle);
    }
};
```

```cpp
// ✅ 新方式：Node统一管理Timer
class UFlowCondition_Timer {
    // ✅ 不需要FTimerHandle！

    TArray<FFlowConditionEventRequest> GetRequiredEvents() {
        // ✅ 只告诉节点：我需要Timer
        FFlowConditionEventRequest Request;
        Request.EventType = EFlowConditionEventType::Timer;
        Request.Parameters.Add(TEXT("Duration"), FString::SanitizeFloat(Duration));
        return { Request };
    }

    bool OnConditionEvent(EFlowConditionEventType EventType) {
        // ✅ 节点通知：Timer完成了
        if (EventType == EFlowConditionEventType::Timer) {
            bCompleted = true;
            return true;  // 重新评估
        }
        return false;
    }

    void SaveConditionState() {
        // ✅ 只保存业务状态
        Writer << bCompleted;
    }
};

class UFlowNode_WaitCondition {
    TMap<FName, FTimerHandle> TimerHandles;  // ✅ 节点持有所有Timer

    void RegisterEventListeners() {
        // ✅ 节点统一注册
        TArray<FFlowConditionEventRequest> Events = Condition->GetRequiredEvents();
        for (const auto& Request : Events) {
            RegisterEvent(Request);  // 创建Timer
        }
    }

    void UnregisterEventListeners() {
        // ✅ 节点统一清理
        for (auto& Pair : TimerHandles) {
            GetWorld()->GetTimerManager().ClearTimer(Pair.Value);
        }
        TimerHandles.Empty();
    }
};
```

---

## 🎯 核心原则

### 2077的Token机制三大原则

```
1. Token精准投放
   ✓ Token绑定节点ID + Pin名称
   ✓ 不是全局广播
   ✓ 类似信件投递到具体地址

2. 中心化管理
   ✓ ExecutionContext统一管理事件
   ✓ Node统一持有资源句柄
   ✓ Condition只提供逻辑接口

3. 职责分离
   ✓ Node = Token管理器 + 事件监听器管理器
   ✓ Condition = 纯逻辑 + 状态数据
```

---

## 📊 量化改进

### 代码量减少

| 条件类型 | 旧实现行数 | 新实现行数 | 减少比例 |
|---------|-----------|-----------|---------|
| Timer   | ~150行    | ~80行     | **50%** |
| Actor   | ~200行    | ~100行    | **50%** |
| Complex | ~300行    | ~150行    | **50%** |

### 复杂度降低

```
旧方式需要实现：
✓ InitializeCondition (设置Timer)
✓ CleanupCondition (清理Timer)
✓ OnTimerComplete (Timer回调)
✓ SaveConditionState (保存Timer状态)
✓ LoadConditionState (恢复Timer)
✓ GetWorld() / GetTimerManager()
= 6个方法 + 复杂的资源管理

新方式只需实现：
✓ GetRequiredEvents (告诉节点需要什么)
✓ OnConditionEvent (处理事件)
✓ EvaluateCondition (判断逻辑)
✓ SaveConditionState (保存状态)
= 4个方法 + 简单的状态管理
```

---

## 🔧 关键接口

### 1. Condition接口（新）

```cpp
// ★ 告诉节点需要什么事件
TArray<FFlowConditionEventRequest> GetRequiredEvents(const FFlowConditionContext& Context)
{
    FFlowConditionEventRequest Request;
    Request.EventType = EFlowConditionEventType::Timer;  // 或Actor/Component/等
    Request.Parameters.Add(TEXT("Duration"), TEXT("3.0"));
    Request.ObservedActor = TargetActor;  // 如果需要监听特定Actor
    Request.RelatedTag = MyTag;  // 如果需要监听特定Tag
    return { Request };
}

// ★ 节点通知事件发生
bool OnConditionEvent(EFlowConditionEventType EventType, const FFlowConditionContext& EventContext)
{
    if (EventType == EFlowConditionEventType::Timer)
    {
        bCompleted = true;
        return true;  // true = 需要重新评估条件
    }
    return false;  // false = 不需要重新评估
}

// ★ 纯逻辑判断
EFlowConditionResult EvaluateCondition(const FFlowConditionContext& Context)
{
    return bCompleted
        ? EFlowConditionResult::Fulfilled
        : EFlowConditionResult::NotFulfilled;
}
```

### 2. Node接口（新）

```cpp
// ★ 注册所有事件（类似2077的RegisterEvent）
void RegisterEventListeners()
{
    TArray<FFlowConditionEventRequest> Events = Condition->GetRequiredEvents(Context);
    for (const FFlowConditionEventRequest& Request : Events)
    {
        RegisterEvent(Request);  // 根据类型注册Timer/Delegate等
    }
}

// ★ 注销所有事件（类似2077的UnregisterEvent）
void UnregisterEventListeners()
{
    // 清理所有Timer
    for (auto& Pair : TimerHandles) {
        GetWorld()->GetTimerManager().ClearTimer(Pair.Value);
    }

    // 清理所有Delegate
    for (auto& Pair : ObservedActors) {
        Actor->OnDestroyed.RemoveAll(this);
    }

    // 清理所有资源
}

// ★ 统一事件入口
void OnConditionEventTriggered(EFlowConditionEventType EventType, AActor* Actor, FGameplayTag Tag)
{
    // 通知Condition有事件发生
    bool bShouldReEvaluate = Condition->OnConditionEvent(EventType, EventContext);

    if (bShouldReEvaluate)
    {
        // 重新评估条件
        EFlowConditionResult Result = Condition->EvaluateCondition(Context);
        HandleResult(Result);
    }
}
```

---

## 🚀 使用流程

### 事件流向

```
1. 节点启动
   Node.StartWaiting()
     ↓
   Node.RegisterEventListeners()
     ↓
   查询 Condition.GetRequiredEvents()
     ↓
   返回 [Timer事件请求]
     ↓
   Node.RegisterTimerEvent()
     ↓
   创建Timer并存储到TimerHandles

2. 事件触发
   Timer完成
     ↓
   Node.OnTimerComplete()
     ↓
   Node.OnConditionEventTriggered(Timer)
     ↓
   通知 Condition.OnConditionEvent(Timer)
     ↓
   返回 true（需要重新评估）
     ↓
   Node.EvaluateAndHandleCondition()
     ↓
   调用 Condition.EvaluateCondition()
     ↓
   返回 Fulfilled
     ↓
   Node.HandleConditionFulfilled()

3. 节点结束
   Node.StopWaiting()
     ↓
   Node.UnregisterEventListeners()
     ↓
   清理所有Timer和Delegate
```

---

## 💡 实战示例

### 示例1：等待3秒

```cpp
UCLASS()
class UFlowCondition_Timer : public UFlowConditionBase
{
    UPROPERTY(EditAnywhere)
    float Duration = 3.0f;

    bool bCompleted = false;

    // 告诉节点：我需要3秒Timer
    virtual TArray<FFlowConditionEventRequest> GetRequiredEvents_Implementation(...)
    {
        FFlowConditionEventRequest TimerRequest;
        TimerRequest.EventType = EFlowConditionEventType::Timer;
        TimerRequest.Parameters.Add(TEXT("Duration"), TEXT("3.0"));
        return { TimerRequest };
    }

    // 节点通知：Timer完成
    virtual bool OnConditionEvent_Implementation(EFlowConditionEventType EventType, ...)
    {
        if (EventType == EFlowConditionEventType::Timer)
        {
            bCompleted = true;
            return true;
        }
        return false;
    }

    // 判断是否完成
    virtual EFlowConditionResult EvaluateCondition_Implementation(...)
    {
        return bCompleted ? EFlowConditionResult::Fulfilled : EFlowConditionResult::NotFulfilled;
    }
};
```

### 示例2：等待Actor进入范围

```cpp
UCLASS()
class UFlowCondition_ActorInRange : public UFlowConditionBase
{
    UPROPERTY(EditAnywhere)
    AActor* TargetActor;

    UPROPERTY(EditAnywhere)
    float Range = 100.0f;

    // 告诉节点：我需要监听Actor移动
    virtual TArray<FFlowConditionEventRequest> GetRequiredEvents_Implementation(...)
    {
        FFlowConditionEventRequest MoveRequest;
        MoveRequest.EventType = EFlowConditionEventType::ActorMoved;
        MoveRequest.ObservedActor = TargetActor;
        return { MoveRequest };
    }

    // 节点通知：Actor移动了
    virtual bool OnConditionEvent_Implementation(EFlowConditionEventType EventType, ...)
    {
        return EventType == EFlowConditionEventType::ActorMoved;  // 重新检查距离
    }

    // 判断距离
    virtual EFlowConditionResult EvaluateCondition_Implementation(...)
    {
        if (!TargetActor || !Context.Instigator)
            return EFlowConditionResult::Failed;

        float Distance = FVector::Dist(
            Context.Instigator->GetActorLocation(),
            TargetActor->GetActorLocation()
        );

        return Distance <= Range
            ? EFlowConditionResult::Fulfilled
            : EFlowConditionResult::NotFulfilled;
    }
};
```

### 示例3：等待多个事件

```cpp
UCLASS()
class UFlowCondition_Complex : public UFlowConditionBase
{
    UPROPERTY(EditAnywhere)
    float TimeLimit = 10.0f;

    UPROPERTY(EditAnywhere)
    AActor* TargetActor;

    bool bTimerCompleted = false;
    bool bActorDestroyed = false;

    // 告诉节点：我需要Timer + Actor销毁事件
    virtual TArray<FFlowConditionEventRequest> GetRequiredEvents_Implementation(...)
    {
        TArray<FFlowConditionEventRequest> Events;

        // Timer事件
        FFlowConditionEventRequest TimerRequest;
        TimerRequest.EventType = EFlowConditionEventType::Timer;
        TimerRequest.Parameters.Add(TEXT("Duration"), FString::SanitizeFloat(TimeLimit));
        Events.Add(TimerRequest);

        // Actor销毁事件
        FFlowConditionEventRequest DestroyRequest;
        DestroyRequest.EventType = EFlowConditionEventType::ActorDestroyed;
        DestroyRequest.ObservedActor = TargetActor;
        Events.Add(DestroyRequest);

        return Events;
    }

    // 节点通知：某个事件发生了
    virtual bool OnConditionEvent_Implementation(EFlowConditionEventType EventType, ...)
    {
        if (EventType == EFlowConditionEventType::Timer)
        {
            bTimerCompleted = true;
            return true;
        }
        else if (EventType == EFlowConditionEventType::ActorDestroyed)
        {
            bActorDestroyed = true;
            return true;
        }
        return false;
    }

    // 判断：Timer完成 OR Actor被销毁
    virtual EFlowConditionResult EvaluateCondition_Implementation(...)
    {
        if (bTimerCompleted || bActorDestroyed)
            return EFlowConditionResult::Fulfilled;

        return EFlowConditionResult::NotFulfilled;
    }
};
```

---

## 📋 检查清单

### 实现新Condition时的检查项

- [ ] ✅ **不持有**任何资源句柄（FTimerHandle、FDelegateHandle等）
- [ ] ✅ **不调用**GetWorld()->GetTimerManager()
- [ ] ✅ **实现**GetRequiredEvents()告诉节点需要什么
- [ ] ✅ **实现**OnConditionEvent()处理事件通知
- [ ] ✅ **实现**EvaluateCondition()纯逻辑判断
- [ ] ✅ SaveConditionState()**只保存**业务数据
- [ ] ✅ LoadConditionState()**只恢复**业务数据

### Node统一管理的资源

- [ ] ✅ 所有Timer（TimerHandles）
- [ ] ✅ 所有Delegate（DelegateHandles）
- [ ] ✅ 所有观察的Actor（ObservedActors）
- [ ] ✅ 所有观察的Component（ObservedComponents）
- [ ] ✅ Token数组（ConditionTokens）

---

## 🎓 关键概念

### Token精准投放 vs 全局广播

```cpp
// ❌ 全局广播（旧方式）
EventDispatcher.Broadcast(EMyEvent::TimerComplete);  // 所有监听者都收到

// ✅ Token精准投放（新方式）
FFlowConditionToken Token;
Token.TargetPinName = PIN_Fulfilled;  // 只投放到Fulfilled这个Pin
Token.Type = EFlowConditionTokenType::Signal;
ActivateToken(Token);  // 只有这个节点的Fulfilled Pin会触发
```

### 中心化管理 vs 分散管理

```cpp
// ❌ 分散管理（旧方式）
class ConditionA { FTimerHandle TimerA; }
class ConditionB { FTimerHandle TimerB; }
class ConditionC { FTimerHandle TimerC; }
// 每个Condition自己管理，难以追踪、易遗漏清理

// ✅ 中心化管理（新方式）
class Node {
    TMap<FName, FTimerHandle> TimerHandles;  // 统一持有
    void UnregisterEventListeners() {
        // 统一清理，不会遗漏
    }
}
```

---

## ⚠️ 常见错误

### 错误1：Condition持有资源

```cpp
// ❌ 错误
class UFlowCondition_Wrong {
    FTimerHandle MyTimer;  // ❌ Condition不应该持有Timer
};

// ✅ 正确
class UFlowCondition_Correct {
    // 不持有任何资源句柄
    bool bCompleted;  // 只有状态数据
};
```

### 错误2：Condition自己注册事件

```cpp
// ❌ 错误
void UFlowCondition_Wrong::InitializeCondition(...) {
    GetWorld()->GetTimerManager().SetTimer(...);  // ❌ 不应该自己注册
}

// ✅ 正确
TArray<FFlowConditionEventRequest> UFlowCondition_Correct::GetRequiredEvents(...) {
    FFlowConditionEventRequest Request;
    Request.EventType = EFlowConditionEventType::Timer;
    return { Request };  // ✅ 告诉节点需要什么
}
```

### 错误3：保存Timer状态

```cpp
// ❌ 错误
void UFlowCondition_Wrong::SaveConditionState(...) {
    RemainingTime = GetWorld()->GetTimerManager().GetTimerRemaining(MyTimer);  // ❌
}

// ✅ 正确
void UFlowCondition_Correct::SaveConditionState(...) {
    Writer << bCompleted;  // ✅ 只保存业务状态
}
```

---

## 🎯 记忆口诀

```
节点管资源，条件管逻辑
Token精准投，事件中心化
告诉我需要，通知我发生
评估纯判断，保存只状态
```

---

生成时间：2026-02-13
参考：Cyberpunk 2077 Token Mechanism
