# SyncEntry编辑器使用指南

## 编辑器界面概览

基于RED Engine的Workspot编辑器，界面布局如下：

```
┌─────────────────────────────────────────────────────────────────┐
│ Workspot Editor                                                  │
├──────────┬──────────────────────────┬──────────────────────────┤
│          │                          │ Tools 标签页               │
│ 属性面板  │   AnimGraph 节点视图      │ ├─ Animations             │
│          │   ┌────────┐             │ ├─ AnimInputs              │
│ Global   │   │ Output │             │ ├─ Validator output        │
│ Tags     │   └────────┘             │ ├─ Debug objects           │
│ Block... │        ↓                 │ └─ Workspot tree ← 核心面板│
│          │   ┌─────────────┐        │                            │
│          │   │Workspot Hub │        │  Workspot Tree 视图：      │
│          │   └─────────────┘        │  ├─ 🎨 Root sequence       │
│          │                          │  │  ├─ 🟢 Entry anim       │
│ Preview  │   （图形化节点编辑）      │  │  ├─ 🟡 Sequence         │
│          │                          │  │  │  ├─ 🟢 Anim          │
│ 3D预览   │                          │  │  │  └─ 🔵 Pause         │
│          │                          │  │  └─ 🟢 Exit anim        │
└──────────┴──────────────────────────┴──────────────────────────┤
│ Properties 面板（选中节点的属性编辑区）                           │
│ ├─ Global Props                                                  │
│ ├─ Workspot properties                                           │
│ └─ 具体Entry的属性（m_animName, m_slotName, m_isSynchronized等）│
└──────────────────────────────────────────────────────────────────┘
```

---

## 一、打开Workspot资源

### 1.1 打开现有Workspot

**方式A：通过资源浏览器**
```
1. File → Open
2. 浏览到 Workspot 资源目录
   例如：workspots/master/master_generic__sit_chair__sit_around__01/workspot
3. 选择 .workspot 文件
4. 点击 Open
```

**方式B：通过项目浏览器**
```
在项目浏览器中找到 .workspot 文件 → 双击打开
```

### 1.2 创建新Workspot

```
1. File → New → Workspot Resource
2. 选择保存位置
3. 输入文件名（推荐命名规范：<角色>__<动作类型>__<动作名称>__<编号>）
4. 自动创建 Root sequence 节点
```

---

## 二、创建SyncMasterEntryAnim节点

### 2.1 添加节点到树中

**步骤1：选择父容器节点**

在右侧 **Workspot tree** 面板中：
```
1. 找到要添加子节点的容器（通常是 Sequence 或 RandomList）
2. 单击选中该容器节点（节点会高亮显示）
```

**步骤2：右键打开上下文菜单**

```
1. 在选中的容器节点上右键点击
2. 上下文菜单会显示所有可添加的节点类型
```

**上下文菜单选项**（基于源码 line 2293-2308）：
```
┌──────────────────────────────────┐
│ Add Animation                    │  ← AnimClip
│ Add Animation Item               │  ← AnimClipWithItem
│ Add Sequence                     │  ← Sequence容器
│ Add Reaction Sequence            │  ← ReactionSequence
│ Add Conditional Sequence         │  ← ConditionalSequence
│ Add Random List                  │  ← RandomList容器
│ Add Selector                     │  ← Selector
│ Add Fast Exit                    │  ← FastExit
│ Add Exit Anim                    │  ← ExitAnim
│ Add Entry Anim                   │  ← EntryAnim（普通）
│ Add Sync Master Entry Anim       │  ← ⭐ 这个！
│ Add Sync Anim                    │  ← SyncAnimClip
│ Add Motion Anim                  │  ← MotionAnimClip
│ Add Pause                        │  ← PauseClip
│ Add Look At Driven Turn          │  ← LookAtDrivenTurn
│ Add Tag Node                     │  ← TagNode
├──────────────────────────────────┤
│ Fill All Animations              │  ← 批量填充
│ Fill Stand Enter Exit Animations │
│ Fill Exit Animations             │
│ Fill Entry Animations            │
│ Fill Reaction Animations         │
└──────────────────────────────────┘
```

**步骤3：选择 "Add Sync Master Entry Anim"**

```
点击菜单中的 "Add Sync Master Entry Anim"
→ 节点会立即添加到树中
→ 新节点显示为：
   🟢 Entry synced master : <未设置> | <未设置>
```

### 2.2 节点在树中的显示

