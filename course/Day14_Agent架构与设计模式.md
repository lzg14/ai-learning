# Day 15：Agent 架构与设计模式

> Agent 的架构决定它能处理多复杂的任务。本课学习主流 Agent 设计模式：ReAct / Plan-and-Execute / 多 Agent 协作。

## Part 1：主流 Agent 模式解析

### Agent 模式对比

| 模式 | 特点 | 适用场景 | 代表框架 |
|------|------|----------|----------|
| **ReAct** | 边推理边行动，循环 | 简单到中等复杂任务 | LangChain Agent |
| **Plan-and-Execute** | 先规划后执行 | 复杂多步骤任务 | LangChain AgentExecutor |
| **AutoGPT** | 自主拆解任务 | 开放式目标 | AutoGPT / GPT Engineer |
| **BabyAGI** | 目标驱动任务生成 | 持续性目标 | BabyAGI |
| **多 Agent 协作** | 多个专业 Agent 分工 | 复杂系统 | MetaGPT / ChatDev |

### ReAct 模式详解

```
ReAct = Reasoning + Acting（推理 + 行动）

循环过程：
1. 推理（Thought）：思考下一步该做什么
2. 行动（Action）：执行工具或调用函数
3. 观察（Observation）：获取行动结果
4. 回到第 1 步，继续循环直到完成任务
```

---

## Part 2：Plan-and-Execute 模式

### 核心思路

```
传统模式：边想边做
目标 → 推理 → 行动 → 推理 → 行动 → ... → 结果

Plan-and-Execute：先想好再做
目标 → 制定完整计划 → 按计划执行 → 结果
```

### 代码实现

```python
"""
Plan-and-Execute 模式
先规划，后执行
"""
import os
from dotenv import load_dotenv
from langchain_community.chat_models import ChatZhipuAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import JsonOutputParser

load_dotenv()

llm = ChatZhipuAI(
    api_key=os.getenv("ZHIPU_API_KEY"),
    model="glm-4-flash",
    temperature=0.0
)

# ========== 第一步：规划器 ==========

planner_prompt = ChatPromptTemplate.from_messages([
    ("system", """你是一个任务规划专家。
用户提出目标后，你需要：
1. 分析目标
2. 将目标拆解成 3-7 个具体步骤
3. 说明每个步骤需要什么工具或信息

输出 JSON 格式：
{"goal": "目标", "steps": [{"step": 1, "task": "任务描述", "tool": "需要的工具"}]}
"""),
    ("human", "{goal}")
])

planner = planner_prompt | llm | JsonOutputParser()

# ========== 第二步：执行器 ==========

executor_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个任务执行助手，按照步骤完成任务。"),
    ("human", "请执行以下步骤：\n{current_step}")
])

def execute_step(step):
    chain = executor_prompt | llm
    return chain.invoke({"current_step": step["task"]})

# ========== 第三步：Plan-and-Execute 主循环 ==========

def plan_and_execute(goal):
    """Plan-and-Execute 主函数"""
    print(f"🎯 目标：{goal}\n")

    # 1. 规划
    print("📋 制定计划中...")
    plan = planner.invoke({"goal": goal})
    print(f"计划：{len(plan['steps'])} 个步骤\n")

    for s in plan["steps"]:
        print(f"  Step {s['step']}：{s['task']} (需要工具: {s.get('tool', '无')})")

    print("\n" + "="*50)
    print("🚀 开始执行...\n")

    # 2. 按顺序执行
    results = []
    for step in plan["steps"]:
        print(f"执行 Step {step['step']}：{step['task']}")
        result = execute_step(step)
        print(f"  结果：{result.content}\n")
        results.append({"step": step["step"], "result": result.content})

    return results

# 测试
results = plan_and_execute("帮我分析一下 Python 和 JavaScript 两个语言的区别，并推荐入门选择")
```

### Plan-and-Execute vs ReAct

| 对比 | Plan-and-Execute | ReAct |
|------|-----------------|-------|
| 决策时机 | 先规划后执行 | 边做边想 |
| 透明度 | 计划预先可见 | 逐步推理可见 |
| 适合任务 | 步骤明确的多步骤任务 | 探索性、动态决策 |
| token 消耗 | 略高（先规划） | 较低（实时推理） |
| 错误恢复 | 计划失败需重新规划 | 单步失败可重试 |

---

## Part 3：多 Agent 协作架构

### 三种架构模式

**模式一：层级式（Hierarchical）**

```
Manager Agent（管理者）
    ├── Researcher Agent（研究）
    ├── Coder Agent（编码）
    └── Reviewer Agent（审查）
```

**模式二：对等式（Peer-to-Peer）**

```
Agent A ↔ Agent B ↔ Agent C
（通过共享消息总线协作）
```

**模式三：流水线式（Pipeline）**

