  直接回答

  WorkspotInstance 被废弃与 External/Internal Workspot 的存在和构成 ❌ 没有关系。

  这是两个完全独立的维度：

  维度对比

  维度 1：存储方式（External vs Internal）

  关注问题：Workspot 定义的数据存在哪里？
  - External: 引用外部 .workspot 文件
  - Internal: 数据嵌入 .scenesolution 文件

  影响对象：SceneWorkspot, WorkspotData
  时间轴：一直存在，从未改变

  维度 2：实例化方式（旧 vs 新）

  关注问题：如何在场景中使用 Workspot？
  - 旧方式: SceneWorkspotInstance (单独的实例对象)
  - 新方式: ChangeWorkEvent (事件即实例)

  影响对象：场景节点、事件系统
  时间轴：2019年4月废弃旧方式

  证据

  1. 两者可以任意组合

  | 存储方式 | +   | 实例化方式       | 结果        |
  |----------|-----|------------------|-------------|
  | External | +   | WorkspotInstance | ❌ 已废弃   |
  | External | +   | ChangeWorkEvent  | ✅ 当前使用 |
  | Internal | +   | WorkspotInstance | ❌ 已废弃   |
  | Internal | +   | ChangeWorkEvent  | ✅ 可以使用 |

  2. 废弃原因与存储无关

  WorkspotInstance 被废弃的真正原因：
  // scnEditorResource.h:53
  VER_4, // 2019, Apr 15: New SceneSpot implementation
                            (get rid of SceneWorkspotInstance).
         // Add ChangeWorkEvent and StopWorkEvent.

  废弃理由：
  1. ❌ 架构冗余：定义 → 实例 → 事件（三层太多）
  2. ✅ 简化为：定义 → 事件（两层足够）
  3. 事件本身就包含所有实例信息（位置、参数等）

  与 External/Internal 完全无关！

  3. External/Internal 在废弃前后都存在

  2019年4月之前：
  ├─ SceneWorkspot (External/Internal) ✅ 存在
  ├─ SceneWorkspotInstance ✅ 存在
  └─ 事件系统 ✅ 存在

  2019年4月之后：
  ├─ SceneWorkspot (External/Internal) ✅ 仍然存在（未改变）
  ├─ SceneWorkspotInstance ❌ 废弃
  └─ ChangeWorkEvent/StopWorkEvent ✅ 新增（替代 WorkspotInstance）

  类比说明

  这就像：

  维度 1（External vs Internal）：
  餐厅菜谱存在哪里？
  - External: 查阅总部统一菜谱书
  - Internal: 厨师自己记在小本子上

  维度 2（WorkspotInstance 废弃）：
  如何点菜？
  - 旧方式: 先预定桌位 → 再点菜 → 再上菜
  - 新方式: 点菜时直接包含桌位信息

  废弃原因：预定桌位这一步是多余的

  菜谱存储位置（External/Internal）与点菜流程（实例化方式）没有关系！

  最终答案

  WorkspotInstance 被废弃的原因：
  ✅ 架构简化（三层变两层）
  ✅ 减少冗余管理
  ✅ 事件本身可以包含实例信息

  与 External/Internal 的关系：
  ❌ 完全无关
  ❌ 两者是正交的独立概念
  ❌ External/Internal 在废弃前后都没有改变

  简单说：WorkspotInstance 被废弃是因为不需要单独的"实例"对象，和 Workspot 数据存在哪里（External/Internal）没有任何关系。
                                                                                                                                 