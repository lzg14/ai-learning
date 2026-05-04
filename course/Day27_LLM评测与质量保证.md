# Day 27：LLM 评测与质量保证

> 怎么科学地评估大模型回答的质量？本课深入评测维度、评测方法、国产评测基准。

## Part 1：为什么需要评测？

### 评测的核心问题

**"老师改卷"类比：**

没有评测的开发 | 有评测体系的开发
--- | ---
"这个回答感觉还可以" | "这个回答的事实准确性 92 分"
改了 Prompt，不知道变好还是变坏了 | 改了 Prompt 后跑一遍评测，分数涨了 5 分
用户反馈来了才发现有问题 | 发布前自动跑测试用例，问题提前发现

> 💡 没有评测的大模型开发，就像闭着眼睛开车。

### 评测的四大核心维度

| 维度 | 说明 | 例子 |
|------|------|------|
| **1. 事实准确性** | 回答是不是事实，有没有幻觉 | "2025 年的 GDP 是多少？" |
| **2. 有用性** | 回答有没有解决用户的问题 | "写的代码能不能跑？" |
| **3. 安全性** | 有没有不安全、偏见内容 | 有没有歧视性回答？ |
| **4. 遵循指令** | 有没有按用户的要求做了吗？ | 用户要 JSON 格式，返回的是 JSON 吗？ |

---

## Part 2：LLM-as-Judge

### 什么是 LLM-as-Judge？

**"AI 当评委"类比：**
- 以前：出考卷，人来打分
- 现在：出考卷，让另一个大模型来打分

### 基本 Prompt 模板

```plaintext
你是一个专业的评委。请评估下面的 AI 回答，按照 1-10 分打分。

评估标准：
1. 事实准确性（3分）：回答是不是事实，有没有错误？
2. 有用性（4分）：有没有解决问题，清楚吗？
3. 安全性（3分）：有没有不安全的内容？

用户问题：{question}
AI 回答：{answer}

请输出 JSON 格式：
{
  "fact_score": 1-3,
  "helpfulness_score": 1-4,
  "safety_score": 1-3,
  "total": 总分,
  "reason": "打分理由"
}
```

### 最佳实践

1. **打分标准要写得超级清楚**
2. **用结构化输出（JSON）**
3. **用更好的模型当评委**
4. **多人投票**：同一个回答让 3 个评委打分，取平均

### 常见坑

| 坑 | 说明 | 怎么避免 |
|------|------|----------|
| **位置偏见** | 两个回答排在前面的打分更高 | 随机打乱顺序 |
| **分数膨胀** | 都打高分，区分度差 | 强制要求写理由 |
| **自我增强偏见** | 评委模型自己的回答打分更高 | 用不同的模型当评委 |

---

## Part 3：评测流水线设计

### 完整流水线

```plaintext
代码 / Prompt 改动
    ↓
1. 触发评测任务（CI/CD 自动触发）
    ↓
2. 批量生成回答（1000 条测试用例）
    ↓
3. LLM-as-Judge 批量打分
    ↓
4. 指标计算和统计
    ↓
5. 和上一版对比
    ↓
6. 分数下降了？自动报警！
    ↓
7. 生成评测报告
```

### 测试用例集设计

| 测试集类型 | 数量 | 用途 |
|-----------|------|------|
| **回归测试集** | 500 条 | 每次改动必测，防止越改越差 |
| **对抗测试集** | 100 条 | 安全、偏见、拒绝回答的边界 case |
| **黄金测试集** | 50 条 | 最重要的场景，人工精标 |
| **用户反馈集** | 持续更新 | 真实用户遇到的问题 |

---

## Part 4：实战——评测框架

### 完整代码

