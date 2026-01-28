# WorkspotTree的GlobalProps（全局道具系统）

## 一、核心概念

### 1.1 什么是GlobalProps？

**GlobalProps（全局道具）** 是Workspot系统中用于**在特定骨骼位置放置物品/道具的机制**。

```cpp
// workspotResource.h: 313-321
struct WorkspotGlobalProp
{
    CName m_id = "[Enter Prop ID Here]";  // 道具唯一标识
    CName m_boneName = CName::NONE();      // 骨骼名称（挂载点）
    TResAsyncRef<ent::EntityTemplate> m_prop; // 道具实体模板
};
```

**关键特点**：
- 道具**绑定到骨骼**上（通过 `m_boneName`）
- 道具会在骨骼的3D空间位置生成
- 可以在Workspot执行期间动态显示/隐藏
- 主要用于手持物品、桌面物品等场景道具

---

## 二、数据结构详解

### 2.1 WorkspotGlobalProp 结构

```cpp
struct WorkspotGlobalProp
{
    CName m_id;        // 道具ID（如："coffee_cup", "phone", "book"）
    CName m_boneName;  // 骨骼名称（如："RightHand", "LeftHand"）
    TResAsyncRef<ent::EntityTemplate> m_prop;  // 道具资源路径
};
```

**字段说明**：

#### 1. m_id（道具标识）
```
用途：
- 唯一标识这个道具
- 在脚本或Entry中引用这个道具
- 用于道具的显示/隐藏控制

示例：
- "coffee_cup"  # 咖啡杯
- "phone_01"    # 手机
- "newspaper"   # 报纸
- "drink_glass" # 酒杯
```

#### 2. m_boneName（骨骼挂载点）
```
用途：
- 指定道具挂载在哪个骨骼上
- 道具的位置=骨骼的3D空间位置

常用骨骼名称：
- "RightHand"    # 右手
- "LeftHand"     # 左手
- "Head"         # 头部
- "Hips"         # 臀部
- "Spine"        # 脊柱
```

#### 3. m_prop（道具资源）
```
用途：
- 指向道具的实体模板（EntityTemplate）
- 定义道具的3D模型、材质、碰撞等

示例路径：
base\props\food\coffee_cup_01.ent
base\props\electronics\phone_modern_01.ent
base\props\furniture\table_items\newspaper.ent
```

### 2.2 在WorkspotTree中的位置

```cpp
// workspotResource.h: 346
class WorkspotTree
{
    TResAsyncRef<anim::Rig> m_workspotRig;  // ← 全局骨骼Rig

    red::DynArray<WorkspotGlobalProp> m_globalProps;  // ← GlobalProps数组

    // ...
};
```

**关系**：
- `m_workspotRig`：提供骨骼结构
- `m_globalProps`：定义在哪些骨骼上放置哪些道具
- 道具位置 = 骨骼在动画中的位置

---

## 三、工作原理

### 3.1 道具生成流程

```
1. Workspot启动
   └─ 读取 m_globalProps 列表

2. 加载Rig骨骼
   └─ 从 m_workspotRig 加载骨骼数据

3. 遍历每个GlobalProp
   ├─ 读取 m_boneName（如"RightHand"）
   ├─ 在Rig中查找对应骨骼
   ├─ 获取骨骼的世界空间位置
   └─ 在该位置生成道具实体（m_prop）

4. 动画播放时
   └─ 骨骼移动 → 道具跟随移动
```

### 3.2 代码实现

**Setup阶段**：
```cpp
// workspotGlobalItem.cpp: 24-35
void WorkspotGlobalItemManager::Setup(const SetupContext& setupContext)
{
    m_transform = setupContext.m_transform;
    m_rigAsyncRef = setupContext.m_rig;  // ← 保存骨骼Rig
    m_props.Reserve(setupContext.m_globalProps.Size());

    for (auto& prop : setupContext.m_globalProps)  // ← 遍历所有GlobalProps
    {
        if(prop.m_prop.IsValid())
        {
            m_props.PushBack(WorkspotGlobalItemInstance(prop));
        }
    }
}
```

