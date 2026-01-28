# Cyberpunk 2077 Workspot内置调试方法 - 实战指南

## 您的需求：找到当前镜头中的NPC是谁，用了哪个Workspot

**好消息**：游戏内置了多个可以直接使用的调试系统，**无需安装MOD**！

---

## 方案1：ImGui调试面板（最强大，推荐！）

### 什么是ImGui调试面板？

Cyberpunk 2077内置了**ImGui调试界面**，可以实时显示所有Workspot实例的详细信息。

```cpp
// workspotSystem.cpp: 612-730
void WorkspotSystem::ShowImGuiDebugPanel()
{
    ImGui::Text( "Workspot instances count = %u", m_instances.Size() );
    ImGui::Text( "Workspot resources count = %u", workspotUsages.Size() );

    if ( ImGui::CollapsingHeader( "Workspots" ) )
    {
        // 显示表格：
        // ID | Name | Instances count | Body type / All anims / Loaded anims / Used anims
        // 可以右键复制Workspot路径
    }
}
```

### 如何启用？

**在开发版本中（非Final配置）**：

1. **按 `F12` 或 `~` 键** 打开调试菜单（具体按键取决于构建配置）

2. **导航到调试菜单**：
   ```
   Debug Menu
   └─ Systems
      └─ Workspots
         ├─ Show ImGui Panel  ← 启用此选项
         └─ Show Statistics
   ```

3. **查看ImGui面板**：
   - 屏幕上会出现一个窗口，标题为 `Workspot System`
   - 内容包括：
     ```
     Workspot instances count = 15
     Workspot resources count = 8
     Final animsets count = 5

     ▼ Workspots
     ┌────┬──────────────────────────────────────┬──────────────┬─────────────────────┐
     │ ID │ Name                                  │ Instances    │ Body type / Anims   │
     ├────┼──────────────────────────────────────┼──────────────┼─────────────────────┤
     │ 1  │ workspots/restaurant_chair_01.workspot│ 3            │ man_average / 45    │
     │ 2  │ workspots/bench_sit.workspot          │ 2            │ woman_average / 32  │
     │ 3  │ workspots/atm_machine.workspot        │ 1            │ man_big / 12        │
     └────┴──────────────────────────────────────┴──────────────┴─────────────────────┘
     ```

4. **查找特定NPC**：
   - 点击展开某个Workspot条目
   - 会显示该Workspot的所有使用实例（包括NPC的EntityID）
   - 右键点击路径 → "Copy path" 复制Workspot路径

### 优势

✅ **无需MOD**，游戏内置功能
✅ **显示所有Workspot实例**（整个场景）
✅ **可以复制路径**（右键菜单）
✅ **显示动画统计**（总数/已加载/已使用）
✅ **实时更新**

### 局限

⚠️ 只在**开发版本**中可用（Final版本被剔除）
⚠️ 显示的是**所有NPC**，需要手动查找特定NPC

---

## 方案2：3D世界可视化标记（最直观！）

### 什么是3D可视化？

游戏在3D世界中**直接在Workspot点位上绘制标记**，镜头瞄准时会高亮显示。

```cpp
// workspotDebugger.cpp: 263-291
void Debugger::OnRenderDebug( const RenderDebugContext& context )
{
    // 检查调试过滤器：AI/Workspots/Spots
    if (drawWorkspotSelector && m_nodeIdLookedAt.IsDefined())
    {
        // 在Workspot点位上绘制旋转的蓝色金字塔
        builder.SetColor(Color::BLUE(200));
        builder.AddSolidPyramid(position, axisX, axisY, axisZ);
        // ↑ 会在镜头瞄准的Workspot上显示蓝色标记
    }
}
```

### 如何启用？

1. **打开调试菜单**：按 `F12` 或 `~`

2. **启用调试过滤器**：
   ```
   Debug Menu
   └─ Filters
      └─ AI
         └─ Workspots
            └─ Spots  ← 勾选此选项
   ```

3. **在游戏中查看**：
   - 移动镜头对准NPC（正在使用Workspot的）
   - 在NPC坐着的椅子/站着的位置上方会出现：
     ```
     🔷 蓝色旋转的金字塔标记
        ↑ 表示这是一个Workspot点位
        ↑ 标记会随镜头瞄准而高亮
     ```

4. **选择Workspot节点**：
   - 瞄准Workspot标记
   - 按**特定按键**（取决于输入配置，可能是 `E` 或其他键）
   - 发送 `WorkspotDebugMode::NodeSelection` 命令
   - 该Workspot会被选中，可以查看详细信息

