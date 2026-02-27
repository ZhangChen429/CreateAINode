# SectionNode 技术实现深度解析

> 从源码到架构：SectionNode的工程实现剖析

---

## 📋 文档定位

**面向读者：**
- 引擎程序员（想实现类似系统）
- 技术美术（需要深入理解机制）
- 高级设计师（需要了解技术限制）

**内容范围：**
- 源码级实现分析
- 架构设计决策
- 性能优化技巧
- 扩展点设计

---

## 🏗️ 第一部分：类设计剖析

### A. 核心类定义

```cpp
// 文件：scnsSectionNode.h

namespace scn {

class SectionNode : public SceneGraphNode {
    RTTI_DECLARE_TYPE( SectionNode )
    RED_USE_MEMORY_POOL( PoolSceneResources );  // 使用场景资源池
    RED_BASE_CLASS( SceneGraphNode );           // 继承基类

public:
    // === 类型定义 ===
    using ActorBehavior = SectionInternals::ActorBehavior;

    // === 插座定义（静态常量）===
    struct InputSocket {
        static const SocketName in = 0;       // 正常进入
        static const SocketName cancel = 1;   // 取消信号
    };

    struct OutputSocket {
        static const SocketName out = 0;            // 正常输出
        static const SocketName cancelFwd = 1;      // 转发取消
        static const SocketName transmitSignal = 2; // 透传信号
    };

    // === 状态类（嵌套类）===
    class State : public SceneGraphNode::State {
    public:
        // 从节点账户创建状态
        static State CreateFromNodeAccount(
            const NodeAccount& nodeAccount
        );

        // Token相关
        void SetFromTokenData( TokenVariantData data );
        TokenVariantData GetAsTokenData() const;
    };

private:
    // === 核心数据成员 ===
    red::DynArray< THandle< SceneEvent > > m_events;   // 事件列表
    SceneTime m_refrncDuration;                        // 参考时长
    red::DynArray< ActorBehavior > m_actorBehaviors;   // 演员行为
    Bool m_isFocusClue;                                // 是否焦点线索
    SectionCommon m_common;                            // 通用数据

    // 注意：使用PoolSceneResources内存池，避免碎片化
};

} // namespace scn
```

### B. 关键设计决策解析

#### 决策1：为什么继承SceneGraphNode？

```cpp
class SectionNode : public SceneGraphNode {
    // 而不是独立的顶层类
};
```

**原因分析：**

1. **统一的节点处理管线**
   - SceneGraphNode提供了通用的执行框架
   - DoProcess、DoGetEvents等虚函数
   - Section可以像其他节点一样被图执行器处理

2. **插座系统复用**
   - 继承InputSocket/OutputSocket机制
   - 复用信号传播基础设施
   - 不需要重新发明连接系统

3. **多态性支持**
   ```cpp
   // 统一处理所有节点
   void ExecuteNode(SceneGraphNode* node) {
       node->DoProcess(...);  // Section和其他节点统一接口
   }
   ```

#### 决策2：为什么使用DynArray而非StaticArray？

```cpp
red::DynArray< THandle< SceneEvent > > m_events;
// 而不是 red::StaticArray<...>
```

**原因：**

1. **场景复杂度不可预测**
   - 简单Section可能只有5个事件
   - 复杂Section可能有100+事件
   - 动态数组避免内存浪费或不足

2. **支持运行时修改**
   - 编辑器中可以动态添加/删除事件
   - 编译时优化可以移除/合并事件

3. **内存池配合**
   ```cpp
   red::DynArray< ... > m_events{ PoolSceneResources() };
   // 从场景资源池分配，生命周期与场景一致
   ```

#### 决策3：为什么需要SectionCommon？

```cpp
SectionCommon m_common;  // 看似无用的字段
```

**实际作用：**

```cpp
// SectionCommon包含：
struct SectionCommon {
    // 共享配置
    SectionSettings settings;

    // 编译时数据
    CompiledData compiledData;

    // 缓存数据
    CachedMetrics metrics;
};

// 好处：
// 1. 多个Section共享相同配置时节省内存
// 2. 编译时优化的中间结果
// 3. 性能分析的元数据
```

---

## 🔄 第二部分：执行流程分析

### A. Section激活流程

