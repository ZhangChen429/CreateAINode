# 让 AI 稳定输出指定格式 JSON 的渐进式 Prompt 策略
## 一、最小可验证的 JSON 结构
先从最简单的开始，逐步验证 LLM 的输出稳定性。

### Level 1: 最小原子结构
**目标 JSON 结构**：
```json
{
  "atoms": [
    {
      "id": "hanako_sit",
      "type": "MoveTo",
      "actor": "Hanako"
    },
    {
      "id": "v_sit",
      "type": "MoveTo",
      "actor": "V"
    }
  ],
  "relations": [
    "hanako_sit then v_sit"
  ]
}
```

**Prompt 模板**：
```
你是叙事编译器。将导演描述转换为 JSON 格式。

导演描述：
"Hanako 先坐下，V 跟着坐"

输出格式：
{
  "atoms": [
    {"id": "动作标识", "type": "动作类型", "actor": "角色名"}
  ],
  "relations": ["关系描述"]
}

可用的动作类型：
- Speak: 说话
- MoveTo: 移动/坐下/站起
- Gesture: 手势/动作
- LookAt: 看向
- Wait: 等待

关系语法：
- "A then B" 表示 A 完成后 B 开始
- "A while B" 表示 A 和 B 同时进行
- "after A and B, do C" 表示 A 和 B 都完成后执行 C

要求：
1. id 必须是小写字母和下划线组成
2. type 必须从可用类型中选择
3. 只输出 JSON，不要有其他文字

请输出 JSON：
```

### Level 2: 添加更多细节
**目标 JSON 结构**：
```json
{
  "atoms": [
    {
      "id": "hanako_sit",
      "type": "MoveTo",
      "actor": "Hanako",
      "target": "chair_hanako",
      "animation": "sit_formal"
    },
    {
      "id": "hanako_speak",
      "type": "Speak",
      "actor": "Hanako",
      "content": "请坐，V",
      "tone": "formal"
    }
  ],
  "relations": [
    "hanako_sit then hanako_speak"
  ],
  "tempo": "slow"
}
```

**Prompt 模板**：
```
你是叙事编译器。将导演描述转换为 JSON 格式。

导演描述：
"Hanako 坐下后，用正式的语气说'请坐，V'，整体节奏慢一点"

输出格式：
{
  "atoms": [
    {
      "id": "动作标识",
      "type": "动作类型",
      "actor": "角色名",
      "target": "目标对象（可选）",
      "content": "台词内容（可选）",
      "tone": "语气（可选）",
      "animation": "动画提示（可选）"
    }
  ],
  "relations": ["关系描述"],
  "tempo": "节奏（fast/medium/slow）"
}

可用的动作类型：
- Speak: 说话（需要 content 和 tone）
- MoveTo: 移动/坐下/站起（需要 target）
- Gesture: 手势/动作（需要 animation）
- LookAt: 看向（需要 target）
- Wait: 等待

关系语法：
- "A then B" 表示 A 完成后 B 开始
- "A while B" 表示 A 和 B 同时进行
- "after A and B, do C" 表示 A 和 B 都完成后执行 C

要求：
1. id 使用 "角色_动作" 格式，小写字母和下划线
2. type 必须从可用类型中选择
3. 根据动作类型填写必需的字段
4. 只输出 JSON，不要有其他文字

请输出 JSON：
```

## 二、使用 Few-Shot Examples 提高稳定性
**策略**：提供 3-5 个标准示例，让模型参考输出格式

