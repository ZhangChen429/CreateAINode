# Cyberpunk 2077 Workspot游戏内调试方法

## 核心问题：如何在游戏中实时调试Workspot？

基于源代码分析，这里是完整的游戏内调试激活方法。

---

## 一、ShadowDebugger调试系统（游戏内实时调试）

### 1.1 ShadowDebugger是什么？

```cpp
// workspotShadowDebugger.cpp
class ShadowDebugger : public DebuggingTool
{
    // 实时追踪所有NPC的Workspot活动
    // 显示：
    //   - 动画历史（最近7个动画）
    //   - 事件历史（最近15条消息）
    //   - Workspot资源路径
    //   - EntityID
};
```

**核心功能**：
- ✅ 自动追踪场景中**所有NPC**的Workspot活动
- ✅ 在屏幕上显示选定NPC的实时信息
- ✅ 检测常见问题（缺失动画、过多跳过、Rig不匹配）
- ✅ 数据持久化45秒（NPC离开Workspot后仍可查看）

---

## 二、激活调试功能的方法

### 方法1：通过RedScript发送调试命令（推荐）

#### 步骤1：启用ShadowDebugger

```swift
// RedScript代码（在CET控制台或MOD中执行）

// 1. 获取WorkspotSystem
let workspotSystem = GameInstance.GetWorkspotSystem(GetGameInstance());

// 2. 创建调试命令数据
let debugData = new DebuggerCommandData();
debugData.m_mode = WorkspotDebugMode.ShadowActivate; // 启用ShadowDebugger

// 3. 发送到特定NPC（或所有NPC）
let targetNPC = Game.GetTargetingSystem().GetLookAtObject(player);
let npcID = targetNPC.GetEntityID();

workspotSystem.SendCommand(npcID, CMD_DebuggerCmd, debugData);
```

#### 步骤2：切换显示特定NPC的详细信息

```swift
// 切换显示/隐藏特定NPC的调试面板
let debugData = new DebuggerCommandData();
debugData.m_mode = WorkspotDebugMode.ShadowToogleDebugData;

workspotSystem.SendCommand(npcID, CMD_DebuggerCmd, debugData);
```

**屏幕显示示例**：
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- Id: [0x123456789ABCDEF] Resource: [workspots/restaurant_chair_01.workspot]
- Events history - - - - - - - - - - - -
  Workspot started - previously was workspots/bench_sit.workspot
  User entity at/detached Loc[100.50, 200.30, 45.00], Cor[0.00, 0.00, 0.00]
  Exit anim picked [sit_to_walk]
  Item action executed - EquipItem [food_item][LeftHand]
- Anim history - - - - - - - - - - - -
  walk_to_sit
  sit_look_menu
  sit_eat
  sit_drink
  sit_idle
  sit_to_walk
  stand
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### 步骤3：禁用ShadowDebugger

```swift
let debugData = new DebuggerCommandData();
debugData.m_mode = WorkspotDebugMode.ShadowDeactivate;

workspotSystem.SendCommand(npcID, CMD_DebuggerCmd, debugData);
```

---

### 方法2：通过CET (Cyber Engine Tweaks) 控制台

如果您安装了CET MOD，可以在游戏中直接使用Lua控制台：

```lua
-- 打开CET控制台 (默认热键: ~)

-- 激活ShadowDebugger
function ActivateWorkspotDebugger()
    local player = Game.GetPlayer()
    local target = Game.GetTargetingSystem():GetLookAtObject(player)

    if target then
        local npcID = target:GetEntityID()
        local ws = Game.GetWorkspotSystem()

        -- 发送激活命令
        local data = DebuggerCommandData.new()
        data.m_mode = WorkspotDebugMode.ShadowActivate
        ws:SendCommand(npcID, CMD_DebuggerCmd, data)

        print("ShadowDebugger activated for NPC: " .. tostring(npcID))
    else
        print("No NPC targeted!")
    end
end

-- 调用函数
ActivateWorkspotDebugger()

-- 切换显示
function ToggleWorkspotDisplay()
    local player = Game.GetPlayer()
    local target = Game.GetTargetingSystem():GetLookAtObject(player)

    if target then
        local npcID = target:GetEntityID()
        local ws = Game.GetWorkspotSystem()

        local data = DebuggerCommandData.new()
        data.m_mode = WorkspotDebugMode.ShadowToogleDebugData
        ws:SendCommand(npcID, CMD_DebuggerCmd, data)

        print("Toggled debug display for NPC: " .. tostring(npcID))
    end
end
```