```cpp
// 伪代码：Section被激活时的完整流程

void SectionNode::OnActivate(
    ProcessingContext& context,
    NodeAccount& nodeAccount
) {
    // === 阶段1：预处理 ===
    // 1.1 加载演员
    for (auto& actorBehavior : m_actorBehaviors) {
        ActorEntity* actor = context.actorRegistry->GetActor(
            actorBehavior.actorId
        );

        if (!actor) {
            // 异步加载演员实体
            actor = LoadActorAsync(actorBehavior.actorId);
        }

        // 绑定到Section
        BindActor(actor, actorBehavior);
    }

    // 1.2 预加载资源
    PreloadResources();

    // 1.3 初始化状态
    nodeAccount.SetState(State::Initial);

    // === 阶段2：生成ExecutionStream ===
    // 2.1 遍历所有事件
    ExecutionStream stream;
    for (auto& event : m_events) {
        // 根据事件类型生成动作
        if (auto* dialogEvent = Cast<DialogLineEvent>(event)) {
            // 对话事件 → ActionPlayDialog
            ActionPlayDialog action;
            action.timestamp = dialogEvent->GetStartTime();
            action.duration = dialogEvent->GetDuration();
            action.voiceFile = dialogEvent->GetVoicePath();
            stream.AddAction(action);
        }
        else if (auto* animEvent = Cast<AnimationEvent>(event)) {
            // 动画事件 → ActionPlayAnimation
            // ...
        }
        // ... 其他事件类型
    }

    // 2.2 应用时间缩放
    if (m_common.settings.timeScale != 1.0f) {
        stream.ApplyTimeScale(m_common.settings.timeScale);
    }

    // 2.3 索引优化
    stream.Reindex();

    // === 阶段3：注册到执行器 ===
    context.executionContext->RegisterStream(
        nodeAccount.nodeId,
        stream
    );

    // === 阶段4：触发输出信号（如果需要）===
    if (m_common.settings.triggerOnStart) {
        TriggerOutputSocket(OutputSocket::transmitSignal);
    }
}
```

### B. Section执行中的处理

```cpp
void SectionNode::DoProcess(
    ProcessingResult& result,
    ProcessingArgs args,
    NodeAccount& nodeAccount,
    const ProcessingContext& processingContext,
    ExecutionContext& executionContext
) const {
    // === 每帧调用 ===

    // 1. 检查中断条件（如果Section配置了中断）
    if (m_common.interruptSettings.enabled) {
        if (CheckInterruptConditions(processingContext)) {
            // 触发中断流程
            result.requestInterrupt = true;
            result.interruptReason = GetInterruptReason();
            return;
        }
    }

    // 2. 更新Section内部时间
    SceneTime currentTime = nodeAccount.GetCurrentTime();
    SceneTime deltaTime = args.deltaTime;

    SceneTime newTime = currentTime + deltaTime;
    nodeAccount.SetCurrentTime(newTime);

    // 3. 执行ExecutionStream
    auto& stream = executionContext.GetStream(nodeAccount.nodeId);

    // 获取当前时间段内到期的动作
    auto actions = stream.GetActionsInRange(currentTime, newTime);

    for (auto& action : actions) {
        // 执行动作（委托给ActionsExecutor）
        executionContext.actionsExecutor->Execute(action);
    }

    // 4. 检查是否完成
    if (newTime >= m_refrncDuration) {
        // Section执行完毕
        result.completed = true;

        // 触发输出插座
        TriggerOutputSocket(OutputSocket::out, result);

        // 清理资源
        CleanupSection(nodeAccount);
    }
}
```

### C. Section中断处理

```cpp
// 中断发生时的处理
void SectionNode::OnInterrupt(
    InterruptContext& context,
    NodeAccount& nodeAccount
) {
    // === 保存状态 ===
    State state;
    state.savedTime = nodeAccount.GetCurrentTime();
    state.savedVariables = nodeAccount.GetVariables();

    // 保存演员状态
    for (auto& actor : GetBoundActors()) {
        state.actorStates[actor->GetId()] = actor->SaveState();
    }

    // 生成StateToken（紧凑表示）
    StateToken token = state.Serialize();
    context.SaveToken(nodeAccount.nodeId, token);

    // === 暂停ExecutionStream ===
    auto& stream = context.executionContext->GetStream(nodeAccount.nodeId);
    stream.Pause();

    // === 释放临时资源 ===
    // 但保留关键状态
    ReleaseTemporaryResources();

    // === 触发中断信号 ===
    TriggerOutputSocket(OutputSocket::cancel, context.result);
}

// 中断返回时的处理
void SectionNode::OnResume(
    ResumeContext& context,
    NodeAccount& nodeAccount
) {
    // === 恢复状态 ===
    StateToken token = context.GetSavedToken(nodeAccount.nodeId);
    State state = State::Deserialize(token);

    // 恢复时间
    nodeAccount.SetCurrentTime(state.savedTime);

    // 恢复变量
    nodeAccount.SetVariables(state.savedVariables);

    // 恢复演员
    for (auto& [actorId, actorState] : state.actorStates) {
        auto* actor = GetActor(actorId);
        actor->RestoreState(actorState);
    }

    // === 重新加载必要资源 ===
    ReloadNecessaryResources();

    // === 恢复ExecutionStream ===
    auto& stream = context.executionContext->GetStream(nodeAccount.nodeId);
    stream.Resume();
}
```

