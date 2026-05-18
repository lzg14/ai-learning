# Ext19：DeepResearch——自动深度研究智能体复现

> 本扩展阅读整理自 Datawhale 开源教程 [hello-agents 第十四章](https://github.com/datawhalechina/hello-agents)，有删改。
>
> 关联阅读：[Ext17 ARIS——让 Claude Code 在你睡觉时做科研](./Ext17_ARIS_AutoResearchInSleep.md)

## 背景

为什么需要 Deep Research Agent？

| 痛点 | 说明 |
|------|------|
| **信息过载** | 搜索引擎返回上千条结果，需要逐条点击阅读才能找到有用信息 |
| **缺乏结构** | 找到的信息碎片化，没有系统性组织 |
| **重复劳动** | 每次研究新话题都要重复"搜索→阅读→总结→整理"的过程 |

**Deep Research Agent 的核心价值**：

1. **节省时间**：将 1-2 小时的研究工作压缩到 5-10 分钟
2. **提升质量**：系统化的研究过程，避免遗漏重要信息
3. **可追溯**：记录所有搜索结果和来源，便于验证和引用
4. **可扩展**：轻松添加新的搜索引擎、数据源和分析工具

---

## 一、TODO-Driven 研究范式

### 1.1 核心思想

传统搜索只能回答单一问题，而深度研究需要回答一系列相关问题。**TODO-Driven 研究范式**将复杂研究主题分解为多个子任务（TODO），逐一执行并整合结果。

**核心公式**：将"研究"这一复杂任务，转化为**"规划 → 执行 → 整合"**的过程。

### 1.2 传统方法 vs TODO-Driven 方法

**传统搜索方式**：

```
用户输入：Datawhale 是什么样的组织？
搜索引擎：返回 10-20 条链接
用户：逐条点击链接、阅读内容、做笔记
结果：信息碎片化，缺乏系统性
```

**TODO-Driven 方式：系统化研究**：

```
用户输入：Datawhale 是什么样的组织？

系统规划：
  ├─ TODO 1：Datawhale 基本信息（组织定位）
  ├─ TODO 2：Datawhale 主要项目（核心内容）
  ├─ TODO 3：Datawhale 社区文化（价值观）
  └─ TODO 4：Datawhale 影响力（社会贡献）

执行：逐一完成每个 TODO
整合：生成结构化研究报告
```

### 1.3 三阶段研究流程

```
┌─────────────────────────────────────────────────────┐
│              TODO-Driven 三阶段研究流程               │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Stage 1：规划（Research Planning Expert）           │
│    输入：研究主题 + 当前日期                          │
│    输出：3-5 个子任务的 JSON 列表                     │
│    每个子任务包含：title / intent / query            │
│                                                     │
│  Stage 2：执行（Task Summarization Expert）          │
│    对每个 TODO 执行：搜索 → 摘要 → 结构化知识         │
│    工具支持：SearchTool + NoteTool                   │
│                                                     │
│  Stage 3：报告（Report Writing Expert）              │
│    整合所有 TODO 的摘要                              │
│    生成最终研究报告                                   │
│    输出格式：Markdown + 可视化                       │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 二、三 Agent 协作架构

### 2.1 Agent 职责划分

| Agent | 职责 | 输入 | 输出 |
|-------|------|------|------|
| **TODO Planner**（研究规划专家） | 将研究主题分解为 3-5 个子任务 | 研究主题 + 当前日期 | JSON 格式的 TODO 列表 |
| **Task Summarizer**（任务摘要专家） | 对每个 TODO 执行搜索和摘要 | 单个 TODO 任务 | 结构化摘要 |
| **Report Writer**（报告写作专家） | 整合所有摘要生成最终报告 | 全部 TODO 摘要 | Markdown 格式研究报告 |

### 2.2 Agent 协作模式

三个 Agent 之间是**顺序协作**关系：

```
用户输入研究主题
      ↓
[TODO Planner]
生成 3-5 个子任务
      ↓
[Task Summarizer] × N
逐一执行每个子任务的搜索和摘要
      ↓
[Report Writer]
整合所有摘要，生成最终报告
      ↓
返回用户
```

**顺序协作的特点**：

| 特点 | 说明 |
|------|------|
| **线性流程** | Agent 按固定顺序执行 |
| **输入输出清晰** | 每个 Agent 的输入来自上一个 Agent 的输出 |
| **无并发** | 同一时间只有一个 Agent 在工作 |

### 2.3 DeepResearchAgent 核心实现

```python
class DeepResearchAgent:
    def run(self, research_topic: str) -> str:
        # 1. 规划阶段
        self._emit_event({"type": "status", "message": "Planning research tasks..."})
        todo_list = self.planner.plan_todo_list(research_topic)
        self._emit_event({"type": "tasks", "tasks": todo_list})

        # 2. 执行阶段
        task_summaries = []
        for task in todo_list:
            self._emit_event({
                "type": "status",
                "message": f"Researching: {task['title']}"
            })
            summary = self.summarizer.summarize_task(task)
            task_summaries.append(summary)

        # 3. 报告阶段
        self._emit_event({"type": "status", "message": "Generating final report..."})
        final_report = self.writer.write_report(task_summaries)

        return final_report
```

---

## 三、工具系统设计

### 3.1 SearchTool 多源搜索

Deep Research 的核心是搜索能力。系统支持多种搜索引擎：

| 搜索引擎 | 特点 | 适用场景 |
|---------|------|---------|
| **DuckDuckGo** | 免费、无需 API Key | 快速原型、个人使用 |
| **Tavily** | 专为 AI 设计，结构化结果 | 正式项目 |
| **Perplexity** | 实时网络+大模型摘要 | 需要最新信息 |
| **SearXNG** | 开源自托管 | 隐私敏感场景 |
| **Advanced 模式** | 多引擎组合，自动路由 | 高质量研究 |

### 3.2 NoteTool 笔记管理

每个子任务执行过程中的搜索结果、摘要内容，都通过 NoteTool 管理：

```python
class NoteTool:
    """笔记工具——存储和管理研究过程中的中间结果"""

    def add_note(self, task_id: str, content: str, note_type: str = "search_result"):
        """添加笔记"""
        pass

    def get_notes(self, task_id: str) -> List[dict]:
        """获取某任务的全部笔记"""
        pass

    def summarize_notes(self, task_id: str) -> str:
        """对笔记进行摘要"""
        pass
```

### 3.3 ToolRegistry 工具注册

所有工具通过统一的 ToolRegistry 管理，支持动态添加新工具：

```python
class ToolRegistry:
    """工具注册中心——统一管理所有可用工具"""

    def __init__(self):
        self.tools = {}

    def register(self, name: str, tool: BaseTool):
        """注册工具"""
        self.tools[name] = tool

    def get(self, name: str) -> BaseTool:
        """获取工具"""
        return self.tools.get(name)

    def list_tools(self) -> List[str]:
        """列出所有工具"""
        return list(self.tools.keys())
```

---

## 四、技术架构

### 4.1 四层架构

```
┌─────────────────────────────────────────────────────┐
│              Deep Research Agent 技术架构             │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Frontend Layer（Vue3 + TypeScript）                 │
│    - 全屏模态对话框 UI                               │
│    - Markdown 结果可视化                             │
│    - 实时进度展示                                    │
│                                                     │
│  Backend Layer（FastAPI）                            │
│    - API 路由：/research/stream（SSE 流式返回）       │
│    - 研究状态管理                                    │
│                                                     │
│  Agent Layer（HelloAgents 框架）                    │
│    - 三个专业 Agent：Planner / Summarizer / Writer  │
│    - 两个核心工具：SearchTool / NoteTool             │
│                                                     │
│  External Service Layer                             │
│    - 搜索引擎（DuckDuckGo / Tavily / Perplexity）    │
│    - LLM 提供商（OpenAI / 智谱 / DeepSeek）           │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 4.2 数据流过程

```
1. 用户输入：在前端输入研究主题
2. 前端发送：通过 SSE 连接 /research/stream
3. 后端接收：FastAPI 创建研究状态
4. 规划阶段：调用研究规划 Agent，分解为 3 个子任务
5. 执行阶段：对每个子任务调用搜索工具和摘要 Agent
6. 报告阶段：调用报告生成 Agent，整合所有结果
7. 流式返回：通过 SSE 实时推送进度和结果到前端
```

---

## 五、前端交互设计

### 5.1 全屏模态对话框

Deep Research 是一个**沉浸式研究体验**，用户输入研究主题后，系统以全屏模态方式展示研究过程和结果。

**核心 UI 元素**：

| 元素 | 说明 |
|------|------|
| 研究主题输入框 | 顶部，支持多行输入 |
| 实时进度面板 | 显示当前阶段和 TODO 完成状态 |
| 流式结果区 | Markdown 格式，实时滚动更新 |
| 可视化图表 | 研究结果的数据可视化 |

### 5.2 实时进度展示

通过 SSE（Server-Sent Events）实现实时进度推送：

```javascript
// 前端 SSE 连接
const eventSource = new EventSource('/research/stream', {
  method: 'POST',
  body: JSON.stringify({ topic: 'Datawhale 是什么样的组织' })
});

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data);

  if (data.type === 'status') {
    // 更新状态：正在规划 / 正在研究 TODO 1 / 正在生成报告
    updateStatus(data.message);
  } else if (data.type === 'tasks') {
    // 展示 TODO 列表
    renderTaskList(data.tasks);
  } else if (data.type === 'partial_report') {
    // 流式追加报告内容
    appendReportContent(data.content);
  }
};
```

---

## 六、与 ARIS 的对比

| 维度 | ARIS（Ext17） | Deep Research（本章） |
|------|--------------|---------------------|
| **核心定位** | 科研场景的自主研究 Agent | 通用场景的深度研究助手 |
| **研究范围** | 论文检索 + 实验设计 + 报告 | 主题研究 + 多源搜索 + 报告 |
| **Agent 数量** | 1 个主 Agent | 3 个专业 Agent 协作 |
| **工具生态** | Claude Code + 浏览器 | 多种搜索引擎 + 笔记工具 |
| **前端形态** | 独立 Web 应用 | 模态对话框 + 流式输出 |
| **代表场景** | "睡觉时让 AI 做科研" | "5 分钟搞懂一个新领域" |

---

## 七、核心认知

### 7.1 TODO-Driven 的优势

| 优势 | 说明 |
|------|------|
| **可控性强** | 每个子任务有明确的目标和范围 |
| **质量可靠** | 专用 Agent 保证每个阶段的质量 |
| **易于调试** | 可以单独调试每个子任务 |
| **可扩展性好** | 可以轻松添加新的子任务或修改现有子任务 |

### 7.2 多 Agent 协作的设计原则

1. **职责单一**：每个 Agent 只负责一件事
2. **接口清晰**：Agent 之间通过结构化数据通信
3. **可追溯**：记录每个 Agent 的输入输出
4. **容错设计**：单个 Agent 失败不影响整体流程

---

## 八、延伸思考

### 8.1 Deep Research 的局限性

| 局限 | 说明 |
|------|------|
| **搜索质量依赖** | 如果搜索引擎没有覆盖到相关信息，研究结果会缺失 |
| **事实准确性** | LLM 摘要可能出现幻觉，需要来源追溯 |
| **深度 vs 广度** | 当前的 TODO 数量有限（3-5 个），可能无法覆盖极细粒度的研究 |
| **实时性** | 搜索引擎的索引有时延，最新事件可能搜不到 |

### 8.2 可能的改进方向

1. **动态 TODO 扩展**：根据研究进展自动添加新 TODO
2. **多轮研究迭代**：第一轮研究结果触发第二轮深入挖掘
3. **引用溯源**：每个结论标注具体来源，支持点击跳转
4. **交互式研究**：用户可以中途补充问题或指定重点

---

## 九、总结

Deep Research Agent 展示了一个完整的自动化研究系统应该如何设计：

1. **TODO-Driven 研究范式**：将复杂研究分解为可管理的子任务
2. **三 Agent 协作**：规划 → 执行 → 报告，职责清晰
3. **工具生态**：多源搜索 + 笔记管理，支持完整研究流程
4. **前端体验**：全屏沉浸式 + 实时流式输出

如果说 ARIS 是"让 AI 替你做科研"，Deep Research 就是"让 AI 帮你快速搞懂任何事"。两者代表了 Agent 在研究和信息处理领域的两个方向：**深度自主** vs **广度辅助**。
