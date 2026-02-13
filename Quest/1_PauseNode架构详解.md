# PauseNode 节点架构详解

## 一、概述

PauseNode是Cyberpunk 2077 Quest系统中的核心节点类型，用于暂停Quest信号流，直到指定的Condition条件满足后才继续执行。

---

## 二、类继承结构

```cpp
DisableableNodeDefinition (可禁用节点定义 - 基类)
    ↓
SignalStoppingNodeDefinition (信号停止节点定义)
    ↓
PauseConditionNodeDefinition (暂停条件节点定义) ★核心类
```

**继承说明:**
- `DisableableNodeDefinition`: 提供节点启用/禁用功能
- `SignalStoppingNodeDefinition`: 提供信号拦截和事件监听功能
- `PauseConditionNodeDefinition`: 实现基于条件的暂停逻辑

---

## 三、完整类定义

### 3.1 头文件 (pauseConditionNode.h)

**文件路径:** `common/gameQuest/include/pauseConditionNode.h`

```cpp
/*
* Copyright (c) 2016-2018 CD Projekt Red. All Rights Reserved.
*/

#pragma once

#include "signalStoppingNode.h"

namespace quest
{
    class IBaseCondition;
    class EventListenerWrapper;
    class QuestResave;

    class QUEST_API PauseConditionNodeDefinition : public SignalStoppingNodeDefinition
    {
        RTTI_DECLARE_TYPE( PauseConditionNodeDefinition )

        friend QuestResave;

    private:
        typedef SignalStoppingNodeDefinition Super;

    public:
        // 构造和析构
        PauseConditionNodeDefinition();
        virtual ~PauseConditionNodeDefinition() override;

        // 生命周期回调
        virtual void OnPostLoad( const PostLoadContext& context ) override;
        virtual void OnLeavingPhase( NodeExecutionContext& executionContext,
                                     SignalKillingReason reason ) const override;

        // 节点配置
        virtual void OnCollectSockets( SocketInfoArray& sockets ) const override;
        virtual String GetFriendlyName() const override;
        virtual String GetDescription() const override;

        // 调试和验证
        virtual void DebugGetFactName( red::DynArray< String >& outFacts ) const override;
        virtual void OnValidate( NodeValidationContext& validationContext ) const override;

        // 事件系统
        virtual CName GetEventName() const override;
        virtual red::SharedPtr< EventListenerWrapper > CreateEventListener(
            NodeExecutionContext& executionContext ) const override;

        // Condition访问器
        void SetCondition( const THandle< IBaseCondition >& condition )  { m_condition = condition; }
        Bool HasCondition() const                                        { return static_cast< Bool >( m_condition ); }
        Bool HasDeprecatedCondition() const;

        virtual bool ftGetFulfillInfo(FulfillInfoArray& fulfillInfo) const override;

    protected:
        // ★核心执行逻辑
        virtual NodeResult OnExecute( NodeExecutionContext& executionContext,
                                      const CName inputSocket,
                                      OutputSocketArray& outputSockets ) const override;

        virtual Bool OnLoadGame( NodeExecutionContext& executionContext,
                                 Bool finalNode ) const override;

        // Scene刺激相关
        virtual void DoStimulate( scn::ExecutionStream& outStream,
                                  scn::ActiondefStore& actiondefStore,
                                  Uint32 combinationOffsetMsec,
                                  const scn::IStimulationOracle& oracle,
                                  const scn::ActionSignature& signature ) const override;

        virtual scn::Msec DoEstimateDuration( const scn::IStimulationOracle& oracle ) const override;

    protected:
        // ★核心成员变量：保存条件对象
        THandle< IBaseCondition > m_condition;
    };
} // namespace quest
```

**关键设计点:**

1. **m_condition成员**: 保存条件对象的句柄，这是PauseNode的核心
2. **OnExecute方法**: 实现条件检查和Token拦截逻辑
3. **EventListener机制**: 通过事件监听器异步等待条件满足

---

### 3.2 实现文件 (pauseConditionNode.cpp)

**文件路径:** `common/gameQuest/src/pauseConditionNode.cpp`

