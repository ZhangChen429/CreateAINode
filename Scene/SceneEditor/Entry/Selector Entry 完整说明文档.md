# Selector Entry 完整说明文档

## 一、Selector 是什么？

Selector 是一个智能容器，用于管理和随机选择多个不同姿态（idle）的 Sequence，并自动处理它们之间的姿态转换。



```
class Selector : public RandomList  // 继承随机选择能力

{

&#x20;   CName m\_idleAnim;  // 基础姿态（回退用）

&#x20;   // + 自动 transition 处理

&#x20;   // + 姿态回退机制

};
```

## 二、解决什么核心问题？

### 核心痛点：多姿态切换时的动画穿模

#### 应用场景

NPC 坐在椅子上，有多种行为：



* 拿手机打电话（右手举到耳边）

* 看平板电脑（双手托着平板）

* 翘二郎腿（腿交叉）

* 抽烟（右手拿烟）

#### 无 Selector 的问题

❌ 使用 Random List 直接选择：



```
Random List

├─ Anim: 打电话 (右手在耳边)

└─ Anim: 看平板 (双手托平板)
```

从 "打电话" 切换到 "看平板"：



* 右手突然从耳边瞬移到胸前 → 穿模！

* 没有 "收起手机、拿出平板" 的过程 → 不自然！

#### 有 Selector 的解决方案

✅ Selector 自动管理姿态转换：



```
Selector: idle:sit\_base (基础坐姿)

├─ Sequence: idle:sit\_phone (打电话姿态)

└─ Sequence: idle:sit\_tablet (看平板姿态)
```

切换流程：

`sit_phone → [transition: 收手机] → sit_base → [transition: 拿平板] → sit_tablet`

↑ 自动播放过渡动画 ↑                 ↑ 自动播放过渡动画 ↑

### 核心解决的三大问题

#### 问题 1：姿态不兼容导致穿模



* **问题**：从 A 姿态直接跳到 B 姿态 → 手 / 脚 / 道具位置突变 → 穿模、不自然

* **Selector 解决**：自动查找并播放 transition 动画 → `A → [A__2__B transition] → B`

#### 问题 2：缺少中间状态



* **问题**：某些姿态之间没有直接的 transition 动画（例如：sit\_phone → sit\_cigarette 无对应动画）

* **Selector 解决**：使用基础 idle 作为中间状态 → `sit_phone → [回到] sit_base → [再转到] sit_cigarette`（这两个动画容易制作）

#### 问题 3：管理复杂度爆炸



* **问题**：5 个不同姿态需 5×4=20 个 transition 动画（每个姿态到其他姿态）

* **Selector 解决**：通过基础 idle 回退，只需 5×2=10 个（每个姿态到 / 从基础姿态）

## 三、与其他容器的对比



| 容器          | 用途                 | Idle 管理   | Transition          |
| ----------- | ------------------ | --------- | ------------------- |
| Sequence    | 顺序播放一组动画           | 有单一 idle  | 入口 / 出口有 transition |
| Random List | 随机选择一个动画           | 有单一 idle  | 入口 / 出口有 transition |
| Selector    | 随机选择不同姿态的 Sequence | 管理多个 idle | 自动处理姿态间 transition  |

## 四、实际应用场景

### 场景 1：餐厅顾客



```
Selector: idle:sit\_eating\_base

├─ Sequence: idle:sit\_eating\_fork (用叉子)

├─ Sequence: idle:sit\_drinking\_wine (喝红酒)

├─ Sequence: idle:sit\_phone (看手机)

└─ Sequence: idle:sit\_talking (聊天手势)
```

自动处理：用叉子 → 放下叉子 → 拿酒杯 → 喝酒

### 场景 2：站立 NPC



```
Selector: idle:stand\_neutral

├─ Sequence: idle:stand\_phone (打电话)

├─ Sequence: idle:stand\_cigarette (抽烟)

├─ Sequence: idle:stand\_arms\_crossed (抱臂)

└─ Sequence: idle:stand\_hands\_pocket (插兜)
```

自动处理所有姿态切换的 transition

### 场景 3：办公室员工



```
Selector: idle:sit\_desk\_base

├─ Sequence: idle:sit\_typing (打字)

├─ Sequence: idle:sit\_reading (看文件)

├─ Sequence: idle:sit\_thinking (思考)

└─ Sequence: idle:sit\_phone (接电话)
```

管理复杂的办公行为切换

## 五、核心价值（一句话总结）

Selector 让多个不同姿态的行为能够自然流畅地切换，避免穿模和突变，同时大幅降低动画师的工作量。

## 六、关键配置要点

### ✅ 必须做的



1. 给 Selector 设置基础 idle：`Selector.m_idleAnim = "sit_base"`（回退用）

