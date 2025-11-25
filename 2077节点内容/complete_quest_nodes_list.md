# Cyberpunk 2077 完整 Quest 节点类型列表

> 源文件目录: `D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev\src\common\gameQuest\include`
>
> 生成日期: 2025-11-23
>
> 总计: 192 个头文件

## 目录

1. [核心节点定义 (Core Node Definitions)](#1-核心节点定义)
2. [管理器节点 (Manager Nodes)](#2-管理器节点)
3. [AI 命令节点 (AI Command Nodes)](#3-ai-命令节点)
4. [逻辑节点 (Logical Nodes)](#4-逻辑节点)
5. [条件节点 (Condition Nodes)](#5-条件节点)
6. [流程控制节点 (Flow Control Nodes)](#6-流程控制节点)
7. [多人游戏节点 (Multiplayer Nodes)](#7-多人游戏节点)
8. [杂项节点 (Miscellaneous Nodes)](#8-杂项节点)

---

## 1. 核心节点定义 (Core Node Definitions)

### 基础节点类
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `NodeDefinition` | node.h | 所有Quest节点的基类 |
| `DisableableNodeDefinition` | disableableNode.h | 可禁用的节点基类 |
| `SignalStoppingNodeDefinition` | signalStoppingNode.h | 可阻断信号传播的节点基类 |
| `TypedSignalStoppingNodeDefinition` | signalStoppingNode.h | 带类型的信号阻断节点 |
| `StartEndNodeDefinition` | startEndNode.h | 开始/结束节点基类 |
| `GraphDefinition` | graph.h | Quest图定义 |
| `SocketDefinition` | socket.h | Socket定义 |

### 开始/结束节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `StartNodeDefinition` | startNode.h | Quest开始节点 |
| `EndNodeDefinition` | endNode.h | Quest结束节点 |

### I/O 节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `IONodeDefinition` | ioNode.h | 输入/输出节点基类 |
| `InputNodeDefinition` | inputNode.h | 输入节点 |
| `OutputNodeDefinition` | outputNode.h | 输出节点 |

---

## 2. 管理器节点 (Manager Nodes)

### 2.1 角色管理器 (Character Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `CharacterManagerNodeDefinition` | characterManagerNode.h | 角色管理器主节点 |

#### 节点类型 (Node Types)
| 节点类型 | 所在文件 | 描述 |
|---------|---------|------|
| `ICharacterManager_NodeType` | characterManagerNodeType.h | 角色管理器节点类型接口 |
| `ICharacterManager_NodeSubType` | characterManagerNodeType.h | 角色管理器子类型接口 |
| `CharacterManagerParameters_NodeType` | characterManagerNodeTypeImpl.h | 参数管理 |
| `CharacterManagerCombat_NodeType` | characterManagerNodeTypeImpl.h | 战斗管理 |
| `CharacterManagerVisuals_NodeType` | characterManagerNodeTypeImpl.h | 视觉效果管理 |

#### 参数子类型 (Parameters SubTypes)
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `CharacterManagerParameters_SetAttitudeGroupForPuppet` | characterManagerNodeTypeImpl.h | 设置AI态度组 |
| `CharacterManagerParameters_SetGroupsAttitude` | characterManagerNodeTypeImpl.h | 设置组态度 |
| `CharacterManagerParameters_SetMortality` | characterManagerNodeTypeImpl.h | 设置生死状态 |
| `CharacterManagerParameters_SetAnimset` | characterManagerNodeTypeImpl.h | 设置动画集 |
| `CharacterManagerParameters_SetLowGravity` | characterManagerNodeTypeImpl.h | 设置低重力 |
| `CharacterManagerParameters_EnableBumps` | characterManagerNodeTypeImpl.h | 启用碰撞 |
| `CharacterManagerParameters_SetStatusEffect` | characterManagerNodeTypeImpl.h | 设置状态效果 |
| `CharacterManagerParameters_SetReactionPreset` | characterManagerNodeTypeImpl.h | 设置反应预设 |
| `CharacterManagerParameters_SetGender` | characterManagerNodeTypeImpl.h | 设置性别 |
| `CharacterManagerParameters_SetAsCrowdObstacle` | characterManagerNodeTypeImpl.h | 设为人群障碍物 |
| `CharacterManagerParameters_SetProgressionBuild` | characterManagerNodeTypeImpl.h | 设置进度构建 |
| `CharacterManagerParameters_SetLifePath` | characterManagerNodeTypeImpl.h | 设置人生轨迹 |
| `CharacterManagerParameters_HealPlayer` | characterManagerNodeTypeImpl.h | 治疗玩家 |

#### 战斗子类型 (Combat SubTypes)
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `CharacterManagerCombat_ModifyHealth` | characterManagerNodeTypeImpl.h | 修改生命值 |
| `CharacterManagerCombat_Kill` | characterManagerNodeTypeImpl.h | 杀死角色 |
| `CharacterManagerCombat_EquipWeapon` | characterManagerNodeTypeImpl.h | 装备武器 |
| `CharacterManagerCombat_SetWeaponState` | characterManagerNodeTypeImpl.h | 设置武器状态 |
| `CharacterManagerCombat_SetDeathDirection` | characterManagerNodeTypeImpl.h | 设置死亡方向 |
| `CharacterManagerCombat_ChangeLevel` | characterManagerNodeTypeImpl.h | 改变等级 |
| `CharacterManagerCombat_ManageRagdoll` | characterManagerNodeTypeImpl.h | 管理布娃娃系统 |
| `CharacterManagerCombat_AssignSquad` | characterManagerNodeTypeImpl.h | 分配小队 |
| `CharacterManagerParameters_SetCombatSpace` | characterManagerNodeTypeImpl.h | 设置战斗空间 |

#### 视觉子类型 (Visuals SubTypes)
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `CharacterManagerVisuals_EntityAppearanceOperationBase` | characterManagerNodeTypeImpl.h | 实体外观操作基类 |
| `CharacterManagerVisuals_ChangeEntityAppearance` | characterManagerNodeTypeImpl.h | 改变实体外观 |
| `CharacterManagerVisuals_PrefetchEntityAppearance` | characterManagerNodeTypeImpl.h | 预加载实体外观 |
| `CharacterManagerVisuals_GenitalsManager` | characterManagerNodeTypeImpl.h | 生殖器管理 |
| `CharacterManagerVisuals_BreastSizeController` | characterManagerNodeTypeImpl.h | 胸部大小控制 |
| `CharacterManagerVisuals_SetBrokenNoseStage` | characterManagerNodeTypeImpl.h | 设置鼻梁破损阶段 |

---

### 2.2 实体管理器 (Entity Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `EntityManagerNodeDefinition` | entityManagerNode.h | 实体管理器主节点 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `IEntityManager_NodeType` | entityManagerNodeType.h | 实体管理器节点类型接口 |
| `EntityManagerSetAttachment_NodeType` | entityManagerNodeTypeImpl.h | 设置附着 |
| `EntityManagerSetDestructionState_NodeType` | entityManagerNodeTypeImpl.h | 设置破坏状态 |
| `EntityManagerManageBinkComponent_NodeType` | entityManagerNodeTypeImpl.h | 管理Bink组件 |
| `EntityManagerSetMeshAppearance_NodeType` | entityManagerNodeTypeImpl.h | 设置网格外观 |
| `EntityManagerEnablePlayerTPPRepresentation_NodeType` | entityManagerNodeTypeImpl.h | 启用玩家第三人称表示 |
| `EntityManagerToggleComponent_NodeType` | entityManagerNodeTypeImpl.h | 切换组件 |
| `EntityManagerChangeAppearance_NodeType` | entityManagerNodeTypeImpl.h | 改变外观 |
| `EntityManagerMountPuppet_NodeType` | entityManagerNodeTypeImpl.h | 骑乘Puppet |
| `EntityManagerSendAnimationEvent_NodeType` | entityManagerNodeTypeImpl.h | 发送动画事件 |
| `EntityManagerSetStat_NodeType` | entityManagerNodeTypeImpl.h | 设置属性 |
| `EntityManagerToggleMirrorsArea_NodeType` | entityManagerNodeTypeImpl.h | 切换镜像区域 |

#### 附着子类型 (Attachment SubTypes)
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `EntityManagerSetAttachment_ToActor` | entityManagerNodeTypeImpl.h | 附着到角色 |
| `EntityManagerDestroyCarriedObject` | entityManagerNodeTypeImpl.h | 销毁携带物体 |
| `EntityManagerSetAttachment_ToNode` | entityManagerNodeTypeImpl.h | 附着到节点 |
| `EntityManagerSetAttachment_ToWorld` | entityManagerNodeTypeImpl.h | 附着到世界 |

---

### 2.3 环境管理器 (Environment Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `EnvironmentManagerNodeDefinition` | environmentManagerNode.h | 环境管理器主节点 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `IEnvironmentManagerNodeType` | environmentManagerNodeType.h | 环境管理器节点类型接口 |
| `PlayEnv_NodeType` | environmentManagerNodeType.h | 播放环境 |
| `PlayEnv_OverrideGlobalLight` | environmentManagerNodeType.h | 覆盖全局光照 |
| `PlayEnv_ForceRelitEnvProbe` | environmentManagerNodeType.h | 强制重新照明环境探针 |
| `PlayEnv_SetWeather` | environmentManagerNodeType.h | 设置天气 |

---

### 2.4 游戏管理器 (Game Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `GameManagerNodeDefinition` | gameManagerNode.h | 游戏管理器主节点 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `IGameManagerNodeType` | gameManagerNodeType.h | 游戏管理器节点类型接口 |
| `IGameManagerNonSignalStoppingNodeType` | gameManagerNodeType.h | 非信号停止节点类型 |
| `TimeDilation_NodeType` | gameManagerNodeType.h | 时间膨胀 |
| `ControlObject_NodeType` | gameManagerNodeType.h | 控制物体 |
| `Replacer_NodeType` | gameManagerNodeType.h | 替换器 |
| `LootPurge_NodeType` | gameManagerNodeType.h | 清除战利品 |
| `ContentTokenManager_NodeType` | gameManagerNodeType.h | 内容令牌管理器 |
| `GameplayRestrictions_NodeType` | gameManagerNodeType.h | 游戏限制 |
| `SetTimer_NodeType` | gameManagerNodeType.h | 设置计时器 |
| `Rumble_NodeType` | gameManagerNodeType.h | 震动 |

#### 时间膨胀子类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `TimeDilation_World` | gameManagerNodeType.h | 世界时间膨胀 |
| `TimeDilation_Player` | gameManagerNodeType.h | 玩家时间膨胀 |
| `TimeDilation_Entity` | gameManagerNodeType.h | 实体时间膨胀 |
| `TimeDilation_Puppet` | gameManagerNodeType.h | Puppet时间膨胀 |

#### 内容令牌子类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `SpawnToken_NodeSubType` | gameManagerNodeType.h | 生成令牌 |
| `RemoveToken_NodeSubType` | gameManagerNodeType.h | 移除令牌 |
| `ForceTokenActivation_NodeSubType` | gameManagerNodeType.h | 强制令牌激活 |
| `BlockTokenActivation_NodeSubType` | gameManagerNodeType.h | 阻止令牌激活 |

---

### 2.5 事件管理器 (Event Manager)

| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `EventManagerNodeDefinition` | eventManagerNode.h | 事件管理器节点 |

---

### 2.6 特效管理器 (FX Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `FXManagerNodeDefinition` | fxManagerNode.h | 特效管理器主节点 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `IFXManagerNodeType` | fxManagerNodeType.h | 特效管理器节点类型接口 |
| `PlayFX_NodeType` | fxManagerNodeType.h | 播放特效 |
| `PreloadFX_NodeType` | fxManagerNodeType.h | 预加载特效 |

---

### 2.7 渲染特效管理器 (Render FX Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `RenderFxManagerNodeDefinition` | renderFxManagerNode.h | 渲染特效管理器主节点 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `IRenderFxManagerNodeType` | renderFxManagerNodeType.h | 渲染特效管理器节点类型接口 |
| `SetFadeInOut_NodeType` | renderFxManagerNodeType.h | 设置淡入淡出 |
| `SetDebugView_NodeType` | renderFxManagerNodeType.h | 设置调试视图 |
| `SetCyberspacePostFX_NodeType` | renderFxManagerNodeType.h | 设置赛博空间后处理 |
| `EnforceScreenSpaceReflectionsUberQuality_NodeType` | renderFxManagerNodeType.h | 强制屏幕空间反射超高质量 |
| `RenderPlane_NodeType` | renderFxManagerNodeType.h | 渲染平面 |
| `SetRenderLayer_NodeType` | renderFxManagerNodeType.h | 设置渲染层 |

---

### 2.8 物品管理器 (Item Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `ItemManagerNodeDefinition` | itemManagerNode.h | 物品管理器主节点 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `IItemManagerNodeType` | itemManagerNodeType.h | 物品管理器节点类型接口 |
| `AddRemoveItem_NodeType` | itemManagerNodeType.h | 添加/移除物品 |
| `DropItemFromSlot_NodeType` | itemManagerNodeType.h | 从槽位丢弃物品 |
| `SetItemTags_NodeType` | itemManagerNodeType.h | 设置物品标签 |
| `TransferItem_NodeType` | itemManagerNodeType.h | 转移物品 |
| `LootTokenManager_NodeType` | itemManagerNodeType.h | 战利品令牌管理器 |
| `UseWeapon_NodeType` | itemManagerNodeType.h | 使用武器 |
| `SetLootInteractionAccess_NodeType` | itemManagerNodeType.h | 设置战利品交互访问 |
| `InjectLoot_NodeType` | itemManagerNodeType.h | 注入战利品 |

---

### 2.9 交互对象管理器 (Interactive Object Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `InteractiveObjectManagerNodeDefinition` | interactiveObjectManagerNode.h | 交互对象管理器主节点 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `IInteractiveObjectManagerNodeType` | interactiveObjectManagerNodeType.h | 交互对象管理器节点类型接口 |
| `SetInteractionState_NodeType` | interactiveObjectManagerNodeType.h | 设置交互状态 |
| `SetInteractionVisualizerOverride` | interactiveObjectManagerNodeType.h | 设置交互可视化覆盖 |
| `Elevator_ManageNPCAttachment_NodeType` | interactiveObjectManagerNodeType.h | 电梯NPC附着管理 |
| `HackingManager_NodeType` | interactiveObjectManagerNodeType.h | 黑客管理器 |
| `Cyberdrill_NodeType` | interactiveObjectManagerNodeType.h | 电钻 |
| `SetConveyorState_NodeType` | interactiveObjectManagerNodeType.h | 设置传送带状态 |
| `DeviceManager_NodeType` | interactiveObjectManagerNodeType.h | 设备管理器 |
| `SetInspectMode_NodeType` | interactiveObjectManagerNodeType.h | 设置检查模式 |

#### 黑客管理子类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `HackingManager_SetEnabled` | interactiveObjectManagerNodeType.h | 设置启用 |
| `HackingManager_SetHacked` | interactiveObjectManagerNodeType.h | 设置已黑客 |

---

### 2.10 触发器管理器 (Trigger Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `TriggerManagerNodeDefinition` | triggerManagerNode.h | 触发器管理器主节点 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `ITriggerManagerNodeType` | triggerManagerNodeType.h | 触发器管理器节点类型接口 |
| `SetTriggerState_NodeType` | triggerManagerNodeType.h | 设置触发器状态 |

---

### 2.11 日志管理器 (Journal Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `JournalNodeDefinition` | journalNode.h | 日志节点主节点 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `IJournal_NodeType` | journalNode.h | 日志节点类型接口 |
| `JournalEntry_NodeType` | journalNode.h | 日志条目 |
| `JournalQuestEntry_NodeType` | journalNode.h | 任务日志条目 |
| `JournalQuestObjectiveCounter_NodeType` | journalNode.h | 任务目标计数器 |
| `JournalTrackQuest_NodeType` | journalNode.h | 追踪任务 |
| `JournalChangeMappinPhase_NodeType` | journalNode.h | 改变地图标记阶段 |
| `JournalBulkUpdate_NodeType` | journalNode.h | 批量更新 |

---

### 2.12 电话管理器 (Phone Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `PhoneManagerNodeDefinition` | phoneManagerNode.h | 电话管理器主节点 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `IPhoneManagerNodeType` | phoneManagerNodeType.h | 电话管理器节点类型接口 |
| `AddRemoveContact_NodeType` | phoneManagerNodeType.h | 添加/移除联系人 |
| `SetPhoneStatus_NodeType` | phoneManagerNodeType.h | 设置电话状态 |
| `CallContact_NodeType` | phoneManagerNodeType.h | 呼叫联系人 |
| `SendMessage_NodeType` | phoneManagerNodeType.h | 发送消息 |
| `OpenMessage_NodeType` | phoneManagerNodeType.h | 打开消息 |
| `CloseMessage_NodeType` | phoneManagerNodeType.h | 关闭消息 |
| `SetCustomStyle_NodeType` | phoneManagerNodeType.h | 设置自定义样式 |
| `Minimize_NodeType` | phoneManagerNodeType.h | 最小化 |
| `RemoveAllContacts_NodeType` | phoneManagerNodeType.h | 移除所有联系人 |
| `SetPhoneRestriction_NodeType` | phoneManagerNodeType.h | 设置电话限制 |

---

### 2.13 场景管理器 (Scene Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `SceneManagerNodeDefinition` | sceneManagerNode.h | 场景管理器主节点 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `ISceneManagerNodeType` | sceneManagerNodeType.h | 场景管理器节点类型接口 |
| `SetTier_NodeType` | sceneManagerNodeType.h | 设置Tier等级 |
| `EnablePlayerGameplayLookAt_NodeType` | sceneManagerNodeType.h | 启用玩家游戏注视 |
| `SetTier2Params_NodeType` | sceneManagerNodeType.h | 设置Tier2参数 |
| `SetTier3Params_NodeType` | sceneManagerNodeType.h | 设置Tier3参数 |
| `SetTier4Params_NodeType` | sceneManagerNodeType.h | 设置Tier4参数 |
| `PlayerLookAt_NodeType` | sceneManagerNodeType.h | 玩家注视 |
| `NPCLookAt_NodeType` | sceneManagerNodeType.h | NPC注视 |
| `PrepareBlendCamera_NodeType` | sceneManagerNodeType.h | 准备混合相机 |
| `CameraClippingPlane_NodeType` | sceneManagerNodeType.h | 相机裁剪平面 |
| `SetFOV_NodeType` | sceneManagerNodeType.h | 设置视野 |
| `SetPossesion_NodeType` | sceneManagerNodeType.h | 设置附体 |

---

### 2.14 UI 管理器 (UI Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `UIManagerNodeDefinition` | uiManagerNode.h | UI管理器主节点 |

#### 节点类型 (部分列表)
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `IUIManagerNodeType` | uiManagerNodeType.h | UI管理器节点类型接口 |
| `AddCombatLogMessage_NodeType` | uiManagerNodeType.h | 添加战斗日志消息 |
| `SwitchNameplate_NodeType` | uiManagerNodeType.h | 切换名牌 |
| `AddBraindanceClue_NodeType` | uiManagerNodeType.h | 添加脑舞线索 |
| `DiscoverBraindanceClue_NodeType` | uiManagerNodeType.h | 发现脑舞线索 |
| `DisplayMessageBox_NodeType` | uiManagerNodeType.h | 显示消息框 |
| `ProgressBar_NodeType` | uiManagerNodeType.h | 进度条 |
| `ProximityProgressBar_NodeType` | uiManagerNodeType.h | 接近度进度条 |
| `ShowDialogIndicator_NodeType` | uiManagerNodeType.h | 显示对话指示器 |
| `HUDVideo_NodeType` | uiManagerNodeType.h | HUD视频 |
| `SetLocationName_NodeType` | uiManagerNodeType.h | 设置位置名称 |
| `WarningMessage_NodeType` | uiManagerNodeType.h | 警告消息 |
| `ShowOnscreen_NodeType` | uiManagerNodeType.h | 屏幕显示 |
| `OverrideLoadingScreen_NodeType` | uiManagerNodeType.h | 覆盖加载屏幕 |
| `GlitchLoadingScreen_NodeType` | uiManagerNodeType.h | 故障加载屏幕 |
| `WaitForAnyKeyLoadingScreen_NodeType` | uiManagerNodeType.h | 等待任意键加载屏幕 |
| `SetUIGameContext_NodeType` | uiManagerNodeType.h | 设置UI游戏上下文 |
| `SetHUDEntryForcedVisibility_NodeType` | uiManagerNodeType.h | 设置HUD条目强制可见性 |
| `QuickItemsManager_NodeType` | uiManagerNodeType.h | 快速物品管理器 |
| `VendorPanel_NodeType` | uiManagerNodeType.h | 商贩面板 |
| `OpenBriefing_NodeType` | uiManagerNodeType.h | 打开简报 |
| `EnableBraindanceFinish_NodeType` | uiManagerNodeType.h | 启用脑舞完成 |
| `SwitchToScenario_NodeType` | uiManagerNodeType.h | 切换到场景 |
| `SetBriefingSize_NodeType` | uiManagerNodeType.h | 设置简报大小 |
| `SetBriefingAlignment_NodeType` | uiManagerNodeType.h | 设置简报对齐 |
| `ShowNarrativeEvent_NodeType` | uiManagerNodeType.h | 显示叙事事件 |
| `ShowCustomTooltip_NodeType` | uiManagerNodeType.h | 显示自定义提示 |
| `Tutorial_NodeType` | uiManagerNodeType.h | 教程 |
| `ToggleMinimapVisibility_NodeSubType` | uiManagerNodeType.h | 切换小地图可见性 |
| `ToggleStealthMappinVisibility_NodeSubType` | uiManagerNodeType.h | 切换潜行地图标记可见性 |
| `ShowHighlight_NodeSubType` | uiManagerNodeType.h | 显示高亮 |
| `ShowBracket_NodeSubType` | uiManagerNodeType.h | 显示括号 |
| `ShowOverlay_NodeSubType` | uiManagerNodeType.h | 显示覆盖层 |
| `ShowPopup_NodeSubType` | uiManagerNodeType.h | 显示弹出窗口 |
| `BriefingSequencePlayer_NodeType` | uiManagerNodeType.h | 简报序列播放器 |
| `TriggerIconGeneration_NodeType` | uiManagerNodeType.h | 触发图标生成 |
| `InputHint_NodeType` | uiManagerNodeType.h | 输入提示 |
| `InputHintGroup_NodeType` | uiManagerNodeType.h | 输入提示组 |
| `ShowLevelUpNotification_NodeType` | uiManagerNodeType.h | 显示升级通知 |
| `ShowCustomQuestNotification_NodeType` | uiManagerNodeType.h | 显示自定义任务通知 |
| `SetMetaQuestProgress_NodeType` | uiManagerNodeType.h | 设置元任务进度 |
| `SetSaveDataLoadingScreen_NodeType` | uiManagerNodeType.h | 设置存档数据加载屏幕 |
| `SetFastTravelBinksGroup_NodeType` | uiManagerNodeType.h | 设置快速旅行视频组 |
| `OpenPhotoMode_NodeType` | uiManagerNodeType.h | 打开照片模式 |
| `ShowPointOfNoReturnPrompt_NodeType` | uiManagerNodeType.h | 显示不归路提示 |
| `FinalBoardsVideosFinished_NodeType` | uiManagerNodeType.h | 最终板视频完成 |
| `FinalBoardsEnableSkipCredits_NodeType` | uiManagerNodeType.h | 最终板启用跳过制作人员名单 |
| `FinalBoardsOpenSpeakerScreen_NodeType` | uiManagerNodeType.h | 最终板打开扬声器屏幕 |

---

### 2.15 车辆管理器 (Vehicle Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `VehicleNodeDefinition` | vehicleManagerNode.h | 车辆管理器主节点 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `IVehicleManagerNodeType` | vehicleManagerNodeType.h | 车辆管理器节点类型接口 |
| `AssignCharacter_NodeType` | vehicleManagerNodeType.h | 分配角色 |
| `SetVehicleCamera_NodeType` | vehicleManagerNodeType.h | 设置车辆相机(已废弃) |
| `RequestVehicleCameraPerspective_NodeType` | vehicleManagerNodeType.h | 请求车辆相机视角 |
| `MoveOnSpline_NodeType` | vehicleManagerNodeType.h | 在样条上移动 |
| `ToggleCombatForPlayer_NodeType` | vehicleManagerNodeType.h | 切换玩家战斗 |
| `ToggleSwitchSeatsForPlayer_NodeType` | vehicleManagerNodeType.h | 切换玩家座位 |
| `MoveOnSplineAndKeepDistance_NodeType` | vehicleManagerNodeType.h | 在样条上移动并保持距离 |
| `MoveOnSplineControlRubberbanding_NodeType` | vehicleManagerNodeType.h | 在样条上移动控制橡皮筋效果 |
| `StartVehicle_NodeType` | vehicleManagerNodeType.h | 启动车辆 |
| `StopVehicle_NodeType` | vehicleManagerNodeType.h | 停止车辆 |
| `FollowObject_NodeType` | vehicleManagerNodeType.h | 跟随物体 |
| `ResetMovement_NodeType` | vehicleManagerNodeType.h | 重置移动 |
| `SetAutopilot_NodeType` | vehicleManagerNodeType.h | 设置自动驾驶 |
| `ToggleBrokenTire_NodeType` | vehicleManagerNodeType.h | 切换轮胎损坏 |
| `ToggleForceBrake_NodeType` | vehicleManagerNodeType.h | 切换强制制动 |
| `FlushAutopilot_NodeType` | vehicleManagerNodeType.h | 刷新自动驾驶 |
| `ToggleTankCustomFPPLockOff_NodeType` | vehicleManagerNodeType.h | 切换坦克自定义FPP锁定 |
| `ToggleWeaponEnabled_NodeType` | vehicleManagerNodeType.h | 切换武器启用 |
| `OverrideSplineSpeed_NodeType` | vehicleManagerNodeType.h | 覆盖样条速度 |
| `Repair_NodeType` | vehicleManagerNodeType.h | 修理 |
| `ToggleDoor_NodeType` | vehicleManagerNodeType.h | 切换车门 |
| `SpawnPlayerVehicle_NodeType` | vehicleManagerNodeType.h | 生成玩家车辆 |
| `Teleport_NodeType` | vehicleManagerNodeType.h | 传送 |
| `ForbiddenTrigger_NodeType` | vehicleManagerNodeType.h | 禁止触发器 |
| `EnableVehicleSummon_NodeType` | vehicleManagerNodeType.h | 启用车辆召唤 |
| `EnablePlayerVehicle_NodeType` | vehicleManagerNodeType.h | 启用玩家车辆 |
| `ToggleWindow_NodeType` | vehicleManagerNodeType.h | 切换车窗 |
| `UnassignAll_NodeType` | vehicleManagerNodeType.h | 取消所有分配 |
| `ForcePhysicsWakeUp_NodeType` | vehicleManagerNodeType.h | 强制物理唤醒 |
| `SetImmovable_NodeType` | vehicleManagerNodeType.h | 设置不可移动 |

---

### 2.16 音频管理器 (Audio Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `AudioNodeDefinition` | audioNode.h | 音频节点主节点 |
| `AudioCharacterManagerNodeDefinition` | audioCharacterManagerNode.h | 角色音频管理器 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `IAudioNodeType` | audioNode.h | 音频节点类型接口 |
| `AudioMixNodeType` | audioNode.h | 音频混合 |
| `AudioParameterNodeType` | audioParameterNode.h | 音频参数 |
| `AudioMusicSyncNodeType` | audioMusicSyncNode.h | 音乐同步 |
| `AudioSwitchNodeType` | audioSwitchNode.h | 音频开关 |
| `AudioFocusNodeType` | audioFocusNode.h | 音频焦点 |
| `AudioLoadingNodeType` | audioLoadingNode.h | 音频加载 |
| `IAudioCharacterManager_NodeType` | audioCharacterManagerNodeType.h | 角色音频管理器接口 |
| `AudioCharacterSystemsManager_NodeType` | audioCharacterSystemsManagerNodeType.h | 角色音频系统管理器 |
| `AudioCharacterManagerBreathing_NodeSubType` | audioCharacterManagerBreathingNodeSubType.h | 呼吸音频 |
| `AudioCharacterManagerFootsteps_NodeSubType` | audioCharacterManagerFootstepsNodeSubType.h | 脚步音频 |

---

### 2.17 行为管理器 (Behaviour Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `BehaviourManagerNodeDefinition` | behaviourManagerNode.h | 行为管理器主节点 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `IBehaviourManager_NodeType` | behaviourManagerNode.h | 行为管理器节点类型接口 |
| `JumpWorkspotAnim_NodeType` | behaviourManagerNode.h | 跳转工作点动画 |
| `StopWorkspot_NodeType` | behaviourManagerNode.h | 停止工作点 |

---

### 2.18 事实数据库管理器 (Facts DB Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `FactsDBManagerNodeDefinition` | factsDBManagerNode.h | 事实数据库管理器主节点 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `IFactsDBManagerNodeType` | factsDBManagerNodeType.h | 事实数据库管理器节点类型接口 |
| `SetVar_NodeType` | factsDBManagerNodeType.h | 设置变量 |

---

### 2.19 地图标记管理器 (Map Pin Manager)

| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `MapPinManagerNodeDefinition` | mapPinManagerNode.h | 地图标记管理器 |
| `MappinManagerNodeDefinition` | mapPinManagerNode.h | 地图标记管理器(新版) |

---

### 2.20 奖励管理器 (Reward Manager)

#### 主节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `RewardManagerNodeDefinition` | rewardManagerNode.h | 奖励管理器主节点 |

#### 节点类型
| 节点类型 | 所在文件 | 功能 |
|---------|---------|------|
| `IRewardManagerNodeType` | rewardManagerNodeType.h | 奖励管理器节点类型接口 |
| `GiveReward_NodeType` | rewardManagerNodeType.h | 给予奖励 |

---

### 2.21 生成管理器 (Spawn Manager)

| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `SpawnManagerNodeDefinition` | spawnManagerNode.h | 生成管理器主节点 |

---

### 2.22 时间管理器 (Time Manager)

| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `TimeManagerNodeDefinition` | timeManagerNode.h | 时间管理器主节点 |

---

### 2.23 视觉模式管理器 (Vision Modes Manager)

| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `VisionModesManagerNodeDefinition` | visionModesManagerNode.h | 视觉模式管理器主节点 |

---

### 2.24 语音集管理器 (Voiceset Manager)

| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `VoicesetManagerNodeDefinition` | voicesetManagerNode.h | 语音集管理器主节点 |

---

### 2.25 录制管理器 (Recording Manager)

| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `RecordingNodeDefinition` | recordingNode.h | 录制节点主节点 |

---

## 3. AI 命令节点 (AI Command Nodes)

### 基础 AI 命令节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `AICommandNodeBase` | aiCommandNodeBase.h | AI命令节点基类 |
| `ConfigurableAICommandNode` | questConfigurableAICommandNode.h | 可配置AI命令节点 |

### AI 命令节点
| 节点类名 | 所在文件 | 功能 |
|---------|---------|------|
| `SendAICommandNodeDefinition` | sendAICommandNode.h | 发送AI命令 |
| `CombatNodeDefinition` | combatNode.h | 战斗节点 |
| `MovePuppetNodeDefinition` | movePuppetNode.h | 移动Puppet |
| `MiscAICommandNode` | miscAICommandNode.h | 杂项AI命令 |
| `TeleportPuppetNodeDefinition` | teleportPuppetNode.h | 传送Puppet |
| `EquipItemNodeDefinition` | equipItemNode.h | 装备物品 |
| `UnequipItemNodeDefinition` | unequipItemNode.h | 卸下物品 |
| `UseWorkspotNodeDefinition` | useWorkspotNode.h | 使用工作点 |
| `RotateToNodeDefinition` | rotateToNode.h | 旋转到 |
| `VehicleNodeCommandDefinition` | vehicleCommandNode.h | 车辆命令 |
| `ForcedBehaviourNodeDefinition` | forcedBehaviourNode.h | 强制行为 |
| `ClearForcedBehavioursNodeDefinition` | forcedBehaviourNode.h | 清除强制行为 |
| `LookAtDrivenTurnsNode` | lookAtDrivenTurnsNode.h | 注视驱动转向 |

### AI 命令参数
| 参数类名 | 所在文件 | 用途 |
|---------|---------|------|
| `AICommandParams` | aiCommandNodeBase.h | AI命令参数基类 |
| `EquipItemParams` | equipItemNode.h | 装备物品参数 |
| `MoveOnSplineParams` | movePuppetNode.h | 在样条上移动参数 |
| `MoveToParams` | movePuppetNode.h | 移动到参数 |
| `PatrolParams` | movePuppetNode.h | 巡逻参数 |
| `FollowParams` | movePuppetNode.h | 跟随参数 |
| `MovePuppetNodeParams` | movePuppetNode.h | 移动Puppet参数 |
| `MiscAICommandNodeParams` | miscAICommandNode.h | 杂项AI命令参数 |
| `ScriptedAICommandParams` | miscAICommandNode.h | 脚本化AI命令参数 |
| `CombatNodeParams` | combatNode.h | 战斗节点参数 |
| `ConstAICommandParams` | sendAICommandNode.h | 常量AI命令参数 |
| `UseWorkspotCommandParams` | sendAICommandNode.h | 使用工作点命令参数 |
| `VehicleCommandParams` | vehicleCommandNode.h | 车辆命令参数 |

---

## 4. 逻辑节点 (Logical Nodes)

### 逻辑基础节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `LogicalBaseNodeDefinition` | logicalBaseNode.h | 逻辑节点基类 |

### 逻辑运算节点
| 节点类名 | 所在文件 | 功能 |
|---------|---------|------|
| `LogicalAndNodeDefinition` | logicalAndNode.h | 逻辑与节点 |
| `LogicalXorNodeDefinition` | logicalXorNode.h | 逻辑异或节点 |
| `LogicalHubNodeDefinition` | logicalHubNode.h | 逻辑Hub节点 |

---

## 5. 条件节点 (Condition Nodes)

### 条件基础类
| 类名 | 所在文件 | 描述 |
|------|---------|------|
| `IBaseCondition` | condition.h | 条件基类接口 |
| `Condition` | condition.h | 条件类 |
| `TypedCondition` | condition.h | 带类型的条件 |
| `LogicalCondition` | logicalCondition.h | 逻辑条件 |

### 条件节点
| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `ConditionNodeDefinition` | conditionNode.h | 条件节点 |
| `PauseConditionNodeDefinition` | pauseConditionNode.h | 暂停条件节点 |

### 对象条件 (Object Conditions)
| 条件类型 | 所在文件 | 功能 |
|---------|---------|------|
| `ObjectCondition` | objectCondition.h | 对象条件 |
| `IObjectConditionType` | objectConditionType.h | 对象条件类型接口 |
| `Interaction_ConditionType` | objectConditionType.h | 交互条件 |
| `Inventory_ConditionType` | objectConditionType.h | 库存条件 |
| `Inspect_ConditionType` | objectConditionType.h | 检查条件 |
| `Scan_ConditionType` | objectConditionType.h | 扫描条件 |
| `EntryScanned_ConditionType` | objectConditionType.h | 条目扫描条件 |
| `Device_ConditionType` | objectConditionType.h | 设备条件 |
| `Destruction_ConditionType` | objectConditionType.h | 破坏条件 |
| `Tagged_ConditionType` | objectConditionType.h | 标记条件 |

### 支付条件 (Payment Conditions)
| 条件类型 | 所在文件 | 功能 |
|---------|---------|------|
| `PaymentCondition` | paymentCondition.h | 支付条件 |
| `IPayment_ConditionType` | paymentConditionType.h | 支付条件类型接口 |
| `PaymentBalanced_ConditionType` | paymentConditionType.h | 平衡支付条件 |
| `PaymentFixedAmount_ConditionType` | paymentConditionType.h | 固定金额支付条件 |

### 属性条件 (Stats Conditions)
| 条件类型 | 所在文件 | 功能 |
|---------|---------|------|
| `StatsCondition` | statsCondition.h | 属性条件 |
| `IStatsConditionType` | statsConditionType.h | 属性条件类型接口 |
| `Stat_ConditionType` | statsConditionType.h | 属性条件 |
| `StreetCredTier_ConditionType` | statsConditionType.h | 街头声望等级条件 |
| `IStatsScriptConditionType` | statsConditionType.h | 属性脚本条件接口 |
| `LifePath_ConditionType` | statsConditionType.h | 人生轨迹条件 |
| `Build_ConditionType` | statsConditionType.h | 构建条件 |

### 系统条件 (System Conditions)
| 条件类型 | 所在文件 | 功能 |
|---------|---------|------|
| `ISystemConditionType` | systemConditionType.h | 系统条件类型接口 |
| `CameraFocus_ConditionType` | systemConditionType.h | 相机焦点条件 |
| `VisionMode_ConditionType` | systemConditionType.h | 视觉模式条件 |
| `Platform_ConditionType` | systemConditionType.h | 平台条件 |
| `InputAction_ConditionType` | systemConditionType.h | 输入动作条件 |
| `InputController_ConditionType` | systemConditionType.h | 输入控制器条件 |
| `Phone_ConditionType` | systemConditionType.h | 电话条件 |
| `PhonePickUp_ConditionType` | systemConditionType.h | 电话接听条件 |
| `Prereq_ConditionType` | systemConditionType.h | 前置条件 |
| `Weather_ConditionType` | systemConditionType.h | 天气条件 |
| `Radio_ConditionType` | systemConditionType.h | 电台条件 |
| `RadioTrack_ConditionType` | systemConditionType.h | 电台曲目条件 |
| `PlaylistTrackChanged_ConditionType` | systemConditionType.h | 播放列表曲目变更条件 |
| `Language_ConditionType` | systemConditionType.h | 语言条件 |
| `GOGReward_ConditionType` | systemConditionType.h | GOG奖励条件 |
| `SaveLock_ConditionType` | systemConditionType.h | 存档锁定条件 |

### 时间条件 (Time Conditions)
| 条件类型 | 所在文件 | 功能 |
|---------|---------|------|
| `TimeCondition` | timeCondition.h | 时间条件 |
| `ITimeConditionType` | timeConditionType.h | 时间条件类型接口 |
| `RealtimeDelay_ConditionType` | timeConditionType.h | 实时延迟条件 |
| `GameTimeDelay_ConditionType` | timeConditionType.h | 游戏时间延迟条件 |
| `TimePeriod_ConditionType` | timeConditionType.h | 时间段条件 |

---

## 6. 流程控制节点 (Flow Control Nodes)

| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `FlowControlNodeDefinition` | flowControlNode.h | 流程控制节点 |
| `SwitchNodeDefinition` | switchNode.h | 开关节点 |
| `RandomizerNodeDefinition` | randomizerNode.h | 随机器节点 |
| `CheckpointNodeDefinition` | checkpointNode.h | 检查点节点 |
| `EmbeddedGraphNodeDefinition` | embeddedGraphNode.h | 嵌入式图节点 |
| `PhaseNodeDefinition` | phaseNode.h | 阶段节点 |
| `DeletionMarkerNodeDefinition` | deletionMarkerNode.h | 删除标记节点 |

---

## 7. 多人游戏节点 (Multiplayer Nodes)

| 节点类名 | 所在文件 | 描述 |
|---------|---------|------|
| `MultiplayerAIDirectorNodeDefinition` | multiplayerAIDirectorNode.h | 多人游戏AI导演节点 |
| `MultiplayerChoiceTokenNodeDefinition` | multiplayerChoiceTokenNode.h | 多人游戏选择令牌节点 |
| `MultiplayerJunctionDialogNodeDefinition` | multiplayerJunctionDialogNode.h | 多人游戏交汇对话节点 |
| `MultiplayerTeleportPuppetNodeDefinition` | multiplayerTeleportPuppetNode.h | 多人游戏传送Puppet节点 |

---

## 8. 杂项节点 (Miscellaneous Nodes)

### 杂项功能节点
| 节点类名 | 所在文件 | 功能 |
|---------|---------|------|
| `BaseObjectNodeDefinition` | baseObjectNode.h | 基础对象节点 |
| `CutControlNodeDefinition` | cutControlNode.h | 剪辑控制节点 |
| `MinigameNodeDefinition` | minigameNode.h | 小游戏节点 |
| `PlaceholderNodeDefinition` | placeholderNode.h | 占位符节点 |
| `PuppeteerNodeDefinition` | puppeteer.h | 操纵者节点 |
| `PuppetAIManagerNodeDefinition` | puppetAIManager.h | Puppet AI管理器节点 |
| `PopulactionControllerNodeDefinition` | populationControllerNode.h | 人口控制器节点 |
| `InstancedCrowdControlNodeDefinition` | instancedCrowdControlNode.h | 实例化人群控制节点 |
| `TransformAnimatorNodeDefinition` | transformAnimatorNode.h | 变换动画器节点 |
| `TeleportVehicleNodeDefinition` | teleportVehicleNode.h | 传送车辆节点 |
| `WorkspotParamNodeDefinition` | workspotParamNode.h | 工作点参数节点 |

---

## 统计信息

### 按类别统计

| 类别 | 数量 | 主要文件类型 |
|------|------|-------------|
| **管理器节点** | 25+ | *ManagerNode.h, *ManagerNodeType.h |
| **AI命令节点** | 15+ | *CommandNode.h, aiCommandNodeBase.h |
| **条件节点** | 40+ | *Condition.h, *ConditionType.h |
| **逻辑节点** | 3 | logical*.h |
| **流程控制节点** | 7 | *ControlNode.h |
| **多人游戏节点** | 4 | multiplayer*.h |
| **核心节点** | 10+ | node.h, *NodeDefinition.h |
| **杂项节点** | 12+ | 各种 |

### 节点类型总览

- **总头文件数**: 192
- **主要节点定义数**: 150+
- **节点子类型数**: 200+
- **条件类型数**: 50+

---

## 节点继承层次结构

```
NodeDefinition (基类)
├── DisableableNodeDefinition
│   ├── AudioCharacterManagerNodeDefinition
│   ├── BaseObjectNodeDefinition
│   ├── ConditionNodeDefinition
│   ├── CutControlNodeDefinition
│   ├── EntityManagerNodeDefinition
│   ├── EnvironmentManagerNodeDefinition
│   ├── EventManagerNodeDefinition
│   ├── FactsDBManagerNodeDefinition
│   ├── FlowControlNodeDefinition
│   ├── FXManagerNodeDefinition
│   ├── InstancedCrowdControlNodeDefinition
│   ├── InteractiveObjectManagerNodeDefinition
│   ├── IONodeDefinition
│   │   ├── InputNodeDefinition
│   │   └── OutputNodeDefinition
│   ├── ItemManagerNodeDefinition
│   ├── JournalNodeDefinition
│   ├── MapPinManagerNodeDefinition
│   ├── MappinManagerNodeDefinition
│   ├── MultiplayerJunctionDialogNodeDefinition
│   ├── PlaceholderNodeDefinition
│   ├── PuppetAIManagerNodeDefinition
│   ├── PuppeteerNodeDefinition
│   ├── RandomizerNodeDefinition
│   ├── RecordingNodeDefinition
│   ├── RenderFxManagerNodeDefinition
│   ├── RewardManagerNodeDefinition
│   ├── StartEndNodeDefinition
│   │   ├── StartNodeDefinition
│   │   └── EndNodeDefinition
│   ├── SwitchNodeDefinition
│   ├── TeleportVehicleNodeDefinition
│   ├── TimeManagerNodeDefinition
│   ├── TriggerManagerNodeDefinition
│   ├── VisionModesManagerNodeDefinition
│   ├── VoicesetManagerNodeDefinition
│   └── WorkspotParamNodeDefinition
│
└── SignalStoppingNodeDefinition
    ├── AICommandNodeBase
    │   ├── ConfigurableAICommandNode
    │   │   ├── CombatNodeDefinition
    │   │   ├── MiscAICommandNode
    │   │   └── MovePuppetNodeDefinition
    │   ├── EquipItemNodeDefinition
    │   ├── SendAICommandNodeDefinition
    │   ├── TeleportPuppetNodeDefinition
    │   ├── UseWorkspotNodeDefinition
    │   └── VehicleNodeCommandDefinition
    │
    ├── AudioNodeDefinition
    ├── BehaviourManagerNodeDefinition
    ├── CharacterManagerNodeDefinition
    ├── CheckpointNodeDefinition
    ├── DeletionMarkerNodeDefinition
    ├── EmbeddedGraphNodeDefinition
    │   └── PhaseNodeDefinition
    ├── ForcedBehaviourNodeDefinition
    ├── ClearForcedBehavioursNodeDefinition
    ├── LookAtDrivenTurnsNode
    ├── LogicalBaseNodeDefinition
    │   ├── LogicalAndNodeDefinition
    │   ├── LogicalHubNodeDefinition
    │   └── LogicalXorNodeDefinition
    ├── MinigameNodeDefinition
    ├── MultiplayerAIDirectorNodeDefinition
    ├── MultiplayerChoiceTokenNodeDefinition
    ├── MultiplayerTeleportPuppetNodeDefinition
    ├── PauseConditionNodeDefinition
    ├── PhoneManagerNodeDefinition
    ├── RotateToNodeDefinition
    ├── SceneManagerNodeDefinition
    ├── SpawnManagerNodeDefinition
    ├── TransformAnimatorNodeDefinition
    ├── TypedSignalStoppingNodeDefinition
    │   └── GameManagerNodeDefinition
    ├── UIManagerNodeDefinition
    ├── UnequipItemNodeDefinition
    └── VehicleNodeDefinition
```

---

## 使用说明

### 如何查找特定功能的节点

1. **按功能分类**:
   - 角色相关 → CharacterManager
   - 物品相关 → ItemManager
   - UI相关 → UIManager
   - 环境相关 → EnvironmentManager
   - AI行为 → AICommand节点

2. **按节点类型**:
   - 管理器节点: *ManagerNodeDefinition
   - AI命令: *CommandNode, AICommandNodeBase
   - 条件判断: *Condition, *ConditionType
   - 逻辑运算: Logical*Node

3. **文件命名规则**:
   - 主节点定义: `*Node.h`
   - 节点类型: `*NodeType.h`
   - 节点类型实现: `*NodeTypeImpl.h`
   - 条件类型: `*ConditionType.h`

---

## 附录：文件列表

完整文件列表请参考源目录:
`D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev\src\common\gameQuest\include`

共计 **192** 个头文件，涵盖了Cyberpunk 2077所有Quest系统节点类型。

---

**文档结束**
