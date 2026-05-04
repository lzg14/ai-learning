# Day 31：事件驱动架构

> 松耦合、高并发、可扩展的 Agent 系统架构。本课深入事件驱动的核心思想、三大组件、优缺点、实战设计。

## Part 1：什么是事件驱动？和传统调用式的区别

### 传统调用式架构的问题

**你是不是也写过这样的代码？**

```python
def user_registered(user):
    # 1. 保存到数据库
    save_user(user)

    # 2. 发欢迎邮件
    send_welcome_email(user)

    # 3. 给新用户发优惠券
    send_new_user_coupon(user)

    # 4. 通知销售跟进
    notify_sales_team(user)

    # 5. 同步数据到CRM
    sync_to_crm(user)

    return "注册成功"
```

**问题来了：**
1. ❌ **任何一步崩了，整个注册就失败了**
2. ❌ **响应越来越慢**：加一个功能就多一步，注册从 100ms 变成 5 秒
3. ❌ **紧耦合**：改一个功能，整个函数都要改
4. ❌ **没法灵活扩展**：想加新功能又要改注册函数
5. ❌ **峰值扛不住**：搞活动一秒钟注册 1 万人，所有服务都被打垮了

> 💡 根本原因：你让"注册"这个动作承担了太多不属于它的责任。注册只需要做一件事：把用户信息存下来。后面那些发邮件、发优惠券、通知销售，都不应该在注册的同步流程里。

---

### 事件驱动架构：把"调用"改成"通知"

**改成事件驱动之后：**

```python
def user_registered(user):
    # 只做一件事：保存用户
    save_user(user)

    # 发一个"用户注册了"的事件，告诉全世界
    event_bus.publish("user.registered", user.to_dict())

    return "注册成功"
```

**其他系统各自监听这个事件：**

```python
# 邮件服务监听
@on_event("user.registered")
def send_welcome_email(user):
    send_email(user.email, "欢迎加入！")

# 优惠券服务监听
@on_event("user.registered")
def send_new_user_coupon(user):
    create_coupon(user.id, "新用户专属5元优惠券")
```

**天差地别：**
1. ✅ 注册不会因为发邮件的服务挂了就失败
2. ✅ 响应快：注册只需要存数据库+发事件，100ms 搞定
3. ✅ 松耦合：加新功能只要加新的监听器，不用改核心代码
4. ✅ 抗峰值：事件都存在消息队列里，下游慢慢处理
5. ✅ 可观测：所有事件都有日志，出了问题追溯清清楚楚

---

### 调用式 vs 事件驱动 核心区别

| 维度 | 调用式（同步） | 事件驱动（异步） |
| --- | --- | --- |
| **耦合度** | 紧耦合 | 松耦合 |
| **响应速度** | 所有步骤加起来的总时间 | 只需要核心步骤的时间 |
| **错误影响** | 一步错，全步错 | 下游错了不影响核心流程 |
| **扩展能力** | 加新功能要改核心代码 | 加新功能只要加新的监听器 |
| **峰值处理** | 峰值来了直接被打垮 | 事件存在队列里，下游按自己的速度处理 |

---

## Part 2：事件驱动架构的三大核心组件

### 事件驱动架构的标准结构

```plaintext
[事件生产者] → [事件总线（消息队列）] → [事件消费者]
```

三大组件缺一不可。

---

### 组件一：事件生产者（Event Producer）

**功能：** 什么事情发生了，就发一个事件出来，告诉全世界"我做了什么"。
**核心原则：** 只发事件，不要关心谁会听，也不要等结果。

**事件设计最佳实践：**

```plaintext
❌ 不好的事件命名："send_email"（你在命令别人做事）
✅ 好的事件命名："user.registered"（你在陈述已经发生的事实）
```

> 💡 事件永远是"过去式"，是已经发生的事情的通知，不是命令别人做什么。

**事件内容要完整：**

```json
{
  "event_id": "evt_123456",
  "event_type": "user.registered",
  "timestamp": "2026-05-02T00:00:00Z",
  "data": {
    "user_id": "u_123",
    "email": "user@example.com",
    "name": "张三"
  }
}
```

---

### 组件二：事件总线 / 消息队列（Event Bus / Message Queue）

**功能：** 事件的中转站，生产者把事件发过来，消费者从这里拿事件。

**2026 年主流消息队列对比：**

| 队列 | 优点 | 缺点 | 适合场景 |
| --- | --- | --- | --- |
| **Kafka** | 吞吐量极高，持久化好 | 运维复杂 | 日志、大数据、超高性能 |
| **RabbitMQ** | 功能丰富，路由灵活 | 吞吐量比 Kafka 低 | 业务系统、通用场景 |
| **Redis Stream** | 简单，不用额外部署 | 功能少 | 简单场景、小型项目 |

> 💡 Agent 系统一般选 RabbitMQ 或者 Redis Stream 就足够了。

---

### 组件三：事件消费者（Event Consumer）

**功能：** 监听自己关心的事件，事件来了就做对应的处理。