2. 每个子 Sequence 设置不同的 idle：

* `Sequence A.m_idleAnim = "sit_phone"`

* `Sequence B.m_idleAnim = "sit_tablet"`

1. 准备 transition 动画：

* `sit_base__2__sit_phone.anim`

* `sit_phone__2__sit_base.anim`

* `sit_base__2__sit_tablet.anim`

* `sit_tablet__2__sit_base.anim`

### ❌ 常见错误



1. 不设置 idle → 无法生成 transition

2. 所有 Sequence 用同一个 idle → Selector 退化成 Random List

3. 忘记准备 transition 动画 → 姿态切换会穿模

## 七、决策树：何时使用 Selector？



```
你的 NPC 是否有多个不同的姿态（手的位置、腿的姿势、持有道具）？

├─ 是 → 使用 Selector

│   └─ 每个姿态创建一个 Sequence

│       └─ Selector 自动管理姿态切换

│

└─ 否（只有一个姿态，多个手势动画）

&#x20;   └─ 使用 Sequence + Random List

&#x20;       └─ 例如：站立时的各种小动作（挠头、伸懒腰、看手表）
```

### 总结公式



* Selector = Random List + 多姿态管理 + 自动 Transition

* 解决问题 = 防穿模 + 流畅切换 + 降低动画量

* 本质：Selector 是一个姿态状态机管理器，它知道角色当前是什么姿态，要切换到什么姿态，并自动插入正确的过渡动画。



***

## 八、Selector 如何找到相应的 TransitionAnim？

### 一、核心机制：继承链赋予的能力



```
IdleGuard ← 这是 transition 魔法的源头！

&#x20;   ↑

ContainerIterator ← 所有容器继承这个能力

&#x20;   ↑

RandomListIterator ← 随机列表继承

&#x20;   ↑

SelectorIterator ← Selector 继承并增强
```

关键：Selector 并不是自己实现 transition 查找，而是继承了 IdleGuard 的自动 transition 能力！

### 二、查找流程（分步骤详解）

#### 步骤 1：IdleGuard 检测 idle 变化

**代码位置**：workspotTreeItems.cpp:325-357



```
// IdleGuard::Next() - 每次选择新节点时调用

virtual void Next(const EntryIterationContext& context) override

{

&#x20;   const EntryClass\* clip = static\_castClass\*>(m\_pointedEntry);

&#x20;   // ✅ 第一步：获取当前和目标 idle

&#x20;   CName fromIdle = context.m\_currentIdleAnim;  // 当前姿态

&#x20;   CName toIdle = clip->m\_idleAnim;             // 目标容器的姿态

&#x20;   // ✅ 第二步：检查是否需要 transition

&#x20;   Bool hasPendingTransition =

&#x20;       m\_state == State::CheckTransition &&

&#x20;       (fromIdle && toIdle && fromIdle != toIdle);  // idle 不同才需要

&#x20;   if (hasPendingTransition)

&#x20;   {

&#x20;       // ✅ 第三步：调用查找函数

&#x20;       DetermineTransitionAnim(

&#x20;           context.m\_customTransitionAnims,  // 自定义 transition 列表

&#x20;           fromIdle,                         // 从哪个 idle

&#x20;           toIdle,                           // 到哪个 idle

&#x20;           m\_transiionAnimName               // \[输出] 找到的动画名

&#x20;       );

&#x20;       m\_state = State::PerformTransition;  // 标记：下次返回 transition

&#x20;       return;  // 停止，先播放 transition

&#x20;   }

&#x20;   // 没有 transition 或已播放完，继续正常流程

&#x20;   BaseIterator::Next(context);

}
```

#### 步骤 2：DetermineTransitionAnim 查找动画

**代码位置**：workspotTreeItems.cpp:248-269



```
void DetermineTransitionAnim(

&#x20;   const red::DynArray\<TransitionAnim>\* customTransitionAnims,

&#x20;   CName fromIdle,

&#x20;   CName toIdle,

&#x20;   CName& transitionAnimName)  // 引用，直接修改输出

{

&#x20;   Bool found = false;

&#x20;   // ✅ 优先级 1：查找自定义 transition

&#x20;   if (customTransitionAnims)

&#x20;   {

&#x20;       for (const TransitionAnim& entry : \*customTransitionAnims)

&#x20;       {

&#x20;           if (entry.m\_fromIdle == fromIdle && entry.m\_toIdle == toIdle)

&#x20;           {

&#x20;               transitionAnimName = entry.m\_transitionAnim;

&#x20;               found = true;

&#x20;               break;

&#x20;           }

&#x20;       }

&#x20;   }

&#x20;   // ✅ 优先级 2：生成标准命名的 transition

&#x20;   if (!found)

&#x20;   {

&#x20;       // 格式：fromIdle\_\_2\_\_toIdle

&#x20;       // 例如：sit\_phone\_\_2\_\_sit\_tablet

&#x20;       transitionAnimName = GenerateTransitionAnimName(fromIdle, toIdle);

&#x20;   }

}
```

