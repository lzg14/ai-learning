# Day 11：Agent 记忆四层架构与上下文管理

> 记忆是 Agent 的核心能力之一。本课学习短期记忆、长期记忆、情景记忆、用户画像四层架构，实现跨会话的智能 Agent。

## Part 1：为什么 Agent 需要记忆

### 对比：没有记忆 vs 有记忆

**没有记忆的 Agent：**

```
用户：我是医生，主要看心脏病
Agent：很高兴认识您！有什么可以帮您？

用户：我擅长什么领域？
Agent：抱歉，我不知道您擅长什么领域，我们之前没有交流过。
```

**有记忆的 Agent：**

```
用户：我是医生，主要看心脏病
Agent：很高兴认识您！您是一位心脏病专科医生，很专业！

用户：我擅长什么领域？
Agent：根据之前的对话，您是一位心脏病专科医生。
```

> 💡 模型本身没有任何"记忆"，就像一台没有硬盘和内存的 CPU。所谓记忆，都是我们每次调用前，从外面"喂"给它的上下文。

### 四层记忆架构

```
┌─────────────────────────────────────────────────────────┐
│                    用户画像 (Profile)                    │
│           结构化存储：职业、技能、偏好、目标              │
├─────────────────────────────────────────────────────────┤
│                    情景记忆 (Episodic)                   │
│         重要对话摘要、关键决策、经验总结                  │
├─────────────────────────────────────────────────────────┤
│                    长期记忆 (Long-term)                  │
│           向量数据库、跨会话知识、语义检索                 │
├─────────────────────────────────────────────────────────┤
│                    短期记忆 (Short-term)                 │
│              当前对话历史、工作上下文                      │
└─────────────────────────────────────────────────────────┘
```

| 记忆类型 | 类比 | 数据量 | 生命周期 |
|---------|------|--------|---------|
| 短期记忆 | RAM | 当前对话 | 会话内 |
| 长期记忆 | SSD/硬盘 | 可无限扩展 | 跨会话 |
| 情景记忆 | 日记本 | 关键摘要 | 跨会话 |
| 用户画像 | 个人档案 | 结构化数据 | 跨会话 |

---

## Part 2：短期记忆——对话历史与上下文管理

### 对话记忆实现

```python
from dataclasses import dataclass
from datetime import datetime
from typing import List

@dataclass
class Message:
    role: str  # user / assistant / system
    content: str
    timestamp: datetime = datetime.now()

class ShortTermMemory:
    """短期记忆：对话历史管理"""

    def __init__(self, max_messages: int = 20):
        self.messages: List[Message] = []
        self.max_messages = max_messages

    def add(self, role: str, content: str):
        """添加消息"""
        self.messages.append(Message(role, content))
        if len(self.messages) > self.max_messages:
            self.compress()

    def compress(self):
        """压缩策略：当消息过多时，压缩历史"""
        if len(self.messages) <= self.max_messages:
            return

        # 保留系统消息和最近的对话
        system_msgs = [m for m in self.messages if m.role == "system"]
        recent = self.messages[-self.max_messages:]

        # 早期消息需要压缩为摘要
        if len(self.messages) > self.max_messages + 5:
            early = self.messages[:-self.max_messages]
            summary = self._generate_summary(early)
            self.messages = system_msgs + [Message("system", f"[早期对话摘要] {summary}")] + recent

    def get_context(self) -> List[dict]:
        """获取对话上下文"""
        return [
            {"role": m.role, "content": m.content}
            for m in self.messages
        ]
```

### 上下文窗口管理

**Prompt Engineering vs Context Engineering**

| 概念 | 关注点 | 比喻 |
|------|--------|------|
| Prompt Engineering | 静态指令 | 给 Agent 一本操作手册 |
| Context Engineering | 动态信息 | 确保手册的哪一页是打开的 |

**五种上下文类型：**

