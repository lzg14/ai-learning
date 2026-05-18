# Day 13：LangChain 框架与实战

> LangChain 是构建 LLM 应用的核心框架。本课从基础组件到 Agent 构建，再到 LangGraph 状态机，全面掌握 LangChain 生态。

## Part 1：LangChain 核心组件

LangChain 四大核心组件：

```
LangChain = Prompt（提示词）+ Model（模型）+ Chain（链）+ Tool（工具）
```

### 1. Prompt（提示词模板）

```python
from langchain.prompts import PromptTemplate

template = "请用{language}语言实现{function}函数"
prompt = PromptTemplate.from_template(template)
formatted = prompt.format(language="Python", function="快速排序")
print(formatted)  # 请用Python语言实现快速排序函数
```

### 2. Model（模型）

```python
from langchain_community.chat_models import ChatZhipuAI
import os

llm = ChatZhipuAI(
    api_key=os.getenv("ZHIPU_API_KEY"),
    model="glm-4-flash",
    temperature=0.0
)
```

### 3. Chain（链）

```python
from langchain.chains import LLMChain

chain = LLMChain(llm=llm, prompt=chat_prompt)
result = chain.invoke({
    "role": "数据分析师",
    "skill": "Python和SQL",
    "task": "分析这份CSV文件"
})
```

### 4. Tool（工具）

工具是 LangChain 给模型"配备"的能力：

| 工具 | 功能 | 适合场景 |
|------|------|----------|
| **计算器** | 精确数学计算 | 数学题、数据统计 |
| **搜索引擎** | 获取实时信息 | 最新新闻、实时数据 |
| **数据库查询** | 查数据库 | 业务数据查询 |
| **文件读写** | 操作文件 | 数据处理、报告生成 |

---

## Part 2：LCEL 语法与 Chain 组合

### LCEL（LangChain Expression Language）

LCEL 是 LangChain 的声明式语法，用 `|` 管道符连接组件：

```python
from langchain_core.output_parsers import StrOutputParser

# LCEL 管道：Prompt → Model → OutputParser
chain = prompt | llm | StrOutputParser()

result = chain.invoke({"language": "Python", "function": "快速排序"})
print(result)  # 直接输出字符串
```

### 多 Chain 组合

```python
from langchain.chains import SimpleSequentialChain

# Chain 1：生成问题
chain1 = LLMChain(
    llm=llm,
    prompt=PromptTemplate.from_template("关于{topic}，提出一个专业问题")
)

# Chain 2：回答问题
chain2 = LLMChain(
    llm=llm,
    prompt=PromptTemplate.from_template("请回答这个问题：{question}")
)

# 串联
overall_chain = SimpleSequentialChain(chains=[chain1, chain2])
result = overall_chain.invoke("大模型")
# Chain 1 先提问题 → Chain 2 再回答问题
```

---

## Part 3：Agent 构建

### Agent 核心概念

Agent = LLM + Tools + 推理循环

```python
from langchain.agents import initialize_agent, Tool
from langchain.tools import tool

# 工具定义
@tool
def search(query: str) -> str:
    """搜索互联网"""
    return f"关于'{query}'的搜索结果..."

@tool
def calculate(expression: str) -> str:
    """计算数学表达式"""
    try:
        return str(eval(expression))
    except:
        return "计算错误"

# 工具列表
tools = [
    Tool(name="search", func=search, description="搜索互联网信息"),
    Tool(name="calculate", func=calculate, description="数学计算"),
]

# 初始化 Agent
agent = initialize_agent(
    tools=tools,
    llm=llm,
    agent="zero-shot-react-description",  # ReAct 推理模式
    verbose=True,
    max_iterations=5,  # 防止无限循环
)

# 执行
result = agent.invoke({"input": "北京人口是多少？人口增长率是多少？"})
```

### Agent 类型对比

| Agent 类型 | 特点 | 适用场景 |
|-----------|------|----------|
| **zero-shot-react-description** | 零样本推理，描述驱动 | 通用任务 |
| **conversational-react** | 对话式，带记忆 | 多轮对话 |
| **self-ask-with-search** | 自我提问+搜索 | 需要外部知识的任务 |
| **structured-chat-zero-shot** | 结构化输出 | 需要固定格式的任务 |

