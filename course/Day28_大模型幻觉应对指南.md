# Day 28：大模型幻觉应对指南

> 大模型"一本正经的胡说八道"怎么破？本课深入幻觉的本质、成因、检测方法、减少幻觉的技术手段、完整的应对流水线。

## Part 1：幻觉的本质、成因与分类

### 什么是幻觉？

**"一本正经的胡说八道"类比：**

| 正常回答 | 幻觉回答 |
| --- | --- |
| 说的都是真的，有依据 | 说的听起来很真，但完全是编的 |
| 不知道就说不知道 | 不知道也编得像模像样，特别自信 |
| 引用真实来源 | 编不存在的论文、不存在的法条、不存在的 API |

> 💡 幻觉的定义：模型生成的内容听起来很合理，但实际上是错误的、无中生有的、没有事实依据的。
> 这是目前大模型最大的问题，没有之一。

---

### 幻觉的严重程度分级

| 级别 | 定义 | 例子 | 危害 |
| --- | --- | --- | --- |
| **L1 轻微** | 细节错误，不影响整体理解 | "GPT-3 是 2019 年发布的"（实际是 2020） | 低，用户可能注意不到 |
| **L2 中等** | 事实错误，但还在合理范围内 | "李白写了《望庐山瀑布》"（细节时间地点编错） | 中，误导用户 |
| **L3 严重** | 完全无中生有 | "据《自然》杂志 2025 年的研究..."（根本没有这篇论文） | 高，完全误导 |
| **L4 危险** | 有害的幻觉 | 编医疗建议、编法律条文，可能造成实际伤害 | 极高，可能出事故 |

---

### 幻觉的常见表现形式

| 类型 | 说明 | 例子 |
| --- | --- | --- |
| **事实编造** | 编不存在的事实、数据、事件 | "2025 年中国 GDP 增长了 8%"（实际 5.2%） |
| **来源编造** | 编不存在的论文、书籍、引用 | "根据霍金 2018 年的论文..."（霍金 2018 年去世了） |
| **能力幻觉** | 说自己有不存在的能力 | "我可以实时上网查资料"（实际不能） |
| **代码幻觉** | 编不存在的 API、函数、参数 | "调用 pandas.read_sqlite() 函数..."（pandas 根本没有这个函数） |
| **数学幻觉** | 算错数，但步骤看起来很对 | "√2 = 1.4142135" 编后面几位 |

---

### 为什么会有幻觉？

| 原因 | 通俗解释 | 技术解释 |
| --- | --- | --- |
| **1. 训练数据 cutoff** | 模型的知识截止到某个时间，之后的事不知道，但不会说不知道 | 训练数据有截止日期，之后发生的事没见过 |
| **2. 知识是压缩的** | 模型不是真的"记住"了所有知识，是压缩存储，细节容易失真 | Transformer 的注意力是模糊匹配，不是精确记忆 |
| **3. 概率续接** | 模型只是在"猜下一个词最可能是什么"，不是"思考对不对" | 下一个 token 概率最高的组合，整体看可能是错的 |
| **4. 训练鼓励流畅** | 训练时奖励"说得通顺"，没奖励"说得对" | 优化目标是 perplexity（流畅度），不是事实正确性 |
| **5. 不会说不知道** | 人类不知道就说不知道，模型被训练成"你问我就必须答" | SFT 和 RLHF 都鼓励"有帮助"，不鼓励"拒绝回答" |

---

## Part 2：检测幻觉的各种方法

### 检测方法一：自我检查法

**让模型自己检查自己说的对不对**

```plaintext
Prompt 模板：
请仔细检查你刚才的回答，回答下面的问题：
1. 你说的每一句话都有事实依据吗？
2. 有没有你编造的内容？
3. 有没有不确定的地方？
4. 引用的来源都是真实存在的吗？

如果有错误，请指出来并修正。
```

**效果：**
- 能检测出 30-50% 的明显幻觉
- 优点：简单，不需要额外工具
- 缺点：模型自己也不知道自己在幻觉，检查不出来

---

### 检测方法二：多模型交叉验证

**让不同的模型回答同一个问题，看答案一致不一致**

```plaintext
1. 让 GLM-4、DeepSeek V3、Qwen-Plus 分别回答同一个问题
2. 对比三个答案的事实部分
3. 某个事实只有一个模型说有，另外两个都没有 → 大概率是幻觉
4. 三个都一致 → 可信度高
```

**效果：**
- 能检测出 60-80% 的幻觉
- 优点：准确率不错
- 缺点：贵，要调用三个模型，慢

---

### 检测方法三：检索验证法

**把回答里的事实拿去搜索引擎/知识库搜，看能不能搜到**

