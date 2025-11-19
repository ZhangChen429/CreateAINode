

#  2077对话编辑器工具 

## Timeline

## SceneNodePreview 场景方案预览功能
 
### Preview Scenario
1. SceneNodePreview
![alt text](image-5.png)
2. 切换使用节点路径
![alt text](image-6.png)
3. Pause 节点的跳过
![alt text](image-8.png)
4. 更长更多信息的Timeline
![alt text](image-7.png)
5. 播放创建Preview Scenario 过程
5. 分析
 Preview Scenario TimeLine 是根据节点的选择动态创建，根据设计师的随时修正生成整条Timeline结果，Preview Scenario TimeLine 是Scene编辑器的分阶段分支线的直观展示
 目的在于帮助设计师在编辑时，快速了解当前场景的动画结构、事件阶段、动画时间轴、玩家选择产生的结果，对于多支线非线性叙述中具有非常直观的帮助。
6. 创建了可按步骤预览的Timeline直观给设计师观察目前的剧情状态提供清晰的事件脉络

## 快速增长的数据库进行状态的控制
![alt text](image-14.png)

## Animation Tree Structure
![alt text](image-15.png)


多标签数据匹配
骨骼动画数据库
1. 命名和标签规则:
![alt text](image-20.png)
Generic_Arverage_Female_sit_Chair(bar)
stand__lh_tablet__01__slow_hours__01
dirt__kneel__2h_elbow_on_knees__01__look_right__01 Dirt(帮派|NPC种类)_(姿态)_(姿态)_(姿态)_(姿态)_(姿态)
lie_ground_0__2h_on_head__01__shuffle__04......
2. 设计师使用时的查找规则
![alt text](image-18.png)


### Animal 的加权裁剪混合
![alt text](image-13.png)

### 所有可添加的AnimationTrack
- 在动画中控制骨骼给设计师带来极大的自由度
![alt text](image-16.png)


##### AnimationTrack
![alt text](image-9.png)

| AnimationTrack                                                           | 描述                             |
| :----------------------------------------------------------------------- | :------------------------------- |
| “播放动画”   Create [actor]  "Play Animation"                            | 普通动画轨道                     |
| “播放消除动画”  Create [actor] "Play Rid Animation"                      |                                  |
| “切换Idle” Create [actor] "Change Idle"                                  | 切换Idle状态                     |
| “添加待机”  Create [actor] "Add Idle"                                    | 添加Idle                         |
| “带混合添加待机”  Create [actor] "Add Idle With Blend"                   | 添加Idle的混合动画               |
| “看向”   Create [actor] "Look At"                                        | 看向姿态                         |
| “额外看向”     Create [actor] "Additional Look At'                       |                                  |
| “姿态修正”  Create [actor] "Pose Correction"                             | 姿态修正手势修正                 |
| “（IK）”  Create [actor] "IK"                                            | 骨骼IK混合                       |
| “更改位置”  Create [actor] "Change Placement"                            |                                  |
| “设置动画特征”  Create [actor] "Set Anim Feature"                        |                                  |
| “更改工作” Create [actor] "Change work"                                  | 更改工作点内容                   |
| “停止工作”   Create [actor] "Stop work"                                  | 停止工作点                       |
| “附加道具”   Create [prop] "Attach Prop"                                 | 附加道具                         |
| “镜头片段”     Create [camera] "Clip"                                    |                                  |
| “播放骑乘动画”    Create [camera] "Play Rid Animation"                   |                                  |
| “播放音频”        Create [other] "Play Audio"                            | 声音轨道                         |
| “播放音频（带时长）” Create [other] "Play AudioDuration"                 | 声音可变Section                  |
| “播放视觉特效（VFX）”  Create [other] "Play VFX"                         | 特效                             |
| “播放视觉特效（带时长）” Create [other] "Play VFX Duration"              | Section特效                      |
| “播放视觉特效（超梦同步）”  Create [other] "Play VFX Braindance"         | 超梦特效                         |
| “提示”     Create [other] "Clue"                                         |                                  |
| “播放视频”    Create [other] "Play Video"                                |                                  |
| “插槽事件”      Create [other] "Socket Event"                            | 添添加eventTrack（中途触发事件） |
| “玩家游戏内看向”   Create [other] "Player Gameplay LookAt"               |                                  |
| “脑波可见性”      Create [other] "Braindance Visibility"                 |                                  |
| “播放 UI 动画”      Create [other] "Play UI Animation"                   |                                  |
| “播放 UI 动画（脑波同步）” Create [other] "Play UI Animation Braindance" |                                  |

![alt text](image-11.png)


![alt text](image-10.png)

## 对话设计师需要可以简单的操控场景的所有物体
![alt text](image-17.png)
给对话设计师最大的权限

## 本地化语言膨胀时间线

![alt text](image-12.png)

### 按比例膨胀时间

## 灯光根据场景基本配置调节



## Light Const Value 


