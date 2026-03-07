# 叙事时序编排产品：为什么先在 UE 实现，应该长什么样

> **核心判断：** 产品本质不是"2077 专属格式工具"，而是把**叙事意图 → 结构化段落 → 可执行时序**这个抽象落地。这个抽象与具体引擎解耦，而 UE 是验证它的最佳第一外壳。

---

## 一、为什么先在 UE 实现更合理

### 1. UE 已经有"时序容器"的土壤

UE 没有 2077 式的 ExecutionStream，但有大量近似物：

| UE 已有能力 | 对应用途 |
|------------|---------|
| Level Sequence / Sequencer | 时间轴 |
| Blueprint | 逻辑流 |
| StateTree / Behavior Tree | 状态与条件控制 |
| DataAsset / UObject | 结构化叙事数据 |
| Editor Utility / Slate | 自定义编辑器 UI |

不需要先造完整引擎，只需把核心抽象嫁接到 UE 现有编辑器生态里。

### 2. UE 最适合做"可视化验证"

只有 JSON 输出，产品价值很难被感知。在 UE 里可以直接看到：节点图、时间轴、镜头、动作片段、对话节奏、中断点。

> 这个产品不是纯后端系统，而是创作工具。**创作工具必须"看得见"。**

### 3. UE 比自研编辑器更容易快速试错

自研完整工具链需要自己解决：资产系统、预览系统、播放器、编辑器 UI、数据导入导出。UE 已经帮你解决了大半。

先在 UE 做，不是为了最终绑定 UE，而是为了先验证：

> **用户到底需要什么样的"叙事意图 → 时序"交互方式。**

### 4. UE 容易接 AI 工作流

UE 里很容易实现 MVP 所需的全部 UI：输入框 / 生成按钮 / 结构化面板 / 时间轴预览 / 导出器。

---

## 二、产品应该长什么样

不做成"聊天框插件"，而是做成：

```
Narrative Section Editor + AI Assistant
```

即一个**段落级叙事编排工作台**。

---

## 三、推荐样式：四栏式工作台

### 左侧：叙事意图输入区

用户写导演语言 / 策划语言，而不是代码。

**输入示例：**
- "Hanako 先坐下，示意 V 坐下，再开始正式对话"
- "整体节奏克制，有 0.5 秒停顿"
- "玩家打断时切到警觉分支"

**结构化标签：**
- 情绪风格 / 节奏风格
- 角色名单
- 必须动作 / 不允许发生的事

**视觉形态：** Prompt 输入框 + 结构化字段 + "生成 Section"按钮

---

### 中间：Section 结构图

AI 不直接给最终资产，而是先给一个 Section 草图。

**节点图视图示例：**

```
[Hanako Sit] --> [Wait Both Seated] --> [Hanako Opening Line]
       \              ^
        \--> [V Sit]--/
```

**列表视图示例：**

```
Section: Hanako Intro
  ├─ Hanako sits
  ├─ V sits
  ├─ Wait until both stable
  ├─ Hanako speaks opening line
  └─ Camera pushes in
```

每个条目可点开编辑：actor / duration / offset / dependency / interrupt policy

**视觉形态：** 节点图 + 列表双视图切换（类 Blueprint 与行为清单的结合）

---

### 右侧：属性面板

**选中 Section 时显示：**
- Section Name / Narrative Goal
- Participants / Timing Style
- Reference Duration / Interrupt Policy
- Tags / Start · End · Cancel sockets

**选中 Event 时显示：**
- Event Type / Actor
- Start Offset / Duration
- Wait Conditions / Signals Produced
- Allow Overlap / Preview Clip / Dialogue Line

---

### 下方：时序预览区

```
         0.0    1.0    2.0    3.0    4.0    5.0
Hanako   [ Sit ---------- ]
V                [ Sit ----- ]
Dialog                         [ Opening Line --- ]
Camera                         [ Push In -------- ]
```

不必像 Sequencer 那么复杂，但**必须有**。因为产品核心就在"时序感"。