---

## 🎨 第三部分：高级特性实现

### A. 层次化时间轴

```cpp
// 时间缩放器设计
class SectionNode {
    virtual scaling::Scaler DoBuildScaler(
        const ProcessingContext& processingContext
    ) const override {

        scaling::Scaler scaler;

        // 1. 基础时间缩放
        scaler.baseScale = m_common.settings.timeScale;

        // 2. 动态调整（如玩家跳过）
        if (processingContext.playerSkipping) {
            scaler.dynamicScale = 2.0f;  // 2倍速
        }

        // 3. 缓动曲线（进入/退出时平滑）
        scaler.easingCurve = EasingCurve::SmoothStep;

        // 4. 时间偏移（Section在父时间轴中的位置）
        scaler.timeOffset = GetParentTimeOffset();

        return scaler;
    }
};

// 时间转换
SceneTime SectionNode::ConvertGlobalToLocalTime(
    SceneTime globalTime
) const {
    auto scaler = DoBuildScaler(currentContext);

    // 全局时间 → Section局部时间
    SceneTime localTime = (globalTime - scaler.timeOffset)
                        / scaler.baseScale
                        / scaler.dynamicScale;

    return localTime;
}
```

### B. 演员生命周期管理

```cpp
// ActorBehavior详细定义
struct ActorBehavior {
    ActorId actorId;                    // 演员ID
    ActorControlMode controlMode;        // 控制模式
    Bool loadOnDemand;                   // 按需加载
    Bool unloadOnExit;                   // 退出时卸载
    AIMode aiMode;                       // AI模式

    // 扩展配置
    struct ExtendedConfig {
        Bool useSceneAI;                 // 使用场景AI
        Bool freezeWhenNotVisible;       // 不可见时冻结
        Float lodDistance;               // LOD距离
    } config;
};

// 演员管理实现
class SectionNode {
private:
    void BindActor(
        ActorEntity* actor,
        const ActorBehavior& behavior
    ) {
        // 1. 接管控制权
        if (behavior.useSceneAI) {
            actor->SwitchToSceneAI();
            actor->SetAIController(GetSectionAIController());
        }

        // 2. 配置LOD
        actor->SetLODDistance(behavior.config.lodDistance);

        // 3. 注册到Section
        m_boundActors.PushBack(actor);

        // 4. 通知ActorRegistry
        actorRegistry->NotifyActorBound(actor->GetId(), this);
    }

    void UnbindActor(ActorEntity* actor) {
        // 1. 释放控制权
        actor->RestoreNormalAI();

        // 2. 如果配置为unloadOnExit，卸载实体
        if (GetActorBehavior(actor).unloadOnExit) {
            actorRegistry->UnloadActor(actor->GetId());
        }

        // 3. 从Section移除
        m_boundActors.Remove(actor);

        // 4. 通知ActorRegistry
        actorRegistry->NotifyActorUnbound(actor->GetId(), this);
    }
};
```

### C. 资源预加载策略

