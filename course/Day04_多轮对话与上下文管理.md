# Day 4：多轮对话与上下文管理

> 本课程学习多轮对话原理、上下文窗口管理，以及如何构建聊天机器人。

## Part 1：多轮对话原理

### 什么是 messages 格式？

LLM 对话使用 `messages` 数组传递上下文：

```python
messages = [
    {"role": "system", "content": "你是一位Python专家"},
    {"role": "user", "content": "什么是for循环？"},
    {"role": "assistant", "content": "for循环是..."},
    {"role": "user", "content": "再举个例子"}
]
```

- **system**：系统指令（设定角色、行为规则）
- **user**：用户的问题
- **assistant**：模型的回复

### 对话上下文如何工作？

每次调用需要把**完整对话历史**都传进去：

```python
from openai import OpenAI
import os
from dotenv import load_dotenv

load_dotenv()

client = OpenAI(
    api_key=os.getenv("ZHIPU_API_KEY"),
    base_url="https://open.bigmodel.cn/api/paas/v4"
)

# 第一轮对话
messages = [{"role": "user", "content": "什么是词元？"}]
response = client.chat.completions.create(model="glm-4-flash", messages=messages)

# 第二轮——把历史都带上！
messages.append({"role": "assistant", "content": response.choices[0].message.content})
messages.append({"role": "user", "content": "它和单词有什么区别？"})

response = client.chat.completions.create(model="glm-4-flash", messages=messages)
```

> ⚠️ **关键**：LLM 没有记忆，每次调用都是独立的。必须传完整对话历史，模型才能"记得"之前说了什么。

---

## Part 2：对话管理技巧

### System Prompt 优化

```python
# ❌ 太模糊
messages = [{"role": "system", "content": "你是一个助手"},
            {"role": "user", "content": "帮我写代码"}]

# ✅ 更精确
messages = [
    {"role": "system", "content": """你是一位资深Python工程师，具有以下特点：
1. 代码风格遵循 PEP 8 规范
2. 喜欢用类型注解（type hint）
3. 回复简洁，直接给出代码和简要说明
4. 如果有更优方案会主动提示"""},
    {"role": "user", "content": "帮我写一个快速排序"}]
```

### 角色扮演技巧

```python
messages = [
    {"role": "system", "content": """你是一位数据分析专家，专长：Python数据分析（Pandas/NumPy）、SQL查询、数据可视化。性格：严谨但耐心，喜欢用数据说话。每次回答后，会给出置信度（高/中/低）。"""},
    {"role": "user", "content": "分析一下这份CSV文件"}]
```

### 对话历史管理

```python
def manage_conversation(messages, max_history=10):
    """只保留最近 N 轮对话，避免超出上下文限制"""
    if len(messages) > max_history * 2 + 1:  # user+assistant算一轮
        return [messages[0]] + messages[-(max_history * 2):]
    return messages
```

---

## Part 3：上下文窗口与截断

### 上下文窗口是什么？

上下文窗口 = 模型一次能处理的最大词元数。

| 模型 | 上下文窗口 |
|---|---|
| GLM-5.1 | 100万词元 |
| DeepSeek V4 | 100万词元 |
| Qwen 3.6-Plus | 100万词元（可扩至1M） |
| Kimi K2.6 | 100万-1M词元 |

> 💡 100万词元 ≈ 50-70万汉字（一部长篇小说）

### 截断策略

```python
def truncate_to_fit(messages, max_tokens, model="glm-4-flash"):
    """智能截断对话历史，确保不超过上下文限制"""
    total = sum(estimate_tokens(m['content']) for m in messages)

    while total > max_tokens and len(messages) > 2:
        messages.pop(1)  # 删除最早的一轮对话（保留 system prompt）
        total = sum(estimate_tokens(m['content']) for m in messages)

    return messages
```

---

## Part 4：实战——命令行聊天机器人

### 完整代码

```python
"""
命令行聊天机器人
支持多轮对话，自动管理上下文
"""
from openai import OpenAI
import os
from dotenv import load_dotenv

load_dotenv()

client = OpenAI(
    api_key=os.getenv("ZHIPU_API_KEY"),
    base_url="https://open.bigmodel.cn/api/paas/v4"
)

def chat_with_model(messages, model="glm-4-flash"):
    response = client.chat.completions.create(
        model=model,
        messages=messages,
        temperature=0.7,
        max_tokens=500
    )
    return response.choices[0].message.content, response.usage.total_tokens

def manage_context(messages, max_history=10):
    if len(messages) > max_history * 2 + 1:
        return [messages[0]] + messages[-(max_history * 2):]
    return messages

def main():
    print("国产大模型聊天机器人")
    messages = [{"role": "system", "content": "你是一个友好、有帮助的AI助手。"}]
    total_tokens = 0

    while True:
        user_input = input("\n你: ").strip()
        if user_input.lower() == "quit":
            print(f"本次对话共使用 {total_tokens} 词元")
            break

        if user_input.lower() == "clear":
            messages = [{"role": "system", "content": "你是一个友好、有帮助的AI助手。"}]
            print("对话历史已清空")
            continue

        messages.append({"role": "user", "content": user_input})
        messages = manage_context(messages)

        try:
            reply, tokens = chat_with_model(messages)
            total_tokens += tokens
            print(f"\nAI: {reply}")
            messages.append({"role": "assistant", "content": reply})
        except Exception as e:
            print(f"错误：{e}")

if __name__ == "__main__":
    main()
```

---

## Part 5：长对话策略

对话越长，维护难度越大。以下是三种核心策略：

### 策略一：摘要压缩

将早期对话压缩成摘要，保留关键信息：

