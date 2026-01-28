# m_slotName 来源与命名指南

## 一、m_slotName的本质

### 1.1 定义

```cpp
// workspotTreeItems.h
class EntryAnim : public IEntry
{
    CName m_slotName;  // 同步槽名称
};
```

**关键点**：
- **类型**：`CName` （RED Engine的字符串名称类型）
- **默认值**：`CName::NONE()` （空名称）
- **性质**：**自由命名的字符串标识符**
- **作用**：用于Master和Slave之间的同步匹配

---

## 二、m_slotName的来源

### 2.1 ✅ 由美术/设计师手动命名

**m_slotName 不是从系统中自动生成的，而是由内容创建者（美术、设计师）在编辑器中手动输入的字符串。**

```
┌─────────────────────────────────────────────┐
│ Properties 面板                              │
├─────────────────────────────────────────────┤
│ m_slotName                                  │
│ ┌─────────────────────────────────────────┐│
│ │ couple_dance_sync  ← 手动输入的字符串  ││
│ └─────────────────────────────────────────┘│
└─────────────────────────────────────────────┘
```

### 2.2 ❌ 不来自预定义列表

- **没有**枚举值列表
- **没有**从配置文件读取
- **没有**从动画系统自动生成
- **完全自由命名**，只要Master和Slave使用相同的名称即可

### 2.3 类比理解

可以类比为：
- **网络通信的频道名称**：Master广播"couple_dance_sync"频道，Slave监听相同频道
- **事件总线的事件名**：Master发送"couple_dance_sync"事件，Slave订阅此事件
- **数据库的外键**：slotName就是连接Master和Slave的"外键"

---

## 三、m_slotName的工作原理

### 3.1 查找匹配流程

当Master触发同步时，系统调用 `GetSyncEntryIdForSlotName()` 查找Slave中对应的Entry：

```cpp
// workspotResource.cpp: 1079-1108
work::WorkEntryId WorkspotTree::GetSyncEntryIdForSlotName(
    CName slotName,      // 要查找的槽名称
    Bool asMaster,       // 是否作为Master
    IContainerEntry* cont
) const
{
    // 递归遍历WorkspotTree中的所有Entry
    for( THandle<IEntry>& record : cont->m_list )
    {
        // 检查EntryAnim
        if( EntryAnim* entry = helper::IsEntry< EntryAnim, CName, &EntryAnim::m_slotName >(
            cont, record.Get(), IEntry::SlowEnter | IEntry::Synchronized, slotName ) )
        {
            if(entry->AllowSync(asMaster))  // 验证是否允许作为Master/Slave
                return entry->m_id;         // 找到！返回Entry ID
        }

        // 检查ExitAnim
        else if( ExitAnim* entry = helper::IsEntry< ExitAnim, CName, &ExitAnim::m_slotName >(
            cont, record.Get(), IEntry::SlowExit | IEntry::Synchronized, slotName ) )
        {
            return entry->m_id;
        }

        // 检查SyncAnimClip
        else if( SyncAnimClip* entry = helper::IsEntry< SyncAnimClip, CName, &SyncAnimClip::m_slotName >(
            cont, record.Get(), IEntry::Synchronized, slotName ) )
        {
            return entry->m_id;
        }

        // 递归查找子容器
        if( IContainerEntry* container = Cast<IContainerEntry>( record.Get() ) )
        {
            WorkEntryId id = GetSyncEntryIdForSlotName( slotName, asMaster, container );
            if( id )
                return id;
        }
    }

    return WorkEntryId::invalid;  // 未找到
}
```

### 3.2 匹配条件

一个Entry能被匹配，需要满足：
1. ✅ `m_slotName` 与查找的名称**完全相同**
2. ✅ `m_isSynchronized = true` （启用同步标志）
3. ✅ `AllowSync(asMaster)` 返回true
   - SyncMasterEntryAnim：只允许 `asMaster = true`
   - 普通EntryAnim：允许任意角色