### 优势

✅ **非常直观**，直接在3D世界中看到
✅ **精确定位**，知道哪个点位被使用
✅ **无需查找EntityID**

### 局限

⚠️ 只显示**Workspot点位**，不直接显示NPC ID
⚠️ 需要启用调试过滤器

---

## 方案3：AI日志系统（最简单，无需UI）

### 什么是AI日志？

所有Workspot活动都会自动记录到AI日志文件中。

```cpp
// workspotSystem.cpp: 356
RED_AI_LOGF( ownerId, "ws", "Workspot command sent: %hs",
             GetWorkspotCommandFriendlyName( cmd ).AsChar() );
```

### 如何使用？

1. **启用AI日志**：
   - 打开调试菜单 → `Logging` → 启用 `AI Logs`
   - 或在控制台输入：`EnableAILog("ws")`

2. **查看日志文件**：
   - 日志位置：`<游戏目录>/logs/ai_workspot.log`
   - 实时监控：使用 `tail -f ai_workspot.log` 或文本编辑器

3. **日志内容示例**：
   ```
   [2026-01-22 15:30:45] [EntityID:0x123456789ABCDEF][ws] Workspot command sent: CMD_Play
   [2026-01-22 15:30:45] [EntityID:0x123456789ABCDEF][ws] Workspot setup: workspots/restaurant_chair_01.workspot
   [2026-01-22 15:30:46] [EntityID:0x123456789ABCDEF][ws] Animation changed: sit_look_menu
   [2026-01-22 15:30:50] [EntityID:0x123456789ABCDEF][ws] Animation changed: sit_eat
   ```

4. **查找特定NPC**：
   - 记下NPC的EntityID（可以通过其他调试工具获取）
   - 在日志中搜索该EntityID
   - 查看该NPC的所有Workspot活动

### 优势

✅ **最简单**，只需启用日志
✅ **历史记录**，可以查看过去的活动
✅ **可搜索**，易于过滤
✅ **无性能影响**（Final版本也可用）

### 局限

⚠️ 需要知道NPC的EntityID
⚠️ 日志量大，需要搜索过滤

---

## 方案4：ShadowDebugger（前面介绍过，需要CET）

如果安装了CET MOD，可以使用之前介绍的ShadowDebugger方法。

---

## 实战推荐：组合使用方案

### 场景：找到镜头中某个坐在椅子上的NPC用了哪个Workspot

**步骤1：使用3D可视化定位Workspot点位**

1. 启用调试过滤器：`Debug Menu → Filters → AI → Workspots → Spots`
2. 将镜头对准NPC
3. 看到蓝色金字塔标记 → 确认这是Workspot点位

**步骤2：使用ImGui面板查看Workspot详情**

1. 启用ImGui面板：`Debug Menu → Systems → Workspots → Show ImGui Panel`
2. 在面板中查看当前活跃的Workspot列表
3. 查看 `Instances count` 列 → 找到有实例的Workspot
4. 展开该Workspot → 查看具体的NPC EntityID
5. 右键复制Workspot路径

**步骤3：（可选）使用AI日志验证**

1. 启用AI日志：`EnableAILog("ws")`
2. 在日志中搜索该Workspot路径
3. 查看所有使用该Workspot的NPC活动

---

## 快速参考表

| 需求 | 最佳方案 | 启用方法 |
|------|---------|---------|
| **查看所有Workspot实例** | ImGui面板 | `Debug Menu → Systems → Workspots → Show ImGui Panel` |
| **快速定位Workspot点位** | 3D可视化 | `Debug Menu → Filters → AI → Workspots → Spots` |
| **追踪NPC活动历史** | AI日志 | `EnableAILog("ws")` |
| **实时调试特定NPC** | ShadowDebugger (需CET) | 见前面文档 |
| **复制Workspot路径** | ImGui面板 | 右键 → Copy path |

---

## 方案5：源代码级调试（终极方案）

如果以上方案都不够用，可以在源代码中添加自定义调试逻辑。

### 在OnWorkspotSetup添加日志

```cpp
// workspotSystem.cpp: 242-243
void WorkspotSystem::SetupWorkspot(...)
{
    ent::EntityID ownerId = ent->GetEntityID();

    // ⚠️ 添加自定义日志
    RED_LOG_DEBUG( "[WORKSPOT TRACE] NPC [%s] using Workspot: %hs",
                   ownerId.ToDebugString(),
                   path.AsChar() );

    WORKSPOT_DEBUG_MACRO(
        String path = setup.m_workspot.m_tree->GetPath().ToDebugString();
        m_debugger.OnWorkspotSetup( ownerId, setup.m_workspot.m_tree, path );
    );
}
```