```cpp
// 智能预加载系统
class SectionResourcePreloader {
public:
    void PreloadSection(
        SectionNode* section,
        PreloadPriority priority
    ) {
        // 1. 扫描Section的所有事件
        SceneEventsRefs events;
        section->DoGetEvents(events);

        // 2. 分类资源
        ResourceBundle bundle;
        for (auto& event : events) {
            ClassifyEventResources(event, bundle);
        }

        // 3. 按优先级排序
        bundle.Sort([](const Resource& a, const Resource& b) {
            return a.priority > b.priority;
        });

        // 4. 异步加载
        for (auto& resource : bundle.resources) {
            AsyncLoad(resource, priority);
        }
    }

private:
    void ClassifyEventResources(
        SceneEvent* event,
        ResourceBundle& bundle
    ) {
        if (auto* dialogEvent = Cast<DialogLineEvent>(event)) {
            // 语音文件：高优先级
            bundle.Add(
                dialogEvent->voiceFile,
                ResourcePriority::High
            );

            // 口型动画：中优先级
            bundle.Add(
                dialogEvent->lipsyncData,
                ResourcePriority::Medium
            );
        }
        else if (auto* vfxEvent = Cast<VfxEvent>(event)) {
            // VFX：低优先级（可以延迟加载）
            bundle.Add(
                vfxEvent->effectAsset,
                ResourcePriority::Low
            );
        }
        // ... 其他事件类型
    }
};

// 预测性预加载
void PredictivePreload(SectionNode* currentSection) {
    // 分析可能的下一个Section
    auto nextSections = currentSection->GetPossibleNextSections();

    for (auto* nextSection : nextSections) {
        // 计算概率
        Float probability = CalculateTransitionProbability(
            currentSection,
            nextSection
        );

        // 根据概率决定优先级
        PreloadPriority priority;
        if (probability > 0.7f) {
            priority = PreloadPriority::High;
        } else if (probability > 0.3f) {
            priority = PreloadPriority::Medium;
        } else {
            priority = PreloadPriority::Low;
        }

        // 异步预加载
        preloader->PreloadSection(nextSection, priority);
    }
}
```

### D. 状态Token序列化

```cpp
// 紧凑的状态表示
class SectionNode::State {
public:
    TokenVariantData GetAsTokenData() const {
        // === 目标：将状态压缩到最小 ===

        TokenVariantData token;

        // 1. 时间位置（4字节）
        token.WriteSceneTime(m_savedTime);

        // 2. 变量快照（变长，但通常<100字节）
        token.WriteVariables(m_savedVariables);

        // 3. 关键状态标志（1字节）
        Uint8 flags = 0;
        if (m_wasInterrupted) flags |= 0x01;
        if (m_playerMadeChoice) flags |= 0x02;
        // ...
        token.WriteUint8(flags);

        // 4. 演员状态索引（不保存完整状态，只保存索引）
        // 演员可以从ActorRegistry重建
        token.WriteActorIds(m_boundActorIds);

        // 总大小：通常100-300字节
        // 对比扁平方案的5-10KB，压缩95%+

        return token;
    }

    void SetFromTokenData(TokenVariantData token) {
        // 反序列化
        m_savedTime = token.ReadSceneTime();
        m_savedVariables = token.ReadVariables();

        Uint8 flags = token.ReadUint8();
        m_wasInterrupted = (flags & 0x01) != 0;
        m_playerMadeChoice = (flags & 0x02) != 0;

        m_boundActorIds = token.ReadActorIds();

        // 重建演员状态（从ActorRegistry）
        for (auto actorId : m_boundActorIds) {
            auto* actor = actorRegistry->GetActor(actorId);
            // actor会根据当前Section配置自动重建状态
        }
    }
};
```

---

## 🔧 第四部分：性能优化技巧

### A. 内存优化

```cpp
// 1. 使用内存池
class SectionNode {
    RED_USE_MEMORY_POOL( PoolSceneResources );

    // 好处：
    // - 减少内存碎片
    // - 批量分配/释放
    // - 生命周期与场景一致
};

// 2. 延迟初始化
class SectionNode {
private:
    mutable THandle< ExecutionStream > m_cachedStream;  // 缓存

    ExecutionStream& GetStream() const {
        if (!m_cachedStream) {
            // 首次访问时生成
            m_cachedStream = GenerateExecutionStream();
        }
        return *m_cachedStream;
    }
};

// 3. 共享数据
struct SectionCommon {
    // 多个Section共享相同的配置
    SharedPtr<SectionSettings> settings;

    // 而不是每个Section都复制一份
};
```

### B. CPU优化