### 3.3 实际运行示例

**场景**：Master触发同步

```cpp
// Master NPC进入SyncMasterEntryAnim
Master: m_slotName = "couple_dance_sync"

// 系统调用同步
WorkspotSynchronizer::PlaySyncAnim(masterNPC, timeToSync, "couple_dance_sync");

// 遍历所有Slave
for( ent::EntityID& slaveNPC : entry.m_masterOf )
{
    // 在Slave的WorkspotTree中查找
    WorkEntryId entryId = GetSyncEntryIdForSlotName(
        slaveNPC,
        false,  // 作为Slave
        "couple_dance_sync"  ← 关键：查找相同的slotName
    );

    if( entryId )  // 找到了！
    {
        // 发送跳转命令
        SendCommandImmediate( slaveNPC, CMD_JumpToEntry, entryId );
    }
}
```

**结果**：
- Slave的WorkspotTree中有 `m_slotName = "couple_dance_sync"` 的Entry → ✅ 找到，同步成功
- Slave的WorkspotTree中没有此Entry → ✗ 未找到，同步失败

---

## 四、命名规范与最佳实践

### 4.1 推荐的命名格式

```
sync_<场景/用途>_<动作类型>_<编号>
```

**示例**：

| slotName | 说明 | 场景 |
|----------|------|------|
| `sync_dance_couple_01` | 双人舞第1套 | 酒吧舞池 |
| `sync_dance_couple_02` | 双人舞第2套 | 酒吧舞池（变体） |
| `sync_fight_combo_punch` | 战斗连击-拳击 | 战斗场景 |
| `sync_fight_combo_kick` | 战斗连击-踢腿 | 战斗场景 |
| `sync_ritual_circle_01` | 群体仪式-圆圈 | 仪式场地 |
| `sync_choir_sing_01` | 合唱第1首 | 教堂/舞台 |
| `sync_toast_drinking_01` | 碰杯喝酒 | 餐厅/酒吧 |
| `sync_handshake_01` | 握手 | 社交场景 |
| `sync_high_five_01` | 击掌 | 庆祝场景 |

### 4.2 命名建议

#### ✅ 好的命名

```
sync_dance_couple_01       # 清晰、描述性强
sync_fight_takedown_01     # 易于理解用途
sync_ritual_group_circle   # 表达了场景和形式
```

**优点**：
- 见名知意
- 便于搜索和管理
- 避免命名冲突
- 团队协作友好

#### ❌ 不好的命名

```
slot1                      # 太泛化，无法理解用途
anim_sync                  # 过于模糊
test_slot                  # 临时名称可能遗留到正式版本
couple_dance              # 缺少前缀，可能与其他系统命名冲突
```

**缺点**：
- 无法快速理解用途
- 容易产生命名冲突
- 难以维护和查找

### 4.3 命名分类建议

**按场景分类**：

```
社交互动：
- sync_handshake_formal_01      # 正式握手
- sync_handshake_casual_01      # 随意握手
- sync_hug_greeting_01          # 拥抱问候
- sync_high_five_01             # 击掌

战斗动作：
- sync_fight_punch_combo_01     # 拳击连击
- sync_fight_kick_combo_01      # 踢腿连击
- sync_fight_takedown_01        # 摔倒
- sync_fight_block_counter_01   # 格挡反击

舞蹈表演：
- sync_dance_couple_waltz_01    # 华尔兹
- sync_dance_couple_tango_01    # 探戈
- sync_dance_group_line_01      # 排舞
- sync_dance_group_circle_01    # 圆圈舞

仪式活动：
- sync_ritual_pray_group_01     # 群体祈祷
- sync_ritual_toast_01          # 敬酒
- sync_ritual_bow_01            # 鞠躬

娱乐活动：
- sync_choir_sing_01            # 合唱
- sync_band_play_guitar_01      # 乐队-吉他
- sync_band_play_drum_01        # 乐队-鼓手
```