**获取道具位置**：
```cpp
// workspotGlobalItem.cpp: 72
WorldTransform transform = prop.GetPosition(
    m_transform,      // Workspot的世界变换
    m_rigResource     // 骨骼资源
);

// 内部逻辑（伪代码）：
GetPosition(parentTransform, rig)
{
    // 1. 在Rig中查找骨骼
    Bone bone = rig.FindBone(m_boneName);

    // 2. 获取骨骼的本地位置
    Vector3 boneLocalPos = bone.GetPosition();

    // 3. 转换到世界空间
    WorldTransform worldPos = parentTransform * boneLocalPos;

    return worldPos;
}
```

---

## 四、编辑器配置

### 4.1 在编辑器中设置GlobalProps

**位置**：
```
Properties 面板 → Global 标签 → Global Props Properties
```

**界面示例**：
```
┌──────────────────────────────────────────────────┐
│ Global Props Properties                          │
├──────────────────────────────────────────────────┤
│ Workspot Rig                                     │
│ ┌──────────────────────────────────────────────┐│
│ │ base\characters\entities\citizen\            ││
│ │   _male_average.rig                          ││
│ └──────────────────────────────────────────────┘│
│                                                  │
│ Global Props   [+] [-]                           │
│ ┌──────────────────────────────────────────────┐│
│ │ [0] ▼                                        ││
│ │   m_id       = "coffee_cup"                  ││
│ │   m_boneName = "RightHand"                   ││
│ │   m_prop     = base\props\food\cup_01.ent    ││
│ │                                              ││
│ │ [1] ▼                                        ││
│ │   m_id       = "newspaper"                   ││
│ │   m_boneName = "LeftHand"                    ││
│ │   m_prop     = base\props\paper\news_01.ent  ││
│ └──────────────────────────────────────────────┘│
└──────────────────────────────────────────────────┘
```

### 4.2 添加新道具的步骤

```
1. 点击 Global Props 右侧的 [+] 按钮
   → 添加新的GlobalProp条目

2. 设置 m_id（道具ID）
   输入唯一标识符，如："drink_glass"

3. 设置 m_boneName（骨骼名称）
   ├─ 点击下拉菜单
   ├─ 从可用骨骼列表中选择（自动填充）
   └─ 或手动输入骨骼名称

4. 设置 m_prop（道具资源）
   ├─ 点击 [...] 浏览按钮
   ├─ 选择道具的EntityTemplate资源
   └─ 路径示例：base\props\food\drink_glass_01.ent

5. 保存Workspot资源
```

### 4.3 可用骨骼列表

编辑器会自动从 `m_workspotRig` 读取可用骨骼：

```cpp
// workspotResource.cpp: 459-465
void WorkspotTree::EDITOR_UpdateGlobalItemSlots()
{
    if(m_workspotRig.Load())
    {
        auto rig = m_workspotRig.Get();
        if (rig)
        {
            m_availableRigSlots.Reserve(rig->NumBones());
            m_availableRigSlots.PushBack(rig->GetBoneNames());

            // 通知属性编辑器更新
            NotifyPropertyChanged("globalProps");
        }
    }
}
```

**结果**：
```
m_boneName 下拉菜单会显示所有可用骨骼：
- Hips
- Spine
- Spine1
- Spine2
- Spine3
- Neck
- Head
- LeftShoulder
- LeftArm
- LeftForeArm
- LeftHand
- RightShoulder
- RightArm
- RightForeArm
- RightHand
- LeftUpLeg
- LeftLeg
- ...
```

---

## 五、实际应用场景

### 场景1：NPC喝咖啡

**Workspot配置**：
```
Global Props:
[0]
  m_id       = "coffee_cup"
  m_boneName = "RightHand"
  m_prop     = base\props\food\coffee_cup_latte_01.ent
```

**Entry序列**：
```
Sequence: drink_coffee
├─ Entry anim: sit_table_idle
├─ Anim: sit_table_grab_cup        # 伸手拿杯子
│  └─ ItemAction: ShowProp("coffee_cup")  # 显示咖啡杯
├─ Anim: sit_table_drink_coffee    # 喝咖啡
├─ Anim: sit_table_put_down_cup    # 放下杯子
│  └─ ItemAction: HideProp("coffee_cup")  # 隐藏咖啡杯
└─ Exit anim: sit_table_to_stand
```

