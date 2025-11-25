# Cyberpunk 2077 Quest 节点类型功能对照表

> 数据来源: complete_quest_nodes_list.md
> 生成日期: 2025-11-23
> 总节点数: 150+

---

## 1. 核心节点 (Core Nodes)

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| NodeDefinition | 所有Quest节点的基类 | node.h |
| DisableableNodeDefinition | 可禁用的节点基类 | disableableNode.h |
| SignalStoppingNodeDefinition | 可阻断信号传播的节点基类 | signalStoppingNode.h |
| TypedSignalStoppingNodeDefinition | 带类型的信号阻断节点 | signalStoppingNode.h |
| StartEndNodeDefinition | 开始/结束节点基类 | startEndNode.h |
| StartNodeDefinition | Quest开始节点 | startNode.h |
| EndNodeDefinition | Quest结束节点 | endNode.h |
| IONodeDefinition | 输入/输出节点基类 | ioNode.h |
| InputNodeDefinition | 输入节点 | inputNode.h |
| OutputNodeDefinition | 输出节点 | outputNode.h |
| GraphDefinition | Quest图定义 | graph.h |
| SocketDefinition | Socket定义 | socket.h |

---

## 2. 管理器节点 (Manager Nodes)

### 2.1 角色管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| CharacterManagerNodeDefinition | 角色管理器主节点 | characterManagerNode.h |
| CharacterManagerParameters_SetAttitudeGroupForPuppet | 设置AI态度组 | characterManagerNodeTypeImpl.h |
| CharacterManagerParameters_SetGroupsAttitude | 设置组态度 | characterManagerNodeTypeImpl.h |
| CharacterManagerParameters_SetMortality | 设置生死状态 | characterManagerNodeTypeImpl.h |
| CharacterManagerParameters_SetAnimset | 设置动画集 | characterManagerNodeTypeImpl.h |
| CharacterManagerParameters_SetLowGravity | 设置低重力 | characterManagerNodeTypeImpl.h |
| CharacterManagerParameters_EnableBumps | 启用碰撞 | characterManagerNodeTypeImpl.h |
| CharacterManagerParameters_SetStatusEffect | 设置状态效果 | characterManagerNodeTypeImpl.h |
| CharacterManagerParameters_SetReactionPreset | 设置反应预设 | characterManagerNodeTypeImpl.h |
| CharacterManagerParameters_SetGender | 设置性别 | characterManagerNodeTypeImpl.h |
| CharacterManagerParameters_SetAsCrowdObstacle | 设为人群障碍物 | characterManagerNodeTypeImpl.h |
| CharacterManagerParameters_SetProgressionBuild | 设置进度构建 | characterManagerNodeTypeImpl.h |
| CharacterManagerParameters_SetLifePath | 设置人生轨迹 | characterManagerNodeTypeImpl.h |
| CharacterManagerParameters_HealPlayer | 治疗玩家 | characterManagerNodeTypeImpl.h |
| CharacterManagerCombat_ModifyHealth | 修改生命值 | characterManagerNodeTypeImpl.h |
| CharacterManagerCombat_Kill | 杀死角色 | characterManagerNodeTypeImpl.h |
| CharacterManagerCombat_EquipWeapon | 装备武器 | characterManagerNodeTypeImpl.h |
| CharacterManagerCombat_SetWeaponState | 设置武器状态 | characterManagerNodeTypeImpl.h |
| CharacterManagerCombat_SetDeathDirection | 设置死亡方向 | characterManagerNodeTypeImpl.h |
| CharacterManagerCombat_ChangeLevel | 改变等级 | characterManagerNodeTypeImpl.h |
| CharacterManagerCombat_ManageRagdoll | 管理布娃娃系统 | characterManagerNodeTypeImpl.h |
| CharacterManagerCombat_AssignSquad | 分配小队 | characterManagerNodeTypeImpl.h |
| CharacterManagerParameters_SetCombatSpace | 设置战斗空间 | characterManagerNodeTypeImpl.h |
| CharacterManagerVisuals_ChangeEntityAppearance | 改变实体外观 | characterManagerNodeTypeImpl.h |
| CharacterManagerVisuals_PrefetchEntityAppearance | 预加载实体外观 | characterManagerNodeTypeImpl.h |
| CharacterManagerVisuals_GenitalsManager | 生殖器管理 | characterManagerNodeTypeImpl.h |
| CharacterManagerVisuals_BreastSizeController | 胸部大小控制 | characterManagerNodeTypeImpl.h |
| CharacterManagerVisuals_SetBrokenNoseStage | 设置鼻梁破损阶段 | characterManagerNodeTypeImpl.h |

