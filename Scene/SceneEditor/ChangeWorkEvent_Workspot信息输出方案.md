# ChangeWorkEvent Workspot 信息输出方案

## 一、需求说明

在 `SceneSolutionResourceMetadataWriter<Writer>::Write` 返回 `ChangeWorkAnimNames` 时，需要额外输出 `s_workPropertyName` 的值（即 WorkEntry 信息），包括：
- **Workspot ID** - 关联的 SceneWorkspot 的 ID
- **Sequence Entry ID** - Workspot 中的具体序列入口 ID
- **动画名称** - TransitionAnim 和 GameplayAnim

## 二、数据结构

### WorkEntry 结构

```cpp
namespace scnb
{
    struct WorkEntry
    {
        SceneWorkspotId m_workspotId;      // Workspot ID
        work::WorkEntryId m_sequenceEntryId; // 序列入口 ID
    };
}
```

### ChangeWorkEvent 的 Work 属性

```cpp
class ChangeWorkEvent
{
    WorkEntry m_work;  // s_workPropertyName 对应的值

    // 获取方法
    WorkEntry GetWorkEntry() const;
};
```

## 三、修改方案

### 方案 A：使用结构体存储（推荐）

创建一个新的数据结构来同时存储动画信息和 Workspot 信息：

```cpp
// 1. 在函数开始处定义新的数据结构
struct ChangeWorkAnimEntry
{
    CName animName;
    red::RUID workspotId;
    work::WorkEntryId sequenceEntryId;
    String animType;  // "transition" 或 "gameplay"
};

red::DynArray<ChangeWorkAnimEntry> changeWorkAnimEntries;
```

```cpp
// 2. 在收集动画时，同时记录 Workspot 信息
else if ( event->IsA<scnb::events::ChangeWorkEvent>() )
{
    THandle<scnb::events::ChangeWorkEvent> changeWorkDescriptor =
        Cast<scnb::events::ChangeWorkEvent>( event );

    // 获取 WorkEntry
    scnb::WorkEntry workEntry = changeWorkDescriptor->GetWorkEntry();

    // 过渡动画
    if ( changeWorkDescriptor->GetTransitionAnimInfo().m_animName != CName::NONE() )
    {
        CName animName = changeWorkDescriptor->GetTransitionAnimInfo().m_animName;
        AddAnimationToMap( animNames, animName );
        AddAnimationToMap( ChangeWorkAnimNames, animName );

        // 记录详细信息
        ChangeWorkAnimEntry entry;
        entry.animName = animName;
        entry.workspotId = workEntry.m_workspotId.GetValue();
        entry.sequenceEntryId = workEntry.m_sequenceEntryId;
        entry.animType = "transition";
        changeWorkAnimEntries.PushBack( entry );
    }

    // 游戏玩法动画
    if ( changeWorkDescriptor->GetGameplayAnimInfo().m_animName != CName::NONE() )
    {
        CName animName = changeWorkDescriptor->GetGameplayAnimInfo().m_animName;
        AddAnimationToMap( animNames, animName );
        AddAnimationToMap( ChangeWorkAnimNames, animName );

        // 记录详细信息
        ChangeWorkAnimEntry entry;
        entry.animName = animName;
        entry.workspotId = workEntry.m_workspotId.GetValue();
        entry.sequenceEntryId = workEntry.m_sequenceEntryId;
        entry.animType = "gameplay";
        changeWorkAnimEntries.PushBack( entry );
    }
}
```

```cpp
// 3. 输出 JSON 时包含 Workspot 信息
writer.String( "ChangeWorkEvent_animations" );
writer.StartObject();
for ( const auto& animName : ChangeWorkAnimNames )
{
    writer.String( animName.Key().AsChar() );
    writer.Int( animName.Value() );
}
writer.EndObject();

// 新增：输出详细的 ChangeWorkEvent 信息
writer.String( "ChangeWorkEvent_details" );
writer.StartArray();
for ( const auto& entry : changeWorkAnimEntries )
{
    writer.StartObject();

    writer.String( "animation_name" );
    writer.String( entry.animName.AsChar() );

    writer.String( "animation_type" );
    writer.String( entry.animType.AsChar() );

    writer.String( "workspot_id" );
    writer.String( String::Printf( "%llu", entry.workspotId.CalcHash() ).AsChar() );

    writer.String( "sequence_entry_id" );
    writer.Uint( entry.sequenceEntryId.m_id );

    writer.EndObject();
}
writer.EndArray();
```