#### 步骤 3：GenerateTransitionAnimName 生成动画名

**代码位置**：workspotResource.cpp (推测实现)



```
CName GenerateTransitionAnimName(CName fromIdle, CName toIdle)

{

&#x20;   // 格式：fromIdle\_\_2\_\_toIdle

&#x20;   return CName(String::Printf("%s\_\_2\_\_%s",

&#x20;                               fromIdle.AsChar(),

&#x20;                               toIdle.AsChar()));

}
```

#### 步骤 4：IdleGuard 返回 transition 数据

**代码位置**：workspotTreeItems.cpp:289-305



```
virtual void GetData(WorkspotEntryData& outData) override

{

&#x20;   if (IsTransitionActive())  // m\_state == State::PerformTransition

&#x20;   {

&#x20;       // ✅ 返回 transition 动画的数据

&#x20;       outData.m\_animationName = m\_transiionAnimName;  // 要播放的动画

&#x20;       outData.m\_idleAnimName = clip->m\_idleAnim;      // 目标 idle

&#x20;       outData.m\_entryFlags = IEntry::Animation |

&#x20;                              IEntry::MotionAnim |

&#x20;                              IEntry::TransitionAnim;  // 标记为 transition

&#x20;       outData.m\_blendTime = m\_blendTime;

&#x20;   }

&#x20;   else

&#x20;   {

&#x20;       // 正常返回子节点数据

&#x20;       BaseIterator::GetData(outData);

&#x20;   }

}
```

### 三、Selector 的额外增强：回退机制

Selector 在 IdleGuard 基础上，增加了一层 "找不到 transition 时的回退策略"。

**代码位置**：workspotTreeItems.cpp:1044-1066



```
class SelectorIterator : public RandomListIterator

{

&#x20;   virtual IEntry\* GetNextElement(Int32 index, const EntryIterationContext& context) override

&#x20;   {

&#x20;       // 1. 先用 RandomList 的逻辑选择一个子节点

&#x20;       THandle nextEl = RandomListIterator::GetNextElement(index, context);

&#x20;       THandle\<IContainerEntry> nextContainer = CastontainerEntry>(nextEl);

&#x20;       // 2. 预先检查是否有 transition 动画

&#x20;       CName fromIdle = context.m\_currentIdleAnim;

&#x20;       CName toIdle = nextContainer ? nextContainer->m\_idleAnim : CName::NONE();

&#x20;       CName transitionAnim;

&#x20;       if (fromIdle && toIdle && fromIdle != toIdle)

&#x20;       {

&#x20;           // 调用相同的查找函数

&#x20;           DetermineTransitionAnim(context.m\_customTransitionAnims,

&#x20;                                  fromIdle, toIdle, transitionAnim);

&#x20;       }

&#x20;       // 3. ✅ Selector 特有：检查动画是否真的存在

&#x20;       Bool changeToBaseIdle = !transitionAnim ||                    // 没找到

&#x20;                               !context.m\_animExistFunctor ||         // 或

&#x20;                               !context.m\_animExistFunctor(transitionAnim);  // 动画文件不存在

&#x20;       if (changeToBaseIdle)

&#x20;       {

&#x20;           // 4. ✅ 使用回退策略：用 Selector 的 m\_idleAnim 包裹

&#x20;           m\_idleSequence->m\_list.Clear();

&#x20;           m\_idleSequence->m\_list.PushBack(nextEl);

&#x20;           m\_idleSequence->m\_idleAnim = clip->m\_idleAnim;  // Selector 的基础 idle

&#x20;           nextEl = m\_idleSequence;  // 返回包裹后的节点

&#x20;           // 这样 IdleGuard 会查找两次 transition：

&#x20;           // fromIdle → Selector.m\_idleAnim (第一次)

&#x20;           // Selector.m\_idleAnim → toIdle  (第二次)

&#x20;       }

&#x20;       return nextEl.Get();

&#x20;   }

};
```

### 四、完整调用链（实际例子）

#### 场景设置



* 当前状态：currentIdleAnim = "sit\_neutral"

* Selector: m\_idleAnim = "sit\_base"

  └─ Sequence: m\_idleAnim = "sit\_phone"

  └─ Random List

  └─ Anim: sit\_phone\_talk\_01

#### 调用链展开



1. WorkspotInstance::Update()

2. m\_iterator->Next (context)  // Selector 的 Iterator