### 2.2 实体管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| EntityManagerNodeDefinition | 实体管理器主节点 | entityManagerNode.h |
| EntityManagerSetAttachment_NodeType | 设置附着 | entityManagerNodeTypeImpl.h |
| EntityManagerSetDestructionState_NodeType | 设置破坏状态 | entityManagerNodeTypeImpl.h |
| EntityManagerManageBinkComponent_NodeType | 管理Bink组件 | entityManagerNodeTypeImpl.h |
| EntityManagerSetMeshAppearance_NodeType | 设置网格外观 | entityManagerNodeTypeImpl.h |
| EntityManagerEnablePlayerTPPRepresentation_NodeType | 启用玩家第三人称表示 | entityManagerNodeTypeImpl.h |
| EntityManagerToggleComponent_NodeType | 切换组件 | entityManagerNodeTypeImpl.h |
| EntityManagerChangeAppearance_NodeType | 改变外观 | entityManagerNodeTypeImpl.h |
| EntityManagerMountPuppet_NodeType | 骑乘Puppet | entityManagerNodeTypeImpl.h |
| EntityManagerSendAnimationEvent_NodeType | 发送动画事件 | entityManagerNodeTypeImpl.h |
| EntityManagerSetStat_NodeType | 设置属性 | entityManagerNodeTypeImpl.h |
| EntityManagerToggleMirrorsArea_NodeType | 切换镜像区域 | entityManagerNodeTypeImpl.h |
| EntityManagerSetAttachment_ToActor | 附着到角色 | entityManagerNodeTypeImpl.h |
| EntityManagerDestroyCarriedObject | 销毁携带物体 | entityManagerNodeTypeImpl.h |
| EntityManagerSetAttachment_ToNode | 附着到节点 | entityManagerNodeTypeImpl.h |
| EntityManagerSetAttachment_ToWorld | 附着到世界 | entityManagerNodeTypeImpl.h |

### 2.3 环境管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| EnvironmentManagerNodeDefinition | 环境管理器主节点 | environmentManagerNode.h |
| PlayEnv_NodeType | 播放环境 | environmentManagerNodeType.h |
| PlayEnv_OverrideGlobalLight | 覆盖全局光照 | environmentManagerNodeType.h |
| PlayEnv_ForceRelitEnvProbe | 强制重新照明环境探针 | environmentManagerNodeType.h |
| PlayEnv_SetWeather | 设置天气 | environmentManagerNodeType.h |

### 2.4 游戏管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| GameManagerNodeDefinition | 游戏管理器主节点 | gameManagerNode.h |
| TimeDilation_NodeType | 时间膨胀 | gameManagerNodeType.h |
| TimeDilation_World | 世界时间膨胀 | gameManagerNodeType.h |
| TimeDilation_Player | 玩家时间膨胀 | gameManagerNodeType.h |
| TimeDilation_Entity | 实体时间膨胀 | gameManagerNodeType.h |
| TimeDilation_Puppet | Puppet时间膨胀 | gameManagerNodeType.h |
| ControlObject_NodeType | 控制物体 | gameManagerNodeType.h |
| Replacer_NodeType | 替换器 | gameManagerNodeType.h |
| LootPurge_NodeType | 清除战利品 | gameManagerNodeType.h |
| ContentTokenManager_NodeType | 内容令牌管理器 | gameManagerNodeType.h |
| SpawnToken_NodeSubType | 生成令牌 | gameManagerNodeType.h |
| RemoveToken_NodeSubType | 移除令牌 | gameManagerNodeType.h |
| ForceTokenActivation_NodeSubType | 强制令牌激活 | gameManagerNodeType.h |
| BlockTokenActivation_NodeSubType | 阻止令牌激活 | gameManagerNodeType.h |
| GameplayRestrictions_NodeType | 游戏限制 | gameManagerNodeType.h |
| SetTimer_NodeType | 设置计时器 | gameManagerNodeType.h |
| Rumble_NodeType | 震动 | gameManagerNodeType.h |