**使用流程**：
1. 瞄准（看向）一个使用Workspot的NPC
2. 打开CET控制台（默认按 `~` 键）
3. 输入：`ActivateWorkspotDebugger()`
4. 输入：`ToggleWorkspotDisplay()`
5. 屏幕左上角会显示该NPC的Workspot信息

---

### 方法3：创建MOD绑定到热键

创建一个CET MOD，将调试功能绑定到小键盘：

```lua
-- init.lua (CET MOD文件)

local debuggerActive = false
local displayActive = false

registerForEvent("onInit", function()
    print("[Workspot Debugger] Loaded!")
    print("Numpad 2: Toggle ShadowDebugger")
    print("Numpad 3: Toggle Display")
end)

registerForEvent("onUpdate", function(deltaTime)
    -- 监听小键盘2键
    if Game.GetInputManager():IsActionJustPressed("Numpad2") then
        local player = Game.GetPlayer()
        local target = Game.GetTargetingSystem():GetLookAtObject(player)

        if target then
            local npcID = target:GetEntityID()
            local ws = Game.GetWorkspotSystem()

            local data = DebuggerCommandData.new()

            if not debuggerActive then
                data.m_mode = WorkspotDebugMode.ShadowActivate
                print("[Workspot Debugger] Activated for NPC: " .. tostring(npcID))
            else
                data.m_mode = WorkspotDebugMode.ShadowDeactivate
                print("[Workspot Debugger] Deactivated")
            end

            ws:SendCommand(npcID, CMD_DebuggerCmd, data)
            debuggerActive = not debuggerActive
        end
    end

    -- 监听小键盘3键
    if Game.GetInputManager():IsActionJustPressed("Numpad3") then
        local player = Game.GetPlayer()
        local target = Game.GetTargetingSystem():GetLookAtObject(player)

        if target then
            local npcID = target:GetEntityID()
            local ws = Game.GetWorkspotSystem()

            local data = DebuggerCommandData.new()
            data.m_mode = WorkspotDebugMode.ShadowToogleDebugData
            ws:SendCommand(npcID, CMD_DebuggerCmd, data)

            print("[Workspot Debugger] Toggled display for NPC: " .. tostring(npcID))
        end
    end
end)
```

**使用方法**：
1. 将上述代码保存为 `CET/mods/WorkspotDebugger/init.lua`
2. 重启游戏或重载CET
3. 在游戏中：
   - 瞄准NPC
   - 按 **小键盘2**：启用/禁用ShadowDebugger
   - 按 **小键盘3**：显示/隐藏当前NPC的详细信息

---

## 三、ShadowDebugger显示的信息详解

### 3.1 动画历史 (Anim History)

```cpp
// 最近7个动画名称
m_animHistory.PushBack( animName );
while ( m_animHistory.Size() > MAX_ANIM_HISTORY )  // 7
{
    m_animHistory.RemoveAt( 0 );
}
```

**显示内容**：
```
- Anim history - - - - - - - - - - - -
  walk_to_sit        ← 进入动画
  sit_look_menu      ← 第1个动作
  sit_eat            ← 第2个动作
  sit_drink          ← 第3个动作
  sit_fidget         ← idle动作
  sit_to_walk        ← 退出动画
  stand              ← 离开后的状态
```

---

### 3.2 事件历史 (Events History)

```cpp
// 最近15条事件消息
void AddMessage( ActorData* data, const String& message )
{
    data->m_messages.PushBack( message );
    while( data->m_messages.Size() > MAX_MESSAGE_HISTORY )  // 15
    {
        data->m_messages.RemoveAt( 0 );
    }
}
```

**记录的事件类型**：

| 事件类型 | 触发时机 | 消息示例 |
|---------|---------|---------|
| **Workspot Started** | Setup新Workspot | `Workspot started - previously was workspots/bench.workspot` |
| **Rig Mismatch** | Body Type不匹配 | `Entity rig not present in workspot animset setup - possible streaming/motion issues` |
| **User Attached** | NPC移动到点位 | `User entity at/detached Loc[100.0, 200.0, 45.0], Cor[0.0, 0.0, 0.0]` |
| **Exit Picked** | 选择退出动画 | `Exit anim picked [sit_to_walk]` |
| **Missing Animation** | 动画文件缺失 | `Missing animation [sit_custom] - possible glitch/blend` |
| **Skip Overflow** | 跳过过多动画 | `Too many animations skipped in a row - possible glitch` |
| **Item Action** | 执行物品操作 | `T[123456789] Item action executed - EquipItem [food][LeftHand]` |