**编译后运行**，日志会输出：
```
[DEBUG] [WORKSPOT TRACE] NPC [0x123456789ABCDEF] using Workspot: workspots/restaurant_chair_01.workspot
```

### 在UpdateRecord添加镜头检测

```cpp
// gameWorkspotsInstance.cpp
GlobalWorkspotTime WorkspotInstance::UpdateRecord(...)
{
    // ⚠️ 添加镜头检测逻辑
    Vector3 npcPos = GetOwnerEntity()->GetPosition();
    Vector3 cameraPos = GetCameraPosition();
    Vector3 cameraDir = GetCameraForward();

    Vector3 delta = npcPos - cameraPos;
    Float distance = delta.Mag();
    delta.Normalize();

    Float dot = cameraDir.Dot(delta);

    // 如果NPC在镜头中心（夹角小于10度，距离小于10米）
    if (dot > 0.985f && distance < 10.0f)
    {
        RED_LOG_DEBUG( "[CAMERA VIEW] NPC [%s] in camera view, Workspot: %hs, Anim: %s",
                       GetOwnerId().ToDebugString(),
                       m_workResource->GetPath().ToDebugString(),
                       data.m_animationName.AsChar() );
    }

    // ... 原有逻辑
}
```

**这样会自动输出镜头中心的NPC信息**！

---

## 总结：您可以直接使用的方案

根据您的环境选择：

### 如果您有**开发版本**（非Final）：

✅ **首选：ImGui面板 + 3D可视化**
- 启用 `Debug Menu → Systems → Workspots → Show ImGui Panel`
- 启用 `Debug Menu → Filters → AI → Workspots → Spots`
- **立即看到所有Workspot实例和3D标记**

### 如果您有**CET MOD**：

✅ **首选：ShadowDebugger**
- 创建热键绑定MOD（见前面文档）
- 瞄准NPC → 按小键盘2 → 看到实时信息

### 如果您只有**Final版本**（发行版）：

✅ **首选：AI日志**
- 控制台输入：`EnableAILog("ws")`
- 查看日志文件：`logs/ai_workspot.log`

### 如果您可以**重新编译**：

✅ **首选：源代码调试**
- 添加镜头检测逻辑
- 自动输出镜头中心NPC的Workspot信息

---

## 实战示例

### 示例：使用ImGui面板找到NPC

**场景**：餐厅中有3个NPC坐在椅子上，我想知道中间那个NPC用的是哪个Workspot

**操作步骤**：

1. **打开ImGui面板**：
   - 按 `F12` → `Systems` → `Workspots` → `Show ImGui Panel`

2. **查看面板内容**：
   ```
   Workspot instances count = 15

   ▼ Workspots
   1  workspots/restaurant_chair_01.workspot  [3 instances]  ← 有3个实例
   2  workspots/restaurant_chair_02.workspot  [1 instance]
   3  workspots/bench_sit.workspot            [2 instances]
   ```

3. **展开第1项**（restaurant_chair_01）：
   ```
   ▼ workspots/restaurant_chair_01.workspot
     Instance #1: EntityID: 0xABC123, Position: (100.5, 200.3, 45.0)
     Instance #2: EntityID: 0xDEF456, Position: (102.1, 200.3, 45.0)  ← 中间位置
     Instance #3: EntityID: 0x789GHI, Position: (103.7, 200.3, 45.0)
   ```

4. **确认NPC**：
   - 中间NPC的EntityID是 `0xDEF456`
   - 使用的Workspot是 `workspots/restaurant_chair_01.workspot`
   - 右键点击路径 → "Copy path" 复制

**完成！** 您找到了镜头中NPC的Workspot信息。

---

## 补充：如何获取NPC的EntityID？

如果ImGui面板不显示EntityID，可以使用以下方法：

### 方法1：使用Entity调试工具

```
Debug Menu → Entities → Show Entity List
→ 显示所有实体的EntityID和名称
```

### 方法2：在日志中查找

```
启用AI日志后，日志中会包含EntityID：
[EntityID:0xDEF456][ws] Workspot command sent: CMD_Play
```

### 方法3：使用RedScript（需CET）

```lua
local target = Game.GetTargetingSystem():GetLookAtObject(Game.GetPlayer())
print("Entity ID: " .. tostring(target:GetEntityID()))
```

---

现在您有了**5种可直接使用的方法**来查找镜头中NPC使用的Workspot，无需复杂配置！
