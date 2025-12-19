
  "From A to B" 过渡动画命名规律

  1. 基本结构

  [起始姿势]__to__[结束姿势]__[转向]__[变体]

  2. 具体示例分析

  示例1：坐到站
  corpo__sit_chair_table_keyboard__2h_on_table__01
    __to__
  stand_table_keyboard__2h_front__01
    __turn0__01
  - 起始：企业角色坐在椅子上，双手放在桌上键盘处
  - 结束：站立在桌前键盘处，双手在前方
  - 转向：turn0（无转向，0度）
  - 变体：01（第一个变体）

  示例2：站到坐
  stand__2h_on_sides__01
    __to__
  corpo__sit_chair_table_keyboard__2h_table__01
    __turn90__01
  - 起始：站立，双手放在身体两侧
  - 结束：企业角色坐姿，双手在桌上
  - 转向：turn90（转向90度）
  - 变体：01

  3. 命名规律总结

  必须成对出现：
  - 从A到B的过渡动画通常会配套一个从B到A的反向过渡
  - 例如：坐→站 和 站→坐

  转向角度标记：
  - turn0 - 无转向
  - turn90 - 转向90度
  - turn180 - 转向180度（如需要）

  双向对称性：
  A__to__B__turn0__01  ←→  B__to__A__turn0__01

  组织方式：
  - 这些过渡动画通常集中在专门的 transitions.anims 文件中
  - 而不是分散在单独的文件中

  4. 设计意图

  这种命名方式的目的是：
  1. 状态机设计 - 支持游戏中角色在不同姿势间平滑切换
  2. 动画融合 - 确保从任意姿势A能够自然过渡到姿势B
  3. 可读性 - 清晰表达动画的起点和终点
  4. 双向性 - 明确每个过渡都有对应的反向动画

  这是一个非常系统化的动画管理方式，确保NPC在不同动作状态间切换时不会出现突兀的跳变。

   需要 TO 过渡动画的场景

  1. 循环姿势（Idle/Loop Poses）之间的切换

  坐姿 ↔ 站姿
  sit_chair_xxx → stand_xxx
  stand_xxx → sit_chair_xxx
  - 最常见的过渡类型
  - 涵盖不同类型的椅子和站姿

  不同坐姿之间
  sit_chair_table_keyboard__2h_on_table
    → sit_chair_table_keyboard__2h_table

  不同站姿之间
  stand__2h_on_sides (双手垂放)
    → stand__lh_tablet (左手拿平板)
    → stand__2h_front (双手在前)

  2. 工作/交互姿势的切换

  使用道具的姿势
  stand (空手) → stand__lh_tablet (拿平板)
  stand (空手) → stand_table_keyboard (使用键盘)

  不同工作姿势
  sit_xxx → work_sit_xxx
  idle_xxx → work_xxx

  3. 转向变化的同一姿势

  pose_xxx__turn0 → pose_xxx__turn90
  pose_xxx__turn90 → pose_xxx__turn180
  - 同一个基础姿势但朝向不同

  4. NOT需要 TO 的动画类型

  以下类型通常不需要 __to__ 过渡：

  ❌ 单次执行动作（One-shot Actions）
  - 攻击动画（attack）
  - 受击动画（hit_reaction）
  - 死亡动画（death）
  - 拾取物品（pickup）
  - 开门（door_open）

  ❌ 移动动画（Locomotion）
  - 行走（walk）
  - 奔跑（run）
  - 跳跃（jump）
  - 翻滚（dodge）

  ❌ 过场动画（Cinematic）
  - 剧情场景专用动画
  - 通常是预设序列

  ❌ 战斗动画
  - 连招动画自带衔接
  - 通过动画状态机处理

  规律总结

  需要 TO 过渡的共同特征：

  1. 持续性姿势 - 角色会在该姿势下停留一段时间
  2. 可互换性 - 角色可能随时从A姿势切换到B姿势
  3. 循环播放 - 这些姿势本身是循环动画
  4. 静态等待 - 通常是NPC的idle状态或工作状态

  文件组织方式：
  - 集中存放在 xxx_transitions.anims 文件中
  - 例如：corpo_female_transitions.anims
  - 一个transitions文件包含该角色类型的所有姿势转换

  命名对称性：
  A__to__B__turnX__01  ←必然对应→  B__to__A__turnX__01

  这种设计确保了NPC在开放世界中可以自然地在各种日常姿势间切换，避免动作突变。