# Day 16：ReAct 推理与反思机制

> ReAct = Reasoning（推理）+ Acting（行动），是 Agent 解决复杂问题的核心范式。本课深入 ReAct 原理、反思机制、以及如何让 Agent "学会思考"。

## Part 1：ReAct 深度原理

### 什么是 ReAct？

**"侦探破案"类比：**

| 步骤 | 侦探做什么 | ReAct 对应什么 |
|---|---|---|
| 1 | 观察现场，思考："门是开的，说明嫌疑人可能有钥匙" | **推理（Reasoning）** - 模型生成思考过程 |
| 2 | "我需要去查一下谁有这扇门的钥匙" | **决定行动** - 模型决定下一步调用什么工具 |
| 3 | 去物业查了钥匙记录，发现张三有 | **观察（Observation）** - 工具返回的结果 |
| 4 | "张三有钥匙，但他上周出国了，不在场证明成立" | **再推理** - 根据新信息继续思考 |
| 5 | "那我需要再查一下监控录像" | **再行动** - 继续调用工具 |
| 6 | 循环，直到破案 | **ReAct 循环** |

> 💡 关键点：传统的思维链（CoT）只是"想"，而 ReAct 是"想了之后去做，做了之后根据结果再想"。这就是 Agent 和纯对话模型的本质区别。

### ReAct vs 纯思维链（CoT）对比

| 维度 | 纯思维链（CoT） | ReAct |
|---|---|---|
| **信息来源** | 模型内部知识 | 模型知识 + 外部工具 |
| **可验证性** | 无法验证，容易幻觉 | 每一步都有工具结果支撑 |
| **错误修正** | 错了就一路错下去 | 可以根据新信息修正 |
| **适合问题** | 常识推理、数学题 | 需要外部信息的复杂问题 |
| **透明度** | 只有思考过程 | 思考 + 行动 + 观察，完全透明 |

### ReAct 的六步标准流程

```plaintext
用户问题
    ↓
1. 🤔 思考（Thought）
   模型生成："要回答这个问题，我需要先知道XXX"
    ↓
2. 🎯 决定行动（Action）
   模型决定："我应该调用工具 YYY，参数是 ZZZ"
    ↓
3. 🔧 执行行动（Action Input）
   实际调用工具，传入参数
    ↓
4. 👀 观察结果（Observation）
   拿到工具返回的结果
    ↓
5. 🔄 判断是否继续
   如果信息足够 → 输出最终答案
   如果信息不够 → 回到第 1 步继续思考
    ↓
6. ✅ 最终答案（Final Answer）
   整合所有信息，给出最终回答
```

---

## Part 2：反思机制设计

### 什么是 Self-Reflection（自我反思）？

**"做作业检查"类比：**

| 角色 | 做什么 | 反思对应什么 |
|---|---|---|
| 学生 | 先把作业做完 | Agent 先用 ReAct 解决问题 |
| 学生 | 做完之后，检查一遍："这道题我算对了吗？" | **反思第一步：自我评估** |
| 学生 | "这一步好像算错了，重新算一遍" | **反思第二步：错误识别** |
| 学生 | 重新计算，修正错误 | **反思第三步：自我修正** |
| 老师 | 最后老师批改打分 | **反思第四步：外部反馈** |

> 💡 没有反思的 Agent 就像一个从不检查作业的学生，做了就交，错了也不知道。反思机制让 Agent 有了"元认知"能力 — 知道自己什么时候错了，知道怎么改正。

### 反思的三个层次

| 层次 | 反思内容 | 作用 | 难度 |
|---|---|---|---|
| **Level 1 - 结果检查** | "我的答案对吗？" | 验证最终结果正确性 | ⭐ 简单 |
| **Level 2 - 过程检查** | "我的推理步骤有问题吗？" | 回溯整个思考过程 | ⭐⭐ 中等 |
| **Level 3 - 策略反思** | "我是不是用错方法了？" | 从更高层面调整策略 | ⭐⭐⭐ 难 |

### 如何实现反思机制？

**方法一：双模型架构（Actor + Critic）**

```plaintext
┌─────────────┐
│   Actor     │  负责解决问题（ReAct 循环）
│  (执行模型) │
└──────┬──────┘
       ↓  生成答案
┌──────▼──────┐
│   Critic    │  负责批评和反思
│  (批评模型) │  "这个答案对吗？有什么问题？"
└──────┬──────┘
       ↓  给出批评意见
┌──────▼──────┐
│   Actor     │  根据批评意见修正答案
│  (修正执行) │
└─────────────┘
```

