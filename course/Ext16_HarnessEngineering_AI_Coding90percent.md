> ⚠️ **数据更新时间：2026年5月**
>
> 📢 **来源**：阿里云开发者 | 新安

# Ext16：Harness Engineering——AI Coding 率从 25% 提升至 90%

---

## 一、为什么要聊 Harness Engineering

2025 年是 AI Coding Agent 全面爆发的一年。Cursor、Claude Code、Copilot Workspace 让开发者感受到 Agentic Coding 的力量——但 Anthropic 数据显示，日常工作中约 60% 的时间在使用 AI 辅助，能够"完全委托"给 Agent 的任务比例却仅为 **0-20%**。

这个巨大落差揭示了核心矛盾：模型能力已经足够强，但从"能力"到"可信赖的工程产出"之间，还横亘着一道系统性鸿沟。Harness Engineering 就是用来弥合这道鸿沟的。

> **效果数据**：AI 代码率从 **24.86% 提升至 90.54%**，个人维度从 14.24% 提升至 87.85%。

---

## 二、Harness Engineering 是什么

### 2.1 三次范式跃迁

| 阶段 | 关注点 | 隐喻 |
|------|--------|------|
| **Prompt Engineering（2022-2024）** | 单次交互优化 | 写好一封邮件 |
| **Context Engineering（2025）** | 给 Agent 看什么 | 给邮件附上正确附件 |
| **Harness Engineering（2026）** | 跨会话的完整系统架构 | 设计 Agent 的工作环境 |

Mitchell Hashimoto 的操作性定义：
> "Every time you discover an agent has made a mistake, you take the time to engineer a solution so that it can never make that mistake again."

### 2.2 Anthropic 四种失败模式

| 模式 | 描述 | 解法 |
|------|------|------|
| **One-shot Syndrome** | 复杂需求在单上下文内完成，上下文超过 40% 后质量快速衰退 | 分阶段执行，结构化拆分 |
| **Premature Victory Declaration** | Agent 宣布完成但编译都不通过 | 质量门禁强制验证 |
| **Premature Feature Completion** | 功能未端到端验证就标记完成 | Browser Automation 自动化截图验证 |
| **Cold Start Problem** | 每次新会话大量 Token 用于重建上下文 | 进度持久化到文件系统 |

核心结论：Anthropic 指出——**"Agents are incapable of accurately evaluating their own work."** Agent 缺乏外部的结构化约束和反馈机制。

### 2.3 四根支柱

1. **上下文架构（Context Architecture）**：上下文分层加载、按需获取（L1 常驻/L2 触发/L3 按需），控制在 ~100 行作为索引和地图
2. **Agent 专业化（Agent Specialization）**：Planner/Generator/Evaluator 三角色分离，Anthropic："将做事的 Agent 和评判的 Agent 分开是一个强有力的杠杆"
3. **持久化记忆（Persistent Memory）**：进度持久化在文件系统而非上下文窗口，跨会话任务成为可能
4. **结构化执行（Structured Execution）**：永远不让 Agent 在未经审查和批准书面计划之前写代码

---

## 三、AI Coding 的现状与挑战

| 挑战 | 表现 | 根因 |
|------|------|------|
| **认知负担** | 企业级 Java 十万行代码，Agent 不知道架构约束 | 隐性知识未系统化记录 |
| **质量缺口** | 语法正确但业务语义存在微妙错误 | 不能依赖 Agent 自我审查 |
| **熵累积** | Agent 模仿代码库中 Suboptimal Pattern，代码逐渐腐化 | 缺乏 Golden Principles 强制约束 |
| **角色转移** | 从"写代码"变成"设计 Agent 工作环境 + 编写规范 + 管理验收" | 文档从给人看变成 Agent 认识世界的唯一窗口 |

---

## 四、Harness Engineering 实战

### 4.1 四要素架构（.harness/ 目录）

| 要素 | 作用 | 核心内容 |
|------|------|----------|
| **Rules（规则体系）** | 告诉 Agent 标准是什么 | 工程结构约束、编码规范、分层架构约定 |
| **Skills（技能体系）** | 告诉 Agent 怎么做 | 需求分析 SOP、编码分层规范、评审检查清单 |
| **Wiki（知识库）** | 告诉 Agent 系统是什么样的 | 链路梳理、数据模型、核心业务流程 |
| **Changes（变更管理）** | 记录 Agent 做了什么 | 每个需求的全过程文档，完整 Audit Trail |

### 4.2 Application Owner Agent

角色定义文件约 **400 行**，存于 `.harness/agents/`。五个核心模块：

1. **Role & Context**：身份定位 + 核心背景（模块结构、技术栈、核心约束），控制在 20-30 行
2. **Configuration Hub Index**：Rules/Skills/Wiki/MCP 四大组件的路径、职责、触发场景
3. **七项核心职责**：需求理解、任务拆解、任务分发协调、验收把关、知识库维护
4. **Workflow Orchestration**：10 阶段流程的完整调度逻辑——触发条件、Skill 加载、产出路径、质量门禁
5. **沟通原则与硬性约束**：必须做到和禁止做的两张清单

### 4.3 十阶段 Pipeline

```
需求输入 → 需求澄清 → 需求评审 → 任务拆解 → 编码实现
→ 编码评审 → 单元测试 → CI 验证 → 部署验证 → 交付确认
```

每阶段三要素：**触发条件 → Skill 注入 → 质量门禁**。

**失败回退路径**：CI 失败 + 测试用例数为 0 → 回退阶段 5；编译错误 → 回退阶段 3。

