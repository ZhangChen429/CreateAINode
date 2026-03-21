
# 关于 AI 叙事产品中 Graph 的定位与落地路径

## 核心结论
对 AI 叙事产品来说，Graph 不是第一输入界面，但不是可有可无。

- 对人类创作者：Graph 可以不是必需入口
- 对系统内部表示：Graph / DAG / 依赖结构 几乎不可缺

Graph 可以从“作者直接操作对象”降级为“系统内部编排结果”，但很难被完全消灭。

---

## 一、为什么你会觉得 Graph 可有可无
你真正想做的不是让策划继续手动画节点，而是：
1. 让策划说出意图
2. 让 AI 自动拆成时序、依赖、等待、分支
3. 最后系统再决定怎么组织执行

这个判断是对的。从产品交互上，你完全可以不把 Graph 当第一界面。用户更愿意写：
- “Hanako 先坐下，再示意 V 坐下，然后开始正式对话”
- “如果玩家离开视线，打断这一段”
- “这一段节奏慢一点”

而不是自己画：`Start -> SitHanako -> SitV -> Wait -> Dialogue`。所以你反感“Graph-first”，这个是合理的。

---

## 二、但为什么 Graph 又不能真的消失
一旦进入可执行层，系统一定要回答这些问题：
- 谁依赖谁
- 哪些并行
- 哪些串行
- 哪些是等待汇合点
- 哪些是条件分支
- 哪些是中断跳转
- 哪些会回流
- 哪些不能同时发生

这些关系本质上就是图结构问题。哪怕你不叫它 Graph，内部也还是：
- DAG
- 状态迁移图
- 依赖网
- 时序网络

只是名字不同。所以 Graph 不是产品核心心智，但它是执行编译层的骨架。

---

## 三、如果你想先落地“小单元”，最好的切入点是什么
不要一上来做完整 Scene Graph 编辑器，先做 3 个最小单元能力。

### 1. 先做“叙事原子”而不是“完整 section”
先定义最小可执行原子，5 种就够：
- Speak
- MoveTo/TakePosition
- Gesture/LookAt
- Wait
- Signal

这一步的目标是定义 AI 最终要产出的最小执行语义。

例子：
```json
{
  "type": "Speak",
  "actor": "Hanako",
  "content": "V先生，感谢你前来。",
  "tone": "formal",
  "startPolicy": "after:both_seated"
}
```

这一步非常关键，因为如果原子定义不稳，后面 section、timeline、导出全都会漂。

### 2. 再做“依赖关系推断”
不先画 graph，而是先让 AI 把一句话拆成：
- 原子列表
- 依赖列表

例如输入：
> Hanako 先坐下，V 跟着坐，等两人稳定后开始对话。

输出：
```json
{
  "atoms": [
    "HanakoSit",
    "VSit",
    "WaitBothStable",
    "HanakoSpeak"
  ],
  "dependencies": [
    ["HanakoSit", "WaitBothStable"],
    ["VSit", "WaitBothStable"],
    ["WaitBothStable", "HanakoSpeak"]
  ]
}
```

注意这里已经是 graph 了，但不是用户手画的 graph，而是 AI 推出来的 graph。这就是你的核心产品思路。

### 3. 再做“时间建议器”
当原子和依赖有了，再做：
- start offset
- duration
- pause suggestion
- overlap suggestion

例如：
```json
{
  "HanakoSit": { "start": 0, "duration": 2.8 },
  "VSit": { "start": 1.2, "duration": 2.4 },
  "HanakoSpeak": { "start": 4.0, "duration": 3.6 }
}
```

到这一步，你就已经有一个很像 ExecutionStream 的雏形了。

---

## 四、落地路线选择

### 路线 A：先做“无图输入，有图内核”（最推荐）
1.  **意图输入**：输入是自然语言或结构化表单（场景目标、参与角色、动作描述、氛围、节奏、中断要求）
2.  **中间层 IR**：定义你自己的结构（NarrativeAtom、Dependency、Section、InterruptPolicy、TempoHint）
3.  **编译器**：把自然语言转成原子、依赖、时间建议
4.  **可视化**：最后再显示成 list、timeline、graph（可选）

这样 graph 就变成结果视图而不是输入负担。

