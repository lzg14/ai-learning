# Day 15：A2A 协议与 Agent 间通信标准

> MCP 让 Agent 能**用工具**，A2A 让 Agent 能**互相交流**。本课学习 Google 的 Agent-to-Agent 协议，理解 Agent 间通信的核心模式。

## Part 1：为什么需要 A2A？

### 从 MCP 到 A2A

回顾整个课程的技术演进：

```
Day06  MCP  →  Agent 与工具的通信标准
Day14  Agent 架构  →  单个 Agent 的内部设计
Day15  A2A  →  Agent 与 Agent 的通信标准
Day20  多 Agent 协作  →  多 Agent 系统设计
```

- **MCP（Model Context Protocol）**：Agent 如何调用工具、访问数据
- **A2A（Agent-to-Agent）**：Agent 如何与其他 Agent 协作完成任务

### 为什么要让 Agent 互相通信？

**场景：用户想预订一次商务旅行**

```
传统方式：一个 Agent 做所有事
Agent → 查航班 API → 查酒店 API → 查天气 API → 生成行程
问题：一个 Agent 需要所有工具的权限、所有领域的知识

A2A 方式：多个 Agent 协作
Travel Agent  ←→  Flight Agent（专业订票）
             ←→  Hotel Agent（专业订房）
             ←→  Weather Agent（专业查天气）
优势：每个 Agent 只做自己最擅长的事
```

### A2A 协议的核心价值

| 价值 | 说明 |
|------|------|
| **互操作性** | 不同厂商的 Agent 可以互相通信 |
| **专业化** | Agent 可以专注于自己的领域 |
| **可组合性** | 像搭积木一样组合多个 Agent |
| **松耦合** | Agent 之间不依赖对方的内部实现 |
| **可发现** | Agent 能自动发现其他 Agent 的能力 |

### 历史背景

| 时间 | 事件 |
|------|------|
| 2024 年 11 月 | MCP 协议开源（Anthropic） |
| 2025 年 4 月 | A2A 协议发布（Google） |
| 2025 年 12 月 | MCP 捐赠给 Linux Foundation Agentic AI Foundation |
| 2026 年初 | A2A 捐赠给 Linux Foundation，150+ 组织支持 |

> 两个协议最终都归于同一基金会管理，标志着 Agent 通信标准走向统一。

---

## Part 2：A2A 协议核心概念

### 整体架构

```plaintext
┌────────────────────────────────────────────┐
│              客户端 Agent                    │
│  (发起任务、接收结果)                        │
└──────────────────┬─────────────────────────┘
                   │ A2A 协议 (HTTP/JSON)
    ┌──────────────┼──────────────┬──────────┐
    ▼              ▼              ▼          ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│ 服务Agent│ │ 服务Agent│ │ 服务Agent│   ...
│ 航班专家  │ │ 酒店专家  │ │ 天气专家  │
└─────────┘ └─────────┘ └─────────┘
```

### 四大核心组件

| 组件 | 说明 | 类比 |
|------|------|------|
| **Agent Card** | Agent 的"名片"，描述能力、接口 | 餐馆菜单 |
| **A2A 请求/响应** | 标准化的通信格式 | 点餐流程 |
| **任务管理** | 跟踪任务状态（进行中/已完成/失败） | 订单追踪 |
| **推送通知** | 服务端主动推送更新 | 叫号系统 |

### Agent Card

Agent Card 是一个 JSON 文件，描述了 Agent 的能力和如何与它通信：

```json
{
  "name": "Flight Booking Agent",
  "description": "专业航班查询与预订 Agent",
  "url": "https://api.example.com/a2a",
  "version": "1.0.0",
  "capabilities": {
    "skills": [
      {
        "id": "search_flights",
        "name": "航班搜索",
        "description": "根据日期和目的地搜索航班",
        "input_schema": {
          "type": "object",
          "properties": {
            "origin": {"type": "string", "description": "出发城市"},
            "destination": {"type": "string", "description": "到达城市"},
            "date": {"type": "string", "description": "出发日期"}
          },
          "required": ["origin", "destination", "date"]
        }
      },
      {
        "id": "book_flight",
        "name": "预订航班",
        "description": "预订指定航班",
        "input_schema": {
          "type": "object",
          "properties": {
            "flight_id": {"type": "string"},
            "passenger_name": {"type": "string"}
          },
          "required": ["flight_id", "passenger_name"]
        }
      }
    ]
  },
  "authentication": {
    "scheme": "bearer",
    "credentials_url": "https://auth.example.com/token"
  }
}
```

### Agent Card 发现机制

