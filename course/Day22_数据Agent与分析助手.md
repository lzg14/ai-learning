# Day 24：数据 Agent 与分析助手

> 本课深入数据 Agent 的架构设计、自动 SQL 生成、数据分析和可视化报表生成。

## Part 1：数据 Agent 概述

### 什么是数据 Agent？

```
传统数据分析：
用户 → 写 SQL → 执行 → 看结果 → 画图

数据 Agent：
用户：帮我分析一下这个月的销售情况
         ↓
Agent：理解了，我来帮你查数据
         ↓
自动生成 SQL → 执行 → 分析 → 生成报告 → 可视化
         ↓
返回：销售趋势图 + 关键发现 + 建议
```

### 核心能力

| 能力 | 说明 |
|------|------|
| **自然语言转 SQL** | 用户说需求，Agent 生成查询语句 |
| **数据分析** | 统计分析、趋势分析、异常检测 |
| **报告生成** | 自动生成分析报告 |
| **可视化** | 生成图表建议 |

---

## Part 2：架构设计

### 核心组件

| 组件 | 作用 |
|------|------|
| **NL to SQL** | 将自然语言转为 SQL |
| **Schema Introspector** | 理解数据库结构 |
| **Query Executor** | 执行查询 |
| **Result Analyzer** | 分析查询结果 |
| **Report Generator** | 生成报告 |

### 数据库 Schema 理解

```python
class SchemaIntrospector:
    """数据库结构理解"""

    def __init__(self, db_connection):
        self.db = db_connection

    def get_tables(self) -> List[str]:
        """获取所有表名"""
        query = "SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'"
        return self.db.execute(query)

    def get_columns(self, table_name: str) -> List[Dict]:
        """获取表的列信息"""
        query = f"""
        SELECT column_name, data_type, is_nullable
        FROM information_schema.columns
        WHERE table_name = '{table_name}'
        """
        return self.db.execute(query)

    def build_schema_context(self) -> str:
        """构建 schema 上下文"""
        tables = self.get_tables()
        schema_parts = []

        for table in tables:
            columns = self.get_columns(table)
            cols_str = ", ".join([f"{c['column_name']} ({c['data_type']})" for c in columns])
            schema_parts.append(f"- {table}: {cols_str}")

        return "\n".join(schema_parts)
```

---

## Part 3：自然语言转 SQL

### 核心实现

```python
from langchain_community.chat_models import ChatZhipuAI
import os
from dotenv import load_dotenv

load_dotenv()

llm = ChatZhipuAI(
    api_key=os.getenv("ZHIPU_API_KEY"),
    model="glm-4-flash",
    temperature=0.0
)

class NL2SQL:
    """自然语言转 SQL"""

    def __init__(self, schema_context: str):
        self.schema_context = schema_context

    def generate_sql(self, question: str) -> str:
        """将自然语言转为 SQL"""
        prompt = f"""根据以下数据库结构，将自然语言问题转换为 SQL 查询：

数据库结构：
{self.schema_context}

问题：{question}

要求：
1. 只返回 SQL 语句，不要其他内容
2. SQL 要语法正确
3. 如果问题不明确，做合理假设

SQL：
"""
        response = llm.invoke(prompt).content
        # 提取 SQL（去掉可能的 markdown 格式）
        if "```sql" in response:
            response = response.split("```sql")[1].split("```")[0]
        return response.strip()

    def validate_sql(self, sql: str) -> Dict:
        """验证 SQL 语法"""
        # 基本验证
        issues = []
        if "DROP" in sql.upper() or "DELETE" in sql.upper():
            issues.append("危险操作：包含删除语句")
        if "INSERT" in sql.upper() or "UPDATE" in sql.upper():
            issues.append("写操作：SQL 不是查询语句")
        if not sql.strip().upper().startswith("SELECT"):
            issues.append("语法错误：不是 SELECT 语句")

        return {
            "valid": len(issues) == 0,
            "issues": issues
        }
```

---

## Part 4：数据分析与报告生成

### 结果分析

```python
class ResultAnalyzer:
    """查询结果分析"""

    def __init__(self):
        self.llm = llm

    def analyze(self, sql: str, result: List[Dict]) -> Dict:
        """分析查询结果"""
        if not result:
            return {"type": "empty", "summary": "查询结果为空"}

        # 计算基本统计
        import statistics

        numeric_cols = [k for k, v in result[0].items()
                       if isinstance(v, (int, float))]

        stats = {}
        for col in numeric_cols:
            values = [r[col] for r in result if col in r]
            if values:
                stats[col] = {
                    "count": len(values),
                    "sum": sum(values),
                    "avg": statistics.mean(values),
                    "min": min(values),
                    "max": max(values)
                }

        # 用 LLM 做智能分析
        analysis_prompt = f"""分析以下查询结果：

SQL：{sql}
数据：{result[:5]}

请分析：
1. 数据的主要特征
2. 是否有异常值
3. 有什么值得关注的发现
4. 可以给什么建议

返回 JSON 格式：
{{"findings": ["发现1", "发现2"], "recommendations": ["建议1"]}}
"""
        analysis = self.llm.invoke(analysis_prompt).content

        return {
            "type": "data",
            "count": len(result),
            "stats": stats,
            "analysis": analysis
        }