### 路线 B：先做“Section 草案生成器”
如果你觉得原子太细，也可以从 section 入手。输入一句话，输出：
- section goal
- participants
- events
- duration
- dependencies
- interrupt policy

这是比完整 graph 更小、更稳的单元。

### 路线 C：先做“时序分析器”
如果你连生成都觉得太早，就先做分析：
- 输入一个已有 flow / graph / sequence
- 输出：哪些等待是冗余、哪些依赖缺失、哪些节奏过紧、哪些地方可以并行

这个最容易落地，也最容易看到价值。

---

## 五、与 FlowGraph 插件的关系
### 不要把你的产品定义成“更好的 FlowGraph”
那会把你锁死在：
- 节点编辑体验
- 连线效率
- UI 优化
- graph 操作 ergonomics

这不是你真正的差异点。你的差异点应该是：让 Graph 从人工创作结果，变成 AI 编译结果。

所以你不是在和 FlowGraph 拼“画图更爽”，而是在定义：
- 不画图也能生成 graph
- graph 只是解释层和验证层
- 用户操作对象是意图，不是节点

### 但你仍然可以借 FlowGraph 做过渡
虽然不建议把产品本质做成 FlowGraph 增强版，但可以把它当一个过渡宿主：
- 输入自然语言
- AI 生成原子和依赖
- 自动落到 FlowGraph 节点上
- 用户再微调

好处是：
- 不用自己先做整套 graph editor
- 可以快速验证 AI 到 graph 的映射
- 可以借现有生态演示产品能力

策略：产品理念不是 Graph-first，但工程实现可以暂时借 Graph 宿主落地。

---

## 六、最先落地的“小单元”（推荐 3 个）
1.  **Narrative Atom Schema**：先定义最小原子类型（Speak、Move、Pose、Wait、Branch、Signal、CameraCue），这是根基。
2.  **Intent -> Dependency Parser**：把一句导演描述拆成原子、顺序/并行/等待关系，这是 AI 最有价值的一层。
3.  **Timeline Preview**：把 parser 输出变成时间轴预览，这个会立刻让人理解产品价值。

---

## 七、产品的正确世界观
Graph 不是创作起点，而是叙事意图被编译后的结构投影。

所以你不是要“消灭 graph”，而是要：
- 弱化 graph 作为输入工具的地位
- 强化 intent / section / timing 作为主交互层
- 把 graph 留给解释、验证和导出

---

## 八、第一周的最小验证任务
定义一个最小 JSON IR：
```json
{
  "sectionGoal": "",
  "participants": [],
  "atoms": [],
  "dependencies": [],
  "tempo": "",
  "interruptPolicy": {}
}
```

然后先只支持 3 类输入句子：
1.  顺序句：“A 然后 B”
2.  并行句：“A 的同时 B”
3.  等待句：“等 A 和 B 完成后，执行 C”

只要这三种通了，你就已经证明你的核心产品路线了。

---

如果你愿意，我下一步可以直接帮你出这三样里的任意一个：
1. 最小 Narrative Atom / Section IR 设计
2. Intent -> Dependency 的提示词模板
3. 一个不依赖 Graph-first 的 MVP 路线图

你想先看哪一个？


# Graph 在 AI 叙事产品中的定位：从操作手段到可视化结果

## 核心结论
未来 Graph 仍然存在，但更像是叙事意图编译后的可视化结果、调试结果和兼容输出，而不再是唯一的一线创作入口。

这与“MCP 操作 Graph”有本质区别：
- **MCP 操作 Graph**：AI 仍然在“编辑图”，是编辑器自动化。
- **AI 原生叙事**：AI 在“理解意图并编译结构”，Graph 只是结构的一种展示/导出结果。

---

## 一、Graph 的角色转变
Graph 从“第一创作界面”退化成“中间结果、解释视图、调试视图、人工修正视图”。理想流程如下：
1. 人先表达叙事意图
2. AI 解析成结构化叙事单元
3. 编译成依赖/时序结构
4. 系统把它渲染成 Graph / Timeline / Section Tree
5. 人如果需要，再在 Graph 上做局部修正

Graph 不消失，只是地位变化了。

---

## 二、与“MCP 操作 Graph”的核心区别

### 1. 操作对象不同
- **MCP 操作 Graph**
  - 操作对象：`node`、`edge`、`pin`、`property`
  - AI 行为：新建节点、连线、填字段、调位置
  - 本质：自动化编辑器操作

