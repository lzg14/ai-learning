# AI Agent 学习课程大纲

> 面向零基础学习者，40 天系统掌握 AI Agent 开发技能。
> 使用国产大模型（GLM、DeepSeek、Qwen、Kimi），覆盖 2025-2026 最新技术趋势。

---

## 课程结构概览

| 阶段 | 天数 | 核心内容 |
|------|------|----------|
| **筑基阶段** | Day01-15 | 基础概念、Agent 基础能力、框架与协议 |
| **核心阶段** | Day16-30 | 推理与规划、开发实战、对齐与安全 |
| **进阶阶段** | Day31-40 | 高级架构、前沿专题 |

---

## 筑基阶段（Day01-15）

打好 AI 和 Agent 开发的坚实基础。

### 基础概念（Day01-04）

| Day | 课程名称 | 核心内容 |
|-----|----------|---------|
| [Day01](course/Day01_AI发展脉络与基本概念.md) | AI 发展脉络与基本概念 | AI 发展史、ML/DL/LLM/Agent 概念区分、国产模型崛起、Python 环境搭建 |
| [Day02](course/Day02_大模型基本原理.md) | 大模型基本原理 | Transformer 原理、Attention 机制、预训练+微调范式、国产模型对比、API 调用 |
| [Day03](course/Day03_Prompt工程：从基础到进阶.md) | Prompt 工程：从基础到进阶 | 四层结构、零样本/少样本、结构化输出（JSON 模式）、温度与参数调优、常见坑 |
| [Day04](course/Day04_多轮对话与上下文管理.md) | 多轮对话与上下文管理 | 对话历史维护、滑动窗口、Token 管理、会话持久化、长对话策略 |

### Agent 基础能力（Day05-10）

| Day | 课程名称 | 核心内容 |
|-----|----------|---------|
| [Day05](course/Day05_Function_Calling原理与实战.md) | Function Calling 原理与实战 | 函数调用原理、三种模式（单/并行/链式）、国产模型实现、进阶用法 |
| [Day06](course/Day06_MCP协议与工具生态.md) | MCP 协议与工具生态 | MCP 架构（Client/Server/Tools/Resources/Prompts）、Function Calling 关系、开源生态（2026 更新） |
| [Day07](course/Day07_RAG与知识库入门.md) | RAG 与知识库入门 | Embedding 原理、向量检索、文档切分策略（Chunking）、Chroma/Milvus、LangChain RAG 实战 |
| [Day08](course/Day08_知识图谱与结构化记忆.md) | 知识图谱与结构化记忆 | 知识图谱构建、实体关系抽取、Graph RAG 原理、国产图谱方案 |
| [Day09](course/Day09_多模态大模型.md) | 多模态大模型 | 视觉-语言模型原理、多模态 Agent 架构、国产多模态方案、图文理解实战 |
| [Day10](course/Day10_Context%20Engineering：上下文工程.md) | Context Engineering：上下文工程 | 上下文工程与 Prompt 工程的区别、动态上下文管理、上下文选择/压缩/隔离、常见上下文故障 |

### Agent 框架与协议（Day11-15）

| Day | 课程名称 | 核心内容 |
|-----|----------|---------|
| [Day11](course/Day11_Agent记忆四层架构与上下文管理.md) | Agent 记忆四层架构与上下文管理 | 感官记忆/工作记忆/长期记忆/元认知、Session 管理、摘要压缩 |
| [Day12](course/Day12_AI_Agent框架实践.md) | AI Agent 框架实践 | 主流框架对比（LangChain/CrewAI/LlamaIndex/AutoGen/MetaGPT）、CrewAI 实战、国产框架适配 |
| [Day13](course/Day13_LangChain框架与实战.md) | LangChain 框架与实战 | Chain 组件、Prompt 模板、输出解析器、Agent 构建、Retrieval Chain、实战案例 |
| [Day14](course/Day14_Agent架构与设计模式.md) | Agent 架构与设计模式 | 四种设计模式（ReAct/Plan-and-Execute/AutoGPT/多 Agent）、模式选择决策树、状态机 |
| [Day15](course/Day15_A2A协议与Agent间通信标准.md) | A2A 协议与 Agent 间通信标准 | Agent-to-Agent 协议原理、Agent Card 发现机制、MCP vs A2A 对比、跨平台 Agent 协作实战 |

---

## 核心阶段（Day16-30）

掌握 Agent 核心能力，具备生产级开发水平。

### 推理与规划（Day16-20）

| Day | 课程名称 | 核心内容 |
|-----|----------|---------|
| [Day16](course/Day16_ReAct推理与反思机制.md) | ReAct 推理与反思机制 | 思考-行动-观察循环、自我反思三层次（结果/过程/策略）、Actor+Critic 架构、错误修正 |
| [Day17](course/Day17_任务规划与分解.md) | 任务规划与分解 | 目标分解（Hierarchical Planning）、依赖分析、任务排序、Plan-and-Execute 模式 |
| [Day18](course/Day18_长上下文窗口管理与优化.md) | 长上下文窗口管理与优化 | 长文本处理技术（滑动窗口/稀疏注意力/RingAttention/YaRN）、国产方案对比 |
| [Day19](course/Day19_工作流编排与状态机.md) | 工作流编排与状态机 | 工作流引擎对比、状态机设计模式、LangGraph 实战、错误处理与重试、国产编排方案 |
| [Day20](course/Day20_多Agent协作系统设计.md) | 多 Agent 协作系统设计 | 角色化 Agent 设计五原则、流水线/层级/对等三种协作模式、Agent 间通信机制、软件开发团队实战 |

