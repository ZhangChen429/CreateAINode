# Workspot 核心机制：嵌套迭代器递归

---

## 一、关键数据结构

`ContainerIterator` 通过 `m_nestedIter` 递归持有子迭代器，形成迭代器调用链——每一层只管理自己的遍历逻辑。

```mermaid
graph TD
    subgraph ContainerIterator
        MI["m_nestedIter\nred::UniquePtr&lt;EntryIterator&gt;\n当前活跃的子迭代器"]
        CE["m_contEntry\nconst IContainerEntry*\n指向容器本身"]
        IX["m_index\nInt32\n当前遍历索引"]
    end

    subgraph 迭代器层次结构
        BASE["EntryIterator（抽象基类）"]
        LEAF["BaseAnimClipIterator\n├─ AnimClipIterator\n└─ MotionAnimIterator"]
        CONT["ContainerIterator\n├─ SequenceIterator\n├─ RandomListIterator\n└─ SelectorIterator"]
        BASE --> LEAF
        BASE --> CONT
    end

    subgraph 节点层次结构
        IENTRY["IEntry（统一接口）\n+ CreateIterator()\n+ ContainEntry()"]
        LEAF2["叶子节点\nAnimClip / EntryAnim / ExitAnim"]
        CONT2["容器节点\nSequence / Selector / RandomList"]
        IENTRY --> LEAF2
        IENTRY --> CONT2
    end

    CONT -- "管理" --> MI
```

```cpp
// workspotTreeItems.cpp:603
class ContainerIterator : public IdleGuard<IContainerEntry, EntryIterator> {
    red::UniquePtr<EntryIterator> m_nestedIter;  // 当前活跃的子迭代器
    const IContainerEntry* m_contEntry;          // 指向容器本身
    Int32 m_index;                               // 当前遍历索引
};
```

---

## 二、GetData 的层层委托机制

### 委托调用链

```mermaid
flowchart TD
    A["RootSequenceIterator::GetData()"]
    A1["outData.m_idleAnimName = 'stand'"]
    B["SelectorIterator::GetData()"]
    B1["outData.m_idleAnimName = 'stand'（覆盖）"]
    C["InnerSequenceIterator::GetData()"]
    C1["outData.m_idleAnimName = 'sit'（最终值）"]
    D["AnimClipIterator::GetData()"]
    D1["outData.m_animationName = 'sit_eat_food'\noutData.m_blendTime = 0.5\noutData.m_entryFlags = Animation"]

    A --> A1 --> B
    B --> B1 --> C
    C --> C1 --> D
    D --> D1

    RESULT["最终 outData\nm_animationName: sit_eat_food\nm_idleAnimName: sit\nm_blendTime: 0.5"]
    D1 --> RESULT
```

```cpp
// workspotTreeItems.cpp:616-631
virtual void GetData( WorkspotEntryData& outData ) override {
    TBaseClass::GetData( outData );  // 1. 父类处理过渡动画

    if ( !IsTransitionActive() ) {
        if ( m_contEntry->m_idleAnim ) {
            outData.m_idleAnimName = m_contEntry->m_idleAnim;  // 2. 填充 idle
        }
        if ( m_nestedIter ) {
            m_nestedIter->GetData( outData );  // 3. ⭐ 委托给子迭代器
        }
    }
}
```

> **层层覆盖原则**：外层容器的 `m_idleAnim` 会被内层覆盖；外层容器负责姿态过渡检测（IdleGuard）；最终返回的是最内层叶子节点的动画数据。

---

## 三、Next 的递归推进机制

```mermaid
flowchart TD
    START["ContainerIterator::Next()"] --> TA{IsTransitionActive?}
    TA -->|是| RET["return（等待过渡完成）"]
    TA -->|否| LOOP["进入 while 循环\nm_index < m_maxCount"]

    LOOP --> NI{m_nestedIter\n存在且有效?}

    NI -->|是| RN["m_nestedIter->Next()\n递归调用子迭代器"]
    RN --> NV{子迭代器\n仍然有效?}
    NV -->|是| BREAK["break（返回上层）"]
    NV -->|否| NEXT["子迭代器耗尽\n推进 m_index"]

    NI -->|否| NEXT
    NEXT --> OB{m_index >= maxCount?}
    OB -->|是| DONE["break（容器遍历完成）"]
    OB -->|否| GET["GetNextElement(m_index)\n获取下一个子 Entry"]
    GET --> NULL{entry 为空?}
    NULL -->|是| LOOP
    NULL -->|否| CREATE["entry->CreateIterator()\n⭐ 工厂模式创建子迭代器"]
    CREATE --> INIT["m_nestedIter->Next()\n立即推进子迭代器"]
    INIT --> VALID{子迭代器有效?}
    VALID -->|是| BREAK
    VALID -->|否| LOOP
```