| 类型 | 说明 | 示例 |
|------|------|------|
| Instructions | 提示词、系统消息 | 系统提示词、角色设定 |
| Knowledge | 事实、数据库信息 | RAG 获取的知识 |
| Tools | 函数定义、API | 搜索工具、代码执行 |
| History | 对话历史 | 多轮对话记录 |
| Preferences | 用户偏好 | 语言风格、回答方式 |

---

## Part 3：长期记忆——向量数据库与语义检索

### Embedding 原理

```
文本: "我喜欢吃苹果"
    ↓ (Embedding 模型)
向量: [0.23, -0.45, 0.78, 0.12, ...]  (1536 维)

文本: "I like eating apples"
    ↓ (同一 Embedding 模型)
向量: [0.24, -0.44, 0.79, 0.11, ...]  (位置接近！)

文本: "今天天气很冷"
    ↓
向量: [-0.55, 0.12, 0.03, 0.88, ...]  (位置远离水果话题)
```

**核心原理：** 语义相近的文本，向量在空间中距离更近。

### ChromaDB 快速上手

```python
import chromadb
from chromadb.config import Settings

# 创建客户端
client = chromadb.PersistentClient(
    path="./chroma_memory",
    settings=Settings(allow_reset=True)
)

# 创建集合
collection = client.create_collection(
    name="agent_memory",
    metadata={"description": "Agent长期记忆库"}
)

# 添加文档
collection.add(
    documents=[
        "用户职业：数据分析师",
        "用户专长：Python、SQL、机器学习",
        "用户偏好：喜欢详细的技术解释"
    ],
    ids=["doc_1", "doc_2", "doc_3"]
)

# 检索
results = collection.query(
    query_texts=["用户会什么编程语言？"],
    n_results=2
)
print(results)
```

### 智谱 + ChromaDB 实战

```python
import os
import chromadb
from zhipuai import ZhipuAI

client = ZhipuAI(api_key=os.getenv("ZHIPU_API_KEY"))

class LongTermMemory:
    """基于 ChromaDB 的长期记忆"""

    def __init__(self, collection_name: str = "agent_memory"):
        self.client = chromadb.PersistentClient(path="./chroma_memory")
        self.collection = self.client.get_or_create_collection(name=collection_name)
        self.id_counter = 0

    def _embed(self, text: str) -> list:
        """调用智谱 API 生成向量"""
        response = client.embeddings.create(
            model="embedding-3",
            input=text,
        )
        return response.data[0].embedding

    def add_memory(self, content: str, memory_type: str = "general", importance: int = 5):
        """添加记忆"""
        self.id_counter += 1
        embedding = self._embed(content)
        self.collection.add(
            documents=[content],
            embeddings=[embedding],
            metadatas=[{"type": memory_type, "importance": importance}],
            ids=[f"mem_{self.id_counter}"]
        )

    def retrieve(self, query: str, top_k: int = 5) -> list:
        """语义检索"""
        query_embedding = self._embed(query)
        results = self.collection.query(
            query_embeddings=[query_embedding],
            n_results=top_k,
            include=["documents", "metadatas", "distances"]
        )

        memories = []
        if results["documents"] and results["documents"][0]:
            for i, doc in enumerate(results["documents"][0]):
                memories.append({
                    "content": doc,
                    "metadata": results["metadatas"][0][i],
                    "similarity": 1 - results["distances"][0][i],
                })
        return memories

# 使用
memory = LongTermMemory()
memory.add_memory("用户张伟是一名数据科学家", memory_type="profile", importance=9)
memory.add_memory("用户偏好简洁的回答", memory_type="preference", importance=8)

results = memory.retrieve("用户的技术背景是什么？")
for r in results:
    print(f"[{r['similarity']:.2f}] {r['content']}")
```

---

## Part 4：情景记忆与用户画像

### 用户画像数据结构

