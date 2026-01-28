# AI日志系统 - 快速使用指南

## 适用条件
- ✅ **所有版本可用**（包括Final发行版）
- ✅ 无需MOD
- ✅ 无需重新编译
- ⭐ **推荐：如果ImGui不可用，这是最佳选择**

## 操作步骤

### 1. 启用AI日志

**方式A：通过调试菜单（如果有）**
```
Debug Menu → Logging → AI Logs → 勾选 "ws"
```

**方式B：通过控制台**
```
按 ~ 打开控制台
输入：EnableAILog("ws")
```

### 2. 查看日志文件

日志位置：
```
<游戏目录>/logs/ai_workspot.log
```

或者在游戏安装目录的 `logs` 文件夹中查找。

### 3. 实时监控日志

**Windows（使用PowerShell）：**
```powershell
Get-Content "路径\ai_workspot.log" -Wait -Tail 50
```

**使用文本编辑器：**
- 推荐：Notepad++、VS Code等支持自动刷新的编辑器
- 打开日志文件后保持窗口开启

### 4. 日志内容示例

```log
[2026-01-22 15:30:45] [EntityID:0x123456789ABCDEF][ws] Workspot command sent: CMD_Play
[2026-01-22 15:30:45] [EntityID:0x123456789ABCDEF][ws] Workspot setup: workspots/restaurant_chair_01.workspot
[2026-01-22 15:30:46] [EntityID:0x123456789ABCDEF][ws] Animation changed: sit_look_menu
[2026-01-22 15:30:50] [EntityID:0x123456789ABCDEF][ws] Animation changed: sit_eat
[2026-01-22 15:30:55] [EntityID:0x123456789ABCDEF][ws] Workspot command sent: CMD_SlowExit
```

### 5. 找到特定NPC的信息

#### 问题：如何获取NPC的EntityID？

**方式A：在日志中查找（推荐）**
1. 游戏中走到NPC附近
2. 观察NPC的行为（如：坐下、站起等）
3. 在日志中找最新的记录
4. EntityID就在日志行中：`[EntityID:0x123456789ABCDEF]`

**方式B：通过时间戳**
1. 记住NPC执行动作的时间（如：15:30:45）
2. 在日志中搜索该时间段的记录
3. 找到对应的EntityID

**方式C：通过Workspot名称**
如果您知道NPC使用的是哪种类型的Workspot（如餐厅椅子）：
1. 在日志中搜索 `restaurant_chair`
2. 找到符合时间和位置的记录

### 6. 过滤和搜索

**搜索特定Workspot：**
```
在日志中搜索：restaurant_chair_01.workspot
```

**搜索特定EntityID：**
```
在日志中搜索：EntityID:0x123456789ABCDEF
```

**搜索特定命令：**
```
搜索：CMD_Play       # 开始使用Workspot
搜索：CMD_SlowExit   # 退出Workspot
```

## 实战示例

### 场景：餐厅中的NPC坐下吃饭

**步骤1：启用日志**
```
控制台输入：EnableAILog("ws")
```

**步骤2：观察NPC**
- 看着NPC走到椅子旁
- 等待NPC坐下

**步骤3：查看日志**
打开 `ai_workspot.log`，看到最新记录：
```log
[2026-01-22 16:45:30] [EntityID:0xABC123DEF456][ws] Workspot setup: workspots/restaurant_chair_01.workspot
[2026-01-22 16:45:30] [EntityID:0xABC123DEF456][ws] Workspot command sent: CMD_Play
[2026-01-22 16:45:31] [EntityID:0xABC123DEF456][ws] Animation changed: walk_to_sit
[2026-01-22 16:45:35] [EntityID:0xABC123DEF456][ws] Animation changed: sit_look_menu
[2026-01-22 16:45:40] [EntityID:0xABC123DEF456][ws] Animation changed: sit_eat
```

**结果：**
- NPC的EntityID：`0xABC123DEF456`
- 使用的Workspot：`workspots/restaurant_chair_01.workspot`
- 动画序列：`walk_to_sit → sit_look_menu → sit_eat`

## 优势
- ✅ **所有版本可用**（Final版本也行）
- ✅ 历史记录完整，可以回溯
- ✅ 可以搜索和过滤
- ✅ 无性能影响
- ✅ 不需要UI，适合远程调试

## 缺点
- ⚠️ 需要知道NPC的EntityID（或通过时间推断）
- ⚠️ 日志量大时需要搜索
- ⚠️ 不是实时可视化

## 技巧：组合使用

**如果您有其他调试工具：**

1. **ImGui面板（如果可用）** → 获取EntityID
2. **AI日志** → 查看该NPC的详细活动历史

**如果只有AI日志：**
1. 通过时间戳定位
2. 通过Workspot类型过滤
3. 观察NPC行为 → 在日志中找对应时间的记录

## 日志关键字说明

| 日志内容 | 含义 |
|---------|------|
| `Workspot setup: xxx.workspot` | NPC加载了哪个Workspot |
| `CMD_Play` | 开始执行Workspot |
| `CMD_SlowExit` | 正常退出 |
| `CMD_FastExit` | 快速退出（被打断） |
| `CMD_Stop` | 强制停止 |
| `Animation changed: xxx` | 切换到新动画 |

## 代码位置（供参考）

```cpp
// workspotSystem.cpp: 356
RED_AI_LOGF( ownerId, "ws", "Workspot command sent: %hs",
             GetWorkspotCommandFriendlyName( cmd ).AsChar() );
```

所有Workspot命令都会通过这个宏记录到AI日志中。
