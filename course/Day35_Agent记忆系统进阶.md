# Day 35：Agent 记忆系统进阶

> 从"用完就忘"到"越用越聪明"。本课深入记忆检索、遗忘机制、长期学习、记忆与知识的融合。

## Part 1：为什么需要高级记忆系统？

### 传统记忆的问题

**你是不是也遇到过这种情况：**
1. 你教了 Agent 一个新知识，用了几次，然后过几天再问，它又忘了
2. Agent 学了新东西之后，之前会的东西反而不会了
3. Agent 记了一堆东西，但需要用的时候找不到

这就是**记忆系统不完善**的问题！

---

### 高级记忆系统的五大核心能力

| 能力 | 说明 | 类比 |
| --- | --- | --- |
| **1. 增量学习** | 学了新东西不忘记旧东西 | 人学了英语再学法语，不会把中文忘了 |
| **2. 选择性记忆** | 重要的多记，不重要的少记 | 人会记住重要的，忘掉鸡毛蒜皮的 |
| **3. 快速检索** | 需要的时候能快速找到 | 人不会忘记怎么骑自行车 |
| **4. 遗忘机制** | 主动忘掉没用的，保护隐私 | 人会自动忘掉不重要的 |
| **5. 知识融合** | 新旧知识能整合 | 人能把新学的和旧知识连起来 |

---

## Part 2：记忆架构设计

### 四层记忆架构

```plaintext
┌─────────────────────────────────────────┐
│           第一层：工作记忆                  │
│    当前对话上下文，在思考时活跃使用           │
├─────────────────────────────────────────┤
│           第二层：短期记忆                  │
│    最近的经验和结论，会被压缩到长期记忆      │
├─────────────────────────────────────────┤
│           第三层：长期记忆                  │
│    重要的事实、技能、偏好，持久存储          │
├─────────────────────────────────────────┤
│           第四层：元认知                   │
│    关于记忆的记忆，知道自己知道什么          │
└─────────────────────────────────────────┘
```

---

### 记忆的数据结构

```python
@dataclass
class Memory:
    """记忆单元"""
    id: str
    content: str
    memory_type: MemoryType  # 事实、经验、技能、偏好
    importance: float  # 0-1 重要性
    accessibility: float  # 0-1 可访问性
    created_at: float
    last_accessed: float
    access_count: int
    embedding: List[float]  # 用于检索
    metadata: Dict  # 额外信息

class MemoryType(Enum):
    FACT = "fact"  # 事实性知识
    EXPERIENCE = "experience"  # 经验
    SKILL = "skill"  # 技能
    PREFERENCE = "preference"  # 偏好
    CONTEXT = "context"  # 上下文
```

---

## Part 3：记忆检索技术

### 语义检索

**核心思想：** 不只是匹配关键词，而是找"意思相近"的记忆。

```python
class MemoryRetriever:
    """记忆检索器"""

    def __init__(self, embedding_model):
        self.embedding_model = embedding_model
        self.memories: List[Memory] = []

    async def retrieve(self, query: str, top_k: int = 5) -> List[Memory]:
        """检索相关记忆"""
        # 1. 把查询向量化
        query_embedding = self.embedding_model.encode(query)

        # 2. 计算相似度
        similarities = []
        for memory in self.memories:
            sim = self._cosine_similarity(query_embedding, memory.embedding)
            similarities.append((memory, sim))

        # 3. 排序返回 Top K
        similarities.sort(key=lambda x: x[1], reverse=True)
        return [m for m, _ in similarities[:top_k]]

    def _cosine_similarity(self, a: List[float], b: List[float]) -> float:
        """余弦相似度"""
        dot = sum(x * y for x, y in zip(a, b))
        norm_a = sum(x * x for x in a) ** 0.5
        norm_b = sum(x * x for x in b) ** 0.5
        return dot / (norm_a * norm_b + 1e-8)
```

---

### 混合检索策略

| 检索方法 | 优点 | 缺点 |
| --- | --- | --- |
| **精确匹配** | 准确 | 不灵活 |
| **语义相似** | 灵活 | 可能不精确 |
| **时间衰减** | 新的更重要 | 老的容易被忽略 |
| **重要性加权** | 重要的优先 | 可能忽视新信息 |

