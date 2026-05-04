# Day 36：自动规划与推理

> 从"走一步看一步"到"走一步想十步"。本课深入算法蒸馏、世界模型、蒙特卡洛树搜索、反思机制。

## Part 1：传统 Agent 规划方法的局限

### 现在的 Agent 是怎么"思考"的？

绝大多数 Agent 的思考方式是这样的：

```plaintext
用户提了个需求 → 我现在要做什么？ → 选一个工具调用 → 拿到结果 → 接下来要做什么？ → 再选一个工具 → ... 直到做完
```

这就像什么？**走一步看一步，摸着石头过河。**

---

### 会遇到什么问题？

1. **短视**：只看眼前这一步，看不到三步以后会出问题
2. **不会复盘**：上次这么做失败了，下次还这么做
3. **不会选路**：面前有三条路，随便选一条
4. **死胡同里死循环**：进了死胡同还在里面反复试
5. **不会提前准备**：做第一步的时候不会想到第三步需要什么

> 💡 人是怎么思考的？拿到一个问题，先在脑子里推演一下："如果我这么做，第一步会怎么样，第二步会怎么样，第三步可能出什么问题"

---

### 下一代规划推理的四个核心能力

| 能力 | 人类是怎么做的 | 现在的 Agent 有没有 |
| --- | --- | --- |
| **1. 算法级推理** | 解数学题按固定步骤来 | ❌ 每次都重新想 |
| **2. 结果预判** | 做之前先想想后果 | ❌ 做了才知道 |
| **3. 多路径评估** | 有好几个方案，选最好的 | ❌ 想到什么做什么 |
| **4. 自我反思** | 做完了复盘，下次改进 | ❌ 做错了下次还错 |

---

## Part 2：算法蒸馏与世界模型

### 算法蒸馏（Algorithm Distillation）

**核心思想：** 人类研究了几十年的经典算法，为什么不让大模型把这些算法学会？

```plaintext
1. 生成大量某算法的执行轨迹
2. 把这些轨迹当成训练数据，让大模型去学习
3. 大模型学会之后，遇到同类问题，自动就会用这个算法的思路去解
```

**"解题罐头"类比：**
- 经典算法 = 人类总结了几十年的解题方法
- 算法蒸馏 = 把这些解题方法做成罐头，封装进大模型里

---

### 世界模型（World Model）

**核心思想：** 人在做事情之前，会在脑子里"模拟"一下这么做会发生什么，Agent 也应该有这个能力。

**"沙盘推演"类比：**
- 世界模型 = Agent 脑子里的沙盘
- Agent 要做什么动作，先在沙盘上演一遍
- 推演出来结果好，就真的做；结果不好，就换个动作

---

### 世界模型的三大核心能力

| 能力 | 功能说明 |
| --- | --- |
| **1. 状态预测** | 给定当前状态和动作，预测下一个状态会是什么 |
| **2. 结果评估** | 预测之后，评估这个状态是好还是坏 |
| **3. 多步推演** | 不是只看下一步，而是看之后的 N 步 |

---

## Part 3：蒙特卡洛树搜索（MCTS）

### 什么是 MCTS？

**核心思想：** 面前有很多条路可以走，我每条路都试着往前走几步试试，看看哪条路最好，然后选最好的那条路真的走。

**四步走：**
1. **选择**：从当前已经探索过的路径里，选一个最有希望的节点
2. **扩展**：在这个节点下面，拓展几个没试过的新动作
3. **模拟**：从这些新动作开始，用世界模型模拟到结束
4. **回溯**：把模拟的结果反向传回来，更新这条路径的评分

---

## Part 4：实战——智能规划 Agent

### 完整代码：带 MCTS 的智能规划 Agent

