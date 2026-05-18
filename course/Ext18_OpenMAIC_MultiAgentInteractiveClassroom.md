> ⚠️ **数据更新时间：2026年5月**
>
> 📢 **来源**：THU-MAIC 开源项目 | GitHub 17.4k Stars

# Ext18：OpenMAIC——多 Agent 驱动的互动课堂

---

## 一、背景：MOOC 的困局与 MAIC 的野心

MOOC（大型开放在线课程）从 2012 年兴起至今，始终没有真正解决在线学习的核心问题：**被动接受 vs. 主动探索**。视频lecture只能听、白板只能看、测验只能答——学习者始终是观众，不是参与者。

清华大学团队提出了 **MAIC**（Multi-Agent Interactive Classroom）的愿景——将 AI Agent 技术引入课堂生成与互动，让学习变成一场真实发生的对话。

**OpenMAIC** 是该愿景的开源实现：一个将任何主题或文档转化为多 Agent 互动学习体验的平台。核心特点是：AI 老师和 AI 同学同时存在，可以实时讨论、语音讲解、白板绘图、与学习者互动。

> **核心理念**：学习不是单向传递，而是多向对话。OpenMAIC 让知识在师生与同伴的实时互动中自然生长。

---

## 二、系统拓扑：两阶段生成流水线

OpenMAIC 的内容生成采用两阶段流水线：

| 阶段 | 输入 | 输出 | 核心逻辑 |
|------|------|------|----------|
| **大纲生成** | 主题描述 / 参考文档 | 结构化课堂大纲 | AI 分析输入内容，拆解知识结构 |
| **场景生成** | 大纲条目 | 幻灯片 / 测验 / 交互式模拟 / PBL 活动 | 每个大纲条目转化为富媒体场景 |

场景类型覆盖：

| 类型 | 说明 |
|------|------|
| **幻灯片** | AI 老师配合聚光灯和激光笔动作进行语音讲解 |
| **测验** | 交互式测验（单选 / 多选 / 简答），AI 实时判分和反馈 |
| **交互式模拟** | 基于 HTML 的动态实验——物理模拟器、流程图等 |
| **PBL（项目制学习）** | 选择角色，与 AI 智能体协作完成结构化项目 |

---

## 三、核心机制：LangGraph 多 Agent 编排

OpenMAIC 的多 Agent 互动基于 **LangGraph** 实现的状态机编排：

### 3.1 智能体角色

| 角色 | 职责 |
|------|------|
| **AI 老师** | 主导授课、讲解概念、操作白板、回答问题 |
| **AI 同学（多个）** | 发起讨论、参与辩论、提出疑问、制造认知冲突 |

### 3.2 讨论模式

| 模式 | 描述 |
|------|------|
| **课堂讨论** | 智能体主动发起讨论话题，学习者可随时加入或被点名互动 |
| **圆桌辩论** | 多个不同人设的智能体围绕话题展开讨论，配合白板讲解 |
| **自由问答** | 随时提问，AI 老师通过幻灯片、图表或白板进行解答 |

### 3.3 动作引擎

OpenMAIC 的智能体可执行 **28+ 种动作类型**：

- **语音** — TTS 语音讲解，支持多种语音服务商和音色克隆（VoxCPM2）
- **白板绘图** — 逐步推导方程、绘制流程图、直观讲解概念
- **聚光灯 / 激光笔** — 在幻灯片上高亮重点区域
- **元素渲染** — 文本、图片、形状、表格、图表、LaTeX 公式

> 白板是 OpenMAIC 的核心交互媒介——AI 老师可以在共享白板上实时绘图，学习者看到的是完整的推导过程，而非最终结果。

---

## 四、深度交互模式：从听讲走向探索

v0.2.0 引入的深度交互模式是 OpenMAIC 与其他课堂生成工具拉开差距的核心功能：

### 五种交互界面

| 模式 | 描述 | 典型场景 |
|------|------|----------|
| **3D 可视化** | 三维可视化呈现抽象结构 | 化学分子结构、历史场景重建 |
| **模拟实验** | 流程模拟和实验环境，观察动态变化 | 物理电路、经济学供需曲线 |
| **游戏** | 知识小游戏，通过交互挑战加深理解和记忆 | 桌游教学、历史决策模拟 |
| **思维导图** | 结构化知识组织，建立整体概念框架 | 课程总结、框架对比 |
| **在线编程** | 浏览器内编码和即时运行，边写边学边迭代 | Python 入门、数据分析 |

### AI 教师引导

深度交互模式下，AI 老师不只是旁观者——可以**主动操作界面引导学生**：

- 高亮关键区域
- 设置实验条件
- 在恰当时机提供提示
- 引导注意力到特定元素

### 多设备适配

所有交互界面完全响应式——桌面、iPad、手机均可使用。

---

## 五、OpenClaw 集成：在聊天应用里生成课堂