```cpp
/*
* Copyright (c) 2016-2018 CD Projekt Red. All Rights Reserved.
*/

#include "build.h"
#include "pauseConditionNode.h"
#include "eventListenerWrapper.h"
#include "condition.h"

#ifdef DEBUG_GAMEPLAY
    #pragma optimize("",off)
#endif

// RTTI类型注册
RTTI_BEGIN_TYPE_IN_NAMESPACE( PauseConditionNodeDefinition, quest );
    RTTI_PARENT_TYPE( quest::SignalStoppingNodeDefinition );
    RTTI_DESCRIPTIVE_NAME( Pause );
    RTTI_PROPERTY( m_condition ).editable().inlined();
RTTI_END_TYPE();

namespace quest
{
    // ============================================================================
    // 构造和析构
    // ============================================================================

    PauseConditionNodeDefinition::PauseConditionNodeDefinition()
    {
    }

    PauseConditionNodeDefinition::~PauseConditionNodeDefinition()
    {
    }

    // ============================================================================
    // 生命周期管理
    // ============================================================================

    void PauseConditionNodeDefinition::OnPostLoad( const PostLoadContext& context )
    {
        Super::OnPostLoad( context );

        // 为条件对象设置扩展节点ID
        if ( m_condition )
        {
            GenericIdGenerator generator;
            m_condition->SetExtendedNodeId( GetId(), generator );
        }
    }

    void PauseConditionNodeDefinition::OnLeavingPhase(
        NodeExecutionContext& executionContext,
        SignalKillingReason reason ) const
    {
        // 通知条件对象，节点正在离开当前阶段
        if ( m_condition )
        {
            m_condition->OnLeavingPhase( executionContext, reason );
        }
    }

    // ============================================================================
    // Socket配置
    // ============================================================================

    void PauseConditionNodeDefinition::OnCollectSockets( SocketInfoArray& sockets ) const
    {
        Super::OnCollectSockets( sockets );

        // 定义输入和输出Socket
        sockets.PushBack( SocketInfo( RED_NAME_CONSTEXPR( "In" ), SocketType::Input ) );
        sockets.PushBack( SocketInfo( RED_NAME_CONSTEXPR( "Out" ), SocketType::Output ) );
    }

    // ============================================================================
    // 显示名称和描述
    // ============================================================================

    String PauseConditionNodeDefinition::GetFriendlyName() const
    {
        // 如果有条件，显示条件的简称
        if ( m_condition )
        {
            return m_condition->GetFriendlyAcronym();
        }
        return "Pause";
    }

    String PauseConditionNodeDefinition::GetDescription() const
    {
        // 如果有条件，显示条件的描述
        if ( m_condition )
        {
            return m_condition->GetDescription();
        }
        return STR_UNDEFINED_DESCRIPTION;
    }

    // ============================================================================
    // 事件监听器
    // ============================================================================

    CName PauseConditionNodeDefinition::GetEventName() const
    {
        // 返回条件的调试名称
        return m_condition->GetDebugName();
    }

    red::SharedPtr< EventListenerWrapper > PauseConditionNodeDefinition::CreateEventListener(
        NodeExecutionContext& executionContext ) const
    {
        // 委托给条件对象创建事件监听器
        return m_condition->CreateEventListener( executionContext );
    }

    // ============================================================================
    // 验证
    // ============================================================================

    void PauseConditionNodeDefinition::OnValidate( NodeValidationContext& validationContext ) const
    {
        if ( m_condition )
        {
            m_condition->OnValidate( validationContext, *this );
        }
        else
        {
            // 没有定义条件，这是一个错误
            QUEST_VALIDATION_ROOT_UNDEFINED_TYPE();
        }
    }

    Bool PauseConditionNodeDefinition::HasDeprecatedCondition() const
    {
        if ( m_condition )
        {
            return m_condition->IsDeprecated();
        }

        // 如果未定义，认为它没有过时
        return false;
    }

    // ============================================================================
    // 调试支持
    // ============================================================================

    void PauseConditionNodeDefinition::DebugGetFactName( red::DynArray< String >& outFacts ) const
    {
        Super::DebugGetFactName( outFacts );
        if ( HasCondition() && m_condition )
        {
            m_condition->DebugGetFactName( outFacts );
        }
    }

    bool PauseConditionNodeDefinition::ftGetFulfillInfo(FulfillInfoArray& fulfillInfo) const
    {
        if (m_condition)
        {
            return m_condition->ftGetFulfillInfo(fulfillInfo);
        }

        return false;
    }

    // ============================================================================
    // ★核心执行逻辑
    // ============================================================================

    NodeResult PauseConditionNodeDefinition::OnExecute(
        NodeExecutionContext& executionContext,
        const CName inputSocket,
        OutputSocketArray& outputSockets ) const
    {
        // 返回 NodeResult::LeaveNode 如果条件已满足，信号立即继续传递
        // 返回 NodeResult::StayInNode 如果条件未满足，添加事件监听器，信号不继续传递

        // 检查条件是否存在
        GAME_DATA_ERROR( m_condition,
                        Quests,
                        "Missing condition in %hs node %hs [%hs]",
                        GetClass()->GetName().AsChar(),
                        executionContext.DebugGetFriendlyFullPathStringWithNode().AsChar(),
                        executionContext.DebugGetFullPathStringWithNode().AsChar() );

        if ( m_condition )
        {
            // ★检查条件是否已满足
            if ( m_condition->IsFulfilled( executionContext ) )
            {
                // 条件已满足，清理事件监听器
                if ( IsEventRegistered( executionContext ) )
                {
                    UnregisterEvent( executionContext );
                }

                // 激活输出Socket，允许信号继续传递
                outputSockets.AddOutput( RED_NAME_CONSTEXPR( "Out" ) );

                // 离开节点，继续执行Quest流程
                return NodeResult::LeaveNode;
            }

            // 条件未满足，注册事件监听器
            if ( !IsEventRegistered( executionContext ) )
            {
                // ★注册事件监听器，等待条件满足
                RegisterEvent( executionContext );
            }

            // ★停留在节点，暂停Quest流程
            return NodeResult::StayInNode;
        }

        // 没有条件定义，直接放行
        outputSockets.AddOutput( RED_NAME_CONSTEXPR( "Out" ) );
        return NodeResult::LeaveNode;
    }

    // ============================================================================
    // 存档加载
    // ============================================================================

    Bool PauseConditionNodeDefinition::OnLoadGame(
        NodeExecutionContext& executionContext,
        Bool finalNode ) const
    {
        // 从存档加载时，重新注册事件监听器
        GAME_ASSERT( !IsEventRegistered( executionContext ), Quests, "Event is already registered" );
        RegisterEvent( executionContext );

        return true;
    }

    // ============================================================================
    // Scene刺激
    // ============================================================================

    void PauseConditionNodeDefinition::DoStimulate(
        scn::ExecutionStream& outStream,
        scn::ActiondefStore& actiondefStore,
        Uint32 combinationOffsetMsec,
        const scn::IStimulationOracle& oracle,
        const scn::ActionSignature& signature ) const
    {
        // 委托给条件对象处理刺激
        if ( m_condition )
        {
            m_condition->Stimulate( outStream, actiondefStore, combinationOffsetMsec, oracle, signature );
        }
    }

    scn::Msec PauseConditionNodeDefinition::DoEstimateDuration(
        const scn::IStimulationOracle& oracle ) const
    {
        // 估算节点执行时长
        return m_condition ? m_condition->EstimateDuration( oracle ) : scn::constants::immediateTime;
    }

} // namespace quest

#ifdef DEBUG_GAMEPLAY
    #pragma optimize("",on)
#endif
```