```
你是叙事编译器。将导演描述转换为 JSON 格式。

参考示例：

示例 1：
输入："Hanako 站起来后退"
输出：
{
  "atoms": [
    {"id": "hanako_stand", "type": "MoveTo", "actor": "Hanako", "target": "standing"},
    {"id": "hanako_back", "type": "MoveTo", "actor": "Hanako", "target": "safe_distance"}
  ],
  "relations": ["hanako_stand then hanako_back"],
  "tempo": "fast"
}

示例 2：
输入："V 拔枪的同时，Hanako 看向 V"
输出：
{
  "atoms": [
    {"id": "v_draw_weapon", "type": "Gesture", "actor": "V", "animation": "draw_weapon"},
    {"id": "hanako_look", "type": "LookAt", "actor": "Hanako", "target": "V"}
  ],
  "relations": ["v_draw_weapon while hanako_look"],
  "tempo": "fast"
}

示例 3：
输入："等 Hanako 和 V 都坐好后，Hanako 开始说话"
输出：
{
  "atoms": [
    {"id": "hanako_sit", "type": "MoveTo", "actor": "Hanako", "target": "chair_hanako"},
    {"id": "v_sit", "type": "MoveTo", "actor": "V", "target": "chair_v"},
    {"id": "wait_both", "type": "Wait", "actor": "System"},
    {"id": "hanako_speak", "type": "Speak", "actor": "Hanako", "content": "我们开始吧", "tone": "formal"}
  ],
  "relations": [
    "hanako_sit then wait_both",
    "v_sit then wait_both",
    "wait_both then hanako_speak"
  ],
  "tempo": "slow"
}

现在处理：
输入："{user_input}"
输出：
```

## 三、使用 JSON Schema 约束（如果 API 支持）
### Claude API 的 Structured Output 实现
```python
import anthropic
import json

client = anthropic.Anthropic(api_key="your_key")

# 定义 JSON Schema
NARRATIVE_SCHEMA = {
    "type": "object",
    "required": ["atoms", "relations"],
    "properties": {
        "atoms": {
            "type": "array",
            "items": {
                "type": "object",
                "required": ["id", "type", "actor"],
                "properties": {
                    "id": {
                        "type": "string",
                        "pattern": "^[a-z_]+$"
                    },
                    "type": {
                        "enum": ["Speak", "MoveTo", "Gesture", "LookAt", "Wait"]
                    },
                    "actor": {"type": "string"},
                    "target": {"type": "string"},
                    "content": {"type": "string"},
                    "tone": {"type": "string"},
                    "animation": {"type": "string"}
                }
            }
        },
        "relations": {
            "type": "array",
            "items": {"type": "string"}
        },
        "tempo": {
            "enum": ["fast", "medium", "slow"]
        }
    }
}

# 调用 API
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=2048,
    messages=[{
        "role": "user",
        "content": f"""
你是叙事编译器。将导演描述转换为 JSON 格式。

导演描述：
"Hanako 请 V 坐下，同时自顾自喝一口酒"

可用的动作类型：
- Speak: 说话（需要 content 和 tone）
- MoveTo: 移动/坐下/站起（需要 target）
- Gesture: 手势/动作（需要 animation）
- LookAt: 看向（需要 target）
- Wait: 等待

关系语法：
- "A then B" 表示 A 完成后 B 开始
- "A while B" 表示 A 和 B 同时进行

要求：
1. id 使用 "角色_动作" 格式，小写字母和下划线
2. type 必须从可用类型中选择
3. 只输出 JSON
"""
    }],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "narrative_output",
            "schema": NARRATIVE_SCHEMA
        }
    }
)

result = json.loads(response.content[0].text)
print(json.dumps(result, indent=2, ensure_ascii=False))
```

## 四、渐进式验证策略
### 测试用例集
```python
TEST_CASES = [
    # Level 1: 简单顺序
    {
        "input": "Hanako 坐下",
        "expected_atoms": 1,
        "expected_relations": 0
    },

    # Level 2: 两个动作顺序
    {
        "input": "Hanako 先坐下，然后说话",
        "expected_atoms": 2,
        "expected_relations": 1
    },

    # Level 3: 并行动作
    {
        "input": "Hanako 说话的同时喝酒",
        "expected_atoms": 2,
        "expected_relations": 1,
        "expected_relation_type": "while"
    },

    # Level 4: 等待汇合
    {
        "input": "等 Hanako 和 V 都坐好后，开始对话",
        "expected_atoms": 4,  # hanako_sit, v_sit, wait, speak
        "expected_relations": 3
    },

    # Level 5: 复杂场景
    {
        "input": "Hanako 请 V 坐下，同时自顾自喝一口酒，紧张剧情",
        "expected_atoms": 4,
        "expected_tempo": "slow"
    }
]

def validate_output(output, test_case):
    """验证输出是否符合预期"""
    errors = []

    # 检查必需字段
    if "atoms" not in output:
        errors.append("缺少 atoms 字段")
    if "relations" not in output:
        errors.append("缺少 relations 字段")

    # 检查数量
    if len(output.get("atoms", [])) != test_case.get("expected_atoms"):
        errors.append(f"atoms 数量不符：期望 {test_case['expected_atoms']}，实际 {len(output['atoms'])}")

    # 检查 id 格式
    for atom in output.get("atoms", []):
        if not re.match(r'^[a-z_]+$', atom.get("id", "")):
            errors.append(f"id 格式错误：{atom.get('id')}")

    # 检查 type 是否合法
    valid_types = ["Speak", "MoveTo", "Gesture", "LookAt", "Wait"]
    for atom in output.get("atoms", []):
        if atom.get("type") not in valid_types:
            errors.append(f"type 不合法：{atom.get('type')}")

    return errors
```