---

## Part 4：对话记忆

```python
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationChain

# 对话记忆
memory = ConversationBufferMemory(
    memory_key="chat_history",
    return_messages=True
)

# Conversational Agent
conversational_agent = initialize_agent(
    tools=tools,
    llm=llm,
    agent="conversational-react-description",
    memory=memory,
    verbose=True
)

# 多轮对话
conversational_agent.run("北京今天天气怎么样？")
conversational_agent.run("需要带伞吗？")  # Agent 能记住上文
```

### 记忆类型对比

| 类型 | 机制 | 适合场景 |
|------|------|----------|
| **BufferMemory** | 存储完整历史 | 短对话 |
| **SummaryMemory** | 自动摘要历史 | 长对话 |
| **VectorStoreMemory** | 向量检索历史 | 超长对话 |
| **TokenBufferMemory** | 按 Token 数限制 | 成本敏感场景 |

---

## Part 5：Retrieval Chain 与 RAG

### 文档加载与切分

```python
from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter

# 加载文档
loader = TextLoader("./docs/guide.txt")
documents = loader.load()

# 切分
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50
)
chunks = text_splitter.split_documents(documents)
```

### 构建 RAG Chain

```python
from langchain_community.embeddings import ZhipuAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain.chains import RetrievalQA

# Embedding + 向量库
embeddings = ZhipuAIEmbeddings(api_key=os.getenv("ZHIPU_API_KEY"))
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    persist_directory="./chroma_db"
)

# RAG Chain
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",
    retriever=vectorstore.as_retriever(search_kwargs={"k": 3}),
    return_source_documents=True
)

# 查询
result = qa_chain({"query": "模型支持哪些语言？"})
print(result['result'])
```

---

## Part 6：LangGraph 状态机简介

LangGraph 是 LangChain 的扩展，支持有状态、有循环的工作流。

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, List

# 定义状态
class AgentState(TypedDict):
    messages: List[str]
    next_step: str

# 定义节点
def step1(state: AgentState) -> AgentState:
    state["messages"].append("执行步骤1")
    state["next_step"] = "step2"
    return state

def step2(state: AgentState) -> AgentState:
    state["messages"].append("执行步骤2")
    state["next_step"] = "end"
    return state

# 构建图
builder = StateGraph(AgentState)
builder.add_node("step1", step1)
builder.add_node("step2", step2)
builder.set_entry_point("step1")
builder.add_conditional_edges("step1", lambda s: s["next_step"])
builder.add_edge("step2", END)

