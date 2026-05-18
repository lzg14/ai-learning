# Ext23：Agent Skills — 微软 AI 工程技能包

> addyosmani/agent-skills 是微软工程师维护的 AI 工程技能库，涵盖 22 个专项技能，覆盖从任务规划、代码编写到安全审计的完整开发周期。本课深入解析每个技能的定位、核心用法，以及对我们 PM 工作流的借鉴意义。

**官方仓库**：https://github.com/addyosmani/agent-skills
**Star**：7.7k（截至 2026-05）
**维护方**：Microsoft（addyosmani）

---

## Part 1：技能包全景

### 6 阶段开发周期

agent-skills 按软件工程生命周期分为 6 个阶段：

| 阶段 | 技能数 | 定位 |
|------|--------|------|
| **DEFINE** | 2 | 任务定义与验收标准 |
| **PLAN** | 2 | 任务拆解与规划 |
| **BUILD** | 6 | 代码编写与实现 |
| **VERIFY** | 5 | 质量验证与测试 |
| **REVIEW** | 3 | 代码审查与反馈 |
| **SHIP** | 4 | 部署与发布 |

### 22 个专项技能一览

```
DEFINE (2)
├── requirements-analysis-and-specification     # 需求分析与规格编写
└── planning-and-task-breakdown                # 任务拆解与验收标准

PLAN (2)
├── design-phase-and-architecture-planning     # 系统设计阶段
└── incremental-implementation                 # 增量实现策略

BUILD (6)
├── autonomous-coding-and-implementation        # 自主编码实现
├── git-workflow-and-versioning                # Git 工作流与版本控制
├── test-driven-development                     # 测试驱动开发
├── debugging-and-error-recovery               # 调试与错误恢复
├── incremental-implementation                  # 增量实现策略（重复）
└── (另有 2 个未列出名称)

VERIFY (5)
├── code-review-and-quality                     # 代码审查与质量保障
├── automated-testing-and-ci-cd                # 自动化测试与 CI/CD
├── performance-profiling-and-optimization     # 性能分析与优化
├── security-analysis-and-vulnerability-detection  # 安全分析与漏洞检测
└── (另有 1 个未列出名称)

REVIEW (3)
├── code-review-and-quality                     # 代码审查与质量保障
├── documentation-and-knowledge-management     # 文档与知识管理
└── (另有 1 个未列出名称)

SHIP (4)
├── shipping-and-launch                        # 发布与上线
├── monitoring-and-observability              # 监控与可观测性
├── maintenance-and-long-term-support          # 维护与长期支持
└── (另有 1 个未列出名称)
```

---

## Part 2：核心技能详解

### 2.1 DEFINE：需求分析与规格编写

**对应技能**：`requirements-analysis-and-specification`

这是整个周期的起点。好的规格文档需要回答：
- 做什么？（功能范围）
- 不做什么？（明确边界）
- 如何验收？（acceptance criteria 可测试）
- 优先级？（P0/P1/P2）

**核心原则**：
- 规格优先于代码（Spec first）
- acceptance criteria 必须可验证，不能写"系统应该运行良好"
- 重大决策要记录"为什么这样做"，不只是"做了什么"

### 2.2 PLAN：任务拆解与验收标准

**对应技能**：`planning-and-task-breakdown`

任务拆解的核心是**可独立交付的子任务**：

```
大任务 → 子任务 → 子子任务
         ↓
    每个子任务满足：
    1. 有明确的输入/输出
    2. 可单独测试
    3. 完成后有可见的进展
```

**关键概念：Definition of Done（DoD）**
一个任务只有满足以下条件才算完成：
- 代码写完了
- 单元测试通过了
- 代码 review 通过了
- 文档更新了
- 在 staging 环境验证了

### 2.3 BUILD：Git 工作流与版本控制

**对应技能**：`git-workflow-and-versioning`

agent-skills 推荐的 Git 实践：

```
Commit 规范（Atomic Commit）：
- 每个 commit 做一个逻辑修改
- 控制在 ~100 行以内
- commit message 格式：[类型] 简短描述

类型：feat | fix | docs | test | refactor | chore

示例：
feat: add multi-agent协作 task breakdown
fix: normalize 2026-05-01_article.md metadata
docs: update SPEC.md glossary link rules
```

**分支策略**：
- `main`：稳定可部署
- `feature/`：功能分支
- `fix/`：修复分支
- `review/`：代码 review 分支

### 2.4 BUILD：测试驱动开发（TDD）

**对应技能**：`test-driven-development`

TDD 三步循环：

```
Red（红）：写一个失败的测试
    ↓
Green（绿）：写最少的代码让它通过
    ↓
Refactor（重构）：优化代码，测试保证不破坏
    ↓
循环
```

**TDD 的核心价值**：
- 测试即文档（测试读得懂，需求就理解了）
- 回归保护（改 bug 不引入新 bug）
- 驱动设计（先想怎么测，再想怎么写）

### 2.5 VERIFY：自动化测试与 CI/CD

**对应技能**：`automated-testing-and-ci-cd`

分层测试金字塔：

```
        /\
       /  \      E2E Tests（少量，端到端）
      /----\     Integration Tests（适量，服务间）
     /      \    Unit Tests（大量，函数级别）
    /--------\   Static Analysis（lint, type check）
```

**CI/CD 流水线**：
```
Code Push → Lint → Unit Test → Integration Test → Build → Deploy
           ↓        ↓            ↓              ↓        ↓
         失败停     失败停        失败停          失败停    失败停
```