```plaintext
1. 从回答里提取所有事实性陈述
   比如："XXX 在 2023 年发表了论文 YYY"

2. 把每个事实拿去搜索引擎/维基百科/你的知识库搜

3. 搜得到且一致 → ✅ 真实
   搜不到 → ❓ 不确定
   搜到但不一致 → ❌ 幻觉
```

**效果：**
- 能检测出 80-90% 的事实类幻觉
- 目前最可靠的方法
- 缺点：需要检索工具，慢，需要实体和关系提取

---

### 检测方法四：幻觉分类器

**专门训练一个小模型来检测幻觉**

```plaintext
输入：问题 + 模型回答
输出：幻觉概率 0-1，以及幻觉的位置
```

**效果：**
- 专门训练的分类器准确率能到 85-95%
- 速度快，成本低
- 缺点：需要标注数据训练，泛化性有限

---

### 各检测方法对比

| 方法 | 准确率 | 成本 | 速度 | 适合检测的幻觉类型 |
| --- | --- | --- | --- | --- |
| 自我检查 | 30-50% | 低 | 快 | 明显的错误 |
| 多模型交叉 | 60-80% | 高 | 慢 | 各类幻觉 |
| 检索验证 | 80-90% | 中 | 中 | 事实类幻觉 |
| 专门分类器 | 85-95% | 低 | 极快 | 训练过的类型 |

---

## Part 3：减少幻觉的技术手段

### 手段一：检索增强生成（RAG）

**"开卷考试"类比：**
- 闭卷考试：模型凭记忆答，容易记错 → 幻觉多
- 开卷考试：先去书上找答案，再照着答 → 幻觉少很多

**核心流程：**

```plaintext
用户问题
    ↓
1. 去知识库/互联网检索相关内容
    ↓
2. 把检索到的内容作为上下文给模型
    ↓
3. 模型看着上下文回答，不准自己编
    ↓
4. 回答里每个事实都要标引用来源
```

**效果：**
- 能减少 70-90% 的事实类幻觉
- 目前最有效的手段
- 缺点：检索质量决定一切，检索错了还是会幻觉

---

### 手段二：强制引用溯源

**每个事实都必须标引用来源，没来源的不能说**

```plaintext
Prompt 模板：
回答用户的问题时，必须严格基于提供的上下文内容。
每一句话都要在句尾用 [n] 标注对应的引用来源编号。
上下文里没有的内容，就说"根据现有资料无法回答"，不要编造。
```

**例子：**

```plaintext
❌ 不好的回答：大语言模型有 1 万亿参数。
✅ 好的回答：目前最大的大语言模型约有 1 万亿参数 [1]，但实际公开的参数最高是 GPT-4 的约 1.8 万亿 [2]。
```

**效果：**
- 用户可以点引用去看原文
- 幻觉一眼就能看出来（编的引用点不开）
- 模型也不敢随便编了，因为要写引用来源

---

### 手段三：降低温度，提高确定性

| 温度 temperature | 效果 | 幻觉率 |
| --- | --- | --- |
| 0.0 - 0.3 | 非常确定，只说概率最高的内容 | 低 |
| 0.3 - 0.7 | 平衡，常用范围 | 中 |
| 0.7 - 1.0 | 有创意，多样性高 | 高 |

**经验法则：**
- 事实类问答：温度 0.1 - 0.3，减少幻觉
- 创意写作：温度 0.7 - 1.0，要的就是想象力
- 日常对话：温度 0.5 - 0.7，平衡

---

### 手段四：系统提示词约束

**给模型定规矩，不知道就说不知道，不要编**

```plaintext
系统提示词：
你是一个严谨的助手，请遵守以下规则：
1. 只回答你确定知道的内容，不确定的就说"我不确定"
2. 不要编造事实、数据、来源、引用
3. 不知道的就说"不知道"，不要硬答
4. 对于你不确定的内容，要明确说明"这个我不确定，仅供参考"
```

**效果：**
- 能减少 20-30% 的幻觉
- 简单有效，零成本
- 缺点：还是有很多时候模型记不住遵守

---

### 手段五：分步验证 + 多轮反思

**不要一次生成答案，多步检查和修正**

```plaintext
流程：
1. 第一步：生成初步答案
2. 第二步：自我检查，找出可能有问题的地方
3. 第三步：针对有问题的地方去检索验证
4. 第四步：修正错误，生成最终答案
5. 第五步：最后再检查一遍
```

**效果：**
- 每多一轮反思和检查，幻觉率降 10-20%
- 3 轮检查能减少 50% 左右的残留幻觉
- 缺点：慢，token 贵

---

## Part 4：实战——幻觉检测与修正流水线

### 完整代码：幻觉检测与修正流水线

