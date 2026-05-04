> ⚠️ **数据更新时间：2026年2月**
>
> 📢 **来源**：赛博禅心（人人都是产品经理）| https://www.woshipm.com/ai/6345230.html

# Ext14：GLM-5技术报告全解读——a16z称"最好的开源模型"

> 智谱最新40页技术报告揭秘核心突破：744B总参数（激活40B）的MoE架构、仅需20B token即追平DeepSeek效果的DSA稀疏注意力机制、以及完全异步的Slime Agent RL训练框架。

---

## 一、基座：744B参数，40B激活

GLM-5沿用MoE架构（Mixture of Experts），总参数744B，每次推理激活40B，256个专家，80层。

对比上一代GLM-4.5：
- 总参数从355B翻到744B
- 激活参数从32B涨到40B
- 预训练数据从23T token增加到28.5T token

**在Artificial Analysis Intelligence Index上，GLM-5得分50，开源第一。**

a16z称：**"最好的开源模型"**——在文本竞技场和代码竞技场里都排开源第一，整体和Claude Opus 4.5、Gemini 3 Pro同档。

---

## 二、三大架构改动

### 1. MLA + Muon Split注意力

GLM-5用的注意力机制叫MLA（Multi-latent Attention），和DeepSeek-V3同源，通过压缩KV缓存的维度来节省显存。

团队发现：用Muon优化器配MLA时，效果追不上更简单的GQA-8方案。

**解法：Muon Split**——对整块投影矩阵做正交化，改成按每个注意力头单独做，不同的头按自己的节奏更新。效果追平了GQA-8，附带收益：注意力分数在训练过程中自动保持稳定，不用额外裁剪。

### 2. 多Token预测（MTP）

DeepSeek-V3只用1个MTP层训练，推理时预测2个token，但训练和推理方式不一致导致第二个token猜中率偏低。

GLM-5的做法：**训练时用3个MTP层，但这3层共享同一套参数**。推理时内存开销和DeepSeek-V3一样（因为参数只有一套），但猜中率更高。

实测：同样4步推测解码，GLM-5的平均接受长度2.76，DeepSeek-V3.2是2.55。

### 3. DSA稀疏注意力——最核心的效率改动

传统注意力计算是全量的——每个token都要和所有其他token算一遍关系。当上下文从100个token增长到1万个token时，运算量增长1万倍。

**DSA（DeepSeek Sparse Attention）的思路**：加一个轻量级"索引器"，先快速扫一遍所有token，找出和当前token最相关的那些（top-k，k=2048），只对这部分做注意力计算。

**关键数据**：GLM-5用**20B token**做DSA适配，追上了DeepSeek花**943.7B token**训出来的效果——近50倍效率差距。

长序列注意力计算降低1.5-2倍。Agent推理时动辄200K上下文，GPU成本直接砍一半。

---

## 三、预训练数据

三个来源都做了升级：

- **网页数据**：新增基于句子嵌入的DCLM分类器，训练"世界知识分类器"从中低质量网页里捞长尾知识
- **代码数据**：刷新主要代码托管平台快照，unique token增加28%；给Scala、Swift、Lua等低资源语言训练专用分类器
- **数学与科学**：从网页、书籍、论文里收集，LLM打分只保留最具教育价值的部分；严格排除合成数据和AI生成数据

---

## 四、中期训练：三阶段上下文扩展

1. **32K**（1T token）
2. **128K**（500B token）
3. **200K**（50B token）

新增200K阶段主要为了处理超长文档和多文件代码库。

---

## 五、后训练全流程

GLM-5的后训练是一条完整流水线：

**SFT → Reasoning RL → Agentic RL → General RL → 跨阶段在线蒸馏**

### 三种思考模式

- **交错思考（Interleaved Thinking）**：每次响应和工具调用前都思考一轮，提升指令遵循和生成质量
- **保留思考（Preserved Thinking）**：Coding Agent场景里，多轮对话之间保留所有思考内容，不重新推导，适合长程复杂任务
- **轮级思考（Turn-level Thinking）**：按轮次控制开关，简单请求关掉思考降延迟，复杂任务打开提精度

### Reasoning RL

算法基于GRPO + IcePop。核心改动是明确区分"训练模型"和"推理模型"，去掉KL正则项加速训练。纯on-policy，group size 32，batch size 32。

### Agentic RL

基于自研的**Slime RL**框架——完全异步的Agent RL训练框架。

---

## 六、核心洞察

1. **MoE架构是主流**：744B总参/40B激活是当前最优的效率平衡点
2. **DSA稀疏注意力是最大亮点**：20B token追平943B效果，效率提升近50倍
3. **MTP共享参数设计**：3层MTP共享一套参数，推理内存不增加但猜中率提升
4. **后训练流水线完整**：SFT→RL→蒸馏的完整链路
5. **国产开源模型登顶**：在Artificial Analysis上拿到开源最高分，与闭源顶级模型同档
