# Day 3：Prompt 工程——从基础到进阶

> 本课程系统学习 Prompt Engineering，掌握与 LLM 高效沟通的核心技能，从基础四层结构到结构化输出进阶。

## Part 1：Prompt 工程概念与四层结构

### 什么是 Prompt Engineering？

**Prompt（提示词）** 是与 LLM 沟通的方式。**Prompt Engineering（提示工程）** 是优化提示词的艺术。

```
同样的模型，不同的 Prompt → 效果天差地别
```

**为什么 Prompt 重要？**

| 模型能力 | Prompt 质量 | 效果 |
|----------|-------------|------|
| 强 | 差 | 浪费模型能力，输出不稳定 |
| 强 | 好 | 稳定、可控、高质量的输出 |
| 弱 | 好 | 最大限度发挥模型能力 |

### Prompt 的四层结构

可以把 Prompt 想象成一篇文章的结构：

| 层级 | 类比 | 作用 |
|------|------|------|
| **System 层** | 杂志的"办刊宗旨" | 设定整体风格、身份、边界 |
| **Context 层** | 文章的前言/背景 | 提供必要的背景信息 |
| **Example 层** | 作文的"范文" | 通过示例说明格式和风格 |
| **Question 层** | 作文的"题目" | 明确要回答什么 |

> 好的 Prompt 不是"一句话说明白"，而是"层层递进、结构清晰"。

### System Prompt 的"人设"作用

System Prompt 就像给模型设定了一个"人设"：

```python
# ❌ 没有 System Prompt → 模型是"普通人"
{"role": "user", "content": "解释量子计算"}

# ✅ 有 System Prompt → 模型是"物理学教授"
{"role": "system", "content": "你是一位物理学教授，用通俗易懂的语言解释物理概念"}
{"role": "user", "content": "解释量子计算"}
```

---

## Part 2：三条基本原则

### 原则一：清晰（Clear）

明确告诉模型你想要什么，不要让它猜。

```python
# ❌ 模糊
{"role": "user", "content": "写点代码"}

# ✅ 清晰
{"role": "user", "content": "用 Python 写一个函数，接受一个列表，返回列表中的偶数"}
```

### 原则二：具体（Specific）

提供足够的细节和约束条件。

```python
# ❌ 模糊
{"role": "user", "content": "解释一下Python"}

# ✅ 具体
{"role": "user", "content": "用通俗语言解释 Python 中列表和元组的区别，假设用户是零基础初学者，需要举例"}
```

### 原则三：上下文（Context）

提供必要的背景信息，帮助模型更好地理解需求。

```python
# ❌ 无上下文
{"role": "user", "content": "怎么优化？"}

# ✅ 有上下文
{"role": "user", "content": "我有一段 Python 代码运行很慢，使用了嵌套 for 循环处理 10万条数据，请问如何优化性能？"}
```

---

## Part 3：零样本与少样本提示

### Zero-shot（零样本）

直接提问，不给任何示例：

```python
response = client.chat.completions.create(
    model="glm-4-flash",
    messages=[{
        "role": "user",
        "content": "将以下句子分类为正面或负面：'今天天气真好'"
    }]
)
print(response.choices[0].message.content)
# 输出：正面
```

### Few-shot（少样本）

先给几个示例，再提问：

```python
response = client.chat.completions.create(
    model="glm-4-flash",
    messages=[{
        "role": "user",
        "content": """将以下句子分类为正面或负面。

示例：
'这家餐厅太好吃了' → 正面
'服务态度很差' → 负面

请分类：'价格很实惠'
"""
    }]
)
print(response.choices[0].message.content)
# 输出：正面
```

### Few-Shot 的"例题"作用

Few-Shot 就像做数学题前先看例题：

```
没有例题（Zero-shot）：
"3 + 5 = ?" → 模型随便猜答案

有例题（Few-shot）：
"2 + 3 = 5
4 + 6 = 10
7 + 8 = ?" → 模型学会了"加法模式"，给出正确答案 15
```

> 示例让模型理解"你想让它做什么"，而不是"你要它输出什么"。