**方法二：单模型自我对话**
- 同一个模型，换不同的 Prompt：
  1. 作为"解决者"："请解决这个问题..."
  2. 作为"批评者"："请评估上面的解答，指出问题..."
  3. 作为"修正者"："请根据上面的批评，重新解答..."

---

## Part 3：错误回溯与自我修正

### 错误回溯的五步法

```plaintext
第 1 步：收集完整轨迹
   ↓
保存所有的思考（Thought）、行动（Action）、观察（Observation）
这就是 Agent 的"黑匣子"记录

第 2 步：标记错误点
   ↓
找到"哪一步开始出问题了"
是工具调用错了？还是推理错了？还是理解用户问题错了？

第 3 步：分析错误原因
   ↓
- 工具调用错了？参数传错了？
- 推理逻辑错了？漏了什么前提？
- 观察结果理解错了？误解了工具返回？
- 用户问题理解错了？意图识别错了？

第 4 步：从错误点重来
   ↓
回到出错的那一步，修正之后继续执行

第 5 步：总结经验教训
   ↓
把这次的错误和修正方法记到"经验库"里
下次遇到类似问题，不要再犯
```

---

## Part 4：实战——构建自我反思 Agent

### 完整代码

```python
"""
带自我反思机制的 ReAct Agent
支持：推理 → 行动 → 反思 → 修正 完整循环
"""
import os
from dotenv import load_dotenv
from langchain_community.chat_models import ChatZhipuAI
from langchain.agents import Agent, initialize_agent, Tool
from langchain.tools import tool
from typing import List, Dict

load_dotenv()

llm = ChatZhipuAI(
    api_key=os.getenv("ZHIPU_API_KEY"),
    model="glm-4-flash",
    temperature=0.0
)

# 工具定义
@tool
def search(query: str) -> str:
    """搜索互联网"""
    return f"搜索结果：关于'{query}'的相关信息..."

@tool
def calculate(expression: str) -> str:
    """计算数学表达式"""
    try:
        result = eval(expression)
        return str(result)
    except:
        return "计算错误"

# ReAct Agent
tools = [search, calculate]
agent = initialize_agent(
    tools=tools,
    llm=llm,
    agent="zero-shot-react-description",
    verbose=True
)

# 反思函数
def reflect_on_result(question: str, result: str) -> Dict:
    """反思结果，检查是否有问题"""
    reflection_prompt = f"""请评估以下回答的质量：

问题：{question}
回答：{result}

请检查：
1. 回答是否解决了问题？
2. 是否有事实错误或逻辑错误？
3. 是否需要补充或修正？

返回JSON格式：
{{"needs_revision": true/false, "issues": ["问题1", "问题2"], "suggestions": ["建议1"]}}
"""
    response = llm.invoke(reflection_prompt)
    return response.content

# 执行流程
def agent_with_reflection(question: str) -> str:
    # 1. ReAct 解决问题
    result = agent.invoke({"input": question})
    answer = result["output"]

    # 2. 反思评估
    reflection = reflect_on_result(question, answer)

    # 3. 如果需要修正，重新生成
    if "true" in reflection.lower() and "needs_revision" in reflection.lower():
        correction_prompt = f"""请根据以下反馈修正你的回答：

原问题：{question}
原回答：{answer}
反馈：{reflection}

请生成修正后的回答：
"""
        answer = llm.invoke(correction_prompt).content

    return answer

# 测试
result = agent_with_reflection("北京人口是多少？比上海多还是少？")
print(result)
```

---

## 📌 今日要点

- ✅ 理解了 ReAct 的"侦探破案"类比（推理 → 行动 → 观察 → 再推理）
- ✅ 掌握了 ReAct 六步标准流程
- ✅ 理解了自我反思的"做作业检查"类比
- ✅ 学会了反思的三个层次（结果检查 → 过程检查 → 策略反思）
- ✅ 掌握了 Actor + Critic 双模型架构

---

## 自测题

1. ReAct 和纯思维链（CoT）最本质的区别是什么？
2. 自我反思的三个层次分别是什么？各有什么作用？
3. Actor + Critic 双模型架构中，两个角色分别做什么？
4. ReAct 无限循环的原因是什么？如何避免？

---

## 动手作业

1. 实现一个带自我反思的 Agent，检验回答的正确性
2. 给 Agent 添加最大步数限制，防止死循环
3. 设计一个"论文写作 Agent"，包含研究者、写作者、评审者三个角色

## 📌 课程更新 (2026-05-06)

> 以下内容由 AI 评审（≥70分）推荐补充

### AI安全（评分：87）
- **标题：** 提示词注入攻击如何防范:从Indirect Prompt Injection到RAG供应链的分层防御  
- **来源：** https://blog.csdn.net/fei  
- **推荐理由：** ReAct场景下的AI安全威胁与防御策略
