# 通过.gamedef启动 - ImGui调试面板启用指南

## 关键信息

`.gamedef` 是RED Engine的项目配置文件，通过它启动意味着您使用的是CDPR内部开发工具。

---

## 方案1：通过启动参数启用调试功能（最简单）

### 启动时添加命令行参数

无论您如何启动游戏（通过编辑器还是直接运行），添加以下参数：

```bash
# 启用调试菜单
-debugmenu

# 启用所有调试功能
-debug

# 启用ImGui面板
-imgui

# 完整示例
Cyberpunk2077.exe -debugmenu -debug -imgui
```

### 可能的启动方式

**方式A：通过RED Engine编辑器**
```
工具栏 → Run → 添加启动参数
或
项目设置 → Launch Options → 添加参数
```

**方式B：直接运行可执行文件**
```
D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev\bin\x64.Debug\Cyberpunk2077.exe -debugmenu -imgui
```

**方式C：创建快捷方式**
```
右键 Cyberpunk2077.exe → 创建快捷方式
右键快捷方式 → 属性 → 目标
在目标路径后添加：-debugmenu -imgui
```

---

## 方案2：检查构建配置（确认Debug版本）

### 检查bin目录结构

您的bin目录结构：
```
D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev\bin\
├─ x64.Debug\         ← ✅ Debug版本（调试功能已编译）
├─ x64.Release\       ← Release Dev版本（可能有）
└─ x64.Final\         ← Final版本（如果有，调试功能被剔除）
```

**确认方法**：
```bash
# 查看bin目录
ls D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev\bin

# 如果看到 x64.Debug → 调试功能已启用
# 如果只有 x64.Final → 需要重新构建Debug版本
```

---

## 方案3：通过RED Engine构建系统重新编译

### 使用RED Build工具

如果您有RED Engine的构建工具（通常在 `redtoolkit` 或 `internal` 目录）：

```bash
# 构建Debug版本
cd D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev
.\redtoolkit\build.bat --config Debug --platform x64

# 或者使用内部构建脚本
.\internal\scripts\build.py --debug
```

### 通过.gamedef配置

虽然.gamedef是二进制文件，但可能有对应的编辑工具：

```
RED Editor → Project Settings
→ Build Configuration → 选择 Debug
→ Rebuild Project
```

---

## 方案4：运行时配置文件

### 查找配置文件

RED Engine通常会有运行时配置文件：

```
可能的位置：
D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev\bin\x64.Debug\config\
D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev\bin\x64.Debug\r6\config\
D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev\config\
```

**查找文件**：
- `debug.ini` 或 `debug.config`
- `engine.ini` 或 `engine.config`
- `user.ini` 或 `user.config`

**可能的配置项**：
```ini
[Debug]
EnableDebugMenu=true
EnableImGui=true
ShowWorkspotDebug=true

[Workspots]
ShowImGuiPanel=true
Show3DMarkers=true
```

---

## 方案5：游戏内启用（如果F12可用）

如果启动后按 **F12** 能打开调试菜单：

```
Debug Menu
├─ Systems
│  └─ Workspots
│     └─ Show ImGui Panel  ← 勾选
├─ Filters
│  └─ AI
│     └─ Workspots
│        └─ Spots  ← 勾选（3D可视化）
└─ Logging
   └─ AI Logs
      └─ ws  ← 启用（日志系统）
```

---

## 快速诊断流程

### 步骤1：确认当前构建类型

```bash
# 查看bin目录
cd D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev\bin
ls

# 查看结果
x64.Debug    → ✅ Debug版本，调试功能应该可用
x64.Release  → ⚠️ 可能是Dev版本，检查是否有调试功能
x64.Final    → ✗ 发布版本，调试功能被剔除
```

### 步骤2：运行游戏测试

```bash
# 进入Debug目录
cd x64.Debug

# 直接运行（添加调试参数）
./Cyberpunk2077.exe -debugmenu -imgui

# 或者通过启动脚本（如果有）
./launch_debug.bat
```

### 步骤3：验证调试功能