### Zero-shot vs Few-shot 对比

| 场景 | Zero-shot | Few-shot |
|------|-----------|----------|
| 简单任务（常识问答） | 效果好 | 可用可不用 |
| 需要特定格式/风格 | 难控制 | 通过示例引导 |
| 复杂推理 | 效果不稳定 | 效果更稳定 |
| 训练示例难获取 | 更适合 | 需要准备示例 |

**实战建议：先试 Zero-shot，不满意再加 Few-shot 示例。**

---

## Part 4：结构化输出与 JSON 模式

### 为什么需要结构化输出？

自由文本格式不稳定，结构化输出 100% 可解析：

```python
# ❌ 不可靠：AI可能返回各种格式
response = llm.chat("返回一个用户的JSON：姓名、年龄、邮箱")
# 可能返回：
# {"name": "张三", "age": 25, "email": "zhang@example.com"}
# 或者：
# 姓名：张三\n年龄：25\n邮箱：zhang@example.com
```

### Function Calling 参数约束

```python
# 定义带完整约束的工具
tools = [{
    "type": "function",
    "function": {
        "name": "create_order",
        "description": "创建订单",
        "parameters": {
            "type": "object",
            "properties": {
                "product_id": {
                    "type": "string",
                    "description": "商品ID"
                },
                "quantity": {
                    "type": "integer",
                    "minimum": 1,
                    "maximum": 100,
                    "description": "购买数量"
                },
                "payment_method": {
                    "type": "string",
                    "enum": ["wechat", "alipay", "credit_card"],
                    "description": "支付方式"
                }
            },
            "required": ["product_id", "quantity", "payment_method"]
        }
    }
}]
```

### JSON 输出格式实战

```python
response = client.chat.completions.create(
    model="glm-4-flash",
    messages=[
        {"role": "system", "content": "你是一个数据提取助手。用户输入一段文字，你提取关键信息并以JSON格式返回。"},
        {"role": "user", "content": "提取以下文字中的人名、地点、机构：\n'张三在美国加州大学伯克利分校任教，与李四教授共同研究。'"}
    ],
    response_format={"type": "json_object"}
)
import json
result = json.loads(response.choices[0].message.content)
print(result)
# {'人名': ['张三', '李四'], '地点': ['美国加州'], '机构': ['伯克利分校']}
```

### JSON Schema 常用约束

| 关键字 | 说明 | 示例 |
|-------|------|------|
| `type` | 数据类型 | string, number, integer, boolean, array, object |
| `minimum/maximum` | 数值范围 | minimum: 0, maximum: 100 |
| `minLength/maxLength` | 字符串长度 | minLength: 1, maxLength: 255 |
| `pattern` | 正则约束 | pattern: "^[A-Z]{2}[0-9]{4}$" |
| `enum` | 枚举值 | enum: ["A", "B", "C"] |
| `required` | 必填字段 | required: ["name", "email"] |

### 表格输出

```python
response = client.chat.completions.create(
    model="glm-4-flash",
    messages=[{
        "role": "user",
        "content": """以表格形式列出常见编程语言的创始人、创立年份和擅长领域：

| 语言 | 创始人 | 年份 | 擅长领域 |"""
    }]
)
print(response.choices[0].message.content)
```

---

## Part 5：温度与最大词元数调优

### temperature 参数

temperature 控制输出的**随机性**：

```
temperature = 0.0  → 几乎确定输出，每次结果相同（适合翻译、代码）
temperature = 0.3  → 稍有变化（适合需要一致性的任务）
temperature = 0.7  → 平衡创造性与确定性（适合日常对话）
temperature = 1.0  → 高随机性，高创造性（适合写诗、头脑风暴）
```

```python
# 确定性强（适合编程）
response = client.chat.completions.create(
    model="glm-4-flash",
    messages=[{"role": "user", "content": "解释什么是递归"}],
    temperature=0.0,
    max_tokens=200
)

# 创意性强
response = client.chat.completions.create(
    model="glm-4-flash",
    messages=[{"role": "user", "content": "写一首关于AI的小诗"}],
    temperature=0.9,
    max_tokens=200
)
```

