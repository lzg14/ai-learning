# Day 24：Agent 生产化与可观测性

> Agent 在开发环境跑得好，不等于在生产环境也跑得好。本课学习如何使用 Langfuse 进行监控、追踪、评估，以及 Agent 生产部署的最佳实践。

## Part 1：为什么需要可观测性？

### Agent 的"黑盒"问题

传统的 API 监控很简单——记录请求、响应、状态码、耗时。但 Agent 完全不同：

```plaintext
用户输入："帮我查一下张三的订单"

Agent 内部（黑盒）：
  1. 理解意图 → 查询订单
  2. 调用工具 → 搜索用户 "张三"
     → 找到 3 个匹配
  3. 调用工具 → 查询订单状态
     → 返回 5 个订单
  4. LLM 推理 → 哪个是最新的？
  5. 生成回复 → "张三最新的订单是..."

问题：如果 Agent 回答错了，是哪一步出的问题？
```

### 可观测性的三大支柱

| 支柱 | Agent 中的含义 | 工具 |
|------|---------------|------|
| **Logging（日志）** | 记录 Agent 的每一步思考、每个工具调用 | Langfuse Traces |
| **Metrics（指标）** | Token 消耗、延迟、成功率、成本 | Prometheus + Grafana |
| **Tracing（追踪）** | 完整的调用链：输入 → 思考 → 工具 → 输出 | Langfuse Spans |

### 没有可观测性的后果

```plaintext
✗ 线上 Agent 回答错误 → 不知道哪一步错了
✗ 用户投诉响应慢 → 不知道哪个环节耗时
✗ Token 消耗暴增 → 不知道哪个调用浪费了
✗ 无法评估质量 → 不知道 Agent 表现好坏
✓ 有了可观测性，以上问题全部可答
```

---

## Part 2：Langfuse 入门

### 什么是 Langfuse？

**Langfuse** 是专为 LLM 应用设计的开源可观测性平台，提供：

- **Traces**：完整的请求追踪
- **Spans**：每个步骤的详细信息
- **Evaluations**：自动评估 LLM 输出质量
- **Prompt Management**：Prompt 版本管理
- **Cost Tracking**：Token 消耗和成本追踪

### 安装与配置

```bash
# 自托管（推荐生产使用）
docker compose -f docker-compose.yml up

# 或者使用云服务（https://cloud.langfuse.com）
```

```python
# 安装 SDK
pip install langfuse

# 配置
import os
from langfuse import Langfuse

os.environ["LANGFUSE_PUBLIC_KEY"] = "pk-..."
os.environ["LANGFUSE_SECRET_KEY"] = "sk-..."
os.environ["LANGFUSE_HOST"] = "http://localhost:3000"  # 自托管地址

langfuse = Langfuse()
```

### 核心概念

```python
"""
Langfuse 的数据模型：

Trace（追踪）
  └── Span（跨度）
       ├── Span
       │    ├── Span
       │    └── ...
       ├── Generation（LLM 调用）
       └── Event（事件）

- Trace = 一次完整的 Agent 请求
- Span = 一个步骤（工具调用、推理等）
- Generation = 一次 LLM API 调用
- Event = 一个日志事件
"""
```

---

## Part 3：Agent 追踪实战

### 基本追踪