### 输出示例

```json
{
    "scene_solutions": {
        "base\\quest\\testgym\\meethanako.scenesolution": {
            "ChangeWorkEvent_animations": {
                "sit_barstool_bar_lean270__lh_on_bar__02": 2,
                "stand__2h_front__01": 1
            },
            "ChangeWorkEvent_details": [
                {
                    "animation_name": "sit_barstool_bar_lean270__lh_on_bar__02",
                    "animation_type": "transition",
                    "workspot_id": "12345678901234",
                    "sequence_entry_id": 5
                },
                {
                    "animation_name": "sit_barstool_bar_lean270__lh_on_bar__02",
                    "animation_type": "gameplay",
                    "workspot_id": "12345678901234",
                    "sequence_entry_id": 5
                },
                {
                    "animation_name": "stand__2h_front__01",
                    "animation_type": "transition",
                    "workspot_id": "98765432109876",
                    "sequence_entry_id": 12
                }
            ]
        }
    }
}
```

## 四、方案 B：使用嵌套对象（可选）

如果希望将动画信息直接嵌套在一起：

```json
{
    "ChangeWorkEvent_animations_with_workspot": {
        "sit_barstool_bar_lean270__lh_on_bar__02": {
            "count": 2,
            "workspots": [
                {
                    "workspot_id": "12345678901234",
                    "sequence_entry_id": 5,
                    "animation_type": "transition"
                },
                {
                    "workspot_id": "12345678901234",
                    "sequence_entry_id": 5,
                    "animation_type": "gameplay"
                }
            ]
        }
    }
}
```

实现代码：

```cpp
// 使用 HashMap 存储动画和对应的 Workspot 列表
struct WorkspotInfo
{
    red::RUID workspotId;
    work::WorkEntryId sequenceEntryId;
    String animType;
};

red::HashMap<CName, red::DynArray<WorkspotInfo>> changeWorkAnimWithWorkspots;

// 收集时
WorkspotInfo info;
info.workspotId = workEntry.m_workspotId.GetValue();
info.sequenceEntryId = workEntry.m_sequenceEntryId;
info.animType = "transition";
changeWorkAnimWithWorkspots[animName].PushBack(info);

// 输出时
writer.String( "ChangeWorkEvent_animations_with_workspot" );
writer.StartObject();
for ( const auto& pair : changeWorkAnimWithWorkspots )
{
    writer.String( pair.Key().AsChar() );
    writer.StartObject();

    writer.String( "count" );
    writer.Int( pair.Value().Size() );

    writer.String( "workspots" );
    writer.StartArray();
    for ( const auto& info : pair.Value() )
    {
        writer.StartObject();
        writer.String( "workspot_id" );
        writer.String( String::Printf( "%llu", info.workspotId.CalcHash() ).AsChar() );
        writer.String( "sequence_entry_id" );
        writer.Uint( info.sequenceEntryId.m_id );
        writer.String( "animation_type" );
        writer.String( info.animType.AsChar() );
        writer.EndObject();
    }
    writer.EndArray();

    writer.EndObject();
}
writer.EndObject();
```

## 五、完整代码修改位置

### 1. 在 line 700 附近添加数据结构定义

```cpp
// Line 700-706 之后添加
struct ChangeWorkAnimEntry
{
    CName animName;
    red::RUID workspotId;
    work::WorkEntryId sequenceEntryId;
    String animType;
};
red::DynArray<ChangeWorkAnimEntry> changeWorkAnimEntries;

// 同理为 StopWorkEvent 也可以添加
struct StopWorkAnimEntry
{
    CName animName;
    red::RUID workspotId;
    work::WorkEntryId sequenceEntryId;
    String animType;
};
red::DynArray<StopWorkAnimEntry> stopWorkAnimEntries;
```

