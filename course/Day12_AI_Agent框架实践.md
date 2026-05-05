# Day 12：AI Agent 框架实践

> 主流 Agent 框架（LangChain / CrewAI / LlamaIndex）对比，实战用 CrewAI 构建多 Agent 团队。

## Part 1：Agent 框架全景对比

### 主流框架一览

| 框架 | 定位 | 上手难度 | 国产适配 | 代表场景 |
|---|---|---|---|---|
| **LangChain** | 全能型框架 | ⭐⭐⭐ 中等 | ✅ 智谱/DeepSeek 支持 | 通用 Agent 开发 |
| **LlamaIndex** | 知识检索 RAG | ⭐⭐ 中等 | ✅ 支持 | RAG、知识库 |
| **CrewAI** | 多 Agent 编排 | ⭐⭐ 简单 | ✅ 支持 | 团队协作、流水线 |
| **AutoGen** | 多 Agent 对话 | ⭐⭐⭐ 中等 | ⚠️ 部分支持 | 多 Agent 对话 |
| **MetaGPT** | 软件开发团队 | ⭐⭐⭐⭐ 复杂 | ⚠️ 实验性 | 智能软件开发 |
| **ChatDev** | 虚拟软件公司 | ⭐⭐⭐ 复杂 | ⚠️ 部分支持 | 完整软件开发流程 |

### 框架选择指南

```plaintext
需求判断：
├── 构建 RAG 知识库？     → LlamaIndex
├── 多 Agent 团队协作？  → CrewAI
├── 通用 LLM 应用？      → LangChain
├── 多 Agent 对话协作？  → AutoGen
└── 完整软件生成？       → MetaGPT / ChatDev
```

---

## Part 2：CrewAI 多 Agent 编排

### CrewAI 核心概念

```
CrewAI 四大核心概念：
Crew = Agents（角色）+ Tasks（任务）+ Process（流程）+ Crew（团队）

- Agent：定义角色、目标、工具
- Task：具体工作单元
- Process：Agents 之间的协作方式
- Crew：整合所有组件的团队
```

### 安装

```bash
pip install crewai crewai-tools
```

### Agent 定义示例

```python
from crewai import Agent, Task, Crew, Process

researcher = Agent(
    role="高级研究员",
    goal="收集高质量的行业信息",
    backstory="""
    你是一位有10年经验的行业研究员，
    擅长从公开资料中提取关键信息，
    写简洁准确的分析报告。
    """,
    tools=[SerplyWrapper],
    verbose=True
)

writer = Agent(
    role="技术文档工程师",
    goal="将复杂技术内容转化为易懂的文档",
    backstory="你是一位专业的技术写作者，擅长用简洁清晰的语言解释复杂概念。",
    verbose=True
)
```

### Task 定义

```python
research_task = Task(
    description="研究大模型在金融领域的应用现状",
    expected_output="一份包含5个应用场景的分析报告",
    agent=researcher
)

write_task = Task(
    description="将研究报告转化为通俗易懂的学习文档",
    expected_output="一份适合初学者的学习指南",
    agent=writer
)
```

---

## Part 3：实战——CrewAI 团队 Agent

### 完整 CrewAI 项目

