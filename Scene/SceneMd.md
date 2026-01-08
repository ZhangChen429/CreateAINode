# 游戏与演出Editor


## 一、效果复现与问题解决
### 1.1 验证工具与配置
- 采用 `Validate` 验证工具及配套配置，校验动画是否符合规则。


### 1.2 动效定位与播放逻辑
- **位移信息记录**：通过 `Mark` 叙事点定位系统，记录动画中难以直观呈现的位移信息；结合 `WorkSpot` 点位信息，在 `QDILD` 设计中实现 `End` 点位与 `Start` 点位的 `NPC` 置信度衔接。
- **动画播放策略**：支持 `TPAnimation`、`ActorLocation`、`Blend` 三种播放方式，结合剧情约束选择合理方案（例如：玩家与 `NPC` 侧面相对视角时，播放玩家身后 `NPC` 的 `TPAnimation`）。


### 1.3 流程控制
- 多 `Section` 需合并为单一 `Timeline` 执行。
- 通过 `CodeDown` 与 `OnlyOneMode` 组合，触发下一个 `Section` 行为，同时设置 `MaxTimeLimit` 限制流程时长。
- 若 `Preview` 失效，通过验证工具进行修复。


## 二、演出与玩家互动设计
### 2.1 视角与行为引导
- 剧情演出中限制玩家视角（例如：玩家坐于椅子上），同时让身后 `NPC` 执行传送动画。
- 通过 `FPP` 相机动画，限定玩家观察角度与内容。
- 利用 `Fact` 数据库实现三次情绪递增，强化角色行为表现力。


### 2.2 动画与演出的收束逻辑
- **触发前提**：检测摄像机是否处于玩家注视范围内。
- **剧情约束**：例如“进入森严大楼需上交武器”，在关键信息演出时，通过其他 `NPC` 强制将武器按在地上。
- **状态检测**：捕捉玩家观看方向，判断是否处于被观察状态；非观察状态下动态调整内容，完成收束。
- **循环反馈**：通过 `Fact` 数据库实现 `Remaind` 循环，在不同情境下提供 `remaind` 反馈。


## 三、动画库组织方案
### 3.1 核心过渡动作
以 `stand`、`legCross`、`Knees`、`lead` 等动作作为动画切换的中间状态。


### 3.2 动画分类
分为**通用动画**与**静帧动画**两类。


### 3.3 层级分类规则
1. **大类**：`player`、`NPC`、`Weapon`
2. **中类**：按“武器”“性别”细分
3. **模块化复用**：
   - 单个 `Idle` 包含随机 `Idle` 分布
   - 三维分类维度：功能、角色类型/派系、场景


### 3.4 命名规范
- 基础前缀：
  - `pma_` = 玩家-男性-平均体型
  - `pwa_` = 玩家-女性-平均体型
  - `ma_` = NPC-男性-平均体型
  - `mb_/mf/mm_` = 大/胖/巨大体型
  - `cw_` = 义体
  - `w_` = 武器
- 示例：
  - `pma_w_handgun_araska_kenshin`（玩家-男-手枪-荒坂-剑心）
  - `ma_corpo_handgunboth_action_combat`（NPC-公司-双手持枪-战斗动作）
  - `synd_ couple_taking_selfie_01_female_average`（帮派-情侣自拍-01-女性-平均体型）


## 四、新发现内容
- `Fact` 实现 `Remained` 流程
- `LookDirect` 功能
- `Section` 控制逻辑
- 细分主题4


要不要我帮你补充一份**动画库命名规范的简化对照表**？








~~~C++


  a) PlaySkAnimDescriptor（播放骨骼动画）

  if ( event->IsA<tools::PlaySkAnimDescriptor>() )
  {
      THandle<tools::PlaySkAnimDescriptor> skeletalAnimDescriptor = Cast<tools::PlaySkAnimDescriptor>( event );
      AddAnimationToMap( animNames, skeletalAnimDescriptor->m_animName );
  }
  - 示例动画："stand__2h_front__01", "walk_0__to__stand__2h_front__01__turn0__01"

  b) ChangeIdleAnimDescriptor（切换待机动画）

  else if ( event->IsA<tools::ChangeIdleAnimDescriptor>() )
  {
      THandle<tools::ChangeIdleAnimDescriptor> changeIdleAnimDescriptor = Cast<tools::ChangeIdleAnimDescriptor>( event );
      AddAnimationToMap( animNames, changeIdleAnimDescriptor->m_animName );           // 主动画
      AddAnimationToMap( animNames, changeIdleAnimDescriptor->m_idleAnimName );       // 待机动画
      AddAnimationToMap( animNames, changeIdleAnimDescriptor->m_addIdleAnimName );    // 附加待机动画

      if ( changeIdleAnimDescriptor->m_customTransitionAnim != CName::NONE() )
      {
          AddAnimationToMap( animNames, changeIdleAnimDescriptor->m_customTransitionAnim );  // 过渡动画
      }
  }
  - 一个事件可以贡献最多 4 个动画名称

  c) ChangeWorkEvent（切换工作状态事件）

  else if ( event->IsA<scnb::events::ChangeWorkEvent>() )
  {
      THandle<scnb::events::ChangeWorkEvent> changeWorkDescriptor = Cast<scnb::events::ChangeWorkEvent>( event );
      if ( changeWorkDescriptor->GetTransitionAnimInfo().m_animName != CName::NONE() )
      {
          AddAnimationToMap( animNames, changeWorkDescriptor->GetTransitionAnimInfo().m_animName );  // 过渡动画
      }
      if ( changeWorkDescriptor->GetGameplayAnimInfo().m_animName != CName::NONE() )
      {
          AddAnimationToMap( animNames, changeWorkDescriptor->GetGameplayAnimInfo().m_animName );    // 游戏玩法动画
      }
  }

  d) StopWorkEvent（停止工作事件）

  else if ( event->IsA<scnb::events::StopWorkEvent>() )
  {
      THandle<scnb::events::StopWorkEvent> stopWorkDescriptor = Cast<scnb::events::StopWorkEvent>( event );
      if ( stopWorkDescriptor->GetAnimationInfo().m_animName != CName::NONE() )
      {
          AddAnimationToMap( animNames, stopWorkDescriptor->GetAnimationInfo().m_animName );
      }
      if ( stopWorkDescriptor->GetGameplayAnimInfo().m_animName != CName::NONE() )
      {
          AddAnimationToMap( animNames, stopWorkDescriptor->GetGameplayAnimInfo().m_animName );
      }
  }

  2. Internal Workspots 的 WorkspotTree（行 747-760）

  for( const THandle< const scnb::SceneWorkspot > workspot : sceneEditorResource->GetWorkspots() )
  {
      if ( workspot->IsExternal() )
      {
          externalWorkspots.PushBack( workspot->GetExternalResourcePath() );
      }
      else
      {
          const THandle<work::WorkspotTree> workspotTree = workspot->GetWorkspotTree();
          GatherWorkspotTreeAnimations( workspotTree, animNames );  // ← 第二个来源

          internalWorkspots.PushBack( workspot->GetName() );
      }
  }

~~~



PlaySkAnimDescriptor类型 
ChangeIdleAnimDescriptor类型
ChangeWorkEvent类型
StopWorkEvent类型