### 2.5 事件管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| EventManagerNodeDefinition | 事件管理器节点 | eventManagerNode.h |

### 2.6 特效管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| FXManagerNodeDefinition | 特效管理器主节点 | fxManagerNode.h |
| PlayFX_NodeType | 播放特效 | fxManagerNodeType.h |
| PreloadFX_NodeType | 预加载特效 | fxManagerNodeType.h |

### 2.7 渲染特效管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| RenderFxManagerNodeDefinition | 渲染特效管理器主节点 | renderFxManagerNode.h |
| SetFadeInOut_NodeType | 设置淡入淡出 | renderFxManagerNodeType.h |
| SetDebugView_NodeType | 设置调试视图 | renderFxManagerNodeType.h |
| SetCyberspacePostFX_NodeType | 设置赛博空间后处理 | renderFxManagerNodeType.h |
| EnforceScreenSpaceReflectionsUberQuality_NodeType | 强制屏幕空间反射超高质量 | renderFxManagerNodeType.h |
| RenderPlane_NodeType | 渲染平面 | renderFxManagerNodeType.h |
| SetRenderLayer_NodeType | 设置渲染层 | renderFxManagerNodeType.h |

### 2.8 物品管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| ItemManagerNodeDefinition | 物品管理器主节点 | itemManagerNode.h |
| AddRemoveItem_NodeType | 添加/移除物品 | itemManagerNodeType.h |
| DropItemFromSlot_NodeType | 从槽位丢弃物品 | itemManagerNodeType.h |
| SetItemTags_NodeType | 设置物品标签 | itemManagerNodeType.h |
| TransferItem_NodeType | 转移物品 | itemManagerNodeType.h |
| LootTokenManager_NodeType | 战利品令牌管理器 | itemManagerNodeType.h |
| UseWeapon_NodeType | 使用武器 | itemManagerNodeType.h |
| SetLootInteractionAccess_NodeType | 设置战利品交互访问 | itemManagerNodeType.h |
| InjectLoot_NodeType | 注入战利品 | itemManagerNodeType.h |

### 2.9 交互对象管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| InteractiveObjectManagerNodeDefinition | 交互对象管理器主节点 | interactiveObjectManagerNode.h |
| SetInteractionState_NodeType | 设置交互状态 | interactiveObjectManagerNodeType.h |
| SetInteractionVisualizerOverride | 设置交互可视化覆盖 | interactiveObjectManagerNodeType.h |
| Elevator_ManageNPCAttachment_NodeType | 电梯NPC附着管理 | interactiveObjectManagerNodeType.h |
| HackingManager_NodeType | 黑客管理器 | interactiveObjectManagerNodeType.h |
| HackingManager_SetEnabled | 设置启用 | interactiveObjectManagerNodeType.h |
| HackingManager_SetHacked | 设置已黑客 | interactiveObjectManagerNodeType.h |
| Cyberdrill_NodeType | 电钻 | interactiveObjectManagerNodeType.h |
| SetConveyorState_NodeType | 设置传送带状态 | interactiveObjectManagerNodeType.h |
| DeviceManager_NodeType | 设备管理器 | interactiveObjectManagerNodeType.h |
| SetInspectMode_NodeType | 设置检查模式 | interactiveObjectManagerNodeType.h |

### 2.10 触发器管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| TriggerManagerNodeDefinition | 触发器管理器主节点 | triggerManagerNode.h |
| SetTriggerState_NodeType | 设置触发器状态 | triggerManagerNodeType.h |

### 2.11 日志管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| JournalNodeDefinition | 日志节点主节点 | journalNode.h |
| JournalEntry_NodeType | 日志条目 | journalNode.h |
| JournalQuestEntry_NodeType | 任务日志条目 | journalNode.h |
| JournalQuestObjectiveCounter_NodeType | 任务目标计数器 | journalNode.h |
| JournalTrackQuest_NodeType | 追踪任务 | journalNode.h |
| JournalChangeMappinPhase_NodeType | 改变地图标记阶段 | journalNode.h |
| JournalBulkUpdate_NodeType | 批量更新 | journalNode.h |

