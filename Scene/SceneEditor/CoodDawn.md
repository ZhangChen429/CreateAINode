
  Workspot 的动画机制

  1. Workspot 有两类动画：

  Idle 动画（可循环）

  - 短暂、可循环
  - 可以随时中断
  - 例如：站立、坐着等待

  Gesture 动画（随机播放，时长不定）

  - 较长、不循环
  - 随机触发，时长各异
  - 例如：喝咖啡（3秒）、看报纸（8秒）、伸懒腰（5秒）

  2. 问题场景：

  时间轴:
  0s -------- 5s -------- 10s ------- 15s ------- 20s
              ↑                       ↑
          某个Gesture开始         Change Work事件
          (比如8秒的"看报纸")      (需要切换到站立)

  问题：
  - 在 5秒 时，Workspot 系统随机选择播放了一个 8秒 的 Gesture（看报纸）
  - 在 15秒 时，你的 Change Work 事件要求 Actor 切换到站立
  - 但是 Gesture 还要播放到 13秒！
  - 如果强制中断，动画会很突兀

  Cooldown 的解决方案

  Cooldown 的作用：提前进入 IdleOnlyMode

  看编译代码（scnbCompiler.cpp:1335-1343）：

  if( changeWorkEvent->GetMaxGestureDurationTimeSTU() != scn::SceneTime{ 0 } )
  {
      // ① 创建一个 "IdleOnlyMode" 节点
      THandle< quest::NodeDefinition > idleOnlyNode = CreateIdleOnlyModeEvent( entityRef );

      // ② 计算 IdleOnly 开始时间
      const scn::SceneTime maxGestureDuration = changeWorkEvent->GetMaxGestureDurationTimeSTU();
      const scn::SceneTime idleOnlyStartTime =
          changeWorkEvent->m_startTime - maxGestureDuration;
          //       ↑                          ↑
          //  Change Work时间            Cooldown时长

      // ③ 在 Change Work 之前插入 IdleOnlyMode
      compileEventQuestNode( ..., idleOnlyStartTime, maxGestureDuration, ... );
  }

  实际执行流程：

  假设：
  - Change Work 在 15秒
  - Cooldown = 8秒（最长的 Gesture 是"看报纸"8秒）

  编译后的时间轴:
  0s -------- 7s -------- 15s
              ↑           ↑
          IdleOnly开始  Change Work
          (自动插入)    (用户设置)

  执行时：
  0s-7s:   Workspot 正常运行（可能播放 Gesture）
  7s:      ★ 进入 IdleOnlyMode（停止播放新的 Gesture）
  7s-15s:  只播放 Idle 循环（不再播放 Gesture）
  15s:     Change Work 执行（Actor 在 Idle 状态，可以立即切换）

  为什么需要 Cooldown？

  核心原因：Gesture 播放时间不可预测

  // scnbWorkspotUtils.cpp:260-287
  if( editor.GetPerformerStateBeforeEvent( eventId, actorId, previewPerformerState ) )
  {
      const scn::PerformerState::WorkState workState = ...;

      // 问题：我们不知道 Actor 当前在播放哪个 Gesture
      // 解决：提前获取所有 Gesture 的最大时长

      if( const THandle< const work::WorkspotTree > tree = workState.m_workspotParams.m_tree )
      {
          // 遍历 Workspot 中所有 Gesture，找出最长的一个
          outDuration = ExtractMaxWorkspotGestureDuration(
              *editor.GetScenesolution(),
              actorId,
              *tree,  // 遍历这个 WorkspotTree
              workState.m_entryId,
              workState.m_excludedGestures,
              workState.m_maxAnimTimeLimit,
              ...
          );
      }
  }

  Workspot 的随机性：

  在游戏运行时，Workspot 系统会：
  1. 循环播放 Idle
  2. 在 Idle 循环中，随机选择一个 Gesture 播放
  3. Gesture 播放完后，回到 Idle
  4. 继续随机...
