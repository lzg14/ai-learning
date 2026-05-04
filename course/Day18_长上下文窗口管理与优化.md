# Day 18：长上下文窗口管理与优化

> 窗口越大不代表越好，合理管理才是关键。本课深入 Token 计数、滑动窗口、上下文压缩、重要信息保留等核心技术。

## Part 1：Token 计数

### Token 估算基础

| 类型 | 估算 | 示例 |
|------|------|------|
| 中文 | 1 汉字 ≈ 1.3 Tokens | "你好世界" ≈ 4 Tokens |
| 英文 | 1 单词 ≈ 1.3 Tokens | "hello world" ≈ 2 Tokens |
| 代码 | 1 行代码 ≈ 2-3 Tokens | 一行 Python ≈ 4-6 Tokens |

### Tiktoken 精确计数

```python
import tiktoken

# GPT-4 编码器
enc = tiktoken.encoding_for_model("gpt-4")

text = "你好，世界！这是一段测试文本。"
tokens = enc.encode(text)
print(f"Token 数量: {len(tokens)}")
print(f"Token IDs: {tokens}")
```

### 各模型上下文限制

| 模型 | 上下文限制 | 约等于多少汉字 |
|------|------------|----------------|
| GLM-4 | 128K | 约 10 万字 |
| DeepSeek | 128K | 约 10 万字 |
| Qwen | 128K | 约 10 万字 |
| Kimi | 200K | 约 15 万字 |

---

## Part 2：滑动窗口

### 基本策略

```plaintext
新消息进来 → 计算总 Token → 超过上限 → 删除最早的消息
```

### 实现滑动窗口

```python
class SlidingWindowContext:
    """滑动窗口上下文管理"""

    def __init__(self, max_tokens: int = 8000):
        self.max_tokens = max_tokens
        self.messages = []
        self.enc = tiktoken.encoding_for_model("gpt-4")

    def count_tokens(self, text: str) -> int:
        return len(self.enc.encode(text))

    def total_tokens(self) -> int:
        return sum(self.count_tokens(m["content"]) for m in self.messages)

    def add_message(self, role: str, content: str):
        """添加消息，自动滑动"""
        self.messages.append({"role": role, "content": content})

        # 如果超过上限，删除最早的消息
        while self.total_tokens() > self.max_tokens and len(self.messages) > 2:
            removed = self.messages.pop(0)
            print(f"移除旧消息: {removed['content'][:30]}...")

    def get_context(self) -> list:
        return self.messages
```

### 系统提示词保护

```
保留系统提示词
    ↓
系统提示词（固定保留） + 最近 N 条消息（滑动窗口）
```

---

## Part 3：上下文压缩

### 三种压缩策略

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| **摘要压缩** | 把历史对话总结成摘要 | 长对话压缩 |
| **去重压缩** | 删除重复、无关的内容 | 内容冗余 |
| **结构化压缩** | 提取实体、关系、关键事实 | 实体密集型 |

### 摘要压缩实现

```python
from langchain_community.chat_models import ChatZhipuAI

llm = ChatZhipuAI(
    api_key=os.getenv("ZHIPU_API_KEY"),
    model="glm-4-flash",
    temperature=0.0
)

def compress_conversation(messages: list, max_messages: int = 10) -> list:
    """压缩对话历史"""
    if len(messages) <= max_messages:
        return messages

    # 保留最近的对话
    recent = messages[-max_messages:]
    early = messages[:-max_messages]

    # 用 LLM 生成摘要
    early_content = "\n".join([f"{m['role']}: {m['content']}" for m in early])

    summary_prompt = f"""请将以下对话总结成简短的摘要，保留关键信息：

{early_content}

摘要格式：
- 用户主要说了什么？
- AI 主要回答了什么？
- 有哪些关键信息需要记住？
"""

    summary = llm.invoke(summary_prompt).content

    return [
        {"role": "system", "content": f"[早期对话摘要] {summary}"}
    ] + recent
```

### 结构化压缩

```python
def extract_structured_info(messages: list) -> dict:
    """提取结构化信息"""
    context = "\n".join([f"{m['role']}: {m['content']}" for m in messages])

    extract_prompt = f"""从以下对话中提取关键信息：

{context}

请提取并返回 JSON 格式：
{{
  "user_profile": {{"name": "", "role": "", "preferences": []}},
  "entities": ["实体1", "实体2"],
  "facts": ["关键事实1", "关键事实2"],
  "decisions": ["已做决策1"]
}}
"""

    result = llm.invoke(extract_prompt).content
    return result
```

---

## Part 4：重要性评分

### 评分维度

| 维度 | 说明 | 权重 |
|------|------|------|
| **时效性** | 最近的更重要 | 高 |
| **相关性** | 和当前查询相关的更重要 | 高 |
| **情感强度** | 用户强调过的更重要 | 中 |
| **用户标识** | 用户标记为重要的永远保留 | 最高 |

### 混合策略实现

```python
def build_priority_context(messages: list, current_query: str) -> str:
    """基于重要性评分构建上下文"""

    # 1. 系统提示词永远保留
    system_prompt = next((m for m in messages if m["role"] == "system"), {}).get("content", "")

    # 2. 标记重要的永远保留
    important = [m for m in messages if m.get("important", False)]

    # 3. 最近的对话
    recent = messages[-10:] if len(messages) > 10 else messages

    # 4. 相关的对话（简单关键词匹配）
    query_terms = current_query.lower().split()
    relevant = [
        m for m in messages
        if any(term in m["content"].lower() for term in query_terms)
        and m not in recent
    ]

    # 构建上下文
    parts = [f"[系统] {system_prompt}"]
    parts.extend([f"[重要] {m['content']}" for m in important])
    parts.extend([f"[最近] {m['content']}" for m in recent])
    parts.extend([f"[相关] {m['content']}" for m in relevant[:5]])

    return "\n\n".join(parts)
```