```python
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context
import time

langfuse = Langfuse()


class TrackedAgent:
    """带追踪的 Agent"""

    @observe()  # 自动追踪这个方法
    async def process(self, user_input: str) -> str:
        """处理用户输入"""

        # 更新追踪名称
        langfuse_context.update_current_trace(
            name="agent-process",
            input=user_input,
            session_id="session-123",
            user_id="user-zhangsan",
            metadata={"source": "web", "version": "1.0.0"}
        )

        # 步骤 1：理解意图
        intent = await self.analyze_intent(user_input)
        print(f"意图: {intent}")

        # 步骤 2：调用工具
        result = await self.call_tool(intent)

        # 步骤 3：生成回复
        reply = await self.generate_reply(user_input, result)

        # 更新追踪输出
        langfuse_context.update_current_trace(
            output=reply
        )

        return reply

    @observe(name="analyze-intent", as_type="span")
    async def analyze_intent(self, text: str) -> str:
        """分析用户意图"""
        # 这里实际会调用 LLM
        time.sleep(0.1)
        return "查询订单"

    @observe(name="call-tool", as_type="span")
    async def call_tool(self, intent: str) -> dict:
        """调用工具"""
        # 记录工具调用细节
        langfuse_context.update_current_span(
            input={"intent": intent},
            metadata={"tool": "order_search", "args": {"user": "张三"}}
        )
        time.sleep(0.2)
        result = {"orders": ["order-001", "order-002"]}
        langfuse_context.update_current_span(output=result)
        return result

    @observe(name="generate-reply", as_type="generation")  # generation 类型记录 LLM 调用
    async def generate_reply(self, user_input: str, tool_result: dict) -> str:
        """使用 LLM 生成回复"""
        # 记录 LLM 调用详情
        langfuse_context.update_current_generation(
            model="glm-4-flash",
            model_parameters={"temperature": 0.7, "max_tokens": 500},
            input=[{"role": "user", "content": f"用户问：{user_input}，工具结果：{tool_result}"}],
            output="张三有 2 个订单，最新的是 order-002",
            usage={"prompt_tokens": 150, "completion_tokens": 30}
        )
        return "张三有 2 个订单，最新的是 order-002"
```

### 手动追踪（更灵活的用法）

```python
class ManualTracking:
    """手动追踪——更精确的控制"""

    def __init__(self, langfuse: Langfuse):
        self.langfuse = langfuse

    async def run_agent(self, user_input: str):
        """手动创建和管理追踪"""

        # 1. 创建 Trace
        trace = self.langfuse.trace(
            name="manual-agent",
            input=user_input,
            session_id="session-456",
            tags=["production", "v2"]
        )

        # 2. 创建 Span（LLM 调用）
        llm_span = trace.span(
            name="llm-call-1",
            input={"model": "glm-4-flash", "messages": [{"role": "user", "content": user_input}]}
        )

        # 模拟 LLM 调用
        time.sleep(0.3)
        llm_output = "意图：查询订单"

        llm_span.end(output=llm_output)

        # 3. 创建 Span（工具调用）
        tool_span = trace.span(
            name="tool-call",
            input={"tool": "search_orders", "args": {"keyword": "张三"}}
        )

        # 模拟工具调用
        time.sleep(0.2)
        tool_result = {"orders": ["order-001", "order-002"]}

        tool_span.end(output=tool_result)

        # 4. 结束 Trace
        trace.update(output=f"处理完成: {len(tool_result['orders'])} 个订单")

        return {
            "trace_id": trace.id,
            "llm_output": llm_output,
            "tool_result": tool_result
        }
```

---

## Part 4：Agent 评估（Evaluation）

### 为什么需要评估？

```plaintext
传统 API：返回 200 OK → 成功；返回 500 → 失败
Agent：   回复了文字 → 成功？不一定！
          "张三没有订单" （实际上有） → 这叫幻觉，但返回了 200 OK
```

Agent 需要**内容级别的评估**，不仅仅是 HTTP 状态码。

### Langfuse 评估系统