## 五、实用的 Prompt 技巧
### 技巧 1: 明确输出格式
❌ 不好的 Prompt：
```
"把这段话转成 JSON"
```

✅ 好的 Prompt：
```
"只输出 JSON，不要有任何其他文字。格式如下：
{
  "atoms": [...],
  "relations": [...]
}"
```

### 技巧 2: 使用分隔符
```
你是叙事编译器。

===输入===
"Hanako 先坐下，V 跟着坐"

===输出格式===
{
  "atoms": [...],
  "relations": [...]
}

===要求===
1. id 使用小写字母和下划线
2. type 必须从可用类型中选择
3. 只输出 JSON

===开始输出===
```

### 技巧 3: 使用检查清单
```
在输出 JSON 之前，请检查：
□ 所有 id 都是小写字母和下划线
□ 所有 type 都在可用类型列表中
□ 所有必需字段都已填写
□ relations 使用了正确的语法

检查完成后，输出 JSON：
```

### 技巧 4: 使用角色设定
```
你是一个严格的 JSON 生成器。你的职责是：
1. 理解导演的叙事描述
2. 将其转换为标准的 JSON 格式
3. 绝不输出 JSON 之外的任何内容
4. 确保输出的 JSON 可以被 JSON.parse() 解析

你的输出将直接被程序读取，任何格式错误都会导致程序崩溃。

现在开始工作：
```

