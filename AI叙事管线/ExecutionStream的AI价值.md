# 先在 UE 实现叙事时序编排产品的完整方案
## 核心判断
你的产品本质不是「2077 专属格式工具」，而是：
**叙事意图 → 结构化段落 → 可执行时序**

这套抽象与具体引擎解耦，而 UE 是最适合做**第一性验证**的载体，因为它天然具备：
1. 编辑器扩展能力强
2. Sequencer / Blueprint / StateTree / Dialogue 等可视化基础
3. 可直接将 AI 输出变为「可见、可预览、可修改」的资产

因此，UE 适合作为 AI 时序编排产品的第一个外壳。

---

## 一、为什么先在 UE 实现更合理
### 1. UE 已有「时序容器」土壤
UE 没有完全一样的 ExecutionStream，但有大量近似底层能力：
- Level Sequence / Sequencer：时间轴
- Blueprint：逻辑流
- StateTree / Behavior Tree：状态与条件控制
- DataAsset / UObject：结构化叙事数据
- Editor Utility / Slate：自定义编辑器 UI

无需从零造引擎，只需将核心抽象**嫁接**到 UE 现有生态。

### 2. UE 最适合做「可视化验证」
纯 JSON 输出价值难以感知，在 UE 中可直接预览：
- 节点图
- 时间轴
- 镜头
- 动作片段
- 对话节奏
- 中断点

产品是**创作工具**，创作工具必须「看得见」。

### 3. 比自研编辑器更容易快速试错
自研独立工具链需要从头实现：
- 资产系统
- 预览系统
- 播放器
- 编辑器 UI
- 导入导出

UE 已解决大半，优先验证：
**用户到底需要什么样的「叙事意图 → 时序」交互方式**。

### 4. UE 极易接入 AI 工作流
MVP 最简形态即可实现：
- 输入框
- 「生成 Section 草案」按钮
- 结构化面板
- 时间轴预览
- 导出器

---

## 二、UE 端产品形态：Narrative Section Editor + AI Assistant
不做普通聊天框插件，而是**段落级叙事编排工作台**。

### 推荐样式：四栏式工作台
#### 左侧：叙事意图输入区
使用导演/策划语言输入，例如：
- Hanako 先坐下，示意 V 坐下，再开始正式对话
- 整体节奏克制，有 0.5 秒停顿
- 玩家打断时切到警觉分支

支持结构化字段：
- 情绪风格
- 节奏风格
- 角色名单
- 必须动作
- 禁止行为

视觉：Prompt 输入框 + 结构化字段 + 生成按钮。

#### 中间：Section 结构图
AI 输出 Section 草图，而非最终资产，例如：
```
Section: Hanako Intro
- Hanako sits
- V sits
- Wait until both stable
- Hanako speaks opening line
- Camera pushes in
```

支持编辑：
- actor
- duration
- offset
- dependency
- interrupt policy

视觉：节点图 + 列表双视图切换。

#### 右侧：属性面板
- 选中 Section：名称、叙事目标、参与者、时序风格、中断策略等
- 选中 Event：事件类型、演员、偏移、时长、依赖、信号等

#### 下方：时序预览区
简化版时间轴（甘特图形式），直观展示时序与重叠关系。

---

## 三、MVP 最小可行产品（UE 内）
只保留 5 个核心 UI 部件：
1. Prompt 输入框
2. AI 生成 Section 草案卡片
3. 事件列表
4. 时间轴预览
5. 应用/导出按钮

可导出至：
- 自定义 DataAsset
- Blueprint Graph
- Sequencer 轨道草案
- JSON

---

## 四、在 UE 里不要一开始做什么
1. **不要做成纯聊天助手**
   无结构、无时序、无可视化、无编辑结果，价值极低。

2. **不要直接全量写入 Sequencer**
   Sequencer 过重，先做轻量 Section/Timeline 视图，稳定后再导出。

3. **不要一上来做全自动完整场景生成**
   从：
   - 单个 Section
   - 单段 Beat
   - 2~5 个原子动作
   开始，先验证「AI 是否真的能省脑力」。

---

## 五、UE 内部数据结构设计（三层）
### 1. NarrativeIntentAsset
保存用户输入与 AI 解析的高层目标：
- prompt
- narrative goal
- tone
- participants
- open questions

### 2. SectionAsset
Section 级容器：
- section id
- title
- goal
- duration
- actors
- interrupt policy
- event list

### 3. EventIntent
叙事原子：
- type
- actor
- content
- start offset
- duration
- dependencies
- signals

支持资产保存、面板展示、运行时格式导出。

---

## 六、用户实际操作流程
1. **输入导演意图**
   Hanako 先落座，示意 V 落座，等两人稳定后开始正式对话，整体节奏克制且有压迫感。

2. **AI 生成 Section 草案**
   - 名称：Hanako Formal Intro
   - 角色：Hanako, V
   - 风格：formal / slow / dominant
   - 时长：8.2s
   - 事件列表

3. **可视化界面微调**
   修改停顿、重叠、动画速度等。

4. **导出**
   导出为运行时资产、Blueprint、Sequencer 标记或 JSON。

---

## 七、为什么比直接做 2077 式工具更好
当前要验证的不是：
- 某个字段如何映射引擎内部类
而是：
- 用户如何描述叙事意图
- AI 如何拆解结构
- 哪种可视化最让设计师信任
- 哪一层最适合人工审核

这些问题，UE 完全足够验证。

---

## 九、插件命名与界面风格
### 推荐名称
- Narrative Orchestrator
- Scene Timing Copilot
- Section Composer
- Temporal Narrative Editor

### 页面感觉
- 左侧：Notion / Prompt 风格
- 中间：Blueprint-lite 结构图
- 右侧：Details 面板
- 下方：Sequencer-lite 时间轴

融合文本、节点、属性、时序，而非单一形态。

---

## 十、一句话总结
- 为什么先在 UE 实现？
  UE 拥有结构化资产、可视化编辑、时间轴预览与编辑器扩展能力，足够作为「叙事意图 → 可执行时序」的第一验证平台。

- 最终长成什么样？
  **Section 级叙事编排工作台**：左边输意图，中间生结构，右边改属性，下方预览时序。

---

## 下一步可输出内容（任选其一）
1. UE 插件界面草图
2. UE Narrative Section 完整数据结构设计
3. MVP 页面流程图