OpenMAIC 集成了 [OpenClaw](https://github.com/openclaw/openclaw)（Ext15 介绍过），支持直接在飞书、Slack、Discord、Telegram、WhatsApp 等 20+ 聊天应用中生成和查看课堂。

**使用方式**：告诉 OpenClaw "教我量子物理" → 搞定。

**两种模式**：

| 模式 | 配置方式 | 适用场景 |
|------|----------|----------|
| **托管模式** | 在 open.maic.chat 获取访问码，配置到 OpenClaw | 零部署，即开即用 |
| **本地部署** | clone 仓库、配置 API Key、启动服务 | 需要定制或私有化部署 |

> OpenMAIC 的 OpenClaw Skill 位于 `skills/openmaic/` 目录，已上架 ClawHub，一行命令安装：`clawhub install openmaic`

---

## 六、导出与扩展

| 格式 | 说明 |
|------|------|
| **PowerPoint (.pptx)** | 可编辑幻灯片，包含图片、图表和 LaTeX 公式 |
| **交互式 HTML** | 自包含网页，含交互式模拟实验 |
| **课堂 ZIP** | 完整课堂导出（课程结构 + 媒体文件） |

### 支持的 LLM 提供商

OpenMAIC 内置了极为丰富的服务商支持：

- OpenAI / Anthropic / Google Gemini / DeepSeek
- 通义千问 Qwen / Kimi / 智谱 GLM / 腾讯混元 / 小米 MiMo / 豆包
- MiniMax（**原生支持**，有专属环境变量配置示例）
- Ollama（本地）/ Lemonade（本地全栈）

---

## 七、技术架构速览

```
OpenMAIC/
├── app/                        # Next.js App Router
│   ├── api/                    #   约 18 个服务端 API 端点
│   │   ├── generate/          #     场景生成流水线
│   │   ├── generate-classroom/ #     异步课堂生成
│   │   └── chat/               #     多智能体讨论（SSE 流式）
│   └── classroom/[id]/         #   课堂回放页面
│
├── lib/
│   ├── generation/             #   两阶段课堂生成流水线
│   ├── orchestration/          #   LangGraph 多智能体编排（导演图）
│   ├── playback/               #   回放状态机（idle → playing → live）
│   ├── action/                 #   28+ 种动作执行引擎
│   ├── ai/                     #   LLM 服务商抽象层
│   └── export/                 #   PPTX & HTML 导出
│
├── components/
│   ├── slide-renderer/         #   Canvas 幻灯片编辑器
│   ├── scene-renderers/       #   测验/交互/PBL 场景渲染器
│   └── whiteboard/            #   SVG 白板绘图系统
│
└── packages/
    ├── pptxgenjs/              #   定制化 PowerPoint 生成
    └── mathml2omml/            #   MathML → Office Math 转换
```

---

## 八、总结与反思

### OpenMAIC 的本质

OpenMAIC 真正推进的是 **AI Agent 在教育场景的工程化落地**——不是 PPT 生成器，而是一个完整的多 Agent 协作系统：多个 AI 角色实时互动、生成内容、响应学习者行为。它比单纯的 RAG 或单 Agent 更接近"真实课堂"的交互体验。

### 核心优势

| 优势 | 说明 |
|------|------|
| **多 Agent 原生** | LangGraph 状态机编排，非简单的多 Agent 对话 |
| **深交互模式** | 3D/模拟/游戏/编程/思维导图，让学习者真正动手 |
| **OpenClaw 集成** | 在聊天应用里直接生成课堂，降低使用门槛 |
| **完整 Export** | PPTX + HTML + ZIP，可接入真实教学流程 |
| **服务商广泛** | 覆盖国内外主流 LLM，MiniMax 也有原生示例 |

### 核心不足

| 局限 | 说明 |
|------|------|
| **生成质量依赖模型** | 弱模型生成的课堂内容质量有限 |
| **部署复杂度** | 需要 Node.js + pnpm + API Key，对非技术用户不友好 |
| **效果评估缺失** | 没有看到系统性的学习效果对比数据 |
| **LangGraph 锁死** | 强依赖 LangGraph，切换编排框架成本高 |

### 更大价值

OpenMAIC 的白板实时绘图 + 多 Agent 讨论机制，理论上可以被提取出来复用到其他多 Agent 场景——不仅仅是课堂，任何需要"AI 逐步推导给用户看"的场景（代码调试、架构设计讨论、案例分析）都可以借鉴这套动作引擎设计。

---

## 参考资料

- [OpenMAIC GitHub（THU-MAIC）](https://github.com/THU-MAIC/OpenMAIC)
- [Live Demo：open.maic.chat](https://open.maic.chat/)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [JCST 2026 论文：From MOOC to MAIC](https://jcst.ict.ac.cn/en/article/doi/10.1007/s11390-025-6000-0)

---

## 关联课程

- **Day20** [多 Agent 系统开发实战](../course/Day20_多Agent系统开发实战.md)——OpenMAIC 是 LangGraph 多 Agent 编排在教育场景的深度实践
- **Day12** [Agent 框架对比](../course/Day12_Agent框架对比.md)——OpenMAIC 的编排层基于 LangGraph 实现，与主流框架的关系值得对照
- **Day14** [Agent 架构与设计模式](../course/Day14_Agent架构与设计模式.md)——多角色（老师/同学）协作与动作执行引擎的设计思路
- **Ext15** [OpenClaw——首个"一人独角兽"的开工拉齐](../course/Ext15_OpenClaw_首个一人独角兽.md)——OpenMAIC 内置了 OpenClaw 集成，两者协同使用