```python
import httpx
from typing import Optional, Dict, Any

class AgentCardClient:
    """Agent Card 客户端——发现和解析 Agent Card"""

    def __init__(self):
        self.cache = {}

    async def discover(self, agent_url: str) -> Optional[Dict[str, Any]]:
        """发现 Agent 的能力"""
        if agent_url in self.cache:
            return self.cache[agent_url]

        try:
            async with httpx.AsyncClient() as client:
                # 标准 Agent Card 路径
                response = await client.get(
                    f"{agent_url}/.well-known/agent-card.json",
                    timeout=10
                )
                response.raise_for_status()
                card = response.json()
                self.cache[agent_url] = card
                return card
        except Exception as e:
            print(f"发现 Agent {agent_url} 失败: {e}")
            return None

    def find_skill(self, card: Dict, skill_id: str) -> Optional[Dict]:
        """在 Agent Card 中查找特定技能"""
        for skill in card.get("capabilities", {}).get("skills", []):
            if skill["id"] == skill_id:
                return skill
        return None

    def match_skills(self, card: Dict, query: str) -> list:
        """根据查询匹配 Agent 的技能"""
        skills = card.get("capabilities", {}).get("skills", [])
        query_lower = query.lower()
        matches = []
        for skill in skills:
            if (query_lower in skill["name"].lower() or
                query_lower in skill["description"].lower()):
                matches.append(skill)
        return matches

# 使用示例
async def main():
    client = AgentCardClient()

    # 发现航班 Agent
    card = await client.discover("https://flight-agent.example.com")
    if card:
        print(f"发现 Agent: {card['name']}")
        print(f"能力: {[s['name'] for s in card['capabilities']['skills']]}")

        # 匹配技能
        matches = client.match_skills(card, "航班")
        print(f"匹配的技能: {[m['name'] for m in matches]}")
```

---

## Part 3：A2A 请求与响应

### 任务生命周期

```
发起任务（send_task）
    │
    ├─→ completed（同步完成）
    ├─→ working（异步进行中）→ ... → completed
    ├─→ input_required（需要更多信息）
    └─→ failed（失败）
```

### A2A 消息格式

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional, List
import json
import uuid
from datetime import datetime


class TaskStatus(str, Enum):
    SUBMITTED = "submitted"       # 已提交
    WORKING = "working"           # 处理中
    INPUT_REQUIRED = "input_required"  # 需要更多输入
    COMPLETED = "completed"       # 完成
    FAILED = "failed"             # 失败
    CANCELED = "canceled"         # 已取消


@dataclass
class A2AMessage:
    """A2A 消息"""
    role: str                      # "agent" 或 "user"
    content: str
    metadata: dict = field(default_factory=dict)


@dataclass
class A2ATask:
    """A2A 任务"""
    id: str = field(default_factory=lambda: f"task-{uuid.uuid4().hex[:8]}")
    status: TaskStatus = TaskStatus.SUBMITTED
    messages: List[A2AMessage] = field(default_factory=list)
    artifacts: List[dict] = field(default_factory=list)  # 输出产物

    def add_message(self, role: str, content: str, **metadata):
        self.messages.append(A2AMessage(role=role, content=content, metadata=metadata))

    def to_dict(self) -> dict:
        return {
            "id": self.id,
            "status": self.status.value,
            "messages": [
                {"role": m.role, "content": m.content, **m.metadata}
                for m in self.messages
            ],
            "artifacts": self.artifacts
        }


class A2ARequest:
    """A2A 请求构建器"""

    @staticmethod
    def send_task(target_agent: str, task: A2ATask) -> dict:
        """构建 send_task 请求"""
        return {
            "jsonrpc": "2.0",
            "method": "tasks/send",
            "params": {
                "id": task.id,
                "message": {
                    "role": "agent",
                    "content": task.messages[-1].content if task.messages else "",
                    "metadata": {}
                }
            },
            "id": 1
        }

    @staticmethod
    def get_task_status(target_agent: str, task_id: str) -> dict:
        """查询任务状态"""
        return {
            "jsonrpc": "2.0",
            "method": "tasks/get",
            "params": {"id": task_id},
            "id": 2
        }

    @staticmethod
    def cancel_task(target_agent: str, task_id: str) -> dict:
        """取消任务"""
        return {
            "jsonrpc": "2.0",
            "method": "tasks/cancel",
            "params": {"id": task_id},
            "id": 3
        }
