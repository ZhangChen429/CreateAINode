 ---
  一、基础节点类型（4个）

  1. CompositeTreeNode

  📁 aiCompositeNode.h:20功能：组合节点基类，可以有多个子节点（最多16个），管理子节点的执行顺序

  2. DecoratorNode

  📁 aiDecoratorNode.h:20功能：装饰器节点基类，包装并修改单个子节点的行为

  3. LeafTreeNode

  📁 aiLeafNode.h:15功能：叶子节点基类，没有子节点，代表行为树的终端动作

  4. FSMTreeNode

  📁 aiFSMNode.h:27功能：有限状态机节点，管理多个状态和状态间的转换

  ---
  二、控制流节点（4个）

  5. SequenceTreeNode ⛓️

  📁 aiSequenceNode.h:17功能：顺序执行子节点，所有子节点成功才成功，任一失败则失败（类似逻辑 AND）

  6. SelectorTreeNode 🎯

  📁 aiSelectorNode.h:17功能：按序执行子节点直到一个成功，任一成功则成功，全部失败才失败（类似逻辑 OR）

  7. ParallelNode ⚡

  📁 aiParallelNode.h:15功能：并行执行多个子节点，可配置等待所有/任一/特定子节点完成

  8. IfElseNode 🔀

  📁 aiIfElseNode.h:21功能：条件分支节点，根据表达式执行 true 或 false 分支

  ---
  三、装饰器节点（7个）

  9. TaskNode 📋

  📁 aiTaskNode.h:23功能：任务节点抽象基类，在子节点前或期间执行任务

  10. ConditionNode ❓

  📁 aiConditionNode.h:23功能：条件节点抽象基类，在执行子节点前评估条件

  11. EventHandlerNode 📡

  📁 aiEventHandlerNode.h:18功能：监听特定 AI 事件，事件触发时执行子节点

  12. LimiterNode ⏱️

  📁 aiLimiterNode.h:13功能：限制子节点每帧激活次数，实现延迟激活调度

  13. MaybeNode 🎲

  📁 aiMaybeNode.h:24功能：可配置的装饰器，能让子节点成功/失败/重复

  14. PassiveConditionNode 👁️

  📁 aiPassiveConditionNode.h:21功能：持续评估被动条件并响应变化

  15. CommandHandlerNode 🎮

  📁 aiCommandHandlerNode.h:20功能：处理 AI 命令，支持等待和重试逻辑

  ---
  四、任务/条件节点（4个）

  16. InstantTaskNode ⚡📋

  📁 aiInstantTaskNode.h:17功能：立即执行任务不监控，激活后直接继续执行子节点

  17. MonitorTaskNode 👀📋

  📁 aiMonitorTaskNode.h:17功能：持续监控任务执行，支持超时

  18. InstantConditionNode ⚡❓

  📁 aiInstantConditionNode.h:15功能：即时评估条件一次，根据结果继续执行（编辑器简称 "IC"）

  19. MonitorConditionNode 👀❓

  📁 aiMonitorConditionNode.h:15功能：持续监控条件，支持超时（编辑器简称 "MC"）

  ---
  五、叶子节点（2个）

  20. SucceederNode ✅

  📁 aiCompleteNodes.h:15功能：激活后立即成功，用于测试或强制成功

  21. FailerNode ❌

  📁 aiCompleteNodes.h:32功能：激活后立即失败，用于测试或强制失败

  ---
  六、状态机节点（1个）

  22. FSMState 🔄

  📁 aiFSMStateNode.h:31功能：FSM 中的单个状态，包含行为根节点并管理到其他状态的转换

  ---
  七、嵌套树节点（3个）

  23. NestedTree 🌳

  📁 aiIncludedTreeNode.h:24功能：嵌套树抽象基类，执行嵌套的行为树

  24. IncludedTree 📦

  📁 aiIncludedTreeNode.h:79功能：通过参数映射动态获取并执行行为树引用

  25. Subtree 🌲

  📁 aiIncludedTreeNode.h:114功能：执行静态定义的参数化行为树作为子树

  ---
  总结

  总共 25 个子类 构成了完整的 AI 行为树系统：

  - 4个基础节点：Composite、Decorator、Leaf、FSM
  - 4个控制流节点：Sequence、Selector、Parallel、IfElse
  - 11个装饰器系列：各种任务、条件、事件处理器
  - 2个叶子节点：Succeeder、Failer
  - 3个嵌套树节点：支持模块化行为树设计
  - 1个状态机状态节点：FSMState

  这个系统设计非常完善，支持复杂的 AI 行为组合，包括条件判断、并行执行、事件响应、状态机等高级特性。



  
  # TreeNode 子类完整列表

  ## 一、基础节点类型

  | 序号 | 类名              | 文件路径          | 行号 | 功能描述                                                         | 图示 |
  | ---- | ----------------- | ----------------- | ---- | ---------------------------------------------------------------- | ---- |
  | 1    | CompositeTreeNode | aiCompositeNode.h | 20   | 组合节点基类，可以有多个子节点（最多16个），管理子节点的执行顺序 |      |
  | 2    | DecoratorNode     | aiDecoratorNode.h | 20   | 装饰器节点基类，包装并修改单个子节点的行为                       |      |
  | 3    | LeafTreeNode      | aiLeafNode.h      | 15   | 叶子节点基类，没有子节点，代表行为树的终端动作                   |      |
  | 4    | FSMTreeNode       | aiFSMNode.h       | 27   | 有限状态机节点，管理多个状态和状态间的转换                       |      |

  ## 二、控制流节点

  | 序号 | 类名 | 文件路径 | 行号 | 父类 | 功能描述 |
  |------|------|----------|------|------|----------|
  | 5 | SequenceTreeNode | aiSequenceNode.h | 17 | CompositeTreeNode |顺序执行子节点，所有子节点成功才成功，任一失败则失败（逻辑AND） |
  | 6 | SelectorTreeNode | aiSelectorNode.h | 17 | CompositeTreeNode |按序执行子节点直到一个成功，任一成功则成功，全部失败才失败（逻辑OR） |
  | 7 | ParallelNode | aiParallelNode.h | 15 | CompositeTreeNode |并行执行多个子节点，可配置等待所有/任一/特定子节点完成 |
  | 8 | IfElseNode | aiIfElseNode.h | 21 | CompositeTreeNode | 条件分支节点，根据表达式执行true或false分支 |

  ## 三、装饰器节点

  | 序号 | 类名 | 文件路径 | 行号 | 父类 | 功能描述 |
  |------|------|----------|------|------|----------|
  | 9 | TaskNode | aiTaskNode.h | 23 | DecoratorNode | 任务节点抽象基类，在子节点前或期间执行任务 |
  | 10 | ConditionNode | aiConditionNode.h | 23 | DecoratorNode | 条件节点抽象基类，在执行子节点前评估条件 |
  | 11 | EventHandlerNode | aiEventHandlerNode.h | 18 | DecoratorNode | 监听特定AI事件，事件触发时执行子节点 |
  | 12 | LimiterNode | aiLimiterNode.h | 13 | DecoratorNode | 限制子节点每帧激活次数，实现延迟激活调度 |
  | 13 | MaybeNode | aiMaybeNode.h | 24 | DecoratorNode | 可配置的装饰器，能让子节点成功/失败/重复 |
  | 14 | PassiveConditionNode | aiPassiveConditionNode.h | 21 | DecoratorNode | 持续评估被动条件并响应变化 |
  | 15 | CommandHandlerNode | aiCommandHandlerNode.h | 20 | DecoratorNode | 处理AI命令，支持等待和重试逻辑 |

  ## 四、任务/条件节点

  | 序号 | 类名 | 文件路径 | 行号 | 父类 | 功能描述 |
  |------|------|----------|------|------|----------|
  | 16 | InstantTaskNode | aiInstantTaskNode.h | 17 | TaskNode | 立即执行任务不监控，激活后直接继续执行子节点 |
  | 17 | MonitorTaskNode | aiMonitorTaskNode.h | 17 | TaskNode | 持续监控任务执行，支持超时 |
  | 18 | InstantConditionNode | aiInstantConditionNode.h | 15 | ConditionNode |即时评估条件一次，根据结果继续执行（编辑器简称"IC"） |
  | 19 | MonitorConditionNode | aiMonitorConditionNode.h | 15 | ConditionNode |持续监控条件，支持超时（编辑器简称"MC"） |

  ## 五、叶子节点

  | 序号 | 类名 | 文件路径 | 行号 | 父类 | 功能描述 |
  |------|------|----------|------|------|----------|
  | 20 | SucceederNode | aiCompleteNodes.h | 15 | LeafTreeNode | 激活后立即成功，用于测试或强制成功 |
  | 21 | FailerNode | aiCompleteNodes.h | 32 | LeafTreeNode | 激活后立即失败，用于测试或强制失败 |

  ## 六、状态机节点

  | 序号 | 类名 | 文件路径 | 行号 | 父类 | 功能描述 |
  |------|------|----------|------|------|----------|
  | 22 | FSMState | aiFSMStateNode.h | 31 | TreeNode | FSM中的单个状态，包含行为根节点并管理到其他状态的转换 |

  ## 七、嵌套树节点

  | 序号 | 类名 | 文件路径 | 行号 | 父类 | 功能描述 |
  |------|------|----------|------|------|----------|
  | 23 | NestedTree | aiIncludedTreeNode.h | 24 | TreeNode | 嵌套树抽象基类，执行嵌套的行为树 |
  | 24 | IncludedTree | aiIncludedTreeNode.h | 79 | NestedTree | 通过参数映射动态获取并执行行为树引用 |
  | 25 | Subtree | aiIncludedTreeNode.h | 114 | NestedTree | 执行静态定义的参数化行为树作为子树 |

  ## 统计汇总

  | 类别 | 数量 | 说明 |
  |------|------|------|
  | 基础节点类型 | 4 | Composite、Decorator、Leaf、FSM |
  | 控制流节点 | 4 | Sequence、Selector、Parallel、IfElse |
  | 装饰器系列 | 7 | 各种任务、条件、事件处理器 |
  | 任务/条件节点 | 4 | Instant和Monitor的Task/Condition变体 |
  | 叶子节点 | 2 | Succeeder、Failer |
  | 状态机节点 | 1 | FSMState |
  | 嵌套树节点 | 3 | NestedTree、IncludedTree、Subtree |
  | **总计** | **25** | 完整的AI行为树系统 |


    TreeNode 子类参数完整列表

  📋 参数类型说明

  - 构造参数: 创建节点时必需的参数
  - 配置参数: Definition中定义的配置项（前缀 m_）
  - 运行时变量: 运行时实例状态（前缀 i_，类型 TInstanceVar）



  
  ---
  一、基础节点类型

  1. CompositeTreeNode (组合节点基类)

  文件: aiCompositeNode.h:20

  | 参数类型 | 参数名        | 类型                           | 说明           |
  |------|------------|------------------------------|--------------|
  | 构造参数 | parentNode | TreeNode*                    | 父节点指针        |
  | 构造参数 | def        | CompositeTreeNodeDefinition& | 节点定义         |
  | 成员变量 | m_children | StaticArray<TreeNode*, 16>   | 子节点数组（最多16个） |
  | 配置参数 | m_children | DynArray<THandle>            | 子节点定义列表      |

  ---
  2. DecoratorNode (装饰器节点基类)

  文件: aiDecoratorNode.h:20

  | 参数类型  | 参数名             | 类型                       | 说明      |
  |-------|-----------------|--------------------------|---------|
  | 构造参数  | parentNode      | TreeNode*                | 父节点指针   |
  | 构造参数  | definition      | DecoratorNodeDefinition& | 节点定义    |
  | 成员变量  | m_child         | TreeNode*                | 单个子节点   |
  | 运行时变量 | i_childIsActive | Bool                     | 子节点是否激活 |
  | 配置参数  | m_child         | THandle                  | 子节点定义   |

  ---
  3. LeafTreeNode (叶子节点基类)

  文件: aiLeafNode.h:15

  | 参数类型 | 参数名        | 类型                      | 说明         |
  |------|------------|-------------------------|------------|
  | 构造参数 | parentNode | TreeNode*               | 父节点指针      |
  | 构造参数 | def        | LeafTreeNodeDefinition& | 节点定义       |
  | 备注   | -          | -                       | 无额外参数，纯虚基类 |

  ---
  4. FSMTreeNode (有限状态机)

  文件: aiFSMNode.h:27

  | 参数类型  | 参数名                      | 类型                       | 说明          |
  |-------|--------------------------|--------------------------|-------------|
  | 构造参数  | parentNode               | TreeNode*                | 父节点指针       |
  | 构造参数  | def                      | FSMTreeNodeDefinition&   | 节点定义        |
  | 成员变量  | m_states                 | DynArray<FSMState*>      | 状态列表（最多32个） |
  | 成员变量  | m_transitions            | DynArray<FSMTransition*> | 转换列表（最多48个） |
  | 成员变量  | m_initialState           | FSMState*                | 初始状态指针      |
  | 运行时变量 | i_currentState           | FSMStateWrapper          | 当前状态        |
  | 运行时变量 | i_activatedStates        | Uint8                    | 已激活状态索引     |
  | 运行时变量 | i_delayedStateActivation | Bool                     | 是否延迟状态激活    |
  | 运行时变量 | i_isCompleted            | Bool                     | 是否已完成       |
  | 配置参数  | m_states                 | DynArray<THandle>        | 状态定义列表      |
  | 配置参数  | m_transitions            | DynArray<THandle>        | 转换定义列表      |
  | 配置参数  | m_initialState           | THandle                  | 初始状态定义      |

  ---
  二、控制流节点

  5. SequenceTreeNode (顺序节点)

  文件: aiSequenceNode.h:17

  | 参数类型  | 参数名                      | 类型                          | 说明         |
  |-------|--------------------------|-----------------------------|------------|
  | 构造参数  | parentNode               | TreeNode*                   | 父节点指针      |
  | 构造参数  | def                      | SequenceTreeNodeDefinition& | 节点定义       |
  | 运行时变量 | i_activeBranch           | Uint8 (ChildIdx)            | 当前执行的子节点索引 |
  | 继承    | (CompositeTreeNode的所有参数) | -                           | 继承父类参数     |

  ---
  6. SelectorTreeNode (选择器节点)

  文件: aiSelectorNode.h:17

  | 参数类型  | 参数名                      | 类型                          | 说明          |
  |-------|--------------------------|-----------------------------|-------------|
  | 构造参数  | parentNode               | TreeNode*                   | 父节点指针       |
  | 构造参数  | def                      | SelectorTreeNodeDefinition& | 节点定义        |
  | 运行时变量 | i_activeBranch           | Uint8 (ChildIdx)            | 当前尝试的子节点索引  |
  | 运行时变量 | i_cut                    | Bool                        | 是否剪枝（cut操作） |
  | 继承    | (CompositeTreeNode的所有参数) | -                           | 继承父类参数      |

  ---
  7. ParallelNode (并行节点)

  文件: aiParallelNode.h:15

  | 参数类型  | 参数名                      | 类型                      | 说明         |
  |-------|--------------------------|-------------------------|------------|
  | 构造参数  | parentNode               | TreeNode*               | 父节点指针      |
  | 构造参数  | definition               | ParallelNodeDefinition& | 节点定义       |
  | 成员变量  | m_waitFor                | WaitFor                 | 等待策略枚举     |
  | 运行时变量 | i_activationCounter      | Uint32                  | 激活计数器（防重入） |
  | 运行时变量 | i_childStates            | Array                   | 子节点状态数组    |
  | 运行时变量 | i_isActive               | Bool                    | 节点是否激活     |
  | 运行时变量 | i_completionStatus       | CompletionStatus        | 完成状态       |
  | 配置参数  | m_waitFor                | WaitFor                 | 等待策略配置     |
  | 继承    | (CompositeTreeNode的所有参数) | -                       | 继承父类参数     |

  WaitFor 枚举值:
  - LeftChild - 等待左子节点
  - RightChild - 等待右子节点
  - AllChildren / BothChildren - 等待所有子节点
  - AnyChild - 等待任意子节点

  ChildState 枚举值:
  - Inactive - 未激活
  - Active - 激活中
  - Completed - 已完成

  ---
  8. IfElseNode (条件分支节点)

  文件: aiIfElseNode.h:21

  | 参数类型  | 参数名                      | 类型                    | 说明                            |
  |-------|--------------------------|-----------------------|-------------------------------|
  | 构造参数  | parent                   | TreeNode*             | 父节点指针                         |
  | 构造参数  | definition               | IfElseNodeDefinition& | 节点定义                          |
  | 成员变量  | m_condition              | PassiveExpression*    | 被动条件表达式                       |
  | 成员变量  | m_updateHelper           | UpdateOnceHelper      | 更新辅助对象                        |
  | 运行时变量 | i_activeBranch           | Uint8                 | 当前激活分支（0=true, 1=false, 2=无效） |
  | 运行时变量 | i_isCompleted            | Bool                  | 是否已完成                         |
  | 配置参数  | m_condition              | ExpressionDefPtr      | 条件表达式定义                       |
  | 继承    | (CompositeTreeNode的所有参数) | -                     | 继承父类参数                        |

  分支索引常量:
  - c_trueBranchIdx = 0 - True分支
  - c_falseBranchIdx = 1 - False分支
  - c_invalidBranch = 2 - 无效分支

  ---
  三、装饰器节点

  9. TaskNode (任务节点基类)

  文件: aiTaskNode.h:23

  | 参数类型 | 参数名                  | 类型                  | 说明     |
  |------|----------------------|---------------------|--------|
  | 构造参数 | parentNode           | TreeNode*           | 父节点指针  |
  | 构造参数 | definition           | TaskNodeDefinition& | 节点定义   |
  | 成员变量 | m_task               | Task*               | 任务实例指针 |
  | 配置参数 | m_task               | THandle             | 任务定义   |
  | 继承   | (DecoratorNode的所有参数) | -                   | 继承父类参数 |

  ---
  10. ConditionNode (条件节点基类)

  文件: aiConditionNode.h:23

  | 参数类型 | 参数名                  | 类型                       | 说明                  |
  |------|----------------------|--------------------------|---------------------|
  | 构造参数 | parentNode           | TreeNode*                | 父节点指针               |
  | 构造参数 | definition           | ConditionNodeDefinition& | 节点定义                |
  | 成员变量 | m_condition          | Condition*               | 条件实例指针              |
  | 成员变量 | m_resultIfFailed     | CompletionStatus         | 条件失败时的结果            |
  | 配置参数 | m_condition          | THandle                  | 条件定义                |
  | 配置参数 | m_resultIfFailed     | CompletionStatus         | 失败结果配置 (默认=FAILURE) |
  | 继承   | (DecoratorNode的所有参数) | -                        | 继承父类参数              |

  ---
  11. EventHandlerNode (事件处理节点)

  文件: aiEventHandlerNode.h:18

  | 参数类型 | 参数名                  | 类型                          | 说明      |
  |------|----------------------|-----------------------------|---------|
  | 构造参数 | parentNode           | TreeNode*                   | 父节点指针   |
  | 构造参数 | definition           | EventHandlerNodeDefinition& | 节点定义    |
  | 成员变量 | m_resolver           | EventResolver*              | 事件解析器实例 |
  | 成员变量 | m_eventName          | CName                       | 监听的事件名称 |
  | 配置参数 | m_resolver           | THandle                     | 事件解析器定义 |
  | 配置参数 | m_eventName          | CName                       | 事件名称配置  |
  | 继承   | (DecoratorNode的所有参数) | -                           | 继承父类参数  |

  ---
  12. LimiterNode (限制器节点)

  文件: aiLimiterNode.h:13

  | 参数类型  | 参数名                               | 类型                     | 说明                 |
  |-------|-----------------------------------|------------------------|--------------------|
  | 构造参数  | parent                            | TreeNode*              | 父节点指针              |
  | 构造参数  | definition                        | LimiterNodeDefinition& | 节点定义               |
  | 成员变量  | m_limit                           | Uint32                 | 每帧激活次数限制           |
  | 成员变量  | m_delayChild                      | Bool                   | 是否延迟子节点激活          |
  | 成员变量  | m_delayChildIfAttaching           | Bool                   | 附加时是否延迟激活          |
  | 运行时变量 | i_lastActivation                  | UpdateId::Type         | 上次激活时间ID           |
  | 运行时变量 | i_counter                         | Uint32                 | 激活计数器              |
  | 运行时变量 | i_isEnqueued                      | Bool                   | 是否已入队              |
  | 配置参数  | m_activationLimitPerFrame         | Uint32                 | 每帧激活限制 (默认=1)      |
  | 配置参数  | m_delayChildActivation            | Bool                   | 延迟子节点配置 (默认=false) |
  | 配置参数  | m_delayChildActivationIfAttaching | Bool                   | 附加延迟配置 (默认=false)  |
  | 继承    | (DecoratorNode的所有参数)              | -                      | 继承父类参数             |

  ---
  13. MaybeNode (概率节点)

  文件: aiMaybeNode.h:24

  | 参数类型 | 参数名                  | 类型                          | 说明                   |
  |------|----------------------|-----------------------------|----------------------|
  | 成员变量 | m_boundedActivator   | BoundedActivationsScheduler | 有界激活调度器              |
  | 成员变量 | m_onChildSuccess     | MaybeNodeAction             | 子节点成功时的动作            |
  | 成员变量 | m_onChildFailure     | MaybeNodeAction             | 子节点失败时的动作            |
  | 配置参数 | m_onChildSuccess     | MaybeNodeAction             | 成功时动作配置 (默认=Succeed) |
  | 配置参数 | m_onChildFailure     | MaybeNodeAction             | 失败时动作配置 (默认=Succeed) |
  | 继承   | (DecoratorNode的所有参数) | -                           | 继承父类参数               |

  MaybeNodeAction 枚举值:
  - Succeed - 直接返回成功
  - Fail - 直接返回失败
  - RepeatChild - 重复执行子节点

  ---
  14. PassiveConditionNode (被动条件节点)

  文件: aiPassiveConditionNode.h:21

  | 参数类型  | 参数名                  | 类型                                    | 说明                |
  |-------|----------------------|---------------------------------------|-------------------|
  | 构造参数  | parentNode           | TreeNode*                             | 父节点指针             |
  | 构造参数  | def                  | PassiveConditionNodeDefinition&       | 节点定义              |
  | 成员变量  | m_definition         | const PassiveConditionNodeDefinition& | 定义引用              |
  | 成员变量  | m_condition          | PassiveCondition*                     | 被动条件实例            |
  | 成员变量  | m_updateHelper       | UpdateOnceHelper                      | 更新辅助对象            |
  | 运行时变量 | i_lastValue          | Bool                                  | 上次条件值             |
  | 运行时变量 | i_callbackToken      | AsyncCallbackToken                    | 异步回调令牌            |
  | 配置参数  | m_condition          | THandle                               | 被动条件定义            |
  | 配置参数  | m_resultIfFailed     | CompletionStatus                      | 失败结果 (默认=FAILURE) |
  | 继承    | (DecoratorNode的所有参数) | -                                     | 继承父类参数            |

  ---
  15. CommandHandlerNode (命令处理节点)

  文件: aiCommandHandlerNode.h:20

  | 参数类型  | 参数名                       | 类型                            | 说明                |
  |-------|---------------------------|-------------------------------|-------------------|
  | 构造参数  | parent                    | TreeNode*                     | 父节点指针             |
  | 构造参数  | definition                | CommandHandlerNodeDefinition& | 节点定义              |
  | 成员变量  | m_commandName             | CName                         | 命令名称              |
  | 成员变量  | m_useInheritance          | Bool                          | 是否使用继承查找命令        |
  | 成员变量  | m_contextMask             | TCommandContextMask           | 命令上下文掩码           |
  | 成员变量  | m_commandOut              | THandle                       | 命令输出映射            |
  | 成员变量  | m_runningSignal           | CName                         | 运行信号名称            |
  | 成员变量  | m_waitForCommand          | Bool                          | 是否等待命令            |
  | 成员变量  | m_retryIfCommandEnqueued  | Bool                          | 命令入队时是否重试         |
  | 成员变量  | m_resultIfNoCommand       | CompletionStatus              | 无命令时的结果           |
  | 成员变量  | m_resultIfChildFailed     | CompletionStatus              | 子节点失败时的结果         |
  | 运行时变量 | i_command                 | CommandPtr                    | 当前命令指针            |
  | 运行时变量 | i_callbackToken           | AsyncCallbackToken            | 异步回调令牌            |
  | 运行时变量 | i_commandCompletionStatus | CommandState                  | 命令完成状态            |
  | 运行时变量 | i_signalSet               | Bool                          | 信号是否已设置           |
  | 运行时变量 | i_isWaiting               | Bool                          | 是否正在等待            |
  | 运行时变量 | i_isEnqueuedForUpdate     | Bool                          | 是否已入队更新           |
  | 运行时变量 | i_callbackId              | CommandQueue::TCallbackId     | 回调ID              |
  | 配置参数  | m_commandName             | CName                         | 命令名称配置            |
  | 配置参数  | m_useInheritance          | Bool                          | 继承配置              |
  | 配置参数  | m_contexts                | DynArrayCommandContexts::Type | 上下文列表             |
  | 配置参数  | m_commandOut              | THandle                       | 命令输出配置            |
  | 配置参数  | m_runningSignal           | CName                         | 运行信号配置            |
  | 配置参数  | m_waitForCommand          | Bool                          | 等待命令配置 (默认=false) |
  | 配置参数  | m_retryIfCommandEnqueued  | Bool                          | 重试配置 (默认=false)   |
  | 配置参数  | m_resultIfNoCommand       | CompletionStatus              | 无命令结果配置           |
  | 配置参数  | m_resultIfChildFailed     | CompletionStatus              | 子节点失败结果配置         |
  | 继承    | (DecoratorNode的所有参数)      | -                             | 继承父类参数            |

  ---
  四、任务/条件节点

  16. InstantTaskNode (即时任务节点)

  文件: aiInstantTaskNode.h:17

  | 参数类型 | 参数名             | 类型                         | 说明            |
  |------|-----------------|----------------------------|---------------|
  | 构造参数 | parentNode      | TreeNode*                  | 父节点指针         |
  | 构造参数 | definition      | InstantTaskNodeDefinition& | 节点定义          |
  | 继承   | (TaskNode的所有参数) | -                          | 继承父类参数        |
  | 备注   | -               | -                          | 无额外参数，激活时立即执行 |

  ---
  17. MonitorTaskNode (监控任务节点)

  文件: aiMonitorTaskNode.h:17

  | 参数类型  | 参数名             | 类型                         | 说明     |
  |-------|-----------------|----------------------------|--------|
  | 构造参数  | parentNode      | TreeNode*                  | 父节点指针  |
  | 构造参数  | definition      | MonitorTaskNodeDefinition& | 节点定义   |
  | 成员变量  | m_timeout       | Float                      | 超时时间配置 |
  | 运行时变量 | i_timeout       | Float                      | 剩余超时时间 |
  | 运行时变量 | i_firstUpdate   | Bool                       | 是否首次更新 |
  | 配置参数  | m_timeout       | Float                      | 超时时间配置 |
  | 继承    | (TaskNode的所有参数) | -                          | 继承父类参数 |

  ---
  18. InstantConditionNode (即时条件节点)

  文件: aiInstantConditionNode.h:15

  | 参数类型 | 参数名                  | 类型                              | 说明            |
  |------|----------------------|---------------------------------|---------------|
  | 构造参数 | parentNode           | TreeNode*                       | 父节点指针         |
  | 构造参数 | definition           | InstantConditionNodeDefinition& | 节点定义          |
  | 继承   | (ConditionNode的所有参数) | -                               | 继承父类参数        |
  | 备注   | -                    | -                               | 无额外参数，激活时检查一次 |
  | 编辑器  | 缩写                   | "IC"                            | 编辑器显示缩写       |

  ---
  19. MonitorConditionNode (监控条件节点)

  文件: aiMonitorConditionNode.h:15

  | 参数类型  | 参数名                  | 类型                              | 说明      |
  |-------|----------------------|---------------------------------|---------|
  | 构造参数  | parentNode           | TreeNode*                       | 父节点指针   |
  | 构造参数  | definition           | MonitorConditionNodeDefinition& | 节点定义    |
  | 成员变量  | m_timeout            | Float                           | 超时时间配置  |
  | 运行时变量 | i_timeout            | Float                           | 剩余超时时间  |
  | 运行时变量 | i_firstActivation    | Float                           | 首次激活时间  |
  | 配置参数  | m_timeout            | Float                           | 超时时间配置  |
  | 继承    | (ConditionNode的所有参数) | -                               | 继承父类参数  |
  | 编辑器   | 缩写                   | "MC"                            | 编辑器显示缩写 |

  ---
  五、叶子节点

  20. SucceederNode (成功节点)

  文件: aiCompleteNodes.h:15

  | 参数类型 | 参数名                 | 类型                       | 说明        |
  |------|---------------------|--------------------------|-----------|
  | 构造参数 | parentNode          | TreeNode*                | 父节点指针     |
  | 构造参数 | def                 | SucceederNodeDefinition& | 节点定义      |
  | 继承   | (LeafTreeNode的所有参数) | -                        | 继承父类参数    |
  | 备注   | -                   | -                        | 激活后立即返回成功 |

  ---
  21. FailerNode (失败节点)

  文件: aiCompleteNodes.h:32

  | 参数类型 | 参数名                 | 类型                    | 说明        |
  |------|---------------------|-----------------------|-----------|
  | 构造参数 | parentNode          | TreeNode*             | 父节点指针     |
  | 构造参数 | def                 | FailerNodeDefinition& | 节点定义      |
  | 继承   | (LeafTreeNode的所有参数) | -                     | 继承父类参数    |
  | 备注   | -                   | -                     | 激活后立即返回失败 |

  ---
  六、状态机节点

  22. FSMState (FSM状态节点)

  文件: aiFSMStateNode.h:31

  | 参数类型 | 参数名                | 类型                              | 说明            |
  |------|--------------------|---------------------------------|---------------|
  | 构造参数 | parentFSM          | TreeNode*                       | 父FSM节点指针      |
  | 构造参数 | def                | FSMStateDefinition&             | 状态定义          |
  | 成员变量 | m_outTransitions   | StaticArray<FSMTransition*, 16> | 出口转换数组（最多16个） |
  | 成员变量 | m_behaviorRoot     | TreeNode*                       | 状态内的行为树根节点    |
  | 成员变量 | m_definition       | const FSMStateDefinition&       | 定义引用          |
  | 配置参数 | m_behaviorRoot     | THandle                         | 行为树根节点定义      |
  | 配置参数 | m_isInitial        | Bool                            | 是否为初始状态       |
  | 配置参数 | m_isExit           | Bool                            | 是否为退出状态       |
  | 配置参数 | m_completionStatus | StateCompletionStatus           | 完成时的状态        |

  StateCompletionStatus 枚举值:
  - ForwardBehaviorStatus - 转发行为状态
  - Failure - 失败
  - Success - 成功

  TransitionIdx 常量:
  - invalidTransition = -1 - 无效转换
  - maxTransitionsSize = 16 - 最大转换数

  ---
  七、嵌套树节点

  23. NestedTree (嵌套树基类)

  文件: aiIncludedTreeNode.h:24

  | 参数类型  | 参数名                       | 类型                    | 说明                |
  |-------|---------------------------|-----------------------|-------------------|
  | 构造参数  | parentNode                | TreeNode*             | 父节点指针             |
  | 构造参数  | definition                | NestedTreeDefinition& | 节点定义              |
  | 成员变量  | m_lateInitialization      | Bool                  | 是否延迟初始化           |
  | 运行时变量 | i_treeInitialized         | Bool                  | 树是否已初始化           |
  | 运行时变量 | i_hasInitializationEvents | Bool                  | 是否有初始化事件          |
  | 运行时变量 | i_treeInstance            | Instance              | 嵌套树实例             |
  | 运行时变量 | i_updatableActivated      | Bool                  | 可更新对象是否激活         |
  | 配置参数  | m_lateInitialization      | Bool                  | 延迟初始化配置 (默认=true) |
  | 配置参数  | m_initializeOnEvent       | DynArray              | 初始化触发事件列表         |

  ---
  24. IncludedTree (引用树)

  文件: aiIncludedTreeNode.h:79

  | 参数类型 | 参数名               | 类型                            | 说明      |
  |------|-------------------|-------------------------------|---------|
  | 构造参数 | parentNode        | TreeNode*                     | 父节点指针   |
  | 构造参数 | definition        | IncludedTreeDefinition&       | 节点定义    |
  | 成员变量 | m_definition      | const IncludedTreeDefinition& | 定义引用    |
  | 配置参数 | m_treeReference   | THandle                       | 树引用参数映射 |
  | 继承   | (NestedTree的所有参数) | -                             | 继承父类参数  |

  ---
  25. Subtree (静态子树)

  文件: aiIncludedTreeNode.h:114

  | 参数类型 | 参数名               | 类型                       | 说明     |
  |------|-------------------|--------------------------|--------|
  | 构造参数 | parentNode        | TreeNode*                | 父节点指针  |
  | 构造参数 | definition        | SubtreeDefinition&       | 节点定义   |
  | 成员变量 | m_definition      | const SubtreeDefinition& | 定义引用   |
  | 配置参数 | m_tree            | THandle                  | 子树定义   |
  | 继承   | (NestedTree的所有参数) | -                        | 继承父类参数 |

  ---
  📊 通用参数说明

  所有 TreeNode 基类共有的参数:

  | 参数类型  | 参数名                | 类型                        | 说明              |
  |-------|--------------------|---------------------------|-----------------|
  | 成员变量  | m_definition       | const TreeNodeDefinition& | 节点定义引用          |
  | 成员变量  | m_parentNode       | const TreeNode*           | 父节点指针           |
  | 运行时变量 | i_debugContext     | game::DebugContextPtr     | 调试上下文（DEBUG模式）  |
  | 运行时变量 | i_debugLogScope    | AI::DebugLogScope         | 调试日志范围（DEBUG模式） |
  | 运行时变量 | i_activationStatus | ActivationStatus          | 激活状态（DEBUG模式）   |
  | 调试变量  | m_debugNodeName    | CName                     | 调试节点名称（DEBUG模式） |
  | 调试变量  | m_debugPath        | String                    | 调试路径（DEBUG模式）   |

  CompletionStatus 枚举（通用）:

  - FAILURE - 失败
  - SUCCESS - 成功

  ActivationStatus 枚举（通用）:

  - NOT_ACTIVATED - 未激活
  - ACTIVATING - 激活中
  - ACTIVATED - 已激活
  - DEACTIVATING - 停用中

  ---
  🎯 参数命名规范

  - m_ 前缀: 成员变量（配置参数）
  - i_ 前缀: 运行时实例变量（TInstanceVar）
  - c_ 前缀: 编译时常量
  - 无前缀: 临时变量或局部变量