### 2.12 电话管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| PhoneManagerNodeDefinition | 电话管理器主节点 | phoneManagerNode.h |
| AddRemoveContact_NodeType | 添加/移除联系人 | phoneManagerNodeType.h |
| SetPhoneStatus_NodeType | 设置电话状态 | phoneManagerNodeType.h |
| CallContact_NodeType | 呼叫联系人 | phoneManagerNodeType.h |
| SendMessage_NodeType | 发送消息 | phoneManagerNodeType.h |
| OpenMessage_NodeType | 打开消息 | phoneManagerNodeType.h |
| CloseMessage_NodeType | 关闭消息 | phoneManagerNodeType.h |
| SetCustomStyle_NodeType | 设置自定义样式 | phoneManagerNodeType.h |
| Minimize_NodeType | 最小化 | phoneManagerNodeType.h |
| RemoveAllContacts_NodeType | 移除所有联系人 | phoneManagerNodeType.h |
| SetPhoneRestriction_NodeType | 设置电话限制 | phoneManagerNodeType.h |

### 2.13 场景管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| SceneManagerNodeDefinition | 场景管理器主节点 | sceneManagerNode.h |
| SetTier_NodeType | 设置Tier等级 | sceneManagerNodeType.h |
| EnablePlayerGameplayLookAt_NodeType | 启用玩家游戏注视 | sceneManagerNodeType.h |
| SetTier2Params_NodeType | 设置Tier2参数 | sceneManagerNodeType.h |
| SetTier3Params_NodeType | 设置Tier3参数 | sceneManagerNodeType.h |
| SetTier4Params_NodeType | 设置Tier4参数 | sceneManagerNodeType.h |
| PlayerLookAt_NodeType | 玩家注视 | sceneManagerNodeType.h |
| NPCLookAt_NodeType | NPC注视 | sceneManagerNodeType.h |
| PrepareBlendCamera_NodeType | 准备混合相机 | sceneManagerNodeType.h |
| CameraClippingPlane_NodeType | 相机裁剪平面 | sceneManagerNodeType.h |
| SetFOV_NodeType | 设置视野 | sceneManagerNodeType.h |
| SetPossesion_NodeType | 设置附体 | sceneManagerNodeType.h |