```python
from langfuse import Langfuse

langfuse = Langfuse()


class AgentEvaluator:
    """Agent 评估器"""

    def __init__(self, judge_llm):
        self.judge_llm = judge_llm

    def evaluate_correctness(self, trace_id: str, expected: str, actual: str):
        """评估回答正确性"""

        # 创建评估
        langfuse.score(
            trace_id=trace_id,
            name="correctness",
            value=1.0 if expected in actual else 0.0,
            comment=f"期望包含: {expected}"
        )

    def evaluate_llm(self, trace_id: str, question: str, answer: str):
        """用 LLM 评估回答质量"""

        prompt = f"""评估以下 AI 回答的质量（1-5分）：

问题：{question}
回答：{answer}

评分标准：
5 = 完美：准确、完整、有帮助
4 = 良好：基本正确，略有不足
3 = 一般：部分正确，但有明显问题
2 = 差：大部分错误
1 = 完全错误

只返回数字分数："""

        score = self.judge_llm.invoke(prompt)
        try:
            score_value = float(score.content.strip())
        except:
            score_value = 3.0

        langfuse.score(
            trace_id=trace_id,
            name="llm-quality",
            value=score_value,
            comment=f"LLM 评估得分: {score_value}/5"
        )

    def evaluate_detailed(self, trace_id: str, question: str, answer: str, context: str):
        """多维度评估"""

        prompt = f"""对以下 AI 回答进行多维度评估。

问题：{question}
参考上下文：{context}
AI 回答：{answer}

请对以下维度评分（1-5分），返回 JSON：

{{
    "accuracy": <分数>,      # 准确性：与事实是否一致
    "completeness": <分数>,   # 完整性：是否回答了所有问题
    "helpfulness": <分数>,    # 有用性：是否对用户有帮助
    "conciseness": <分数>,    # 简洁性：是否简明扼要
    "safety": <分数>          # 安全性：是否包含有害内容
}}
"""

        result = self.judge_llm.invoke(prompt)
        try:
            scores = json.loads(result.content)
        except:
            scores = {"accuracy": 3, "completeness": 3, "helpfulness": 3,
                      "conciseness": 3, "safety": 5}

        # 记录每个维度的评分
        for dimension, score in scores.items():
            langfuse.score(
                trace_id=trace_id,
                name=dimension,
                value=score
            )

        return scores
```

### 自动评估 Pipeline

```python
import json
import asyncio
from datetime import datetime
from typing import List, Dict


class EvaluationPipeline:
    """自动评估流水线"""

    def __init__(self, langfuse: Langfuse, judge_llm):
        self.langfuse = langfuse
        self.judge_llm = judge_llm
        self.evaluator = AgentEvaluator(judge_llm)
        self.results = []

    def evaluate_batch(self, test_cases: List[Dict], agent_fn) -> dict:
        """批量评估 Agent"""
        summary = {
            "total": len(test_cases),
            "passed": 0,
            "failed": 0,
            "avg_score": 0.0,
            "details": []
        }

        for case in test_cases:
            # 运行 Agent
            trace_id = f"eval-{datetime.now().timestamp()}"
            actual_answer = agent_fn(case["question"])

            # 评估
            if "expected" in case:
                self.evaluator.evaluate_correctness(
                    trace_id, case["expected"], actual_answer
                )

            scores = self.evaluator.evaluate_detailed(
                trace_id, case["question"], actual_answer,
                case.get("context", "")
            )

            # 统计
            avg = sum(scores.values()) / len(scores)
            passed = avg >= 4.0

            summary["avg_score"] += avg
            if passed:
                summary["passed"] += 1
            else:
                summary["failed"] += 1

            summary["details"].append({
                "question": case["question"],
                "scores": scores,
                "passed": passed
            })

        summary["avg_score"] /= len(test_cases)
        return summary

    def generate_report(self, summary: dict) -> str:
        """生成评估报告"""
        report = f"""
## Agent 评估报告
生成时间：{datetime.now().isoformat()}

### 总体统计
- 总用例数：{summary['total']}
- 通过：{summary['passed']}
- 失败：{summary['failed']}
- 通过率：{summary['passed']/summary['total']:.1%}
- 平均分：{summary['avg_score']:.2f}/5.0

### 失败用例详情
"""
        for detail in summary["details"]:
            if not detail["passed"]:
                report += f"\n- 问题：{detail['question']}"
                report += f"\n  评分：{detail['scores']}"

        report += "\n\n### 改进建议\n"
        if summary["avg_score"] < 4.0:
            report += "- ⚠️ 平均分低于 4.0，需要优化 Agent 性能\n"
        if summary["failed"] > summary["total"] * 0.2:
            report += "- ⚠️ 失败率超过 20%，建议检查 Agent 核心逻辑\n"

        return report
```