### 2. 修改 line 763-776 的 ChangeWorkEvent 处理

```cpp
else if ( event->IsA<scnb::events::ChangeWorkEvent>() )
{
    THandle<scnb::events::ChangeWorkEvent> changeWorkDescriptor =
        Cast<scnb::events::ChangeWorkEvent>( event );

    // 获取 WorkEntry 信息
    scnb::WorkEntry workEntry = changeWorkDescriptor->GetWorkEntry();

    // 过渡动画
    if ( changeWorkDescriptor->GetTransitionAnimInfo().m_animName != CName::NONE() )
    {
        CName animName = changeWorkDescriptor->GetTransitionAnimInfo().m_animName;
        AddAnimationToMap( animNames, animName );
        AddAnimationToMap( ChangeWorkAnimNames, animName );

        // 记录详细信息
        ChangeWorkAnimEntry entry;
        entry.animName = animName;
        entry.workspotId = workEntry.m_workspotId.GetValue();
        entry.sequenceEntryId = workEntry.m_sequenceEntryId;
        entry.animType = "transition";
        changeWorkAnimEntries.PushBack( entry );
    }

    // 游戏玩法动画
    if ( changeWorkDescriptor->GetGameplayAnimInfo().m_animName != CName::NONE() )
    {
        CName animName = changeWorkDescriptor->GetGameplayAnimInfo().m_animName;
        AddAnimationToMap( animNames, animName );
        AddAnimationToMap( ChangeWorkAnimNames, animName );

        // 记录详细信息
        ChangeWorkAnimEntry entry;
        entry.animName = animName;
        entry.workspotId = workEntry.m_workspotId.GetValue();
        entry.sequenceEntryId = workEntry.m_sequenceEntryId;
        entry.animType = "gameplay";
        changeWorkAnimEntries.PushBack( entry );
    }
}
```

### 3. 修改 line 912-928，在输出后添加详细信息

```cpp
// 原有的输出
writer.String( "ChangeWorkEvent_animations" );
writer.StartObject();
for ( const auto& animName : ChangeWorkAnimNames )
{
    writer.String( animName.Key().AsChar() );
    writer.Int( animName.Value() );
}
writer.EndObject();

// 新增：输出详细的 Workspot 信息
writer.String( "ChangeWorkEvent_details" );
writer.StartArray();
for ( const auto& entry : changeWorkAnimEntries )
{
    writer.StartObject();

    writer.String( "animation_name" );
    writer.String( entry.animName.AsChar() );

    writer.String( "animation_type" );
    writer.String( entry.animType.AsChar() );

    writer.String( "workspot_id" );
    writer.String( String::Printf( "%llu", entry.workspotId.CalcHash() ).AsChar() );

    writer.String( "sequence_entry_id" );
    writer.Uint( entry.sequenceEntryId.m_id );

    writer.EndObject();
}
writer.EndArray();
```

### 4. 同样处理 StopWorkEvent（可选）

StopWorkEvent 也有 WorkEntry，可以用相同的方式处理。

## 六、优势

1. **向后兼容** - 保留原有的 `ChangeWorkEvent_animations` 字段
2. **详细信息** - 新增 `ChangeWorkEvent_details` 提供完整的 Workspot 上下文
3. **可追溯性** - 可以追踪每个动画来自哪个 Workspot 和哪个 Entry
4. **易于分析** - 便于分析 Workspot 的使用情况和动画分布

## 七、使用场景

有了 Workspot 信息后，可以：
1. **追踪动画来源** - 知道某个动画是在哪个 Workspot 中定义的
2. **分析 Workspot 使用** - 统计哪些 Workspot 被使用最多
3. **优化资源** - 识别重复的 Workspot 定义
4. **调试问题** - 快速定位动画问题的源头

---

**创建日期**：2026-01-12
**作者**：Claude Code (Sonnet 4.5)