```

### A2A 服务器端实现

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI(title="A2A Agent Server")


class A2ARequestModel(BaseModel):
    """A2A 请求模型"""
    jsonrpc: str = "2.0"
    method: str
    params: dict
    id: int


class SimpleAgent:
    """最简单的 A2A Agent"""

    def __init__(self, name: str, skills: list):
        self.name = name
        self.skills = skills
        self.tasks = {}

    def handle_task(self, task_id: str, message: dict) -> dict:
        """处理任务"""
        content = message.get("content", "")

        # 简单的回复逻辑
        if "hello" in content.lower():
            reply = f"你好！我是 {self.name}，有什么可以帮你的？"
        elif "skill" in content.lower():
            skill_names = [s["name"] for s in self.skills]
            reply = f"我的技能包括：{', '.join(skill_names)}"
        else:
            reply = f"收到消息：{content}"

        return {
            "id": task_id,
            "status": "completed",
            "messages": [
                {"role": "agent", "content": reply}
            ],
            "artifacts": []
        }


# 实例化一个简单的 Agent
weather_agent = SimpleAgent("天气助手", [
    {"id": "get_weather", "name": "查询天气"}
])


@app.post("/a2a")
async def handle_a2a(request: A2ARequestModel):
    """处理 A2A 请求"""
    if request.method == "tasks/send":
        task_id = request.params.get("id", "unknown")
        message = request.params.get("message", {})
        result = weather_agent.handle_task(task_id, message)

        return {
            "jsonrpc": "2.0",
            "result": result,
            "id": request.id
        }

    elif request.method == "tasks/get":
        task_id = request.params.get("id", "")
        task = weather_agent.tasks.get(task_id)
        if not task:
            raise HTTPException(status_code=404, detail="Task not found")
        return {
            "jsonrpc": "2.0",
            "result": task,
            "id": request.id
        }

    raise HTTPException(status_code=400, detail=f"Unknown method: {request.method}")


@app.get("/.well-known/agent-card.json")
async def get_agent_card():
    """返回 Agent Card"""
    return {
        "name": weather_agent.name,
        "description": "一个简单的天气查询 Agent",
        "url": "http://localhost:8000/a2a",
        "version": "1.0.0",
        "capabilities": {
            "skills": [
                {
                    "id": "get_weather",
                    "name": "查询天气",
                    "description": "查询指定城市的天气",
                    "input_schema": {
                        "type": "object",
                        "properties": {
                            "city": {"type": "string"}
                        },
                        "required": ["city"]
                    }
                }
            ]
        }
    }
```

---

## Part 4：A2A 客户端实现

### 通用 A2A 客户端

```python
import httpx
import json
from typing import Optional, Dict, Any
from datetime import datetime


class A2AClient:
    """A2A 客户端——与其他 Agent 通信"""

    def __init__(self, agent_name: str = "MyAgent"):
        self.agent_name = agent_name
        self.card_client = AgentCardClient()

    async def send_task(
        self,
        target_url: str,
        content: str,
        task_id: Optional[str] = None
    ) -> Optional[Dict[str, Any]]:
        """向目标 Agent 发送任务"""
        if task_id is None:
            task_id = f"task-{datetime.now().strftime('%Y%m%d%H%M%S')}"

        request_body = {
            "jsonrpc": "2.0",
            "method": "tasks/send",
            "params": {
                "id": task_id,
                "message": {
                    "role": "agent",
                    "content": content
                }
            },
            "id": 1
        }

        try:
            async with httpx.AsyncClient() as client:
                response = await client.post(
                    target_url,
                    json=request_body,
                    timeout=30
                )
                response.raise_for_status()
                return response.json()
        except Exception as e:
            print(f"向 {target_url} 发送任务失败: {e}")
            return None

    async def query_task(
        self,
        target_url: str,
        task_id: str
    ) -> Optional[Dict[str, Any]]:
        """查询任务状态"""
        request_body = {
            "jsonrpc": "2.0",
            "method": "tasks/get",
            "params": {"id": task_id},
            "id": 2
        }

        try:
            async with httpx.AsyncClient() as client:
                response = await client.post(
                    target_url,
                    json=request_body,
                    timeout=10
                )
                response.raise_for_status()
                return response.json()
        except Exception as e:
            print(f"查询任务 {task_id} 失败: {e}")
            return None

    async def discover_and_call(
        self,
        agent_url: str,
        skill_query: str,
        input_data: dict
    ) -> Optional[str]:
        """发现 Agent 能力并调用匹配的技能"""
        # 1. 获取 Agent Card
        card = await self.card_client.discover(agent_url)
        if not card:
            return "无法发现目标 Agent"

        # 2. 匹配技能
        matched = self.card_client.match_skills(card, skill_query)
        if not matched:
            skills = [s["name"] for s in card.get("capabilities", {}).get("skills", [])]
            return f"未找到匹配的技能。可用技能: {', '.join(skills)}"

        # 3. 调用第一个匹配的技能
        skill = matched[0]
        content = json.dumps({
            "skill": skill["id"],
            "input": input_data
        })

        result = await self.send_task(agent_url, content)
        if result and "result" in result:
            messages = result["result"].get("messages", [])
            if messages:
                return messages[-1].get("content", "")

        return "调用失败"


# 使用示例
async def demo():
    client = A2AClient("TravelAgent")

    # 发现天气 Agent
    card = await client.card_client.discover("http://localhost:8000")
    if card:
        print(f"发现 Agent: {card['name']}")
        print(f"版本: {card['version']}")

    # 发送任务
    result = await client.send_task(
        "http://localhost:8000/a2a",
        "你好，请问今天北京天气怎么样？"
    )
    if result:
        print(f"Agent 回复: {result}")
```

