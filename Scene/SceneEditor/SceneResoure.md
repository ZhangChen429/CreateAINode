
# SceneEditorResource 数据框架结构（赛博朋克2077 .scenesolution 核心）
## SceneEditorResource 数据框架概览
`SceneEditorResource` 是 Cyberpunk 2077 场景解决方案（`.scenesolution` 文件）的核心编辑器资源类，它组织了整个场景的所有数据。

---

## 一、顶层结构
```
SceneEditorResource (.scenesolution 文件)
│
├─► m_sceneDescriptor        - 场景描述符（核心场景图数据）
├─► m_backendData             - 后端数据（图遍历和编译相关）
│
├─► Outline 数据（场景资源）
│   ├─► m_actors              - SceneActor[] 演员列表
│   ├─► m_props               - SceneProp[] 道具列表
│   ├─► m_vehicles            - SceneVehicle[] 载具列表
│   ├─► m_referencePoints     - SceneReferencePoint[] 参考点列表
│   └─► m_effects             - SceneEffect[] 特效列表
│
├─► 动画和工作点资源
│   ├─► m_ridAssocs           - RidAssoc[] RID动画关联（.re 文件导入数据）
│   ├─► m_workspots           - SceneWorkspot[] 工作点列表
│   └─► m_workspotInstances   - SceneWorkspotInstance[] 工作点实例
│
├─► 场景执行控制
│   ├─► m_interruptionScenarios - InterruptionScenario[] 中断场景
│   └─► m_eventExecutionTags    - EventExecutionTag[] 事件执行标签
│
├─► 场景元数据
│   ├─► m_solutionVariables   - 场景变量
│   ├─► m_screenplayData      - 剧本数据
│   ├─► m_previewScenariosData- 预览场景数据
│   ├─► m_version             - 场景版本号
│   ├─► m_mode                - 场景模式（Develop/PostRelease）
│   └─► m_sceneSolutionHash   - 场景哈希值
```

---

## 二、核心数据流：SceneDescriptor
`SceneDescriptor` 是场景图的核心容器，包含所有场景节点：
```cpp
class SceneDescriptor
{
    red::DynArray<THandle<NodeDescriptor>> m_nodes;  // 所有场景节点

    // ID 生成器
    Uint64 m_trackIdGenerator;         // 轨道ID生成器
    Uint64 m_eventIdGenerator;         // 事件ID生成器
    Uint64 m_eventsGroupIdGenerator;   // 事件组ID生成器

    // 节点映射表（优化查找）
    red::HashMap<Uint32, THandle<NodeDescriptor>> m_nodesMap;
    red::HashMap<Uint32, THandle<NodeDescriptor>> m_internalNodesMap;
};
```

### 节点类型（SceneGraphNodeType）
`SceneDescriptor` 中的节点有多种类型：
- Section - 章节节点（时间线上的主要分段）
- Quest - 任务节点（嵌入的任务逻辑块）
- Start - 起始节点
- End - 结束节点
- Choice - 选择节点
- 等等...

---

## 三、NodeDescriptor 结构（节点的通用数据）
每个节点（无论是 Section 还是 Quest）都继承自 `NodeDescriptor` 基类：
```cpp
class NodeDescriptor
{
    scn::NodeId m_id;                    // 节点唯一ID
    String m_caption;                    // 节点标题
    String m_comment;                    // 节点注释
    Float m_timelineLength;              // 时间线长度

    // 核心数据
    red::DynArray<THandle<EventDescriptor>> m_events;        // 事件/Clip数组 ⭐
    red::DynArray<THandle<TrackDescriptor>> m_tracks;        // 轨道数组 ⭐
    red::DynArray<THandle<SocketDescriptor>> m_sockets;      // 图节点插槽
    red::DynArray<THandle<EventsGroupDescriptor>> m_eventsGroups; // 事件组

    // ID池
    ExternalNodeIdsPool m_externalNodeIdsPool;  // 外部节点ID池
};
```

### 关键数据：Events（EventDescriptor[]）
这是动画 Clip 的来源！每个 `NodeDescriptor` 包含一个 `m_events` 数组，存储该节点所有的事件（Clip）：
```cpp
// 从节点获取所有事件
const red::DynArray<THandle<EventDescriptor>>& GetEvents() const;
```

### TrackDescriptor 结构（轨道）
轨道用于组织和分类事件：
```cpp
class TrackDescriptor
{
    TrackStamp m_stamp;          // 轨道标识（ID、父ID、类型）
    String m_name;               // 轨道名称
    red::RUID m_subjectId;       // 关联的演员/道具ID

    // 轨道状态
    Bool m_isMute;               // 是否静音
    Bool m_isPinned;             // 是否固定
    Bool m_isVisible;            // 是否可见
    Bool m_isExpanded;           // 是否展开
    Bool m_isLocked;             // 是否锁定
};
```

