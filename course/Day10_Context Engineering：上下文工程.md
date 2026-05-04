# Day 10：Context Engineering——上下文工程

> Prompt Engineering 是"写指令"，Context Engineering 是"管信息"。本课学习如何让 Agent 在动态、复杂的环境中管理上下文。

## Part 1：为什么需要 Context Engineering？

### Prompt Engineering 的局限

Prompt Engineering 关注的是**静态指令设计**——给定一个系统角色、几条规则、几个示例，一次设定终身使用。但 Agent 面临的真实世界是动态的：

```
用户：查一下张三的订单
Agent：好的，张三有 3 个订单...

用户：最新的那个什么时候发货？
Agent：需要知道"最新的" = 刚刚讨论过的第 3 个订单
        但模型已经忘了"张三"是谁了...
```

**问题：** Agent 的上下文是随时间变化、随交互增加的。Prompt Engineering 管不了这个。

### Context Engineering 的定义

**Context Engineering（上下文工程）** = 设计和管理 Agent 在运行过程中需要的所有动态信息。

| 对比 | Prompt Engineering | Context Engineering |
|------|-------------------|-------------------|
| **焦点** | 静态指令设计 | 动态信息管理 |
| **变更频率** | 一次设定，几乎不变 | 每次交互都在变 |
| **管理内容** | 角色、规则、格式 | 对话历史、检索结果、工具输出 |
| **核心挑战** | 写清楚 | 选什么、怎么压缩、怎么隔离 |
| **失败模式** | 指令不清晰 | 信息过载/遗漏/混淆 |

> 好的 Prompt 让 Agent 知道"怎么做事"，好的 Context 让 Agent 知道"事做到哪了"。

### 图书馆员的类比

想象你是一个图书馆员（Agent），一个读者（用户）来找你查资料：

- **Prompt Engineering = 你的职业素养**：知道怎么分类、怎么检索、怎么回答
- **Context Engineering = 你对当前读者的记忆**：他刚才问了什么、已经查到了什么、对什么方向感兴趣

没有好的 Context Engineering，Agent 每次都要"重新认识用户"。

---

## Part 2：上下文的四种类型

Agent 需要管理的上下文来自四个不同来源：

| 类型 | 来源 | 示例 | 管理难度 |
|------|------|------|----------|
| **指令上下文** | System Prompt、规则 | 角色设定、输出格式 | ⭐ 低 |
| **对话上下文** | 用户和 Agent 的历史对话 | 之前问了什么、答了什么 | ⭐⭐ 中 |
| **数据上下文** | 检索到的文档、工具返回 | RAG 片段、API 结果 | ⭐⭐⭐ 高 |
| **工具上下文** | 已调用的工具及其状态 | 哪些工具已执行、结果是什么 | ⭐⭐ 中 |

### 上下文管理矩阵

```python
class ContextManager:
    """上下文管理器"""

    def __init__(self):
        self.instruction_context = []
        self.dialogue_context = []
        self.data_context = []
        self.tool_context = []

    def add_instruction(self, instruction: str):
        """添加指令"""
        self.instruction_context.append({
            "content": instruction,
            "priority": "high",
            "immutable": True  # 指令不可变
        })

    def add_dialogue(self, role: str, content: str):
        """添加对话"""
        self.dialogue_context.append({
            "role": role,
            "content": content,
            "timestamp": len(self.dialogue_context)
        })

    def add_data(self, source: str, content: str, relevance: float):
        """添加数据"""
        self.data_context.append({
            "source": source,
            "content": content,
            "relevance": relevance,
            "timestamp": len(self.data_context)
        })

    def add_tool_result(self, tool: str, args: dict, result: str):
        """添加工具结果"""
        self.tool_context.append({
            "tool": tool,
            "args": args,
            "result": result,
            "timestamp": len(self.tool_context)
        })

    def build_context(self, max_tokens: int = 4000) -> str:
        """构建最终上下文"""
        parts = []

        # 1. 指令上下文（始终包含）
        for inst in self.instruction_context:
            parts.append(f"[指令] {inst['content']}")

        # 2. 最近的对话（保留最后 5 轮）
        recent = self.dialogue_context[-10:] if len(self.dialogue_context) > 10 else self.dialogue_context
        for d in recent:
            parts.append(f"[{d['role']}] {d['content'][:200]}")

        # 3. 高相关性数据
        data_sorted = sorted(self.data_context, key=lambda x: x["relevance"], reverse=True)
        for d in data_sorted[:3]:
            parts.append(f"[数据:{d['source']}] {d['content'][:300]}")

        # 4. 最近的工具结果
        for t in self.tool_context[-3:]:
            parts.append(f"[工具:{t['tool']}] {t['result'][:200]}")

        return "\n\n".join(parts)
```

---

## Part 3：上下文选择策略

不是所有上下文都重要，关键是要**选对**。