---

## Part 5：生产监控与告警

### 关键监控指标

| 指标 | 说明 | 告警阈值 | 影响 |
|------|------|---------|------|
| **P50/P95 延迟** | 响应时间分布 | P95 > 10s | 用户体验 |
| **Token 消耗** | 每次请求消耗的 Token | 异常增长 > 50% | 成本 |
| **工具调用成功率** | 工具调用成功的比例 | < 95% | 功能可用性 |
| **评估分数** | 回答质量 LLM 评分 | < 3.5/5 | 回答质量 |
| **错误率** | 异常和报错比例 | > 5% | 系统稳定性 |

### 监控实现

```python
import time
import statistics
from collections import deque
from datetime import datetime, timedelta
from typing import Optional


class AgentMonitor:
    """Agent 生产监控器"""

    def __init__(self, window_minutes: int = 10):
        self.window = timedelta(minutes=window_minutes)
        self.requests = deque()
        self.errors = deque()
        self.token_usage = deque()

    def record_request(self, duration: float, tokens: int, success: bool):
        """记录一次请求"""
        now = datetime.now()
        self.requests.append({
            "timestamp": now,
            "duration": duration,
            "success": success
        })
        self.token_usage.append({
            "timestamp": now,
            "tokens": tokens
        })

        # 清理过期数据
        self._cleanup()

    def record_error(self, error_type: str, detail: str):
        """记录错误"""
        self.errors.append({
            "timestamp": datetime.now(),
            "type": error_type,
            "detail": detail
        })
        self._cleanup()

    def get_latency(self, percentile: int = 95) -> Optional[float]:
        """获取延迟百分位"""
        if not self.requests:
            return None
        durations = [r["duration"] for r in self.requests]
        return statistics.quantiles(durations, n=100)[percentile - 1]

    def get_error_rate(self) -> float:
        """获取错误率"""
        if not self.requests:
            return 0.0
        errors = sum(1 for r in self.requests if not r["success"])
        return errors / len(self.requests)

    def get_tokens_per_request(self) -> float:
        """获取平均 Token 消耗"""
        if not self.token_usage:
            return 0.0
        return sum(t["tokens"] for t in self.token_usage) / len(self.token_usage)

    def _cleanup(self):
        """清理过期数据"""
        cutoff = datetime.now() - self.window
        self.requests = deque(
            r for r in self.requests if r["timestamp"] > cutoff
        )
        self.errors = deque(
            e for e in self.errors if e["timestamp"] > cutoff
        )
        self.token_usage = deque(
            t for t in self.token_usage if t["timestamp"] > cutoff
        )

    def health_check(self) -> dict:
        """健康检查"""
        error_rate = self.get_error_rate()
        p95_latency = self.get_latency(95)

        status = "healthy"
        issues = []

        if error_rate > 0.05:
            status = "degraded"
            issues.append(f"错误率 {error_rate:.1%} > 5%")
        if p95_latency and p95_latency > 10:
            status = "degraded"
            issues.append(f"P95 延迟 {p95_latency:.1f}s > 10s")

        return {
            "status": status,
            "error_rate": error_rate,
            "p95_latency": p95_latency,
            "avg_tokens": self.get_tokens_per_request(),
            "issues": issues,
            "request_count": len(self.requests)
        }
```

### 集成到 FastAPI

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import time

app = FastAPI()
monitor = AgentMonitor()

@app.middleware("http")
async def monitor_middleware(request: Request, call_next):
    """监控中间件"""
    start = time.time()
    try:
        response = await call_next(request)
        duration = time.time() - start
        monitor.record_request(
            duration=duration,
            tokens=response.headers.get("X-Token-Count", 0),
            success=response.status_code < 500
        )
        return response
    except Exception as e:
        duration = time.time() - start
        monitor.record_request(duration=duration, tokens=0, success=False)
        monitor.record_error("internal_error", str(e))
        return JSONResponse(
            status_code=500,
            content={"error": "Internal Server Error"}
        )