```python
class HybridRetriever:
    """混合检索器"""

    def __init__(self):
        self.keyword_index: Dict[str, List[Memory]] = defaultdict(list)
        self.semantic_retriever = MemoryRetriever()

    async def retrieve(self, query: str, top_k: int = 5) -> List[Memory]:
        """混合检索"""
        # 1. 关键词匹配
        keyword_results = self._keyword_search(query)

        # 2. 语义检索
        semantic_results = await self.semantic_retriever.retrieve(query, top_k * 2)

        # 3. 混合评分
        scores = {}
        for memory in set(keyword_results + semantic_results):
            score = self._calculate_score(memory, query)
            scores[memory.id] = score

        # 4. 排序返回
        sorted_memories = sorted(
            [m for m in keyword_results + semantic_results],
            key=lambda m: scores[m.id],
            reverse=True
        )

        return sorted_memories[:top_k]

    def _calculate_score(self, memory: Memory, query: str) -> float:
        """计算综合评分"""
        semantic_score = self.semantic_retriever._cosine_similarity(
            self.semantic_retriever.embedding_model.encode(query),
            memory.embedding
        )

        # 时间衰减：新记忆权重更高
        time_weight = 1.0 / (1.0 + 0.1 * (time.time() - memory.last_accessed) / 86400)

        # 重要性权重
        importance_weight = memory.importance

        # 访问频率权重
        frequency_weight = math.log(1 + memory.access_count) / 10

        return semantic_score * 0.4 + importance_weight * 0.3 + time_weight * 0.2 + frequency_weight * 0.1
```

---

## Part 4：遗忘机制与记忆压缩

### 遗忘机制

**为什么需要遗忘？**
1. 保护隐私：敏感信息不应该永久存储
2. 节省空间：不需要记住所有细节
3. 减少干扰：无关的记忆会干扰检索

### 遗忘策略

```python
class MemoryForgetting:
    """记忆遗忘"""

    def __init__(self):
        self.forgetting_threshold = 0.1

    def should_forget(self, memory: Memory) -> bool:
        """判断是否应该遗忘"""
        # 1. 重要性太低
        if memory.importance < self.forgetting_threshold:
            return True

        # 2. 太久没访问
        days_since_access = (time.time() - memory.last_accessed) / 86400
        if days_since_access > 90 and memory.access_count < 2:
            return True

        # 3. 隐私标记
        if memory.metadata.get("is_private", False):
            if days_since_access > 30:
                return True

        return False

    def calculate_decay(self, memory: Memory) -> float:
        """计算记忆衰减"""
        # 基于时间的自然遗忘
        days = (time.time() - memory.last_accessed) / 86400

        # 艾宾浩斯遗忘曲线（简化版）
        decay = math.exp(-0.1 * days) * memory.importance

        return max(0.0, decay)
```

---

### 记忆压缩

```python
class MemoryCompressor:
    """记忆压缩器"""

    def __init__(self, llm):
        self.llm = llm

    async def compress(self, memories: List[Memory]) -> List[Memory]:
        """压缩多条记忆"""
        if len(memories) < 3:
            return memories

        # 1. 按类型分组
        by_type = defaultdict(list)
        for m in memories:
            by_type[m.memory_type].append(m)

        compressed = []

        # 2. 每组压缩成一个摘要
        for memory_type, group in by_type.items():
            if len(group) <= 2:
                compressed.extend(group)
            else:
                summary = await self._summarize(group)
                summary.id = f"compressed_{memory_type.value}_{int(time.time())}"
                summary.memory_type = memory_type
                summary.created_at = time.time()
                compressed.append(summary)

        return compressed

    async def _summarize(self, memories: List[Memory]) -> Memory:
        """生成摘要"""
        content = "\n".join([m.content for m in memories])

        prompt = f"""把以下记忆压缩成一个简洁的摘要：

{content}

要求：
1. 保留关键信息
2. 去掉重复细节
3. 摘要不超过 100 字
"""

        summary_text = "这是压缩后的摘要..."  # 实际调用 LLM

        return Memory(
            id="",
            content=summary_text,
            memory_type=memories[0].memory_type,
            importance=sum(m.importance for m in memories) / len(memories),
            accessibility=0.5,
            created_at=time.time(),
            last_accessed=time.time(),
            access_count=0,
            embedding=[0.0] * 768,
            metadata={"is_compressed": True, "original_count": len(memories)}
        )
```

---

## Part 5：实战——高级记忆 Agent

