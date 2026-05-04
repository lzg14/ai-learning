# Day 20：多 Agent 协作系统设计

> 一个 Agent 能力有限，多个 Agent 协作可以解决更复杂的问题。本课深入角色化 Agent 设计、三种经典协作模式、Agent 间通信机制，以及完整的实战案例。

## Part 1：为什么需要多 Agent 协作？

### 单个 Agent 的局限性

**"一个人做项目"类比：** 想象你一个人做一个软件项目，既要当产品经理、程序员、测试工程师、运维...结果是什么？

| 问题 | 说明 |
|------|------|
| **精力分散** | 什么都做，什么都做不精 |
| **切换成本高** | 思路来回跳 |
| **视角单一** | 自己写的代码，自己看不出问题 |
| **效率低下** | 什么都自己来，进度慢 |

> 单个 Agent 就像一个人做整个项目——什么都能干，但什么都干不精。多 Agent 协作就像组建一个团队，每个人专精一个领域，分工合作。

### 单 Agent vs 多 Agent 对比

| 维度 | 单 Agent | 多 Agent |
|------|----------|----------|
| **专业度** | 什么都会一点，但都不精 | 每个 Agent 专精一个领域 |
| **视角多样性** | 只有一个视角，容易钻牛角尖 | 多个视角，互相制衡 |
| **并行能力** | 一次只能做一件事 | 多个 Agent 可以并行工作 |
| **容错性** | 错了就一路错下去 | 一个 Agent 错了，其他可以纠正 |
| **复杂度上限** | 中等复杂度任务 | 可以解决非常复杂的任务 |

---

## Part 2：角色化 Agent 设计五原则

| 原则 | 说明 | 反例 |
|------|------|------|
| **单一职责** | 每个 Agent 只负责一件事 | 一个 Agent 又写代码又测试 |
| **明确人设** | 给 Agent 明确的性格和背景 | 没有 System Prompt 的 "万能" Agent |
| **专用工具** | 给每个角色合适的工具 | 给测试 Agent 代码执行工具 |
| **固定流程** | 明确协作顺序和交互方式 | 没有标准的协作流程 |
| **仲裁机制** | 意见不一致时谁来拍板 | 两个 Agent 互相否定无法推进 |

### 角色定义示例

```python
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class AgentRole:
    name: str
    role: str
    goal: str
    backstory: str
    tools: List[str] = None

PRODUCT_MANAGER = AgentRole(
    name="PM",
    role="产品经理",
    goal="写出清晰的需求文档",
    backstory="你是一位资深产品经理，擅长理解用户需求，撰写 PRD。"
)

ARCHITECT = AgentRole(
    name="Architect",
    role="架构师",
    goal="设计合理的技术方案",
    backstory="你是一位经验丰富的架构师，擅长系统设计。"
)
```

---

## Part 3：三种经典协作模式

### 模式一：流水线模式（Pipeline）

**"工厂流水线"类比：** 每个工人只干自己那一步，干完了传给下一个。

```
用户需求 → [产品 Agent] → [架构 Agent] → [开发 Agent] → [测试 Agent] → 最终交付
```

**优点：** 简单清晰、易实现
**缺点：** 只能串行、回溯成本高

### 模式二：层级模式（Hierarchy）

**"公司组织架构"类比：** 项目经理 Agent 统筹协调，各专业 Agent 汇报。

```
       [项目经理 Agent]
      ┌──────┴──────────┐
      ↓                  ↓
[产品 Agent]        [技术负责人 Agent]
                    ┌────┴────┐
                    ↓          ↓
              [前端 Agent] [后端 Agent]
```

**优点：** 可并行、权责清晰
**缺点：** 管理者可能成瓶颈

### 模式三：对等模式（Peer-to-Peer）

**"圆桌会议"类比：** 几个专家围坐在一起讨论，最后投票或协商达成一致。

```
[正方 Agent] ↔ [反方 Agent] ↔ [评委 Agent]
```

**优点：** 多视角讨论、决策质量高
**缺点：** 效率低、可能陷入争论

---

## Part 4：Agent 间通信机制

### 三种通信方式

| 方式 | 说明 | 类比 |
|------|------|------|
| **共享内存** | 所有 Agent 读写同一个"黑板" | 团队共享的 Notion 页面 |
| **消息传递** | Agent 之间直接发消息 | 团队成员之间的私聊 |
| **混合模式** | 状态存黑板、交互用消息 | 大部分实际项目用这个 |

### 消息总线实现