```python
from dataclasses import dataclass, asdict, field
from datetime import datetime
import json

@dataclass
class UserProfile:
    """用户画像"""
    user_id: str
    name: str = "未知"
    role: str = "未知"
    skills: list = field(default_factory=list)
    preferences: dict = field(default_factory=dict)
    goals: str = ""
    notes: str = ""
    created_at: str = field(default_factory=lambda: datetime.now().isoformat())

    def save(self, path: str = None):
        path = path or f"profile_{self.user_id}.json"
        with open(path, "w", encoding="utf-8") as f:
            json.dump(asdict(self), f, ensure_ascii=False, indent=2)

    @classmethod
    def load(cls, user_id: str) -> "UserProfile":
        path = f"profile_{user_id}.json"
        try:
            with open(path, "r", encoding="utf-8") as f:
                return cls(**json.load(f))
        except FileNotFoundError:
            return cls(user_id=user_id)

    def update(self, field: str, value):
        if hasattr(self, field):
            current = getattr(self, field)
            if isinstance(current, list) and isinstance(value, str):
                if value not in current:
                    current.append(value)
            elif isinstance(current, dict) and isinstance(value, dict):
                current.update(value)
            else:
                setattr(self, field, value)
            self.save()
```

### 情景记忆：重要对话摘要

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import List

@dataclass
class EpisodicMemory:
    """情景记忆：重要对话的摘要"""
    summary: str  # 摘要内容
    key_points: List[str]  # 关键要点
    decisions: List[str]  # 关键决策
    timestamp: str = field(default_factory=lambda: datetime.now().isoformat())
    confidence: float = 0.8

class EpisodicMemoryManager:
    """情景记忆管理器"""

    def __init__(self, llm_backend):
        self.llm = llm_backend
        self.memories: List[EpisodicMemory] = []

    def create_episode(self, conversation: List[dict], importance: float = 0.8) -> EpisodicMemory:
        """从对话中创建情景记忆"""
        prompt = f"""分析以下对话，生成情景记忆：
对话：
{self._format_conversation(conversation)}

请返回JSON格式：
{{
  "summary": "对话摘要（一句话）",
  "key_points": ["要点1", "要点2", "要点3"],
  "decisions": ["决策1", "决策2"]
}}"""

        result = self.llm.chat([{"role": "user", "content": prompt}])
        import json
        data = json.loads(result)

        episode = EpisodicMemory(
            summary=data.get("summary", ""),
            key_points=data.get("key_points", []),
            decisions=data.get("decisions", []),
            confidence=importance
        )
        self.memories.append(episode)
        return episode
```

---

## Part 5：综合实战——四层记忆 Agent

```python
"""
四层记忆 Agent
结合：短期记忆 + 长期记忆 + 情景记忆 + 用户画像
"""
import os
import json
from dataclasses import dataclass, asdict, field
from datetime import datetime
from typing import List
from zhipuai import ZhipuAI
import chromadb

client = ZhipuAI(api_key=os.getenv("ZHIPU_API_KEY"))

@dataclass
class UserProfile:
    user_id: str
    name: str = "未知"
    role: str = "未知"
    skills: list = field(default_factory=list)
    preferences: dict = field(default_factory=dict)
    goals: str = ""

    def save(self):
        with open(f"profile_{self.user_id}.json", "w", encoding="utf-8") as f:
            json.dump(asdict(self), f, ensure_ascii=False, indent=2)

    @classmethod
    def load(cls, user_id: str):
        try:
            with open(f"profile_{user_id}.json", "r", encoding="utf-8") as f:
                return cls(**json.load(f))
        except:
            return cls(user_id=user_id)