### Token 预算分配

```python
class TokenBudget:
    """Token 预算分配器"""

    def __init__(self, total_budget: int = 4000):
        self.total = total_budget
        self.allocations = {
            "system": 500,        # System Prompt
            "recent_dialogue": 1000,  # 最近对话
            "retrieved_data": 1500,   # 检索数据
            "tool_results": 500,      # 工具结果
            "user_input": 500,        # 当前用户输入
        }

    def allocate(self, context_type: str, tokens_needed: int) -> int:
        """分配 Token，返回实际可用的 Token 数"""
        budget = self.allocations.get(context_type, 100)
        allocated = min(budget, tokens_needed)
        print(f"{context_type}: 申请 {tokens_needed} Token，分配 {allocated} Token")
        return allocated

    def dynamic_allocate(self, context_type: str, tokens_needed: int, priority: int):
        """动态分配，可以借用其他类型的预算"""
        budget = self.allocations.get(context_type, 100)
        if tokens_needed <= budget:
            return tokens_needed

        # 高优先级可以借用低优先级的预算
        shortage = tokens_needed - budget
        if priority >= 4:  # 高优先级
            for other_type, other_budget in sorted(
                self.allocations.items(),
                key=lambda x: {"system": 5, "user_input": 4,
                              "retrieved_data": 3, "recent_dialogue": 2,
                              "tool_results": 1}.get(x[0], 0)
            ):
                if other_type == context_type:
                    continue
                if other_budget > 100:
                    borrow = min(shortage, other_budget - 100)
                    self.allocations[other_type] -= borrow
                    shortage -= borrow
                    print(f"  从 {other_type} 借调 {borrow} Token")
                    if shortage <= 0:
                        break

        return tokens_needed - shortage
```

### 优先级队列

```python
from heapq import heappush, heappop

class ContextPriorityQueue:
    """上下文优先级队列"""

    def __init__(self):
        self.queue = []  # (priority, timestamp, content)

    def add(self, content: str, priority: int):
        """添加上下文，priority 越高越重要"""
        heappush(self.queue, (-priority, len(self.queue), content))

    def get_top_k(self, k: int) -> list:
        """获取最重要的 K 条"""
        result = []
        temp = []
        for _ in range(min(k, len(self.queue))):
            neg_prio, ts, content = heappop(self.queue)
            result.append(content)
            temp.append((neg_prio, ts, content))
        # 恢复队列
        for item in temp:
            heappush(self.queue, item)
        return result
```

---

## Part 4：上下文压缩技术

当 Token 预算不够时，需要压缩。

### 三种压缩方法

| 方法 | 压缩率 | 信息损失 | 实现复杂度 |
|------|--------|----------|-----------|
| **摘要压缩** | 50-70% | 中 | ⭐⭐ |
| **关键信息提取** | 70-90% | 低 | ⭐⭐⭐ |
| **结构化表示** | 80-95% | 低 | ⭐⭐⭐⭐ |

### 关键信息提取实现

```python
def extract_key_information(context: str, llm) -> str:
    """从上下文中提取关键信息"""
    prompt = f"""分析以下上下文，只提取关键信息。

上下文：
{context}

请提取：
1. 实体（人名、地名、组织等）
2. 事实性陈述（已经确认的信息）
3. 待办事项（需要继续处理的事情）
4. 用户偏好（用户提到的喜好、习惯）

如果信息为空，返回"无"。

格式：
实体：...
事实：...
待办：...
偏好：...
"""
    result = llm.invoke(prompt)
    return result.content

def compress_dialogue_history(messages: list, llm, max_tokens: int = 1000) -> list:
    """压缩对话历史"""
    system = messages[0] if messages[0]["role"] == "system" else None
    recent = messages[-6:]  # 保留最近 3 轮
    early = messages[1:-6] if len(messages) > 6 else []

    if early:
        early_text = "\n".join([f"{m['role']}: {m['content']}" for m in early])
        summary = extract_key_information(early_text, llm)

        compressed = [
            {"role": "system", "content": f"{system['content'] if system else ''}\n\n[历史摘要] {summary}"},
            *recent
        ]
        return compressed

    return messages
```

---

## Part 5：上下文隔离

不同用户、不同会话之间的上下文不能互相污染。

### 上下文隔离模型

```python
class IsolatedContext:
    """隔离的上下文容器"""

    def __init__(self, session_id: str):
        self.session_id = session_id
        self.data = {}

    def set(self, key: str, value: str):
        self.data[key] = value

    def get(self, key: str) -> str:
        return self.data.get(key, "")

    def clear(self):
        self.data.clear()


class ContextIsolationManager:
    """上下文隔离管理器"""

    def __init__(self):
        self.sessions = {}  # {session_id: IsolatedContext}

    def get_session(self, session_id: str) -> IsolatedContext:
        """获取或创建会话上下文"""
        if session_id not in self.sessions:
            self.sessions[session_id] = IsolatedContext(session_id)
        return self.sessions[session_id]

    def switch_session(self, session_id: str):
        """切换会话"""
        return self.get_session(session_id)

    def cleanup_session(self, session_id: str, max_age_hours: int = 24):
        """清理过期会话"""
        if session_id in self.sessions:
            del self.sessions[session_id]
            print(f"已清理会话 {session_id}")
```

