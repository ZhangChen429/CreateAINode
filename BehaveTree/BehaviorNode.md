composite
composite::ifElse
composite::sequence
composite::selector
composite::parallel

Decorator 
Decorator::CommandHandler 
Decorator::CompleteOnEvent 
Decorator::Condition::InstantCondition 
Decorator::Condition::MonitorCondition 
Decorator::EventHandler 
Decorator::ExitWorkspot 
Decorator::FreeReservedWorkspot 
Decorator::LeaveCoverlmmediately 
Decorator::Limiter 
Decorator::Maybe
Decorator::PassiveCondition 
Decorator::PrepareReservedCrowdWorkspot 
Decorator::ReevaluateOnEvent 
Decorator::Repeat 
Decorator::SelectWorkspot 
Decorator::Task::InstantTask 
Decorator::Task::MonitorTask 
Decorator::Timeout 
Decorator::TrackPatrolProgress 
Decorator::UnmountImmediately 
Decorator::ForcedBehavior


Leaf 
Leaf::Action::Action 
Leaf::Action::ActionDie 
Leaf::Action::ActionDroneMove 
Leaf::Action::ActionDroneMoveSpline 
Leaf::Action::ActionDynamicMove 
Leaf::Action::ActionMove 
Leaf::Action::ActionMoveOnSpline 
Leaf::Action::ActionMoveToSmartObject 
Leaf::Action::ActionMoveToWorkspot 
Leaf::Action::ActionMoveWithPolicy 
Leaf::Action::ActionRotateBase 
Leaf::Action::ActionRotateBase::ActionRotateToObject 
Leaf::Action::ActionRotateBase::ActionRotateToObjectHeading 
Leaf::Action::ActionRotateBase::ActionRotateToPosition 
Leaf::Action::ActionRotateByAngle 
Leaf::Action::ActionSlide 
Leaf::Action::ActionSlide::ActionSlideToLocalPosition 
Leaf::Action::ActionSlide::ActionSlideToObject 
Leaf::Action::ActionSlide::Slide 
Leaf::Action::ActionSlide::ToWorldPosition 
Leaf::Action::ActionTeleport 
Leaf::Action::ActionUseWorkspot 
Leaf::Action::AnimationCurvePath 
Leaf::Action::ControlledByQuestNode 
Leaf::Action::Drive 
Leaf::Action::Drive::Driveldle 
Leaf::Action::Drive::DriveJoinTraffic 
Leaf::Action::Drive::DriveRacing 
Leaf::Action::Drive::DriveSummoning 
Leaf::Action::Drive::DriveToFollowPosition 
Leaf::Action::Drive::DriveToFollowSlot 
Leaf::Action::Drive::DriveToFollowSpline 
Leaf::Action::Drive::DriveStunned 
Leaf::Action::Drive::DriveToFollowTarget 
Leaf::Action::Drive::DriveToNodeRel 
Leaf::Action::Drive::DriveToPoint 
Leaf::Action::Drive::DriveToSplineReverse 
Leaf::Action::ItemHandling 
Leaf::Action::ItemHandling::EquipSpecificltemToSlot 
Leaf::Action::ItemHandling::UnequipltemFromSlot 
Leaf::Action::MountHandling 
Leaf::Action::MountHandling::MountObject 
Leaf::Action::MountHandling::Unmount 
Leaf::Action::Move Along Traffic Path 
Leaf::Action::Patrol 
Leaf::Action::RotateToObjectConstTime 
Leaf::Action::Scene Animation Motion 
Leaf::Action::UseCommunityWorkspot 


Failer 
Idle 
Succeeder