@app.get("/health")
async def health():
    """健康检查端点"""
    return monitor.health_check()

@app.get("/metrics")
async def metrics():
    """Prometheus 指标端点"""
    return {
        "latency_p50": monitor.get_latency(50),
        "latency_p95": monitor.get_latency(95),
        "latency_p99": monitor.get_latency(99),
        "error_rate": monitor.get_error_rate(),
        "avg_tokens": monitor.get_tokens_per_request(),
        "request_count": len(monitor.requests)
    }
```

---

## Part 6：A/B 测试

### 为什么需要 A/B 测试？

```plaintext
场景：你改了 System Prompt，想让 Agent 回复更简洁

❌ 感觉"新版本回复更好了" → 这是幻觉，不是数据
✅ A/B 测试：
   A 组（旧 Prompt）：50% 流量
   B 组（新 Prompt）：50% 流量
   对比：平均分、耗时、用户反馈
```

### A/B 测试实现

```python
import random
from enum import Enum


class Variant(str, Enum):
    A = "control"    # 对照组
    B = "treatment"  # 实验组


class ABTest:
    """A/B 测试管理器"""

    def __init__(self, experiment_name: str, b_traffic: float = 0.5):
        self.name = experiment_name
        self.b_traffic = b_traffic
        self.results = {Variant.A: [], Variant.B: []}

    def assign(self, user_id: str) -> Variant:
        """分配用户到实验组"""
        if random.random() < self.b_traffic:
            return Variant.B
        return Variant.A

    def record(self, variant: Variant, metrics: dict):
        """记录实验结果"""
        self.results[variant].append(metrics)

    def analyze(self) -> dict:
        """分析实验结果"""
        analysis = {}

        for variant in Variant:
            data = self.results[variant]
            if not data:
                analysis[variant.value] = {"count": 0, "error": "无数据"}
                continue

            # 计算各项指标的平均值
            agg = {"count": len(data)}
            for key in data[0].keys():
                values = [d[key] for d in data if isinstance(d.get(key), (int, float))]
                if values:
                    agg[f"avg_{key}"] = sum(values) / len(values)

            analysis[variant.value] = agg

        # 对比分析
        if analysis[Variant.A.value].get("count", 0) > 0 and \
           analysis[Variant.B.value].get("count", 0) > 0:
            a_score = analysis[Variant.A.value].get("avg_score", 0)
            b_score = analysis[Variant.B.value].get("avg_score", 0)
            improvement = ((b_score - a_score) / a_score * 100) if a_score > 0 else 0

            analysis["conclusion"] = {
                "winner": Variant.B.value if b_score > a_score else Variant.A.value,
                "improvement": f"{improvement:+.1f}%",
                "confidence": "需要更多数据" if len(data) < 100 else "统计显著"
            }

        return analysis


# 使用示例
ab_test = ABTest("prompt-v2", b_traffic=0.5)


def get_agent_prompt(variant: Variant) -> str:
    """根据实验组返回不同的 Prompt"""
    if variant == Variant.A:
        return "你是一个有帮助的AI助手。请详细回答用户的问题。"
    else:
        return "你是一个简洁的AI助手。请用3句话以内回答用户的问题。"


def handle_request(user_id: str, question: str) -> str:
    """处理请求（带A/B测试）"""
    # 分配实验组
    variant = ab_test.assign(user_id)

    # 使用对应的 Prompt
    prompt = get_agent_prompt(variant)
    # ... 调用 LLM

    # 记录结果
    ab_test.record(variant, {
        "score": 4.5,
        "latency": 1.2,
        "tokens": 150
    })

    return f"[{variant.value}] Agent回答"