---

## Part 5：MCP vs A2A 对比

### 定位不同

```plaintext
MCP: Agent ←→ 工具/数据     （纵向：Agent 调用资源）
A2A: Agent ←→ Agent        （横向：Agent 间协作）

两者互补，不是竞争关系！
```

### 详细对比

| 维度 | MCP | A2A |
|------|-----|-----|
| **全称** | Model Context Protocol | Agent-to-Agent Protocol |
| **发起方** | Anthropic | Google |
| **定位** | Agent 调用工具的协议 | Agent 间通信的协议 |
| **通信模式** | 客户端-服务器 | 对等网络 |
| **传输层** | JSON-RPC over stdio/SSE/Streamable HTTP | JSON-RPC over HTTP |
| **核心概念** | Resources, Tools, Prompts | Agent Card, Task, Message |
| **状态管理** | 无状态（每次请求独立） | 有状态（任务生命周期） |
| **推送机制** | SSE 服务端推送 | 支持推送通知 |
| **适用场景** | 工具调用、数据访问 | Agent 协作、任务委托 |
| **生态规模** | 10K+ 服务器，97M+ SDK 下载 | 150+ 组织支持 |

### 协同工作示例

```python
class HybridAgent:
    """同时支持 MCP 和 A2A 的 Agent"""

    def __init__(self, name: str):
        self.name = name
        self.mcp_tools = {}     # MCP 工具
        self.a2a_agents = {}    # A2A Agent 连接

    def register_mcp_tool(self, name: str, tool_fn):
        """注册 MCP 工具"""
        self.mcp_tools[name] = tool_fn

    def register_a2a_agent(self, name: str, agent_url: str):
        """注册 A2A Agent"""
        self.a2a_agents[name] = agent_url

    async def handle_request(self, request: str) -> str:
        """处理用户请求，同时使用 MCP 和 A2A"""

        # 1. 先用 MCP 工具查询本地数据
        if "天气" in request and "mcp_weather" in self.mcp_tools:
            local_data = await self.mcp_tools["mcp_weather"](request)
            if local_data:
                return f"本地数据: {local_data}"

        # 2. 本地不行，通过 A2A 调用其他 Agent
        if "天气" in request and "weather_agent" in self.a2a_agents:
            client = A2AClient(self.name)
            result = await client.send_task(
                self.a2a_agents["weather_agent"],
                request
            )
            if result:
                return f"天气 Agent: {result}"

        return "无法处理该请求"

    async def serve_a2a(self, message: dict) -> dict:
        """处理其他 Agent 发来的 A2A 请求"""
        content = message.get("content", "")
        # Agent 可以使用本地的 MCP 工具
        return {
            "status": "completed",
            "content": f"{self.name} 处理完成"
        }
```

---

## Part 6：实战——构建 A2A Agent 网络

### 多 Agent 协作流程

