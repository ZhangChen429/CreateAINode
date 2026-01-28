# ImGui调试面板 - 快速使用指南

## 适用条件
- ✅ 游戏版本：开发版（非Final）
- ✅ 无需MOD
- ✅ 无需重新编译

## 操作步骤

### 1. 打开调试菜单
按键：**F12** 或 **~**（根据配置不同）

如果没有反应，说明您是Final版本，跳转到【方案3：AI日志】

### 2. 启用ImGui面板
导航到：
```
Debug Menu
└─ Systems
   └─ Workspots
      └─ Show ImGui Panel  ← 勾选此项
```

### 3. 查看面板内容

屏幕上会出现实时更新的窗口：

```
┌─────────────────────────────────────────────────────────┐
│ Workspot System                                         │
├─────────────────────────────────────────────────────────┤
│ Workspot instances count = 15                           │
│ Workspot resources count = 8                            │
│ Final animsets count = 5                                │
│                                                          │
│ ▼ Workspots                                             │
│ ┌────┬──────────────────────────┬──────────┬─────────┐ │
│ │ ID │ Name                      │ Instances│ Body    │ │
│ ├────┼──────────────────────────┼──────────┼─────────┤ │
│ │ 1  │ restaurant_chair_01       │ 3        │ man_avg │ │
│ │ 2  │ bench_sit                 │ 2        │ woman   │ │
│ │ 3  │ atm_machine               │ 1        │ man_big │ │
│ └────┴──────────────────────────┴──────────┴─────────┘ │
└─────────────────────────────────────────────────────────┘
```

### 4. 找到镜头中的NPC

**方法A：根据位置判断**
1. 记住NPC所在的场景位置（如：餐厅、ATM机旁）
2. 在面板中查找对应的Workspot名称
3. 点击展开该Workspot条目
4. 查看 `Instances count`（实例数量）
5. 如果有多个实例，根据位置信息判断

**方法B：结合3D可视化**
1. 同时启用：`Debug Menu → Filters → AI → Workspots → Spots`
2. 将镜头对准NPC
3. 看到蓝色金字塔标记
4. 在ImGui面板中找到对应的Workspot

### 5. 复制Workspot路径

在面板中：
1. 右键点击Workspot路径
2. 选择 **Copy path**
3. 得到完整路径：`workspots/restaurant_chair_01.workspot`

## 实战示例

**场景**：餐厅中有3个NPC坐在椅子上，想知道中间那个用的哪个Workspot

**步骤**：

1. 按F12打开调试菜单
2. 启用 `Systems → Workspots → Show ImGui Panel`
3. 看到面板显示：
   ```
   1  workspots/restaurant_chair_01.workspot  [3 instances]
   ```
4. 展开该条目：
   ```
   ▼ workspots/restaurant_chair_01.workspot
     Instance #1: EntityID: 0xABC123, Position: (100.5, 200.3, 45.0)
     Instance #2: EntityID: 0xDEF456, Position: (102.1, 200.3, 45.0) ← 中间位置
     Instance #3: EntityID: 0x789GHI, Position: (103.7, 200.3, 45.0)
   ```
5. 确认：中间NPC的EntityID是 `0xDEF456`，使用 `restaurant_chair_01.workspot`

## 优势
- ✅ 显示所有活跃的Workspot实例
- ✅ 显示EntityID和位置信息
- ✅ 可以右键复制路径
- ✅ 实时更新
- ✅ 无需编写任何代码

## 注意事项
- ⚠️ 只在开发版本可用
- ⚠️ 显示所有NPC，需要手动筛选
- ⚠️ 如果NPC太多，可能需要滚动查找