```

### 报告生成

```python
class ReportGenerator:
    """分析报告生成"""

    def __init__(self):
        self.llm = llm

    def generate(self, question: str, analysis: Dict) -> str:
        """生成分析报告"""
        prompt = f"""根据以下分析结果，生成一份简洁的分析报告：

问题：{question}
分析类型：{analysis['type']}

{self._format_analysis(analysis)}

报告要求：
1. 结构清晰，有标题和小节
2. 数据要可视化描述（不用生成图片）
3. 结论要明确
4. 300 字以内
"""
        return self.llm.invoke(prompt).content

    def _format_analysis(self, analysis: Dict) -> str:
        """格式化分析结果"""
        if analysis["type"] == "empty":
            return "数据为空"
        else:
            lines = [f"数据量：{analysis['count']} 条"]
            if "stats" in analysis:
                lines.append("统计摘要：")
                for col, stat in analysis["stats"].items():
                    lines.append(f"- {col}: 平均 {stat['avg']:.2f}")
            return "\n".join(lines)
```

---

## Part 5：实战——数据分析 Agent

### 完整代码

```python
"""
数据分析 Agent
支持：自然语言查询 → SQL 生成 → 执行 → 分析 → 报告
"""
import os
from dotenv import load_dotenv
from dataclasses import dataclass
from typing import List, Dict, Optional

load_dotenv()

# 模拟数据库连接（实际使用时替换为真实连接）
class MockDB:
    def execute(self, sql: str) -> List[Dict]:
        # 模拟返回数据
        if "sales" in sql.lower():
            return [
                {"date": "2026-01", "product": "A", "amount": 1000, "count": 10},
                {"date": "2026-02", "product": "A", "amount": 1200, "count": 12},
                {"date": "2026-03", "product": "B", "amount": 800, "count": 8},
            ]
        return []

@dataclass
class AnalysisResult:
    question: str
    sql: str
    data: List[Dict]
    analysis: Dict
    report: str

class DataAgent:
    """数据分析 Agent"""

    def __init__(self, db, schema_context: str):
        self.db = db
        self.nl2sql = NL2SQL(schema_context)
        self.analyzer = ResultAnalyzer()
        self.reporter = ReportGenerator()

    def analyze(self, question: str) -> AnalysisResult:
        """执行分析"""
        print(f"📊 分析问题：{question}\n")

        # 1. 生成 SQL
        print("🔧 生成 SQL...")
        sql = self.nl2sql.generate_sql(question)
        print(f"SQL：{sql}")

        # 2. 验证 SQL
        validation = self.nl2sql.validate_sql(sql)
        if not validation["valid"]:
            print(f"⚠️ SQL 警告：{validation['issues']}")

        # 3. 执行查询
        print("\n📡 执行查询...")
        data = self.db.execute(sql)
        print(f"获取 {len(data)} 条数据")

        # 4. 分析结果
        print("\n🧠 分析数据...")
        analysis = self.analyzer.analyze(sql, data)

        # 5. 生成报告
        print("\n📝 生成报告...")
        report = self.reporter.generate(question, analysis)

        return AnalysisResult(
            question=question,
            sql=sql,
            data=data,
            analysis=analysis,
            report=report
        )

# ========== 使用示例 ==========

if __name__ == "__main__":
    # 模拟数据库结构
    schema = """
    - sales: date (date), product (varchar), amount (decimal), count (int)
    - users: id (int), name (varchar), created_at (date)
    - orders: id (int), user_id (int), total (decimal), status (varchar)
    """

    db = MockDB()
    agent = DataAgent(db, schema)

    result = agent.analyze("帮我分析一下最近三个月的销售情况")
    print("\n" + "="*50)
    print("分析报告：")
    print(result.report)
```

---

## 📌 今日要点

- ✅ 理解了数据 Agent 的定义和核心能力
- ✅ 掌握了数据库 Schema 理解方法
- ✅ 学会了自然语言转 SQL 的实现
- ✅ 实现了查询结果分析和报告生成
- ✅ 实战了完整的数据分析 Agent

---

## 自测题

1. 数据 Agent 的核心能力有哪些？
2. NL2SQL 的基本原理是什么？
3. 如何验证生成的 SQL 是否安全？
4. 报告生成需要哪些输入？

---

## 动手作业

1. 实现一个 NL2SQL 模块
2. 给数据分析 Agent 添加图表生成建议
3. 设计一个"销售数据 Dashboard Agent"