### 最终策略：五层上下文

```plaintext
第 1 层：系统提示词（永远保留）
    ↓
第 2 层：用户标记的重要信息（永远保留）
    ↓
第 3 层：最近 K 轮对话（完整保留）
    ↓
第 4 层：更早对话（压缩成摘要保留）
    ↓
第 5 层：无关内容（删除）
```

---

## Part 5：长上下文前沿技术

### 主流模型上下文窗口对比（2026）

| 模型 | 上下文窗口 | 核心技术 |
|------|-----------|----------|
| **Gemini 1.5/2.0** | 2M tokens | Gemini架构 + 稀疏注意力 |
| **Kimi K2** | 200万 tokens | Mooncake 架构 + 降噪注意力 |
| **GLM-5** | 1M tokens | 改进的 FlashAttention |
| **DeepSeek V4** | 1M tokens | MLA + 稀疏注意力 |
| **Claude 3.5/4** | 200K tokens | 优化的 Transformer |
| **GPT-4** | 128K tokens | 标准 Transformer |

### RingAttention：环状注意力机制

RingAttention 让模型可以在多个 GPU 上分布式处理超长序列，突破单机显存限制：

```python
# RingAttention 的核心理念：分块循环处理
def ring_attention_simulate(query, keys, values, num_blocks=4):
    """
    模拟 RingAttention 的分块处理
    将长序列分成多块，循环在各 GPU 上处理
    """
    block_size = len(keys) // num_blocks
    outputs = []

    for i in range(num_blocks):
        # 每个 GPU 处理一块
        start = i * block_size
        end = start + block_size
        block_keys = keys[start:end]
        block_values = values[start:end]

        # 计算注意力
        scores = query @ block_keys.T
        weights = softmax(scores)
        output = weights @ block_values
        outputs.append(output)

        print(f"Block {i+1}/{num_blocks} 处理完成")

    # 聚合所有块的输出
    return concatenate(outputs)

# 应用场景：100万 token 的文档
# 传统方法：OOM（显存溢出）
# RingAttention：O(N) 线性扩展，在每个 GPU 上处理 25万 token
```

**关键优势：**
- 线性显存扩展（不是平方级）
- 可以处理任意长度的上下文
- 跨 GPU 通信量最小化

### YaRN：位置编码扩展

YaRN（Yet another RoPE extensioN）解决模型**推理时遇到比训练时更长的序列**的问题：

```python
# 问题：模型训练时只见过 128K 长度的位置编码
# 推理时传入 200K → 位置编码超出范围 → 效果断崖下跌

def yarn_position_scaling(position_ids, max_trained=128000, max_infer=200000):
    """
    YaRN 位置缩放：将推理时的位置映射到训练时的范围内

    原理：不改变位置编码的结构，只改变位置的"频率"
    """
    # 缩放因子
    scale = max_trained / max_infer  # 128K / 200K ≈ 0.64

    # 对每个位置进行缩放
    scaled_positions = [int(pos * scale) for pos in position_ids]

    print(f"原始位置范围：0-{max_infer}")
    print(f"缩放后范围：0-{int(max_infer * scale)}")
    print(f"缩放因子：{scale}")

    return scaled_positions

# YaRN 效果：Llama 2 从 4K 扩展到 32K+，性能几乎不损
```

### 稀疏注意力

不是所有 token 都需要关注所有其他 token，稀疏注意力只关注"重要的"token：

| 方法 | 关注范围 | 计算复杂度 |
|------|---------|-----------|
| **全局注意力** | 特殊 token 关注全局 | O(L) |
| **滑动窗口** | 只关注附近 N 个 token | O(N×L) |
| **稀疏全局** | 随机选择 token 关注全局 | O(log L × L) |
| **MQA/GQA** | 多查询/分组查询注意力 | O(L) 显存减半 |

### 长上下文的实际应用场景

| 场景 | 上下文需求 | 关键技术 |
|------|-----------|----------|
| **长文档分析** | 100K-1M tokens | RingAttention + 滑动窗口 |
| **代码库理解** | 50K-500K tokens | 稀疏注意力 + 结构感知 |
| **多轮对话** | 10K-100K tokens | 滑动窗口 + 摘要压缩 |
| **Agent 记忆** | 100K-1M tokens | 层级记忆 + 关键信息提取 |

---

## 📌 今日要点

- ✅ 掌握了 Token 估算：中文 1 汉字 ≈ 1.3 Tokens
- ✅ 理解了滑动窗口策略，自动删除旧消息
- ✅ 学会了三种压缩策略：摘要、去重、结构化
- ✅ 理解了重要性评分的四个维度
- ✅ 掌握了五层上下文混合策略

---

## 自测题

1. Token 的估算规则是什么？1 万字大概多少 Token？
2. 滑动窗口的基本策略是什么？
3. 三种压缩策略分别适用于什么场景？
4. 重要性评分的四个维度是什么？

---

## 动手作业

1. 实现一个滑动窗口上下文管理器
2. 给上下文管理器添加摘要压缩功能
3. 实现基于重要性评分的混合策略