**效果**：
1. NPC坐下（咖啡杯不可见）
2. 动画播放"grab_cup"→ 咖啡杯出现在右手
3. 咖啡杯跟随右手骨骼移动
4. 动画播放"put_down_cup"→ 咖啡杯消失

### 场景2：阅读报纸

**Global Props配置**：
```
[0]
  m_id       = "newspaper"
  m_boneName = "LeftHand"
  m_prop     = base\props\paper\newspaper_folded_01.ent

[1]
  m_id       = "coffee_mug"
  m_boneName = "RightHand"
  m_prop     = base\props\food\mug_ceramic_01.ent
```

**Entry序列**：
```
Sequence: morning_routine
├─ Anim: sit_bench_idle
├─ Anim: sit_bench_grab_paper
│  └─ ItemAction: ShowProp("newspaper")
├─ Anim: sit_bench_read_newspaper      # 读报纸（左手）
├─ Anim: sit_bench_grab_coffee
│  └─ ItemAction: ShowProp("coffee_mug")
├─ Anim: sit_bench_drink_and_read      # 同时喝咖啡和读报纸
└─ ...
```

**效果**：
- 左手：报纸跟随LeftHand骨骼
- 右手：咖啡杯跟随RightHand骨骼
- 双手可以同时持有不同道具

### 场景3：手机通话

**Global Props配置**：
```
[0]
  m_id       = "smartphone"
  m_boneName = "RightHand"
  m_prop     = base\props\electronics\phone_modern_01.ent
```

**Entry序列**：
```
Sequence: phone_call
├─ Anim: stand_idle
├─ Anim: stand_pull_out_phone
│  └─ ItemAction: ShowProp("smartphone")
├─ Anim: stand_phone_to_ear         # 手机贴到耳朵
├─ Anim: stand_talking_on_phone     # 通话循环
├─ Anim: stand_phone_from_ear
├─ Anim: stand_put_away_phone
│  └─ ItemAction: HideProp("smartphone")
└─ Anim: stand_idle
```

---

## 六、道具控制

### 6.1 显示/隐藏道具

**方式1：通过ItemAction**

Entry可以包含ItemAction来控制道具：

```cpp
// Entry中的ItemAction列表
red::DynArray<THandle<IWorkspotItemAction>> m_itemActions;

// 显示道具
ShowPropAction {
    m_propId = "coffee_cup"
}

// 隐藏道具
HidePropAction {
    m_propId = "coffee_cup"
}
```

**方式2：通过API**

```cpp
// workspotGlobalItem.h: 87-89
void RegisterGlobalPropToSlot(const game::data::RecordID slotId, const Uint32 itemIndex);
void ResetProp(const Uint32 itemIndex);
void ResetProp(const game::data::RecordID slotId);
```

### 6.2 道具状态管理

```cpp
class WorkspotGlobalItemInstance
{
    CName m_id;  // 道具ID
    THandle<ent::Entity> m_loadedEntity;  // 生成的实体
    game::data::RecordID m_currentComponentSlot;  // 当前插槽

    void Reset(const Transform& parentNodePosition, const THandle<anim::Rig>& rig);
    void Clear();
};
```

**Reset（重置）**：
- 将道具重置到初始骨骼位置
- 不销毁实体

**Clear（清理）**：
- 销毁道具实体
- 释放资源

---

## 七、GlobalProps vs Entry的物品系统

### 7.1 两种物品系统对比

| 特性 | GlobalProps | Entry ItemActions |
|------|------------|------------------|
| **定义位置** | WorkspotTree全局 | 单个Entry |
| **生命周期** | 整个Workspot | Entry执行期间 |
| **挂载方式** | 绑定到骨骼 | 可以绑定到骨骼或组件插槽 |
| **用途** | 持久性道具（杯子、手机） | 临时道具、动态物品 |
| **性能** | 预加载，常驻内存 | 按需加载/卸载 |

