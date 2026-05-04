# Day 38：大规模 RAG 架构

> 几百个文档的 RAG 谁都会做，但百万级文档的 RAG 就需要架构设计了。本课讲解怎么从小文档库扩展到百万级。

## Part 1：RAG 的扩展性挑战

### 文档多了之后会遇到什么问题？

**100 个文档的时候：**
- 什么都不用想，Chroma/FAISS 一把梭，直接用，效果还不错
- 检索一次几十毫秒，快得很

**10,000 个文档的时候：**
- 检索变慢了，一次要几百毫秒
- 召回率下降了，很多相关的文档搜不出来

**100,000 个文档的时候：**
- 单机向量数据库顶不住了
- 检索一次要几秒，用户体验很差
- 召回率惨不忍睹

**1,000,000 个文档的时候：**
- 大部分人直接放弃了

---

### 不同规模的 RAG 架构选择

| 文档规模 | 推荐架构 | 向量数据库 | 检索策略 |
| --- | --- | --- | --- |
| < 1000 | 简单 RAG | Chroma/FAISS | 直接向量检索 |
| 1000 - 10 万 | 标准 RAG | Milvus 单节点 | 混合检索 + Rerank |
| 10 万 - 100 万 | 高级 RAG | Milvus 分布式 | 分层索引 + 多路召回 |
| > 100 万 | 企业级 RAG | Elasticsearch + 向量插件 | 多级缓存 |

---

### 百万级 RAG 的三大核心挑战

| 挑战 | 表现 | 解决方向 |
| --- | --- | --- |
| **1. 检索质量下降** | 真正相关的被淹没在噪声里 | 分层索引、分类、标签 |
| **2. 性能太慢** | 一次检索要好几秒 | 分布式向量数据库、缓存 |
| **3. 成本太高** | 向量计算贵，存储贵 | 冷热分层、成本优化 |

---

## Part 2：分层索引与多级检索架构

### 核心思路：不要在 100 万文档里直接搜！

**就像你找书不会在图书馆里一本书一本书翻，你会先找分类，再找书架，再找哪一层。**

```plaintext
用户提问
    ↓
【第一层：分类路由】
    这个问题属于哪个类别？
    ↓ 先过滤！
【第二层：摘要检索】
    不是直接搜全文，先搜每个文档的摘要
    ↓
【第三层：全文检索】
    在这 N 个文档里做全文精细检索
    ↓
【第四层：Rerank 重排序】
    把前 K 个结果重新排序
    ↓
最终结果
```

**这个架构能把百万级文档的检索问题，变成在几千个文档里检索的问题！**

---

### 第一层：文档分类与元数据过滤

**建索引的时候就给每个文档打好标签：**

```json
{
  "category": "技术",
  "product": "A产品",
  "date": "2024-03-15",
  "importance": "核心文档",
  "author": "张三"
}
```

**检索的时候先过滤，再检索：**

```plaintext
用户问："A 产品 2024 年的技术架构是什么？"
    ↓
第一步：过滤
    分类 = 技术
    产品 = A产品
    时间 >= 2024-01-01
    100万文档 → 过滤后只剩下 5000 篇
```

---

### 第二层：摘要检索

**问题：** 长文档有几万字，直接切成块，重要信息可能在某个块里，但检索的时候搜不到

**解决方案：** 每个文档先写一个摘要，先检索摘要，找到相关的文档，再在这个文档里全文检索

```plaintext
建索引的时候：
    每个长文档 → 写一个 200 字的摘要 → 摘要向量化

检索的时候：
    第一步：搜摘要索引 → 找到最相关的 10 个文档
    第二步：只在这 10 个文档里搜 → 找到最相关的片段
```

---

### 多级检索的效果对比

| 检索方式 | 100 万文档召回率 | 检索时间 |
| --- | --- | --- |
| 直接全文向量检索 | 30% | 3-5 秒 |
| 过滤 + 全文向量检索 | 55% | 1-2 秒 |
| 过滤 + 摘要检索 + 全文检索 | 80% | 500-800ms |
| 过滤 + 摘要检索 + 全文检索 + Rerank | 90%+ | 800ms-1.2秒 |

---

## Part 3：分布式向量数据库与性能优化

### 主流分布式向量数据库对比

| 数据库 | 最大规模 | 性能（QPS 单节点） | 特点 |
| --- | --- | --- | --- |
| **Milvus** | 百亿级 | 10000+ | 最成熟，生态最好 |
| **Qdrant** | 十亿级 | 8000+ | Rust 写的，快，API 友好 |
| **Weaviate** | 十亿级 | 5000+ | Go 写的，生态好 |
| **Pinecone** | 百亿级 | 10000+ | Serverless，不用运维 |

---

### 性能优化技巧

#### 1. 索引类型选择

| 索引类型 | 建索引速度 | 检索速度 | 召回率 | 适合场景 |
| --- | --- | --- | --- | --- |
| FLAT | 快 | 慢（O(n)） | 100% | 小数据量 |
| IVF | 中 | 中 | 90-95% | 中等数据量 |
| HNSW | 慢 | 快 | 95-98% | 大数据量 |

