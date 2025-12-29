● 非常好的观察！全局数据系统确实容易成为性能瓶颈。让我为你总结CDPR在Fact数据库中采用的核心优化策略：

  关键性能优化策略

  1. 分表设计 - 避免全局锁瓶颈

  FactsDB (无锁)
  ├── Table 1 (USER_TABLE) → 独立LightMutex
  ├── Table 2 (DELAY) → 独立LightMutex
  └── Table 10 (FLAGS) → 独立LightMutex

  好处：不同表可以并行访问，避免全局锁争用。大部分任务只操作USER_TABLE，其他系统表互不干扰。

  ---
  2. 轻量级锁 + 极短临界区

  Value Get(ValueID vid) const {
      RED_SCOPE_LOCK(m_lock);  // 仅在这一行持锁
      return m_map.Find(vid, v) ? v : 0;
  }

  优化点：
  - 使用 LightMutex（轻量级互斥锁），开销比标准mutex小
  - 临界区只包含Map查找，通常<100ns
  - 不使用读写锁（RWLock开销更大，读写比例不适合）

  ---
  3. 字符串哈希 - 避免重复计算

  // 初始化时
  FactID factId("q115_completed");  // "q115_completed" → Hash(0x12345678)

  // 运行时
  Get(factId);  // 直接用Uint32查找，无字符串操作

  性能提升：
  - 字符串→Hash只在创建时执行一次
  - 后续查找全是整数运算
  - Hash冲突极低（32位空间）

  ---
  4. 延迟创建 + 智能指针

  // 回调数组只在需要时创建
  red::Map<ValueID, UniquePtr<DynArray<Callback>>> m_callbacks;

  RegisterCallback(vid, func) {
      if (!m_callbacks[vid])  // 没回调就不创建
          m_callbacks[vid] = CreateArray();
  }

  节省内存：大部分Fact没有监听器，避免为每个Fact预分配数组。

  ---
  5. 内存池 - 减少碎片化

  class FactsTable {
      RED_USE_MEMORY_POOL(PoolGMPL_QuestFDB);
      red::Map<...> m_map{PoolGMPL_QuestFDB()};
  };

  好处：
  - 预分配大块内存，避免频繁malloc/free
  - 减少内存碎片
  - 更好的缓存局部性

  ---
  6. 原子操作 - 避免竞态条件

  // 错误做法（竞态）
  int val = Get("kills");
  val++;
  Set("kills", val);  // 可能丢失其他线程的更新！

  // 正确做法（原子）
  Do("kills", [](int& v) { v++; });  // 读-改-写原子执行

  关键：Do() 在一次锁操作内完成"读取→修改→写入→触发回调"，保证一致性。

  ---
  7. 批量处理 - 减少锁竞争

  // 游戏循环每帧调用
  void QuestSystem::Tick() {
      // 收集本帧所有Fact变化
      ProcessEvents();  // 持锁很短

      // 批量处理回调（无锁）
      TriggerCallbacks();
  }

  优化：通过缓冲区交换技术，减少锁获取次数。

  ---
  8. Map vs HashMap 的权衡

  CDPR选择 red::Map（红黑树）而非 HashMap 的原因：

  | 特性        | Map (红黑树) | HashMap        |
  |-------------|--------------|----------------|
  | 查找        | O(log n)     | O(1)           |
  | 内存        | 节点开销小   | 需要bucket数组 |
  | Cache友好性 | 中等         | 差（跳跃访问） |
  | 迭代        | 有序，快     | 无序，慢       |

  结论：Fact数量通常在数百~数千级别，O(log n)依然很快（<10次比较），而Map更省内存且缓存友好。

  ---
  9. 热点数据监控

  struct PerformanceMonitor {
      CircularBuffer<PerformanceNodeData> m_nodeDataBuffer;
      // 记录最慢的任务节点
  };

  实时优化：通过性能监控识别热点Fact，可以：
  - 发现不合理的高频访问
  - 优化任务逻辑设计
  - 调试死锁和卡顿问题

  ---
  性能指标

  根据代码分析，典型操作的性能：

  | 操作             | 耗时估算     | 说明             |
  |------------------|--------------|------------------|
  | Get              | 50-200ns     | 锁+红黑树查找    |
  | Set              | 100-500ns    | 锁+查找+回调     |
  | RegisterCallback | 200-1000ns   | 锁+数组创建      |
  | Do (无回调)      | 100-300ns    | 原子修改         |
  | Do (有回调)      | 视回调复杂度 | 可能触发连锁反应 |

  ---
  关键设计哲学

  CDPR的优化思路是：

  1. 避免过度工程：不用RWLock、无缓存层（直接访问），保持简单
  2. 分而治之：多表设计降低锁粒度
  3. 按需分配：回调数组延迟创建
  4. 预计算优化：字符串Hash预计算
  5. 原子保证：Do() 确保一致性

  这种设计在中等并发、高频读写的任务系统中表现优秀。游戏中同时活跃的任务有限（通常<50个），Fact访问频率虽高但不至于成为瓶颈。

  ---
  可能的瓶颈

  需要注意的性能风险：

  1. 回调风暴：一个Fact触发大量回调，每个回调又触发其他Fact
  2. 锁可重入开销：LightMutex需要支持可重入（回调中访问Fact）
  3. 保存/加载：遍历所有Fact，可能有卡顿

  但从代码看，CDPR通过任务设计规范避免了这些问题。