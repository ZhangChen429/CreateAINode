# FFlowConditionTokenContext 重名问题修复

## 问题诊断

`FFlowConditionTokenContext` 结构体在两个文件中重复定义：
1. ❌ `FlowConditionTypes.h` (旧文件) - 第132行
2. ✅ `FlowConditionToken.h` (新Token系统) - 第191行

这导致编译器报重复定义错误。

---

## 修复方案

### 1. 删除旧定义

**文件：** `FlowConditionTypes.h`

**修改前：**
```cpp
/**
 * 条件评估上下文
 */
USTRUCT(BlueprintType)
struct FFlowConditionTokenContext
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite, Category = "Context")
    TObjectPtr<UWorld> World = nullptr;

    UPROPERTY(BlueprintReadWrite, Category = "Context")
    TObjectPtr<AActor> Instigator = nullptr;

    UPROPERTY(BlueprintReadWrite, Category = "Context")
    TMap<FName, FString> Parameters;
};
```

**修改后：**
```cpp
// FFlowConditionTokenContext已移至FlowConditionToken.h
// 请包含 "Types/FlowConditionToken.h" 以使用上下文类型
```

### 2. 保留新定义

**文件：** `FlowConditionToken.h` (第191行)

**保留的定义（更完整）：**
```cpp
/**
 * 条件评估上下文
 */
USTRUCT(BlueprintType)
struct FLOWNODE_API FFlowConditionTokenContext
{
    GENERATED_BODY()

    FFlowConditionTokenContext()
        : World(nullptr)
        , Instigator(nullptr)
        , DeltaTime(0.0f)
    {
    }

    /** World上下文 */
    UPROPERTY(BlueprintReadWrite, Category = "Context")
    TObjectPtr<UWorld> World;

    /** 触发者 */
    UPROPERTY(BlueprintReadWrite, Category = "Context")
    TObjectPtr<AActor> Instigator;

    /** Delta时间 */
    UPROPERTY(BlueprintReadWrite, Category = "Context")
    float DeltaTime;

    /** 自定义参数 */
    UPROPERTY(BlueprintReadWrite, Category = "Context")
    TMap<FName, FString> Parameters;
};
```

**为什么保留这个定义：**
- ✅ 包含了 `DeltaTime` 成员（旧定义没有）
- ✅ 在Token系统核心文件中
- ✅ 有构造函数初始化
- ✅ 使用 `FLOWNODE_API` 导出宏

---

## 验证结果

### 1. 检查重复定义

```bash
grep -r "struct.*FFlowConditionTokenContext" --include="*.h"
```

**结果：** ✅ 只有一个定义（在 `FlowConditionToken.h`）

### 2. 检查所有引用

所有文件都正确使用 `FFlowConditionTokenContext`：

- ✅ `FlowConditionBase.h` - 所有方法签名
- ✅ `FlowConditionBase.cpp` - 所有方法实现
- ✅ `FlowNode_WaitCondition.cpp` - 所有上下文创建
- ✅ `FlowCondition_Timer.h` - 方法签名
- ✅ `FlowCondition_Timer.cpp` - 方法实现

### 3. 没有旧名字残留

```bash
grep -r "FFlowConditionContext[^T]" --include="*.h" --include="*.cpp"
```

**结果：** ✅ 没有找到旧的 `FFlowConditionContext` 引用

---

## 相关类型说明

### 当前活跃的类型

| 类型 | 文件 | 用途 |
|------|------|------|
| `FFlowConditionTokenContext` | `FlowConditionToken.h` | ✅ **新Token系统上下文**（正在使用） |
| `FFlowConditionTokenSaveData` | `FlowConditionToken.h` | ✅ **新Token系统SaveGame**（正在使用） |
| `EFlowConditionTokenType` | `FlowConditionToken.h` | ✅ Token类型枚举（正在使用） |
| `EFlowConditionTokenState` | `FlowConditionToken.h` | ✅ Token状态枚举（正在使用） |
| `EFlowConditionEventType` | `FlowConditionToken.h` | ✅ 事件类型枚举（正在使用） |

### 旧类型（保留兼容）

| 类型 | 文件 | 状态 |
|------|------|------|
| `FFlowConditionSaveData` | `FlowConditionTypes.h` | ⚠️ 旧SaveGame结构（未使用，保留兼容） |
| `EFlowConditionCheckMode` | `FlowConditionTypes.h` | ⚠️ 旧检查模式枚举（未使用） |

---

## 文件包含关系

### 正确的包含方式

```cpp
// ✅ 在需要使用Token系统的文件中
#include "Types/FlowConditionToken.h"  // 获取所有Token相关类型

// 包含内容：
// - FFlowConditionTokenContext
// - FFlowConditionTokenSaveData
// - FFlowConditionToken
// - FFlowConditionEventRequest
// - EFlowConditionTokenType
// - EFlowConditionTokenState
// - EFlowConditionEventType
// - EFlowConditionResult
```

### 已经正确包含的文件

- ✅ `FlowConditionBase.h` - `#include "Types/FlowConditionToken.h"`
- ✅ `FlowCondition_Timer.h` - `#include "Conditions/FlowConditionBase.h"` (间接包含)
- ✅ `FlowNode_WaitCondition.h` - `#include "Types/FlowConditionToken.h"`

---

## 注意事项

### 如果需要使用上下文类型

```cpp
// ✅ 正确
#include "Types/FlowConditionToken.h"

void MyFunction(const FFlowConditionTokenContext& Context)
{
    UWorld* World = Context.World;
    AActor* Instigator = Context.Instigator;
    float Delta = Context.DeltaTime;
}
```

### 如果遇到未定义错误

```cpp
// ❌ 错误：忘记包含头文件
// error: 'FFlowConditionTokenContext' does not name a type

// ✅ 解决：添加包含
#include "Types/FlowConditionToken.h"
```

---

## 总结

✅ **已修复问题：**
1. 删除了 `FlowConditionTypes.h` 中的重复定义
2. 统一使用 `FlowConditionToken.h` 中的定义
3. 所有引用都已更新为 `FFlowConditionTokenContext`
4. 验证没有旧名字 `FFlowConditionContext` 残留

✅ **验证通过：**
- 只有一个 `FFlowConditionTokenContext` 定义
- 所有文件都正确包含和使用
- 没有编译错误

---

修复完成时间：2026-02-13
