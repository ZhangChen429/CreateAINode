# 叙事意图最小单元与 SectionNode 架构分析

> **核心结论：** 叙事意图的最小单元不是"台词文本"，也不等同于"整个 SectionNode"，而是介于两者之间的**可执行叙事原子（Narrative Action Intent）**。SectionNode 则是把一组叙事原子组织成可播放、可中断、可缩放、可恢复的**段落容器**。

---

## 一、为什么"文本台词"不是最小单元

如果把最小单元定义成一行台词文本，会丢掉太多执行信息。

例如一句：

> "Hanako：V先生，感谢你前来。"

在执行层还缺少以下信息：

- 谁说，对谁说
- 什么时候说
- 在什么姿态 / 位置下说
- 是否要等前置动作完成
- 是否允许打断
- 说完触发什么
- 是否伴随镜头 / 表情 / lookat / gesture
- 它属于哪个段落 / section
- 是主推进内容还是附属事件

因此，文本更像是**语义内容载荷**，而不是**执行单元**。

---

## 二、为什么"SectionNode"也不是最小单元

从源码分析，SectionNode 本质上包含：

- `m_events`
- `m_refrncDuration`
- `m_actorBehaviors`
- `m_isFocusClue`
- `m_common`

并且在 `DoStimulate()` 里，它会：

1. 先生成一个 **primary ActionRecord**，代表 section 自己
2. 再遍历 `m_events`，让每个 event 各自刺激生成 **secondary records**

这说明：

> **SectionNode 不是一个具体动作，而是一个"段落级编排容器"。**

它更像：一段对话、一段表演、一个小节拍、一个连续 scene beat。

---

## 三、SectionNode 的准确定义

```
SectionNode = 段落级叙事执行容器
```

它承载的不是一句文本，而是**一段可执行叙事片段**，具有：

- 一个整体时长
- 一组内部事件
- 一组参与角色约束
- 一套开始 / 结束 / 取消接口
- 一套运行状态与恢复能力
- 一套缩放 / 快进 / 编辑器跳转能力

---

## 四、SectionNode 的八类核心组成

### 1. 段落身份

SectionNode 首先要有"它是谁"，至少包含：

- section id / name / beat name
- 所属 scene / chapter
- 入口条件与退出条件

```json
{
  "sectionId": "hanako_intro_01",
  "label": "Hanako 初次会面开场",
  "role": "intro beat"
}
```

---

### 2. 段落时长

`m_refrncDuration` 非常核心，说明 SectionNode 天然是一个**时间段**，不是纯逻辑块。

在 AI 产品里，section 应该承载的不是"文本总和"，而是：

> 一个有整体节奏轮廓的时序窗口

必须包含：参考时长、缩放后时长、内部事件时间分布。

---

### 3. 内部事件集合 `m_events`（最关键）

`DoStimulate()` 明确说明：
- section 自己生成 primary record
- `m_events` 生成 secondary records

因此：

> **SectionNode 的叙事内容主体不是自己，而是内部 SceneEvent 集合。**

AI 生成一个 section 时，最重要的是生成：
- 事件列表
- 事件之间的相对位置
- 事件绑定的 actor / 资源 / 信号 / 触发

---

### 4. Actor 参与约束

`ActorBehavior` 包含：
- `m_actorId`
- `m_behaviorMode`（`OnlyIfAlive` / `EvenIfDead`）

这不是装饰信息，而是**执行约束**。Section 应包含：
- 参与角色及其必要性
- 生存 / 状态约束
- 角色职责

---

### 5. 段落控制接口

SectionNode 有明确 socket：

| 方向 | Socket |
|------|--------|
| 输入 | `in`、`cancel` |
| 输出 | `out`、`cancelFwd`、`transmitSignal` |

在 AI 产品建模里，section 必须包含：
- start condition
- end signal
- cancel behavior
- outgoing signals

否则它只是"内容块"，不是"流程块"。

---

### 6. 可恢复 / 可跳转状态

SectionNode 提供：
- `DoGenerateRestorationToken`
- `DoGenerateStateToken`
- `DoEditor_JumpToTimepos`
- `DoEditor_StartProcessing` / `DoEditor_StopProcessing`

这说明 section 是**可以进入、恢复、跳转、停止的可运行状态块**，而非一次性文本片段。

工业级产品的 section 层模型必须考虑：
- 当前进度
- 可恢复点
- 跳转语义
- 中断后的重入策略

---

### 7. 缩放能力

SectionNode 会 `BuildScaler()`，用事件分布生成缩放器，天然支持：

> 同一段 section，在不同条件下调整整体时长，但尽量保留内部结构。

这非常像导演语言里的：
- "这一段紧一点"
- "这里停顿再长一点"
- "让这一段更压迫"

这对 AI 产品极重要，因为用户往往不想精确改毫秒，而是描述**节奏感**。这正适合 section 级建模，而不是单台词级建模。

---

### 8. 段落级语义标签

如 `m_isFocusClue`，说明 section 还能承担更高层叙事语义：
- 焦点线索
- 独占交互
- 特殊呈现模式

这类标签是提示词可用的**强语义锚点**。

---

## 五、叙事意图的三层结构

不要只找一个单点，而应分三层：

### 第 1 层：叙事原子（最小单元）

```
Narrative Action Intent
```

最小执行语义单元，不是文本，而是：

- Hanako 说一句正式欢迎词
- V 坐下
- 服务员上酒
- 镜头切近
- Hanako 停顿半秒看向 V

这些才是真正最小的"可执行叙事原子"，可能会映射到一个或多个 SceneEvent / ActionRecord。

---

### 第 2 层：段落单元（主工作单元）

