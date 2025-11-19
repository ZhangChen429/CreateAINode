  IExecutableItem 是什么？

  IExecutableItem 是 Scene系统（过场动画/剧情系统）的原子执行单元，类似于设计模式中的 Command Pattern（命令模式）。

  核心概念

  // scnsExecutableItem.h:9-56
  class IExecutableItem
  {
      // 生命周期状态
      enum class ExecutionState
      {
          Created,     // 初始状态
          Prepared,    // 准备完成（仅调用一次）
          Executing,   // 正在执行（持续时间型）
          Executed,    // 执行完成
          Sustained,   // 执行完成但需要保持状态
          Cancelled,   // 已取消
      };

      // 执行结果
      enum class Resolution
      {
          InProgress,  // 仍在进行
          Finished,    // 完成
          Sustains,    // 需要维持状态
      };

      // 核心接口
      virtual Resolution DoExecute() = 0;  // 必须实现
      virtual Continuation DoPrepare();    // 可选
      virtual void DoCancel();             // 可选
  };

  ---
  为什么重要？

  IExecutableItem 是 Scene 系统控制游戏世界的操作指令，每一个 ExecutableItem 代表：

  - 一个动作（播放动画、LookAt、IK）
  - 一个状态改变（设置AI为Cinematic模式、改变外观）
  - 一个场景操作（传送NPC、装备武器、挂载载具）
  - 一个系统调用（管理Facts、配置摄像机、控制UI）

  类比理解

  Scene系统 = 导演
  IExecutableItem = 剧组工作人员的具体任务清单

  导演说："播放这个过场动画"
      ↓
  生成多个 ExecutableItem：
  - ExecutableItem_SetCinematicMode (把AI设为Cinematic)
  - ExecutableItem_UseWorkspot (让NPC坐到座位上)
  - ExecutableItem_AnimationMotion (播放对话动画)
  - ExecutableItem_LookAt (让NPC看向玩家)
  - ExecutableItem_AICommand (发送特殊指令)
  - ExecutableItem_ConfigureCamera (配置摄像机)

  ---
  IExecutableItem 的子类生态（30+种类型）

  从我找到的代码可以看到，Scene系统有 30+ 种 ExecutableItem：

  1. AI控制类（最重要）

  | 类名                              | 作用               | 文件位置                                    |
  |---------------------------------|------------------|-----------------------------------------|
  | ExecutableItem_SetCinematicMode | 设置AI为Cinematic模式 | scnsExecutableItem_CinematicMode.cpp:17 |
  | ExecutableItem_AICommand        | 向AI发送命令          | scnsExecutableItem_AICommand.cpp:139    |
  | ExecutableItem_SetSceneTier     | 设置StoryTier      | scnsExecutableItem_SetSceneTier.cpp:42  |

  2. 动画和动作类

  | 类名                              | 作用          |
  |---------------------------------|-------------|
  | ExecutableItem_AnimationMotion  | 播放动画        |
  | ExecutableItem_ActionRootMotion | 播放根运动动画     |
  | ExecutableItem_UseWorkspot      | 使用工作位（坐、站等） |
  | ExecutableItem_LookAt           | 控制视线        |
  | ExecutableItem_IK               | 反向动力学控制     |
  | ExecutableItem_ChangeFacialIdle | 改变面部表情      |

  3. 实体管理类

  | 类名                                 | 作用     |
  |------------------------------------|--------|
  | ExecutableItem_Teleport            | 传送实体   |
  | ExecutableItem_AttachToEntity      | 附加到实体  |
  | ExecutableItem_AppearanceChange    | 改变外观   |
  | ExecutableItem_EquipItem           | 装备物品   |
  | ExecutableItem_Mount               | 挂载载具   |
  | ExecutableItem_ActivateSceneEntity | 激活场景实体 |

  4. 摄像机和视觉类

  | 类名                                        | 作用      |
  |-------------------------------------------|---------|
  | ExecutableItem_ConfigureCamera            | 配置摄像机   |
  | ExecutableItem_PlayerLookAt               | 控制玩家视角  |
  | ExecutableItem_ToggleComponentsVisibility | 切换组件可见性 |

  5. 系统管理类

  | 类名                         | 作用            |
  |----------------------------|---------------|
  | ExecutableItem_ManageFacts | 管理Fact系统      |
  | ExecutableItem_Community   | 管理Community系统 |

  ---
  ExecutableItem 的执行流程

  调度器：SideeffectsExecutor

  Scene系统使用 SideeffectsExecutor 来管理 ExecutableItem：

  // scnsSideeffectsExecutor.h:19-69
  class SideeffectsExecutor
  {
      struct ProcessingItems
      {
          red::DynArray< red::UniquePtr< IExecutableItem > > m_executables;
          red::DynArray< red::UniquePtr< IExecutableItem > > m_sustainedExecutables;
      };

      void Prepare( const SideeffectsSnapshot& snapshot );
      Bool Execute();  // 调度执行所有Items
      void Cancel();
  };

  执行生命周期

  1. Scene开始播放
      ↓
  2. 生成 SideeffectsSnapshot（快照）
      ↓
  3. 从快照创建所有需要的 ExecutableItem
      ↓
  4. SideeffectsExecutor::Prepare()
      ├─ 调用每个Item的 DoPrepare()
      └─ 进入 Executing 状态
      ↓
  5. 每帧调用 SideeffectsExecutor::Execute()
      ├─ 遍历所有 m_executables
      ├─ 调用每个Item的 DoExecute()
      ├─ 检查返回值：
      │   ├─ Resolution::Finished → 移除Item
      │   ├─ Resolution::InProgress → 保留继续执行
      │   └─ Resolution::Sustains → 移到 m_sustainedExecutables
      └─ 管理 sustained Items
      ↓
  6. Scene结束或中断
      └─ 调用所有Item的 DoCancel()

  ---
  实际案例：ExecutableItem_SetCinematicMode

  让我们看一个具体的例子：

  // scnsExecutableItem_CinematicMode.cpp:17-31
  IExecutableItem::Resolution ExecutableItem_SetCinematicMode::DoExecute()
  {
      // 1. 找到目标NPC
      if (auto target = FindTarget<game::Puppet>())
      {
          // 2. 获取AI组件
          if (AI::CAgent *agent = target->GetAI())
          {
              // 3. 检查当前StoryTier
              if(agent->GetStoryTier() != m_params.m_aiTier)
              {
                  // 4. 设置新的StoryTier（通常是Cinematic）
                  agent->SetStoryTier(m_params.m_aiTier);
              }
          }
      }
      // 5. 立即完成（一次性操作）
      return Resolution::Finished;
  }

  这个 ExecutableItem 做了什么？

  1. 找到Scene中指定的NPC
  2. 修改其 StoryTier 为 Cinematic
  3. 触发AI行为树的条件判断
  4. AI行为树切换到 Story 分支
  5. Scene系统接管控制

  ---
  ExecutableItem_AICommand：Scene向AI发送命令

  这是另一个关键的子类：

  // scnsExecutableItem_AICommand.cpp:139-175
  IExecutableItem::Resolution ExecutableItem_AICommand::DoExecute()
  {
      if (!m_commandPtr)
          return Resolution::Finished;

      const Target target = FindTarget();

      // 检查命令状态
      Bool commandEnqueued = m_commandPtr->GetState() == AI::CommandState::Enqueued;
      Bool commandExecuting = m_commandPtr->GetState() == AI::CommandState::Executing;

      // 如果命令还在队列或执行中，保持等待
      if (!commandEnqueued && !commandExecuting)
      {
          m_commandPtr->SetListener(nullptr);
          m_commandPtr.Reset();
          return Resolution::Finished;  // 命令完成
      }

      return Resolution::InProgress;  // 继续等待命令完成
  }

  工作原理：

  1. Scene创建 ExecutableItem_AICommand
  2. Item内部创建具体的 AICommand（如 MoveTo, PlayAnimation等）
  3. 将命令发送到NPC的 CommandQueue
  4. 每帧检查命令是否完成
  5. 命令完成后，ExecutableItem 返回 Finished

  ---
  与 Quest系统的关系

  Quest系统 和 Scene系统 都使用类似的架构：

  | 系统      | 执行单元            | 调度器                 |
  |---------|-----------------|---------------------|
  | Scene系统 | IExecutableItem | SideeffectsExecutor |
  | Quest系统 | NodeDefinition  | Quest Graph Engine  |

  共同点：
  - 都可以设置 StoryTier
  - 都可以发送 AICommand
  - 都可以管理游戏世界状态

  区别：
  - Scene系统：精确同步的剧情表演（过场动画、对话）
  - Quest系统：松散异步的任务流程（任务目标、触发器）

  ---
  ExecutableItem 的设计优势

  1. 解耦设计

  Scene资源 (.scene文件)
      ↓ (加载时)
  生成 Snapshot (数据描述)
      ↓ (执行时)
  创建 ExecutableItem (行为对象)
      ↓
  执行具体逻辑

  数据和行为分离，资源可以序列化。

  2. 多态执行

  // 统一接口调用
  for (auto& item : m_executables)
  {
      item->DoExecute();  // 多态调用不同子类
  }

  3. 生命周期管理

  - Prepare：预加载资源、查找目标
  - Execute：执行逻辑（可以多帧）
  - Sustain：保持状态（如IK需要持续更新）
  - Cancel：清理资源、恢复状态

  4. 并行执行

  SideeffectsExecutor 可以同时执行多个 ExecutableItem：

  同一帧：
  - ExecutableItem_SetCinematicMode (设置AI模式)
  - ExecutableItem_LookAt (控制视线)
  - ExecutableItem_AnimationMotion (播放动画)
  - ExecutableItem_ConfigureCamera (配置摄像机)

  ---
  总结

  IExecutableItem 是 Cyberpunk 2077 Scene系统的核心抽象，它：

  1. 是什么：Scene系统的原子执行单元（Command Pattern）
  2. 为什么重要：这是Scene系统控制游戏世界的唯一方式
  3. 有哪些：30+ 种子类，覆盖AI、动画、实体、摄像机、系统管理
  4. 如何工作：
    - 由 SideeffectsExecutor 调度
    - 多态执行 DoExecute()
    - 管理生命周期状态
    - 支持并行和持续执行
  5. 与StoryTier的关系：
    - ExecutableItem_SetCinematicMode 设置 StoryTier
    - 触发AI行为树切换分支
    - 实现Scene对AI的控制权接管

  这就是为什么 ExecutableItem_SetCinematicMode 在我们之前的分析中如此关键——它是Scene系统"夺取"AI控制权的关键执行单元。