### 2.14 UI 管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| UIManagerNodeDefinition | UI管理器主节点 | uiManagerNode.h |
| AddCombatLogMessage_NodeType | 添加战斗日志消息 | uiManagerNodeType.h |
| SwitchNameplate_NodeType | 切换名牌 | uiManagerNodeType.h |
| AddBraindanceClue_NodeType | 添加脑舞线索 | uiManagerNodeType.h |
| DiscoverBraindanceClue_NodeType | 发现脑舞线索 | uiManagerNodeType.h |
| DisplayMessageBox_NodeType | 显示消息框 | uiManagerNodeType.h |
| ProgressBar_NodeType | 进度条 | uiManagerNodeType.h |
| ProximityProgressBar_NodeType | 接近度进度条 | uiManagerNodeType.h |
| ShowDialogIndicator_NodeType | 显示对话指示器 | uiManagerNodeType.h |
| HUDVideo_NodeType | HUD视频 | uiManagerNodeType.h |
| SetLocationName_NodeType | 设置位置名称 | uiManagerNodeType.h |
| WarningMessage_NodeType | 警告消息 | uiManagerNodeType.h |
| ShowOnscreen_NodeType | 屏幕显示 | uiManagerNodeType.h |
| OverrideLoadingScreen_NodeType | 覆盖加载屏幕 | uiManagerNodeType.h |
| GlitchLoadingScreen_NodeType | 故障加载屏幕 | uiManagerNodeType.h |
| WaitForAnyKeyLoadingScreen_NodeType | 等待任意键加载屏幕 | uiManagerNodeType.h |
| SetUIGameContext_NodeType | 设置UI游戏上下文 | uiManagerNodeType.h |
| SetHUDEntryForcedVisibility_NodeType | 设置HUD条目强制可见性 | uiManagerNodeType.h |
| QuickItemsManager_NodeType | 快速物品管理器 | uiManagerNodeType.h |
| VendorPanel_NodeType | 商贩面板 | uiManagerNodeType.h |
| OpenBriefing_NodeType | 打开简报 | uiManagerNodeType.h |
| EnableBraindanceFinish_NodeType | 启用脑舞完成 | uiManagerNodeType.h |
| SwitchToScenario_NodeType | 切换到场景 | uiManagerNodeType.h |
| SetBriefingSize_NodeType | 设置简报大小 | uiManagerNodeType.h |
| SetBriefingAlignment_NodeType | 设置简报对齐 | uiManagerNodeType.h |
| ShowNarrativeEvent_NodeType | 显示叙事事件 | uiManagerNodeType.h |
| ShowCustomTooltip_NodeType | 显示自定义提示 | uiManagerNodeType.h |
| Tutorial_NodeType | 教程 | uiManagerNodeType.h |
| ToggleMinimapVisibility_NodeSubType | 切换小地图可见性 | uiManagerNodeType.h |
| ToggleStealthMappinVisibility_NodeSubType | 切换潜行地图标记可见性 | uiManagerNodeType.h |
| ShowHighlight_NodeSubType | 显示高亮 | uiManagerNodeType.h |
| ShowBracket_NodeSubType | 显示括号 | uiManagerNodeType.h |
| ShowOverlay_NodeSubType | 显示覆盖层 | uiManagerNodeType.h |
| ShowPopup_NodeSubType | 显示弹出窗口 | uiManagerNodeType.h |
| BriefingSequencePlayer_NodeType | 简报序列播放器 | uiManagerNodeType.h |
| TriggerIconGeneration_NodeType | 触发图标生成 | uiManagerNodeType.h |
| InputHint_NodeType | 输入提示 | uiManagerNodeType.h |
| InputHintGroup_NodeType | 输入提示组 | uiManagerNodeType.h |
| ShowLevelUpNotification_NodeType | 显示升级通知 | uiManagerNodeType.h |
| ShowCustomQuestNotification_NodeType | 显示自定义任务通知 | uiManagerNodeType.h |
| SetMetaQuestProgress_NodeType | 设置元任务进度 | uiManagerNodeType.h |
| SetSaveDataLoadingScreen_NodeType | 设置存档数据加载屏幕 | uiManagerNodeType.h |
| SetFastTravelBinksGroup_NodeType | 设置快速旅行视频组 | uiManagerNodeType.h |
| OpenPhotoMode_NodeType | 打开照片模式 | uiManagerNodeType.h |
| ShowPointOfNoReturnPrompt_NodeType | 显示不归路提示 | uiManagerNodeType.h |
| FinalBoardsVideosFinished_NodeType | 最终板视频完成 | uiManagerNodeType.h |
| FinalBoardsEnableSkipCredits_NodeType | 最终板启用跳过制作人员名单 | uiManagerNodeType.h |
| FinalBoardsOpenSpeakerScreen_NodeType | 最终板打开扬声器屏幕 | uiManagerNodeType.h |