**SyncMasterEntryAnim 节点显示格式**（源码 line 1629）：
```
Entry synced master : <m_animName> | <m_slotName>

示例：
Entry synced master : dance_couple_01 | couple_dance_sync
```

**普通 EntryAnim（启用同步时）显示格式**（源码 line 1560）：
```
Entry anim: <m_animName> | <m_slotName>

示例：
Entry anim: dance_couple_01 | couple_dance_sync
```

**区别**：
- SyncMasterEntryAnim 显示 "Entry synced master"
- 普通 EntryAnim（m_isSynchronized=true）显示 "Entry anim"

---

## 三、配置SyncMasterEntryAnim属性

### 3.1 选中节点查看属性

```
1. 在 Workspot tree 面板中点击选中 SyncMasterEntryAnim 节点
2. 底部 Properties 面板会显示该节点的所有属性
3. 切换到 "Properties" 标签（如果不在该标签）
```

### 3.2 核心同步属性配置

**属性面板中的关键字段**：

#### 1. m_animName（动画名称）⭐⭐⭐

```
┌──────────────────────────────────────┐
│ m_animName                           │
│ ┌──────────────────────────────────┐│
│ │ dance_couple_01                  ││  ← 输入动画名称
│ └──────────────────────────────────┘│
└──────────────────────────────────────┘
```

**设置方法**：
- 直接输入动画名称字符串
- 或点击右侧 🔍 按钮浏览可用动画

**注意**：
- Master和Slave必须使用**相同**或**对应变体**的动画
- 动画时长必须一致

#### 2. m_slotName（同步槽名称）⭐⭐⭐⭐⭐

```
┌──────────────────────────────────────┐
│ m_slotName                           │
│ ┌──────────────────────────────────┐│
│ │ couple_dance_sync                ││  ← 同步槽名称
│ └──────────────────────────────────┘│
└──────────────────────────────────────┘
```

**关键作用**：
- Master和Slave通过**相同的slotName**建立同步关系
- 系统通过slotName匹配来查找对应的Entry

**命名建议**：
```
sync_<场景>_<动作类型>_<编号>

示例：
- sync_dance_couple_01      # 双人舞
- sync_fight_combo_punch    # 战斗连击
- sync_ritual_group_01      # 群体仪式
- sync_choir_sing_01        # 合唱
```

#### 3. m_isSynchronized（同步标志）⭐⭐⭐

```
┌──────────────────────────────────────┐
│ m_isSynchronized                     │
│ ☑ true                               │  ← SyncMasterEntryAnim强制为true
└──────────────────────────────────────┘
```

**说明**：
- SyncMasterEntryAnim 自动设置为 true，**无法修改**
- 这是强制同步标志

#### 4. m_syncOffset（同步偏移）⭐⭐⭐⭐

```
┌──────────────────────────────────────┐
│ m_syncOffset                         │
│ ┌──────────────────────────────────┐│
│ │ Position: (0.0, 0.0, 0.0)        ││  ← Master通常为(0,0,0)
│ │ Rotation: (0.0, 0.0, 0.0)        ││
│ └──────────────────────────────────┘│
└──────────────────────────────────────┘
```

**说明**：
- Master的offset通常为 `(0, 0, 0)`
- Slave的offset表示相对于Master的位置偏移

#### 5. m_idleAnim（底层Idle动画）

```
┌──────────────────────────────────────┐
│ m_idleAnim                           │
│ ┌──────────────────────────────────┐│
│ │ stand__2h_on_sides__01           ││  ← 底层循环动画
│ └──────────────────────────────────┘│
└──────────────────────────────────────┘
```

#### 6. m_blendOutTime（混出时间）

```
┌──────────────────────────────────────┐
│ m_blendOutTime                       │
│ ┌──────────────────────────────────┐│
│ │ 0.3                              ││  ← 秒
│ └──────────────────────────────────┘│
└──────────────────────────────────────┘
```

### 3.3 完整属性示例

**Master Workspot 配置**：
```
Entry synced master : dance_couple_01 | couple_dance_sync

Properties:
├─ m_animName        = "dance_couple_01"
├─ m_idleAnim        = "stand__2h_on_sides__01"
├─ m_slotName        = "couple_dance_sync"       ← 关键
├─ m_isSynchronized  = true                      ← 自动
├─ m_syncOffset      = Position(0, 0, 0)         ← Master为原点
├─ m_blendOutTime    = 0.3
└─ m_blendInTime     = 0.3
```

