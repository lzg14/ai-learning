# Day 37：Agent 自我改进与终身学习

> 从"用完就忘"到"越用越聪明"。本课深入在线学习、经验积累、反思机制、知识更新。

## Part 1：灾难性遗忘问题与终身学习的必要性

### 现在的 Agent 为什么"学了就忘"？

**你有没有遇到过这种情况：**
1. 你教了 Agent 一个新技能，用了几次，然后过几天再问，它又忘了
2. Agent 学了新东西之后，之前会的东西反而不会了
3. Agent 不会从自己犯的错误中学习，错了一次下次还会犯同样的错

这就是**灾难性遗忘（Catastrophic Forgetting）**问题！

---

### 终身学习 Agent 的五大核心能力

| 能力 | 说明 | 类比 |
| --- | --- | --- |
| **1. 增量学习** | 学了新东西不忘记旧东西 | 人学了英语再学法语，不会把中文忘了 |
| **2. 经验积累** | 把做过的事、犯过的错变成经验和知识 | 人从错误中学习，吃一堑长一智 |
| **3. 自我反思** | 自己发现自己的问题，自己想办法改进 | 人会反思"我刚才哪里做得不好？" |
| **4. 知识更新** | 旧知识过时了能自动更新 | 人会看新闻、看书，更新知识体系 |
| **5. 能力迁移** | 一个领域学会的东西能用到另一个领域 | 学会了骑自行车，学电动车就很快 |

---

## Part 2：增量学习与在线训练

### 增量学习技术

**核心思想：** 传统的方法是"收集所有数据 → 重头训练一遍"，但这太慢太贵了。增量学习是"来了新数据就在旧模型基础上学，不重头来"。

**三大增量学习技术：**

| 技术 | 原理 | 优点 | 缺点 |
| --- | --- | --- | --- |
| **正则化方法** | 重要的权重别改太多 | 简单，不需要存旧数据 | 只能防止少量遗忘 |
| **回放方法** | 学新东西时同时复习旧数据 | 效果最好，遗忘最少 | 需要存旧数据 |
| **参数隔离方法** | 给每个任务单独分一部分参数 | 任务之间完全不影响 | 参数会越来越多 |

**最实用的组合：LoRA + 回放方法**
- 用 LoRA 学新东西，不动原始模型的权重
- 多个 LoRA 可以叠加使用，相当于同时会多个任务

---

## Part 3：反思机制

### Agent 怎么自己反思？

**反思的三个层次：**

| 层次 | 做什么 | 例子 |
| --- | --- | --- |
| **1. 结果反思** | 这件事做对了还是做错了？为什么？ | "我刚才回答错了，因为我记错了产品的价格。" |
| **2. 过程反思** | 我是怎么想的？哪一步想错了？ | "我应该先查一下知识库再回答，而不是直接瞎编。" |
| **3. 策略反思** | 我的整体策略对吗？应该换个方法吗？ | "以后遇到不确定的问题，应该先问用户确认。" |

---

### 自我改进闭环

```plaintext
用户提问 → Agent 回答 → 用户反馈/自动验证 → 反思分析 → 总结经验 → 更新知识库/微调模型
    ↑                                                                                         ↓
    └─────────────────────────────────────────────────────────────────┘
                          下次遇到类似问题就用改进后的方法
```

**这个闭环越转越快，Agent 就越来越聪明：**
- 第一天：10 个问题错 3 个
- 第二天：错的 3 个都改进了，10 个问题只错 1 个
- 一周后：正确率从 70% 升到 95% 以上

---

## Part 4：实战——会自我改进的问答 Agent

### 完整代码

