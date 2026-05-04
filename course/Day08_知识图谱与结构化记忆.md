# Day 9：知识图谱与结构化记忆

> 知识图谱让 Agent 能"理解"实体之间的关系，而不只是记住文本。本节深入实体提取、关系建模、图谱存储与推理。

## Part 1：知识图谱基础

### 为什么需要知识图谱？

RAG 只检索文本片段，知识图谱让 Agent 理解**实体之间的关系**：

```
RAG 检索：
"张三是工程师" → 作为一段文本返回

知识图谱：
(张三) -[职业]-> (工程师)
(张三) -[擅长]-> (Python)
(张三) -[居住地]-> (北京)

→ Agent 可以推理："在北京的 Python 工程师有哪些？"
```

### 三元组结构

知识图谱的核心是**三元组**：

```
(实体1) -[关系]-> (实体2)
```

**示例：**

| 三元组 | 含义 |
|--------|------|
| `(张三) -[是]-> (工程师)` | 张三是工程师 |
| `(张三) -[擅长]-> (Python)` | 张三擅长 Python |
| `(张三) -[在]-> (北京)` | 张三在北京 |
| `(Python) -[属于]-> (编程语言)` | Python 是编程语言 |

---

## Part 2：实体关系提取

### 用 LLM 自动提取

```python
from langchain.prompts import ChatPromptTemplate
from langchain.output_parsers import JsonOutputParser
from pydantic import BaseModel
from typing import List

# 定义输出格式
class Triple(BaseModel):
    entity1: str
    relation: str
    entity2: str

class KnowledgeTriples(BaseModel):
    triples: List[Triple]

# 提取 Prompt
prompt = ChatPromptTemplate.from_messages([
    ("system", """从文本中提取实体和关系，返回 JSON 格式。
三元组格式：(实体1) -[关系]-> (实体2)
只提取有意义的实体和关系，跳过无用的。"""),
    ("user", "{text}")
])

# 使用国产模型提取
from openai import OpenAI
import os
from dotenv import load_dotenv

load_dotenv()
client = OpenAI(api_key=os.getenv("ZHIPU_API_KEY"), base_url="https://open.bigmodel.cn/api/paas/v4")

def extract_triples(text: str):
    response = client.chat.completions.create(
        model="glm-5.1",
        messages=[
            {"role": "system", "content": """从文本中提取实体和关系，返回 JSON 格式。
三元组格式：(实体1) -[关系]-> (实体2)
只提取有意义的实体和关系。"""},
            {"role": "user", "content": text}
        ]
    )
    return response.choices[0].message.content

# 测试
text = "李明是阿里巴巴的工程师，擅长 Python 开发，住在北京。"
triples = extract_triples(text)
print(triples)
```

### 批量提取并存储

```python
def build_knowledge_graph(documents: List[str]):
    """从文档批量构建知识图谱"""
    all_triples = []

    for doc in documents:
        triples = extract_triples(doc)
        all_triples.extend(parse_triples(triples))

    # 存入图数据库
    for triple in all_triples:
        save_to_graph(triple)

    return all_triples
```

---

## Part 3：Neo4j 实战

### Neo4j 简介

Neo4j 是最流行的图数据库，国产替代有 NebulaGraph。

| 数据库 | 特点 | 国产支持 |
|--------|------|----------|
| Neo4j | 最流行，Cypher 查询语言 | ✅ 支持 |
| NebulaGraph | 国产开源，性能高 | ✅ 原生支持 |
| HugeGraph | 百度开源 | ✅ 原生支持 |

### Cypher 查询语言

```cypher
// 创建节点
CREATE (p:Person {name: "张三", age: 30})
CREATE (s:Skill {name: "Python"})

// 创建关系
CREATE (p)-[:擅长]->(s)

// 查询：找到会 Python 的人
MATCH (p:Person)-[r:擅长]->(s:Skill {name: "Python"})
RETURN p.name, p.age

// 查询：多跳推理
MATCH (p:Person)-[:擅长]->(s:Skill)-[:属于]->(c:Category)
WHERE c.name = "编程语言"
RETURN p.name
```

### Python 操作 Neo4j

