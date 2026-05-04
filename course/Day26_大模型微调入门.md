# Day 26：大模型微调入门

> RAG 是"借别人的知识"，微调是"教模型新的技能"。本课学习什么时候需要微调、怎么准备数据、完整微调流程、LoRA 实战、评估方法，以及与 RAG 的最佳组合。

## Part 1：RAG vs 微调——什么时候用哪个？

### 灵魂拷问：我需要微调吗？

**很多人上来就说"我要微调一个模型"，但其实 90% 的场景根本不需要微调！**

| 场景 | 推荐方案 |
|------|----------|
| ❌ 我想给模型加一些知识，让它知道我们公司的产品信息 | **RAG 就够了！** 不需要微调 |
| ❌ 我想让模型回答用户手册里的内容 | **RAG 就够了！** 不需要微调 |
| ❌ 我想让模型说话更符合我们的风格 | **好的 Prompt + 几个例子就够了** |
| ✅ 我想让模型学会一种特定的输出格式 | 可以微调，效果更好 |
| ✅ 我想让模型学会一种特定的推理方式 | 必须微调 |
| ✅ 我想用小模型达到大模型的效果 | 必须微调 |

### RAG 和微调的核心区别

| 维度 | RAG | 微调 Fine-tuning |
|------|------|-----------------|
| **核心能力** | 注入新知识 | 教会新技能/新风格 |
| **知识更新** | 方便，加文档就行 | 麻烦，每次都要重新微调 |
| **来源追溯** | 可以，能看到回答来自哪个文档 | 不行，不知道模型从哪学的 |
| **需要的数据量** | 几条就行 | 至少几百条，最好几千条 |
| **成本** | 低（无需 GPU 训练） | 中等，几百到几千块 |
| **开发周期** | 几天 | 几周 |

### 黄金法则

```
能不用微调就不用微调！
如果 Prompt 工程能解决的，就不要用 Few-Shot；
如果 Few-Shot 能解决的，就不要用微调；
只有当你试了所有方法都不行的时候，再考虑微调！
```

---

## Part 2：微调的三个最佳适用场景

### 场景 1：降本增效
- 你现在用 GPT-4 或国产大模型，每次调用成本较高
- 用大模型的回答去微调一个小模型（7B）
- 微调后的小模型能达到大模型 80-90% 的效果
- 成本降到原来的 1/10，速度快 5 倍

### 场景 2：特定格式/特定风格
- 客服 Agent，要求回答必须符合特定格式和话术
- 微调后，模型每次回答都和你家客服一模一样
- 不需要在 Prompt 里写一大堆格式说明

### 场景 3：特定推理能力
- 让模型学会一种特定的思考方式/推理链
- 每次都严格按照 ReAct 格式思考和行动
- 让模型学会行业特定的分析框架

---

## Part 3：微调数据准备实战

### 数据是微调成功的决定性因素

微调成功 = 80% 数据质量 + 20% 训练技巧

### 数据格式：ChatML

主流国产模型通用 ChatML 格式：

```json
{
  "messages": [
    {"role": "system", "content": "你是我们公司的客服代表。回复要礼貌、专业、简洁。"},
    {"role": "user", "content": "我买的东西坏了，怎么退换货？"},
    {"role": "assistant", "content": "非常抱歉给您带来不便！请您按照以下步骤操作：\n1. 登录 App 进入「我的订单」\n2. 找到该商品点击「申请售后」\n3. 选择「退换货」并上传照片\n4. 等待审核，通常 24 小时内会有回复"}
  ]
}
```

### 数据采集与生成