class FourLayerMemory:
    """四层记忆系统"""

    def __init__(self, user_id: str):
        self.user_id = user_id
        self.profile = UserProfile.load(user_id)
        self.short_term: List[dict] = []
        self.episodes: List[dict] = []
        self._init_vector_store()

    def _init_vector_store(self):
        self.chroma_client = chromadb.PersistentClient(path=f"./memory_{self.user_id}")
        try:
            self.long_term = self.chroma_client.get_or_create_collection("long_term_memory")
        except:
            self.chroma_client.delete_collection("long_term_memory")
            self.long_term = self.chroma_client.get_or_create_collection("long_term_memory")

    def _embed(self, text: str) -> list:
        response = client.embeddings.create(model="embedding-3", input=text)
        return response.data[0].embedding

    def add_short_term(self, role: str, content: str):
        self.short_term.append({"role": role, "content": content, "time": datetime.now().isoformat()})
        if len(self.short_term) > 20:
            self.short_term = self.short_term[-20:]

    def update_profile(self, field: str, value):
        if field == "skills" and value not in self.profile.skills:
            self.profile.skills.append(value)
        elif hasattr(self.profile, field):
            setattr(self.profile, field, value)
        self.profile.save()

    def add_long_term(self, content: str, memory_type: str = "general"):
        embedding = self._embed(content)
        self.long_term.add(
            documents=[content],
            embeddings=[embedding],
            metadatas=[{"type": memory_type}],
            ids=[f"lt_{datetime.now().timestamp()}"]
        )

    def search_long_term(self, query: str, top_k: int = 3) -> list:
        query_emb = self._embed(query)
        results = self.long_term.query(
            query_embeddings=[query_emb],
            n_results=top_k,
            include=["documents", "distances"]
        )
        if results["documents"] and results["documents"][0]:
            return [{"content": d, "similarity": 1 - dist}
                    for d, dist in zip(results["documents"][0], results["distances"][0])]
        return []

    def build_context(self, query: str) -> str:
        parts = ["=== 用户画像 ==="]
        parts.append(f"职业：{self.profile.role}")
        parts.append(f"技能：{', '.join(self.profile.skills) or '暂无'}")

        parts.append("\n=== 相关长期记忆 ===")
        for mem in self.search_long_term(query):
            parts.append(f"- {mem['content']}")

        parts.append("\n=== 当前对话 ===")
        for msg in self.short_term[-5:]:
            parts.append(f"{msg['role']}: {msg['content'][:50]}...")

        return "\n".join(parts)

class MemoryAgent:
    """四层记忆 Agent"""

    SYSTEM_PROMPT = """你是一个专业的 AI 助手，擅长记住用户的重要信息。"""

    def __init__(self, user_id: str):
        self.memory = FourLayerMemory(user_id)
        self.memory_counter = 0

    def chat(self, user_input: str) -> str:
        context = self.memory.build_context(user_input)

        messages = [
            {"role": "system", "content": self.SYSTEM_PROMPT},
            {"role": "system", "content": f"\n用户背景：\n{context}"},
            {"role": "user", "content": user_input}
        ]

        response = client.chat.completions.create(
            model="glm-4-flash",
            messages=messages
        )
        answer = response.choices[0].message.content

        self.memory.add_short_term("user", user_input)
        self.memory.add_short_term("assistant", answer)

        return answer

# 使用示例
if __name__ == "__main__":
    agent = MemoryAgent(user_id="demo_user")

    conversation = [
        "你好，我叫陈伟，是一名全栈工程师",
        "我主要做 Python 后端和 React 前端",
        "现在想学习 AI Agent 开发",
        "我偏好边做项目边学习"
    ]

    for user_input in conversation:
        print(f"[用户]: {user_input}")
        response = agent.chat(user_input)
        print(f"[助手]: {response}")

    print(f"\n用户技能: {agent.memory.profile.skills}")
```

---

## 今日要点

- ✅ 理解四层记忆架构：短期 / 长期 / 情景 / 用户画像
- ✅ 掌握短期记忆：对话历史管理、上下文压缩
- ✅ 掌握长期记忆：ChromaDB 向量存储、语义检索
- ✅ 掌握用户画像：结构化存储、持久化
- ✅ 实现四层记忆 Agent，跨会话积累知识

---

## 自测题

1. 四层记忆架构分别对应什么？各自的作用是什么？
2. 为什么模型本身没有记忆？记忆从哪里来？
3. 短期记忆的压缩策略是什么？
4. 向量数据库检索的原理是什么？

---

## 动手作业

1. 实现"记忆过期"机制：超过一定时间未访问的记忆自动降级或删除
2. 给四层记忆 Agent 添加"记忆搜索"功能
3. 设计一个"学习进度跟踪 Agent"，记住用户学过什么、还差什么