- **AI 原生叙事**
  - 操作对象：`narrative goal`、`beat`、`participant`、`atomic intent`、`dependency`、`tempo`、`interrupt policy`
  - AI 行为：理解“这段想表达什么”、拆成叙事原子、推断依赖、生成时序、再决定要不要投影成 graph
  - 本质：语义编排系统

### 2. Graph 的地位不同
- **MCP 操作 Graph**
  - Graph 是源事实：改 graph = 改真相，graph 是唯一主结构

- **AI 原生叙事**
  - Graph 是派生表示：Intent/IR 才是主结构，graph 是从 IR 渲染出来的一个视图，可以同时有 timeline view / section view / graph view

### 3. 对用户的认知负担不同
- **MCP 操作 Graph**
  - 用户仍需理解：什么节点该连哪里、什么 wait 怎么接、分支怎么合并、cancel 怎么回去，只是 AI 替他做了一部分手工。

- **AI 原生叙事**
  - 用户更关注：这段要达成什么叙事目标、谁参与、节奏是快还是慢、谁等谁、玩家打断后怎么办，这是更高层的创作语言。

### 4. 可替换性不同
- **MCP 操作 Graph**
  - 绑定具体编辑器强，换一个 graph 插件，很多能力要重做。

- **AI 原生叙事**
  - Graph 只是输出层之一，更容易输出到 FlowGraph、UE Blueprint-like graph、自研 SceneGraph，甚至不输出 graph，只输出 timeline/JSON，这就是平台能力。

---

## 三、直观类比
- **MCP 操作 Graph**：像是“让 AI 帮你用 Photoshop 画图层”。
- **AI 原生叙事**：像是“你先说你想表达什么，系统自动生成海报结构，Photoshop 图层只是其中一个导出结果”。

前者是编辑器自动化，后者是语义编译。

---

## 四、MCP 操作 Graph 的价值定位
MCP 操作 Graph 有价值，但定位更适合做：
- 过渡方案
- 工程接入层
- 现有工具兼容层
- 自动修图层

例如：
- 让 AI 在现有 FlowGraph 里自动建 section 节点
- 自动补 wait 节点
- 自动连线
- 自动布局
- 自动修正常见错误

它是“接入既有生产流程”的优秀手段，但不是产品最根本的抽象创新。

---

## 五、理想产品架构（两者结合）
1.  **Intent Layer**
    用户输入：叙事意图、节奏要求、角色关系、中断要求。

2.  **Narrative IR Layer**
    系统内部保存：Section、Atoms、Dependencies、Signals、InterruptPolicy、TempoHints。

3.  **Projection Layer**
    把 IR 投影成：Graph、Timeline、Section list、JSON。

4.  **MCP / Tooling Layer**
    如果需要对接现有编辑器：通过 MCP 把 Graph 写进 FlowGraph / UE 编辑器，通过工具 API 改现有资产。

结论：MCP 操作 Graph 是“落地手段”，不是“产品本体”。

---

## 六、走错方向的风险
如果把产品直接做成“AI 帮你操作 Graph”，会有几个风险：
1.  被锁死在编辑器层，产品创新变成自动摆节点、自动连线、自动填参数，越来越像“高级宏”。
2.  很难形成自己的中间层资产，没有自己的 IR，最后只能依附某个 Graph 系统。
3.  很难支持多种视图，因为真相只有 graph，一切都得从 graph 反推。

---

## 七、产品定义建议
不要定义为：
- “AI FlowGraph”
- “AI 帮你画节点”

而是定义为：
- “叙事意图编译器”
- “Section/Timing 生成器”
- “Narrative Orchestration System”

Graph 在这里是：
- explanation view
- debug view
- compatibility export
- optional manual override surface

---

## 八、一句话总结
- 你的方向是对的：Graph 可以从“主要操作手段”降级为“生成出来的可视化和修正层”。
- 与 MCP 操作 Graph 的区别：MCP 操作 Graph 是 AI 在“编辑图”，而 AI 原生叙事是 AI 在“理解意图并编译结构”，Graph 只是结构的一种展示/导出结果。

---

