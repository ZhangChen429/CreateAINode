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