```cpp
// 1. ExecutionStream索引
void SectionNode::OnActivate() {
    auto stream = GenerateExecutionStream();

    // 构建索引以加速时间查询
    stream->Reindex();
    // O(n) → O(log n)
}

// 2. 事件缓存
class SectionNode {
private:
    mutable Bool m_eventsCached = false;
    mutable red::DynArray<SceneEvent*> m_cachedEvents;

    virtual void DoGetEvents(SceneEventsRefs& outEvents) const override {
        if (!m_eventsCached) {
            // 首次缓存
            for (auto& eventHandle : m_events) {
                m_cachedEvents.PushBack(eventHandle.Get());
            }
            m_eventsCached = true;
        }

        // 直接返回缓存
        outEvents = m_cachedEvents;
    }
};

// 3. 批量处理
void ExecuteMultipleSections(
    red::ArraySpan<SectionNode*> sections
) {
    // 批量预加载资源
    ResourceBatch batch;
    for (auto* section : sections) {
        section->CollectResources(batch);
    }
    batch.LoadAll();  // 一次性加载

    // 批量激活
    for (auto* section : sections) {
        section->Activate();
    }
}
```

### C. 资源优化

```cpp
// 1. 引用计数管理
class SectionNode {
    void OnActivate() {
        // 增加资源引用
        for (auto& resource : GetRequiredResources()) {
            resource->AddRef();
        }
    }

    void OnDeactivate() {
        // 减少资源引用（可能触发卸载）
        for (auto& resource : GetRequiredResources()) {
            resource->Release();
        }
    }
};

// 2. 流式加载
void PreloadSection(SectionNode* section) {
    // 分阶段加载

    // 阶段1：关键资源（阻塞）
    LoadCriticalResources(section);

    // 阶段2：次要资源（异步）
    AsyncLoadSecondaryResources(section);

    // 阶段3：可选资源（后台）
    BackgroundLoadOptionalResources(section);
}

// 3. 资源卸载策略
void OnSectionExit(SectionNode* section) {
    // 检查资源是否被其他Section需要
    for (auto& resource : section->GetLoadedResources()) {
        if (!IsNeededByOtherSections(resource)) {
            // 安全卸载
            UnloadResource(resource);
        }
    }
}
```

---

## 🎯 第五部分：扩展点设计

### A. 自定义Section类型

```cpp
// 扩展SectionNode以支持特定功能

// 示例1：战斗Section
class CombatSectionNode : public SectionNode {
public:
    struct CombatSettings {
        red::DynArray<EnemyWave> waves;     // 敌人波次
        CombatDifficulty difficulty;        // 难度
        Bool allowRetreat;                  // 允许撤退
    };

private:
    CombatSettings m_combatSettings;

    virtual void DoProcess(...) const override {
        // 扩展：战斗逻辑
        if (AllEnemiesDefeated()) {
            TriggerOutputSocket(OutputSocket::out);
        }

        // 调用基类
        SectionNode::DoProcess(...);
    }
};

// 示例2：选择Section
class ChoiceSectionNode : public SectionNode {
public:
    struct Choice {
        CName choiceId;
        LocalizedString text;
        red::DynArray<Condition> conditions;  // 显示条件
        OutputSocketId targetSocket;          // 选择后的输出
    };

private:
    red::DynArray<Choice> m_choices;
    Float m_timeoutDuration;

    virtual void DoProcess(...) const override {
        // 显示选择UI
        ShowChoiceUI(m_choices);

        // 等待玩家选择或超时
        if (PlayerMadeChoice()) {
            auto choice = GetPlayerChoice();
            TriggerOutputSocket(choice.targetSocket);
        }
        else if (TimedOut()) {
            TriggerOutputSocket(OutputSocket::out);  // 默认
        }
    }
};
```

### B. 插件系统

```cpp
// Section插件接口
class ISectionPlugin {
public:
    virtual void OnSectionActivate(SectionNode* section) = 0;
    virtual void OnSectionUpdate(SectionNode* section, Float deltaTime) = 0;
    virtual void OnSectionDeactivate(SectionNode* section) = 0;
};

// 示例：性能分析插件
class PerformanceAnalyzerPlugin : public ISectionPlugin {
    void OnSectionActivate(SectionNode* section) override {
        m_startTime = GetCurrentTime();
        m_peakMemory = 0;
    }

    void OnSectionUpdate(SectionNode* section, Float deltaTime) override {
        // 采样性能数据
        Uint64 currentMemory = GetMemoryUsage();
        m_peakMemory = Max(m_peakMemory, currentMemory);
    }

    void OnSectionDeactivate(SectionNode* section) override {
        Float duration = GetCurrentTime() - m_startTime;

        // 输出报告
        LOG("Section %s: Duration=%.2fs, PeakMemory=%lluMB",
            section->GetName(), duration, m_peakMemory / 1024 / 1024);
    }

private:
    Float m_startTime;
    Uint64 m_peakMemory;
};

// Section注册插件
void SectionNode::RegisterPlugin(ISectionPlugin* plugin) {
    m_plugins.PushBack(plugin);
}
```