### 不同场景的推荐参数

| 场景 | temperature | max_tokens | 示例 |
|------|-------------|------------|------|
| 代码生成/翻译 | 0.0 - 0.2 | 根据需求 | 生成标准化代码 |
| 事实问答 | 0.0 - 0.3 | 200-500 | 解释概念 |
| 日常对话 | 0.6 - 0.8 | 300-800 | 聊天、问答 |
| 创意写作 | 0.8 - 1.0 | 500-2000 | 写故事、诗 |

---

## Part 6：Prompt 优化的常见坑

### 坑 1：Prompt 太长反而效果差

```python
# ❌ 塞入大量无关信息
{"role": "user", "content": "我叫张三，我今年30岁，我住在纽约，我喜欢跑步...请帮我写一封邮件"}

# ✅ 只给相关信息
{"role": "user", "content": "写一封商务邮件，主题是产品演示邀请，需要正式语气"}
```

### 坑 2：多个指令混在一起

```python
# ❌ 多个任务一次说
{"role": "user", "content": "帮我写一首诗关于春天，然后翻译成英文，再总结一下中心思想"}

# ✅ 一次一个任务
{"role": "user", "content": "帮我写一首诗关于春天"}
```

### 坑 3：示例和实际任务格式不一致

```python
# ❌ 示例格式和实际问题格式不一致
{"role": "user", "content": "示例：'很好' → positive\n示例：'很差' → negative\n请分类：今天的选择题全对了（）"}

# ✅ 示例和问题格式一致
{"role": "user", "content": "示例：'很好' → positive\n示例：'很差' → negative\n请分类：'今天的选择题全对了' → "}
```

---

## Part 7：思维链提示（CoT）简介

### 什么是思维链？

思维链（Chain-of-Thought，CoT）是让模型**展示推理过程**而不是直接给出答案的技巧。

```python
# ❌ 直接问
{"role": "user", "content": "小明有5个苹果，给了小红2个，又买了3个，现在有几个？"}

# ✅ 思维链
{"role": "user", "content": "小明有5个苹果，给了小红2个，又买了3个，现在有几个？\n让我们一步一步思考：\n1. 小明开始有5个苹果\n2. 给了小红2个，5-2=3\n3. 又买了3个，3+3=6\n所以答案是6"}
```

### CoT 的核心价值

| 对比 | 直接回答 | 思维链 |
|------|---------|--------|
| 数学推理 | 容易算错 | 步骤清晰，不易出错 |
| 逻辑推理 | 容易跳步 | 每一步可验证 |
| 复杂决策 | 结果不可控 | 过程可干预 |

**关键洞察：** CoT 不是给模型"答案"，而是给模型"思考方式"。

---

## 今日要点

- 理解了 Prompt 的四层结构：System、Context、Example、Question
- 掌握了 Prompt 的三大核心原则：清晰、具体、上下文
- 学会了 Zero-shot 和 Few-shot 提示技巧
- 掌握了结构化输出（JSON 模式、Function Calling 约束）
- 理解了 temperature 和 max_tokens 的作用及推荐值
- 熟悉了常见 Prompt 坑和思维链提示（CoT）

---

## 自测题

1. Prompt 的四层结构是什么？各自的作用是什么？
2. Zero-shot 和 Few-shot 提示有什么区别？各适合什么场景？
3. 如何让模型输出固定 JSON 格式？Function Calling 的参数约束怎么做？
4. 生成代码和写诗，分别应该用什么 temperature 值？
5. 什么是思维链（CoT）？和直接问答的核心区别是什么？

---

## 动手作业

1. 分别用 Zero-shot 和 Few-shot 方式做情感分类，对比效果差异
2. 让模型以 JSON 格式输出一个人的信息（姓名、年龄、职业、爱好）
3. 尝试不同 temperature 值（0.0 / 0.5 / 1.0）生成同一段创意写作，观察差异
4. 用思维链提示解决一道逻辑推理题，对比直接提问的效果