---

## 四、MVP 应该包含的 5 个 UI 部件

| # | 部件 | 内容 |
|---|------|------|
| 1 | Prompt 输入框 | 输入导演意图 |
| 2 | Section 草案卡片 | 标题 / 目标 / 角色 / 预计时长 / 事件数 / 风格 |
| 3 | 事件列表 | Event 名 / actor / start / duration / depends on / signal in-out |
| 4 | 时间轴预览 | 简单甘特图 |
| 5 | 应用按钮 | 导出到 DataAsset / Blueprint graph / Sequencer 轨道草案 |

---

## 五、在 UE 里不要一开始做成什么

**不要做成纯聊天助手**
用户问一句、AI 回一段话，但没有结构、时序、可视化、可编辑结果——这不够。

**不要直接接 Sequencer 全量写入**
Sequencer 太重。先做自己的轻量 section / timeline 视图，等模型稳定后再导出到 Sequencer / Blueprint / DataAsset。

**不要一开始做全自动完整场景生成**
应该从单个 section、单段 beat、2~5 个原子动作开始，先验证"AI 是否真的能帮人省脑力"。

---

## 六、UE 内部数据结构设计

建议定义三层数据对象：

### 1. `NarrativeIntentAsset`
保存用户输入和 AI 解析出的高层目标。

```
- prompt
- narrative goal
- tone
- participants
- open questions
```

### 2. `SectionAsset`
对应 section 级容器。

```
- section id / title / goal
- duration
- actors
- interrupt policy
- event list
```

### 3. `EventIntent`
对应叙事原子。

```
- type / actor / content
- start offset / duration
- dependencies
- signals
```

三层结构让 UE 能自然地存资产、面板展示、导出运行时格式。

---

## 七、用户实际操作流程

```
Step 1  输入导演意图
        "Hanako 先落座，示意 V 落座，等两人稳定后开始正式对话，整体节奏克制且有压迫感。"

        ↓

Step 2  AI 生成 Section 草案
        - 名称：Hanako Formal Intro
        - 角色：Hanako, V
        - 风格：formal / slow / dominant
        - 时长建议：8.2s
        - 事件：Hanako sit → V sit → wait both stable → Hanako line → camera push-in

        ↓

Step 3  用户在可视化界面微调
        - 把停顿从 0.5s 改成 0.8s
        - 让镜头和台词重叠
        - 把 V 坐下改成更快动画

        ↓

Step 4  导出
        - 自定义 runtime asset
        - Blueprint 节点
        - Sequencer 标记
        - JSON
```

---

## 八、为什么这种方式比"直接做 2077 式工具"更好

现在真正需要验证的不是某个字段如何映射到引擎内部类，而是：

- 用户如何描述叙事意图
- AI 如何把意图拆成结构
- 什么可视化方式最容易让设计师信任
- 哪个层级最适合作为人工审核单位

这些问题，UE 完全足够验证。

---

## 九、产品命名与页面气质

**命名方向：**
- Narrative Orchestrator
- Scene Timing Copilot
- Section Composer
- Temporal Narrative Editor

**页面气质：**

```
左边  Notion / Prompt 风格的意图输入
中间  Blueprint-lite 结构图
右边  Details 属性面板
下方  Sequencer-lite 时间轴
```

不是纯聊天，不是纯蓝图，不是纯表格——**三者融合**。

---

## 十、一句话总结

**为什么先在 UE 实现？**
UE 已具备结构化资产、可视化编辑、时间轴预览和编辑器扩展能力，足够作为"叙事意图 → 可执行时序"产品的第一个验证平台。

**实现出来应该是什么样？**
一个 Section 级叙事编排工作台：左边输入意图，中间生成 section 结构，右边编辑属性，下方预览时间轴。

---

## 下一步可选方向

1. **UE 插件界面草图** — 四栏工作台的具体布局设计
2. **UE Narrative Section 数据结构设计** — 完整字段定义与类型约束
3. **MVP 页面流程图** — 用户操作路径与交互状态
