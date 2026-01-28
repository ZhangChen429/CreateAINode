# 源码编译版本 - 启用ImGui调试面板完整指南

## 关键：调试功能的编译开关

根据源码分析，ImGui调试面板受以下宏控制：

```cpp
// workspotDebugger.h: 10-22
#if !defined( RED_CONFIGURATION_FINAL ) || defined(USE_PROFILER)
#define WORKSPOT_DEBUG_ENABLED
#endif

#ifdef WORKSPOT_DEBUG_ENABLED
#define WORKSPOT_DEBUG_MACRO( x ) x  // 调试代码会被编译进去
#else
#define WORKSPOT_DEBUG_MACRO( x )    // 调试代码被剔除
#endif
```

**结论**：
- ✅ **Debug 配置**：调试功能自动启用
- ✅ **Release (Dev) 配置**：调试功能启用
- ✅ **定义了 USE_PROFILER**：调试功能启用
- ✗ **Final 配置**：调试功能被完全剔除

---

## 📋 步骤1：检查当前构建配置

### 方式A：查看Visual Studio配置

打开您的解决方案（.sln文件），查看顶部工具栏：

```
┌──────────────────────────────────────┐
│ 解决方案配置：                        │
│ ┌────────────────────────────────┐  │
│ │ Debug          │ ▼             │  │  ← 当前配置
│ └────────────────────────────────┘  │
│                                      │
│ 解决方案平台：                        │
│ ┌────────────────────────────────┐  │
│ │ x64            │ ▼             │  │
│ └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

**配置说明**：
- **Debug**：调试版本，所有调试功能启用，性能较低
- **Release**：发行版本，可能分为：
  - **Release (Dev)**：开发用发行版，调试功能启用
  - **Release (Final)**：最终发行版，调试功能剔除

### 方式B：查看可执行文件属性

1. 找到编译出的 `Cyberpunk2077.exe`
2. 右键 → 属性 → 详细信息
3. 查看 **文件描述** 或 **产品版本**
4. 如果包含 "Debug" 或 "Dev" 字样，说明是开发版本

### 方式C：直接运行测试

1. 启动编译出的游戏
2. 按 **F12** 或 **~** 键
3. 如果出现调试菜单 → ✅ 调试功能已启用
4. 如果没反应 → ✗ 当前是Final配置

---

## 🎯 步骤2：启用ImGui面板（无需重新编译）

**前提**：当前构建配置不是Final

### 2.1 游戏内启用

**方法1：通过调试菜单**
```
1. 启动游戏
2. 按 F12 键
3. 导航到：
   Debug Menu
   └─ Systems
      └─ Workspots
         └─ Show ImGui Panel  ← 勾选
```

**方法2：通过控制台命令（如果支持）**
```
按 ~ 打开控制台
输入：debug.workspot.showImGui 1
```

### 2.2 验证是否启用

如果成功，屏幕左上角会出现：

```
┌─────────────────────────────────────────┐
│ Workspot System                         │
├─────────────────────────────────────────┤
│ Workspot instances count = 15           │
│ Workspot resources count = 8            │
│ Final animsets count = 5                │
│                                          │
│ ▼ Workspots                             │
│ ┌────┬────────────┬──────────┬────────┐│
│ │ ID │ Name       │ Instances│ ...    ││
│ └────┴────────────┴──────────┴────────┘│
└─────────────────────────────────────────┘
```

---

## 🔧 步骤3：如果需要重新编译

### 情况1：当前是Final配置

**解决方案A：切换到Dev配置**

1. 打开Visual Studio
2. 顶部工具栏：**解决方案配置** → 选择 **Debug** 或 **Release (Dev)**
3. 重新编译：
   ```
   生成 → 重新生成解决方案
   ```
4. 等待编译完成（可能需要30分钟到数小时）

**解决方案B：在Final配置中启用Profiler**

修改项目属性：
```
项目 → 属性
└─ C/C++
   └─ 预处理器
      └─ 预处理器定义
         → 添加：USE_PROFILER