```

---

## Part 7：生产部署 Checklist

### 架构总览

```plaintext
用户请求
    │
    ▼
┌──────────────────────────────────┐
│            API 网关              │
│    (限流、鉴权、路由、A/B测试)    │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│         Agent 编排层             │
│    (意图识别、任务规划、工具调用)  │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│         LLM 推理服务             │
│    (vLLM/TGI, 负载均衡, 量化)    │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│        可观测性平台               │
│  (Langfuse + Prometheus + 告警)   │
└──────────────────────────────────┘
```

### 部署前 Checklist

```plaintext
□ 性能
   □ 延迟优化（P95 < 5s）
   □ 并发处理（压测确认 QPS）
   □ Token 预算控制（防止无限消耗）

□ 可靠性
   □ 优雅降级（LLM 不可用时返回缓存）
   □ 重试机制（工具调用失败重试）
   □ 超时控制（所有外部调用设超时）

□ 监控
   □ Langfuse 追踪接入
   □ 关键指标（延迟、错误率、Token）
   □ 告警规则（P95 > 10s 告警）

□ 安全
   □ 输入过滤（防止 Prompt 注入）
   □ 输出审核（敏感信息过滤）
   □ 权限控制（工具调用权限）

□ 运维
   □ 健康检查端点（/health）
   □ 优雅关闭（SIGTERM 处理）
   □ 日志收集（结构化日志）
   □ 回滚方案（保留旧版本模型）
```

### 生产级 Agent 启动模板

```python
import asyncio
import signal
import sys
from typing import Optional

from fastapi import FastAPI, Request
from contextlib import asynccontextmanager


class ProductionAgent:
    """生产级 Agent 模板"""

    def __init__(self):
        self.monitor = AgentMonitor()
        self.langfuse = Langfuse()
        self.running = True

    async def startup(self):
        """启动时加载"""
        print("Loading model...")
        # 加载模型、工具等
        print("Agent ready")

    async def shutdown(self, sig: signal.Signals):
        """优雅关闭"""
        print(f"Received {sig.name}, shutting down...")
        self.running = False
        # 保存未完成的追踪
        # 关闭连接池
        print("Shutdown complete")

    async def process(self, user_input: str, trace_id: str) -> str:
        """核心处理逻辑"""
        with self.langfuse.trace(name="production-agent", id=trace_id) as trace:
            # 处理逻辑
            result = f"Processed: {user_input}"
            trace.update(output=result)
            return result


@asynccontextmanager
async def lifespan(app: FastAPI):
    """应用生命周期"""
    agent = ProductionAgent()
    await agent.startup()
    yield
    await agent.shutdown(signal.SIGTERM)


app = FastAPI(lifespan=lifespan)


@app.post("/agent/chat")
async def chat(request: Request):
    """Agent 聊天接口"""
    body = await request.json()
    # 生产级处理
    return {"response": "ok"}
```

---

## 今日要点

- 理解了 Agent 可观测性的重要性（三大支柱：日志、指标、追踪）
- 掌握了 Langfuse 的核心概念（Trace、Span、Generation、Score）
- 学会了使用 Langfuse 进行 Agent 追踪和评估
- 实现了生产监控系统（延迟、错误率、Token 消耗）
- 掌握了 A/B 测试方法
- 理解了生产部署的最佳实践和 Checklist

---

## 自测题

1. Agent 可观测性的三大支柱是什么？
2. Langfuse 的 Trace、Span、Generation 分别代表什么？
3. Agent 评估和传统 API 评估的核心区别是什么？
4. P50、P95、P99 延迟分别代表什么？
5. A/B 测试在 Agent 生产化中的作用是什么？

---

## 动手作业

1. 搭建 Langfuse 自托管环境（Docker Compose）
2. 为一个 Agent 接入 Langfuse 追踪（Trace + Span + Generation）
3. 实现一个包含延迟、错误率、Token 消耗的监控面板
4. （进阶）设计一个完整的 A/B 测试实验，对比两个 System Prompt 的效果