**消费者设计最佳实践：**
1. ✅ **幂等性**：同一个事件来两次，结果要一样
2. ✅ **错误处理与重试**：处理失败了要重试
3. ✅ **不要阻塞其他消费者**：你的消费者慢，不要影响其他消费者

---

## Part 3：事件驱动的优缺点与适用场景

### 优点

1. **极致的松耦合**
   - 生产者和消费者完全不知道对方的存在
   - 加新功能只要加新消费者，不用改任何现有代码

2. **天然的抗峰值能力**
   - 不管瞬间来多少事件，都先存在队列里
   - 下游消费者按自己的处理速度慢慢吃

3. **响应速度快**
   - 核心流程只需要做最核心的事
   - 其他所有异步流程都后台慢慢做

4. **非常适合多 Agent 协作**
   - Agent 之间不用互相调用，只要发事件
   - 一个 Agent 完成了自己的工作，发个事件，下一个 Agent 自动接上

---

### 缺点和坑

1. **复杂度上升**
   - 从线性流程变成了分散的流程
   - 出了问题要跨多个系统查日志

2. **一致性问题**
   - 同步调用要么全成功要么全失败
   - 事件驱动是最终一致性

3. **重复事件和乱序问题**
   - 网络波动可能导致同一个事件被投递两次
   - 必须做好幂等和顺序保证

---

### 什么时候适合用事件驱动？

| ✅ 非常适合用 | ❌ 不要用，用同步调用 |
| --- | --- |
| 不需要立刻拿到结果的操作 | 必须要拿到结果才能继续的操作 |
| 一个事件会触发多个下游动作 | 只有一个下游，而且强依赖结果 |
| 高并发、有峰值的场景 | 简单的小项目 |
| 多系统、多 Agent 协作 | 需要强一致性的场景 |

> 💡 最佳实践：核心的、需要强一致的用同步调用，其他所有辅助的、不需要立刻结果的、多系统协作的，全部用事件驱动。

---

## Part 4：实战——事件驱动 Agent 协作系统

### 完整代码：事件驱动的多 Agent 协作

