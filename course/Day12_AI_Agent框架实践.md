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

## Part 5：低代码平台——Coze / Dify / n8n

> 本节内容整理自 Datawhale 开源教程 [hello-agents 第五章](https://github.com/datawhalechina/hello-agents)，有删改。

### 5.1 为什么需要低代码平台

纯代码开发 Agent 的维护成本和开发周期会随着业务逻辑复杂度上升而急剧增加。低代码平台的核心价值：

| 价值 | 说明 |
|------|------|
| **降低技术门槛** | 封装复杂技术细节为可视化节点，非技术人员也能参与 Agent 设计 |
| **提升开发效率** | 想法快速验证，POC 构建从"几天编码"压缩到"几小时" |
| **更好的可视化** | 图形化查看 Agent 运行轨迹，直观看到数据在各节点的流转 |
| **沉淀最佳实践** | 平台内置 ReAct 模板、优化后的知识库检索引擎、标准工具集成规范 |

> 低代码平台不是替代代码，而是提供更高层次的抽象——让你从"实现细节"转向"业务逻辑"。

### 5.2 两大流派：流程驱动 vs AI Native

| 流派 | 代表平台 | 本质 | 适用场景 |
|------|----------|------|----------|
| **流程驱动型** | Dify、Coze、n8n | 流程编排为主，LLM 作为后端数据处理 | 快速原型、企业流程自动化 |
| **AI Native 型** | LangChain、AutoGen | 以 AI 能力为核心，真正 AI 驱动 | 复杂推理、多 Agent 协作 |

**hello-agents 教程专注于 AI Native Agent 的构建**——本节补充低代码平台知识，是为了让你在需要快速交付时多一个选择。

### 5.3 Coze（字节跳动）

**核心定位**：零代码 / 低代码 Agent 构建，主打"搭乐高"式体验。

| 维度 | 说明 |
|------|------|
| **目标用户** | AI 应用入门用户、产品经理、运营人员、个人创作者 |
| **核心优势** | 丰富的插件商店、支持一键发布到抖音/飞书/微信公众号 |
| **Agent 类型** | 单 Agent（自主规划模式）、单 Agent（对话流模式）、多 Agent 模式 |
| **资源库** | 工作流、知识库、卡片、提示词库等"装备"集中管理 |

**Coze 工作台核心模块**：

```
左侧边栏
├── 项目空间    — 所有已开发/复制的 Agent 或应用
├── 资源库      — 工作流、知识库、卡片、提示词库
│               "模型决定下限，资源库决定 Agent 的上限"
├── 效果评估    — Agent 表现追踪
├── 空间配置    — 插件/模型统一管理
└── API 管理    — 企业集成
```

**与代码框架的核心区别**：Coze 已经封装好了工具链和发布渠道，不需要写一行代码就能跑一个接入微信/抖音的 Agent。

### 5.4 Dify（开源）

**核心定位**：开源、全功能 LLM 应用开发与运营平台，覆盖原型→生产部署全流程。

| 维度 | 说明 |
|------|------|
| **目标用户** | 有一定技术背景的开发者、需要构建可扩展企业级 AI 应用的团队 |
| **核心能力** | Agent 工作流、RAG Pipeline、数据标注、微调 |
| **部署方式** | 支持私有化部署，代码完全开源 |
| **特点** | 后端服务 + 模型运营一体化，为企业级应用提供坚实底座 |

**Dify vs Coze**：

| 对比项 | Dify | Coze |
|--------|------|------|
| 开源 | ✅ 完全开源 | ❌ 闭源 |
| 部署 | 支持私有化 | 仅云端 / 国内版 |
| 多 Agent | 支持 | 支持 |
| 生态 | 社区活跃度较低 | 插件生态丰富 |
| 适用场景 | 企业内部 / 私有化需求 | 快速 toC 应用 |

### 5.5 n8n（开源工作流自动化）

**核心定位**：本质是工作流自动化工具，不是专门的 LLM 平台，近年来积极集成 AI 能力。

| 维度 | 说明 |
|------|------|
| **目标用户** | 需要将 AI 能力深度嵌入现有业务流程、实现高度定制自动化的开发者和企业 |
| **核心优势** | "连接"能力——数百个预设节点，连接各类 SaaS / 数据库 / API |
| **LLM 集成** | 在工作流中嵌入 LLM 节点，作为整个自动化链的一环 |
| **学习曲线** | 相对陡峭（功能多而灵活） |
| **特点** | 非专门 LLM 平台，但 AI 集成能力已非常成熟 |

**n8n vs Dify vs Coze 横向对比**：

| 维度 | Coze | Dify | n8n |
|------|------|------|-----|
| **定位** | 零代码 Agent 平台 | LLM 应用开发平台 | 工作流自动化 + AI |
| **开源** | ❌ | ✅ | ✅ |
| **私有部署** | ❌ | ✅ | ✅ |
| **LLM 专用能力** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **连接器数量** | 中等 | 中等 | ⭐⭐⭐⭐⭐（数百个） |
| **上手难度** | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **适用场景** | 快速 toC Agent | 企业级 LLM 应用 | 复杂业务自动化 |

### 5.6 框架 + 低代码平台综合选型决策树

```plaintext
项目类型？
├── 快速原型 / POC？
│   ├── 纯 API 调用 → Coze（零代码，即开即用）
│   ├── 需要 RAG → Dify（内置 Pipeline）
│   └── 复杂多系统串联 → n8n（连接器最丰富）
│
├── AI Native Agent 开发（代码为主）？
│   ├── 通用 LLM 应用 → LangChain
│   ├── 多 Agent 团队 → CrewAI
│   ├── RAG 知识库 → LlamaIndex
│   └── 复杂状态机 → LangGraph
│
├── 企业内部 / 私有化需求？
│   └── Dify（开源可私有部署）
│
└── 生产级应用？
    ├── 快速交付 → 低代码平台选一个
    └── 深度定制 → LangChain / LangGraph
```

### 5.7 hello-agents 教程的低代码平台学习路径

hello-agents 第五章采用"对比学习"方式讲解这三个平台：

```
第一步：理解为什么需要低代码平台
↓（明确区分流程驱动 vs AI Native）
第二步：逐个上手 Coze / Dify / n8n
↓（各自跑通一个简单 Agent）
第三步：用代码框架复现相同功能
↓（从"用轮子"到"造轮子"）
第四步：对比两种方式的适用场景
```

> 这种对比学习的价值在于：学了低代码平台之后，你不会迷失在"什么都用 Coze/Dify"的舒适区里，而是清楚知道什么时候该用代码框架、什么时候该用低代码平台。

---

## 📌 今日要点

- ✅ 掌握了主流 Agent 框架（LangChain / CrewAI / LlamaIndex / AutoGen / MetaGPT）
- ✅ 学会了 CrewAI 的 Agent / Task / Crew / Process 四大概念
- ✅ 实战了 CrewAI 团队 Agent（课程设计→内容编写→质量审核）
- ✅ 理解了 Process 三种模式（Sequential / Hierarchical / Tripolar）
- ✅ 理解了低代码平台（Coze / Dify / n8n）的适用场景与局限性

---

## 自测题

1. CrewAI 四大核心概念是什么？
2. Process.Hierarchical 和 Process.Sequential 的区别？
3. CrewAI 如何接入国产模型（智谱/DeepSeek）？
4. 什么场景适合用 LlamaIndex 而不是 LangChain？
5. 低代码平台（Coze/Dify/n8n）和代码框架的本质区别是什么？
6. Dify 和 Coze 各自适合什么场景？

---

## 动手作业

1. 用 CrewAI 构建一个"论文写作团队"（researcher / writer / reviewer / editor）
2. 对比 LangChain Agent 和 CrewAI 在同一任务上的表现
3. 用 LlamaIndex 构建一个本地知识库 RAG 系统
4. 用 Coze 或 Dify 零代码搭建一个聊天 Agent，对比和代码框架的体验差异

## 📌 课程更新 (2026-05-06)

> 以下内容由 AI 评审（≥70分）推荐补充

### 多Agent协作（评分：88）
- **标题：** 智能体学习12:多智能体协作(Multi-Agent Collaboration)  
- **来源：** https://blog.csdn.net/weixin_519 Extra  
- **推荐理由：** 多Agent协作是2025年AI Agent前沿方向