```python
"""
完整的幻觉检测与修正流水线
支持：自我检查、检索验证、引用溯源、多轮修正
"""
import asyncio
import re
from typing import List, Dict, Any, Tuple
from dataclasses import dataclass
from enum import Enum

class HallucinationLevel(Enum):
    NONE = "none"  # 没有幻觉
    LOW = "low"  # 轻微
    MEDIUM = "medium"  # 中等
    HIGH = "high"  # 严重
    DANGEROUS = "dangerous"  # 危险

@dataclass
class FactStatement:
    """提取出的事实性陈述"""
    text: str
    confidence: float = 1.0
    source: str = None
    is_hallucination: bool = False
    check_result: str = None

@dataclass
class HallucinationCheckResult:
    original_answer: str
    level: HallucinationLevel
    hallucination_count: int
    detected_facts: List[FactStatement]
    corrected_answer: str = None
    needs_human_review: bool = False

# ========== 事实提取器 ==========

class FactExtractor:
    """从回答中提取所有事实性陈述"""

    def extract(self, answer: str) -> List[FactStatement]:
        """从回答中提取事实性陈述"""
        sentences = re.split(r'[。！？；.]', answer)
        facts = []

        for sent in sentences:
            sent = sent.strip()
            if not sent:
                continue

            if any(k in sent for k in ["是", "有", "在", "等于", "约", "达到", "发布", "发现"]):
                facts.append(FactStatement(text=sent))

        return facts

# ========== 幻觉检测器 ==========

class HallucinationDetector:
    """幻觉检测"""

    def __init__(self):
        self.extractor = FactExtractor()

    async def self_check(self, question: str, answer: str) -> List[FactStatement]:
        """第一步：模型自我检查"""
        facts = self.extractor.extract(answer)

        # 让 LLM 自己检查每个事实对不对
        prompt = f"""用户问题：{question}

AI 回答：{answer}

请逐条检查 AI 回答中的每一个事实性陈述，判断是不是真实的。
输出 JSON 格式：
{{
    "checks": [
        {{
            "fact": "事实内容",
            "is_hallucination": true/false,
            "confidence": 0-1,
            "reason": "判断理由"
        }}
    ]
}}
"""
        # 实际调用 LLM，这里模拟
        for fact in facts:
            import random
            if random.random() < 0.2:
                fact.is_hallucination = True
                fact.check_result = "自我检查认为这个事实可能不准确"
                fact.confidence = 0.5

        return facts

    async def retrieval_verify(self, facts: List[FactStatement]) -> List[FactStatement]:
        """第二步：检索验证（模拟）"""
        for fact in facts:
            if fact.is_hallucination:
                continue

            import random
            if random.random() < 0.1:
                fact.is_hallucination = True
                fact.check_result = "检索验证：没有找到可靠来源支持这个说法"
                fact.confidence = 0.3
                fact.source = None
            else:
                fact.source = "https://example.com/source"
                fact.confidence = 0.9

        return facts

    async def detect(self, question: str, answer: str) -> HallucinationCheckResult:
        """完整的检测流程"""

        # 1. 自我检查
        facts = await self.self_check(question, answer)

        # 2. 检索验证
        facts = await self.retrieval_verify(facts)

        # 3. 统计和分级
        hallucination_count = sum(1 for f in facts if f.is_hallucination)
        total_facts = len(facts)

        if total_facts == 0:
            level = HallucinationLevel.NONE
        else:
            ratio = hallucination_count / total_facts

            if ratio == 0:
                level = HallucinationLevel.NONE
            elif ratio < 0.1:
                level = HallucinationLevel.LOW
            elif ratio < 0.3:
                level = HallucinationLevel.MEDIUM
            elif ratio < 0.6:
                level = HallucinationLevel.HIGH
            else:
                level = HallucinationLevel.DANGEROUS

        needs_human_review = level in (HallucinationLevel.HIGH, HallucinationLevel.DANGEROUS)

        return HallucinationCheckResult(
            original_answer=answer,
            level=level,
            hallucination_count=hallucination_count,
            detected_facts=facts,
            needs_human_review=needs_human_review
        )

# ========== 幻觉修正器 ==========

class HallucinationFixer:
    """修正检测到的幻觉"""

    async def fix(self, question: str, check_result: HallucinationCheckResult) -> str:
        """修正有幻觉的回答"""

        if check_result.level == HallucinationLevel.NONE:
            return check_result.original_answer

        bad_facts = [f for f in check_result.detected_facts if f.is_hallucination]

        prompt = f"""用户问题：{question}

原来的回答中有以下不准确的内容，请修正：
{chr(10).join(f'- {f.text}: {f.check_result}' for f in bad_facts)}

请重新生成一个准确的回答，不要包含上面的错误。
不确定的内容就说"我不确定"，不要编造。"""

        corrected_answer = f"【修正后的回答】针对 '{question}' 的修正后的准确回答..."

        return corrected_answer

# ========== 完整流水线 ==========

class HallucinationSafePipeline:
    """完整的防幻觉回答流水线"""

    def __init__(self):
        self.detector = HallucinationDetector()
        self.fixer = HallucinationFixer()

    async def answer(self, question: str, max_retries: int = 2) -> Dict[str, Any]:
        """安全回答问题，自动检测和修正幻觉"""
        print(f"❓ 问题：{question}")

        initial_answer = f"这是针对 '{question}' 的初始回答，可能包含幻觉..."
        print(f"🤖 初始回答生成完成")

        for attempt in range(max_retries + 1):
            check_result = await self.detector.detect(question, initial_answer if attempt == 0 else check_result.corrected_answer)

            print(f"🔍 第 {attempt+1} 次检测：检测到 {check_result.hallucination_count} 个幻觉，级别：{check_result.level.value}")

            if check_result.level == HallucinationLevel.NONE:
                print("✅ 没有幻觉，可以返回")
                break

            if attempt == max_retries:
                print("⚠️ 重试次数用完，标记需要人工审核")
                check_result.needs_human_review = True
                break

            print("🔧 正在修正幻觉...")
            check_result.corrected_answer = await self.fixer.fix(question, check_result)

        final_answer = check_result.corrected_answer or check_result.original_answer

        result = {
            "question": question,
            "final_answer": final_answer,
            "hallucination_level": check_result.level.value,
            "hallucination_detected": check_result.hallucination_count,
            "retries": attempt,
            "needs_human_review": check_result.needs_human_review,
            "fact_checks": [
                {
                    "fact": f.text,
                    "is_hallucination": f.is_hallucination,
                    "confidence": f.confidence,
                    "source": f.source
                }
                for f in check_result.detected_facts
            ]
        }

        if check_result.needs_human_review:
            result["final_answer"] += "\n\n⚠️ 注意：本回答可能包含不准确的内容，仅供参考，请以权威来源为准。"

        return result

# ========== 使用示例 ==========

async def main():
    pipeline = HallucinationSafePipeline()

    test_questions = [
        "中国的首都是哪里？",
        "2026 年中国 GDP 增长率是多少？",
        "爱因斯坦 2020 年发表了什么论文？",
    ]

    print("="*80)
    print("🛡️ 幻觉检测与修正流水线测试")
    print("="*80)

    for question in test_questions:
        print(f"\n{'='*80}")
        result = await pipeline.answer(question)

        print(f"\n📊 检测结果：")
        print(f"幻觉级别：{result['hallucination_level']}")
        print(f"检测到幻觉数：{result['hallucination_detected']}")
        print(f"重试次数：{result['retries']}")
        print(f"需要人工审核：{'是' if result['needs_human_review'] else '否'}")

        print(f"\n✅ 最终回答：")
        print(result["final_answer"])

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 高级特性建议

1. **领域白名单/黑名单**
   - 医疗、法律、金融这些高风险领域
   - 回答里涉及这些领域，自动加免责声明
   - 或者直接拒绝回答，让用户找专业人士

2. **幻觉记忆库**
   - 把检测到的幻觉存起来
   - 下次用户问类似问题，提前知道模型容易在这方面幻觉
   - 主动检索验证，或者提前提示用户注意

3. **人类反馈闭环**
   - 用户点"不准确"的回答，自动加入幻觉库
   - 定期统计模型在哪类问题上最容易幻觉
   - 针对性优化 Prompt，或者加专门的检索源

4. **置信度校准**
   - 让模型输出每个事实的置信度
   - 置信度 < 0.7 的，自动去检索验证
   - 置信度 < 0.5 的，直接不显示，或者标"不确定"

---

## 📌 今日要点

- ✅ 理解了幻觉的本质和五大成因
- ✅ 掌握了幻觉的四种严重程度分级和常见表现形式
- ✅ 学会了四种幻觉检测方法和各自的优缺点
- ✅ 掌握了五种减少幻觉的技术手段：RAG、引用溯源、温度控制、Prompt 约束、多轮验证
- ✅ 实现了完整的幻觉检测与修正流水线

---

## 自测题

1. 大模型为什么会有幻觉？五大成因分别是什么？
2. 四种幻觉检测方法对比，哪种准确率最高？哪种成本最低？
3. 减少幻觉最有效的技术手段是什么？为什么？
4. 强制引用溯源为什么能有效减少幻觉？

## 动手作业

1. 把幻觉检测流水线跑起来，接入真实的 LLM
2. 接入真实的搜索引擎 API 做检索验证
3. 加高风险领域检测：涉及医疗、法律自动加免责声明