### C. 事件系统扩展

```cpp
// 自定义事件类型
class CustomSceneEvent : public SceneEvent {
    RTTI_DECLARE_TYPE(CustomSceneEvent);

public:
    // 自定义数据
    CName customAction;
    red::DynArray<Variant> parameters;

    // 执行逻辑
    virtual void Execute(ExecutionContext& context) override {
        // 调用自定义系统
        customSystem->ExecuteAction(customAction, parameters);
    }
};

// Section扫描并处理自定义事件
void SectionNode::DoGetEvents(SceneEventsRefs& outEvents) const {
    for (auto& eventHandle : m_events) {
        auto* event = eventHandle.Get();

        // 支持自定义事件
        if (auto* customEvent = Cast<CustomSceneEvent>(event)) {
            // 特殊处理
            HandleCustomEvent(customEvent);
        }

        outEvents.PushBack(event);
    }
}
```

---

## 🔬 第六部分：调试与分析

### A. 调试接口

```cpp
class SectionNode {
#ifndef RED_CONFIGURATION_FINAL
    // === 编辑器专用接口 ===

    // 跳转到特定时间点
    virtual void DoEditor_JumpToTimepos(
        LiveEditResult& result,
        MemoryControl& memoryControl,
        NodeProcessingState& nodeProcessingState,
        const ProcessingContext& processingContext,
        NodeRunstateStorage& nodeRunstateStorage,
        const TimeWindow currentNodeTwActivity,
        const scn::SceneTime jumpToTime
    ) const override {
        // 1. 保存当前状态
        auto currentState = SaveCurrentState(nodeProcessingState);

        // 2. 快速重建到目标时间
        // 不需要实际执行，直接设置状态
        nodeProcessingState.SetCurrentTime(jumpToTime);

        // 3. 激活到该时间点的所有事件
        ActivateEventsUpToTime(jumpToTime, nodeRunstateStorage);

        // 4. 通知LiveEdit系统
        result.success = true;
        result.newTimePos = jumpToTime;
    }

    // 开始处理（调试模式）
    virtual void DoEditor_StartProcessing(...) const override {
        // 在编辑器中独立测试Section
        // 不需要完整的场景上下文
    }

    // 停止处理
    virtual void DoEditor_StopProcessing(...) const override {
        // 清理调试状态
    }
#endif
};
```

### B. 性能分析

```cpp
// 性能分析宏
#define SECTION_PROFILE_SCOPE(sectionName) \
    ScopedSectionProfiler __profiler(sectionName)

class ScopedSectionProfiler {
public:
    ScopedSectionProfiler(const char* name)
        : m_name(name), m_startTime(GetHighResTime()) {}

    ~ScopedSectionProfiler() {
        Float duration = GetHighResTime() - m_startTime;
        g_profiler->RecordSectionTime(m_name, duration);
    }

private:
    const char* m_name;
    Float m_startTime;
};

// 使用
void SectionNode::DoProcess(...) const {
    SECTION_PROFILE_SCOPE(GetName());

    // Section执行逻辑...
}

// 分析报告
void PrintSectionProfile() {
    auto stats = g_profiler->GetSectionStats();

    LOG("Section Performance Report:");
    for (auto& [name, stat] : stats) {
        LOG("  %s: Avg=%.2fms, Max=%.2fms, Count=%u",
            name.AsChar(),
            stat.avgDuration,
            stat.maxDuration,
            stat.callCount);
    }
}
```

### C. 可视化工具

```cpp
// Section可视化数据导出
struct SectionVisualizationData {
    CName sectionName;
    SceneTime startTime;
    SceneTime duration;
    red::DynArray<CName> boundActors;
    red::DynArray<EventVisualization> events;

    struct EventVisualization {
        CName eventType;
        SceneTime timestamp;
        Color color;
    };
};

SectionVisualizationData SectionNode::ExportVisualizationData() const {
    SectionVisualizationData data;
    data.sectionName = GetName();
    data.startTime = GetStartTime();
    data.duration = m_refrncDuration;

    // 导出演员
    for (auto& behavior : m_actorBehaviors) {
        data.boundActors.PushBack(behavior.actorId);
    }

    // 导出事件（用于时间轴可视化）
    for (auto& event : m_events) {
        EventVisualization vis;
        vis.eventType = event->GetType()->GetName();
        vis.timestamp = event->GetStartTime();
        vis.color = GetEventColor(event->GetType());
        data.events.PushBack(vis);
    }

    return data;
}
```