### 4.4 项目级命名规范文档

建议创建项目级的命名规范文档：

```markdown
# Workspot SlotName 命名规范

## 格式
sync_<category>_<action>_<variant>

## 分类代码
- dance:  舞蹈
- fight:  战斗
- social: 社交
- ritual: 仪式
- work:   工作
- sport:  运动

## 示例
sync_dance_couple_waltz_01
sync_fight_combo_punch_heavy
sync_social_handshake_formal

## 禁止使用的名称
- test*
- temp*
- slot*
- anim*
- 单字母或数字
```

---

## 五、在编辑器中设置slotName

### 5.1 Master设置

```
1. 选中 SyncMasterEntryAnim 节点
2. 在 Properties 面板找到 m_slotName
3. 输入命名（如："sync_dance_couple_01"）
4. 按 Enter 确认
5. 节点显示更新为：
   Entry synced master : dance_couple_01 | sync_dance_couple_01
                                            ↑ slotName显示在这里
```

### 5.2 Slave设置

```
1. 选中 EntryAnim 节点
2. 在 Properties 面板找到 m_slotName
3. 输入**与Master完全相同**的名称
4. 勾选 m_isSynchronized = true
5. 节点显示更新为：
   Entry anim: dance_couple_01 | sync_dance_couple_01
                                  ↑ slotName显示在这里
```

### 5.3 快速复制技巧

避免手动输入导致拼写错误：

**方法1：属性复制**
```
1. 在Master的Properties面板中，选中m_slotName的值
2. Ctrl+C 复制
3. 切换到Slave的Workspot
4. 在m_slotName字段中 Ctrl+V 粘贴
```

**方法2：节点复制（如果结构相似）**
```
1. 复制整个Master的Entry节点
2. 粘贴到Slave的Workspot中
3. 只修改需要改变的属性（如m_syncOffset）
4. m_slotName自动保持一致
```

---

## 六、验证slotName配置

### 6.1 编辑器内验证

**检查1：节点显示名称**
```
✅ 正确：Entry synced master : dance_01 | sync_dance_couple_01
✅ 正确：Entry anim: dance_01 | sync_dance_couple_01

❌ 错误：Entry anim: dance_01  （缺少 | slotName 部分）
         → 说明m_isSynchronized = false 或 m_slotName未设置
```

**检查2：Validator输出**
```
保存时查看 Validator output 标签：
- 没有关于sync的错误/警告 → ✅ 配置正确
- 出现 "Sync slot not found" 警告 → ❌ Slave中缺少对应的Entry
```

### 6.2 搜索工具验证

在Workspot tree面板使用搜索功能：

```
1. 在搜索框输入 slotName（如："sync_dance_couple_01"）
2. 查看搜索结果：
   - Master Workspot中应该有1个结果（SyncMasterEntryAnim）
   - 每个Slave Workspot中应该有1个结果（EntryAnim）
3. 逐个检查，确保名称完全一致
```

### 6.3 游戏内调试验证

**使用AI日志**：
```bash
# 启用日志
EnableAILog("ws")

# 查看日志
grep "sync_dance_couple_01" logs/ai_workspot.log

# 应该看到类似输出：
[EntityID:0xABC][ws] PlaySyncAnim: slotName=sync_dance_couple_01
[EntityID:0xDEF][ws] GetSyncEntryIdForSlotName: found entry for sync_dance_couple_01
```

**使用ImGui面板**：
```
F12 → Systems → Workspots → Show ImGui Panel
查看 Workspot列表，确认Master和Slave都在使用相同的slotName
```

---

## 七、常见问题

### 问题1：slotName拼写错误

**症状**：Slave没有同步到Master

**原因**：
```
Master: m_slotName = "sync_dance_couple_01"
Slave:  m_slotName = "sync_dance_copule_01"  ← 拼写错误（copule）
```