---

### 3.3 Workspot资源路径

```cpp
OnWorkspotSetup( ... )
{
    data->m_resourcePath = path;
}
```

**显示内容**：
```
- Id: [0x123456789ABCDEF] Resource: [workspots/restaurant_chair_01.workspot]
```

**用途**：
- 快速定位NPC当前使用的WorkspotTree文件
- 验证是否使用了正确的Workspot

---

## 四、其他调试模式

### 4.1 所有可用的调试模式

```cpp
// workspotDebugger.h: 41-63
enum WorkspotDebugMode
{
    VisualLogToogle      = RED_FLAG(1),   // 可视化日志开关
    VisualLogOn          = RED_FLAG(2),   // 启用可视化日志
    VisualLogOff         = RED_FLAG(3),   // 禁用可视化日志
    VisualStateToogle    = RED_FLAG(4),   // 状态显示开关
    VisualStateOn        = RED_FLAG(5),   // 启用状态显示
    VisualStateOff       = RED_FLAG(6),   // 禁用状态显示
    RecorderOn           = RED_FLAG(7),   // 启用录制
    RecorderOff          = RED_FLAG(8),   // 禁用录制
    PlaybackOn           = RED_FLAG(9),   // 启用回放
    PlaybackOff          = RED_FLAG(10),  // 禁用回放
    StopAll              = RED_FLAG(11),  // 停止所有调试
    Invalid              = RED_FLAG(12),  // 无效
    FunctionalTests      = RED_FLAG(13),  // 功能测试
    AnimObject           = RED_FLAG(14),  // 动画对象调试
    ProfilerOn           = RED_FLAG(15),  // 启用性能分析
    ProfilerOff          = RED_FLAG(16),  // 禁用性能分析
    ShadowActivate       = RED_FLAG(17),  // ⚠️ 启用ShadowDebugger
    ShadowDeactivate     = RED_FLAG(18),  // ⚠️ 禁用ShadowDebugger
    ShadowToogleDebugData= RED_FLAG(19),  // ⚠️ 切换显示数据
    NodeSelection        = RED_FLAG(20),  // 节点选择
};
```

### 4.2 AnimObject调试模式

```swift
// 激活AnimObject调试器（追踪动画对象）
let debugData = new DebuggerCommandData();
debugData.m_mode = WorkspotDebugMode.AnimObject;

workspotSystem.SendCommand(npcID, CMD_DebuggerCmd, debugData);
```

**用途**：专注于动画对象的调试（AnimGraph、Rig、AnimSet等）

---

### 4.3 FunctionalTests调试模式

```swift
// 激活功能测试调试器
let debugData = new DebuggerCommandData();
debugData.m_mode = WorkspotDebugMode.FunctionalTests;

workspotSystem.SendCommand(npcID, CMD_DebuggerCmd, debugData);
```

**用途**：
- 自动化测试Workspot功能
- 调用RedScript事件进行验证
- 记录详细的测试日志

---

## 五、实战调试场景

### 场景1：NPC动画不播放

**症状**：NPC移动到Workspot点位后，没有播放动画

**调试步骤**：

1. **激活ShadowDebugger**：
   ```lua
   -- 瞄准NPC
   ActivateWorkspotDebugger()
   ToggleWorkspotDisplay()
   ```

2. **查看事件历史**：
   ```
   - Events history - - - - - - - - - - - -
     Workspot started - previously was None
     Entity rig not present in workspot animset setup ← ⚠️ 问题！
   ```

3. **根因**：NPC的Rig类型（如 `man_average`）不在Workspot的AnimSet支持列表中

4. **解决方案**：
   - 检查WorkspotTree的AnimSet配置
   - 添加缺失的Body Type支持

---

### 场景2：动画频繁跳变

**症状**：动画播放不流畅，有明显的跳变

**调试步骤**：

1. **查看事件历史**：
   ```
   - Events history - - - - - - - - - - - -
     Too many animations skipped in a row - possible glitch ← ⚠️ 警告！
     Missing animation [sit_custom_01] - possible glitch/blend
   ```

2. **查看动画历史**：
   ```
   - Anim history - - - - - - - - - - - -
     sit_idle
     sit_idle        ← 同一个动画反复出现
     sit_idle
     sit_custom_01   ← 缺失，导致跳过
     sit_idle
   ```