3. SelectorIterator::GetNextElement ()

   ├─ 选中：Sequence (sit\_phone)

   ├─ 检查：fromIdle="sit\_neutral", toIdle="sit\_phone"

   ├─ 调用 DetermineTransitionAnim (...)

   ├─ 查找：sit\_neutral\_\_2\_\_sit\_phone

   ├─ 检查文件是否存在：animExistFunctor ("sit\_neutral\_\_2\_\_sit\_phone")

   ├─ 结果：不存在

   └─ 启用回退：用 m\_idleSequence 包裹

   └─ m\_idleSequence.m\_idleAnim = "sit\_base"

4. ContainerIterator::Next ()  // 处理包裹后的 Sequence

5. IdleGuard::Next ()  // 第一次检测

   ├─ fromIdle = "sit\_neutral"

   ├─ toIdle = "sit\_base" (临时 Sequence 的 idle)

   ├─ 调用 DetermineTransitionAnim (...)

   ├─ 查找：sit\_neutral\_\_2\_\_sit\_base

   ├─ 找到！

   └─ m\_state = PerformTransition

6. WorkspotInstance::GetData()

7. IdleGuard::GetData ()

   └─ 返回：animationName = "sit\_neutral\_\_2\_\_sit\_base"

8. 系统播放：sit\_neutral\_\_2\_\_sit\_base

9. 播放完成，currentIdleAnim = "sit\_base"

10. ContainerIterator::Next () 继续

11. IdleGuard::Next ()  // 第二次检测

    ├─ fromIdle = "sit\_base"

    ├─ toIdle = "sit\_phone" (实际目标 Sequence)

    ├─ 调用 DetermineTransitionAnim (...)

    ├─ 查找：sit\_base\_\_2\_\_sit\_phone

    ├─ 找到！

    └─ m\_state = PerformTransition

12. 系统播放：sit\_base\_\_2\_\_sit\_phone

13. 播放完成，currentIdleAnim = "sit\_phone"

14. 播放 Sequence 内的实际动画：sit\_phone\_talk\_01

### 五、查找优先级总结



1. **自定义 Transition（最高）**



```
// 在 WorkspotTree.m\_customTransitionAnims 中定义

{

&#x20;   m\_fromIdle = "sit\_neutral",

&#x20;   m\_toIdle = "sit\_phone",

&#x20;   m\_transitionAnim = "custom\_pickup\_phone"  // ← 使用这个

}
```



1. **标准命名 Transition**



```
// 格式：fromIdle\_\_2\_\_toIdle

GenerateTransitionAnimName("sit\_neutral", "sit\_phone")

→ "sit\_neutral\_\_2\_\_sit\_phone"  // ← 查找这个动画文件
```



1. **Selector 回退（仅 Selector）**



```
// 如果上述都找不到或文件不存在

// 分两步：

// 第一步：sit\_neutral → sit\_base

// 第二步：sit\_base → sit\_phone
```



1. **直接 Blend（最后手段）**



```
// 如果所有 transition 都不存在

// 系统直接 blend 两个动画

// → 可能穿模！
```

### 六、关键代码位置速查表



| 功能               | 文件                    | 行号                  | 函数                                 |
| ---------------- | --------------------- | ------------------- | ---------------------------------- |
| 检测 idle 变化       | workspotTreeItems.cpp | 325-357             | IdleGuard::Next()                  |
| 查找 transition    | workspotTreeItems.cpp | 248-269             | DetermineTransitionAnim()          |
| 生成动画名            | workspotResource.cpp  | -                   | GenerateTransitionAnimName()       |
| 返回 transition 数据 | workspotTreeItems.cpp | 289-305             | IdleGuard::GetData()               |
| Selector 回退机制    | workspotTreeItems.cpp | 1044-1066           | SelectorIterator::GetNextElement() |
| 继承链定义            | workspotTreeItems.cpp | 272, 603, 888, 1033 | 类定义                                |

### 七、可视化流程图



```mermaid
graph TD
    A[Selector.Next() 被调用] --> B[SelectorIterator::GetNextElement()]
    B -->|"1. 随机选择一个 Sequence<br>2. 预先检查 transition"| C[DetermineTransitionAnim()]
    C -->|"├─ 查找自定义 transition？<br>└─ 生成标准命名 transition"| D{找到动画？}
    D -->|是| E[IdleGuard::Next()]
    D -->|否| F{是 Selector？}
    F -->|否| G[直接 blend可能穿模)]
    F -->|是| H[用基础 idle 包裹]
    H --> E
    E -->|"检测到 idle 变化m_state = PerformTransition"| I[IdleGuard::GetData()]
    I -->|"返回 transition 动画数据"| J[系统播放 transition 动画]
```

> （注：文档部分内容可能由 AI 生成）