### 完整代码

```python
"""
高级记忆 Agent
包含：四层记忆、语义检索、遗忘机制、记忆压缩
"""
import asyncio
import time
import math
import hashlib
from typing import List, Dict, Any, Optional
from dataclasses import dataclass, field
from enum import Enum
from collections import defaultdict
import random

class MemoryType(Enum):
    FACT = "fact"
    EXPERIENCE = "experience"
    SKILL = "skill"
    PREFERENCE = "preference"
    CONTEXT = "context"

@dataclass
class Memory:
    id: str
    content: str
    memory_type: MemoryType
    importance: float = 0.5
    accessibility: float = 0.5
    created_at: float = field(default_factory=time.time)
    last_accessed: float = field(default_factory=time.time)
    access_count: int = 0
    embedding: List[float] = field(default_factory=lambda: [random.random() for _ in range(128)])
    metadata: Dict = field(default_factory=dict)

class WorkingMemory:
    """工作记忆 - 当前对话上下文"""

    def __init__(self, max_size: int = 10):
        self.messages: List[Dict] = []
        self.max_size = max_size
        self.current_task: str = ""

    def add(self, role: str, content: str):
        """添加消息"""
        self.messages.append({"role": role, "content": content, "time": time.time()})
        if len(self.messages) > self.max_size:
            self.messages.pop(0)

    def get_context(self) -> str:
        """获取上下文"""
        return "\n".join([f"{m['role']}: {m['content']}" for m in self.messages])

    def clear(self):
        """清空"""
        self.messages = []
        self.current_task = ""

class ShortTermMemory:
    """短期记忆 - 最近的经验"""

    def __init__(self, max_size: int = 100):
        self.memories: List[Memory] = []
        self.max_size = max_size
        self.compression_threshold = 50

    def add(self, memory: Memory):
        """添加记忆"""
        memory.id = f"stm_{hashlib.md5(str(time.time()).encode()).hexdigest()[:8]}"
        self.memories.append(memory)

        if len(self.memories) > self.max_size:
            self._trigger_compression()

    def _trigger_compression(self):
        """触发压缩"""
        print(f"📦 短期记忆达到 {len(self.memories)} 条，触发压缩...")

        # 保留最近的和重要的
        self.memories.sort(key=lambda m: (m.last_accessed, m.importance), reverse=True)
        self.memories = self.memories[:self.max_size // 2]

    def get_recent(self, count: int = 10) -> List[Memory]:
        """获取最近的记忆"""
        sorted_memories = sorted(self.memories, key=lambda m: m.created_at, reverse=True)
        return sorted_memories[:count]

class LongTermMemory:
    """长期记忆 - 重要事实和知识"""

    def __init__(self):
        self.memories: List[Memory] = []
        self.type_index: Dict[MemoryType, List[Memory]] = defaultdict(list)
        self.keyword_index: Dict[str, List[Memory]] = defaultdict(list)

    def add(self, memory: Memory):
        """添加记忆"""
        memory.id = f"ltm_{hashlib.md5(str(time.time()).encode()).hexdigest()[:8]}"
        self.memories.append(memory)

        # 更新索引
        self.type_index[memory.memory_type].append(memory)

        for keyword in memory.content.split()[:5]:
            if len(keyword) > 2:
                self.keyword_index[keyword.lower()].append(memory)

    def search(self, query: str, memory_type: MemoryType = None, top_k: int = 5) -> List[Memory]:
        """搜索记忆"""
        candidates = []

        if memory_type:
            candidates = self.type_index.get(memory_type, [])
        else:
            candidates = self.memories

        # 关键词匹配
        query_words = query.lower().split()
        results = []
        for memory in candidates:
            score = sum(1 for w in query_words if w in memory.content.lower())
            if score > 0:
                results.append((memory, score))

        # 排序
        results.sort(key=lambda x: (x[1], x[0].importance), reverse=True)
        return [m for m, _ in results[:top_k]]

    def should_forget(self, memory: Memory) -> bool:
        """判断是否应该遗忘"""
        days_since_access = (time.time() - memory.last_accessed) / 86400

        if memory.importance < 0.1 and days_since_access > 90:
            return True
        if memory.metadata.get("is_private") and days_since_access > 30:
            return True

        return False

class AdvancedMemoryAgent:
    """高级记忆 Agent"""

    def __init__(self):
        self.working_memory = WorkingMemory()
        self.short_term_memory = ShortTermMemory()
        self.long_term_memory = LongTermMemory()

    async def remember(self, content: str, memory_type: MemoryType,
                       importance: float = 0.5, metadata: Dict = None):
        """存储记忆"""
        memory = Memory(
            id="",
            content=content,
            memory_type=memory_type,
            importance=importance,
            metadata=metadata or {}
        )

        if memory_type == MemoryType.CONTEXT:
            self.working_memory.add("assistant", content)
        elif importance > 0.6:
            self.long_term_memory.add(memory)
        else:
            self.short_term_memory.add(memory)

        print(f"💾 记住：[{memory_type.value}] {content[:50]}... (重要性: {importance})")

    async def recall(self, query: str, memory_type: MemoryType = None) -> str:
        """回忆"""
        # 1. 先查工作记忆
        if self.working_memory.messages:
            recent = self.working_memory.messages[-1]["content"]
            if query.lower() in recent.lower():
                print(f"🧠 从工作记忆找到：{recent[:50]}...")
                return recent

        # 2. 查长期记忆
        long_results = self.long_term_memory.search(query, memory_type)
        if long_results:
            for m in long_results:
                m.access_count += 1
                m.last_accessed = time.time()
                print(f"🧠 从长期记忆找到：{m.content[:50]}...")
                return m.content

        # 3. 查短期记忆
        short_results = self.short_term_memory.get_recent()
        for m in short_results:
            if query.lower() in m.content.lower():
                m.access_count += 1
                m.last_accessed = time.time()
                print(f"🧠 从短期记忆找到：{m.content[:50]}...")
                return m.content

        return None

    def get_stats(self) -> Dict:
        """获取统计"""
        return {
            "working_memory_size": len(self.working_memory.messages),
            "short_term_count": len(self.short_term_memory.memories),
            "long_term_count": len(self.long_term_memory.memories),
            "total_memories": len(self.short_term_memory.memories) + len(self.long_term_memory.memories)
        }

async def main():
    print("=" * 70)
    print("🧠 高级记忆 Agent 测试")
    print("=" * 70)

    agent = AdvancedMemoryAgent()

    # 1. 存储各种类型的记忆
    print("\n📝 存储记忆：")
    await agent.remember("用户叫张三，喜欢喝咖啡", MemoryType.PREFERENCE, 0.8)
    await agent.remember("北京今天的天气是晴天，25度", MemoryType.FACT, 0.6)
    await agent.remember("用户问过我怎么写 Python", MemoryType.EXPERIENCE, 0.7)
    await agent.remember("Python 列表推导式语法：[x for x in items]", MemoryType.SKILL, 0.9)
    await agent.remember("用户刚才说想要一个生日蛋糕", MemoryType.CONTEXT, 0.5)

    # 2. 回忆测试
    print("\n🔍 回忆测试：")

    result = await agent.recall("用户叫什么名字")
    print(f"问题：用户叫什么名字 → 回答：{result}")

    result = await agent.recall("Python")
    print(f"问题：Python 相关 → 回答：{result}")

    result = await agent.recall("天气")
    print(f"问题：天气相关 → 回答：{result}")

    # 3. 统计
    stats = agent.get_stats()
    print(f"\n📊 记忆统计：")
    print(f"  工作记忆：{stats['working_memory_size']} 条")
    print(f"  短期记忆：{stats['short_term_count']} 条")
    print(f"  长期记忆：{stats['long_term_count']} 条")
    print(f"  总计：{stats['total_memories']} 条")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 📌 今日要点

- ✅ 理解了为什么需要高级记忆系统
- ✅ 掌握了四层记忆架构：工作记忆、短期记忆、长期记忆、元认知
- ✅ 学会了记忆检索技术：语义检索、混合检索
- ✅ 理解了遗忘机制和记忆压缩
- ✅ 实现了完整的高级记忆 Agent

---

## 自测题

1. 四层记忆架构分别是什么？各有什么作用？
2. 混合检索是怎么工作的？为什么比单一检索好？
3. 为什么需要遗忘机制？什么情况下应该遗忘？
4. 记忆压缩的目的是什么？

## 动手作业

1. 把上面的高级记忆 Agent 跑起来
2. 添加真实的 Embedding 模型
3. 实现更复杂的遗忘策略