```python
"""
带世界模型和蒙特卡洛树搜索的智能规划 Agent
可以在行动之前模拟多个方案，选最优的执行
"""
import asyncio
import random
import math
from typing import List, Dict, Any, Optional
from dataclasses import dataclass, field
from enum import Enum

class ActionType(Enum):
    SEARCH = "搜索信息"
    WRITE = "写内容"
    REVIEW = "检查修改"
    ASK_USER = "询问用户"

@dataclass
class State:
    """任务状态"""
    task: str
    progress: float = 0.0  # 0-1 完成度
    quality: float = 0.5   # 0-1 质量评分
    steps_taken: List[str] = field(default_factory=list)
    is_terminal: bool = False

@dataclass
class Action:
    """可执行的动作"""
    type: ActionType
    description: str
    params: Dict[str, Any] = field(default_factory=dict)

class WorldModel:
    """世界模型：预测做某个动作之后状态会怎么变"""

    async def predict_next_state(self, current_state: State, action: Action) -> State:
        """预测执行动作后的下一个状态"""
        next_state = State(
            task=current_state.task,
            progress=current_state.progress,
            quality=current_state.quality,
            steps_taken=current_state.steps_taken + [action.description],
            is_terminal=False
        )

        if action.type == ActionType.SEARCH:
            next_state.progress += 0.1
            next_state.quality += random.uniform(-0.05, 0.15)
        elif action.type == ActionType.WRITE:
            next_state.progress += 0.3
            next_state.quality += random.uniform(-0.1, 0.2)
        elif action.type == ActionType.REVIEW:
            next_state.progress += 0.05
            next_state.quality += random.uniform(0.05, 0.2)
        elif action.type == ActionType.ASK_USER:
            next_state.progress += 0.05
            next_state.quality += random.uniform(0, 0.1)

        if next_state.progress >= 1.0:
            next_state.is_terminal = True

        next_state.quality = max(0.0, min(1.0, next_state.quality))
        return next_state

    def evaluate_state(self, state: State) -> float:
        """评估一个状态的好坏，分数越高越好"""
        progress_score = state.progress
        quality_score = state.quality * 0.8
        step_penalty = len(state.steps_taken) * 0.02
        total_score = progress_score + quality_score - step_penalty
        return max(0, total_score)

class MCTSNode:
    """蒙特卡洛树的节点"""

    def __init__(self, state: State, parent=None, action_taken=None):
        self.state = state
        self.parent = parent
        self.action_taken = action_taken
        self.children: List[MCTSNode] = []
        self.visit_count: int = 0
        self.total_score: float = 0.0

    def is_fully_expanded(self) -> bool:
        return len(self.children) >= 4

    def get_ucb_score(self, exploration_weight: float = 1.4) -> float:
        """UCB评分，平衡探索和利用"""
        if self.visit_count == 0:
            return float('inf')

        average_score = self.total_score / self.visit_count
        exploration_term = exploration_weight * math.sqrt(
            math.log(self.parent.visit_count) / self.visit_count
        )
        return average_score + exploration_term

class MCTSPlanner:
    """蒙特卡洛树搜索规划器"""

    def __init__(self, world_model: WorldModel):
        self.world_model = world_model
        self.possible_actions = [
            Action(ActionType.SEARCH, "搜索相关资料"),
            Action(ActionType.WRITE, "写初稿"),
            Action(ActionType.REVIEW, "检查和修改内容"),
            Action(ActionType.ASK_USER, "询问用户确认需求")
        ]

    async def search(self, initial_state: State, num_simulations: int = 50) -> Action:
        """做 N 次模拟搜索，返回最优的动作"""
        root = MCTSNode(initial_state)

        for i in range(num_simulations):
            node = root

            # 选择：从当前已经探索过的路径里，选一个最有希望的节点
            while node.is_fully_expanded() and not node.state.is_terminal:
                node = max(node.children, key=lambda n: n.get_ucb_score())

            # 扩展：如果不是终止状态，扩展一个新动作
            if not node.state.is_terminal:
                tried_actions = [child.action_taken for child in node.children]
                for action in self.possible_actions:
                    if action not in tried_actions:
                        next_state = await self.world_model.predict_next_state(node.state, action)
                        child = MCTSNode(next_state, parent=node, action_taken=action)
                        node.children.append(child)
                        node = child
                        break

            # 模拟：从这个节点开始，随机走到结束
            current_state = node.state
            while not current_state.is_terminal and len(current_state.steps_taken) < 10:
                random_action = random.choice(self.possible_actions)
                current_state = await self.world_model.predict_next_state(current_state, random_action)

            # 回溯：把模拟的结果反向传回来
            score = self.world_model.evaluate_state(current_state)
            while node is not None:
                node.visit_count += 1
                node.total_score += score
                node = node.parent

        # 选择访问次数最多的子节点
        best_child = max(root.children, key=lambda n: n.visit_count)
        return best_child.action_taken

class ReflectionAgent:
    """带反思能力的 Agent"""

    def __init__(self):
        self.world_model = WorldModel()
        self.planner = MCTSPlanner(self.world_model)
        self.history: List[Dict] = []

    async def execute_task(self, task: str) -> List[Dict]:
        """执行一个任务，边规划边执行边反思"""
        current_state = State(task=task)
        step = 0

        print(f"\n🎯 开始执行任务：{task}")
        print("=" * 60)

        while not current_state.is_terminal and step < 10:
            step += 1

            # 用 MCTS 规划下一步
            best_action = await self.planner.search(current_state, num_simulations=30)

            print(f"\n📍 步骤 {step}：")
            print(f"   选择动作：{best_action.description}")
            print(f"   当前进度：{current_state.progress * 100:.0f}%")
            print(f"   当前质量：{current_state.quality * 100:.0f}%")

            # 执行动作
            current_state = await self.world_model.predict_next_state(current_state, best_action)

            # 记录历史
            self.history.append({
                "step": step,
                "action": best_action.description,
                "progress": current_state.progress,
                "quality": current_state.quality
            })

            # 反思：检查是否需要调整策略
            if current_state.quality < 0.4:
                print(f"   ⚠️ 质量偏低，考虑调整策略...")
                # 可以触发重新规划

        print("\n" + "=" * 60)
        print(f"✅ 任务完成！")
        print(f"   总步骤：{step}")
        print(f"   最终进度：{current_state.progress * 100:.0f}%")
        print(f"   最终质量：{current_state.quality * 100:.0f}%")

        return self.history

    def reflect(self) -> str:
        """反思执行过程，总结经验"""
        if not self.history:
            return "没有执行历史"

        avg_quality = sum(h["quality"] for h in self.history) / len(self.history)
        total_steps = len(self.history)

        reflections = []

        if avg_quality > 0.7:
            reflections.append("整体执行质量不错")
        elif avg_quality < 0.5:
            reflections.append("执行质量有待提高，需要改进策略")

        if total_steps > 8:
            reflections.append("步骤偏多，可能需要优化流程")

        # 分析每一步
        low_quality_steps = [h for h in self.history if h["quality"] < 0.5]
        if low_quality_steps:
            reflections.append(f"有 {len(low_quality_steps)} 个步骤质量偏低，下次需要特别注意")

        return "；".join(reflections) if reflections else "执行顺利，无需特别反思"

async def main():
    print("=" * 70)
    print("🧠 智能规划 Agent 测试（MCTS + 世界模型 + 反思）")
    print("=" * 70)

    agent = ReflectionAgent()

    # 测试任务
    tasks = [
        "写一篇关于人工智能的科普文章",
        "分析市场趋势报告",
    ]

    for task in tasks:
        history = await agent.execute_task(task)
        reflection = agent.reflect()
        print(f"\n💭 反思：{reflection}")
        agent.history = []

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 📌 今日要点

- ✅ 理解了传统 Agent"走一步看一步"的规划方式的 5 个核心痛点
- ✅ 掌握了算法蒸馏的核心思想
- ✅ 学会了世界模型的原理：在行动之前先模拟后果
- ✅ 掌握了蒙特卡洛树搜索的四步流程：选择、扩展、模拟、回溯
- ✅ 实现了带 MCTS 规划、反思能力的智能 Agent

---

## 自测题

1. 为什么说"走一步看一步"的规划方式是短视的？
2. 算法蒸馏和直接在提示词里告诉大模型"用 XX 算法"有什么区别？
3. 世界模型为什么不能模拟太多步？
4. 蒙特卡洛树搜索的 UCB 评分为什么要有"探索项"？

## 动手作业

1. 把上面的 MCTS 规划 Agent 跑起来
2. 加一个反思改进的功能
3. 调整 UCB 的探索权重，看看探索权重高和低的时候 Agent 的行为有什么不一样