---

## 四、核心逻辑详解

### 4.1 OnExecute方法流程图

```
OnExecute( executionContext, inputSocket, outputSockets )
    ↓
检查 m_condition 是否存在？
    │
    ├─ 否 → 输出"Out" → 返回LeaveNode (无条件直接放行)
    │
    └─ 是 → 调用 m_condition->IsFulfilled( executionContext )
                ↓
           条件是否满足？
                │
                ├─ 是 → 注销EventListener
                │       → 输出"Out"
                │       → 返回LeaveNode (继续执行)
                │
                └─ 否 → 注册EventListener
                        → 返回StayInNode (暂停等待)
```

### 4.2 事件监听机制

**注册流程:**
```cpp
if ( !IsEventRegistered( executionContext ) )
{
    RegisterEvent( executionContext );  // 注册监听器
}
```

**注销流程:**
```cpp
if ( IsEventRegistered( executionContext ) )
{
    UnregisterEvent( executionContext );  // 注销监听器
}
```

**监听器创建:**
```cpp
red::SharedPtr< EventListenerWrapper > CreateEventListener(
    NodeExecutionContext& executionContext ) const
{
    return m_condition->CreateEventListener( executionContext );
}
```

---

## 五、Socket配置

### 5.1 Socket定义

```cpp
void OnCollectSockets( SocketInfoArray& sockets ) const
{
    Super::OnCollectSockets( sockets );

    // 输入Socket: "In"
    sockets.PushBack( SocketInfo( RED_NAME_CONSTEXPR( "In" ), SocketType::Input ) );

    // 输出Socket: "Out"
    sockets.PushBack( SocketInfo( RED_NAME_CONSTEXPR( "Out" ), SocketType::Output ) );
}
```

### 5.2 Socket连接示例

```
[上游节点] → [In] PauseNode [Out] → [下游节点]
                     ↓
                [Condition]
                 (等待满足)
```

---

## 六、Condition集成

### 6.1 设置Condition

```cpp
void SetCondition( const THandle< IBaseCondition >& condition )
{
    m_condition = condition;
}
```

### 6.2 Condition接口