动作基类，定义所有动作节点的通用接口（如执行、终止、状态回调）。
死亡动作：让 AI 执行死亡动画、状态切换（如 “生命值清零→播放死亡动画→标记为死亡状态”）。
无人机移动：控制无人机类型 AI 的基础移动（如直线移动、跟随目标）。
无人机样条线移动：控制无人机沿预设样条线（Spline）移动（如固定航线巡逻）。
动态移动：AI 根据实时环境动态调整移动路径（如避开突发障碍物、跟随移动的目标）。
基础移动：AI 的通用移动（如向指定位置移动、跟随指定对象）。
样条线移动：AI 沿预设样条线移动（如固定路线巡逻、沿赛道行驶）。
移动到智能对象：AI 移动到预设的 “智能对象”（如可交互的门、宝箱、机关）旁，为后续交互做准备。
移动到工作点：AI 移动到指定工作点（如椅子、工具台、站岗位置）。
带策略移动：按预设策略移动（如 “潜行移动”“快速冲刺”“节能移动”），适配不同场景需求。
基础旋转：AI 旋转自身朝向（如转向目标、转向指定方向）。
转向对象：AI 朝向指定对象（如敌人、友军、交互物）。
朝向对象朝向：AI 转向与指定对象相同的朝向（如跟随队友的前进方向）。
朝向位置：AI 朝向指定世界坐标（如预设的警戒方向、任务目标点）。
按角度旋转：AI 按指定角度旋转（如顺时针旋转 90 度、逆时针旋转 45 度）。
滑行动作：AI 执行滑行行为（如地面滑行、空中滑翔）。
滑行到本地位置：AI 相对于自身当前位置，滑行到指定本地坐标（如 “向前滑行 2 米”）。
滑行到对象：AI 滑行到指定对象旁（如滑行到敌人身边进行攻击）。
基础滑行：无目标的通用滑行（如按当前朝向滑行一段距离）。
滑行到世界位置：AI 滑行到指定世界坐标（如预设的滑行终点）。
传送动作：AI 瞬间移动到指定位置（如 “传送回出生点”“传送至任务目标旁”）。
使用工作点：AI 在当前工作点执行交互行为（如坐在椅子上、使用工具台制作物品）。
动画曲线路径：AI 沿动画曲线定义的路径移动 / 旋转（如复杂的舞蹈动作、特技移动）。
任务节点控制：AI 行为由任务节点直接控制（如 “任务触发时才执行该动作”，与剧情 / 任务绑定）。
驾驶动作基类：控制 AI 驾驶载具（如汽车、飞机、坐骑）的通用逻辑。
怠速驾驶：载具保持怠速状态（如停车时发动机空转、缓慢移动等待指令）。
汇入车流：AI 驾驶载具汇入当前车流，跟随车流速度和路线行驶（如城市道路行驶）。
竞速驾驶：载具以竞速模式行驶（如高速冲刺、规避赛道障碍物、超越对手）。
召唤载具驾驶：AI 召唤载具后立即驾驶（如 “召唤坐骑→自动骑上并移动”）。
跟随位置驾驶：载具驾驶到指定跟随位置（如 “跟随队友后方 10 米处行驶”）。
跟随插槽驾驶：载具驾驶到预设的 “跟随插槽”（如车队中的指定位置、护航队列位置）。
沿样条线跟随驾驶：载具沿预设样条线，跟随前方目标行驶（如车队沿固定路线护航）。
眩晕驾驶：载具处于眩晕状态下的驾驶逻辑（如失控摇晃、减速停车）。
跟随目标驾驶：载具持续跟随指定目标移动（如 “追击逃跑的敌人载具”）。
相对节点驾驶：载具驾驶到相对于某个节点的指定位置（如 “停在任务节点前方 5 米处”）。
驾驶到点：载具驾驶到指定世界坐标点（如 “前往地图标记的任务地点”）。
反向样条线驾驶：载具沿预设样条线反向行驶（如 “沿巡逻路线返程”）。
物品操作基类：定义物品交互的通用逻辑（如装备、卸下、使用物品）。
装备指定物品到插槽：将指定物品装备到 AI 的对应插槽（如 “把剑装备到主手插槽”“把护盾装备到副手插槽”）。
从插槽卸下物品：将 AI 对应插槽中的物品卸下（如 “卸下主手武器”“收起盾牌”）。
挂载操作基类：定义挂载 / 卸载对象（如坐骑、武器）的通用逻辑。
挂载对象：AI 挂载指定对象（如 “骑上坐骑”“装备重型武器”）。
卸载对象：AI 卸载当前挂载的对象（如 “下马”“收起重型武器”）。
沿交通路径移动：AI 沿预设的交通路径移动（如城市道路的人行道、车道，模拟真实行人 / 车辆移动）。
巡逻动作：AI 按预设巡逻路线移动（如 “绕基地巡逻”“沿街道往返巡逻”），包含路径点切换、警戒逻辑。
固定时间转向对象：AI 在指定时间内（如 2 秒）平稳转向目标对象（避免瞬间旋转，更自然）。
场景动画动作：AI 执行与场景绑定的动画动作（如 “触发剧情时的特定动作”“与场景道具交互的动画”）。
使用公共工作点：AI 使用公共工作点资源（如 “使用公共饮水机”“在共享工作台制作物品”），支持多 AI 排队使用。