---

## 四、创建Slave的同步Entry

### 4.1 创建Slave Workspot

Slave不使用 `SyncMasterEntryAnim`，而是使用**普通的 EntryAnim** 并启用同步标志。

**步骤1：添加普通EntryAnim**

```
1. 打开 Slave 的 Workspot 资源
2. 在容器节点上右键
3. 选择 "Add Entry Anim"（不是 Add Sync Master Entry Anim）
4. 新节点显示为：
   🟢 Entry anim: <未设置>
```

**步骤2：配置Slave的属性**

选中新添加的 EntryAnim，在 Properties 面板配置：

```
Entry anim: dance_couple_01 | couple_dance_sync

Properties:
├─ m_animName        = "dance_couple_01"         ← 与Master相同
├─ m_idleAnim        = "stand__2h_on_sides__01"
├─ m_slotName        = "couple_dance_sync"       ← ⭐ 与Master相同！
├─ m_isSynchronized  = true                      ← ⭐ 手动启用
├─ m_syncOffset      = Position(1.5, 0, 0)       ← Slave的位置偏移
├─ m_blendOutTime    = 0.3
└─ m_blendInTime     = 0.3
```

**关键步骤**：
1. `m_slotName` 必须与 Master 的 **完全相同**
2. `m_isSynchronized` 必须手动勾选为 `true`
3. `m_syncOffset` 设置为相对Master的位置偏移

### 4.2 验证节点显示

配置完成后，节点显示应该变为：
```
Entry anim: dance_couple_01 | couple_dance_sync
```

如果 `m_isSynchronized = false`，则只显示：
```
Entry anim: dance_couple_01
```
（没有 `| <slotName>` 部分）

---

## 五、实战示例：创建双人舞同步

### 场景：两个NPC跳双人舞

#### 5.1 创建Master Workspot

**文件名**：`dance_master__sit_chair__couple_dance__01.workspot`

**步骤1：创建树结构**

```
Root sequence
└─ Sequence: main_sequence
   ├─ Entry anim: walk_to_dance_position
   ├─ Entry synced master: dance_couple_01 | couple_dance_sync  ← 同步点
   └─ Exit anim: walk_0_to_stand_2h_on_sides_01_to_walk_0_turn0_01
```

**操作**：
1. 右键 Root sequence → Add Sequence
2. 右键新建的 Sequence → Add Entry Anim（走到位置）
3. 右键 Sequence → Add Sync Master Entry Anim（同步舞蹈）
4. 右键 Sequence → Add Exit Anim（离开）

**步骤2：配置SyncMasterEntryAnim**

选中 "Entry synced master" 节点，配置：
```
m_animName       = "dance_couple_01"
m_slotName       = "couple_dance_sync"
m_isSynchronized = true（自动）
m_syncOffset     = Position(0, 0, 0)
```

#### 5.2 创建Slave Workspot

**文件名**：`dance_slave__sit_chair__couple_dance__01.workspot`

**步骤1：创建树结构**

```
Root sequence
└─ Sequence: main_sequence
   ├─ Entry anim: walk_to_dance_position
   ├─ Entry anim: dance_couple_01 | couple_dance_sync  ← 同步点
   └─ Exit anim: walk_0_to_stand_2h_on_sides_01_to_walk_0_turn0_01
```

**操作**：
1. 右键 Root sequence → Add Sequence
2. 右键 Sequence → Add Entry Anim（走到位置）
3. 右键 Sequence → Add Entry Anim（同步舞蹈，注意是普通Entry）
4. 右键 Sequence → Add Exit Anim（离开）

**步骤2：配置同步EntryAnim**

选中第二个 "Entry anim" 节点，配置：
```
m_animName       = "dance_couple_01"          ← 与Master相同
m_slotName       = "couple_dance_sync"        ← ⭐ 与Master相同
m_isSynchronized = true                       ← ⭐ 手动勾选
m_syncOffset     = Position(1.5, 0, 0)        ← 相对Master偏移1.5米
```

#### 5.3 验证配置

**Master节点显示**：
```
🟢 Entry synced master : dance_couple_01 | couple_dance_sync
```

**Slave节点显示**：
```
🟢 Entry anim: dance_couple_01 | couple_dance_sync
```

如果显示中**有 `| couple_dance_sync`** 说明同步配置正确！

#### 5.4 保存资源

