```
// --- JSON 结构定义 ---
// 请严格按照以下 Key-Value 格式输出剧情序列。
AnimWork 是游戏开发管线中的整体的动画库其中有规范的命名
```
master_generic__stand_ground__stand_around__01.workspot
│       │          │             │           │
│       │          │             │           └── 变体编号
│       │          │             └── 行为类型：站立闲逛
│       │          └── 姿态类别：地面站立
│       └── 通用性：可被多种 NPC 使用
└── 资产级别：master（主模板）
```
BackGround 是根据角色的剧情背景填写的信息，NPC的行为上下文
Code_content NPC要给出的关键信息，NPC要给出的关键
Code_Choice_content NPC要给出的选择

[
  {
    "AnimWork": Stand_Phone_Idle,Stand_LookAt_Idle,Stand_Lean_Idle,Sit_Phone_Idle
    "speaker_id": "NPC_ID_Or_Name", // (String) 说话人的ID，例如 "Boss_01" 或 "BackGround": 幼时因为饥饿流浪到寺庙的逐渐成长的小沙弥
    "Code_content" : 山里有个大师富有哲理
    "Code_Choice_content" : 你要和沙弥一起去见还是自己一会去见
    "idle_anim_tag": "Anim.State.Tag",  // (String) 对应动画库的Tag，例如 "Anim."Idle.Angry"
    "target_mark_id": "Mark_Point_ID",  // (String) 对应场景点位ID，例如 "Mark_Bar"
    // 4. 分支选项 (仅在 is_player_decision 为 true 时填写，否则留空数组)
    "Code_Choice_content": [
      {
        "choice_text": "选项按钮上显示的文字", 

      }
      // ... 可以有多个选项
    ]
  }
  // ... 下一个节点
]

```
----

````
[
  {
    "Speaker": "角色名 (String)",
    "Dialogue": "台词内容 (String)",
    "AnimTag": "从动画库中选一个 (String)",
    "MarkPointID": "从点位库中选一个 (String/Null)",
    "IsBranchingNode": false, 
    "Choices": [] // 如果 IsBranchingNode 为 false，此项为空
  },
  {
    "Speaker": "System",
    "Dialogue": "（如果是抉择点，这里写抉择提示，例如：你要怎么回答？）",
    "AnimTag": "Anim.Idle.Normal",
    "MarkPointID": null,
    "IsBranchingNode": true,
    "Choices": [
      {
        "Text": "选项文本",
        "ConsequenceHint": "简短后果预测(给策划看)"
      }
    ]
  }
]
```