```

然后重新编译。

### 情况2：想强制启用调试功能

**修改源码**（不推荐，仅用于测试）：

编辑 `D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev\src\common\gameWorkspots\include\workspotDebugger.h`

```cpp
// 原代码：
#if !defined( RED_CONFIGURATION_FINAL ) || defined(USE_PROFILER)
#define WORKSPOT_DEBUG_ENABLED
#endif

// 修改为（强制启用）：
#define WORKSPOT_DEBUG_ENABLED  // 强制启用，无论配置如何
```

保存后重新编译。

⚠️ **警告**：这会导致Final版本也包含调试代码，增加文件大小和性能开销。

---

## 📊 不同配置的对比

| 配置 | 调试功能 | 性能 | 文件大小 | ImGui可用 |
|------|---------|------|---------|----------|
| **Debug** | ✅ 完全启用 | 慢 | 大 | ✅ |
| **Release (Dev)** | ✅ 启用 | 快 | 中 | ✅ |
| **Release (Final)** | ✗ 剔除 | 最快 | 小 | ✗ |
| **Final + USE_PROFILER** | ✅ 部分启用 | 较快 | 中 | ✅ |

---

## 🚀 推荐工作流程

### 开发调试阶段
```
1. 使用 Debug 或 Release (Dev) 配置
2. 启用所有调试功能
3. ImGui面板始终可用
```

### 性能测试阶段
```
1. 使用 Release (Dev) 配置
2. 可以临时关闭ImGui面板
3. 保留日志功能
```

### 发布阶段
```
1. 使用 Release (Final) 配置
2. 所有调试功能被剔除
3. 最优性能
```

---

## 🔍 调试ImGui面板的代码位置

### ShowImGuiDebugPanel() 实现

```cpp
// workspotSystem.cpp: 612-730
void WorkspotSystem::ShowImGuiDebugPanel()
{
    ImGui::Text( "Workspot instances count = %u", m_instances.Size() );
    ImGui::Text( "Workspot resources count = %u", workspotUsages.Size() );
    ImGui::Text( "Final animsets count = %u", m_finalAnimsets.Size() );

    if ( ImGui::CollapsingHeader( "Workspots" ) )
    {
        ImGui::Columns( 4, "Workspots" );
        ImGui::Separator();

        ImGui::Text( "ID" );
        ImGui::NextColumn();
        ImGui::Text( "Name" );
        ImGui::NextColumn();
        ImGui::Text( "Instances count" );
        ImGui::NextColumn();
        ImGui::Text( "Body type / All anims / Loaded anims / Used anims" );
        ImGui::NextColumn();

        ImGui::Separator();

        // 遍历所有Workspot资源
        for ( auto& [handle, usage] : workspotUsages )
        {
            String path = usage.m_tree->GetPath().ToDebugString();

            // 显示ID
            ImGui::Text( "%u", handle.GetValue() );
            ImGui::NextColumn();

            // 显示路径（可右键复制）
            if ( ImGui::Selectable( path.AsChar() ) )
            {
                ImGui::SetClipboardText( path.AsChar() );
            }
            ImGui::NextColumn();

            // 显示实例数量
            ImGui::Text( "%u", usage.m_instances.Size() );
            ImGui::NextColumn();

            // 显示动画统计
            ImGui::Text( "%s / %u / %u / %u",
                usage.m_bodyType.AsChar(),
                usage.m_allAnimsCount,
                usage.m_loadedAnimsCount,
                usage.m_usedAnimsCount
            );
            ImGui::NextColumn();
        }

        ImGui::Columns( 1 );
        ImGui::Separator();
    }
}
```

### 调用位置

在每帧更新时，如果启用了ImGui调试，系统会调用：

```cpp
// 伪代码
void WorkspotSystem::OnImGuiRender()
{
    if ( g_debugSettings.showWorkspotImGui )  // 用户勾选了"Show ImGui Panel"
    {
        ShowImGuiDebugPanel();
    }
}
```

---

## 🛠️ 自定义ImGui面板（高级）

如果您想添加自己的调试信息：

### 示例：显示当前镜头中心的Workspot

```cpp
// 在 ShowImGuiDebugPanel() 中添加

