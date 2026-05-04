# Day 6：Function Calling 原理与实战

> 本课程掌握 Function Calling 核心能力，让大模型能够调用外部工具。

## Part 1：函数调用原理

### 什么是函数调用？

**函数调用（Function Calling）** 是大模型连接外部世界的桥梁。

```
没有函数调用：
用户 → 大模型 → 只能回答已有知识

有函数调用：
用户 → 大模型 → 判断需要工具 → 调用函数 → 获取实时数据 → 回答
```

### 大模型的能力边界

| 纯大模型能做到 | 大模型做不到（需要函数调用） |
|---|---|
| ✅ 回答常识问题 | ❌ 查实时天气 |
| ✅ 解释概念 | ❌ 查最新股票价格 |
| ✅ 写作、翻译 | ❌ 发送邮件 |
| ✅ 数学计算（近似） | ❌ 精确计算后操作数据库 |

### 函数调用工作流程

```
1. 用户提问："北京今天天气怎么样？"

2. 大模型判断需要调用工具，生成函数调用：
{
  "name": "get_weather",
  "arguments": {"city": "北京"}
}

3. 执行函数，获取结果：
{"temperature": 25, "condition": "晴"}

4. 大模型整合结果回答用户：
"北京今天天气晴朗，气温25度。"
```

### 三种调用模式

| 模式 | 说明 | 例子 |
|------|------|------|
| 单函数调用 | 一次只调用一个函数 | 查天气 |
| 并行调用 | 一次调用多个函数 | 同时查北京、上海天气 |
| 链式调用 | 函数结果作为下一个函数输入 | 先查城市→再查该城市天气 |

---

## Part 2：函数定义与结构化参数

### 标准函数定义

```json
{
  "name": "get_weather",
  "description": "查询指定城市的天气信息",
  "parameters": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string",
        "description": "城市名称，如：北京、上海"
      }
    },
    "required": ["city"]
  }
}
```

### 函数定义要点

| 字段 | 必须 | 说明 |
|------|------|------|
| name | ✅ | 函数名，唯一标识 |
| description | ✅ | 描述函数功能，让模型理解何时调用 |
| parameters | ✅ | 参数定义，告诉模型需要什么信息 |
| required | - | 必填参数列表 |

### 参数类型

| 类型 | 例子 |
|------|------|
| string | "北京"、"hello" |
| number | 25、3.14 |
| boolean | true、false |
| array | ["北京", "上海"] |
| object | {"city": "北京", "date": "2026-05-03"} |

---

## Part 3：实战——国产模型函数调用

### 环境准备

```bash
pip install openai python-dotenv
```

创建 `.env` 文件：
```bash
ZHIPU_API_KEY=你的智谱API密钥
# 或
DEEPSEEK_API_KEY=你的DeepSeek API密钥
```

### 实战一：天气查询函数

```python
from openai import OpenAI
import json
import os
from dotenv import load_dotenv

load_dotenv()

# 选择国产模型
client = OpenAI(
    api_key=os.getenv("ZHIPU_API_KEY"),
    base_url="https://open.bigmodel.cn/api/paas/v4"
)

# 定义函数
functions = [
    {
        "name": "get_weather",
        "description": "查询城市天气",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "城市名称"
                }
            },
            "required": ["city"]
        }
    }
]

# 模拟天气查询
def get_weather(city: str):
    weather_db = {
        "北京": {"temp": 25, "weather": "晴", "pm2.5": 50},
        "上海": {"temp": 28, "weather": "多云", "pm2.5": 80},
        "深圳": {"temp": 30, "weather": "晴", "pm2.5": 40}
    }
    return weather_db.get(city, {"error": "未找到该城市"})

# 对话函数
def chat_with_tools(user_input):
    messages = [{"role": "user", "content": user_input}]

    response = client.chat.completions.create(
        model="glm-5.1",
        messages=messages,
        tools=[
            {"type": "function", "function": functions[0]}
        ],
        tool_choice="auto"
    )

    assistant_msg = response.choices[0].message

    # 如果有函数调用
    if assistant_msg.tool_calls:
        tool_call = assistant_msg.tool_calls[0]
        func_name = tool_call.function.name
        func_args = json.loads(tool_call.function.arguments)

        # 执行函数
        if func_name == "get_weather":
            result = get_weather(func_args["city"])

        # 把结果返回给模型
        messages.append(assistant_msg)
        messages.append({
            "role": "tool",
            "tool_call_id": tool_call.id,
            "content": json.dumps(result, ensure_ascii=False)
        })

        # 再次调用获取最终回答
        final_response = client.chat.completions.create(
            model="glm-5.1",
            messages=messages
        )
        return final_response.choices[0].message.content

    return assistant_msg.content

# 测试
print(chat_with_tools("北京今天天气怎么样？"))
```

### 实战二：多函数并行调用

```python
functions = [
    {
        "name": "get_weather",
        "description": "查询城市天气",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "城市名称"}
            },
            "required": ["city"]
        }
    },
    {
        "name": "get_news",
        "description": "获取最新新闻",
        "parameters": {
            "type": "object",
            "properties": {
                "category": {
                    "type": "string",
                    "description": "新闻类别：tech/business/entertainment"
                }
            }
        }
    }
]

# 用户问："北京天气怎么样？有什么科技新闻？"
# 模型可能并行调用 weather 和 news 两个函数
```

### 实战三：链式调用

```python
functions = [
    {
        "name": "get_location",
        "description": "根据公司名称获取所在城市",
        "parameters": {
            "type": "object",
            "properties": {
                "company": {"type": "string", "description": "公司名称"}
            },
            "required": ["company"]
        }
    },
    {
        "name": "get_weather",
        "description": "查询城市天气",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "城市名称"}
            },
            "required": ["city"]
        }
    }
]

# 用户问："百度大厦今天的天气怎么样？"
# 1. 先调用 get_location("百度大厦") → 返回 "北京"
# 2. 再调用 get_weather("北京") → 返回天气信息
```

---

## 今日要点

- ✅ 理解了函数调用的原理——让大模型连接外部世界
- ✅ 掌握了函数定义的结构（name、description、parameters）
- ✅ 学会了三种调用模式（单函数、并行、链式）
- ✅ 能够使用国产模型实现函数调用

---

## 自测题

1. 函数调用解决了大模型的什么问题？
2. 函数定义中，哪个字段告诉模型什么时候调用这个函数？
3. 单函数、并行、链式三种模式有什么区别？
4. 为什么国产模型更适合国内开发者？

---

## 动手作业

1. 定义一个计算器函数（加减乘除）
2. 实现天气+穿衣建议的组合调用
3. 思考：什么场景需要链式调用？举出例子
