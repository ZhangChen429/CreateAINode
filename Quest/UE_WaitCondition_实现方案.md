# 虚幻引擎 - Wait Until Condition 系统设计方案

## 一、系统概述

将Cyberpunk 2077的PauseNode+Condition机制移植到虚幻引擎，实现灵活的条件等待系统。

### 1.1 核心特性

- ✅ **异步条件等待** - 不阻塞游戏主线程
- ✅ **事件驱动** - 条件满足时自动触发，无需轮询
- ✅ **蓝图友好** - 完整的Blueprint支持
- ✅ **可扩展** - 易于添加新的条件类型
- ✅ **调试友好** - 可视化调试工具
- ✅ **序列化支持** - 支持存档/读档

### 1.2 架构对比

| 2077系统 | 虚幻引擎系统 |
|----------|-------------|
| PauseConditionNode | UWaitConditionNode |
| IBaseCondition | UWaitCondition (基类) |
| EventListener | FConditionDelegate |
| NodeExecutionContext | UObject* Context |
| Quest System | Quest/Sequence System |

---

## 二、核心类设计

### 2.1 类继承体系

```
UObject
    ↓
UWaitCondition (抽象基类)
    ├─ UWaitCondition_Timer (等待时间)
    ├─ UWaitCondition_ActorState (等待Actor状态)
    ├─ UWaitCondition_LevelSequence (等待关卡序列)
    ├─ UWaitCondition_Gameplay (等待Gameplay事件)
    ├─ UWaitCondition_Quest (等待任务阶段)
    ├─ UWaitCondition_Animation (等待动画)
    ├─ UWaitCondition_Custom (自定义条件)
    └─ UWaitCondition_Composite (组合条件 AND/OR)
```

### 2.2 文件结构

```
Source/YourProject/
└── WaitCondition/
    ├── Public/
    │   ├── WaitConditionTypes.h          // 枚举和结构体
    │   ├── WaitCondition.h                // 基类
    │   ├── WaitConditionNode.h            // 节点类
    │   ├── WaitConditionSubsystem.h       // 子系统
    │   └── Conditions/
    │       ├── WaitCondition_Timer.h
    │       ├── WaitCondition_ActorState.h
    │       ├── WaitCondition_LevelSequence.h
    │       ├── WaitCondition_Gameplay.h
    │       ├── WaitCondition_Quest.h
    │       ├── WaitCondition_Animation.h
    │       ├── WaitCondition_Custom.h
    │       └── WaitCondition_Composite.h
    └── Private/
        ├── WaitCondition.cpp
        ├── WaitConditionNode.cpp
        ├── WaitConditionSubsystem.cpp
        └── Conditions/
            ├── WaitCondition_Timer.cpp
            ├── WaitCondition_ActorState.cpp
            ├── WaitCondition_LevelSequence.cpp
            ├── WaitCondition_Gameplay.cpp
            ├── WaitCondition_Quest.cpp
            ├── WaitCondition_Animation.cpp
            ├── WaitCondition_Custom.cpp
            └── WaitCondition_Composite.cpp
```

---

## 三、实现细节

### 3.1 条件状态枚举

```cpp
// 条件评估结果
UENUM(BlueprintType)
enum class EConditionResult : uint8
{
    NotFulfilled,      // 条件未满足
    Fulfilled,         // 条件已满足
    Failed,            // 条件失败（永远无法满足）
    Cancelled          // 条件被取消
};

// 条件检查频率
UENUM(BlueprintType)
enum class EConditionCheckMode : uint8
{
    EventDriven,       // 事件驱动（推荐）
    PollingEveryFrame, // 每帧轮询
    PollingInterval    // 间隔轮询
};

// 组合条件类型
UENUM(BlueprintType)
enum class ECompositeConditionType : uint8
{
    And,               // 所有条件都满足
    Or,                // 任一条件满足
    Not,               // 条件不满足
    XOr                // 恰好一个条件满足
};
```

### 3.2 委托定义

```cpp
// 条件满足时的回调
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnConditionFulfilled, UWaitCondition*, Condition);

// 条件状态变化回调
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(
    FOnConditionStateChanged,
    UWaitCondition*, Condition,
    EConditionResult, NewResult
);
```