```
1. Ctrl+S 或 File → Save
2. 等待动画集生成（如果启用了 Auto Animset Generation）
3. 查看底部 Message Log 确认没有错误
```

---

## 六、多人同步配置

### 场景：3人合唱

#### Master Workspot
```
Entry synced master : sing_together | choir_sync_01
├─ m_slotName    = "choir_sync_01"
└─ m_syncOffset  = Position(0, 0, 0)
```

#### Slave1 Workspot
```
Entry anim: sing_together | choir_sync_01
├─ m_slotName        = "choir_sync_01"      ← 相同
├─ m_isSynchronized  = true
└─ m_syncOffset      = Position(2.0, 0, 0)  ← 位置1
```

#### Slave2 Workspot
```
Entry anim: sing_together | choir_sync_01
├─ m_slotName        = "choir_sync_01"      ← 相同
├─ m_isSynchronized  = true
└─ m_syncOffset      = Position(1.0, 1.7, 0) ← 位置2
```

**关键**：所有3个Workspot都使用相同的 `m_slotName = "choir_sync_01"`

---

## 七、属性面板快速参考

### EntryAnim vs SyncMasterEntryAnim 对比

| 属性 | EntryAnim | SyncMasterEntryAnim |
|-----|----------|-------------------|
| 节点类型 | 普通Entry | 显式Master |
| m_isSynchronized | 手动设置 | ✅ 强制true |
| AllowSync() | 可作Master或Slave | ✅ 只能作Master |
| 树中显示 | Entry anim: | Entry synced master : |
| 右键菜单 | Add Entry Anim | Add Sync Master Entry Anim |
| 推荐用途 | Slave，或灵活Entry | Master专用 |

### 必填属性检查清单

**Master配置**：
- [ ] m_animName 已填写
- [ ] m_slotName 已填写（唯一且有意义）
- [ ] m_isSynchronized = true（自动）
- [ ] m_syncOffset = (0, 0, 0)（通常）

**Slave配置**：
- [ ] m_animName 已填写（与Master相同）
- [ ] m_slotName 已填写（与Master完全相同）
- [ ] m_isSynchronized = true（手动勾选）
- [ ] m_syncOffset 已设置（相对Master的偏移）

---

## 八、预览和测试

### 8.1 编辑器内预览

**启动预览**：
```
1. 点击工具栏的 ▶ 播放按钮
2. 或按 F5
3. 左侧 Preview 面板会显示3D预览
```

**注意**：
- 编辑器内预览只能看到单个Workspot
- 无法预览Master-Slave同步效果
- 需要在游戏中测试实际同步

### 8.2 游戏内测试

**测试流程**：
1. 保存Workspot资源
2. 编译游戏（如果需要）
3. 启动游戏
4. 使用脚本或场景触发Master和Slave的Workspot
5. 观察同步效果

**调试工具**：
- ImGui面板：查看实时Workspot状态
- 3D可视化：查看蓝色同步标记
- AI日志：查看同步命令发送记录

---

## 九、常见问题与技巧

### 问题1：节点显示名称不包含 `| <slotName>`

**原因**：`m_isSynchronized = false`

**解决**：
1. 选中节点
2. 在 Properties 面板中找到 `m_isSynchronized`
3. 勾选为 `true`
4. 节点显示会立即更新

### 问题2：无法添加SyncMasterEntryAnim

**原因**：选中的不是容器节点

**解决**：
1. 只能在容器节点（Sequence, RandomList, Selector等）中添加Entry
2. 确保选中的是容器节点，而不是叶子节点（AnimClip, Pause等）

### 问题3：属性修改后没有保存

**解决**：
1. 修改属性后按 Enter 确认
2. Ctrl+S 保存资源
3. 查看编辑器标题栏是否有 "*" 标记（未保存）

### 问题4：slotName拼写错误

**后果**：Master和Slave无法匹配，同步失败

**预防**：
1. 使用统一的命名规范
2. 复制粘贴slotName，避免手动输入
3. 使用编辑器的搜索功能验证名称一致性

### 技巧1：批量创建Entry

**使用自动填充功能**：
```
右键容器节点 → 选择：
- Fill All Animations           # 填充所有类型
- Fill Entry Animations         # 只填充Entry
- Fill Exit Animations          # 只填充Exit
- Fill Stand Enter Exit Animations  # 填充站立动作
```

**注意**：自动填充的Entry需要手动配置同步属性

### 技巧2：快速复制节点

