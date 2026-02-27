# SectionNode 快速参考卡

> 30秒速查：SectionNode是什么、解决什么、怎么用

---

## ⚡ 一句话总结

**SectionNode是场景图中的"文件夹"节点，将大场景拆分为可管理的逻辑段落。**

---

## 🎯 核心价值（Why）

| 问题 | 痛点指数 | SectionNode解决 |
|------|---------|----------------|
| 场景有450+节点，编辑器打开像迷宫 | 🔥🔥🔥🔥🔥 | 分组折叠，一眼看清 |
| 4个设计师编辑同一场景，每天冲突3-5次 | 🔥🔥🔥🔥🔥 | 独立文件，零冲突 |
| 中断时需要保存5-10KB状态数据 | 🔥🔥🔥🔥 | 只保存200字节Token |
| 内存占用500MB，低端机卡顿 | 🔥🔥🔥🔥 | 按需加载，降80% |
| 调整某段节奏需改100+事件时间戳 | 🔥🔥🔥 | Section独立时间轴 |
| 定位bug需要30分钟 | 🔥🔥🔥 | 3分钟精确定位 |

---

## 📦 本质定义（What）

```cpp
class SectionNode : public SceneGraphNode {
    // 是什么：场景图中的逻辑分组容器

    // 包含什么：
    red::DynArray< THandle< SceneEvent > > m_events;      // 事件列表
    red::DynArray< ActorBehavior > m_actorBehaviors;      // 演员管理
    SceneTime m_refrncDuration;                           // 时长

    // 有什么插座：
    InputSocket::in        // 进入Section
    InputSocket::cancel    // 取消执行
    OutputSocket::out      // 正常完成
    OutputSocket::cancelFwd // 转发取消
};
```

**类比理解：**
- SceneGraph = 一本书
- SectionNode = 章节
- 其他节点 = 段落/句子

---

## 🔧 核心能力（How）

### 1. 逻辑分组
```
q110_scene.scene
├─ Section 1: "church" [10秒]
├─ Section 2: "truck" [30秒]
├─ Section 3: "mall" [90秒]
└─ Section 4: "underground" [40秒]
```

### 2. 独立时间轴
```cpp
Section 3内部时间：0 → 90秒
可以独立缩放：scaleFactor = 1.5x
不影响其他Section
```

### 3. 局部演员管理
```cpp
Section 1: actorBehaviors = [Placide, 5个信徒]
Section 2: actorBehaviors = [Placide, Johnny]
Section切换时自动加载/卸载
```

### 4. 状态Token化
```cpp
中断时保存：
{
  sectionId: "mall",
  timePos: 45.2秒,
  vars: { choice: "A" }
}
恢复时Section自动重建内部状态
```

### 5. 资源批量管理
```cpp
Section.GetEvents() → 所有事件
遍历事件 → 提取资源引用
智能预加载下一个Section
```

---

## 📊 量化对比

| 指标 | 无Section | 有Section | 倍数 |
|------|----------|-----------|------|
| 场景复杂度上限 | 100节点 | 1000+节点 | 10x |
| 团队并行能力 | 2人 | 4-6人 | 2-3x |
| 迭代测试速度 | 5分钟 | 10秒 | 30x |
| 内存占用 | 500MB | 80MB | 0.16x |
| Git冲突频率 | 3-5/天 | 0-1/天 | 0.2x |
| Bug定位时间 | 30分钟 | 3分钟 | 10x |

---

## 💡 设计模式

### 模式1：线性叙事
```
[Start] → Section1 → Section2 → Section3 → [End]
用于：主线剧情、教程
```

### 模式2：分支叙事
```
[Start] → Section1 → ChoiceNode ┬→ Section2A → [End]
                                 └→ Section2B → [End]
用于：对话选择、多结局
```

### 模式3：可跳过段落
```
[Start] → Section1 (optional) → Section2
              ↓ cancel
             Section2
用于：支线对话、可选剧情
```

### 模式4：循环段落
```
Section1 → Section2 → LoopCheck ┬→ Section3 (继续)
                          ↑      └→ Section2 (重试)
                          └─────────┘
用于：战斗失败重试、小游戏
```

---

## 🚨 常见错误

### ❌ 错误1：Section粒度过细
```
❌ Section 1: "NPC说第1句话" [2秒]
❌ Section 2: "NPC说第2句话" [3秒]
❌ Section 3: "NPC说第3句话" [2秒]

问题：切换开销大于收益

✅ 正确：
Section 1: "opening_dialogue" [7秒]
  包含3行对话
```