---

## 📚 第七部分：最佳实践总结

### A. 代码规范

```cpp
// ✅ 好的实践

// 1. 使用内存池
class MySectionNode : public SectionNode {
    RED_USE_MEMORY_POOL( PoolSceneResources );
};

// 2. 虚函数覆盖使用override
virtual void DoProcess(...) const override;

// 3. const正确性
const red::DynArray<ActorBehavior>& GetActorBehaviors() const;

// 4. 前置声明减少依赖
class SectionNode;  // 前置声明
#include "sectionNode.h"  // 只在需要时包含

// ❌ 不好的实践

// 1. 在头文件中包含大量依赖
#include "hugeHeader.h"  // 会拖慢编译

// 2. 公开内部数据
public:
    red::DynArray< THandle< SceneEvent > > m_events;  // 应该private

// 3. 缺少const
void DoSomething();  // 应该是 const如果不修改状态
```

### B. 性能注意事项

```cpp
// ✅ 高效

// 1. 预分配容量
void SectionNode::ReserveCapacity(Uint32 estimatedEventCount) {
    m_events.Reserve(estimatedEventCount);
}

// 2. 使用ArraySpan避免拷贝
red::ArraySpan<const ActorBehavior> GetActorBehaviors() const;

// 3. 缓存频繁访问的数据
mutable SceneTime m_cachedDuration = INVALID_TIME;

// ❌ 低效

// 1. 每次都重新计算
SceneTime GetDuration() const {
    SceneTime total = 0;
    for (auto& event : m_events) {
        total += event->GetDuration();  // 重复计算
    }
    return total;
}

// 2. 不必要的拷贝
red::DynArray<ActorBehavior> GetActorBehaviors() const {
    return m_actorBehaviors;  // 拷贝整个数组！
}
```

### C. 可维护性

```cpp
// ✅ 易维护

// 1. 清晰的命名
struct ActorBehavior {
    ActorId actorId;              // 清晰
    ActorControlMode controlMode;  // 清晰
};

// 2. 注释关键逻辑
// 检查中断条件（每帧调用，需要高效）
if (CheckInterruptConditions()) {
    // 触发中断分支...
}

// 3. 使用枚举而非魔数
enum class SectionPriority {
    Low = 0,
    Medium = 1,
    High = 2
};

// ❌ 难维护

// 1. 神秘缩写
struct ActBhv {
    Aid aid;  // ？？？
    Acm acm;  // ？？？
};

// 2. 魔数
if (priority == 2) {  // 2是什么意思？
    // ...
}
```

---

## 🎓 总结

### 核心技术要点

1. **继承架构**
   - 继承SceneGraphNode复用基础设施
   - 虚函数覆盖实现特定逻辑

2. **内存管理**
   - 使用内存池避免碎片
   - 动态数组支持灵活大小
   - 引用计数管理资源生命周期

3. **性能优化**
   - ExecutionStream索引加速查询
   - 预加载策略减少卡顿
   - 批量处理提高效率

4. **扩展性**
   - 插件系统支持功能扩展
   - 自定义事件类型
   - 调试接口辅助开发

### 实现清单

如果要实现类似系统，需要：

```
□ 基础节点系统
  □ SceneGraphNode基类
  □ 插座连接机制
  □ 信号传播系统

□ SectionNode核心
  □ 事件容器
  □ 演员管理
  □ 时间轴系统
  □ 状态序列化

□ 执行流
  □ ExecutionStream生成
  □ ActionChannel/ControlChannel
  □ 时间缩放器

□ 资源管理
  □ 预加载系统
  □ 引用计数
  □ 流式加载

□ 工具支持
  □ 编辑器集成
  □ 调试接口
  □ 性能分析

□ 优化
  □ 内存池
  □ 索引优化
  □ 批量处理
```

---

*技术实现文档 v1.0*
*基于 Cyberpunk 2077 源码分析*
*2026-02-25*