```python
import json
import random
from typing import List, Dict

class SFTDataGenerator:
    """SFT 微调数据生成器"""

    def __init__(self, template_path: str = None):
        self.templates = []

    def add_template(self, system_prompt: str, user_template: str, assistant_template: str):
        """添加数据模板"""
        self.templates.append({
            "system": system_prompt,
            "user": user_template,
            "assistant": assistant_template
        })

    def generate_from_template(self, variables: Dict[str, List], num_samples: int = 100) -> List[Dict]:
        """从模板生成数据"""
        data = []
        for _ in range(num_samples):
            tmpl = random.choice(self.templates)
            # 填充变量
            filled = {}
            for k, v in variables.items():
                filled[k] = random.choice(v)

            sample = {
                "messages": [
                    {"role": "system", "content": tmpl["system"]},
                    {"role": "user", "content": tmpl["user"].format(**filled)},
                    {"role": "assistant", "content": tmpl["assistant"].format(**filled)}
                ]
            }
            data.append(sample)
        return data

    def validate_format(self, data: List[Dict]) -> List[str]:
        """验证数据格式"""
        errors = []
        for i, sample in enumerate(data):
            if "messages" not in sample:
                errors.append(f"第 {i} 条：缺少 messages 字段")
                continue
            msgs = sample["messages"]
            if not any(m["role"] == "user" for m in msgs):
                errors.append(f"第 {i} 条：缺少 user 消息")
            if not any(m["role"] == "assistant" for m in msgs):
                errors.append(f"第 {i} 条：缺少 assistant 消息")
            # 检查内容非空
            for m in msgs:
                if not m.get("content", "").strip():
                    errors.append(f"第 {i} 条：{m['role']} 内容为空")
        return errors

    def save_to_jsonl(self, data: List[Dict], output_path: str):
        """保存为 JSONL 格式"""
        with open(output_path, "w", encoding="utf-8") as f:
            for sample in data:
                f.write(json.dumps(sample, ensure_ascii=False) + "\n")
        print(f"已保存 {len(data)} 条数据到 {output_path}")

# 使用示例
generator = SFTDataGenerator()
generator.add_template(
    system_prompt="你是电商客服，回复要礼貌专业。",
    user_template="我想退掉{product}，因为{reason}",
    assistant_template="您好！关于{product}的退货申请，已为您记录。请将商品寄回至退货地址，我们收到后会尽快处理退款。"
)

variables = {
    "product": ["手机壳", "蓝牙耳机", "充电宝", "数据线"],
    "reason": ["质量问题", "不想要了", "尺寸不合适", "颜色不喜欢"]
}

dataset = generator.generate_from_template(variables, num_samples=200)
errors = generator.validate_format(dataset)
if errors:
    print(f"发现 {len(errors)} 个格式问题")
    for e in errors[:5]:
        print(f"  - {e}")
else:
    print("数据格式验证通过！")

generator.save_to_jsonl(dataset, "sft_data.jsonl")
```

### 数据质量检查清单

```
□ 每条数据都有 system/user/assistant 三部分
□ 输入真实（模拟真实用户会怎么问）
□ 输出高质量（最好是人工编写或专家审核）
□ 覆盖所有场景（至少 10 种不同的用户问题类型）
□ 数量足够（起步 200 条，理想 1000+ 条）
□ 格式统一（全部用 ChatML 格式）
□ 没有重复数据
□ 没有错误/有害内容（重要！）
□ 训练集/验证集 9:1 划分
```

---

## Part 4：LoRA 微调实战

### LoRA 核心原理

LoRA（Low-Rank Adaptation）：不改模型原来的权重，只训练两个很小的矩阵。

```
原始权重矩阵 W (d×k)    →   冻结不更新
LoRA 低秩矩阵 A (d×r)   →   训练
LoRA 低秩矩阵 B (r×k)   →   训练
r << min(d, k)，通常 r=8 或 16
```

### 完整微调流程

```bash
# 1. 安装 LLaMA Factory
pip install llama-factory

# 2. 准备数据（放到 data/ 目录）
# sft_data.jsonl

# 3. 注册数据集
# 编辑 dataset_info.json，添加：
# "my_agent_data": {
#   "file_name": "sft_data.jsonl",
#   "formatting": "sharegpt",
#   "columns": {
#     "messages": "messages"
#   }
# }

# 4. LoRA 微调（Qwen 7B 为例）
llamafactory-cli train \
    --model_name_or_path Qwen/Qwen2.5-7B-Instruct \
    --stage sft \
    --dataset my_agent_data \
    --template qwen \
    --finetuning_type lora \
    --lora_target q_proj,v_proj \
    --lora_rank 8 \
    --lora_alpha 16 \
    --output_dir qwen-lora-my-agent \
    --per_device_train_batch_size 4 \
    --per_device_eval_batch_size 4 \
    --gradient_accumulation_steps 4 \
    --learning_rate 5e-5 \
    --num_train_epochs 3.0 \
    --lr_scheduler_type cosine \
    --logging_steps 10 \
    --save_steps 100 \
    --eval_steps 100 \
    --evaluation_strategy steps \
    --val_size 0.1 \
    --warmup_steps 100 \
    --fp16 True
```