graph = builder.compile()
result = graph.invoke({"messages": [], "next_step": "step1"})
print(result["messages"])
# ['执行步骤1', '执行步骤2']
```

> LangGraph 适合需要循环、条件分支的复杂 Agent 流程，是 LangChain 生态的未来方向。

---

## Part 5：从零构建 Agent 框架——HelloAgents 设计思路

> 本节内容整理自 Datawhale 开源教程 [hello-agents 第七章](https://github.com/datawhalechina/hello-agents)，有删改。

### 5.1 为什么需要自建框架

市场已有 LangChain、AutoGen 等成熟框架，自建的原因：

| 痛点 | 说明 |
|------|------|
| **过度抽象** | LangChain 为追求通用性引入了大量抽象层和配置项，完成简单任务也需要理解十几个概念 |
| **版本不稳定** | 商业框架频繁变更 API 接口，版本升级后代码跑不通是常态 |
| **黑盒实现** | 核心逻辑封装过紧，难以理解 Agent 内部工作机制，深度定制能力受限 |
| **依赖复杂** | 成熟框架携带大量依赖包，安装包体积大，与其他项目配合时容易冲突 |

**从用户到构建者的能力跃迁**：

- 亲手实现每个组件，真正理解 Agent 的思考过程、工具调用机制
- 完全掌控每一行代码，不受第三方框架设计哲学束缚
- 培养系统设计能力：模块化设计、接口抽象、错误处理

### 5.2 HelloAgents 设计哲学

hello-agents 自研的 HelloAgents 框架围绕一个核心问题设计：**如何在功能完整性和学习友好性之间找到平衡？**

**四大设计原则**：

| 原则 | 说明 |
|------|------|
| **轻量且教学友好** | 核心代码按章节分离，任何有编程基础的人都能完全理解框架工作原理 |
| **基于标准 API** | 基于 OpenAI 标准接口开发，兼容所有兼容 OpenAI API 的服务商 |
| **渐进式学习路径** | 每章在上一章基础上增加功能模块，学习成本自然递增，无概念跳跃 |
| **统一 Tool 抽象** | 除核心 Agent 类外，一切皆是 Tool（Memory/RAG/MCP 均为 Tool） |

### 5.3 HelloAgents 核心架构

```
HelloAgents 架构（简化版）
├── HelloAgentsLLM    — 模型调用层，支持多 Provider（OpenAI/智谱/ModelScope/Ollama）
├── Tool              — 工具抽象基类，一切皆是 Tool
├── Agent             — 核心 Agent 类（推理循环 + 工具调用）
├── Memory            — 记忆系统（ChatHistory/Summary/VectorStore）
├── RAG               — 检索增强生成（作为 Tool 接入）
└── MCP               — 协议支持（作为 Tool 接入）
```

**与 LangChain 的核心区别**：

| 对比项 | LangChain | HelloAgents |
|--------|-----------|-------------|
| 抽象层级 | 多层（Chain/Agent/Tool/Retriever…） | 极简（Agent + Tool 两层） |
| 学习曲线 | 陡峭 | 平缓 |
| Provider 扩展 | 依赖官方集成 | 继承 HelloAgentsLLM Override 即可 |
| 代码可控性 | 黑盒较重 | 全透明 |

### 5.4 自定义 Provider 扩展示例

以新增 ModelScope 支持为例，展示如何通过继承扩展 HelloAgentsLLM：

```python
from hello_agents import HelloAgentsLLM

class MyLLM(HelloAgentsLLM):
    def __init__(self, provider="auto", **kwargs):
        if provider == "modelscope":
            self.provider = "modelscope"
            self.base_url = "https://api-inference.modelscope.cn/v1/"
            self.model = kwargs.get("model") or "Qwen/Qwen2.5-VL-72B-Instruct"
            # ... 解析 API Key 等
            self._client = OpenAI(api_key=self.api_key, base_url=self.base_url)
        else:
            # 其他 Provider 走父类逻辑
            super().__init__(provider=provider, **kwargs)
```

**扩展思路**：
1. 继承 HelloAgentsLLM
2. 在 `__init__` 中拦截特定 provider
3. 其他情况 `super().__init__(...)` 委托给父类

### 5.5 自建框架 vs 使用成熟框架

| 场景 | 推荐 |
|------|------|
| 快速验证想法 / POC | 直接用 LangChain/CrewAI，不重复造轮子 |
| 学习 Agent 内部原理 | 按 hello-agents 第七章思路自己搭 |
| 企业级生产项目 | 评估维护成本后选 LangChain/LangGraph |
| 特定领域深度定制 | 在成熟框架基础上二次开发，或自建精简框架 |

> **核心认知**：自建框架的目的不是替代 LangChain，而是通过亲手构建，理解 Agent 工作原理。在实际工作中，"用轮子"和"造轮子"都要会——根据场景选择合适的策略。

---

## 今日要点

- 理解了 LangChain 四大核心组件：Prompt、Model、Chain、Tool
- 掌握了 LCEL 声明式语法和 Chain 组合
- 学会了 Agent 构建（工具定义、初始化、执行）
- 理解了对话记忆的多种类型和使用场景
- 掌握了 RAG Chain 的完整构建流程
- 了解了 LangGraph 状态机的基本用法
- 理解了自建 Agent 框架（HelloAgents）的设计思路与适用场景

---

## 自测题

1. LangChain 的四大核心组件是什么？
2. LCEL 语法使用什么符号连接组件？有什么优势？
3. Agent 和普通 Chain 的核心区别是什么？
4. RAG Chain 的构建需要哪几个步骤？
5. LangGraph 解决了 LangChain 的什么局限性？

---

## 动手作业

1. 用 LCEL 构建一个"翻译 Chain"（中文→英文→中文，对比一致性）
2. 构建一个带搜索和计算工具的 Agent，解决一个实际问题
3. 用 RetrievalQA 构建一个文档问答系统