```python
def summarize_conversation(messages, llm):
    """将对话历史压缩成摘要"""
    history_text = ""
    for m in messages:
        role = "用户" if m["role"] == "user" else "AI"
        history_text += f"{role}：{m['content'][:100]}...\n"

    summary_prompt = f"""请将以下对话压缩成一段摘要，保留关键信息：
- 用户的需求和目标
- 已经讨论过的方案
- 已经确定的决策
- 待办事项

对话：
{history_text}

摘要（100字以内）："""

    summary = llm.invoke(summary_prompt)
    return summary.content

def maintain_with_summary(messages, llm, max_rounds=5):
    """使用摘要维护长对话"""
    system = messages[0]
    conversation = messages[1:]

    # 只保留最近 N 轮完整对话
    recent = conversation[-(max_rounds * 2):]

    # 更早的对话压缩成摘要
    if len(conversation) > max_rounds * 2:
        early = conversation[:-(max_rounds * 2)]
        summary = summarize_conversation(early, llm)
        return [
            system,
            {"role": "system", "content": f"【历史摘要】{summary}"},
            *recent
        ]

    return messages
```

### 策略二：关键信息提取

不是"压缩"，而是"提取"——只保留对后续对话有用的信息：

```python
def extract_key_info(messages, llm):
    """从对话中提取关键信息"""
    history = "\n".join([f"{m['role']}: {m['content']}" for m in messages[-10:]])
    prompt = f"""分析以下对话，提取：

1. 用户的关键信息（姓名、偏好、需求）
2. 已经确认的事实和决策
3. 待办项和未完成的内容

只提取真实提到的信息，不要编造。

对话：
{history}

返回 JSON：
{{"user_info": {{}}, "decisions": [], "todos": []}}"""

    result = llm.invoke(prompt)
    return result.content
```

### 策略三：滑动窗口

最简单的策略，但最可靠：

```python
def sliding_window(messages, max_tokens=4000, reserve_system=True):
    """滑动窗口截断"""
    if reserve_system:
        system = [messages[0]]
        history = messages[1:]
    else:
        system = []
        history = messages

    total = sum(estimate_tokens(m["content"]) for m in messages)

    while total > max_tokens and len(history) > 2:
        history.pop(0)  # 丢掉最早的消息
        total = sum(estimate_tokens(m["content"]) for m in system + history)

    return system + history
```

### 三种策略对比

| 策略 | 信息保留度 | 实现复杂度 | 适合场景 |
|------|-----------|-----------|----------|
| **滑动窗口** | 低（直接丢掉） | ⭐ 简单 | 简单问答、客服 |
| **摘要压缩** | 中（压缩保留） | ⭐⭐ 中等 | 技术咨询、方案讨论 |
| **关键信息提取** | 高（结构化） | ⭐⭐⭐ 较高 | 项目协作、需求分析 |

---

## Part 6：对话状态持久化

对话状态保存到数据库，支持断点续聊：

```python
import json
import sqlite3
from datetime import datetime

class ConversationStore:
    """对话存储"""

    def __init__(self, db_path="conversations.db"):
        self.conn = sqlite3.connect(db_path)
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS conversations (
                id TEXT PRIMARY KEY,
                messages TEXT,
                created_at TEXT,
                updated_at TEXT,
                metadata TEXT
            )
        """)

    def save(self, conv_id, messages, metadata=None):
        """保存对话"""
        now = datetime.now().isoformat()
        self.conn.execute("""
            INSERT OR REPLACE INTO conversations
            (id, messages, created_at, updated_at, metadata)
            VALUES (?, ?, COALESCE((SELECT created_at FROM conversations WHERE id=?), ?), ?, ?)
        """, (
            conv_id,
            json.dumps(messages, ensure_ascii=False),
            conv_id,
            now,
            now,
            json.dumps(metadata or {}, ensure_ascii=False)
        ))
        self.conn.commit()

    def load(self, conv_id):
        """加载对话"""
        cursor = self.conn.execute(
            "SELECT messages, metadata FROM conversations WHERE id=?",
            (conv_id,)
        )
        row = cursor.fetchone()
        if row:
            return {
                "messages": json.loads(row[0]),
                "metadata": json.loads(row[1])
            }
        return None

    def list_conversations(self):
        """列出所有对话"""
        cursor = self.conn.execute(
            "SELECT id, created_at, updated_at FROM conversations ORDER BY updated_at DESC"
        )
        return cursor.fetchall()

    def delete(self, conv_id):
        """删除对话"""
        self.conn.execute("DELETE FROM conversations WHERE id=?", (conv_id,))
        self.conn.commit()

# 使用
store = ConversationStore()
store.save("conv-001", messages, {"user": "张三", "topic": "Python学习"})
loaded = store.load("conv-001")
print(f"恢复 {len(loaded['messages'])} 条对话消息")
```

- ✅ 理解了 `messages` 数组格式（system/user/assistant）
- ✅ 学会了多轮对话的上下文维护
- ✅ 掌握了 System Prompt 优化和角色扮演技巧
- ✅ 理解了上下文窗口概念（100万词元 ≈ 50-70万汉字）
- ✅ 写出了完整的命令行聊天机器人

---

## 自测题

1. `messages` 数组中 system/user/assistant 的区别是什么？
2. 为什么每次调用 LLM 都要传入完整的对话历史？
3. 当前主流国产模型的上下文窗口有多大？
4. 如果对话太长超出上下文限制会发生什么？
5. `clear` 命令在聊天机器人中的作用是什么？

---

## 动手作业

1. 修改聊天机器人的 System Prompt，将其改为"一位Python教师"
2. 添加"切换模型"功能，让用户可以在 GLM/DeepSeek/Qwen 之间切换
3. 给聊天机器人添加"保存对话"功能，将对话历史保存到文件