```python
from neo4j import GraphDatabase

class KnowledgeGraph:
    def __init__(self, uri, user, password):
        self.driver = GraphDatabase.driver(uri, auth=(user, password))

    def add_triple(self, entity1, relation, entity2):
        """添加三元组"""
        with self.driver.session() as session:
            session.run("""
                MERGE (e1 {name: $e1})
                MERGE (e2 {name: $e2})
                MERGE (e1)-[:`{rel}`]->(e2)
            """, e1=entity1, rel=relation, e2=entity2)

    def query(self, cypher):
        """执行查询"""
        with self.driver.session() as session:
            result = session.run(cypher)
            return [dict(record) for record in result]

    def close(self):
        self.driver.close()

# 使用
kg = KnowledgeGraph("bolt://localhost:7687", "neo4j", "password")

# 添加数据
kg.add_triple("张三", "擅长", "Python")
kg.add_triple("张三", "工作于", "阿里巴巴")
kg.add_triple("阿里巴巴", "是", "公司")

# 查询：谁在阿里巴巴工作且擅长 Python
results = kg.query("""
    MATCH (p:Person)-[:擅长]->(s:Skill {name: 'Python'})
    MATCH (p)-[:工作于]->(c:公司 {name: '阿里巴巴'})
    RETURN p.name
""")
```

---

## Part 4：Graph RAG

### 什么是 Graph RAG？

将知识图谱与 RAG 结合，提升检索和推理能力。

```
传统 RAG：
用户问题 → 向量检索 → 返回文本片段

Graph RAG：
用户问题 → 图谱推理 → 确定相关实体 → 向量检索 → 返回上下文
```

### Graph RAG 实现

```python
class GraphRAG:
    def __init__(self, vector_store, knowledge_graph):
        self.vector_store = vector_store
        self.kg = knowledge_graph

    def search(self, query):
        # 1. 从图谱获取相关实体
        entities = self.kg.extract_entities(query)

        # 2. 根据实体获取相关概念
        related = self.kg.get_related_entities(entities)

        # 3. 扩展检索词
        expanded_query = query + " " + " ".join(related)

        # 4. 向量检索
        results = self.vector_store.similarity_search(expanded_query)

        return results

# 使用
graph_rag = GraphRAG(vector_store, kg)
results = graph_rag.search("谁擅长 Python 且在北京工作？")
```

---

## Part 5：图谱推理

### 推理能力

1. **路径推理**：A→B→C 可以推导出 A→C
2. **属性继承**：狗是哺乳动物 → 狗有哺乳动物的特征
3. **冲突检测**：发现矛盾的关系（如某人是全职学生但同时是全职员工）
4. **社区发现**：找出紧密关联的实体群

### 推理示例

```cypher
// 路径推理：朋友的朋友
MATCH (p:Person {name: "张三"})-[:朋友*2]->(friend_of_friend)
RETURN friend_of_friend.name

// 属性继承：某人的所有技能
MATCH (p:Person {name: "张三"})-[:擅长|使用|精通]->(skill)
RETURN skill.name

// 冲突检测
MATCH (p:Person)-[:是]->(x)
WHERE x.name IN ["学生", "在职员工"]
WITH p, count(x) as roles
WHERE roles > 1
RETURN p.name
```

---

## 今日要点

- ✅ 理解了知识图谱的核心：三元组结构
- ✅ 学会了用 LLM 自动提取实体和关系
- ✅ 掌握了 Neo4j 图数据库的基本操作（Cypher）
- ✅ 理解了 Graph RAG：图谱 + 向量检索的结合
- ✅ 了解了图谱推理能力：路径推理、属性继承、冲突检测

---

## 自测题

1. 知识图谱的三元组结构是什么？
2. 什么是 Graph RAG？与传统 RAG 有什么区别？
3. Cypher 中 `MATCH` 和 `CREATE` 的区别是什么？
4. 知识图谱的推理能力有哪些？

---

## 动手作业

1. 使用 LLM 从一段文本中提取实体和关系
2. 安装 Neo4j 或 NebulaGraph，创建节点和关系
3. 实现一个简单的 Graph RAG 系统