```python
import asyncio
from typing import List, Dict


class AgentNetwork:
    """A2A Agent 网络——管理多个 Agent 的协作"""

    def __init__(self):
        self.agents = {}  # name -> url
        self.client = A2AClient("NetworkOrchestrator")

    def register_agent(self, name: str, url: str, description: str):
        """注册 Agent 到网络"""
        self.agents[name] = {
            "url": url,
            "description": description
        }
        print(f"注册 Agent: {name} ({url})")

    async def find_best_agent(self, query: str) -> List[tuple]:
        """找到最适合处理查询的 Agent"""
        results = []
        for name, info in self.agents.items():
            card = await self.client.card_client.discover(info["url"])
            if card:
                matches = self.client.card_client.match_skills(card, query)
                if matches:
                    results.append((name, info["url"], matches))
        # 按匹配数量排序
        results.sort(key=lambda x: len(x[2]), reverse=True)
        return results

    async def orchestrate(self, query: str) -> str:
        """编排多 Agent 协作"""
        print(f"\n编排任务: {query}")

        # 1. 找到合适的 Agent
        candidates = await self.find_best_agent(query)
        if not candidates:
            return "没有找到能处理该请求的 Agent"

        print(f"找到 {len(candidates)} 个候选 Agent")

        # 2. 调用最佳 Agent
        name, url, skills = candidates[0]
        skill_names = [s["name"] for s in skills]
        print(f"选择 Agent: {name} (技能: {', '.join(skill_names)})")

        result = await self.client.send_task(url, query)
        if result and "result" in result:
            msgs = result["result"].get("messages", [])
            if msgs:
                return f"[{name}] {msgs[-1]['content']}"

        return f"[{name}] 处理失败"


# 构建完整的演示
async def demo_agent_network():
    network = AgentNetwork()

    # 注册 Agent
    network.register_agent(
        "天气助手",
        "http://weather:8000/a2a",
        "查询天气信息"
    )
    network.register_agent(
        "航班助手",
        "http://flight:8000/a2a",
        "查询和预订航班"
    )
    network.register_agent(
        "酒店助手",
        "http://hotel:8000/a2a",
        "查询和预订酒店"
    )

    # 执行任务
    result = await network.orchestrate("明天北京到上海的航班有哪些？")
    print(f"\n最终结果:\n{result}")


# 运行
if __name__ == "__main__":
    asyncio.run(demo_agent_network())
```

### A2A 协议的关键设计原则

```
1. 以 Agent Card 为中心
   - 每个 Agent 必须公开自己的能力
   - 客户端通过 Card 发现和调用

2. 任务驱动的通信
   - 任务有明确的生命周期
   - 支持同步和异步两种模式

3. 松耦合设计
   - Agent 不知道对方的内部实现
   - 只通过标准化的 JSON-RPC 通信

4. 可组合性
   - 多个 Agent 可以组合成更大的系统
   - 支持链式调用和并行调用

5. 安全第一
   - 支持认证和授权
   - Agent 之间需要身份验证
```

---

## Part 7：A2A 与 MCP 的未来

### 标准统一趋势

```
2024      MCP 发布（工具调用标准）
2025      A2A 发布（Agent 通信标准）
     ↓    两者都在快速发展
2026      Linux Foundation Agentic AI Foundation
     ↓    统一管理两大协议
未来      MCP + A2A = Agent 互联网
```

### 预测：Agent 互联网

```
你的个人 Agent
    │
    ├── MCP → 你的日历、邮箱、文件
    ├── MCP → 你订阅的新闻、天气
    │
    ├── A2A → 银行 Agent（查询余额、转账）
    ├── A2A → 购物 Agent（比价、下单）
    ├── A2A → 旅行 Agent（规划行程）
    └── A2A → 医疗 Agent（预约挂号）
```

### 学习路径建议

| 阶段 | 学习内容 | 目标 |
|------|---------|------|
| 入门 | 理解 MCP + A2A 的定位区别 | 知道什么时候用哪个 |
| 基础 | 能实现 MCP 服务器 | 让 Agent 能用工具 |
| 进阶 | 能实现 A2A 服务器 | 让 Agent 能协作 |
| 高级 | 设计多 Agent 协作系统 | 构建 Agent 网络 |
| 前沿 | 关注 Linux Foundation 的标准演进 | 跟上行业发展 |

---

## 今日要点

- 理解了 A2A 协议的核心定位——Agent 间通信标准
- 掌握了 Agent Card 的概念和发现机制
- 学会了 A2A 的请求/响应格式和任务生命周期
- 理解了 MCP 与 A2A 的区别与互补关系
- 实现了 A2A 客户端和服务端基础代码
- 了解了 Agent 互联网的未来趋势

---

## 自测题

1. A2A 和 MCP 的核心区别是什么？它们如何互补？
2. Agent Card 的作用是什么？包含哪些关键信息？
3. A2A 任务有哪几种状态？生命周期是怎样的？
4. 在多 Agent 系统中，A2A 如何实现 Agent 发现？
5. Agent 网络编排的核心流程是什么？

---

## 动手作业

1. 基于 FastAPI 实现一个简单的 A2A Agent 服务器，包含 Agent Card 端点
2. 实现 A2A 客户端，能发现远程 Agent 并调用其技能
3. 构建一个 3 Agent 网络（天气 + 新闻 + 提醒），实现简单的编排
4. （进阶）设计一个同时使用 MCP 工具和 A2A 协作的混合 Agent