```
Narrative Beat / Section（类 SectionNode）
```

一组原子被组织成一个具备以下特征的段落：
- 有边界
- 有节奏
- 有角色约束
- 有开始结束条件
- 可中断可恢复

> 这是**最适合 AI 与设计师交互的主工作单元**。

---

### 第 3 层：图级结构

多个 section 通过**顺序、分支、汇合、条件、中断**连接成 SceneGraph。

---

## 六、SectionNode 推荐产品数据结构

```json
{
  "sectionId": "hanako_intro_01",
  "title": "Hanako 初次见面开场",
  "narrativeGoal": "建立正式、克制、上位者姿态",
  "participants": [
    { "actor": "Hanako", "required": true,  "mode": "OnlyIfAlive" },
    { "actor": "V",      "required": true,  "mode": "OnlyIfAlive" },
    { "actor": "Waiter", "required": false, "mode": "EvenIfDead"  }
  ],
  "referenceDuration": 8200,
  "timingStyle": "formal_slow_burn",
  "events": [
    "Hanako sits",
    "V sits",
    "brief pause",
    "Hanako opening line",
    "camera push-in"
  ],
  "entry": {
    "startSignal": "in"
  },
  "exit": {
    "successSignal": "out",
    "cancelSignal": "cancelFwd"
  },
  "interruptPolicy": {
    "canCancel": true,
    "recoveryMode": "resume_or_branch"
  },
  "semanticTags": ["intro", "negotiation", "focus_clue"]
}
```

**核心结构思想：**

> SectionNode 应该保存"段落级叙事目标 + 参与者约束 + 内部事件编排 + 时间边界 + 控制接口"。

---

## 七、叙事意图与提示词工程的结合

### 原则

> 不要让提示词直接生成最终代码或最终节点文件，而要分三段走。

---

### 第一段：Prompt 生成"叙事意图对象"

**用户输入：**
> Hanako 先坐下，示意 V 落座，等两人都稳定后，用正式但带压迫感的语气开场。

**LLM 第一阶段输出（不是 sectionNode，而是结构化意图）：**

```json
{
  "beatGoal": "formal_dominance_intro",
  "participants": ["Hanako", "V"],
  "atomicIntents": [
    "Hanako sit",
    "V sit",
    "wait both stable",
    "Hanako greet line"
  ],
  "dependencies": [
    ["Hanako sit",  "wait both stable"],
    ["V sit",       "wait both stable"],
    ["wait both stable", "Hanako greet line"]
  ],
  "tempo": "formal_deliberate",
  "uncertainties": [
    "Should V sit immediately or after gesture?",
    "Should greeting overlap with camera move?"
  ]
}
```

---

### 第二段：规则编译器把意图对象编译成 SectionNode / SceneEvent

这一步**不靠纯 LLM**，而由 deterministic compiler 完成：
- 生成 section duration
- 生成 event offset
- 生成 signal / wait
- 生成 sockets / exit behavior
- 校验 actor 冲突

---

### 第三段：提示词工程围绕"结构化输出"而非"文采"

**不该问：**
> "请帮我写一段精彩的开场戏"

**应该问：**
- 这段的目标是什么？
- 哪些角色是必需参与者？
- 有哪些原子动作？哪些可并行？哪些必须等待？
- 这段节奏是什么？
- 哪些地方允许中断？中断后更像取消、暂停还是转分支？

---

## 八、提示词模板示例

```
你是交互叙事时序设计助手。
请把下面的导演意图拆成一个 section 级叙事结构。

要求输出：
1. sectionGoal
2. participants
3. atomicIntents
4. dependencies
5. timingStyle
6. interruptPolicy
7. openQuestions

导演意图：
"Hanako 先坐下，示意 V 落座，等两人稳定后正式开场。
整体氛围克制、带压迫感，不要太快。"
```

**核心目标：** 不是让模型"写内容"，而是让模型——

- 拆结构
- 识别依赖
- 识别时序
- 识别约束
- 暴露不确定性

---

## 九、最终归纳

### Q1：叙事意图的最小单元是什么？

> 最小单元 = **可执行叙事原子（Narrative Action Intent）**

不是文本台词，而是：某角色说一句 / 某角色坐下 / 某个镜头推进 / 某个停顿 / 某个信号等待。

---

### Q2：SectionNode 是什么？

> SectionNode = **一组叙事原子的段落级编排容器**

负责：聚合事件 / 定义段落时长 / 管理参与者约束 / 提供开始·结束·取消接口 / 支持缩放·恢复·编辑器跳转。

---

### Q3：SectionNode 中最重要的成员是什么？

| 优先级 | 成员 | 说明 |
|--------|------|------|
| ★★★★★ | `m_events` 事件集合 | 叙事内容主体 |
| ★★★★★ | `m_refrncDuration` 段落时长 | 节奏边界 |
| ★★★★☆ | `m_actorBehaviors` 角色约束 | 执行合法性 |
| ★★★★☆ | Sockets 控制接口 | 流程可控性 |
| ★★★☆☆ | `m_isFocusClue` 语义标签 | 高层叙事锚点 |

> SectionNode 的核心不是"文本"，而是**"带边界和约束的一组时序化事件"**。

---

### Q4：和提示词工程怎么结合？

```
Prompt → 叙事结构化意图 → 编译器 → SceneEvent / ExecutionStream
```

**而不是让 LLM 直接吐最终引擎资产。**

---

## 十、下一步可选方向

1. **设计 SectionNode 级叙事意图 Schema** — 完整字段定义与约束规则
2. **编写用于生成 SectionNode 草案的 Prompt 模板集** — 覆盖常见叙事场景类型