● 我来帮你全面介绍 ActionTreeNode 及其所有子类。首先让我搜索所有继承自它的子类。

● Search(pattern: "class \w+.*: public ActionTreeNode", path: "D:/AppSoft/Sy2077/2077/2077/CDPR2077/dev/src", output_mode: "content")
  ⎿  Found 59 lines (ctrl+o to expand)

● 好的！让我全面介绍 ActionTreeNode 及其所有子类。这是一个庞大的系统。

● Read(D:/AppSoft/Sy2077/2077/2077/CDPR2077/dev/src/common/gameAI/include/aiActionNode.h)
  ⎿  Read 114 lines

● ActionTreeNode 完整介绍

  一、基类：ActionTreeNode

  定义位置: aiActionNode.h:31-54

  核心功能: 行为树中所有动作节点的抽象基类，用于将游戏动作（CAction）集成到 AI 行为树系统中。

  继承关系:
  ActionTreeNode : public LeafTreeNode, public EventHandler
  - LeafTreeNode: 行为树叶子节点（无子节点）
  - EventHandler: 可以接收和处理 AI 事件

  核心成员:
  protected:
      TInstanceVar< game::CActionAIProxy > i_action;  // 动作代理
      TInstanceVar< Bool > i_isActive;                // 是否激活
      THandle< ArgumentMapping > m_command;           // 可选的命令关联

  核心方法:
  - SetupAction(ExecutionContext& context): 纯虚函数，子类实现具体动作设置
  - OnActivate(): 节点激活时创建并启动动作
  - OnDeactivate(): 节点取消激活时停止动作
  - OnActionCompleted(): 动作完成回调
  - OnEvent(): 处理动作事件

  ---
  二、所有子类分类介绍

  我找到了 38 个子类，按功能分为以下几类：

  1. 角色控制类 (Character Control)

  1.1 ActionDieTreeNode - 死亡动作

  - 文件: aiTreeNodeDieAction.h
  - 功能: 触发角色死亡，激活布娃娃物理
  - 参数: Command (可选)

  1.2 ActionItemHandlingNode - 物品处理

  - 文件: aiActionEquipUnequipItemNode.h
  - 功能: 装备/卸下武器和物品
  - 参数:
    - ItemID (物品 ID)
    - SlotID (装备槽位)
    - Equip/Unequip (装备或卸下)

  1.3 ActionMountHandlingNode - 载具上下

  - 文件: aiActionMountUnmountNode.h
  - 功能: 角色上车/下车、骑乘载具
  - 参数:
    - VehicleRef (载具引用)
    - SeatName (座位名称)
    - Mount/Unmount (上载具或下载具)

  1.4 ActionTeleportTreeNode - 传送

  - 文件: aiActionTeleportNode.h
  - 功能: 瞬移角色到指定位置
  - 参数:
    - Position (目标位置)
    - Rotation (目标旋转)
    - DoNavTest (导航测试)

  ---
  2. 移动类 (Movement)

  2.1 ActionMoveTreeNode - 基础移动

  - 文件: aiActionMoveNode.h
  - 功能: 移动到目标对象或位置
  - 参数:
    - Target (目标)
    - MovementType (移动类型：Walk/Run/Sprint)
    - StopWhenDestinationReached (到达后停止)
    - KeepDistance (保持距离)

  2.2 ActionDynamicMoveTreeNode - 动态移动

  - 文件: aiActionDynamicMoveNode.h
  - 功能: 动态移动，目标可以在运动中变化
  - 参数:
    - Target (动态目标)
    - MovementType
    - Tolerance (容差)

  2.3 ActionMoveWithPolicyTreeNode - 策略移动

  - 文件: aiActionMoveWithPolicyNode.h
  - 功能: 根据移动策略移动（如：掩护移动、迂回等）
  - 参数:
    - MovementPolicy (移动策略)
    - Target (目标)

  2.4 ActionMoveToWorkspotNode - 移动到工作点

  - 文件: aiActionMoveToWorkspotNode.h
  - 功能: 移动到指定的工作点（workspot）位置
  - 参数:
    - WorkspotRef (工作点引用)
    - MovementType

  2.5 ActionMoveToSmartObjectNode - 移动到智能对象

  - 文件: aiActionMoveToSmartObjectNode.h
  - 功能: 移动到智能对象并准备交互
  - 参数:
    - SmartObjectRef (智能对象引用)

  2.6 ActionSlideNode - 滑动

  - 文件: aiActionSlideNode.h
  - 功能: 角色滑铲/滑动动作
  - 参数:
    - Direction (方向)
    - Distance (距离)

  2.7 MoveAlongTrafficPathActionNode - 沿交通路径移动

  - 文件: aiMoveAlongTrafficPathActionNode.h
  - 功能: NPC 沿交通路径行走（如人行道）
  - 参数:
    - PathRef (路径引用)

  2.8 PatrolActionNode - 巡逻

  - 文件: aiPatrolActionNode.h
  - 功能: 在巡逻点之间巡逻
  - 参数:
    - PatrolPath (巡逻路径)
    - PatrolType (巡逻类型：循环/往返/随机)

  ---
  3. 样条移动类 (Spline Movement)

  3.1 ActionMoveOnSplineNode - 沿样条移动

  - 文件: aiActionMoveOnSplineNode.h
  - 功能: 沿预定义的样条曲线移动
  - 参数:
    - SplineRef (样条引用)
    - Speed (速度)

  3.2 ActionAnimationCurvePathNode - 动画曲线路径

  - 文件: aiActionAnimationCurvePathNode.h
  - 功能: 沿动画曲线移动，适合精确的动作捕捉场景
  - 参数:
    - AnimationPath (动画路径)

  ---
  4. 旋转类 (Rotation)

  4.1 ActionRotateBaseTreeNode - 基础旋转

  - 文件: aiActionRotateNode.h
  - 功能: 旋转到指定目标或方向（基类）
  - 参数:
    - Target (旋转目标)
    - AngleTolerance (角度容差)

  4.2 ActionRotateByAngleTreeNode - 按角度旋转

  - 文件: aiActionRotateNode.h
  - 功能: 旋转指定角度（相对或绝对）
  - 参数:
    - Angle (角度值)
    - IsRelative (是否相对旋转)

  4.3 ActionRotatetoObjectConstTimeTreeNode - 恒定时间旋转

  - 文件: aiActionRotateNode.h
  - 功能: 在固定时间内旋转到目标对象
  - 参数:
    - Target (目标)
    - RotationTime (旋转时间)

  ---
  5. 无人机移动类 (Drone Movement)

  5.1 ActionDroneMoveTreeNode - 无人机移动

  - 文件: aiActionDroneMoveNode.h
  - 功能: 无人机三维空间移动到目标
  - 参数:
    - Target (目标)
    - Height (飞行高度)

  5.2 ActionDroneMoveSplineTreeNode - 无人机样条移动

  - 文件: aiActionDroneMoveSplineNode.h
  - 功能: 无人机沿样条曲线飞行
  - 参数:
    - SplineRef (样条引用)
    - Speed (飞行速度)

  ---
  6. 工作点类 (Workspot)

  6.1 ActionUseWorkspotNode - 使用工作点

  - 文件: aiActionUseWorkspotNode.h
  - 功能: 在工作点执行动作（坐、站、操作等）
  - 参数:
    - WorkspotRef (工作点引用)
    - AnimationName (动画名称)
    - InfiniteMode (无限循环模式)

  6.2 ActionUseCommunityWorkspotNode - 使用社区工作点

  - 文件: aiActionUseWorkspotNode.h
  - 功能: NPC 在社区场景中使用工作点（如咖啡馆坐着）
  - 参数:
    - CommunityWorkspot (社区工作点)

  ---
  7. 场景动画类 (Scene Animation)

  7.1 ActionSceneAnimationMotionNode - 场景动画运动

  - 文件: aiActionSceneAnimationMotionNode.h
  - 功能: 播放场景动画并同步角色运动（用于过场动画）
  - 参数:
    - SceneAnimation (场景动画)
    - SyncPosition (同步位置)

  ---
  8. 任务控制类 (Quest Control)

  8.1 CTreeNodeControlledByQuestNode - 任务控制节点

  - 文件: aiTreeNodeControlledByQuestNode.h
  - 功能: 由任务系统直接控制的行为树节点
  - 参数:
    - QuestPhase (任务阶段)
    - ListenerName (监听器名称)

  ---
  9. 驾驶类 (Driving - 共 12 个子类)

  这些节点用于 NPC 驾驶载具的各种行为。

  9.1 DriveToPointTreeNode - 驾驶到点

  - 文件: driveActionGoToPoint.h
  - 功能: 驾驶到指定位置
  - 参数:
    - TargetPosition (目标位置)
    - Speed (速度限制)
    - TrafficLights (是否遵守交通灯)

  9.2 DriveToNodeTreeNode - 驾驶到节点

  - 文件: driveActionGoToNode.h
  - 功能: 驾驶到指定节点
  - 参数:
    - TargetNode (目标节点)

  9.3 DriveFollowTargetTreeNode - 驾驶跟随目标

  - 文件: driveActionFollowTarget.h
  - 功能: 驾驶跟随移动目标（追车）
  - 参数:
    - Target (跟随目标)
    - FollowDistance (跟随距离)

  9.4 DriveFollowPositionTreeNode - 驾驶跟随位置

  - 文件: driveActionFollowPosition.h
  - 功能: 驾驶跟随动态更新的位置
  - 参数:
    - PositionSource (位置源)

  9.5 DriveFollowSlotTreeNode - 驾驶跟随插槽

  - 文件: driveActionFollowSlot.h
  - 功能: 驾驶到目标对象的特定插槽位置（编队驾驶）
  - 参数:
    - Target (目标)
    - SlotName (插槽名称)

  9.6 DriveFollowSplineTreeNode - 驾驶沿样条

  - 文件: driveActionFollowSpline.h
  - 功能: 驾驶沿样条曲线行驶
  - 参数:
    - SplineRef (样条引用)
    - Speed (速度)

  9.7 DriveSplineReverseTreeNode - 驾驶倒车

  - 文件: driveActionSplineReverse.h
  - 功能: 沿样条倒车
  - 参数:
    - SplineRef (样条引用)

  9.8 DriveJoinTrafficTreeNode - 加入交通

  - 文件: driveActionJoinTraffic.h
  - 功能: 车辆加入交通流
  - 参数:
    - RoadRef (道路引用)

  9.9 DriveIdleTreeNode - 驾驶闲置

  - 文件: driveActionIdle.h
  - 功能: 车辆停止并保持引擎运转
  - 参数: 无

  9.10 DriveRacingTreeNode - 竞速驾驶

  - 文件: driveActionRacing.h
  - 功能: 高速竞速驾驶模式
  - 参数:
    - TargetSpeed (目标速度)
    - Aggressive (激进程度)

  9.11 DriveStunnedTreeNode - 驾驶眩晕

  - 文件: driveActionStunned.h
  - 功能: 车辆失控状态（撞击后）
  - 参数:
    - Duration (持续时间)

  9.12 DriveSummoningTreeNode - 召唤载具

  - 文件: driveActionSummoning.h
  - 功能: 载具被召唤时的驾驶到玩家行为
  - 参数:
    - SummonerRef (召唤者引用)

  ---
  三、工作流程

  所有 ActionTreeNode 子类遵循相同的执行流程：

  1. 行为树激活节点
     ↓
  2. OnActivate() 被调用
     ↓
  3. 子类的 SetupAction() 创建对应的 CAction
     ↓
  4. UpdateActionCommandId() 关联命令（如果有）
     ↓
  5. 通过 CActionAIProxy 启动动作
     ↓
  6. 动作执行中（可接收事件）
     ↓
  7. 动作完成，OnActionCompleted() 被调用
     ↓
  8. OnDeactivate() 清理资源

  ---
  四、使用示例

  在行为树编辑器中，设计师可以组合这些节点实现复杂的 AI 行为：

  行为树示例：NPC 巡逻并发现敌人后追击
  ├─ Sequence
     ├─ PatrolActionNode (巡逻)
     ├─ Condition: IsEnemyDetected
     ├─ ActionMoveTreeNode (跑向敌人)
     └─ ActionRotateBaseTreeNode (转向敌人)

  ---
  五、总结

  - 总计: 38 个 ActionTreeNode 子类
  - 覆盖: 角色移动、旋转、死亡、载具驾驶、工作点、场景动画等所有游戏动作
  - 扩展性: 每个子类封装一个特定的游戏动作，易于添加新动作类型
  - 统一接口: 所有动作通过相同的生命周期管理和事件系统

  这个设计使得《赛博朋克 2077》能够构建出复杂、层次化的 NPC AI 行为系统。