### 时间线演示

```mermaid
sequenceDiagram
    participant OUT as OuterSequence
    participant IN1 as InnerSequence1
    participant IN2 as InnerSequence2
    participant AC1 as AnimClip1
    participant AC2 as AnimClip2
    participant AC3 as AnimClip3

    Note over OUT: T=0: 首次 Next()
    OUT->>OUT: m_index=0，nestedIter 不存在
    OUT->>IN1: CreateIterator + Next()
    IN1->>AC1: CreateIterator + Next()
    AC1-->>OUT: Active → 播放 anim1

    Note over OUT: T=1: anim1 播放完
    OUT->>IN1: nestedIter->Next()（递归）
    IN1->>IN1: m_index=0 耗尽 → m_index=1
    IN1->>AC2: CreateIterator + Next()
    AC2-->>OUT: Active → 播放 anim2

    Note over OUT: T=2: anim2 播放完
    OUT->>IN1: nestedIter->Next()（递归）
    IN1->>IN1: m_index=1 耗尽，IsValid=false
    OUT->>OUT: nestedIter 无效，m_index=1
    OUT->>IN2: CreateIterator + Next()
    IN2->>AC3: CreateIterator + Next()
    AC3-->>OUT: Active → 播放 anim3

    Note over OUT: T=3: anim3 播放完
    OUT->>IN2: nestedIter->Next()（递归）
    IN2->>IN2: m_index 耗尽，IsValid=false
    OUT->>OUT: nestedIter 无效，m_index=2 >= maxCount
    OUT-->>OUT: IsValid=false（整体遍历完成）
```

---

## 四、IsValid 的递归验证

```mermaid
flowchart TD
    A["OuterSequence::IsValid()"]
    A --> B{"m_index < m_maxCount ?"}
    B -->|是| TRUE1["✅ true（容器自己还有元素）"]
    B -->|否| C["检查 m_nestedIter->IsValid()"]
    C --> D["InnerSequence::IsValid()"]
    D --> E{"m_index < m_maxCount ?"}
    E -->|是| TRUE2["✅ true"]
    E -->|否| F["检查 m_nestedIter->IsValid()"]
    F --> G["AnimClip::IsValid()"]
    G --> H{"m_stage == Active ?"}
    H -->|是| TRUE3["✅ true"]
    H -->|否| FALSE["❌ false（整条链失效）"]
```

```cpp
// workspotTreeItems.cpp:643-646
virtual Bool IsValid( const CheckConditionContext& context ) const override {
    return ( m_index < m_maxCount ) ||
           ( m_nestedIter && m_nestedIter->IsValid( context ) );
}
```

---

## 五、GoTo 的递归跳转

```mermaid
flowchart TD
    A["GoTo(AnimClip.id)"] --> B{"m_pointedEntry->id\n== 目标 id ?"}
    B -->|是| C["跳转到容器本身\nm_index=-1，nestedIter 重置\nNext()"]
    B -->|否| D["遍历 m_list\n查找包含目标 id 的子 Entry"]
    D --> E{"handle->ContainEntry(id) ?"}
    E -->|否| D
    E -->|是| F["创建子迭代器\nm_nestedIter = handle->CreateIterator()"]
    F --> G["m_nestedIter->GoTo(id)\n递归跳转！"]
    G --> H["OuterSequence 的 OnGoTo(i)\n记录当前索引"]
    G --> I["InnerSequence::GoTo()\n继续递归"]
    I --> J["AnimClip::GoTo()\nm_stage = Active"]
```

```cpp
// workspotTreeItems.cpp:658-682
virtual void GoTo( WorkEntryId id, const EntryIterationContext& context ) override {
    if ( m_pointedEntry->m_id == id ) {
        m_index = -1;
        m_nestedIter.Reset();
        Next( context );
        return;
    }
    for ( Uint32 i = 0; i < listSize; ++i ) {
        if ( handle->ContainEntry( id ) ) {          // 递归查找
            m_nestedIter.Reset( handle->CreateIterator( context ) );
            m_nestedIter->GoTo( id, context );       // 递归跳转！
            OnGoTo( i );
            return;
        }
    }
}
```

---

## 六、完整案例：四层嵌套

### 迭代器栈结构