### Agent 开发实战（Day21-25）

| Day | 课程名称 | 核心内容 |
|-----|----------|---------|
| [Day21](course/Day21_代码Agent开发实战.md) | 代码 Agent 开发实战 | 代码 Agent 架构（规划/编写/执行/调试）、Sandbox 执行环境、测试生成、Coding Agent 优化 |
| [Day22](course/Day22_数据Agent与分析助手.md) | 数据 Agent 与分析助手 | 数据分析 Agent 架构、自然语言查询、报表生成、数据可视化、国产分析方案 |
| [Day23](course/Day23_Computer_Use与浏览器自动化Agent.md) | Computer Use 与浏览器自动化 Agent | Browser Use 原理、Playwright + CDP 控制、视觉驱动的 UI 交互、结构化数据提取 |
| [Day24](course/Day24_Agent生产化与可观测性.md) | Agent 生产化与可观测性 | 生产环境 Agent 监控（Trace/Span）、Langfuse 集成、关键指标（延迟/成本/准确率）、A/B 测试与灰度发布、评估体系构建 |
| [Day25](course/Day25_模型部署与推理优化.md) | 模型部署与推理优化 | 推理框架对比（vLLM/TGI/SGLang/llama.cpp）、PagedAttention、量化技术（GPTQ/AWQ/GGUF）、K8s 部署、AI 网关、国产部署方案 |

### 对齐与安全（Day26-30）

| Day | 课程名称 | 核心内容 |
|-----|----------|---------|
| [Day26](course/Day26_大模型微调入门.md) | 大模型微调入门 | 什么时候用微调 vs RAG、SFT 全流程（数据准备/训练/评估/部署）、LoRA 原理与参数调优、LLaMA Factory 实战、微调+RAG 最佳实践 |
| [Day27](course/Day27_LLM评测与质量保证.md) | LLM 评测与质量保证 | 评测指标（BLEU/ROUGE/METEOR）、评测数据集（C-Eval/CMMLU/MMLU）、国产评测基准、自动化评测 Pipeline |
| [Day28](course/Day28_大模型幻觉应对指南.md) | 大模型幻觉应对指南 | 幻觉原因分析（知识边界/训练数据/解码策略）、检测方法、RAG 校验、置信度校准、事实核查机制 |
| [Day29](course/Day29_对齐技术从RLHF到DPO.md) | 对齐技术：从 RLHF 到 DPO | RLHF 原理（SFT→奖励模型→PPO）、DPO 算法推导、GRPO 等最新方法、国产对齐方案 |
| [Day30](course/Day30_大模型安全与防御体系.md) | 大模型安全与防御体系 | Prompt 注入防御、越狱攻防、数据隐私保护、权限控制、安全评估框架、红队测试、国产安全方案 |

---

## 进阶阶段（Day31-40）

探索前沿技术，完成综合实战。

### 高级架构（Day31-35）

| Day | 课程名称 | 核心内容 |
|-----|----------|---------|
| [Day31](course/Day31_事件驱动架构.md) | 事件驱动架构 | Event-Driven 模式在 Agent 中的应用、消息队列（Kafka/RabbitMQ）、事件溯源、国产中间件 |
| [Day32](course/Day32_Agent插件生态与开放平台设计.md) | Agent 插件生态与开放平台设计 | 插件机制设计、API 网关、开发者平台、MCP 插件市场、国产开放平台 |
| [Day33](course/Day33_Agent推理优化.md) | Agent 推理优化 | 推理加速（批量处理/投机解码/缓存）、智能路由、成本优化策略、国产优化方案 |
| [Day34](course/Day34_具身智能.md) | 具身智能 | 三层架构（感知/决策/行动）、主流机器人平台（宇树/小米）、世界模型、具身 Agent 开发框架 |
| [Day35](course/Day35_Agent记忆系统进阶.md) | Agent 记忆系统进阶 | 记忆检索优化、遗忘机制、长期学习、向量记忆压缩、记忆与个性定制 |

### 前沿与收官（Day36-40）

| Day | 课程名称 | 核心内容 |
|-----|----------|---------|
| [Day36](course/Day36_自动规划与推理.md) | 自动规划与推理 | 自动规划算法、PDDL、Task Decomposition 前沿方法、Tree-of-Thought |
| [Day37](course/Day37_Agent自我改进与终身学习.md) | Agent 自我改进与终身学习 | 在线学习、自我优化循环、经验回放、持续迭代、Agent 能力评估与进化 |
| [Day38](course/Day38_大规模RAG架构.md) | 大规模 RAG 架构 | 百万级文档检索、分布式 RAG（分片/复制）、混合检索（稀疏+稠密）、RAG 评估框架（RAGAS） |
| [Day39](course/Day39_Agent的未来发展趋势与技术展望.md) | Agent 未来趋势与技术展望 | 模型层趋势（开源主导/小模型崛起）、Agent 架构趋势、行业渗透预测、创业与个人机会 |
| [Day40](course/Day40_综合实战：端到端多Agent系统.md) | 综合实战：端到端多 Agent 系统 | 基于所学构建完整多 Agent 系统：需求分析→架构设计→编码实现→测试部署→性能优化→复盘总结 |

---

## 扩展阅读

| 编号 | 内容 |
|------|------|
| Ext01 | 当工具代替我们思考——AI 时代的认知与决策 |
| Ext02 | AI 时代的关系与身份——真实的人与真实的你 |
| Ext03 | AI 时代的社会结构——工作、民主与战争的未来 |
| Ext04 | AI 时代的探索与未来——好奇心与百年后的世界 |