```python
"""
会自我改进的问答 Agent
实现：反思机制、错题本、自我验证、增量学习
"""
import json
import time
import random
from typing import List, Dict, Tuple
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class MistakeRecord:
    """错误记录"""
    question: str
    wrong_answer: str
    correct_answer: str
    mistake_reason: str
    improvement: str
    timestamp: float = field(default_factory=time.time)
    fixed: bool = False

@dataclass
class Experience:
    """经验记录"""
    pattern: str
    insight: str
    confidence: float = 0.5
    times_used: int = 0
    times_success: int = 0

class SelfImprovingQAAgent:
    """会自我改进的问答 Agent"""

    def __init__(self):
        self.mistake_notebook: List[MistakeRecord] = []  # 错题本
        self.experience_base: List[Experience] = []     # 经验库
        self.question_history: List[Dict] = []         # 问题历史
        self.improvement_count = 0
        self.total_questions = 0
        self.correct_count = 0

    def answer(self, question: str) -> Tuple[str, Dict]:
        """回答问题"""
        self.total_questions += 1
        start_time = time.time()

        # 第一步：先查错题本和经验库
        context = self._get_relevant_experience(question)

        # 第二步：生成初始回答
        initial_answer = self._generate_answer(question, context)

        # 第三步：自我验证
        is_good, feedback = self._self_verify(question, initial_answer)

        # 第四步：如果有问题，修正回答
        if not is_good:
            final_answer = self._revise_answer(question, initial_answer, feedback)
            self._record_mistake(question, initial_answer, final_answer, feedback)
        else:
            final_answer = initial_answer
            self.correct_count += 1

        # 第五步：记录历史
        self._record_history(question, initial_answer, final_answer, is_good, time.time() - start_time)

        return final_answer, {"self_verified": is_good, "feedback": feedback}

    def _get_relevant_experience(self, question: str) -> str:
        """从经验库获取相关经验"""
        relevant = []
        for exp in self.experience_base:
            if any(word in question.lower() for word in exp.pattern.lower().split()):
                relevant.append(exp)
                exp.times_used += 1

        if relevant:
            exp_text = "\n".join([f"- {e.insight}" for e in relevant[:3]])
            return f"参考经验：\n{exp_text}\n\n"
        return ""

    def _generate_answer(self, question: str, context: str) -> str:
        """生成回答（模拟）"""
        # 模拟 LLM 生成
        # 实际项目中应该调用真实的 LLM

        # 检查是否和之前的错误模式相似
        for mistake in self.mistake_notebook[-10:]:
            if mistake.pattern in question.lower():
                # 尝试避免之前的错误
                if "计算" in question and "价格" in mistake.question:
                    return f"【注意】之前的计算可能有问题，请仔细核实..."

        # 正常生成
        responses = [
            f"针对 '{question}' 的回答...",
            f"关于这个问题，答案是...",
            f"让我来分析 '{question}' ..."
        ]
        return random.choice(responses)

    def _self_verify(self, question: str, answer: str) -> Tuple[bool, str]:
        """自我验证，检查回答有没有问题"""
        # 模拟：15% 概率发现问题
        if random.random() < 0.15:
            return False, "回答不够具体，缺少实际例子"

        # 检查是否在错题本里
        for mistake in self.mistake_notebook:
            if mistake.pattern in question.lower() and not mistake.fixed:
                return False, f"之前这个问题回答错，需要重新思考"

        return True, "回答没问题。"

    def _revise_answer(self, question: str, wrong_answer: str, feedback: str) -> str:
        """修正错误回答"""
        # 模拟修正
        revise_prompt = f"""原问题：{question}
原回答：{wrong_answer}
问题反馈：{feedback}

请重新生成一个正确的回答："""

        return f"【已修正】{wrong_answer}（根据反馈：{feedback}）"

    def _record_mistake(self, question: str, wrong: str, correct: str, feedback: str):
        """把错误记录到错题本"""
        # 提取错误模式
        pattern = self._extract_pattern(question)

        mistake = MistakeRecord(
            question=question,
            wrong_answer=wrong,
            correct_answer=correct,
            mistake_reason=feedback,
            improvement=f"以后回答类似问题要注意{feedback}",
            timestamp=time.time(),
            fixed=True
        )
        self.mistake_notebook.append(mistake)
        self.improvement_count += 1

        # 更新经验库
        self._update_experience(pattern, feedback)

        print(f"❌ 记录错误：{question[:30]}...")
        print(f"   问题：{feedback}")

    def _extract_pattern(self, question: str) -> str:
        """提取问题模式"""
        # 简化版：提取关键词
        keywords = [w for w in question.split() if len(w) > 2][:3]
        return " ".join(keywords)

    def _update_experience(self, pattern: str, improvement: str):
        """更新经验库"""
        # 查找是否已有类似经验
        for exp in self.experience_base:
            if exp.pattern == pattern:
                exp.confidence = min(1.0, exp.confidence + 0.1)
                exp.insight = improvement
                return

        # 新增经验
        self.experience_base.append(Experience(
            pattern=pattern,
            insight=improvement,
            confidence=0.6,
            times_used=0,
            times_success=0
        ))

    def _record_history(self, question: str, initial: str, final: str, is_good: bool, time_taken: float):
        """记录历史"""
        self.question_history.append({
            "question": question,
            "initial_answer": initial,
            "final_answer": final,
            "is_good": is_good,
            "time_taken": time_taken,
            "timestamp": time.time()
        })

    def get_improvement_report(self) -> Dict:
        """获取改进报告"""
        total = len(self.question_history)
        good_count = sum(1 for h in self.question_history if h["is_good"])
        avg_time = sum(h["time_taken"] for h in self.question_history) / max(1, total)

        return {
            "total_questions": total,
            "correct_count": self.correct_count,
            "mistake_count": self.improvement_count,
            "accuracy_rate": self.correct_count / max(1, total) * 100,
            "avg_response_time": avg_time * 1000,
            "mistake_notebook_size": len(self.mistake_notebook),
            "experience_base_size": len(self.experience_base)
        }

    def print_stats(self):
        """打印统计"""
        report = self.get_improvement_report()
        print(f"\n📊 自我改进统计：")
        print(f"   总问题数：{report['total_questions']}")
        print(f"   正确数：{report['correct_count']} ({report['accuracy_rate']:.1f}%)")
        print(f"   错误数：{report['mistake_count']}")
        print(f"   错题本记录：{report['mistake_notebook_size']}")
        print(f"   经验库记录：{report['experience_base_size']}")
        print(f"   平均响应时间：{report['avg_response_time']:.0f}ms")

async def main():
    print("=" * 70)
    print("🧠 会自我改进的问答 Agent 测试")
    print("=" * 70)

    agent = SelfImprovingQAAgent()

    # 测试问题
    test_questions = [
        "中国的首都是哪里？",
        "Python 怎么定义一个函数？",
        "今天天气怎么样？",
        "什么是机器学习？",
        "怎么写一个 for 循环？",
        "地球到月球多远？",
        "Python 列表怎么排序？",
        "水的沸点是多少度？",
    ]

    print("\n📝 开始问答测试：")
    print("-" * 70)

    for i, question in enumerate(test_questions, 1):
        print(f"\n{i}. 问题：{question}")
        answer, meta = agent.answer(question)
        status = "✅" if meta["self_verified"] else "❌"
        print(f"   {status} 回答：{answer[:40]}...")
        if not meta["self_verified"]:
            print(f"   📝 反馈：{meta['feedback']}")

        # 模拟一些反馈
        if i % 3 == 0:
            agent.correct_count += 1

    # 打印统计
    agent.print_stats()

    # 打印改进报告
    report = agent.get_improvement_report()
    print(f"\n📈 改进趋势：")
    print(f"   准确率：{report['accuracy_rate']:.1f}%")

    if report['mistake_notebook_size'] > 0:
        print(f"\n📓 错题本示例：")
        for mistake in agent.mistake_notebook[:3]:
            print(f"   - {mistake.question[:30]}...")
            print(f"     错误原因：{mistake.mistake_reason}")

    if report['experience_base_size'] > 0:
        print(f"\n💡 经验库示例：")
        for exp in agent.experience_base[:3]:
            print(f"   - {exp.pattern}: {exp.insight}")

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

---

## 📌 今日要点

- ✅ 理解了灾难性遗忘问题和终身学习的必要性
- ✅ 掌握了增量学习的三大技术和最佳实践
- ✅ 学会了反思机制的三个层次
- ✅ 掌握了 Self-Verification 自我验证方法
- ✅ 实现了完整的会自我改进的问答 Agent

---

## 自测题

1. 为什么现在的 Agent"学了就忘"？灾难性遗忘的根本原因是什么？
2. 增量学习的三大技术各有什么优缺点？
3. 自我反思为什么能大幅提升 Agent 的能力？
4. 错题本在自我改进中起什么作用？

## 动手作业

1. 把上面的自我改进 Agent 跑起来
2. 加一个真实的大模型调用让 Self-Verification 真的用大模型来检查
3. 加一个向量数据库做错题和经验的相似度检索