```python
from dataclasses import dataclass
from typing import List, Dict, Optional
from enum import Enum
import json

class MessageType(Enum):
    TASK = "task"       # 任务消息
    RESULT = "result"    # 结果消息
    QUERY = "query"     # 查询消息
    RESPONSE = "response"  # 响应消息
    BROADCAST = "broadcast"  # 广播

@dataclass
class AgentMessage:
    """标准消息格式"""
    id: str
    type: MessageType
    sender: str
    receiver: str  # "broadcast" 表示广播
    content: Dict
    timestamp: str
    trace_id: str  # 追踪整个协作链

class MessageBus:
    """Agent 消息总线"""
    def __init__(self):
        self.messages: List[AgentMessage] = []
        self.subscribers: Dict[str, callable] = {}
        self.message_id = 0

    def publish(self, message: AgentMessage):
        """发布消息"""
        self.messages.append(message)
        print(f"[{message.sender}] → [{message.receiver}]: {str(message.content)[:50]}...")
        # 通知订阅者
        if message.receiver in self.subscribers:
            self.subscribers[message.receiver](message)
        if "broadcast" in self.subscribers:
            self.subscribers["broadcast"](message)

    def subscribe(self, agent_name: str, callback: callable):
        """订阅消息"""
        self.subscribers[agent_name] = callback

    def get_messages(self, agent_name: str) -> List[AgentMessage]:
        """获取某 Agent 的消息"""
        return [
            m for m in self.messages
            if m.receiver == agent_name or m.receiver == "broadcast"
        ]
```

---

## Part 5：实战——软件开发团队 Agent

### BaseAgent 基类

```python
class BaseAgent:
    def __init__(self, role: AgentRole, message_bus: MessageBus):
        self.role = role
        self.bus = message_bus
        self.context = []
        # 订阅自己的消息
        self.bus.subscribe(role.name, self.receive_message)

    def think(self, prompt: str) -> str:
        """让 LLM 思考"""
        messages = [
            {"role": "system", "content": f"{self.role.backstory}\n你的目标：{self.role.goal}"},
            {"role": "user", "content": prompt}
        ]
        # 调用国产模型
        response = llm.invoke(messages)
        return response.content

    def send_message(self, receiver: str, content: Dict, msg_type: MessageType):
        """发送消息"""
        msg = AgentMessage(
            id=str(len(self.bus.messages)),
            type=msg_type,
            sender=self.role.name,
            receiver=receiver,
            content=content,
            timestamp="2026-05-01T00:00:00Z",
            trace_id="trace-001"
        )
        self.bus.publish(msg)

    def receive_message(self, message: AgentMessage):
        """接收消息"""
        self.context.append(message)
        print(f"  {self.role.name} 收到来自 {message.sender} 的消息")
```

### 流水线执行引擎

```python
def run_software_pipeline(requirement: str) -> Dict:
    """流水线执行软件开发"""
    bus = MessageBus()

    # 创建 Agent
    pm = BaseAgent(PRODUCT_MANAGER, bus)
    architect = BaseAgent(ARCHITECT, bus)
    coder = BaseAgent(AgentRole("Coder", "程序员", "写出高质量代码", "你是一位 Python 专家"), bus)
    tester = BaseAgent(AgentRole("Tester", "测试工程师", "发现潜在问题", "你是一位严谨的测试工程师"), bus)

    # Step 1: 产品经理
    prd = pm.think(f"请根据以下需求撰写 PRD：\n{requirement}")
    pm.send_message("Architect", {"prd": prd}, MessageType.RESULT)
    print(" PRD 生成完成")

    # Step 2: 架构师
    design = architect.think(f"请根据以下 PRD 设计技术方案：\n{prd}")
    architect.send_message("Coder", {"design": design}, MessageType.RESULT)
    print(" 技术方案设计完成")

    # Step 3: 程序员
    code = coder.think(f"请根据以下技术方案编写代码：\n{design}")
    coder.send_message("Tester", {"code": code}, MessageType.RESULT)
    print(" 代码编写完成")

    # Step 4: 测试工程师
    test_plan = tester.think(f"请根据以下代码制定测试计划：\n{code}")
    print(" 测试计划制定完成")

    return {"prd": prd, "design": design, "code": code, "test_plan": test_plan}

# 执行
result = run_software_pipeline("开发一个简单的待办事项管理应用")
```

---

## Part 6：协作模式选型指南

| 场景 | 推荐模式 | 原因 |
|------|----------|------|
| 简单流程、步骤固定 | 流水线 | 实现简单、逻辑清晰 |
| 大型项目、多团队 | 层级 | 可扩展、权责分明 |
| 需要创意碰撞、多视角 | 对等 | 讨论充分 |
| 大部分生产系统 | 混合 | 结合各种模式优点 |

**设计要诀：**
1. 先画协作流程图，再写代码
2. 每个 Agent 的职责写在 System Prompt 里
3. 消息格式要规范，字段要完整
4. 所有通信走总线，不要直接调用
5. 加错误处理和重试机制

---

## 今日要点

- 理解了多 Agent 协作的必要性和单 Agent 的局限
- 掌握了角色化 Agent 设计的五原则
- 学会了三种经典协作模式：流水线、层级、对等
- 掌握了消息总线的设计和实现
- 实战了完整的软件开发团队 Agent

---

## 自测题

1. 单 Agent 的局限性有哪些？多 Agent 如何解决？
2. 角色化 Agent 设计五原则是什么？
3. 三种协作模式分别适合什么场景？
4. 消息总线的三个核心方法是什么？
5. 流水线模式和层级模式的核心区别是什么？

---

## 动手作业

1. 实现一个消息总线，支持 Agent 间通信和广播
2. 设计一个"论文写作团队"（研究员 / 写作者 / 审稿人）
3. 给流水线添加错误处理和重试机制