### 2.15 车辆管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| VehicleNodeDefinition | 车辆管理器主节点 | vehicleManagerNode.h |
| AssignCharacter_NodeType | 分配角色 | vehicleManagerNodeType.h |
| SetVehicleCamera_NodeType | 设置车辆相机(已废弃) | vehicleManagerNodeType.h |
| RequestVehicleCameraPerspective_NodeType | 请求车辆相机视角 | vehicleManagerNodeType.h |
| MoveOnSpline_NodeType | 在样条上移动 | vehicleManagerNodeType.h |
| ToggleCombatForPlayer_NodeType | 切换玩家战斗 | vehicleManagerNodeType.h |
| ToggleSwitchSeatsForPlayer_NodeType | 切换玩家座位 | vehicleManagerNodeType.h |
| MoveOnSplineAndKeepDistance_NodeType | 在样条上移动并保持距离 | vehicleManagerNodeType.h |
| MoveOnSplineControlRubberbanding_NodeType | 在样条上移动控制橡皮筋效果 | vehicleManagerNodeType.h |
| StartVehicle_NodeType | 启动车辆 | vehicleManagerNodeType.h |
| StopVehicle_NodeType | 停止车辆 | vehicleManagerNodeType.h |
| FollowObject_NodeType | 跟随物体 | vehicleManagerNodeType.h |
| ResetMovement_NodeType | 重置移动 | vehicleManagerNodeType.h |
| SetAutopilot_NodeType | 设置自动驾驶 | vehicleManagerNodeType.h |
| ToggleBrokenTire_NodeType | 切换轮胎损坏 | vehicleManagerNodeType.h |
| ToggleForceBrake_NodeType | 切换强制制动 | vehicleManagerNodeType.h |
| FlushAutopilot_NodeType | 刷新自动驾驶 | vehicleManagerNodeType.h |
| ToggleTankCustomFPPLockOff_NodeType | 切换坦克自定义FPP锁定 | vehicleManagerNodeType.h |
| ToggleWeaponEnabled_NodeType | 切换武器启用 | vehicleManagerNodeType.h |
| OverrideSplineSpeed_NodeType | 覆盖样条速度 | vehicleManagerNodeType.h |
| Repair_NodeType | 修理 | vehicleManagerNodeType.h |
| ToggleDoor_NodeType | 切换车门 | vehicleManagerNodeType.h |
| SpawnPlayerVehicle_NodeType | 生成玩家车辆 | vehicleManagerNodeType.h |
| Teleport_NodeType | 传送 | vehicleManagerNodeType.h |
| ForbiddenTrigger_NodeType | 禁止触发器 | vehicleManagerNodeType.h |
| EnableVehicleSummon_NodeType | 启用车辆召唤 | vehicleManagerNodeType.h |
| EnablePlayerVehicle_NodeType | 启用玩家车辆 | vehicleManagerNodeType.h |
| ToggleWindow_NodeType | 切换车窗 | vehicleManagerNodeType.h |
| UnassignAll_NodeType | 取消所有分配 | vehicleManagerNodeType.h |
| ForcePhysicsWakeUp_NodeType | 强制物理唤醒 | vehicleManagerNodeType.h |
| SetImmovable_NodeType | 设置不可移动 | vehicleManagerNodeType.h |

### 2.16 音频管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| AudioNodeDefinition | 音频节点主节点 | audioNode.h |
| AudioCharacterManagerNodeDefinition | 角色音频管理器 | audioCharacterManagerNode.h |
| AudioMixNodeType | 音频混合 | audioNode.h |
| AudioParameterNodeType | 音频参数 | audioParameterNode.h |
| AudioMusicSyncNodeType | 音乐同步 | audioMusicSyncNode.h |
| AudioSwitchNodeType | 音频开关 | audioSwitchNode.h |
| AudioFocusNodeType | 音频焦点 | audioFocusNode.h |
| AudioLoadingNodeType | 音频加载 | audioLoadingNode.h |
| AudioCharacterSystemsManager_NodeType | 角色音频系统管理器 | audioCharacterSystemsManagerNodeType.h |
| AudioCharacterManagerBreathing_NodeSubType | 呼吸音频 | audioCharacterManagerBreathingNodeSubType.h |
| AudioCharacterManagerFootsteps_NodeSubType | 脚步音频 | audioCharacterManagerFootstepsNodeSubType.h |

### 2.17 行为管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| BehaviourManagerNodeDefinition | 行为管理器主节点 | behaviourManagerNode.h |
| JumpWorkspotAnim_NodeType | 跳转工作点动画 | behaviourManagerNode.h |
| StopWorkspot_NodeType | 停止工作点 | behaviourManagerNode.h |

### 2.18 事实数据库管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| FactsDBManagerNodeDefinition | 事实数据库管理器主节点 | factsDBManagerNode.h |
| SetVar_NodeType | 设置变量 | factsDBManagerNodeType.h |

### 2.19 地图标记管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| MapPinManagerNodeDefinition | 地图标记管理器 | mapPinManagerNode.h |
| MappinManagerNodeDefinition | 地图标记管理器(新版) | mapPinManagerNode.h |

### 2.20 奖励管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| RewardManagerNodeDefinition | 奖励管理器主节点 | rewardManagerNode.h |
| GiveReward_NodeType | 给予奖励 | rewardManagerNodeType.h |

### 2.21 其他管理器

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| SpawnManagerNodeDefinition | 生成管理器主节点 | spawnManagerNode.h |
| TimeManagerNodeDefinition | 时间管理器主节点 | timeManagerNode.h |
| VisionModesManagerNodeDefinition | 视觉模式管理器主节点 | visionModesManagerNode.h |
| VoicesetManagerNodeDefinition | 语音集管理器主节点 | voicesetManagerNode.h |
| RecordingNodeDefinition | 录制节点主节点 | recordingNode.h |