```
1. 选中已配置好的节点
2. Ctrl+C 复制
3. 选中目标容器
4. Ctrl+V 粘贴
5. 修改具体的动画名称和偏移量
```

### 技巧3：使用搜索功能

在 Workspot tree 面板顶部有搜索框：
```
🔍 Search...

输入关键字快速定位节点：
- 搜索 "sync" → 找到所有同步节点
- 搜索 "couple_dance_sync" → 找到特定slotName的节点
```

---

## 十、高级编辑器功能

### 10.1 AnimGraph 节点视图

中间的图形化节点视图显示了Workspot的输出连接：

```
┌────────┐
│ Output │  ← 最终输出节点
└────┬───┘
     ↓
┌─────────────┐
│Workspot Hub │  ← Workspot系统的接入点
└─────────────┘
```

**说明**：
- Output：AnimGraph的输出，连接到角色的最终姿态
- Workspot Hub：Workspot系统注入动画的节点
- 通常不需要修改这些连接

### 10.2 全局属性配置

在底部 Properties 面板切换到 "Global" 标签：

**Workspot Rig**：
```
指定该Workspot支持的骨骼Rig
例如：base\characters\entities\citizen\_male_average.ent
```

**Global Props**：
```
全局属性数组，可以添加自定义属性
```

**Entities Paths**：
```
实体路径列表，用于动画集生成
```

**Animsets**：
```
自动生成的最终动画集
```

### 10.3 验证器输出

切换到 "Validator output" 标签：
```
保存时自动运行验证器，检查：
- 动画是否存在
- 动画时长是否匹配
- slotName是否唯一
- Entry路径是否有效
```

**查看错误**：
```
🔴 Error: Animation "xxx" not found in animset
⚠️ Warning: Entry duration exceeds time limit
ℹ️ Info: Animation loaded successfully
```

---

## 十一、工作流程总结

### 标准工作流程

```
1. 创建Master Workspot
   ├─ File → New → Workspot Resource
   ├─ 添加 Sequence 容器
   ├─ 右键 → Add Sync Master Entry Anim
   ├─ 配置 m_animName, m_slotName
   └─ Ctrl+S 保存

2. 创建Slave Workspot
   ├─ File → New → Workspot Resource
   ├─ 添加 Sequence 容器
   ├─ 右键 → Add Entry Anim（普通）
   ├─ 配置 m_animName（与Master相同）
   ├─ 配置 m_slotName（与Master相同）
   ├─ 勾选 m_isSynchronized = true
   ├─ 配置 m_syncOffset（位置偏移）
   └─ Ctrl+S 保存

3. 验证配置
   ├─ 检查节点显示名称包含 | <slotName>
   ├─ 检查 Validator output 无错误
   └─ 检查 slotName 拼写一致

4. 游戏内测试
   ├─ 启动游戏
   ├─ 触发Master和Slave的Workspot
   ├─ 使用ImGui面板或AI日志调试
   └─ 验证同步效果
```

### 快捷键参考

| 快捷键 | 功能 |
|-------|------|
| Ctrl+S | 保存资源 |
| Ctrl+C | 复制选中节点 |
| Ctrl+V | 粘贴节点 |
| Delete | 删除选中节点 |
| F5 | 启动预览 |
| Ctrl+F | 搜索节点 |

---

## 十二、总结

### 核心要点

1. **Master使用SyncMasterEntryAnim**：
   - 右键菜单 → Add Sync Master Entry Anim
   - 自动设置 m_isSynchronized = true

2. **Slave使用普通EntryAnim**：
   - 右键菜单 → Add Entry Anim
   - 手动勾选 m_isSynchronized = true

3. **slotName是同步的关键**：
   - Master和Slave必须使用完全相同的slotName
   - 推荐命名：`sync_<场景>_<动作>_<编号>`

4. **syncOffset定义位置关系**：
   - Master通常为 (0, 0, 0)
   - Slave设置相对偏移

5. **节点显示验证同步配置**：
   - 正确配置会显示：`<节点类型> : <animName> | <slotName>`
   - 缺少 `| <slotName>` 说明同步未启用

### 推荐实践

- ✅ 使用统一的slotName命名规范
- ✅ 复制粘贴slotName，避免拼写错误
- ✅ 保存后检查 Validator output
- ✅ 在游戏中测试实际同步效果
- ✅ 使用ImGui面板实时调试

现在您可以在RED Engine的Workspot编辑器中创建完美同步的多NPC动作了！