### 关键参数详解

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| **lora_rank** | 8-64 | 越大效果越好，大部分场景 8 够了 |
| **lora_alpha** | 16-128 | 一般是 rank 的 2 倍 |
| **lora_target** | q_proj,v_proj | 只训练注意力层，最省显存 |
| **learning_rate** | 1e-4 ~ 5e-5 | 不要太大，LoRA 比全量微调需要稍大 LR |
| **epochs** | 2-5 | 3 轮足够了，太多会过拟合 |
| **val_size** | 0.1 | 10% 数据做验证 |
| **per_device_train_batch_size** | 尽量大 | 受显存限制，7B 模型 4-8 合适 |

### 国产模型微调对比

| 模型 | 推荐框架 | 显存需求（LoRA） | 模板 |
|------|----------|------------------|------|
| **Qwen2.5-7B** | LLaMA Factory | ~16GB | qwen |
| **GLM-4-9B** | LLaMA Factory | ~20GB | glm4 |
| **DeepSeek-V2.5** | LLaMA Factory | ~24GB | deepseek |

```python
# GLM-4 微调命令
llamafactory-cli train \
    --model_name_or_path THUDM/glm-4-9b-chat \
    --stage sft \
    --dataset my_agent_data \
    --template glm4 \
    --finetuning_type lora \
    --lora_rank 8 \
    --output_dir glm4-lora-my-agent \
    --per_device_train_batch_size 4 \
    --learning_rate 5e-5 \
    --num_train_epochs 3.0
```

---

## Part 5：微调评估方法

### 评估维度

| 维度 | 方法 | 指标 |
|------|------|------|
| **输出格式** | 检查是否符合目标格式 | 格式正确率 |
| **内容质量** | 人工评分或 LLM 评分 | 1-5 分 |
| **任务完成率** | 在测试集上的成功率 | 通过率 |
| **安全性** | 检查有害内容 | 拒绝率 |
| **灾难性遗忘** | 在通用 benchmark 上测试 | 性能下降比例 |

### 评估实现

```python
def evaluate_sft_model(original_model, finetuned_model, test_data: List[Dict]):
    """评估微调效果"""

    def check_format(output: str, expected_format: str) -> bool:
        """检查输出格式"""
        if expected_format == "json":
            try:
                json.loads(output)
                return True
            except:
                return False
        elif expected_format == "code":
            return "def " in output or "class " in output
        return True

    def llm_score(output: str, expected: str, judge_llm) -> float:
        """用 LLM 打分"""
        prompt = f"""请评估以下回答的质量（1-5分）：

参考回答：{expected}
模型回答：{output}

评分标准：
5=完美 4=良好 3=一般 2=差 1=完全错误

只返回数字分数："""
        score = judge_llm.invoke(prompt)
        try:
            return float(score.content.strip())
        except:
            return 3.0

    results = {
        "format_accuracy": 0,
        "avg_score": 0,
        "total": len(test_data)
    }

    for item in test_data:
        # 格式检查
        format_ok = check_format(item["output"], item.get("format", "text"))
        if format_ok:
            results["format_accuracy"] += 1

        # 内容评分
        score = llm_score(item["output"], item["expected"], original_model)
        results["avg_score"] += score

    results["format_accuracy"] /= len(test_data)
    results["avg_score"] /= len(test_data)

    return results

# 使用
eval_results = evaluate_sft_model(llm, finetuned_model, test_dataset)
print(f"格式正确率：{eval_results['format_accuracy']:.1%}")
print(f"平均评分：{eval_results['avg_score']:.2f}/5.0")
```