**指导原则：** 一个Section = 一个"有意义的叙事单元"（10-120秒）

### ❌ 错误2：Section间强耦合
```
❌ Section 1设置变量 var1 = 5
   Section 2依赖 var1 的具体值

问题：Section 1修改后Section 2可能崩溃

✅ 正确：
   通过事件/信号通信，不依赖内部变量
```

### ❌ 错误3：忘记管理演员
```
❌ Section定义了事件，但没有在actorBehaviors中列出演员

问题：运行时找不到演员，事件无法执行

✅ 正确：
   Section的actorBehaviors必须包含所有涉及的NPC
```

---

## 🎓 最佳实践

### 1. Section命名规范
```
✅ "church_greeting"       (地点_行为)
✅ "truck_ride_dialogue"   (地点_行为_内容)
✅ "mall_combat_phase1"    (地点_行为_阶段)

❌ "section_01"            (无意义)
❌ "stuff"                 (不清晰)
❌ "test"                  (临时名)
```

### 2. Section时长建议
```
最小值：5秒   (太短没意义)
推荐值：10-60秒 (一个完整场景)
最大值：120秒  (超过需拆分)
特殊情况：战斗Section可以更长
```

### 3. 演员绑定策略
```cpp
// 核心NPC：在所有Section中保持加载
actorBehaviors = [
    { actorId: "main_npc", persistent: true }
]

// 临时NPC：按需加载
actorBehaviors = [
    { actorId: "shopkeeper", loadOnDemand: true }
]
```

### 4. 资源预加载策略
```
当前Section执行时：
  ✅ 预加载下一个Section的高优先级资源（语音）
  ✅ 预加载可能分支的中优先级资源
  ❌ 不要预加载太远的Section（内存浪费）
```

---

## 🔍 调试技巧

### 技巧1：独立测试Section
```
编辑器 → 右键Section → "Test in Isolation"
直接跳转到这个Section测试，无需从头开始
```

### 技巧2：时间点跳转
```cpp
// 跳转到Section内的特定时间
editor->JumpToSection("mall_combat", 45.5秒);
快速定位问题时间点
```

### 技巧3：状态检查
```cpp
// 打印Section状态
LOG("Current Section: %s", currentSection->GetName());
LOG("Time Position: %.2f", currentSection->GetTimePos());
LOG("Active Actors: %d", currentSection->GetActorCount());
```

### 技巧4：性能分析
```cpp
// 测量Section性能
ScopedTimer timer("Section_mall_combat");
section->Execute();
// 输出：Section_mall_combat: 2.3ms
```

---

## 📚 进阶主题

### 主题1：嵌套Section（不推荐）
```
Section A
  └─ Sub-Section A1
  └─ Sub-Section A2

问题：增加复杂度，建议扁平化设计
```

### 主题2：动态Section生成
```cpp
// 根据玩家选择动态创建Section
if (playerChoice == "aggressive") {
    nextSection = CreateSection("combat_path");
} else {
    nextSection = CreateSection("stealth_path");
}
```

### 主题3：Section池复用
```cpp
// 对于重复场景（如NPC招呼），使用Section池
SectionPool greetingPool;
auto greeting = greetingPool.Acquire("generic_greeting");
// 使用完毕
greetingPool.Release(greeting);
```

---

## 🎯 快速决策树

```
我应该创建新Section吗？
│
├─ 这段叙事>10秒？
│  ├─ NO → 合并到现有Section
│  └─ YES → 继续
│
├─ 涉及新的NPC？
│  ├─ YES → 创建新Section ✓
│  └─ NO → 继续
│
├─ 需要独立调试？
│  ├─ YES → 创建新Section ✓
│  └─ NO → 继续
│
├─ 多人协作边界？
│  ├─ YES → 创建新Section ✓
│  └─ NO → 继续
│
└─ 时间轴需要独立控制？
   ├─ YES → 创建新Section ✓
   └─ NO → 可以合并到现有Section
```

---

## 🔗 相关资源

- **核心痛点详解**: `SectionNode核心痛点与解决方案.md`
- **代码入口**: `dev/src/common/gameSceneSystem/src/scnsSectionNode.h`
- **系统文档**: `InteractiveScene系统深度解析.md`

---

## ⚡ 30秒记忆口诀

```
Section是分组，不是节点
时间独立，演员局部
状态Token，资源批量
协作无冲，调试高效
```

---

*快速参考卡 v1.0*
*打印此页贴在显示器旁*
