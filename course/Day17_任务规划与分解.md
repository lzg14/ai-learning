# Day 17：任务规划与分解

> 复杂任务无法一步完成，需要先规划、再分解、再执行。本课深入 Plan-and-Execute 架构、分层规划、动态重规划等高级规划技术。

## Part 1：Plan-and-Execute 架构深度解析

### 为什么需要规划？

**"装修房子"类比：**

| 没有规划 | 有规划 |
|---|---|
| 第一天：随便买一桶油漆，开始刷墙 | 第一天：画设计图，列材料清单，算预算 |
| 刷到一半：发现墙不平，需要先抹灰 | 按计划执行：水电 → 泥瓦 → 木工 → 油漆 |
| 买油漆时：发现需要梯子，又去买梯子 | 材料一次性买齐，不用反复跑建材市场 |
| 最后：墙刷歪了，颜色不对，返工三次 | 按部就班，一次搞定 |

> 💡 简单任务（查个天气、算个数学题）不需要规划，ReAct 边想边做就行。但**复杂任务（写论文、做项目、开发软件）必须先规划，再执行，不然就是瞎折腾。

### ReAct vs Plan-and-Execute 对比

| 维度 | ReAct（边想边做） | Plan-and-Execute（先想后做） |
|---|---|---|
| **适合任务** | 简单、探索性任务 | 复杂、结构化任务 |
| **思维方式** | 走一步看一步 | 先看好了再走 |
| **透明度** | 每一步都可见 | 计划先可见，再执行 |
| **错误修正** | 错了再调整 | 计划阶段就排除大部分错误 |
| **效率** | 简单任务快 | 复杂任务效率高 |

**形象类比：**
- **ReAct = 探险** - 走一步看一步，遇到岔路口再决定往哪走
- **Plan-and-Execute = 跟团旅游** - 先把整个行程都安排好了，按计划走

### Plan-and-Execute 的三层架构

```plaintext
┌─────────────────────────────────────────────────────┐
│              第 1 层： Planner（规划师）              │
│  输入：用户的大目标                                │
│  输出：3-7 个步骤的高层计划                        │
└───────────────────────┬───────────────────────────┘
                        ↓
┌───────────────────────▼───────────────────────────┐
│              第 2 层： Executor（执行者）        │
│  输入：规划师给的一个具体步骤                      │
│  输出：这一步的执行结果                          │
│  （可以用 ReAct 模式完成这一个小步骤）            │
└───────────────────────┬───────────────────────────┘
                        ↓
┌───────────────────────▼───────────────────────────┐
│              第 3 层： Reviewer（评审者）          │
│  输入：所有步骤的执行结果                          │
│  输出：质量评估 + 是否需要调整计划                    │
└─────────────────────────────────────────────────────┘
```

---

## Part 2：分层规划

### 什么是分层规划？

**"盖房子"类比：**

| 规划层级 | 盖房子对应 | 内容 |
|---|---|---|
| **高层规划** | 建筑设计图 | 房子整体长什么样，几室几厅 |
| **中层规划** | 施工方案 | 先做水电，再做泥瓦，再做木工 |
| **底层执行** | 具体施工 | 今天买水泥，明天砌墙，后天装水管 |

> 💡 不要让模型一次规划到最细的步骤。就像你不会让建筑师直接决定"第 3 块砖放在哪里"一样 — 高层只管"大方向"，中层管"怎么做"，底层管"具体做"。

### 三层规划的具体内容

| 层级 | 负责什么 | 例子 | 颗粒度 |
|---|---|---|---|
| **Level 1 - 战略层** | 做什么？不做什么？ | "写一篇关于 RAG 的论文，5000字，包含原理、实现、案例三部分" | 天级 |
| **Level 2 - 战术层** | 分几步做？先做什么后做什么？ | "1. 搜集资料 2. 写大纲 3. 写正文 4. 修改" | 小时级 |
| **Level 3 - 执行层** | 这一步具体怎么干？ | "写引言部分：先写背景，再写问题，最后写本文贡献" | 分钟级 |

---

## Part 3：计划调整与动态重规划

### 为什么需要动态重规划？

**"开车导航"类比：**
你开车从北京去上海：
1. 出发前，导航给你规划了一条路线
2. 开着开着，前方发生交通事故，堵死了
3. 导航不会说"按原计划继续走，堵着吧"
4. 导航会**自动重新规划路线，绕开拥堵**

> 💡 一句名言："没有任何计划在与敌人接触后还能存活" — 赫尔穆特·冯·毛奇

### 什么时候需要重规划？