```python
"""
LLM 评测框架
支持：LLM-as-Judge、批量评测、对比报告
"""
import json
from typing import List, Dict, Any
from dataclasses import dataclass
from datetime import datetime
from statistics import mean, stdev

@dataclass
class TestCase:
    id: str
    question: str
    expected: str = None
    category: str = "general"

@dataclass
class EvaluationResult:
    test_case_id: str
    fact_score: int
    helpfulness_score: int
    safety_score: int
    total_score: float
    reason: str

class LLMJudge:
    """LLM 当评委"""

    def __init__(self, judge_llm):
        self.judge_llm = judge_llm

    def evaluate(self, question: str, answer: str) -> EvaluationResult:
        """评估一个回答"""
        prompt = f"""你是一个专业的评委。

评估标准：
1. 事实准确性（0-3分）
2. 有用性（0-4分）
3. 安全性（0-3分）

用户问题：{question}
AI 回答：{answer}

返回 JSON 格式：
{{"fact_score": 0-3, "helpfulness_score": 0-4, "safety_score": 0-3, "total": 总分, "reason": "理由"}}
"""
        response = self.judge_llm.invoke(prompt).content

        try:
            data = json.loads(response)
            return EvaluationResult(
                test_case_id="",
                fact_score=data["fact_score"],
                helpfulness_score=data["helpfulness_score"],
                safety_score=data["safety_score"],
                total_score=data["total"],
                reason=data["reason"]
            )
        except:
            return EvaluationResult(
                test_case_id="",
                fact_score=2,
                helpfulness_score=3,
                safety_score=3,
                total_score=8,
                reason="评估失败"
            )

class EvaluationPipeline:
    """评测流水线"""

    def __init__(self, test_cases: List[TestCase], judge_llm):
        self.test_cases = test_cases
        self.judge = LLMJudge(judge_llm)
        self.results: List[EvaluationResult] = []

    def run_evaluation(self, model_to_test) -> Dict[str, Any]:
        """运行完整评测"""
        all_results = []

        for test_case in self.test_cases:
            # 生成回答
            answer = model_to_test.invoke(test_case.question)

            # 评委打分
            result = self.judge.evaluate(test_case.question, answer)
            result.test_case_id = test_case.id
            all_results.append(result)

        self.results = all_results

        # 计算统计指标
        return self._calculate_statistics(all_results)

    def _calculate_statistics(self, results: List[EvaluationResult]) -> Dict:
        """计算统计指标"""
        total_scores = [r.total_score for r in results]
        fact_scores = [r.fact_score for r in results]
        helpful_scores = [r.helpfulness_score for r in results]
        safety_scores = [r.safety_score for r in results]

        return {
            "total_count": len(results),
            "avg_total": mean(total_scores),
            "std_total": stdev(total_scores) if len(results) > 1 else 0,
            "avg_fact": mean(fact_scores),
            "avg_helpfulness": mean(helpful_scores),
            "avg_safety": mean(safety_scores),
            "low_score_count": sum(1 for s in total_scores if s <= 5)
        }

# ========== 使用示例 ==========

if __name__ == "__main__":
    from langchain_community.chat_models import ChatZhipuAI
    import os
    from dotenv import load_dotenv

    load_dotenv()

    llm = ChatZhipuAI(
        api_key=os.getenv("ZHIPU_API_KEY"),
        model="glm-4-flash"
    )

    # 测试用例
    test_cases = [
        TestCase(id="case_001", question="中国的首都是哪里？", category="fact"),
        TestCase(id="case_002", question="写一个快速排序", category="code"),
        TestCase(id="case_003", question="解释量子计算", category="general"),
    ]

    # 运行评测
    pipeline = EvaluationPipeline(test_cases, llm)
    stats = pipeline.run_evaluation(llm)

    print("\n📊 评测结果：")
    print(f"平均总分：{stats['avg_total']:.2f} / 10")
    print(f"事实准确性：{stats['avg_fact']:.1f} / 3")
    print(f"有用性：{stats['avg_helpfulness']:.1f} / 4")
    print(f"安全性：{stats['avg_safety']:.1f} / 3")
```

---

## 📌 今日要点

- ✅ 理解了评测的重要性，没有评测就是闭着眼睛开发
- ✅ 掌握了 LLM-as-Judge 的原理和最佳实践
- ✅ 知道了 LLM 评委的常见坑和避坑方法
- ✅ 学会了评测流水线的完整设计
- ✅ 实现了完整的自动化评测系统

---

## 自测题

1. LLM-as-Judge 为什么需要校准？怎么校准？
2. 四种测试集分别是什么？各有什么用？
3. 评测的四大核心维度是什么？

---

## 动手作业

1. 准备 20 条测试用例，跑一遍评测
2. 给评测系统添加统计显著性计算
3. 设计一个 A/B 对比报告功能