**解决**：
1. 复制粘贴，不要手动输入
2. 使用编辑器搜索功能验证
3. 保存后查看Validator输出

### 问题2：忘记设置m_isSynchronized

**症状**：节点显示中没有 `| slotName` 部分

**原因**：
```
Slave EntryAnim:
├─ m_slotName = "sync_dance_couple_01"  ✅ 已设置
└─ m_isSynchronized = false             ❌ 未启用
```

**解决**：
1. 选中Slave的EntryAnim节点
2. 在Properties面板勾选 m_isSynchronized = true
3. 节点显示会立即更新，出现 `| slotName`

### 问题3：slotName冲突

**症状**：不同的同步组意外触发了彼此的同步

**原因**：
```
双人舞A组：
- Master: m_slotName = "dance_01"
- Slave:  m_slotName = "dance_01"

双人舞B组：
- Master: m_slotName = "dance_01"  ← 冲突！
- Slave:  m_slotName = "dance_01"
```

**解决**：
1. 使用唯一的slotName
2. 添加序号：`dance_01`, `dance_02`, `dance_03`
3. 或使用完整命名：`sync_dance_couple_01`, `sync_dance_couple_02`

### 问题4：临时名称遗留到正式版本

**症状**：游戏中出现 `test_slot`, `temp_sync` 等命名

**预防**：
1. 建立命名规范，禁止使用 `test*`, `temp*` 前缀
2. 发布前运行检查脚本：
   ```bash
   grep -r "test_slot\|temp_sync" *.workspot
   ```
3. Code Review时检查slotName命名

---

## 八、高级技巧

### 8.1 使用前缀区分项目区域

```
任务特定：
- q115_dance_sync_01    # 任务115专用
- q203_fight_sync_01    # 任务203专用

通用：
- generic_handshake_01  # 通用握手
- generic_toast_01      # 通用敬酒
```

### 8.2 版本控制

```
v1版本：
- sync_dance_couple_v1

v2版本（改进动画）：
- sync_dance_couple_v2

→ 可以在不影响旧内容的情况下迭代
```

### 8.3 批量重命名工具

如果需要批量修改slotName，可以编写脚本：

```python
# 伪代码：批量替换slotName
for workspot in find_all_workspots():
    for entry in workspot.find_all_entries():
        if entry.m_slotName == "old_name":
            entry.m_slotName = "new_name"
    workspot.save()
```

---

## 九、总结

### 核心要点

1. **m_slotName 是自由命名的字符串**
   - 不是从列表中选择
   - 不是自动生成
   - 完全由美术/设计师手动输入

2. **Master和Slave通过相同的slotName建立同步关系**
   - Master: `m_slotName = "sync_dance_01"`
   - Slave:  `m_slotName = "sync_dance_01"` ← 必须完全相同

3. **系统通过递归查找匹配slotName**
   - `GetSyncEntryIdForSlotName()` 遍历整个WorkspotTree
   - 匹配条件：slotName相同 + m_isSynchronized=true

4. **建议使用统一的命名规范**
   - 前缀：`sync_`
   - 格式：`sync_<场景>_<动作>_<编号>`
   - 避免拼写错误：使用复制粘贴

5. **验证配置正确性**
   - 编辑器：检查节点显示包含 `| slotName`
   - 保存：查看Validator输出无错误
   - 游戏内：使用ImGui或AI日志调试

### 快速检查清单

配置新的同步Entry时：
- [ ] Master的m_slotName已设置（如：`sync_dance_couple_01`）
- [ ] Slave的m_slotName与Master完全相同
- [ ] Slave的m_isSynchronized = true
- [ ] 节点显示中包含 `| <slotName>`
- [ ] slotName遵循项目命名规范
- [ ] Validator输出无错误
- [ ] 在游戏中测试同步效果

现在您完全了解了m_slotName的来源和使用方法！
