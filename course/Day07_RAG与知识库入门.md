# Day 8：RAG 与知识库入门

> RAG（检索增强生成）是解决 LLM 知识过时、幻觉问题的主流方案，是 Agent 落地的核心技术之一。

## Part 1：什么是 RAG

### RAG = Retrieval-Augmented Generation（检索增强生成）

**问题背景：**
```
LLM 的知识局限：
❌ 知识有截止日期（模型知识更新滞后）
❌ 无法覆盖垂直领域的专业知识
❌ 容易产生幻觉（编造不准确的信息）
❌ 无法获取实时数据（股价、天气等）
```

**RAG 的解决思路：**
```
用户提问 → 检索相关资料 → 将资料作为上下文给 LLM → 生成准确回答
```

### RAG 工作流程

```
                    ┌─────────────┐
                    │  文档数据    │
                    └──────┬──────┘
                           ▼
                 ┌──────────────────┐
                 │  文档切分 (Chunk) │
                 └────────┬─────────┘
                          ▼
                 ┌──────────────────┐
                 │ Embedding 编码    │ ──→ 向量数据库
                 └────────┬─────────┘
                          ▼
用户提问 ──→ Embedding ──→ 相似度检索 ──→ Top-K 相关文档
                                               │
                                               ▼
                                         LLM + 上下文
                                               │
                                               ▼
                                         最终回答
```

### RAG vs 微调对比

| 对比 | RAG | 微调 (Fine-tuning) |
|---|---|---|
| 知识更新 | 实时，替换文档即可 | 需要重新训练 |
| 成本 | 低（无需训练） | 高（需要GPU训练） |
| 适合场景 | 知识库、实时数据 | 风格调整、特定任务 |
| 幻觉控制 | 较好（基于检索内容） | 一般 |
| 响应速度 | 稍慢（多了检索步骤） | 快 |

---

## Part 2：Embedding 与向量数据库原理

### 什么是 Embedding？

**Embedding（嵌入）** = 把文本转换成数字向量的过程。

```python
# 用向量表示后，可以通过"距离"判断语义相似度
Embedding("苹果水果") ≈ [0.12, -0.34, 0.78, ...]  → 水果类
Embedding("苹果手机") ≈ [0.11, -0.31, 0.75, ...]  → 科技类
Embedding("狗")       ≈ [0.65, 0.21, -0.12, ...]  → 动物类

# "苹果水果"和"狗"的距离 > "苹果水果"和"苹果手机"的距离
```

### 主流向量数据库

| 数据库 | 特点 | 国产支持 |
|---|---|---|
| Milvus | 开源，国产最强 | ✅ 完整支持 |
| Qdrant | Rust 编写，性能高 | ✅ 支持 |
| Chroma | 轻量，适合本地 | ✅ 支持 |
| FAISS | Facebook 开源 | ✅ 支持 |
| Pinecone | 云服务，无需运维 | ✅ 支持 |

---

## Part 3：搭建本地知识库

### 文档切分策略

切分（Chunking）是 RAG 效果的关键：

```python
def chunk_text(text, chunk_size=500, overlap=50):
    """
    将长文本切分成小块
    chunk_size: 每块的目标词元数（中文约250字）
    overlap: 块之间的重叠词元数（避免割裂语义）
    """
    chunks = []
    start = 0

    while start < len(text):
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk)
        start = end - overlap

    return chunks
```

### 搭建向量数据库（Chroma 示例）

```python
import chromadb
from chromadb.config import Settings

client = chromadb.Client(Settings(persist_directory="./chroma_db"))
collection = client.create_collection(name="knowledge_base")

def search_similar(query_embedding, collection, top_k=5):
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=top_k
    )
    return results
```

---

## Part 4：实战——用 LangChain 实现 RAG

### 完整 RAG 实战代码

```python
import os
from dotenv import load_dotenv
from langchain_community.document_loaders import TextLoader, DirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.embeddings import ZhipuAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain.chains import RetrievalQA
from langchain_community.chat_models import ChatZhipuAI

load_dotenv()

# 第一步：加载文档
loader = DirectoryLoader("./docs", glob="**/*.txt", loader_cls=TextLoader)
documents = loader.load()

# 第二步：文档切分
text_splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = text_splitter.split_documents(documents)

# 第三步：生成 Embedding 并存入向量库
embeddings = ZhipuAIEmbeddings(api_key=os.getenv("ZHIPU_API_KEY"), model="embedding-3")
vectorstore = Chroma.from_documents(documents=chunks, embedding=embeddings, persist_directory="./chroma_db")

# 第四步：构建 RAG Chain
llm = ChatZhipuAI(api_key=os.getenv("ZHIPU_API_KEY"), model="glm-5.1", temperature=0.0)
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",
    retriever=vectorstore.as_retriever(search_kwargs={"k": 3}),
    return_source_documents=True
)

# 第五步：查询
def ask_question(question):
    result = qa_chain({"query": question})
    print(f"\n回答：{result['result']}")
    return result
```

---

## Part 5：RAG 常见坑与优化

### RAG 失败的四大原因

| 失败原因 | 现象 | 解决方法 |
|---|---|---|
| 检索错了 | 检索出来的 3 段都和问题没关系 | 换更好的 Embedding 模型、优化 chunk 切分 |
| 漏检索了 | 正确答案在第 4 段，但只检索了 Top-3 | 增加 k 值、用 HyDE 改写问题再检索 |
| 上下文太长 | 检索了 5 段，但模型只看了前两段 | 减少 k 值、用 reranker 重排序 |
| 模型不听话 | 检索结果明明写着A，模型回答时偏说B | 降低 temperature、加强 Prompt 约束 |

### chunk 大小的黄金法则

| 场景 | 推荐 chunk 大小 | 原因 |
|---|---|---|
| FAQ 问答 | 100-300 词元 | 每个问答本身就是完整单元 |
| 技术文档 | 500-800 词元 | 代码和技术说明需要上下文 |
| 小说/散文 | 1000-1500 词元 | 长文本的语义连续性很重要 |
| 法律合同 | 300-500 词元 | 精确度要求极高，不能有歧义 |

**经验值：** 对于中文，500 词元 ≈ 750 汉字，正好是两到三段。

---

## 今日要点

- ✅ 理解了 RAG（检索增强生成）的原理和工作流程
- ✅ 掌握了 Embedding 和向量数据库的核心概念
- ✅ 学会了文档切分的常用策略（固定大小、按段落、递归切分）
- ✅ 了解了主流向量数据库（Milvus、Chroma、FAISS 等）
- ✅ 用 LangChain 实现了完整的 RAG 知识库问答系统

---

## 自测题

1. RAG 解决了 LLM 的哪些局限性？
2. 什么是 Embedding？为什么需要它？
3. 文档切分中 `chunk_overlap` 的作用是什么？
4. Chroma 和 Milvus 有什么区别？

---

## 动手作业

1. 准备一个包含 5 篇文章的知识库，用 Chroma 构建向量数据库
2. 比较不同 `chunk_size`（200/500/1000）对检索效果的影响
3. 实现一个完整的 RAG 问答系统