如果你愿意，我下一步可以继续帮你把这个问题再往前推一步，直接给你：
1. 一张“Intent -> IR -> Graph/Timeline”架构图
2. 一套 Narrative IR 的字段设计
3. 比较“AI操作Graph”和“AI原生叙事系统”的产品路线表

你想先看哪一个？




# Graph 在 AI 叙事产品中的定位：从操作手段到可视化结果

## 核心结论
未来 Graph 仍然存在，但更像是叙事意图编译后的可视化结果、调试结果和兼容输出，而不再是唯一的一线创作入口。

这与“MCP 操作 Graph”有本质区别：
- **MCP 操作 Graph**：AI 仍然在“编辑图”，是编辑器自动化。
- **AI 原生叙事**：AI 在“理解意图并编译结构”，Graph 只是结构的一种展示/导出结果。

---

## 一、Graph 的角色转变
Graph 从“第一创作界面”退化成“中间结果、解释视图、调试视图、人工修正视图”。理想流程如下：
1. 人先表达叙事意图
2. AI 解析成结构化叙事单元
3. 编译成依赖/时序结构
4. 系统把它渲染成 Graph / Timeline / Section Tree
5. 人如果需要，再在 Graph 上做局部修正

Graph 不消失，只是地位变化了。

---

## 二、与“MCP 操作 Graph”的核心区别

### 1. 操作对象不同
- **MCP 操作 Graph**
  - 操作对象：`node`、`edge`、`pin`、`property`
  - AI 行为：新建节点、连线、填字段、调位置
  - 本质：自动化编辑器操作

- **AI 原生叙事**
  - 操作对象：`narrative goal`、`beat`、`participant`、`atomic intent`、`dependency`、`tempo`、`interrupt policy`
  - AI 行为：理解“这段想表达什么”、拆成叙事原子、推断依赖、生成时序、再决定要不要投影成 graph
  - 本质：语义编排系统

### 2. Graph 的地位不同
- **MCP 操作 Graph**
  - Graph 是源事实：改 graph = 改真相，graph 是唯一主结构

- **AI 原生叙事**
  - Graph 是派生表示：Intent/IR 才是主结构，graph 是从 IR 渲染出来的一个视图，可以同时有 timeline view / section view / graph view

### 3. 对用户的认知负担不同
- **MCP 操作 Graph**
  - 用户仍需理解：什么节点该连哪里、什么 wait 怎么接、分支怎么合并、cancel 怎么回去，只是 AI 替他做了一部分手工。

- **AI 原生叙事**
  - 用户更关注：这段要达成什么叙事目标、谁参与、节奏是快还是慢、谁等谁、玩家打断后怎么办，这是更高层的创作语言。

### 4. 可替换性不同
- **MCP 操作 Graph**
  - 绑定具体编辑器强，换一个 graph 插件，很多能力要重做。

- **AI 原生叙事**
  - Graph 只是输出层之一，更容易输出到 FlowGraph、UE Blueprint-like graph、自研 SceneGraph，甚至不输出 graph，只输出 timeline/JSON，这就是平台能力。

---

## 三、直观类比
- **MCP 操作 Graph**：像是“让 AI 帮你用 Photoshop 画图层”。
- **AI 原生叙事**：像是“你先说你想表达什么，系统自动生成海报结构，Photoshop 图层只是其中一个导出结果”。

前者是编辑器自动化，后者是语义编译。

---

## 四、MCP 操作 Graph 的价值定位
MCP 操作 Graph 有价值，但定位更适合做：
- 过渡方案
- 工程接入层
- 现有工具兼容层
- 自动修图层

例如：
- 让 AI 在现有 FlowGraph 里自动建 section 节点
- 自动补 wait 节点
- 自动连线
- 自动布局
- 自动修正常见错误

它是“接入既有生产流程”的优秀手段，但不是产品最根本的抽象创新。

---

## 五、理想产品架构（两者结合）
1.  **Intent Layer**
    用户输入：叙事意图、节奏要求、角色关系、中断要求。

2.  **Narrative IR Layer**
    系统内部保存：Section、Atoms、Dependencies、Signals、InterruptPolicy、TempoHints。

3.  **Projection Layer**
    把 IR 投影成：Graph、Timeline、Section list、JSON。

4.  **MCP / Tooling Layer**
    如果需要对接现有编辑器：通过 MCP 把 Graph 写进 FlowGraph / UE 编辑器，通过工具 API 改现有资产。