游戏启动后：
1. 按 **F12** 键
2. 如果出现调试菜单 → ✅ 成功
3. 导航到 `Systems → Workspots → Show ImGui Panel`
4. 屏幕应该出现 Workspot System 窗口

---

## 常见启动参数参考

基于RED Engine的常见调试参数：

```bash
# 调试相关
-debugmenu          # 启用F12调试菜单
-debug              # 启用所有调试功能
-imgui              # 启用ImGui界面
-console            # 启用控制台（~ 键）

# 性能分析
-profiler           # 启用性能分析器
-stats              # 显示统计信息

# 日志相关
-log                # 启用详细日志
-ailog              # 启用AI日志

# 渲染调试
-renderdebug        # 渲染调试模式
-wireframe          # 线框模式

# 组合使用（推荐）
Cyberpunk2077.exe -debugmenu -imgui -console -profiler
```

---

## 如果需要重新构建

### 使用RED Build System

```bash
cd D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev

# 方式1：命令行构建
.\build.bat Debug x64

# 方式2：通过Python脚本
python .\build.py --config=Debug --platform=x64

# 方式3：使用Make（如果支持）
make config=debug platform=x64
```

### 构建配置说明

```
Debug        → 完整调试信息，性能较慢
Release_Dev  → 优化性能，保留调试功能
Release      → 完全优化，部分调试功能
Final        → 发布版本，无调试功能
```

---

## RED Engine Editor集成

如果您使用RED Engine Editor：

### 通过编辑器启动

```
File → Open Project → 选择 .gamedef 文件
Project → Build Configuration → Debug
Run → Launch with Debugging (F5)
```

### 编辑器内调试设置

```
Tools → Options → Debug
├─ Enable Debug Menu: ✓
├─ Enable ImGui: ✓
├─ Show Workspot Debug: ✓
└─ Auto-load debug symbols: ✓
```

---

## 实战示例：我的典型工作流

```bash
# 1. 进入开发目录
cd D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev

# 2. 确认Debug版本存在
ls bin/x64.Debug

# 3. 启动游戏（带调试参数）
bin/x64.Debug/Cyberpunk2077.exe -debugmenu -imgui -console

# 4. 游戏内启用
#    按F12 → Systems → Workspots → Show ImGui Panel
```

---

## 故障排除

### 问题1：找不到Cyberpunk2077.exe

**位置**：
```
D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev\bin\x64.Debug\Cyberpunk2077.exe
或
D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev\bin\x64.Debug\r6\Cyberpunk2077.exe
```

### 问题2：启动参数不生效

**检查**：
1. 参数格式是否正确（前面有 `-`）
2. 是否有空格分隔多个参数
3. 路径中有空格需要加引号

**正确示例**：
```bash
"D:\AppSoft\Sy2077\2077\2077\CDPR2077\dev\bin\x64.Debug\Cyberpunk2077.exe" -debugmenu -imgui
```

### 问题3：按F12没反应

**可能原因**：
1. 不是Debug版本
2. 启动参数缺少 `-debugmenu`
3. 热键被改了

**解决**：
1. 确认运行的是 x64.Debug 目录下的exe
2. 添加 `-debugmenu` 参数
3. 尝试按 `~` 键打开控制台

---

## 推荐配置

### 日常开发调试

```bash
Cyberpunk2077.exe -debugmenu -imgui -console -log
```

### 性能测试

```bash
Cyberpunk2077.exe -debugmenu -profiler -stats
```

### Workspot专项调试

```bash
Cyberpunk2077.exe -debugmenu -imgui -ailog
```

启动后手动启用：
- ImGui面板：`F12 → Systems → Workspots → Show ImGui Panel`
- 3D可视化：`F12 → Filters → AI → Workspots → Spots`

---

## 总结

**对于通过.gamedef启动的项目**：

1. ✅ **最简单**：添加启动参数 `-debugmenu -imgui`
2. ✅ **确保使用** `x64.Debug` 版本
3. ✅ **游戏内启用**：F12 → Systems → Workspots

**您现在应该能够**：
- 直接运行Debug版本
- 添加必要的启动参数
- 在游戏内启用ImGui面板
- 无需重新编译

如果还是不行，可能需要检查构建脚本或RED Engine编辑器的项目设置。