**5 个 Human-in-the-Loop 确认点**：需求待决、计划评审后、编码评审后、部署环境参数、最终交付。

### 4.4 Skill 体系

**coding-skill** 内含 8 份分层规范（表现层 / 应用层 / 业务层 / 数据层 / 适配层 / 文档层），硬性约束：价格字段用 `long`（单位为分）禁止 `double/float`，外部服务调用必须设超时和降级。

**expert-reviewer** 定义两种评审循环：计划评审（审查 spec.md + tasks.md）和执行评审（审查编码实现），每条意见含：问题描述 + 修改建议 + 优先级（MUST FIX / LOW / INFO）。

### 4.5 变更管理

每个需求在 `.harness/changes/` 下创建独立目录：

```
{变更类型}-{需求名称}-{YYYYMMDD}/
├── summary.md              # 全流程追溯摘要
├── request_analysis/       # 需求分析 + 评审记录（版本递增）
├── coding/                 # 编码报告 + 代码评审报告（版本递增）
├── unit_test/              # 单元测试报告及评审
├── ci_result/              # CI 验证结果
└── deployment/             # 部署验证报告
```

评审文件版本递增（旧版本永不删除），构成完整 Audit Trail。

---

## 五、关键经验

### 5.1 先 Dry Run 再上真实需求

用虚拟需求完整走一遍全流程，空跑发现四个缺陷：CI 门禁忽略测试用例数为 0、简单需求不生成评审报告、摘要文件重复行、部署参数被错误推测。

> **核心启示**：不要期望第一版 Harness 就是完美的，用低成本快速验证、快速修复。

### 5.2 质量门禁必须可程序化验证

> "If it can't be mechanically enforced, the agent will drift."——OpenAI 百万行代码项目核心经验

将"检查 CI 是否通过"改为三个可程序化条件：`status == SUCCESS && total_tests > 0 && passed == total`。**一切不可被机器验证的约束，在 Agent 执行中都是无效约束。**

### 5.3 分离执行与评判是关键杠杆

编码 Agent 和评审 Agent 分离带来显著质量收益：评审 Agent 发现了编码 Agent 遗漏的渠道判断逻辑（潜在线上故障），并检测到 Agent 试图跳过评审阶段强制回退。本质上是将 Code Review 自动化，将质量发现前移到 Human Review 之前。

### 5.4 流程一致性优先于流程效率

仅 2 个文件、6 行代码的小需求，依然走完完整 10 阶段——1 轮评审即通过。好的流程不给简单任务增加显著负担，但能保证不因"这次改动很小"就跳过关键环节。"小改动大事故"在企业级系统中屡见不鲜。

### 5.5 规范是活文档，持续迭代

> 每发现一个错误，就工程化地消除它再次发生的可能性。规范的每一行都对应一个历史失败案例。

当你觉得某条规则"多余"时，往往是因为它背后有一个真实踩过的坑。

---

## 六、效果数据

### 6.1 质量维度对比

| 维度 | 无 Harness | 有 Harness |
|------|-------------|------------|
| 需求理解偏差 | 编码方向跑偏 | 评审阶段前被拦截 |
| 编码质量 | 业务逻辑有隐患 | 评审环节拦截潜在线上问题 |
| 测试覆盖 | 形式化测试 | 18 个有业务价值的测试用例，CI 全通过 |
| 过程可追溯性 | 全靠记忆 | 完整变更文档链，随时可回溯 |
| 流程一致性 | 因人而异，因需求而异 | 10 阶段流程一致执行 |

### 6.2 AI 代码率跃迁

| 时间 | 维度 | AI 采纳行数 | 提交代码行数 | AI 行占比 |
|------|------|-------------|--------------|----------|
| 3 月基线 | 项目维度 | 1,411 | 5,676 | **24.86%** |
| 3 月基线 | 个人维度 | 666 | 4,677 | 14.24% |
| 4 月成熟后 | 项目维度 | 3,063 | 3,383 | **90.54%** |
| 4 月成熟后 | 个人维度 | 3,051 | 3,473 | 87.85% |

> 高 AI 代码率本身不是目标——**在质量可控前提下的高 AI 代码率才有意义**。90% 的 AI 代码经过了完整需求分析、编码评审、单元测试和 CI 验证流程。

---

## 七、总结

### 核心本质

Harness 的价值不在于让 Agent 变得更聪明，而在于让 Agent 的错误**可控、可发现、可修复**。这和软件质量保障思路一脉相承——不指望程序员写出零缺陷代码，而是通过 Code Review、Unit Testing、CI/CD、灰度发布层层拦截。Harness 只不过拦截对象从"程序员"变成了"AI Agent"。

### 投入产出比

前期投入约一周（Rules 定义、Skill 编写、评审规范、模板设计），这是一次性投入。一旦建立，每个后续需求都在框架内高效运转。这些文档资产具有**复利效应**——不仅服务于 AI Agent，也为团队知识管理提供结构化基础设施。

### 未来方向

- **Harness 自我进化**：Agent 自动分析历史失败案例，提出规范改进建议
- **跨项目模板化**：将 10-Stage Pipeline、Review Loop 抽象为可参数化模板
- **更精细的 Agent 角色矩阵**：Performance Auditor、Security Scanner、Documentation Sync Agent

### 结语

> 未来的工程竞争力不再取决于谁的 Prompt 写得更好，而是取决于谁的 Harness 设计得更精密、更可靠、更具可演化性。作为开发者，核心竞争力正从"写代码"转向"设计 Agent 的工作环境"。

---

## 参考资料

- [Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Anthropic: Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- [OpenAI: Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)
