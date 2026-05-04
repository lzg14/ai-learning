# Day 19：工作流编排与状态机

> 复杂 Agent 任务不是一步能完成的，需要多步、多分支、错误处理、重试。本课深入工作流编排、状态机设计、主流编排框架对比。

## Part 1：为什么需要工作流？

### 纯代码的痛点

```python
def agent_task(user_input):
    step1_result = step1(user_input)
    if step1_result.ok:
        step2_result = step2(step1_result)
        if step2_result.ok:
            step3_result = step3(step2_result)
            # ... 无限嵌套
```

> 💡 三层嵌套之后，没人能看懂逻辑是什么，加新分支要死人。

### 工作流解决什么问题？

| 痛点 | 工作流怎么解决 |
|------|----------------|
| 逻辑混乱，嵌套地狱 | 把流程拆成独立的步骤，可视化定义 |
| 无法断点续传 | 每个步骤的状态都持久化，崩了从断点继续 |
| 错误处理到处都是 | 统一的错误处理、重试、降级策略 |
| 流程变更要改代码 | 可视化改流程，甚至业务人员自己就能改 |

---

## Part 2：状态机核心概念

### 什么是状态机？

状态机 = 一组状态 + 一组事件 + 一组转移规则

```plaintext
当系统处于 [状态A] 时，如果发生了 [事件X]，那么系统转移到 [状态B]，并且执行 [动作Y]。
```

### 四大核心概念

| 概念 | 说明 | 例子 |
|------|------|------|
| **1. 状态（State）** | 系统在某个时间点所处的情况 | `待审批`、`处理中`、`已完成`、`失败` |
| **2. 事件（Event）** | 触发状态变化的事情 | `用户提交`、`审批通过`、`执行失败`、`超时` |
| **3. 转移（Transition）** | 从一个状态到另一个状态的规则 | 待审批 + 审批通过 → 已通过 |
| **4. 动作（Action）** | 状态转移的时候要执行的事情 | 转移到已通过的时候，给用户发通知 |

### 状态机最佳实践

1. **所有状态必须互斥**，一个任务在同一时间只能处于一个状态
2. **不要设计太多状态**，一般不超过 10 个
3. **所有异常情况都要设计对应的状态和转移**
4. **每个状态转移都要记录审计日志**
5. **永远留一个"人工介入"的状态和出口**

---

## Part 3：主流工作流框架对比

### 框架对比

| 框架 | 特点 | 适合场景 | 推荐指数 |
|------|------|----------|----------|
| **LangGraph** | 专门为 LLM Agent 设计，原生支持循环 | Agent 专属工作流 | ⭐⭐⭐⭐⭐ |
| **Temporal** | 企业级工作流，断点续传原生支持 | 长周期任务 | ⭐⭐⭐⭐⭐ |
| **Airflow** | Python 生态友好，数据领域用的多 | 数据 ETL、定时任务 | ⭐⭐⭐⭐ |
| **Prefect** | 比 Airflow 现代，API 友好 | 数据 Pipeline | ⭐⭐⭐ |

### Agent 场景选型

| 场景 | 推荐框架 |
|------|----------|
| **Agent 专用工作流** | LangGraph |
| **企业级长周期任务** | Temporal |
| **数据 + Agent 混合流程** | Airflow + LangGraph |
| **简单流程，不想搞复杂** | 自己写个简单状态机 |

---

## Part 4：LangGraph 实战

### LangGraph 核心概念

```
LangGraph = 节点（Node）+ 边（Edge）+ 状态（State）

- 节点：执行特定任务的函数
- 边：连接节点的路径
- 状态：跨节点传递的数据
```

### 完整代码

```python
"""
LangGraph 工作流实战
一个简单的问答 Agent 工作流
"""
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
from langchain_core.messages import BaseMessage, HumanMessage
import operator

# ========== 定义状态 ==========

class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], operator.add]
    user_input: str
    should_continue: bool

# ========== 定义节点函数 ==========

def process_input(state: AgentState) -> AgentState:
    """处理用户输入"""
    user_input = state["user_input"]
    return {
        "messages": [HumanMessage(content=f"用户输入: {user_input}")],
        "should_continue": True
    }

def generate_response(state: AgentState) -> AgentState:
    """生成回答"""
    from langchain_community.chat_models import ChatZhipuAI
    from dotenv import load_dotenv
    import os

    load_dotenv()
    llm = ChatZhipuAI(
        api_key=os.getenv("ZHIPU_API_KEY"),
        model="glm-4-flash"
    )

    messages = state["messages"]
    response = llm.invoke(messages)

    return {
        "messages": [response],
        "should_continue": False
    }

# ========== 构建图 ==========

workflow = StateGraph(AgentState)

# 添加节点
workflow.add_node("process", process_input)
workflow.add_node("generate", generate_response)

# 添加边
workflow.set_entry_point("process")
workflow.add_edge("process", "generate")
workflow.add_edge("generate", END)

# 编译
app = workflow.compile()

# ========== 执行 ==========

if __name__ == "__main__":
    result = app.invoke({
        "user_input": "你好，请介绍一下你自己",
        "messages": [],
        "should_continue": True
    })

    print("=" * 50)
    for msg in result["messages"]:
        print(f"{msg.type}: {msg.content}")
```

### 复杂工作流示例

```python
"""
带条件分支的工作流
"""
from langgraph.graph import StateGraph, END

class WorkflowState(TypedDict):
    task: str
    result: str
    status: str  # pending / processing / completed / failed

def analyze_task(state: WorkflowState) -> WorkflowState:
    """分析任务类型"""
    task = state["task"]
    # 根据任务类型设置状态
    if "分析" in task:
        return {"status": "processing"}
    elif "生成" in task:
        return {"status": "processing"}
    else:
        return {"status": "completed", "result": "任务类型不明确"}

def execute_task(state: WorkflowState) -> WorkflowState:
    """执行任务"""
    # 执行具体任务
    return {"result": "任务执行完成"}

def check_result(state: WorkflowState) -> str:
    """检查结果，决定下一步"""
    if state.get("result"):
        return "completed"
    else:
        return "failed"

# 构建图
workflow = StateGraph(WorkflowState)
workflow.add_node("analyze", analyze_task)
workflow.add_node("execute", execute_task)
workflow.set_entry_point("analyze")
workflow.add_conditional_edges(
    "analyze",
    check_result,
    {
        "completed": END,
        "failed": END,
        "processing": "execute"
    }
)
workflow.add_edge("execute", END)

app = workflow.compile()
```

---

## 📌 今日要点

- ✅ 理解了纯代码的痛点，为什么需要工作流
- ✅ 掌握了状态机的四大核心概念（状态、事件、转移、动作）
- ✅ 学会了主流工作流框架对比和选型
- ✅ 实战了 LangGraph 工作流构建
- ✅ 掌握了条件分支和复杂工作流设计

---

## 自测题

1. 状态机的四大核心概念是什么？
2. LangGraph 的三大核心组件是什么？
3. 如何选择工作流框架？
4. 如何实现带条件分支的工作流？

---

## 动手作业

1. 用 LangGraph 实现一个带记忆的问答工作流
2. 设计一个三步审批工作流（提交 → 审批 → 完成）
3. 给工作流添加错误处理和重试机制