```python
"""
CrewAI 团队 Agent 实战
课程设计团队：课程设计师 → 内容编写 → 质量审核
"""
import os
from dotenv import load_dotenv
from crewai import Agent, Task, Crew, Process
from langchain_community.chat_models import ChatZhipuAI

load_dotenv()

# 使用国产模型
llm = ChatZhipuAI(
    api_key=os.getenv("ZHIPU_API_KEY"),
    model="glm-4-flash",
    temperature=0.0
)

# ========== Agent 定义 ==========

curriculum_designer = Agent(
    role="课程设计师",
    goal="设计系统性的学习课程",
    backstory="你是一位资深的课程设计师，有8年AI教育经验。",
    llm=llm,
    verbose=True
)

content_writer = Agent(
    role="内容编写工程师",
    goal="将课程内容转化为高质量教学文档",
    backstory="你是一位专业的技术写作者，擅长用通俗易懂的语言解释复杂概念。",
    llm=llm,
    verbose=True
)

quality_reviewer = Agent(
    role="质量审核员",
    goal="确保学习内容准确、完整、有价值",
    backstory="你是一位严谨的AI专家，有深度学习研究背景。",
    llm=llm,
    verbose=True
)

# ========== Task 定义 ==========

design_task = Task(
    description="设计一个《AI大模型入门》5天学习课程",
    expected_output="课程大纲（JSON格式），包含5天的学习计划",
    agent=curriculum_designer
)

write_task = Task(
    description="根据课程大纲，撰写详细的Day 1-5学习内容",
    expected_output="5份完整的日学习文档",
    agent=content_writer
)

review_task = Task(
    description="审核编写的内容，指出任何错误或需要改进的地方",
    expected_output="审核报告，指出具体问题和改进建议",
    agent=quality_reviewer
)

# ========== Crew 组装 ==========

learning_crew = Crew(
    agents=[curriculum_designer, content_writer, quality_reviewer],
    tasks=[design_task, write_task, review_task],
    process=Process.hierarchical,  # 层级协作（管理者→执行者→审核）
    manager_llm=llm
)

# 启动团队
result = learning_crew.kickoff(inputs={"topic": "AI大模型与Prompt工程入门"})
print(result)
```

### Process 模式对比

| 模式 | 说明 |
|---|---|
| **Sequential** | 顺序执行，一个接一个 |
| **Hierarchical** | 有管理者 Agent 协调 |
| **Tripolar** | 多个 Agent 同时处理同一任务 |

---

## Part 4：框架选型与落地建议

### 选型决策树

```plaintext
项目类型？
├── 快速原型 / POC？
│   ├── LangChain（最灵活）
│   └── CrewAI（上手快）
│
├── RAG 知识库？
│   └── LlamaIndex（专门优化）
│
├── 多 Agent 团队协作？
│   ├── 简单团队 → CrewAI
│   ├── 复杂对话 → AutoGen
│   └── 软件开发 → MetaGPT
│
└── 生产级应用？
    ├── LangChain（生态最成熟）
    └── LangGraph（状态机，适合复杂流程）
```

### 国产模型适配

```python
# CrewAI + 国产模型适配
from crewai import LLM

# 智谱适配
crew_llm = LLM(
    model="glm-4-flash",
    api_key=YOUR_KEY,
    base_url="https://open.bigmodel.cn/api/paas/v4"
)

# DeepSeek 适配
crew_llm = LLM(
    model="deepseek-chat",
    api_key=YOUR_KEY,
    base_url="https://api.deepseek.com/v1"
)
```

---

## 📌 今日要点

- ✅ 掌握了主流 Agent 框架（LangChain / CrewAI / LlamaIndex / AutoGen / MetaGPT）
- ✅ 学会了 CrewAI 的 Agent / Task / Crew / Process 四大概念
- ✅ 实战了 CrewAI 团队 Agent（课程设计→内容编写→质量审核）
- ✅ 理解了 Process 三种模式（Sequential / Hierarchical / Tripolar）

---

## 自测题

1. CrewAI 四大核心概念是什么？
2. Process.Hierarchical 和 Process.Sequential 的区别？
3. CrewAI 如何接入国产模型（智谱/DeepSeek）？
4. 什么场景适合用 LlamaIndex 而不是 LangChain？

---

## 动手作业

1. 用 CrewAI 构建一个"论文写作团队"（researcher / writer / reviewer / editor）
2. 对比 LangChain Agent 和 CrewAI 在同一任务上的表现
3. 用 LlamaIndex 构建一个本地知识库 RAG 系统

## 📌 课程更新 (2026-05-06)

> 以下内容由 AI 评审（≥70分）推荐补充

### 多Agent协作（评分：88）
- **标题：** 智能体学习12:多智能体协作(Multi-Agent Collaboration)  
- **来源：** https://blog.csdn.net/weixin_519 Extra  
- **推荐理由：** 多Agent协作是2025年AI Agent前沿方向