### 3.3 核心接口设计

```cpp
/**
 * 条件评估上下文
 */
USTRUCT(BlueprintType)
struct FConditionContext
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    UObject* ContextObject = nullptr;

    UPROPERTY(BlueprintReadWrite)
    UWorld* World = nullptr;

    UPROPERTY(BlueprintReadWrite)
    AActor* Instigator = nullptr;

    UPROPERTY(BlueprintReadWrite)
    TMap<FName, FString> Parameters;
};
```

---

## 四、使用场景示例

### 4.1 任务系统集成

```cpp
// 等待玩家到达指定位置
UWaitCondition_ActorState* WaitPlayerAtLocation = NewObject<UWaitCondition_ActorState>();
WaitPlayerAtLocation->TargetActor = PlayerCharacter;
WaitPlayerAtLocation->LocationTarget = FVector(1000, 2000, 100);
WaitPlayerAtLocation->AcceptanceRadius = 200.0f;

// 等待敌人全部死亡
UWaitCondition_Gameplay* WaitEnemiesDead = NewObject<UWaitCondition_Gameplay>();
WaitEnemiesDead->EventTag = FGameplayTag::RequestGameplayTag("Quest.Event.AllEnemiesDead");

// 组合条件：玩家到达位置 AND 敌人全部死亡
UWaitCondition_Composite* CompleteObjective = NewObject<UWaitCondition_Composite>();
CompleteObjective->CompositeType = ECompositeConditionType::And;
CompleteObjective->ChildConditions.Add(WaitPlayerAtLocation);
CompleteObjective->ChildConditions.Add(WaitEnemiesDead);
```

### 4.2 对话系统集成

```cpp
// 等待角色说完话
UWaitCondition_Animation* WaitDialogueFinish = NewObject<UWaitCondition_Animation>();
WaitDialogueFinish->TargetActor = NPCActor;
WaitDialogueFinish->AnimationAsset = DialogueAnim;
WaitDialogueFinish->WaitForCompletion = true;
```

### 4.3 关卡序列集成

```cpp
// 等待Sequencer播放完成
UWaitCondition_LevelSequence* WaitCutscene = NewObject<UWaitCondition_LevelSequence>();
WaitCutscene->LevelSequence = CutsceneSequence;
WaitCutscene->WaitForCompletion = true;
```

---

## 五、蓝图支持

### 5.1 蓝图节点设计

```
┌────────────────────────────────────────┐
│     Wait Until Condition               │
├────────────────────────────────────────┤
│ ►Exec In                               │
│                                         │
│ [Condition] ─────────────────────      │
│                                         │
│ [Timeout (Optional)] ─────────         │
│                                         │
│                          Fulfilled ► ──┤
│                          Timeout ► ────┤
│                          Cancelled ► ──┤
│                          Failed ► ─────┤
└────────────────────────────────────────┘
```

### 5.2 异步蓝图节点

```cpp
/**
 * 异步蓝图节点 - 等待条件满足
 */
UCLASS()
class UAsyncAction_WaitCondition : public UBlueprintAsyncActionBase
{
    GENERATED_BODY()

public:
    UPROPERTY(BlueprintAssignable)
    FOnConditionFulfilled OnFulfilled;

    UPROPERTY(BlueprintAssignable)
    FOnConditionFulfilled OnTimeout;

    UPROPERTY(BlueprintAssignable)
    FOnConditionFulfilled OnCancelled;

    UPROPERTY(BlueprintAssignable)
    FOnConditionFulfilled OnFailed;

    UFUNCTION(BlueprintCallable, Category = "Condition",
              meta = (BlueprintInternalUseOnly = "true", WorldContext = "WorldContextObject"))
    static UAsyncAction_WaitCondition* WaitForCondition(
        UObject* WorldContextObject,
        UWaitCondition* Condition,
        float Timeout = -1.0f
    );

    virtual void Activate() override;
    virtual void Cancel() override;
};
```

---

## 六、性能优化策略

### 6.1 事件驱动优化

