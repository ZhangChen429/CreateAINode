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