PauseNode依赖Condition对象实现以下接口:

```cpp
// 检查条件是否满足
Bool IsFulfilled( const NodeExecutionContext& executionContext ) const;

// 创建事件监听器
red::SharedPtr< EventListenerWrapper > CreateEventListener(
    NodeExecutionContext& executionContext ) const;

// 获取显示名称
String GetFriendlyAcronym() const;

// 获取描述
String GetDescription() const;
```

---

## 七、生命周期管理

### 7.1 加载时初始化

```cpp
void OnPostLoad( const PostLoadContext& context )
{
    Super::OnPostLoad( context );

    if ( m_condition )
    {
        // 为条件设置扩展节点ID
        GenericIdGenerator generator;
        m_condition->SetExtendedNodeId( GetId(), generator );
    }
}
```

### 7.2 离开阶段清理

```cpp
void OnLeavingPhase( NodeExecutionContext& executionContext,
                     SignalKillingReason reason ) const
{
    if ( m_condition )
    {
        // 通知条件对象进行清理
        m_condition->OnLeavingPhase( executionContext, reason );
    }
}
```

### 7.3 存档加载恢复

```cpp
Bool OnLoadGame( NodeExecutionContext& executionContext,
                 Bool finalNode ) const
{
    // 从存档加载后，重新注册事件监听器
    GAME_ASSERT( !IsEventRegistered( executionContext ),
                Quests, "Event is already registered" );
    RegisterEvent( executionContext );

    return true;
}
```

---

## 八、使用示例

### 8.1 等待Section完成

```cpp
// 创建PauseNode
THandle< PauseConditionNodeDefinition > pauseNode = CreateHandle< PauseConditionNodeDefinition >();

// 创建SectionNode条件
THandle< SectionNode_ConditionType > sectionCondition =
    CreateHandle< SectionNode_ConditionType >(
        sceneFile,           // Scene资源
        sectionName,         // Section名称
        SceneConditionType::IsOutside  // 条件类型：Section外部
    );

// 创建SceneCondition包装器
THandle< SceneCondition > condition = CreateHandle< SceneCondition >();
condition->SetCondition( sectionCondition );

// 设置到PauseNode
pauseNode->SetCondition( condition );
```

### 8.2 Quest图配置

```
[StartQuest] → [PauseNode: 等待Section完成] → [NextQuestPhase]
                      ↓
               [Condition: Section.IsOutside]
```

---

## 九、调试技巧

### 9.1 获取友好名称

```cpp
String GetFriendlyName() const
{
    if ( m_condition )
    {
        return m_condition->GetFriendlyAcronym();
    }
    return "Pause";
}
```

编辑器中显示为条件的简称，例如:
- "Pause: SectionOutside"
- "Pause: SceneTalking"

### 9.2 获取详细描述

```cpp
String GetDescription() const
{
    if ( m_condition )
    {
        return m_condition->GetDescription();
    }
    return STR_UNDEFINED_DESCRIPTION;
}
```

### 9.3 调试输出

```cpp
void DebugGetFactName( red::DynArray< String >& outFacts ) const
{
    Super::DebugGetFactName( outFacts );
    if ( HasCondition() && m_condition )
    {
        m_condition->DebugGetFactName( outFacts );
    }
}
```

---

## 十、性能考虑

### 10.1 事件驱动设计

- **优点**: 不需要每帧轮询条件
- **机制**: 通过EventListener等待条件变化
- **开销**: 只在条件变化时触发回调

### 10.2 条件检查优化

```cpp
if ( m_condition->IsFulfilled( executionContext ) )
{
    // 条件满足，立即执行
    // 不需要等待下一帧
}
```

### 10.3 内存管理

- 使用THandle智能指针管理Condition对象
- 自动引用计数，防止内存泄漏
- 支持序列化和反序列化

---

## 十一、常见问题

### Q1: PauseNode何时返回StayInNode?
**A:** 当Condition未满足且EventListener已注册时返回StayInNode，Quest信号流会暂停在此节点。

### Q2: EventListener何时被触发?
**A:** 当Condition关注的游戏事件发生时(例如Scene完成、角色对话结束等)，EventListener会触发回调，再次调用OnExecute检查条件。

### Q3: 如果Condition永远无法满足会发生什么?
**A:** Quest会永久卡在PauseNode，建议在设计时添加超时机制或备用路径。

### Q4: PauseNode支持多个Condition吗?
**A:** 单个PauseNode只支持一个Condition，如需多个条件，可以使用AndCondition或OrCondition组合。

---

**文档版本:** 1.0
**最后更新:** 2026-02-13
**源文件:** `common/gameQuest/src/pauseConditionNode.cpp`