结论：MCP 操作 Graph 是“落地手段”，不是“产品本体”。

---

## 六、走错方向的风险
如果把产品直接做成“AI 帮你操作 Graph”，会有几个风险：
1.  被锁死在编辑器层，产品创新变成自动摆节点、自动连线、自动填参数，越来越像“高级宏”。
2.  很难形成自己的中间层资产，没有自己的 IR，最后只能依附某个 Graph 系统。
3.  很难支持多种视图，因为真相只有 graph，一切都得从 graph 反推。

---

## 七、产品定义建议
不要定义为：
- “AI FlowGraph”
- “AI 帮你画节点”

而是定义为：
- “叙事意图编译器”
- “Section/Timing 生成器”
- “Narrative Orchestration System”

Graph 在这里是：
- explanation view
- debug view
- compatibility export
- optional manual override surface

---

## 八、一句话总结
- 你的方向是对的：Graph 可以从“主要操作手段”降级为“生成出来的可视化和修正层”。
- 与 MCP 操作 Graph 的区别：MCP 操作 Graph 是 AI 在“编辑图”，而 AI 原生叙事是 AI 在“理解意图并编译结构”，Graph 只是结构的一种展示/导出结果。

---

如果你愿意，我下一步可以继续帮你把这个问题再往前推一步，直接给你：
1. 一张“Intent -> IR -> Graph/Timeline”架构图
2. 一套 Narrative IR 的字段设计
3. 比较“AI操作Graph”和“AI原生叙事系统”的产品路线表

你想先看哪一个？


我给你用**最直白、最能落地到代码/策划表/AI提示词**的方式，一次性讲清楚这 7 个东西到底是什么。
不讲虚的，全部对应你做的**叙事时序编排系统**。

---

# 7 个 AI 原生叙事核心概念（人话版）

## 1. narrative goal —— **叙事目标**
**一句话：这段戏到底想干嘛？**

- 不是动作，是**目的**
- 是策划/导演真正想表达的东西

例子：
- 让 Hanako 建立压迫感
- 让玩家意识到被欺骗
- 让两人关系从紧张缓和下来

作用：
AI 靠这个判断**节奏、情绪、镜头、停顿、优先级**。

---

## 2. beat —— **叙事节拍 / 戏核**
**一句话：一小段有意义的情节单位。**

你可以理解为：
**一小段戏 = 一个 beat**

- 不需要节点
- 不需要图
- 就是一段有始有终的小情节

例子：
- Hanako 坐下
- V 犹豫
- 两人对视
- 对话转折

beat 是你未来**Section 的最小组成单元**。

---

## 3. participant —— **参与者**
**一句话：谁在这场戏里？**

- 角色
- 可交互物体
- 相机
- 甚至“氛围”“灯光”都可以算

例子：
- Hanako
- V
- Camera
- Door
- Chair

作用：
AI 自动分配动作、权限、归属、动画、对话。

---

## 4. atomic intent —— **原子意图**
**一句话：不可再拆的最小动作/意图。**

这是你整个系统的**原子基建**。
不能再拆，拆了就无法执行。

标准原子类型（你直接拿去用）：
- Speak（说话）
- LookAt（看向）
- MoveTo（移动到）
- Animate/Pose（姿势）
- Wait（等待）
- Signal（发信号）
- Branch（分支）
- CameraCue（镜头指令）

例子：
```
{
  "type": "Speak",
  "actor": "Hanako",
  "content": "请坐。"
}
```

---

## 5. dependency —— **依赖关系**
**一句话：谁必须等谁？**

这就是你**底层隐形的图（Graph）**，
但用户看不见、不用画。

例子：
- 必须等 Hanako 坐下 → V 才能坐
- 必须等两人都坐好 → 才能开始对话
- 必须等门关上 → 才能进入严肃剧情

表示方式非常简单：
```
"dependencies": [
  ["HanakoSit", "DialogStart"],
  ["VSit", "DialogStart"]
]
```

**这就是 DAG，就是 Graph，但用户不用看见。**

---

## 6. tempo —— **节奏**
**一句话：快还是慢？松还是紧？**

不是具体秒数，是**风格**。

例子：
- slow（慢）
- tight（紧张）
- relaxed（松弛）
- snappy（干脆）
- pause-heavy（多停顿）