```
输入 → Agent1 → Agent2 → Agent3 → 输出
        ↓          ↓         ↓
     临时结果    临时结果   最终结果
```

### 消息总线实现

```python
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class AgentMessage:
    """Agent 消息"""
    sender: str           # 发送者名称
    receiver: str         # 接收者名称（"broadcast" 表示广播）
    content: str          # 消息内容
    message_type: str     # "task" / "result" / "query" / "response"
    metadata: dict = None # 附加信息

class MessageBus:
    """Agent 消息总线"""

    def __init__(self):
        self.messages = []
        self.subscribers = {}

    def publish(self, message: AgentMessage):
        """发布消息"""
        self.messages.append(message)
        print(f"📨 [{message.sender}] → [{message.receiver}]: {message.content[:50]}...")

    def subscribe(self, agent_name: str, callback):
        """订阅消息"""
        self.subscribers[agent_name] = callback

    def get_messages_for(self, agent_name: str):
        """获取某 Agent 的消息"""
        return [
            m for m in self.messages
            if m.receiver == agent_name or m.receiver == "broadcast"
        ]
```

### 多 Agent 示例

```python
class ResearcherAgent:
    """研究 Agent"""
    def __init__(self, llm, message_bus):
        self.llm = llm
        self.bus = message_bus

    def research(self, topic):
        prompt = f"请详细研究以下主题，给出要点：{topic}"
        result = self.llm.invoke(prompt)

        self.bus.publish(AgentMessage(
            sender="Researcher",
            receiver="Coder",
            content=f"研究完成，结果：{result.content}",
            message_type="result"
        ))
        return result.content

class CoderAgent:
    """编码 Agent"""
    def __init__(self, llm, message_bus):
        self.llm = llm
        self.bus = message_bus

    def code(self, task):
        prompt = f"根据以下任务编写代码：{task}"
        result = self.llm.invoke(prompt)
        return result.content

# 协作执行
bus = MessageBus()
researcher = ResearcherAgent(llm, bus)
coder = CoderAgent(llm, bus)

topic_result = researcher.research("RAG 原理")
```

---

## Part 4：Agent 架构的常见坑

### 坑 1：任务太开放

```
❌ 用户："帮我做一个 SaaS 产品" → Agent 卡住了
   Agent 不知道什么算"完成"，也不知道先做什么后做什么
```

**解决：** 所有 Agent 任务必须有明确的停止条件和验收标准。

### 坑 2：循环调用，死循环

```
❌ Agent: 我需要搜索"Python" → 搜索工具返回结果
   Agent: 我需要搜索更多"Python"信息 → 再搜一次
   Agent: 我需要再搜一次"Python"确认 → 又搜一次
   ... 无限循环，烧了几百块
```

**解决：** 必须加最大步数限制（max_steps = 10），到点强制停止。

### 坑 3：工具太多，模型看花眼

```
❌ 给 Agent 20 个工具 → 它不知道该用哪个
   经常选错工具，或者选了对的但参数传错
```

**解决：** 一次不要给超过 5 个工具，根据上下文动态提供。

### Agent 的"公司组织"类比

| Agent 架构 | 相当于什么公司 | 优点 | 缺点 |
|------------|---------------|------|------|
| **单 Agent** | 一人小公司 | 简单、灵活 | 什么都做不精 |
| **Plan-and-Execute** | 瀑布式开发 | 计划清晰、可预测 | 中途变了需要重新规划 |
| **多 Agent 层级式** | 传统大公司 | 专业分工、各司其职 | 沟通成本高、官僚 |
| **多 Agent 对等协作** | 创业团队 | 灵活、创造力强 | 容易乱、没人拍板 |

> 💡 不是越复杂越好。90% 的场景，一个稍微聪明点的单 Agent 就够了。

---

## 📌 今日要点

- ✅ 理解了主流 Agent 模式（ReAct / Plan-and-Execute / AutoGPT / 多 Agent 协作）
- ✅ 掌握了 Plan-and-Execute 架构（先规划后执行）
- ✅ 理解了多 Agent 协作的三种架构（层级/对等/流水线）
- ✅ 实现了消息总线（Agent 间通信）
- ✅ 认识了 Agent 架构的常见坑及解决方案

---

## 自测题

1. Plan-and-Execute 和 ReAct 模式最大的区别是什么？
2. 多 Agent 协作的三种架构模式是什么？
3. 消息总线（MessageBus）在多 Agent 系统中的作用是什么？
4. Agent 为什么会陷入死循环？如何避免？

---

## 动手作业

1. 用 Plan-and-Execute 模式实现一个任务规划 Agent
2. 设计一个"论文写作团队"多 Agent 系统（researcher / writer / reviewer）
3. 给 Agent 添加最大步数限制，防止死循环