```python
"""
事件驱动的多 Agent 协作系统
Agent 之间通过消息队列通信，松耦合、高并发
"""
import asyncio
import json
from typing import Dict, List, Any, Callable
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
import random

class EventType(Enum):
    TASK_CREATED = "task.created"
    TASK_ASSIGNED = "task.assigned"
    TASK_PROCESSING = "task.processing"
    TASK_COMPLETED = "task.completed"
    TASK_FAILED = "task.failed"
    AGENT_RESULT = "agent.result"
    USER_NOTIFICATION = "user.notification"

@dataclass
class Event:
    """事件"""
    event_id: str
    event_type: EventType
    timestamp: str
    data: Dict[str, Any]

class SimpleEventBus:
    """简化的事件总线"""

    def __init__(self):
        self.subscribers: Dict[EventType, List[Callable]] = {}
        self.event_log: List[Event] = []

    def subscribe(self, event_type: EventType, handler: Callable):
        """订阅事件"""
        if event_type not in self.subscribers:
            self.subscribers[event_type] = []
        self.subscribers[event_type].append(handler)
        print(f"✅ 订阅事件：{event_type.value}")

    def publish(self, event: Event):
        """发布事件"""
        print(f"📤 发布事件：{event.event_type.value}")
        self.event_log.append(event)

        if event.event_type in self.subscribers:
            for handler in self.subscribers[event.event_type]:
                asyncio.create_task(handler(event))

class BaseAgent:
    """Agent 基类"""

    def __init__(self, name: str, event_bus: SimpleEventBus):
        self.name = name
        self.event_bus = event_bus
        self.processing_tasks: List[str] = []

    def publish_event(self, event_type: EventType, data: Dict):
        """发布事件"""
        event = Event(
            event_id=f"evt_{random.randint(1000, 9999)}",
            event_type=event_type,
            timestamp=datetime.now().isoformat(),
            data=data
        )
        self.event_bus.publish(event)

    async def handle_event(self, event: Event):
        """处理事件（子类实现）"""
        pass

class PlannerAgent(BaseAgent):
    """规划 Agent：接收用户任务，拆解成子任务"""

    def __init__(self, event_bus: SimpleEventBus):
        super().__init__("规划Agent", event_bus)
        self.event_bus.subscribe(EventType.TASK_CREATED, self.handle_event)

    async def handle_event(self, event: Event):
        """处理任务创建事件"""
        user_task = event.data.get("task")
        print(f"\n📋 {self.name} 收到任务：{user_task}")

        # 拆解任务
        subtasks = [
            {"id": "sub_1", "type": "research", "description": "收集相关信息"},
            {"id": "sub_2", "type": "analysis", "description": "分析数据"},
            {"id": "sub_3", "type": "report", "description": "生成报告"}
        ]

        print(f"📝 拆解为 {len(subtasks)} 个子任务")

        # 发布子任务分配事件
        for subtask in subtasks:
            self.publish_event(EventType.TASK_ASSIGNED, {
                "parent_task": user_task,
                "subtask": subtask,
                "assigned_to": f"{subtask['type']}_agent"
            })

        await asyncio.sleep(0.1)

class ResearchAgent(BaseAgent):
    """研究 Agent：负责收集信息"""

    def __init__(self, event_bus: SimpleEventBus):
        super().__init__("研究Agent", event_bus)
        self.event_bus.subscribe(EventType.TASK_ASSIGNED, self.handle_event)

    async def handle_event(self, event: Event):
        """处理任务分配事件"""
        subtask = event.data.get("subtask", {})
        if subtask.get("type") != "research":
            return

        task_id = subtask.get("id")
        print(f"\n🔍 {self.name} 开始处理：{subtask.get('description')}")

        self.processing_tasks.append(task_id)

        # 模拟研究过程
        await asyncio.sleep(0.5)

        # 发布完成事件
        self.publish_event(EventType.TASK_COMPLETED, {
            "task_id": task_id,
            "agent": self.name,
            "result": f"关于 '{subtask.get('description')}' 的研究成果",
            "status": "success"
        })

        self.processing_tasks.remove(task_id)

class AnalysisAgent(BaseAgent):
    """分析 Agent：负责分析数据"""

    def __init__(self, event_bus: SimpleEventBus):
        super().__init__("分析Agent", event_bus)
        self.event_bus.subscribe(EventType.TASK_ASSIGNED, self.handle_event)

    async def handle_event(self, event: Event):
        """处理任务分配事件"""
        subtask = event.data.get("subtask", {})
        if subtask.get("type") != "analysis":
            return

        task_id = subtask.get("id")
        print(f"\n📊 {self.name} 开始处理：{subtask.get('description')}")

        self.processing_tasks.append(task_id)
        await asyncio.sleep(0.5)

        self.publish_event(EventType.TASK_COMPLETED, {
            "task_id": task_id,
            "agent": self.name,
            "result": f"关于 '{subtask.get('description')}' 的分析结果",
            "status": "success"
        })

        self.processing_tasks.remove(task_id)

class ReportAgent(BaseAgent):
    """报告 Agent：负责生成报告"""

    def __init__(self, event_bus: SimpleEventBus):
        super().__init__("报告Agent", event_bus)
        self.event_bus.subscribe(EventType.TASK_COMPLETED, self.handle_event)

    async def handle_event(self, event: Event):
        """处理任务完成事件"""
        if self.processing_tasks:
            return  # 等待其他任务完成

        task_id = event.data.get("task_id")
        print(f"\n📝 {self.name} 收到完成通知：{task_id}")

        # 收集所有结果
        results = [event.data.get("result", "")]

        # 生成报告
        report = f"""
        ====================================
        分析报告
        ====================================
        {chr(10).join(results)}

        报告生成时间：{datetime.now().isoformat()}
        ====================================
        """

        print(report)

        # 通知用户
        self.publish_event(EventType.USER_NOTIFICATION, {
            "message": "任务已完成！",
            "report": report
        })

async def main():
    print("="*70)
    print("🚀 事件驱动的多 Agent 协作系统")
    print("="*70)

    # 创建事件总线
    event_bus = SimpleEventBus()

    # 创建各个 Agent
    planner = PlannerAgent(event_bus)
    research = ResearchAgent(event_bus)
    analysis = AnalysisAgent(event_bus)
    report = ReportAgent(event_bus)

    # 等待订阅完成
    await asyncio.sleep(0.2)

    # 用户创建一个任务
    print("\n" + "="*70)
    print("👤 用户提交任务：分析市场趋势")
    print("="*70)

    planner.publish_event(EventType.TASK_CREATED, {
        "task": "分析市场趋势",
        "user_id": "user_123"
    })

    # 等待所有 Agent 处理完成
    await asyncio.sleep(2)

    print("\n" + "="*70)
    print(f"📈 事件统计：共发布 {len(event_bus.event_log)} 个事件")
    print("="*70)

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 📌 今日要点

- ✅ 理解了传统调用式架构的问题
- ✅ 掌握了事件驱动架构的核心思想：把"调用"改成"通知"
- ✅ 学会了事件驱动三大组件：事件生产者、事件总线、事件消费者
- ✅ 了解了事件驱动的优缺点和适用场景
- ✅ 实现了完整的事件驱动多 Agent 协作系统

---

## 自测题

1. 传统调用式架构有哪些问题？为什么事件驱动能解决这些问题？
2. 事件驱动架构的三大核心组件是什么？各有什么作用？
3. 消息队列 RabbitMQ、Redis Stream、Kafka 各适合什么场景？
4. 为什么事件要命名成"过去式"？举两个好和坏的例子。

## 动手作业

1. 把上面的事件驱动 Agent 系统跑起来
2. 添加一个新的 Agent（比如"审核 Agent"）来审核报告
3. 给系统加上重试机制和死信队列处理