```mermaid
graph TD
    subgraph "T=1: 播放 walk_to_spot"
        R1["RootSequenceIterator\nm_index=0"]
        E1["EntryAnimIterator\nm_stage=Active"]
        R1 -->|m_nestedIter| E1
    end

    subgraph "T=2: Selector 选中分支B，播放 sit_eat"
        R2["RootSequenceIterator\nm_index=1"]
        S2["SelectorIterator\nm_index=1（分支B）"]
        IS2["InnerSequenceIterator\nm_idleAnim=sit\nm_index=0"]
        AC2["AnimClipIterator\nm_stage=Active\nsit_eat"]
        R2 -->|m_nestedIter| S2
        S2 -->|m_nestedIter| IS2
        IS2 -->|m_nestedIter| AC2
    end

    subgraph "T=3: 播放 walk_away"
        R3["RootSequenceIterator\nm_index=2"]
        EX3["ExitAnimIterator\nm_stage=Active"]
        R3 -->|m_nestedIter| EX3
    end
```

### GetData 调用链与最终结果

```mermaid
sequenceDiagram
    participant R as RootSequenceIterator
    participant S as SelectorIterator
    participant IS as InnerSequenceIterator
    participant AC as AnimClipIterator

    Note over R,AC: GetData() 调用链（播放 sit_eat 时）
    R->>R: outData.m_idleAnimName = "stand"
    R->>S: m_nestedIter->GetData()
    S->>S: outData.m_idleAnimName = "stand"（覆盖）
    S->>IS: m_nestedIter->GetData()
    IS->>IS: outData.m_idleAnimName = "sit"（最终值）
    IS->>AC: m_nestedIter->GetData()
    AC->>AC: outData.m_animationName = "sit_eat"
    AC->>AC: outData.m_blendTime = 0.5
    AC-->>R: 返回完整 outData

    Note over R,AC: 最终结果：animationName=sit_eat / idleAnimName=sit
```

---

## 七、设计模式总结

### 组合模式 + 迭代器模式

```mermaid
graph LR
    subgraph 组合模式（树形结构）
        IE["IEntry\n+ CreateIterator()\n+ ContainEntry()"]
        LEAF3["叶子节点\nAnimClip\nEntryAnim\nExitAnim"]
        CONT3["容器节点\nSequence\nSelector\nRandomList"]
        IE --> LEAF3
        IE --> CONT3
        CONT3 -->|"持有"| LEAF3
        CONT3 -->|"嵌套"| CONT3
    end

    subgraph 迭代器模式（遍历逻辑）
        EI["EntryIterator\n+ Next()\n+ GetData()\n+ IsValid()\n+ GoTo()"]
        LI["叶子迭代器\nAnimClipIterator\nMotionAnimIterator"]
        CI["容器迭代器\nSequenceIterator\nSelectorIterator\n持有 m_nestedIter"]
        EI --> LI
        EI --> CI
        CI -->|"递归持有"| CI
        CI -->|"递归持有"| LI
    end
```

### 核心优势对比

```mermaid
flowchart LR
    subgraph "❌ 简单方案：平铺数组"
        F1["无法表达逻辑关系\nSelector 随机选择"]
        F2["无法表达姿态切换\nstand → sit"]
        F3["无法复用子行为"]
        F4["无法条件过滤"]
    end

    subgraph "✅ 递归方案：嵌套迭代器"
        G1["表达力强\n逻辑嵌套自然"]
        G2["自动姿态管理\nidle 层层覆盖"]
        G3["子行为复用\n统一 IEntry 接口"]
        G4["惰性求值\n节省内存"]
        G5["无限扩展性\n任意深度嵌套"]
    end
```

| 特性 | 实现方式 | 优势 |
|---|---|---|
| 无限嵌套 | 递归迭代器链 | 支持任意深度嵌套 |
| 统一接口 | 所有迭代器实现 Next/GetData/IsValid | 上层代码无需知道嵌套结构 |
| 惰性求值 | 只在 Next 时才创建子迭代器 | 节省内存，支持动态结构 |
| 自动清理 | UniquePtr 自动管理生命周期 | 无内存泄漏 |
| 精确跳转 | GoTo 递归查找 | 支持跳转到任意深度节点 |

---

## 八、关键洞察

```mermaid
graph TD
    subgraph 职责分层
        OUTER["外层容器\n管理执行流程\n顺序 / 随机 / 条件选择"]
        INNER["内层容器\n管理姿态声明\nidle 覆盖传递"]
        LEAF4["叶子节点\n提供具体动画\nAnimClip 数据"]
        OUTER -->|"委托 GetData"| INNER
        INNER -->|"委托 GetData"| LEAF4
    end

    subgraph GetData 最终返回
        RES["动画数据（来自叶子）\n+ idle 信息（来自最内层容器）"]
        LEAF4 --> RES
        INNER --> RES
    end
```

> **核心原则**：容器不返回动画，只负责组织逻辑。`GetData` 永远委托到叶子节点，返回"叶子节点的动画 + 最内层容器的 idle"。

通过**递归迭代器 + 层层委托**，Workspot 实现了无限嵌套与统一接口的完美结合，是教科书级别的组合模式应用。