### 7.2 使用建议

**使用GlobalProps的场景**：
```
✅ 手持道具（咖啡杯、手机、书籍）
✅ 固定道具（桌上的物品）
✅ 在多个Entry中复用的道具
✅ 需要精确跟随骨骼的道具
```

**使用Entry ItemActions的场景**：
```
✅ 一次性道具
✅ 复杂的物品交互
✅ 需要动态替换的物品
✅ 不需要绑定骨骼的物品
```

---

## 八、高级技巧

### 8.1 多个道具在同一骨骼

```
可以定义多个道具在同一骨骼：

[0] m_id = "cup_coffee",  m_boneName = "RightHand"
[1] m_id = "cup_tea",     m_boneName = "RightHand"
[2] m_id = "glass_water", m_boneName = "RightHand"

→ 通过ItemAction在不同时间显示不同道具
```

### 8.2 道具偏移调整

如果道具位置不准确，可以在EntityTemplate中调整：

```
打开道具的.ent文件
→ 调整根组件的Transform
→ 设置Position/Rotation偏移
```

### 8.3 调试道具位置

```cpp
// workspotGlobalItem.cpp: 187
void WorkspotGlobalItemManager::RenderDebug(rend::IDebugDrawer& dd)
{
    for (auto& prop : m_props)
    {
        const Vector3 propSlotPosition = prop.GetPosition(m_transform, m_rigResource).GetPosition().AsVector3();
        builder.SetColor(Color::GRAY());
        builder.AddWireSphere(propSlotPosition, .04f);  // 灰色球体标记道具位置
    }
}
```

**游戏中启用**：
```
Debug Menu → Filters → AI → Workspots → Show Global Items
```

---

## 九、常见问题

### 问题1：道具没有出现

**检查清单**：
- [ ] m_workspotRig是否正确设置？
- [ ] m_boneName是否在Rig中存在？
- [ ] m_prop资源路径是否正确？
- [ ] Entry中是否有ShowProp的ItemAction？

### 问题2：道具位置不对

**可能原因**：
1. 骨骼名称错误
2. 道具EntityTemplate的Transform偏移不正确
3. Rig和动画不匹配

**解决**：
1. 使用编辑器的骨骼可视化工具
2. 调整道具EntityTemplate的根变换
3. 确认动画使用正确的Rig

### 问题3：道具不跟随动画

**原因**：
- 道具没有绑定到正确的骨骼
- 骨骼名称拼写错误

**解决**：
```
1. 检查m_boneName的值
2. 在编辑器中从下拉列表选择骨骼
3. 不要手动输入骨骼名称（容易拼写错误）
```

---

## 十、总结

### 核心要点

1. **GlobalProps 是道具管理系统**
   - 在全局级别定义道具
   - 绑定到骨骼位置
   - 跟随动画移动

2. **三个关键字段**
   - `m_id`：道具标识符
   - `m_boneName`：骨骼挂载点
   - `m_prop`：道具资源

3. **与Rig紧密关联**
   - 依赖m_workspotRig提供骨骼
   - 道具位置=骨骼3D空间位置
   - 编辑器自动读取可用骨骼

4. **通过ItemAction控制**
   - ShowProp：显示道具
   - HideProp：隐藏道具
   - 可在Entry中动态切换

5. **常见应用场景**
   - 手持物品（杯子、手机、武器）
   - 桌面道具（书籍、餐具）
   - 可穿戴物品（帽子、眼镜）

### 与其他概念的区别

```
┌─────────────────────────────────────────┐
│ m_slotName（同步槽）                     │
│ - 用于Entry同步                          │
│ - 字符串标识符                           │
│ - 与道具系统无关                         │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ GlobalProps（全局道具）                  │
│ - 用于物品显示                           │
│ - 绑定到骨骼                            │
│ - 跟随动画移动                           │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ m_workspotRig（骨骼）                    │
│ - 定义骨骼结构                           │
│ - 提供骨骼位置                           │
│ - GlobalProps依赖它                      │
└─────────────────────────────────────────┘
```

现在您完全理解了WorkspotTree的GlobalProps系统！