### A/B 测试

```python
def ab_test(original_model, finetuned_model, test_questions):
    """A/B 测试：原模型 vs 微调模型"""
    results = []
    for q in test_questions:
        orig_answer = original_model.invoke(q)
        ft_answer = finetuned_model.invoke(q)

        results.append({
            "question": q,
            "original": orig_answer,
            "finetuned": ft_answer,
            "improved": len(ft_answer) > len(orig_answer)  # 简单指标
        })

    improved_count = sum(1 for r in results if r["improved"])
    print(f"微调模型在 {improved_count}/{len(results)} 个问题上表现更好")
    return results
```

---

## Part 6：微调常见坑与调试

| 坑 | 表现 | 原因 | 解决方法 |
|------|------|------|----------|
| **过拟合** | 训练 loss 低但评估 loss 高 | 数据太少或 epoch 太多 | 减少 epoch、增加数据量、增大 dropout |
| **灾难性遗忘** | 新技能会了但通用能力下降 | LoRA rank 太大 | 减小 rank（4-8）、使用 WARM 策略 |
| **数据质量问题** | 微调后比没调还差 | 数据有噪声、格式不对 | 洗数据、人工审核、数据增强 |
| **格式不一致** | 训练集和实际输入格式不同 | 模板没选对 | 检查 --template 参数匹配模型 |
| **显存溢出** | OOM 错误 | batch_size 太大或序列太长 | 减小 batch_size、开启 gradient_checkpointing |

```python
# 梯度检查点——显存不够时的救命稻草
llamafactory-cli train \
    --model_name_or_path Qwen/Qwen2.5-7B-Instruct \
    --stage sft \
    --dataset my_agent_data \
    --template qwen \
    --finetuning_type lora \
    --lora_rank 8 \
    --output_dir qwen-lora \
    --per_device_train_batch_size 2 \  # 减小 batch
    --gradient_accumulation_steps 8 \   # 补偿 batch 减小
    --gradient_checkpointing true \      # 开启梯度检查点
    --fp16 true
```

---

## Part 7：微调 + RAG 最佳实践

```plaintext
基础大模型（GLM/Qwen/DeepSeek）
    ↓
SFT 微调（让它学会输出格式和推理方式）
    ↓
DPO 对齐（让它更听话，减少有害输出）
    ↓
RAG 检索（注入你的私有知识和最新信息）
    ↓
Guardrails（安全过滤 + 格式校验）
    ↓
你的专属 Agent！
```

**核心原则：** 微调负责"教技能"，RAG 负责"给知识"，两者互补。

| 责任 | 谁负责 | 例子 |
|------|--------|------|
| 输出格式 | 微调 | 客服 Agent 的标准回复话术 |
| 推理方式 | 微调 | ReAct 思维链格式 |
| 产品知识 | RAG | 最新的产品规格、价格、库存 |
| 实时信息 | RAG | 订单状态、物流信息 |
| 安全对齐 | 微调 + DPO | 拒绝有害请求 |

---

## 今日要点

- 知道了什么时候该用 RAG，什么时候该用微调（90% 场景不需要微调）
- 掌握了 SFT 数据准备的完整流程和格式要求
- 学会了用 LLaMA Factory 进行 LoRA 微调
- 掌握了微调评估的四个维度和 A/B 测试方法
- 了解了微调常见坑的诊断和解决方法
- 理解了微调 + RAG 的最佳组合方式

---

## 自测题

1. 为什么很多人微调效果不好？最大的原因是什么？
2. LoRA 的核心原理是什么？为什么比全量微调快？
3. 微调数据准备最关键的三要素是什么？
4. 什么是灾难性遗忘？LoRA 微调如何避免？
5. 微调和 RAG 的最佳分工是什么？

---

## 动手作业

1. 准备 100 条客服场景的微调数据，格式化为 ChatML JSONL
2. 用 LLaMA Factory 在 Qwen-7B 上做 LoRA 微调
3. 对比微调前后模型在同一问题上的输出差异
4. （进阶）设计一个微调 + RAG 的客服 Agent 架构