**大部分场景选 HNSW 就对了！**

---

#### 2. 缓存策略

```plaintext
用户提问
    ↓
【第一层：精确匹配缓存】
    这个问题之前有人问过吗？
    有 → 直接返回，不用走 RAG
    ↓
【第二层：语义缓存】
    有没有语义很相似的问题？
    有 → 直接返回，不用再检索
    ↓
正常走 RAG 流程
```

**效果：** 80% 的请求会命中缓存，平均响应时间从 1 秒降到 100 毫秒！

---

#### 3. 冷热分层

| 数据层级 | 访问频率 | 存储方式 |
| --- | --- | --- |
| **热数据** | 最近 3 个月（占 20%，占 80% 访问量） | 内存 + SSD |
| **温数据** | 3 个月-1 年（占 30%，占 15% 访问量） | SSD |
| **冷数据** | 1 年以上（占 50%，占 5% 访问量） | 普通硬盘 |

---

## Part 4：实战——大规模 RAG 系统

### 完整代码：分层索引 + 多级检索

```python
"""
大规模 RAG 系统
分层索引 + 多级检索 + 缓存 + 冷热分层
"""
import asyncio
import time
import hashlib
from typing import List, Dict, Any, Optional, Tuple
from dataclasses import dataclass, field
from enum import Enum
import random

class DataTier(Enum):
    HOT = "hot"
    WARM = "warm"
    COLD = "cold"

@dataclass
class Document:
    """文档"""
    id: str
    title: str
    content: str
    summary: str = ""
    category: str = "general"
    product: str = ""
    date: str = ""
    importance: str = "normal"
    tier: DataTier = DataTier.HOT
    access_count: int = 0
    last_accessed: float = field(default_factory=time.time)

@dataclass
class SearchResult:
    """搜索结果"""
    document: Document
    score: float
    matched_chunk: str = ""

@dataclass
class RetrievalConfig:
    """检索配置"""
    enable_routing: bool = True
    enable_summary_search: bool = True
    enable_rerank: bool = True
    top_k_initial: int = 50
    top_k_final: int = 10

class MetadataRouter:
    """元数据路由器"""

    def __init__(self):
        self.category_index: Dict[str, List[Document]] = {}
        self.product_index: Dict[str, List[Document]] = {}
        self.date_index: Dict[str, List[Document]] = {}

    def index(self, doc: Document):
        """索引文档"""
        if doc.category not in self.category_index:
            self.category_index[doc.category] = []
        self.category_index[doc.category].append(doc)

        if doc.product:
            if doc.product not in self.product_index:
                self.product_index[doc.product] = []
            self.product_index[doc.product].append(doc)

        if doc.date:
            year = doc.date[:4] if len(doc.date) >= 4 else ""
            if year not in self.date_index:
                self.date_index[year] = []
            self.date_index[year].append(doc)

    def filter(self, category: str = None, product: str = None,
               min_date: str = None) -> List[Document]:
        """过滤文档"""
        candidates = []

        if category and category in self.category_index:
            candidates = self.category_index[category]
        elif product and product in self.product_index:
            candidates = self.product_index[product]
        else:
            return []

        if min_date:
            min_year = min_date[:4] if len(min_date) >= 4 else ""
            candidates = [d for d in candidates if d.date >= min_date]

        return candidates

class SummaryIndex:
    """摘要索引"""

    def __init__(self):
        self.summaries: List[Tuple[Document, List[float]]] = []

    def index(self, doc: Document, embedding: List[float]):
        """索引摘要"""
        self.summaries.append((doc, embedding))

    def search(self, query_embedding: List[float], doc_ids: set = None,
               top_k: int = 20) -> List[SearchResult]:
        """搜索摘要"""
        results = []

        for doc, embedding in self.summaries:
            if doc_ids and doc.id not in doc_ids:
                continue

            score = self._cosine_similarity(query_embedding, embedding)
            results.append(SearchResult(document=doc, score=score))

        results.sort(key=lambda x: x.score, reverse=True)
        return results[:top_k]

    def _cosine_similarity(self, a: List[float], b: List[float]) -> float:
        dot = sum(x * y for x, y in zip(a, b))
        norm_a = sum(x * x for x in a) ** 0.5
        norm_b = sum(x * x for x in b) ** 0.5
        return dot / (norm_a * norm_b + 1e-8)

class SemanticCache:
    """语义缓存"""

    def __init__(self):
        self.exact_cache: Dict[str, List[SearchResult]] = {}
        self.semantic_cache: Dict[str, Tuple[List[SearchResult], float]] = {}

    def get_exact(self, query: str) -> Optional[List[SearchResult]]:
        """精确匹配"""
        key = hashlib.md5(query.encode()).hexdigest()
        return self.exact_cache.get(key)

    def set_exact(self, query: str, results: List[SearchResult]):
        """设置精确缓存"""
        key = hashlib.md5(query.encode()).hexdigest()
        self.exact_cache[key] = results

    def get_semantic(self, query: str, threshold: float = 0.95) -> Optional[List[SearchResult]]:
        """语义匹配"""
        for key, (results, score) in self.semantic_cache.items():
            if score >= threshold:
                return results
        return None

class LargeScaleRAG:
    """大规模 RAG 系统"""

    def __init__(self):
        self.router = MetadataRouter()
        self.summary_index = SummaryIndex()
        self.cache = SemanticCache()
        self.documents: Dict[str, Document] = {}
        self.config = RetrievalConfig()

    def add_document(self, doc: Document):
        """添加文档"""
        self.documents[doc.id] = doc

        # 元数据索引
        self.router.index(doc)

        # 摘要索引
        embedding = [random.random() for _ in range(128)]  # 实际应该用 Embedding 模型
        self.summary_index.index(doc, embedding)

    async def retrieve(self, query: str, filters: Dict = None) -> List[SearchResult]:
        """检索"""
        start_time = time.time()

        # 1. 检查缓存
        cached = self.cache.get_exact(query)
        if cached:
            print(f"✅ 缓存命中，返回 {len(cached)} 个结果")
            return cached

        # 2. 元数据路由
        if self.config.enable_routing and filters:
            candidates = self.router.filter(
                category=filters.get("category"),
                product=filters.get("product"),
                min_date=filters.get("min_date")
            )
            doc_ids = {doc.id for doc in candidates}
            print(f"📂 路由过滤：剩余 {len(doc_ids)} 个候选文档")
        else:
            doc_ids = None

        # 3. 摘要检索
        query_embedding = [random.random() for _ in range(128)]  # 实际应该用 Embedding 模型

        if self.config.enable_summary_search:
            summary_results = self.summary_index.search(
                query_embedding,
                doc_ids=doc_ids,
                top_k=self.config.top_k_initial
            )
            print(f"📄 摘要检索：找到 {len(summary_results)} 个候选文档")
        else:
            summary_results = []

        # 4. Rerank
        if self.config.enable_rerank:
            # 简化版：直接按摘要相关性排序
            final_results = summary_results[:self.config.top_k_final]
        else:
            final_results = summary_results[:self.config.top_k_final]

        # 5. 缓存结果
        self.cache.set_exact(query, final_results)

        elapsed = (time.time() - start_time) * 1000
        print(f"⏱️ 检索耗时：{elapsed:.0f}ms")

        return final_results

    def get_stats(self) -> Dict:
        """获取统计"""
        return {
            "total_documents": len(self.documents),
            "cached_queries": len(self.cache.exact_cache),
            "categories": len(self.router.category_index),
            "products": len(self.router.product_index)
        }

async def main():
    print("=" * 70)
    print("📚 大规模 RAG 系统测试")
    print("=" * 70)

    rag = LargeScaleRAG()

    # 1. 添加测试文档
    print("\n📄 添加测试文档...")
    for i in range(1000):
        doc = Document(
            id=f"doc_{i}",
            title=f"文档 {i} - 产品 {i % 10}",
            content=f"这是文档 {i} 的详细内容..." * 10,
            summary=f"这是文档 {i} 的摘要...",
            category=["技术", "产品", "运营", "财务"][i % 4],
            product=f"产品 {i % 10}",
            date=f"202{4 + i % 3}-{(i % 12) + 1:02d}-15",
            importance=["核心", "普通", "参考"][i % 3]
        )
        rag.add_document(doc)

    # 2. 测试检索
    print("\n" + "=" * 70)
    print("🔍 测试检索：")
    print("-" * 70)

    queries = [
        ("2024年产品的技术文档", {"category": "技术"}),
        ("产品5的相关文档", {"product": "产品 5"}),
        ("2025年的运营报告", {"category": "运营", "min_date": "2025-01-01"}),
    ]

    for query, filters in queries:
        print(f"\n📝 查询：{query}")
        if filters:
            print(f"   过滤条件：{filters}")

        results = await rag.retrieve(query, filters)

        print(f"   返回结果：{len(results)} 个")
        for i, result in enumerate(results[:3], 1):
            print(f"   {i}. [{result.document.category}] {result.document.title}")
            print(f"      相似度：{result.score:.3f}")

    # 3. 统计
    stats = rag.get_stats()
    print("\n" + "=" * 70)
    print(f"📊 系统统计：")
    print(f"   总文档数：{stats['total_documents']}")
    print(f"   缓存查询数：{stats['cached_queries']}")
    print(f"   分类数：{stats['categories']}")
    print(f"   产品数：{stats['products']}")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 📌 今日要点

- ✅ 知道了不同规模的文档库应该用什么样的 RAG 架构
- ✅ 掌握了百万级文档的分层索引多级检索架构
- ✅ 了解了主流的分布式向量数据库
- ✅ 掌握了 RAG 的三大性能优化神器：索引优化、缓存、冷热分层

---

## 自测题

1. 不同规模的文档库应该用什么架构？各有什么特点？
2. 分层索引多级检索是怎么工作的？为什么能提升召回率？
3. 三大性能优化神器是什么？各有什么效果？
4. 为什么大部分人做百万级 RAG 都会失败？他们缺了什么？

## 动手作业

1. 把上面的 RAG 系统跑起来
2. 添加真实的 Embedding 模型
3. 实现完整的 Rerank 模块