---

## 3. AI 命令节点 (AI Command Nodes)

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| AICommandNodeBase | AI命令节点基类 | aiCommandNodeBase.h |
| ConfigurableAICommandNode | 可配置AI命令节点 | questConfigurableAICommandNode.h |
| SendAICommandNodeDefinition | 发送AI命令 | sendAICommandNode.h |
| CombatNodeDefinition | 战斗节点 | combatNode.h |
| MovePuppetNodeDefinition | 移动Puppet | movePuppetNode.h |
| MiscAICommandNode | 杂项AI命令 | miscAICommandNode.h |
| TeleportPuppetNodeDefinition | 传送Puppet | teleportPuppetNode.h |
| EquipItemNodeDefinition | 装备物品 | equipItemNode.h |
| UnequipItemNodeDefinition | 卸下物品 | unequipItemNode.h |
| UseWorkspotNodeDefinition | 使用工作点 | useWorkspotNode.h |
| RotateToNodeDefinition | 旋转到目标 | rotateToNode.h |
| VehicleNodeCommandDefinition | 车辆命令 | vehicleCommandNode.h |
| ForcedBehaviourNodeDefinition | 强制行为 | forcedBehaviourNode.h |
| ClearForcedBehavioursNodeDefinition | 清除强制行为 | forcedBehaviourNode.h |
| LookAtDrivenTurnsNode | 注视驱动转向 | lookAtDrivenTurnsNode.h |

---

## 4. 逻辑节点 (Logical Nodes)

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| LogicalBaseNodeDefinition | 逻辑节点基类 | logicalBaseNode.h |
| LogicalAndNodeDefinition | 逻辑与节点 | logicalAndNode.h |
| LogicalXorNodeDefinition | 逻辑异或节点 | logicalXorNode.h |
| LogicalHubNodeDefinition | 逻辑Hub节点 | logicalHubNode.h |

---

## 5. 条件节点 (Condition Nodes)

### 5.1 条件基础类

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| IBaseCondition | 条件基类接口 | condition.h |
| Condition | 条件类 | condition.h |
| TypedCondition | 带类型的条件 | condition.h |
| LogicalCondition | 逻辑条件 | logicalCondition.h |
| ConditionNodeDefinition | 条件节点 | conditionNode.h |
| PauseConditionNodeDefinition | 暂停条件节点 | pauseConditionNode.h |

### 5.2 对象条件

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| ObjectCondition | 对象条件 | objectCondition.h |
| Interaction_ConditionType | 交互条件 | objectConditionType.h |
| Inventory_ConditionType | 库存条件 | objectConditionType.h |
| Inspect_ConditionType | 检查条件 | objectConditionType.h |
| Scan_ConditionType | 扫描条件 | objectConditionType.h |
| EntryScanned_ConditionType | 条目扫描条件 | objectConditionType.h |
| Device_ConditionType | 设备条件 | objectConditionType.h |
| Destruction_ConditionType | 破坏条件 | objectConditionType.h |
| Tagged_ConditionType | 标记条件 | objectConditionType.h |

### 5.3 支付条件

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| PaymentCondition | 支付条件 | paymentCondition.h |
| PaymentBalanced_ConditionType | 平衡支付条件 | paymentConditionType.h |
| PaymentFixedAmount_ConditionType | 固定金额支付条件 | paymentConditionType.h |

### 5.4 属性条件

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| StatsCondition | 属性条件 | statsCondition.h |
| Stat_ConditionType | 属性条件 | statsConditionType.h |
| StreetCredTier_ConditionType | 街头声望等级条件 | statsConditionType.h |
| LifePath_ConditionType | 人生轨迹条件 | statsConditionType.h |
| Build_ConditionType | 构建条件 | statsConditionType.h |

