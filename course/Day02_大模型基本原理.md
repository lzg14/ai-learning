# Day 2：大模型基本原理

> 本课程帮助理解大模型的工作原理，包括 Transformer 架构、核心概念，以及国产大模型账号申请和 API 调用。

## Part 1：大模型工作原理

### 一个类比：超级填空高手

把大模型想象成一个**读完了互联网上所有文字的超级填空高手**：

```
输入："我喜欢吃苹果，因为苹果___"
模型输出：它根据学到的统计规律，给每个可能的词打分：
  "很甜"    →  85%
  "有营养"  →  70%
  "好吃"    →  30%
选择概率最高的词输出。
```

### Transformer 工作流程

现代大模型的核心是 **Transformer 架构**，工作流程分为三步：

**1. 输入编码**
```
文字 → 分词 → 转换为词元序列 → Embedding 向量化
```

**2. 注意力计算**
这是 Transformer 的核心，让模型能够"看到"上下文。每个词都能"关注"其他所有词，理解词与词之间的关系。

**3. 输出生成**
```
向量 → 预测下一个词 → 再预测下一个词 → ...
```
模型逐个生成词元，直到遇到结束符或达到最大长度。

### Transformer 三要素

```
Transformer = Embedding + 注意力机制 + 位置编码
```

| 要素 | 作用 |
|------|------|
| Embedding | 把每个词元映射到向量 |
| 注意力机制 | 计算词与词之间的关联强度 |
| 位置编码 | 给词元加上位置信息 |

---

## Part 2：核心概念详解

### 词元（Token）

**词元是大模型处理文本的最小单位。**

| 类型 | 说明 |
|------|------|
| 英文 | 1词元 ≈ 0.75个单词 ≈ 4个字符 |
| 中文 | 1词元 ≈ 0.5-1个汉字 |
| 标点 | 每个标点也算词元 |

**为什么不是按"字"算？**
- 模型不知道"字"，只知道"词元"
- "人工智能"可能是1个词元，也可能是2-3个
- 这就是为什么 API 计费按词元而不是字数

### Embedding 向量化

把文字转换成数字向量，让计算机能计算语义关系：

```
文字世界                          数学世界
"猫" ────────→ [0.2, -0.8, 1.5]   可以计算距离
"狗" ────────→ [0.3, -0.7, 1.4]   猫和狗很近
"汽车" ──────→ [0.9, 0.5, -0.3]   猫和汽车远
```

**余弦相似度**：判断两个向量是否"语义相近"
- 1 = 完全相同
- 0 = 完全无关
- -1 = 完全相反

### 自注意力机制（Self-Attention）

自注意力是 Transformer 最核心的创新。理解它，才能理解大模型为什么这么强大。

**传统方法的问题：RNN**
```
"那只猫在窗台上睡觉，然后它醒了"
         ↑         ↑
RNN必须记住"猫"才能理解"它"指代什么
如果句子很长，信息会丢失
```

**自注意力的解决方案**
```
"那只猫在窗台上睡觉，然后它醒了"

注意力权重：
"它" → 0.95 → "猫"  （它=猫）
"它" → 0.03 → "窗台"
"它" → 0.02 → "睡觉"
```

每个词直接和其他所有词计算关联度，不需要依次传递信息。

**计算过程**
1. 每个词生成三个向量：Query（查询）、Key（键）、Value（值）
2. 用 Query 和所有 Key 做点积，得到注意力分数
3. Softmax 归一化，得到注意力权重
4. 用权重对 Value 加权求和，得到输出

```python
# 简化的注意力计算
attention_scores = Q @ K.T  # Query 和 Key 做点积
attention_weights = softmax(attention_scores / sqrt(d_k))
output = attention_weights @ V  # 用权重对 Value 加权
```

### 多头注意力（Multi-Head Attention）

**为什么需要多头？**
- 每个"头"可以关注不同类型的关系
- 有的头关注语法结构，有的头关注语义关联，有的头关注指代关系

```
输入："小明和小红一起去看电影"

头1（语法）："小明" ↔ "小红"（并列关系）
头2（动作）："看" ↔ "电影"（动宾关系）
头3（指代）："一起" ↔ "去"（方式关系）
```

**参数数量**
- 单头注意力：参数少，关注能力有限
- 多头（如8头、16头）：每个头独立学习，能捕捉更丰富的关系

### 位置编码（Positional Encoding）

Transformer 本身不关心词的顺序，需要额外加入位置信息。

**正弦/余弦编码**
```python
# 位置编码公式
PE(pos, 2i)   = sin(pos / 10000^(2i/d))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d))
```

这样的编码方式：
- 每个位置有唯一编码
- 相对位置可以通过线性变换获得
- 可以处理超长序列

### 前馈神经网络（FFN）

每个 Transformer 层还包含一个前馈网络：

```
输入 → Linear → ReLU → Linear → 输出
```

这通常是模型参数最多的部分（约2/3的参数在这里）。

### Transformer 层结构