AI 根据这个自动生成：
- 时长
- 停顿
- 重叠
- 镜头速度

---

## 7. interrupt policy —— **中断策略**
**一句话：玩家乱搞时怎么办？**

这是叙事游戏最关键的逻辑之一。

例子：
- can_not_interrupt（不可打断）
- interrupt_and_resume（打断后回来继续）
- interrupt_and_cancel（打断直接跳过）
- interrupt_and_branch（打断跳分支）

作用：
AI 自动给每段叙事加**可执行逻辑**，不用策划写节点。

---

# 把它们拼起来，就是你未来的系统
我给你看一段**真实可运行的 AI 原生叙事结构**：

```json
{
  "narrative_goal": "让Hanako主导对话氛围",
  "beat": "Hanako请V坐下",
  "participants": ["Hanako", "V", "Camera"],
  "atomic_intents": [
    {"type": "Animate", "actor": "Hanako", "action": "Sit"},
    {"type": "Animate", "actor": "V", "action": "Sit"},
    {"type": "Wait", "target": "BothSeated"},
    {"type": "Speak", "actor": "Hanako", "content": "我们开始吧。"}
  ],
  "dependencies": [
    ["Hanako Sit", "Wait"],
    ["V Sit", "Wait"],
    ["Wait", "Speak"]
  ],
  "tempo": "slow and formal",
  "interrupt_policy": "can_not_interrupt"
}
```

**这里没有 Graph，
但内部全是 Graph 逻辑。**

---

# 超简总结（你记这版就够）
- **narrative goal：这段戏想干嘛**
- **beat：一小段戏**
- **participant：谁参与**
- **atomic intent：最小可执行动作**
- **dependency：谁等谁**
- **tempo：节奏快慢**
- **interrupt policy：玩家打断怎么办**

---

如果你愿意，我可以下一步直接给你：
**一套完整可落地的 Narrative IR 结构（JSON Schema）**
你直接拿去给 AI 生成、给 UE 读取、做编辑器。

要我直接给你最终版吗？



# AI 原生叙事：从“语义标签”到真正可执行的时序系统

## 结论
你说得非常对：
只列 `narrative goal / beat / participant / atomic intent / dependency / tempo / interrupt policy`，**只是一堆语义标签，不是时序系统，一定会乱**。

AI 原生叙事要真正可执行，必须把**叙事语义层**和**时序编排层彻底拆开**。
上层管“发生什么”，下层管“什么时候发生、怎么排”。

---

# 一、为什么只给概念会乱？
一句导演描述：
> Hanako 先坐下，V 跟着坐，等两人稳定后开始正式对话，整体节奏克制。

里面至少混了 4 类信息：
1. **叙事目标**：正式开场、克制氛围
2. **动作内容**：坐下、对话
3. **时序关系**：先、跟着、等稳定后
4. **节奏风格**：克制、不要太快

全部揉在一层，结构必然混乱。

---

# 二、可落地系统必须拆成三层

## 第 1 层：语义层（发生什么）
只回答：
- 要表达什么
- 谁参与
- 做哪些原子动作

**不涉及精确时间。**

示例：
```json
{
  "goal": "formal_intro",
  "participants": ["Hanako", "V"],
  "intents": [
    "Hanako takes seat",
    "V takes seat",
    "Hanako opens conversation"
  ]
}
```

---

## 第 2 层：关系/约束层（它们之间是什么关系）
回答：
- 先后
- 并行
- 汇合
- 等待
- 互斥
- 中断跳转

这是**拓扑结构**，还不是最终时间轴。

示例：
```json
{
  "relations": [
    { "type": "before", "from": "HanakoSit", "to": "VSit" },
    { "type": "after_stable", "from": "VSit", "to": "HanakoSpeak" },
    { "type": "after_stable", "from": "HanakoSit", "to": "HanakoSpeak" }
  ]
}
```

---

## 第 3 层：时序层（什么时候开始、持续多久）
这一层才是真正的 **ExecutionStream 内核**。
必须有精确、可执行的时序字段：

- start_policy
- duration_policy
- earliest_start
- latest_start
- sync_condition
- overlap_policy
- slack
- tempo_modifier