void WorkspotSystem::ShowImGuiDebugPanel()
{
    // ... 原有代码 ...

    // ⚠️ 新增：显示镜头中心的Workspot
    if ( ImGui::CollapsingHeader( "Camera View Debug" ) )
    {
        Vector3 cameraPos = GetCameraPosition();
        Vector3 cameraDir = GetCameraForward();

        ImGui::Text( "Camera Position: (%.2f, %.2f, %.2f)",
            cameraPos.X, cameraPos.Y, cameraPos.Z );

        // 查找镜头中心最近的Workspot实例
        WorkspotInstance* closestInstance = nullptr;
        Float minDistance = FLT_MAX;

        for ( auto& instance : m_instances )
        {
            Vector3 instancePos = instance.GetPosition();
            Vector3 delta = instancePos - cameraPos;
            Float distance = delta.Mag();

            delta.Normalize();
            Float dot = cameraDir.Dot( delta );

            // 在镜头前方且夹角小于15度
            if ( dot > 0.966f && distance < minDistance )
            {
                minDistance = distance;
                closestInstance = &instance;
            }
        }

        if ( closestInstance )
        {
            ImGui::Separator();
            ImGui::Text( "Closest Workspot in Camera View:" );
            ImGui::Text( "  EntityID: %s",
                closestInstance->GetOwnerId().ToDebugString() );
            ImGui::Text( "  Workspot: %s",
                closestInstance->GetWorkspotPath().AsChar() );
            ImGui::Text( "  Distance: %.2f meters", minDistance );

            // 显示当前动画
            WorkspotEntryData data;
            closestInstance->GetCurrentData( data );
            ImGui::Text( "  Current Anim: %s", data.m_animationName.AsChar() );
        }
        else
        {
            ImGui::Text( "No Workspot in camera view" );
        }
    }
}
```

编译后，ImGui面板会自动显示镜头中心的NPC信息！

---

## ⚡ 快速检查清单

在启动游戏前，确认：

- [ ] 使用的是 Debug 或 Release (Dev) 配置
- [ ] 编译成功，没有错误
- [ ] 可执行文件在正确的输出目录
- [ ] 游戏能正常启动

启动游戏后：

- [ ] 按 F12 能打开调试菜单
- [ ] 导航到 Systems → Workspots
- [ ] 看到 "Show ImGui Panel" 选项
- [ ] 勾选后屏幕出现 Workspot System 窗口

---

## 📞 故障排除

### 问题1：按F12没反应

**可能原因**：
1. 当前是Final配置
2. 调试菜单热键被改了

**解决方案**：
1. 检查构建配置
2. 尝试按 **~** 或 **`** 键
3. 查看源码中的热键配置（可能在inputManager或debugMenu相关代码）

### 问题2：有调试菜单，但没有Workspots选项

**可能原因**：
1. WORKSPOT_DEBUG_ENABLED 未定义
2. ImGui代码被条件编译剔除了

**解决方案**：
1. 检查 `workspotDebugger.h` 中的宏定义
2. 确认 `workspotSystem.cpp` 中的 ShowImGuiDebugPanel() 被编译进去了
3. 在代码中搜索 `#ifdef WORKSPOT_DEBUG_ENABLED`

### 问题3：ImGui窗口不显示

**可能原因**：
1. ImGui系统未初始化
2. 渲染层问题

**解决方案**：
1. 检查其他ImGui面板是否正常（如其他System的面板）
2. 查看日志中是否有ImGui相关错误
3. 确认 `g_debugSettings.showWorkspotImGui` 标志被正确设置

---

## 🎯 总结

**您的情况**（有源码可编译）：

1. **最快方式**：
   - 确认当前配置不是Final
   - 直接运行，按F12启用ImGui
   - **无需重新编译**

2. **如果是Final配置**：
   - 切换到Debug或Release (Dev)
   - 重新编译（一次性）
   - 之后就可以随时使用

3. **高级定制**：
   - 修改 ShowImGuiDebugPanel() 添加自定义信息
   - 添加镜头检测逻辑
   - 自动高亮镜头中的NPC

**核心优势**：有源码意味着您可以：
- 添加任何您需要的调试信息
- 不受现有调试工具限制
- 可以直接在代码中打断点调试

现在您应该能够轻松启用ImGui面板了！