```cpp
// 使用委托避免每帧检查
void UWaitCondition_ActorState::Initialize(const FConditionContext& Context)
{
    if (TargetActor)
    {
        // 绑定Actor移动事件
        TargetActor->OnActorMovedDelegate.AddDynamic(
            this, &UWaitCondition_ActorState::OnActorMoved
        );
    }
}

void UWaitCondition_ActorState::OnActorMoved(AActor* Actor)
{
    // 只在Actor移动时检查条件
    CheckCondition();
}
```

### 6.2 条件缓存

```cpp
class UWaitCondition
{
private:
    // 缓存上次评估结果
    EConditionResult CachedResult = EConditionResult::NotFulfilled;

    // 脏标记
    bool bIsDirty = true;

public:
    void MarkDirty()
    {
        bIsDirty = true;
    }

    EConditionResult Evaluate(const FConditionContext& Context)
    {
        if (!bIsDirty)
        {
            return CachedResult;
        }

        CachedResult = EvaluateInternal(Context);
        bIsDirty = false;
        return CachedResult;
    }
};
```

### 6.3 条件池化

```cpp
// 使用对象池避免频繁创建销毁
class UWaitConditionSubsystem : public UGameInstanceSubsystem
{
private:
    TMap<TSubclassOf<UWaitCondition>, TArray<UWaitCondition*>> ConditionPools;

public:
    UWaitCondition* AcquireCondition(TSubclassOf<UWaitCondition> ConditionClass);
    void ReleaseCondition(UWaitCondition* Condition);
};
```

---

## 七、调试工具

### 7.1 编辑器可视化

```cpp
#if WITH_EDITOR
class FWaitConditionVisualizer : public IComponentVisualizer
{
public:
    virtual void DrawVisualization(
        const UActorComponent* Component,
        const FSceneView* View,
        FPrimitiveDrawInterface* PDI
    ) override;
};
#endif
```

### 7.2 调试命令

```cpp
// 控制台命令
UFUNCTION(Exec)
void DebugWaitConditions()
{
    // 列出所有活跃的条件
    for (UWaitCondition* Condition : ActiveConditions)
    {
        UE_LOG(LogTemp, Warning, TEXT("Condition: %s, State: %s"),
               *Condition->GetName(),
               *UEnum::GetValueAsString(Condition->GetCurrentResult()));
    }
}
```

### 7.3 可视化调试窗口

```
┌─────────────────────────────────────────────┐
│  Wait Condition Debugger                    │
├─────────────────────────────────────────────┤
│ Active Conditions: 3                        │
│                                             │
│ ┌─ WaitPlayerAtLocation                    │
│ │  Status: ● Not Fulfilled                 │
│ │  Progress: 45%                           │
│ │  Distance: 550/200                       │
│ └─                                          │
│                                             │
│ ┌─ WaitEnemiesDead                         │
│ │  Status: ● Fulfilled                     │
│ │  Enemies Remaining: 0/5                  │
│ └─                                          │
│                                             │
│ ┌─ WaitDialogueFinish                      │
│ │  Status: ⏸ Paused                        │
│ │  Time: 2.5s / 5.0s                       │
│ └─                                          │
└─────────────────────────────────────────────┘
```

---

## 八、与虚幻引擎原生系统集成

### 8.1 Gameplay Ability System集成

```cpp
UCLASS()
class UWaitCondition_GameplayAbility : public UWaitCondition
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FGameplayTag AbilityTag;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bWaitForActivation = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bWaitForCompletion = false;

protected:
    virtual EConditionResult EvaluateInternal(const FConditionContext& Context) override;
};
```

### 8.2 Behavior Tree集成

```cpp
UCLASS()
class UBTTask_WaitCondition : public UBTTaskNode
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Instanced)
    UWaitCondition* Condition;

    virtual EBTNodeResult::Type ExecuteTask(
        UBehaviorTreeComponent& OwnerComp,
        uint8* NodeMemory
    ) override;

protected:
    UFUNCTION()
    void OnConditionFulfilled(UWaitCondition* FulfilledCondition);
};
```

### 8.3 Smart Object集成