### 2.6 VERIFY：代码审查与质量保障

**对应技能**：`code-review-and-quality`

代码 review 的本质是**知识传递 + 质量把控**，不是挑刺：

**reviewer 职责**：
- 逻辑是否正确
- 是否有边界情况没处理
- 测试覆盖是否充分
- 代码可读性如何

**被 review 者职责**：
- commit 规模控制在 ~100 行
- 写好 commit message
- 准备好 review 要点（哪些地方需要重点看）

### 2.7 SHIP：监控与可观测性

**对应技能**：`monitoring-and-observability`

可观测性三支柱：

| 维度 | 说明 | 例子 |
|------|------|------|
| **Logs** | 事件记录 | 错误日志、访问日志 |
| **Metrics** | 量化指标 | QPS、延迟、错误率 |
| **Traces** | 调用链路 | 请求在各个服务间的流转 |

**关键原则**：
- 告警要 actionable（可操作），不能写"系统可能有问题"
- SLO/SLI 要明确：延迟 < 200ms，错误率 < 0.1%

---

## Part 3：对我们 PM 工作流的借鉴

### 3.1 当前 PM 流程 vs agent-skills 标准

| 维度 | 我们当前 | agent-skills 标准 | 差距 |
|------|----------|-------------------|------|
| **任务定义** | pm-task.md 列表 | 规格文档 + DoD | 缺明确验收条件 |
| **Commit 规范** | 有但执行不严 | ~100行/原子commit | 已部分对齐 |
| **自检机制** | 无 | 验收清单 | **已补齐（上一轮更新）** |
| **Alert 条件** | 模糊 | stop-the-line rule | **已补齐（上一轮更新）** |
| **进度分期** | 无 | 每阶段 check-in | **已补齐（上一轮更新）** |

### 3.2 已吸收并落地的改进

经过本轮更新，PM SOUL 已融入 agent-skills 最佳实践：

**✅ 已落地**：
1. **验收条件清单**（来自 planning-and-task-breakdown 的 DoD 概念）
   - frontmatter、元信息栏、section 标题、术语链接、相关文章、自检7条

2. **Alert 触发条件**（来自 debugging-and-error-recovery 的 stop-the-line rule）
   - 4 类情况明确：内容异常、格式无法判断、修复失败、年份错误

3. **进度分期汇报**（来自 shipping-and-launch 的分阶段 check-in）
   - 每10篇一记，不主动打扰大管家

4. **Commit 规范**（来自 git-workflow-and-versioning）
   - `fix: normalize YYYY-MM-DD_slug.md` 格式

### 3.3 下一步可继续深挖的方向

| 方向 | 对应技能 | 当前状态 | 建议 |
|------|----------|----------|------|
| 测试覆盖 | `test-driven-development` | 无自动化测试 | ai-info 目前是人工 QA，可探索回归测试脚本 |
| 可观测性 | `monitoring-and-observability` | 无 | 可给 cronjob 加 watch_patterns 监控异常 |
| 知识管理 | `documentation-and-knowledge-management` | 无 | 建立 FAQ / 常见坑文档积累 |

---

## Part 4：22 个技能速查表

| # | 技能名 | 阶段 | 核心一句话 |
|----|--------|------|-----------|
| 1 | requirements-analysis-and-specification | DEFINE | 规格优先于代码，acceptance criteria 必须可测试 |
| 2 | planning-and-task-breakdown | DEFINE | 每个子任务有输入/输出/可独立交付 |
| 3 | design-phase-and-architecture-planning | PLAN | 先想清楚系统架构再动手 |
| 4 | incremental-implementation | PLAN | 小步增量，每步可交付 |
| 5 | autonomous-coding-and-implementation | BUILD | 自主编码，遵守规范 |
| 6 | git-workflow-and-versioning | BUILD | ~100行原子 commit，类型前缀 |
| 7 | test-driven-development | BUILD | Red-Green-Refactor 循环 |
| 8 | debugging-and-error-recovery | BUILD | stop-the-line rule：啥情况该停 |
| 9 | code-review-and-quality | VERIFY | reviewer 知识传递 + 质量把控 |
| 10 | automated-testing-and-ci-cd | VERIFY | 测试金字塔 + CI 失败即停 |
| 11 | performance-profiling-and-optimization | VERIFY | 先 profile 再优化，别猜 |
| 12 | security-analysis-and-vulnerability-detection | VERIFY | 依赖审计 + 注入检测 |
| 13 | documentation-and-knowledge-management | REVIEW | 文档即知识传递 |
| 14 | shipping-and-launch | SHIP | 分阶段 check-in + rollback 预案 |
| 15 | monitoring-and-observability | SHIP | Logs/Metrics/Traces 三支柱 |
| 16 | maintenance-and-long-term-support | SHIP | 技术债管理和版本规划 |
| 17 | （未名） | BUILD | — |
| 18 | （未名） | VERIFY | — |
| 19 | （未名） | REVIEW | — |
| 20 | （未名） | SHIP | — |
| 21 | （未名） | VERIFY | — |
| 22 | （未名） | BUILD | — |

---

## Part 5：参考资料

- **官方仓库**：https://github.com/addyosmani/agent-skills
- **相关课程**：
  - Day17：任务规划与分解
  - Day20：多 Agent 协作系统设计
  - Day24：Agent 生产化与可观测性
  - Day27：LLM 评测与质量保证