### 轨道类型（TrackType）
- Normal - 通用轨道
- Dialog - 对话轨道
- Logic - 逻辑轨道
- SFX/VFX - 音效/视效轨道
- Camera/CameraAnimation - 摄像机轨道
- ActorTrackIdle/Animation/Facial/LookAt - 演员轨道
- PropTrackAnimation/Placement - 道具轨道
- VehicleTrackAnimation - 载具轨道

---

## 四、SceneBackendData 结构（后端编译数据）
```cpp
class SceneBackendData
{
    THandle<scnb::graph::SceneGraphDescriptor> m_graphDescriptor;
    THandle<IGraphDescriptorGlobalData> m_graphDescriptorGlobalData;
};
```

`SceneGraphDescriptor` 提供了场景图的遍历接口：
```cpp
// 核心遍历方法
m_graphDescriptor->IterateNodes([&](tools::IGraphNodeDescriptor* baseGraphnode) {
    // 访问每个场景图节点
    auto graphnode = Cast<scnb::graph::SceneGraphNodeDescriptor>(baseGraphnode);

    // 获取节点的事件列表
    const auto& events = graphnode->GetNode().GetEvents();

    // 处理每个事件...
});
```
这个遍历方法是 `SceneSolutionResourceMetadataWriter` 用来收集所有动画 Clip 的核心机制。

---

## 五、EventDescriptor 层次结构（动画 Clip）
`EventDescriptor` 是所有事件/Clip 的基类，具体的动画事件包括：

### 1. PlaySkAnimDescriptor（骨骼动画）
```cpp
class PlaySkAnimDescriptor : public EventDescriptor
{
    CName m_animName;  // 动画名称 ⭐
    // 其他属性...
};
```

### 2. ChangeIdleAnimDescriptor（待机动画切换）
```cpp
class ChangeIdleAnimDescriptor : public EventDescriptor
{
    CName m_animName;              // 主动画名称 ⭐
    CName m_idleAnimName;          // 待机动画名称 ⭐
    CName m_addIdleAnimName;       // 附加待机动画 ⭐
    CName m_customTransitionAnim;  // 自定义过渡动画 ⭐
    // 其他属性...
};
```

### 3. ChangeWorkEvent（工作动画切换）
```cpp
class ChangeWorkEvent : public EventDescriptor
{
    // 通过方法获取动画信息
    AnimInfo GetTransitionAnimInfo();  // 过渡动画信息 ⭐
    AnimInfo GetGameplayAnimInfo();    // 游戏玩法动画信息 ⭐
};

struct AnimInfo
{
    CName m_animName;  // 动画名称
    // ...
};
```

### 4. StopWorkEvent（停止工作事件）
```cpp
class StopWorkEvent : public EventDescriptor
{
    AnimInfo GetAnimationInfo();      // 基础动画信息 ⭐
    AnimInfo GetGameplayAnimInfo();   // 游戏玩法动画信息 ⭐
};
```

---

## 六、数据访问路径总结
从代码 `sceneSolutionResourceMetadataWriter.h:57` 来看，访问路径如下：
```cpp
// 1. 加载 SceneEditorResource
THandle<tools::SceneEditorResource> sceneEditorResource = ...;

// 2. 获取后端数据
const THandle<tools::SceneBackendData>& sceneBackendData =
    sceneEditorResource->GetBackendData();

// 3. 初始化并遍历场景图
sceneBackendData->m_graphDescriptor->IterateNodes([&](tools::IGraphNodeDescriptor* node) {
    // 4. 获取场景图节点
    auto graphnode = Cast<scnb::graph::SceneGraphNodeDescriptor>(node);

    // 5. 获取节点的事件数组（Clip 数组）
    const red::DynArray<THandle<tools::EventDescriptor>>& events =
        graphnode->GetNode().GetEvents();

    // 6. 遍历每个事件，根据类型提取动画名称
    for (auto& event : events) {
        if (event->IsA<tools::PlaySkAnimDescriptor>()) {
            auto anim = Cast<tools::PlaySkAnimDescriptor>(event);
            CName animName = anim->m_animName;  // 提取动画名称 ✅
        }
        // ... 其他事件类型
    }
});
```

---

## 七、关键设计特点
1. **分层设计**：
   - Editor层（SceneEditorResource）- 存储可编辑数据
   - Backend层（SceneBackendData）- 存储编译和运行时数据

2. **节点-事件-轨道三层结构**：
```
NodeDescriptor (节点)
├─► Events[] (事件/Clip)
└─► Tracks[] (轨道，用于组织事件)
```

3. **多种资源集合**：
   - Outline 资源（Actors, Props, Vehicles）
   - 动画资源（RidAssocs, Workspots）
   - 执行控制（InterruptionScenarios, EventExecutionTags）

4. **ID管理系统**：
   - TrackID、EventID、EventsGroupID 都有专门的生成器
   - 支持外部节点 ID 池（ExternalNodeIdsPool）

> 这个框架支持了 Cyberpunk 2077 复杂的场景系统，包括对话、动画、摄像机、特效等所有元素的编排和管理。