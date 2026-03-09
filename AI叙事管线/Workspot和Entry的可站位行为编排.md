● 如果把 ExecutionStream 理解成“时序执行层”，那 Workspot / Entry 更像“可占位的行为编排层”。

  核心不是“播放一个动画”，而是：

  - 某个角色进入一个可交互位置
  - 在这个位置里按组合规则选择/切换行为
  - 在需要时与别的角色或槽位同步
  - 最后以合适的方式退出

  所以它比普通 Graph 节点更接近 AI 叙事真正需要的“可执行行为单元”。

  ---
  1. Workspot 真正是什么

  从 dev/src/common/gameWorkspots/src/workspotTreeItems.cpp 和 dev/src/common/gameWorkspots/src/workspotResource.cpp 看，Workspot
  不是单个动画资源，而是一个带空间锚点、进入/退出规则、同步槽位、行为组合树的占位行为系统。

  它至少同时包含四层含义：

  第一层：空间占位

  角色不是“做一个动作”这么简单，而是要先回答：

  - 去哪里做
  - 从哪个方向进入
  - 站位/坐位/靠位是什么
  - 退出时往哪里走

  这就是 GetEntryVectors、GetExitVectors、GetFastExitVectors 这一类接口重要的原因。
  它说明 Workspot 先天就是叙事动作的空间落点系统，不是纯时间轴事件。

  第二层：行为容器

  Workspot 内部不是一条平的 clip 列表，而是一棵组合树：

  - Sequence
  - Selector
  - RandomList
  - 各类 AnimClip
  - EntryAnim
  - ExitAnim
  - TagNode
  - 以及同步/反应类节点

  这说明 Workspot 的本质不是“动画集合”，而是一个可组合的行为语法树。

  第三层：同步系统

  从 SyncAnimClip、同步 slot、GetSyncWorkspotTransform、GetSyncEntryIdForSlotName 这些点看，Workspot 还内建了：

  - 多角色同步
  - 位置/朝向对齐
  - 某个槽位该由谁占据
  - 哪个 entry 对应哪个协作位置

  这对叙事非常关键，因为很多戏剧性动作本来就不是单人行为，而是：

  - 握手
  - 递杯子
  - 一人坐下另一人跟坐
  - 对峙时的空间相持
  - 同步看向某个目标

  第四层：持续状态

  m_idleAnim 这类字段很关键。
  它说明一个角色进入 Workspot 后，不是只触发一次行为，而是进入一个持续占位状态：

  - 入场
  - 保持姿态 / idle
  - 响应上下文
  - 切换子行为
  - 再退出

  这比普通 SceneGraph 的“事件节点”更接近真实表演。

  ---
  2. Entry 真正是什么

  Entry 不是“条目”这么简单。
  在这个体系里，IEntry 更像可执行行为构件。

  也就是说，Entry 是 Workspot 行为树里的基本构造块。它可以是：

  - 一个动画片段
  - 一个进入动作
  - 一个退出动作
  - 一个同步动作
  - 一个暂停片段
  - 一个标签过滤节点
  - 一个组合节点（顺序/选择/随机）

  所以 Entry 的意义不是“数据项”，而是：

  “角色在某个占位上下文中，允许执行的一段结构化行为表达式”

  这点非常重要。

  因为如果你从 AI 叙事角度看，一个最小单元不该只是“台词文本”，也不该直接是 SectionNode 这种较大的段落容器；
  更接近底层、真正能落地执行的，恰恰是这种 Entry-like 的行为原子/行为片段。

  ---
  3. 为什么 Entry 比普通 Graph Node 更接近 AI 原生叙事

  你之前一直在追问一个本质问题：

  AI 原生叙事到底操作什么？
  如果不以 Graph 为中心，那它到底编排什么？

  Entry 给了一个很好的答案：
  AI 不一定直接编排 Graph，它可以先编排行为片段的组合。

  Graph 的问题是它太偏“编辑器结构”：

  - 节点
  - 连线
  - 输入输出 pin
  - 可视化流程

  但叙事真正关心的常常不是这个，而是：

  - 谁在什么位置
  - 以什么姿态进入场景
  - 当前处于什么持续状态
  - 能切到哪些反应
  - 与谁同步
  - 什么情况下退出

  这些都更像 Workspot/Entry 的语言，而不是 Graph 的语言。

  所以我会把它们区分成两层：

  Graph-first

  “先画流程图，再想里面塞什么行为。”

  Entry-first

  “先定义可执行行为构件和组合规则，再把它们投影成图。”

  对 AI 原生叙事来说，后者更自然。

  ---
  4. 这对 AI 叙事产品意味着什么

  如果把你想做的系统抽象出来，我认为最重要的不是“AI 帮你画 Graph”，而是“AI 帮你生成行为组合树”。

  也就是：

  ---
  层 1：叙事意图层

  AI 理解：

  - 当前戏剧目标是什么
  - 角色关系是什么
  - 情绪状态是什么
  - 这一拍是对抗、试探、安抚还是压迫

  例如：

  - Hanako 要建立控制感
  - V 要保持警惕但不失礼貌

  这还是语义层。

  ---
  层 2：行为编排层

  AI 把语义翻译成 Entry 组合：

  - Hanako 进入座位
  - 坐下后保持稳定、克制的 idle
  - 在 V 坐定前不开始关键台词
  - V 迟疑后再落座
  - 对话中某个节点切到 lean-back 或 hand gesture
  - 如果玩家打断，则快速退出或切反应

  这一层已经不再是“文本生成”，而是 可执行表演结构生成。

  ---
  层 3：时序层

  再往下才是你前面特别重视的 ExecutionStream：

  - 哪个 Entry 何时开始
  - 哪些并行
  - 哪些依赖信号
  - 哪些是同步点
  - 哪些中断时需要 cancel / fast exit

  也就是说：

  Entry 决定“演什么、怎么演”
  ExecutionStream 决定“什么时候演、按什么依赖演”

  这两个层是互补的，不是替代关系。

  ---
  5. 为什么这比“纯台词驱动”强很多

  如果 AI 只拿到台词文本，它最多能做：

  - 分句
  - 推断语气
  - 生成一些节奏建议

  但它很难稳定落到可执行层。

  而有了 Workspot / Entry 这种中间层，AI 就能把文本翻译成更具体的表演约束：

  - 这句台词前角色要先坐稳
  - 这句不是边走边说，而是坐定后低声说
  - 这段沉默不是“空白”，而是保持 eye contact 的 idle
  - 这句打断后要走 fast exit 而不是普通 exit
  - 两人都 seated 后才允许服务员倒水

  这才是真正的“叙事意图 → 可执行行为”。

  ---
  6. 你可以把它理解成一个更合理的 AI 原生叙事 IR

  如果我们把你想做的产品抽象成 IR，我现在更倾向于这样分：

  A. Beat / Section 层

  较大的段落单位，类似 dev/src/common/gameSceneSystem/src/scnsSectionNode.cpp 里的 section 容器。

  它负责：

  - 段落目标
  - 参与者集合
  - 时长参考
  - 对外触发/取消接口

  B. Entry / Behavior 层

  真正可执行的表演构件层，参考 dev/src/common/gameWorkspots/src/workspotTreeItems.cpp。

  它负责：

  - 进入
  - 占位
  - idle
  - 反应
  - 切换
  - 同步
  - 退出
  - 组合规则

  C. Timing / Dependency 层

  参考 dev/src/common/gameSceneCore/include/scnsExecutionStream.h。

  它负责：

  - start / conclusion
  - signal / stimulation
  - cancel / control
  - future / inprogress / historical 的运行态迁移

  ---
  7. 所以 Graph 在未来应该是什么角色

  到这里就能更明确地回答你前面那个问题了：

  Graph 以后更像生成的可视化，而不是核心操作对象。

  因为真正的核心对象应该是：

  - Beat
  - Entry 组合树
  - 时序依赖
  - 空间占位
  - 同步关系
  - 中断策略

  Graph 只是把这些投影出来，方便：

  - 调试
  - 审核
  - 局部修改
  - 与传统工具兼容

  而不是让作者从零开始手画所有东西。

  ---
  8. 最关键的一句结论

  如果说 SectionNode 是“叙事段落单元”，
  那么 Entry 更像“表演行为单元”；
  如果说 ExecutionStream 是“时序执行单元”，
  那么 Workspot 就是“空间化、可同步、可组合的行为承载单元”。

  这三个层次合起来，才接近一个真正 AI 原生的叙事系统：

  - Section：讲这一小段戏要干什么
  - Entry：这一小段戏里角色具体怎么演
  - ExecutionStream：这些表演如何在时间上被调度执行

  所以你抓 Workspot / Entry 这个方向是对的。
  它比单纯讨论 Graph，更接近“AI 如何真正控制一段戏的可执行结构”。

  下一步如果继续往产品形态推，我会建议直接把它抽象成一句话：

  不是让 AI 画剧情图，而是让 AI 生成“角色占位 + 行为组合 + 时序依赖”的可执行叙事编排。


✻ Crunched for 8m 53s

  1 tasks (0 done, 1 in progress, 0 open)
  ◼ 分析 Workspot 与 Entry