示例：
```json
{
  "schedule": [
    {
      "id": "HanakoSit",
      "start": 0,
      "duration": 2.8
    },
    {
      "id": "VSit",
      "start": 1.0,
      "duration": 2.4
    },
    {
      "id": "HanakoSpeak",
      "start": 4.0,
      "duration": 3.6,
      "requires": ["HanakoSit.stable", "VSit.stable"]
    }
  ]
}
```

到这一步，才是**不乱、可执行**的时序。

---

# 三、不能只用 `dependency` 模糊带过
依赖至少有 6 种完全不同的语义：
1. 先后依赖（A 完 B 始）
2. 可重叠依赖（A 开始后 B 即可开始）
3. 汇合依赖（A、B 都完成 C 才开始）
4. 稳定态依赖（A 进入稳定状态后 B 才开始）
5. 信号依赖（等信号触发）
6. 软依赖（建议优先，可妥协）

所以必须明确约束类型：
```json
{
  "constraintType": "finish_to_start | start_to_start | join | signal_wait | stable_wait | soft_preference"
}
```

---

# 四、真正不乱的模型结构

## A. Narrative Atom（叙事原子）
最小可执行动作：
```json
{
  "id": "HanakoSpeak01",
  "type": "speak",
  "actor": "Hanako",
  "content": "V先生，感谢你前来。",
  "resource": "vo_hanako_intro_01"
}
```

## B. Temporal Constraint（时序约束）
```json
{
  "from": ["HanakoSit", "VSit"],
  "to": "HanakoSpeak01",
  "kind": "join_after_stable",
  "strength": "hard"
}
```

## C. Duration Source（时长来源）
```json
{
  "target": "HanakoSpeak01",
  "mode": "resource"
}
```

## D. Tempo Modifier（节奏修饰器）
节奏只是**影响求解的参数**，不是主结构：
```json
{
  "scope": "section",
  "style": "formal_deliberate",
  "pauseScale": 1.2,
  "overlapPreference": "low"
}
```

---

# 五、AI 真正该控制什么？
AI 不直接排毫秒、不直接画 Graph，而是控制三类核心：

1. **原子拆分**
   把意图拆成最小可执行动作
2. **约束生成**
   推断：finish-to-start / start-to-start / join / wait-signal / interrupt
3. **参数建议**
   duration、offset、overlap、pause

然后交给**时序求解器**生成最终时间轴。

---

# 六、正确的执行流程

1. **语义解析**：自然语言 → 叙事原子
2. **约束归类**：先/后/等/同时 → 标准约束类型
3. **时长绑定**：资源/规则/模板 → 确定时长
4. **节奏修正**：风格 → 调整停顿、重叠
5. **时序求解**：约束 + 时长 → 精确 start/end
6. **投影**：输出 timeline / graph / section / execution stream

---

# 七、你最该先定义的：**时序约束 DSL**
不要先聊抽象概念，先定一套最小时序语法：

- A then B
- A while B
- after A and B, do C
- wait signal X before C
- interrupt C on event Y

AI 的工作就是：
**自然语言 → 这套 DSL → 编译成时序**

结构立刻落地、不再飘。

---

# 八、从“混乱概念”到“清晰分层”

你原来的一组词：
- narrative goal
- beat
- participant
- atomic intent
- dependency
- tempo
- interrupt policy

可以重构成**可工程化的四层**：

### 语义层
- goal
- participants
- atoms

### 约束层
- ordering_constraints
- sync_constraints
- interrupt_constraints

### 时间层
- duration_sources
- offsets
- overlap_rules
- tempo_modifiers

### 输出层
- resolved_schedule（最终时序）

---

# 九、最终回答

你说“只给这些标签依旧是乱的”，**完全正确**。
它们只够描述“发生什么”，不够描述“怎么排、什么时候排”。

要让 AI 叙事真正成为**时序系统**，必须通过：
1. 叙事原子
2. 明确约束类型
3. 时长来源
4. 节奏修饰器
5. 时序求解器

最终编译成：
- start
- end
- duration
- wait
- join
- interrupt

这样才是**不乱、可执行、可落地**的 AI 原生叙事时序系统。

---

如果你需要，我可以下一步直接给你：
1. 一套最小**叙事时序 DSL**
2. 一套可直接在 UE 里用的 **Narrative Atom + Constraint JSON Schema**
3. 一句导演意图 → 完整时序结构的**端到端示例**

你想先出哪一个？