```
Transformer Encoder Block:
┌─────────────────────────────────────┐
│  Input                              │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│  Multi-Head Self-Attention          │
│  (自注意力)                          │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│  Add & Norm                         │
│  (残差连接 + 层归一化)               │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│  Feed Forward                       │
│  (前馈神经网络)                      │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│  Add & Norm                         │
└──────────────┬──────────────────────┘
               ↓
               ↓ (重复 N 层)
```

现代大模型通常有几十到上百层 Transformer（如 GPT-4 猜测有 100+ 层）。

---

## Part 3：国产大模型账号申请

后续课程会使用这些平台的 API 进行实战演示，首先需要申请账号。

### GLM（智谱 AI）

**官网**：https://open.bigmodel.cn/

**申请步骤**：
1. 访问开放平台，点击"注册"
2. 使用手机号验证码注册（或微信）
3. 登录后在"API 密钥管理"创建新的 API Key
4. 获得 Key 后即可调用 API

**SDK 安装**：
```bash
pip install zhipuai
```

**基本调用**：
```python
from zhipuai import ZhipuAI

client = ZhipuAI(api_key="你的API Key")

response = client.chat.completions.create(
    model="glm-4-plus",
    messages=[{"role": "user", "content": "你好"}]
)

print(response.choices[0].message.content)
```

**免费额度**：新用户赠送 18 元体验金

---

### DeepSeek（深度求索）

**官网**：https://platform.deepseek.com/

**申请步骤**：
1. 访问开放平台，点击"登录/注册"
2. 支持手机号、邮箱注册
3. 进入控制台，在"API Key"创建新密钥
4. 充值或使用免费额度

**SDK 安装**：
```bash
pip install openai
```

**基本调用**：
```python
from openai import OpenAI

client = OpenAI(
    api_key="你的API Key",
    base_url="https://api.deepseek.com"
)

response = client.chat.completions.create(
    model="deepseek-chat",
    messages=[{"role": "user", "content": "你好"}]
)

print(response.choices[0].message.content)
```

**免费额度**：每日有一定免费调用量

---

### Qwen（阿里云通义千问）

**官网**：https://bailian.console.aliyun.com/

**申请步骤**：
1. 登录阿里云账号（可用淘宝/支付宝账号）
2. 进入百炼平台，点击"开通服务"
3. 创建 API Key
4. 获取 Key 后通过 dashscope 调用

**SDK 安装**：
```bash
pip install dashscope
```

**基本调用**：
```python
import os
from openai import OpenAI

client = OpenAI(
    api_key="你的API Key",
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1"
)

response = client.chat.completions.create(
    model="qwen-plus",
    messages=[{"role": "user", "content": "你好"}]
)

print(response.choices[0].message.content)
```

**免费额度**：新用户赠送一定免费额度

---

### Kimi（月之暗面）

**官网**：https://platform.moonshot.cn/

**申请步骤**：
1. 访问开放平台，点击"注册"
2. 支持手机号、邮箱注册
3. 在控制台创建 API Key
4. 开始使用

**SDK 安装**：
```bash
pip install openai
```

**基本调用**：
```python
from openai import OpenAI

client = OpenAI(
    api_key="你的API Key",
    base_url="https://api.moonshot.cn/v1"
)

response = client.chat.completions.create(
    model="moonshot-v1-8k",
    messages=[{"role": "user", "content": "你好"}]
)

print(response.choices[0].message.content)
```

**免费额度**：新用户赠送 15 元体验金

---

## Part 4：模型对比与选型

### 国产模型对比

| 模型 | 公司 | 上下文 | 价格 | 核心亮点 |
|---|---|---|---|---|
| GLM-5.1 | 智谱 AI | 100万 | 低 | SWE-Bench 全球第一 |
| DeepSeek V4 | 深度求索 | 100万 | 极低 | 编程+Agent 双强 |
| Qwen 3.6-Plus | 阿里云 | 100万 | 低 | 开源生态最广 |
| Kimi K2.6 | 月之暗面 | 1M | 中等 | 超长上下文 |

> 💡 **性价比推荐**：DeepSeek V4 和 GLM-5.1

### 选型建议

- **中文项目首选**：GLM-5.1 或 Qwen（中文理解更强）
- **编程任务**：DeepSeek V4 或 GLM-5.1
- **成本敏感**：DeepSeek V4（性价比最高）
- **超长上下文**：Kimi K2.6（支持 1M 词元）

---

## 今日收获

- ✅ 理解了大模型的工作原理——概率预测、Transformer 架构
- ✅ 掌握了词元、Embedding、注意力机制的核心概念
- ✅ 理解了自注意力、多头注意力、位置编码的原理
- ✅ 学会了国产四大平台的账号申请和 API 调用

---

## 自测题

1. 词元的官方中文名是什么？哪一天由哪个机构正式公布的？
2. 自注意力机制的核心计算过程是什么？
3. 为什么 Transformer 需要位置编码？
4. 国产四大模型的最新版本是什么？各自的核心优势是什么？

---

## 动手作业

1. 注册至少一个国产大模型平台的账号（建议 GLM 或 DeepSeek）
2. 获取 API Key，尝试运行上面的示例代码
3. 思考：你的项目更适合使用哪个模型？为什么？