```cpp
UCLASS()
class UWaitCondition_SmartObject : public UWaitCondition
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FSmartObjectHandle SmartObjectHandle;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    ESmartObjectSlotState TargetState;

protected:
    virtual EConditionResult EvaluateInternal(const FConditionContext& Context) override;
};
```

---

## 九、序列化与网络支持

### 9.1 存档支持

```cpp
UCLASS()
class UWaitCondition : public UObject
{
    GENERATED_BODY()

public:
    // 序列化保存
    virtual void Serialize(FArchive& Ar) override;

    // 存档数据
    UFUNCTION(BlueprintCallable, Category = "Save")
    FConditionSaveData SaveCondition() const;

    // 加载数据
    UFUNCTION(BlueprintCallable, Category = "Save")
    void LoadCondition(const FConditionSaveData& SaveData);
};

USTRUCT(BlueprintType)
struct FConditionSaveData
{
    GENERATED_BODY()

    UPROPERTY(SaveGame)
    TSubclassOf<UWaitCondition> ConditionClass;

    UPROPERTY(SaveGame)
    TArray<uint8> ConditionData;

    UPROPERTY(SaveGame)
    EConditionResult CurrentResult;

    UPROPERTY(SaveGame)
    float ElapsedTime;
};
```

### 9.2 网络复制

```cpp
UCLASS(Replicated)
class UWaitCondition_Replicated : public UWaitCondition
{
    GENERATED_BODY()

public:
    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps
    ) const override;

    UPROPERTY(ReplicatedUsing = OnRep_ConditionResult)
    EConditionResult ReplicatedResult;

    UFUNCTION()
    void OnRep_ConditionResult();

    // 仅在服务器端评估
    virtual EConditionResult EvaluateInternal(
        const FConditionContext& Context
    ) override;
};
```

---

## 十、实现优先级建议

### Phase 1: 核心系统 (1-2周)
- ✅ WaitCondition基类
- ✅ WaitConditionNode
- ✅ WaitConditionSubsystem
- ✅ 基础条件类型 (Timer, ActorState)

### Phase 2: 蓝图支持 (1周)
- ✅ AsyncAction节点
- ✅ 蓝图函数库
- ✅ 编辑器集成

### Phase 3: 高级条件 (1-2周)
- ✅ LevelSequence条件
- ✅ Animation条件
- ✅ Gameplay条件
- ✅ Composite条件

### Phase 4: 调试工具 (1周)
- ✅ 可视化调试器
- ✅ 控制台命令
- ✅ 编辑器可视化

### Phase 5: 优化与扩展 (1周)
- ✅ 性能优化
- ✅ 网络支持
- ✅ 存档系统

---

## 十一、与2077系统的对应关系

| 2077概念 | 虚幻引擎实现 | 说明 |
|----------|-------------|------|
| PauseConditionNode | UWaitConditionNode | 节点类 |
| IBaseCondition | UWaitCondition | 条件基类 |
| NodeExecutionContext | FConditionContext | 执行上下文 |
| EventListenerWrapper | FConditionDelegate | 事件委托 |
| IsFulfilled() | EvaluateInternal() | 条件评估 |
| RegisterEvent() | Initialize() + Bind | 注册监听 |
| UnregisterEvent() | Cleanup() + Unbind | 注销监听 |
| NodeResult::StayInNode | Return Pending | 等待状态 |
| NodeResult::LeaveNode | Trigger OnFulfilled | 完成状态 |
| SectionNode_ConditionType | UWaitCondition_LevelSequence | 序列条件 |
| TokenVariantData | FConditionContext | 上下文数据 |

---

## 十二、下一步行动

1. **创建完整的C++代码** - 我将为你生成所有核心类的完整实现
2. **蓝图示例** - 创建蓝图使用示例
3. **测试场景** - 构建测试关卡验证功能
4. **文档** - 编写详细的使用文档

准备好开始实现了吗？我可以立即为你生成：
- ✅ 完整的头文件和实现文件
- ✅ 蓝图节点代码
- ✅ 示例用法代码
- ✅ 单元测试框架

需要我现在开始生成代码吗？