3. **根因**：自定义动画 `sit_custom_01` 文件缺失或未加载

4. **解决方案**：
   - 检查动画文件是否存在
   - 验证AnimSet是否正确引用

---

### 场景3：Workspot点位偏移

**症状**：NPC坐下后位置不正确

**调试步骤**：

1. **查看事件历史**：
   ```
   - Events history - - - - - - - - - - - -
     User entity at/detached Loc[100.50, 200.30, 45.00], Cor[5.00, 3.00, 0.00]
                                                           ↑ Correction偏移量很大
   ```

2. **分析**：
   - `Loc` = Workspot点位的世界坐标
   - `Cor` = 修正变换（Correction Transform）
   - 如果Cor偏移量很大（如 5米），说明NPC初始位置与Workspot点位差距很大

3. **解决方案**：
   - 检查EntryAnim是否正确（应该有移动到点位的动画）
   - 验证Workspot的Transform是否正确

---

## 六、开发/调试构建配置

### 6.1 确保调试功能已启用

```cpp
// workspotDebugger.h: 10-22
#if !defined( RED_CONFIGURATION_FINAL ) || defined(USE_PROFILER)
#define WORKSPOT_DEBUG_ENABLED
#endif

#ifdef WORKSPOT_DEBUG_ENABLED
#define WORKSPOT_DEBUG_MACRO( x ) x
#else
#define WORKSPOT_DEBUG_MACRO( x )
#endif
```

**检查方法**：
- 调试功能仅在 **非Final配置** 或定义了 `USE_PROFILER` 时启用
- 如果是Final版本（发行版），调试功能会被完全剔除

### 6.2 编译选项

确保使用以下配置编译：

| 配置 | 调试功能 |
|------|---------|
| **Debug** | ✅ 完全启用 |
| **Release (Dev)** | ✅ 启用（带USE_PROFILER） |
| **Release (Final)** | ✗ 禁用 |

---

## 七、快速参考表

### 常用命令速查

| 操作 | RedScript命令 | 键盘快捷键（MOD） |
|------|--------------|-----------------|
| **启用ShadowDebugger** | `WorkspotDebugMode.ShadowActivate` | 小键盘2 |
| **禁用ShadowDebugger** | `WorkspotDebugMode.ShadowDeactivate` | 小键盘2（再次） |
| **切换显示面板** | `WorkspotDebugMode.ShadowToogleDebugData` | 小键盘3 |
| **启用AnimObject调试** | `WorkspotDebugMode.AnimObject` | - |
| **启用FunctionalTests** | `WorkspotDebugMode.FunctionalTests` | - |

### ShadowDebugger显示内容

| 区域 | 显示内容 | 最大数量 |
|------|---------|---------|
| **Id & Resource** | EntityID + Workspot路径 | 1 |
| **Events History** | 事件消息 | 15条 |
| **Anim History** | 动画名称 | 7个 |

### 数据持久化

```cpp
const Float PERSISTENCE_TIME = 45.f;  // 45秒
```

**说明**：NPC离开Workspot后，数据仍会保留45秒，便于事后分析。

---

## 八、总结

### 回答您的核心问题

**"有没有类似 workspot.EnableShadowDebugger 小键盘2 这样的调试方式？"**

**是的！** 完整流程：

1. **安装CET MOD**（如果还没有）

2. **创建热键绑定MOD**：
   - 创建文件：`CET/mods/WorkspotDebugger/init.lua`
   - 复制上面的热键绑定代码

3. **游戏内使用**：
   - 瞄准一个使用Workspot的NPC
   - 按 **小键盘2**：激活ShadowDebugger
   - 按 **小键盘3**：显示该NPC的调试信息
   - 屏幕左上角会显示实时的Workspot状态

4. **查看信息**：
   - Workspot资源路径
   - 最近7个动画
   - 最近15条事件（包括警告/错误）

### 核心优势

相比于之前文档中的"源代码级调试"：

| 方法 | 优点 | 缺点 |
|------|------|------|
| **源代码调试** | 完整的调用栈、所有变量 | 需要重新编译、断点会暂停游戏 |
| **ShadowDebugger** | 游戏内实时显示、无需重启 | 信息有限（只有最近的数据） |

**最佳实践**：
- 快速定位问题 → 使用 **ShadowDebugger**
- 深入分析根因 → 使用 **源代码调试**

现在您可以像CDPR开发者一样，在游戏中实时调试Workspot了！