### 多租户隔离

```python
class TenantContext:
    """多租户上下文"""

    def __init__(self, tenant_id: str):
        self.tenant_id = tenant_id
        self.system_prompt = self._load_tenant_prompt()

    def _load_tenant_prompt(self) -> str:
        """加载租户专属的 System Prompt"""
        prompts = {
            "tenant_a": "你是 Tenant A 的客服助手，只能回答关于产品 A 的问题。",
            "tenant_b": "你是 Tenant B 的客服助手，只能回答关于产品 B 的问题。",
        }
        return prompts.get(self.tenant_id, "你是通用助手。")

    def build_messages(self, user_input: str) -> list:
        """构建隔离后的 messages"""
        return [
            {"role": "system", "content": self.system_prompt},
            {"role": "user", "content": user_input}
        ]

# 每个租户独立
tenant_a = TenantContext("tenant_a")
tenant_b = TenantContext("tenant_b")

# 即使相同输入，系统 Prompt 也不同
msgs_a = tenant_a.build_messages("帮我查一下价格")
msgs_b = tenant_b.build_messages("帮我查一下价格")
```

---

## Part 6：常见上下文故障与缓解

| 故障类型 | 表现 | 原因 | 解决方法 |
|----------|------|------|----------|
| **上下文中毒** | Agent 被外部内容误导 | Agent 读到了带恶意指令的外部数据 | 输入隔离 + 清洗 |
| **上下文分心** | Agent 关注了无关信息 | 检索结果太多，关键信息被淹没 | 相关性排序 + Token 预算 |
| **上下文混淆** | Agent 把不同用户的数据混在一起 | 会话隔离不当 | 严格的多租户隔离 |
| **上下文冲突** | 新旧信息矛盾 | 过时信息和最新信息同时存在 | 时间戳 + 版本管理 |

### 故障检测与修复

```python
class ContextHealthChecker:
    """上下文健康检查"""

    def __init__(self):
        self.checks = []

    def check_too_long(self, context: str, max_tokens: int = 4000) -> bool:
        """检查是否超出 Token 限制"""
        token_count = len(context)  # 简化估算
        if token_count > max_tokens:
            self.checks.append({
                "type": "TOO_LONG",
                "message": f"上下文过长：{token_count} > {max_tokens} Token"
            })
            return False
        return True

    def check_contradiction(self, context: str) -> list:
        """检查上下文中的矛盾"""
        contradictions = []
        # 简单的矛盾检测
        patterns = [
            ("是", "不是"),
            ("可以", "不可以"),
            ("有", "没有"),
        ]
        for a, b in patterns:
            if a in context and b in context:
                contradictions.append({
                    "type": "CONTRADICTION",
                    "message": f"检测到矛盾：同时出现 '{a}' 和 '{b}'"
                })
        return contradictions

    def check_redundancy(self, context: str) -> int:
        """检查冗余度"""
        sentences = context.split("。")
        unique = set(s.strip() for s in sentences if s.strip())
        redundancy = 1 - (len(unique) / max(len(sentences), 1))
        if redundancy > 0.3:
            self.checks.append({
                "type": "REDUNDANCY",
                "message": f"冗余度 {redundancy:.0%}，建议压缩"
            })
        return redundancy

    def report(self) -> list:
        """生成健康报告"""
        return self.checks
```

---

## 今日要点

- 理解了 Context Engineering 与 Prompt Engineering 的区别——静态指令 vs 动态信息管理
- 掌握了上下文的四种类型：指令、对话、数据、工具
- 学会了上下文选择策略：Token 预算分配、优先级队列
- 掌握了上下文压缩技术：摘要、关键信息提取、结构化表示
- 理解了上下文隔离的重要性：会话隔离、多租户隔离
- 熟悉了常见上下文故障及其缓解方法

---

## 自测题

1. Context Engineering 和 Prompt Engineering 的核心区别是什么？
2. 上下文的四种类型是什么？各有什么管理难度？
3. Token 预算分配的核心思路是什么？
4. 上下文隔离的两种模式是什么？分别适合什么场景？
5. 上下文中毒和上下文混淆的区别是什么？

---

## 动手作业

1. 实现一个 Token 预算分配器，为不同类型的上下文分配不同的 Token 额度
2. 写一个对话历史压缩函数，支持摘要压缩和关键信息提取
3. 实现一个多租户上下文隔离管理器