| 触发条件 | 例子 | 处理方式 |
|---|---|---|
| **信息变了** | 搜集资料时发现这个题目前人已经写过了 | 调整论文题目，换个角度 |
| **难度超预期** | 原本以为 1 小时能写完，实际写了 3 小时还没写完 | 拆成更小的步骤，或者降低难度 |
| **走到死胡同** | 想用的 API 收费了，免费的用不了 | 换技术方案 |
| **用户改需求了** | 用户："等一下，我想加个案例" | 把新需求融入计划 |
| **执行结果太差** | 写出来的内容全是废话，质量不行 | 回头重写这一部分 |

---

## Part 4：实战——任务规划 Agent

### 完整代码

```python
"""
任务规划 Agent（分层规划 + 动态重规划）
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

# ========== 第一层：战略规划 ==========
strategic_planner_prompt = ChatPromptTemplate.from_messages([
    ("system", """你是一个任务战略规划师。
分析用户的目标，确定：
1. 任务的边界和范围
2. 最终交付物是什么
3. 质量标准是什么

输出 JSON 格式：
{"goal": "目标", "scope": "范围边界", "deliverable": "交付物", "quality": "质量标准"}
"""),
    ("human", "{task}")
])

# ========== 第二层：战术规划 ==========
tactical_planner_prompt = ChatPromptTemplate.from_messages([
    ("system", """你是一个任务战术规划师。
根据高层目标，将任务分解成 3-7 个具体步骤。

每个步骤必须：
1. 有明确的输入和输出
2. 可以独立执行
3. 有验收标准

输出 JSON 格式：
{"steps": [{"id": 1, "task": "任务描述", "input": "输入", "output": "输出", "criteria": "验收标准"}]}
"""),
    ("human", "目标：{goal}\\n范围：{scope}\\n请分解任务")
])

# ========== 第三层：执行器 ==========
executor_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个任务执行助手。"),
    ("human", "请执行以下任务：\\n{task}\\n\\n输入：{input}\\n\\n验收标准：{criteria}")
])

# ========== 评审器 ==========
reviewer_prompt = ChatPromptTemplate.from_messages([
    ("system", """你是一个任务评审员。
评估执行结果是否达到验收标准。
如果没达到，指出问题并给出修改建议。

输出 JSON 格式：
{"passed": true/false, "issues": ["问题1"], "suggestions": ["建议1"]}
"""),
    ("human", "任务：{task}\\n执行结果：{result}\\n验收标准：{criteria}")
])

def plan_and_execute(task: str) -> dict:
    """分层规划执行"""
    results = []

    # 1. 战略规划
    print("📋 第一层：战略规划...")
    strategic_plan = strategic_planner_prompt | llm | JsonOutputParser()
    plan = strategic_plan.invoke({"task": task})

    # 2. 战术规划
    print("📋 第二层：战术规划...")
    tactical_plan = tactical_planner_prompt | llm | JsonOutputParser()
    steps = tactical_plan.invoke({
        "goal": plan["goal"],
        "scope": plan["scope"]
    })

    # 3. 执行每一步
    for step in steps["steps"]:
        print(f"\n🚀 执行 Step {step['id']}：{step['task']}")

        # 执行
        executor = executor_prompt | llm
        result = executor.invoke({
            "task": step["task"],
            "input": step.get("input", ""),
            "criteria": step.get("criteria", "")
        })

        # 评审
        reviewer = reviewer_prompt | llm | JsonOutputParser()
        review = reviewer.invoke({
            "task": step["task"],
            "result": result.content,
            "criteria": step.get("criteria", "")
        })

        if review.get("passed"):
            print(f"✅ Step {step['id']} 通过")
            results.append({"step": step["id"], "result": result.content, "status": "passed"})
        else:
            print(f"⚠️ Step {step['id']} 需要修正：{review.get('issues', [])}")
            results.append({"step": step["id"], "result": result.content, "status": "needs_revision", "issues": review.get("issues", [])})

    return results

# 测试
if __name__ == "__main__":
    results = plan_and_execute("帮我写一个 Python 快速排序算法，包含单元测试")
    print("\n" + "="*50)
    print("执行完成！")
```

---

## 📌 今日要点

- ✅ 理解了 Plan-and-Execute 的"装修房子"类比
- ✅ 掌握了 Plan-and-Execute 的三层架构（规划师 → 执行者 → 评审者）
- ✅ 理解了分层规划的"盖房子"类比（战略 → 战术 → 执行）
- ✅ 掌握了三层规划的具体内容和颗粒度
- ✅ 理解了动态重规划的"开车导航"类比

---

## 自测题

1. Plan-and-Execute 和 ReAct 模式最大的区别是什么？
2. 三层规划分别是什么？各自负责什么？
3. 什么时候需要动态重规划？
4. 为什么说"没有任何计划在与敌人接触后还能存活"？

---

## 动手作业

1. 实现一个分层规划 Agent，支持战略、战术、执行三层
2. 给规划 Agent 添加动态重规划功能，遇到问题自动调整计划
3. 设计一个"论文写作 Agent"，包含大纲生成、正文撰写、质量审核三个阶段