### 5.5 系统条件

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| CameraFocus_ConditionType | 相机焦点条件 | systemConditionType.h |
| VisionMode_ConditionType | 视觉模式条件 | systemConditionType.h |
| Platform_ConditionType | 平台条件 | systemConditionType.h |
| InputAction_ConditionType | 输入动作条件 | systemConditionType.h |
| InputController_ConditionType | 输入控制器条件 | systemConditionType.h |
| Phone_ConditionType | 电话条件 | systemConditionType.h |
| PhonePickUp_ConditionType | 电话接听条件 | systemConditionType.h |
| Prereq_ConditionType | 前置条件 | systemConditionType.h |
| Weather_ConditionType | 天气条件 | systemConditionType.h |
| Radio_ConditionType | 电台条件 | systemConditionType.h |
| RadioTrack_ConditionType | 电台曲目条件 | systemConditionType.h |
| PlaylistTrackChanged_ConditionType | 播放列表曲目变更条件 | systemConditionType.h |
| Language_ConditionType | 语言条件 | systemConditionType.h |
| GOGReward_ConditionType | GOG奖励条件 | systemConditionType.h |
| SaveLock_ConditionType | 存档锁定条件 | systemConditionType.h |

### 5.6 时间条件

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| TimeCondition | 时间条件 | timeCondition.h |
| RealtimeDelay_ConditionType | 实时延迟条件 | timeConditionType.h |
| GameTimeDelay_ConditionType | 游戏时间延迟条件 | timeConditionType.h |
| TimePeriod_ConditionType | 时间段条件 | timeConditionType.h |

---

## 6. 流程控制节点 (Flow Control Nodes)

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| FlowControlNodeDefinition | 流程控制节点 | flowControlNode.h |
| SwitchNodeDefinition | 开关节点 | switchNode.h |
| RandomizerNodeDefinition | 随机器节点 | randomizerNode.h |
| CheckpointNodeDefinition | 检查点节点 | checkpointNode.h |
| EmbeddedGraphNodeDefinition | 嵌入式图节点 | embeddedGraphNode.h |
| PhaseNodeDefinition | 阶段节点 | phaseNode.h |
| DeletionMarkerNodeDefinition | 删除标记节点 | deletionMarkerNode.h |

---

## 7. 多人游戏节点 (Multiplayer Nodes)

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| MultiplayerAIDirectorNodeDefinition | 多人游戏AI导演节点 | multiplayerAIDirectorNode.h |
| MultiplayerChoiceTokenNodeDefinition | 多人游戏选择令牌节点 | multiplayerChoiceTokenNode.h |
| MultiplayerJunctionDialogNodeDefinition | 多人游戏交汇对话节点 | multiplayerJunctionDialogNode.h |
| MultiplayerTeleportPuppetNodeDefinition | 多人游戏传送Puppet节点 | multiplayerTeleportPuppetNode.h |

---

## 8. 杂项节点 (Miscellaneous Nodes)

| 节点类型 | 功能描述 | 所在文件 |
|---------|---------|---------|
| BaseObjectNodeDefinition | 基础对象节点 | baseObjectNode.h |
| CutControlNodeDefinition | 剪辑控制节点 | cutControlNode.h |
| MinigameNodeDefinition | 小游戏节点 | minigameNode.h |
| PlaceholderNodeDefinition | 占位符节点 | placeholderNode.h |
| PuppeteerNodeDefinition | 操纵者节点 | puppeteer.h |
| PuppetAIManagerNodeDefinition | Puppet AI管理器节点 | puppetAIManager.h |
| PopulactionControllerNodeDefinition | 人口控制器节点 | populationControllerNode.h |
| InstancedCrowdControlNodeDefinition | 实例化人群控制节点 | instancedCrowdControlNode.h |
| TransformAnimatorNodeDefinition | 变换动画器节点 | transformAnimatorNode.h |
| TeleportVehicleNodeDefinition | 传送车辆节点 | teleportVehicleNode.h |
| WorkspotParamNodeDefinition | 工作点参数节点 | workspotParamNode.h |

---

## 统计总结

| 分类 | 节点数量 |
|------|---------|
| 核心节点 | 12 |
| 管理器节点 | 180+ |
| AI 命令节点 | 14 |
| 逻辑节点 | 4 |
| 条件节点 | 45+ |
| 流程控制节点 | 7 |
| 多人游戏节点 | 4 |
| 杂项节点 | 11 |
| **总计** | **277+** |

---

**文档生成完成**
保存位置: E:\World\2077节点内容\quest_nodes_table.md