## 六、完整的测试脚本
```python
import anthropic
import json
import re

def test_narrative_json_generation():
    client = anthropic.Anthropic(api_key="your_key")

    test_cases = [
        "Hanako 坐下",
        "Hanako 先坐下，然后说话",
        "Hanako 说话的同时喝酒",
        "等 Hanako 和 V 都坐好后，开始对话",
        "Hanako 请 V 坐下，同时自顾自喝一口酒，紧张剧情"
    ]

    prompt_template = """
你是叙事编译器。将导演描述转换为 JSON 格式。

参考示例：

示例 1：
输入："Hanako 站起来后退"
输出：
{
  "atoms": [
    {"id": "hanako_stand", "type": "MoveTo", "actor": "Hanako", "target": "standing"},
    {"id": "hanako_back", "type": "MoveTo", "actor": "Hanako", "target": "safe_distance"}
  ],
  "relations": ["hanako_stand then hanako_back"],
  "tempo": "fast"
}

示例 2：
输入："V 拔枪的同时，Hanako 看向 V"
输出：
{
  "atoms": [
    {"id": "v_draw_weapon", "type": "Gesture", "actor": "V", "animation": "draw_weapon"},
    {"id": "hanako_look", "type": "LookAt", "actor": "Hanako", "target": "V"}
  ],
  "relations": ["v_draw_weapon while hanako_look"],
  "tempo": "fast"
}

可用的动作类型：
- Speak: 说话（需要 content 和 tone）
- MoveTo: 移动/坐下/站起（需要 target）
- Gesture: 手势/动作（需要 animation）
- LookAt: 看向（需要 target）
- Wait: 等待

关系语法：
- "A then B" 表示 A 完成后 B 开始
- "A while B" 表示 A 和 B 同时进行
- "after A and B, do C" 表示 A 和 B 都完成后执行 C

要求：
1. id 使用 "角色_动作" 格式，小写字母和下划线
2. type 必须从可用类型中选择
3. 只输出 JSON，不要有其他文字

现在处理：
输入："{input}"
输出：
"""

    results = []
    for test_input in test_cases:
        print(f"\n测试: {test_input}")

        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=2048,
            messages=[{
                "role": "user",
                "content": prompt_template.format(input=test_input)
            }]
        )

        try:
            output = json.loads(response.content[0].text)
            print("✓ JSON 解析成功")
            print(json.dumps(output, indent=2, ensure_ascii=False))

            # 验证格式
            errors = validate_format(output)
            if errors:
                print("✗ 格式验证失败:")
                for error in errors:
                    print(f"  - {error}")
            else:
                print("✓ 格式验证通过")

            results.append({
                "input": test_input,
                "success": len(errors) == 0,
                "output": output
            })

        except json.JSONDecodeError as e:
            print(f"✗ JSON 解析失败: {e}")
            print(f"原始输出: {response.content[0].text}")
            results.append({
                "input": test_input,
                "success": False,
                "error": str(e)
            })

    # 统计结果
    success_count = sum(1 for r in results if r["success"])
    print(f"\n\n总结: {success_count}/{len(test_cases)} 测试通过")

    return results

def validate_format(output):
    """验证输出格式"""
    errors = []

    # 检查必需字段
    if "atoms" not in output:
        errors.append("缺少 atoms 字段")
        return errors

    if "relations" not in output:
        errors.append("缺少 relations 字段")

    # 检查 atoms
    valid_types = ["Speak", "MoveTo", "Gesture", "LookAt", "Wait"]
    for i, atom in enumerate(output["atoms"]):
        # 检查必需字段
        if "id" not in atom:
            errors.append(f"atoms[{i}] 缺少 id")
        elif not re.match(r'^[a-z_]+$', atom["id"]):
            errors.append(f"atoms[{i}] id 格式错误: {atom['id']}")

        if "type" not in atom:
            errors.append(f"atoms[{i}] 缺少 type")
        elif atom["type"] not in valid_types:
            errors.append(f"atoms[{i}] type 不合法: {atom['type']}")

        if "actor" not in atom:
            errors.append(f"atoms[{i}] 缺少 actor")

    return errors

if __name__ == "__main__":
    test_narrative_json_generation()
```

## 七、最终推荐的 Prompt 模板
```
你是叙事编译器。你的任务是将导演的自然语言描述转换为标准的 JSON 格式。

===可用的动作类型===
- Speak: 说话（需要 content 和 tone）
- MoveTo: 移动/坐下/站起（需要 target）
- Gesture: 手势/动作（需要 animation）
- LookAt: 看向（需要 target）
- Wait: 等待

===关系语法===
- "A then B" 表示 A 完成后 B 开始
- "A while B" 表示 A 和 B 同时进行
- "after A and B, do C" 表示 A 和 B 都完成后执行 C

===参考示例===
输入："Hanako 站起来后退"
输出：
{
  "atoms": [
    {"id": "hanako_stand", "type": "MoveTo", "actor": "Hanako", "target": "standing"},
    {"id": "hanako_back", "type": "MoveTo", "actor": "Hanako", "target": "safe_distance"}
  ],
  "relations": ["hanako_stand then hanako_back"],
  "tempo": "fast"
}

===输出要求===
1. id 使用 "角色_动作" 格式，只能包含小写字母和下划线
2. type 必须从可用类型中选择
3. 根据动作类型填写必需的字段
4. 只输出 JSON，不要有任何其他文字
5. 确保 JSON 格式正确，可以被解析

===现在处理===
输入："{user_input}"
输出：
```

## 方案优势
1. **渐进式** - 从简单到复杂，逐步验证
2. **可测试** - 有明确的测试用例和验证逻辑
3. **可调试** - 每个失败的案例都能定位问题
4. **可扩展** - 可以轻松添加新的动作类型和关系语法

### 总结
1. 让AI稳定输出JSON的核心是**分层约束**（从最小结构到完整结构）+ **明确规则**（字段格式、必填项、取值范围）；
2. 提升稳定性的关键手段包括：Few-Shot示例、JSON Schema强制约束、格式化的Prompt（分隔符、检查清单）；
3. 落地时需配套**验证脚本**，通过测试用例集持续验证输出的正确性，确保JSON可解析、符合业务规则。