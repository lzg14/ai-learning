# Day 40：综合实战——端到端多 Agent 系统

> 将 39 天所学全部串联起来，构建一个生产级的多 Agent 客服系统。本课是课程的终极实战，涵盖架构设计、代码实现、部署运维全流程。

---

## 课程回顾

在开始之前，回顾我们学过的核心知识：

| 阶段 | 天数 | 核心内容 |
|------|------|---------|
| **基础** | Day01-05 | Prompt Engineering、多轮对话、Function Calling |
| **数据** | Day06-09 | MCP 协议、RAG、知识图谱、多模态 |
| **上下文** | Day10-11 | Context Engineering、Agent 记忆架构 |
| **框架** | Day12-15 | LangChain、Agent 架构、A2A 协议 |
| **推理** | Day16-18 | ReAct、任务规划、长上下文管理 |
| **高级** | Day19-23 | 工作流、多 Agent、代码 Agent、Computer Use |
| **生产** | Day24-30 | 可观测性、部署、微调、安全、评估 |
| **前沿** | Day31-39 | 事件驱动、推理优化、具身智能、自我改进 |
| **实战** | **Day40** | **端到端多 Agent 系统** |

---

## Part 1：项目概述

### 项目：智能客服系统

构建一个**电商智能客服系统**，支持：

```
用户可以通过自然语言：
- 查询商品信息（价格、库存、规格）
- 查询订单状态（物流、退款）
- 获取个性化推荐
- 投诉和建议

系统要求：
- 多 Agent 协作（客服 Agent + 商品 Agent + 订单 Agent）
- RAG 知识库（商品手册、退换货政策）
- 人机协作（复杂问题转人工）
- 生产级可观测性
- 安全防护
```

### 系统架构

```plaintext
                     用户入口
                        │
                        ▼
              ┌──────────────────┐
              │   API 网关       │
              │  (限流 + 鉴权)    │
              └──────┬───────────┘
                     │
                     ▼
              ┌──────────────────┐
              │   Router Agent   │
              │  (意图识别 + 路由) │
              └──────┬───────────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │ 商品Agent│ │ 订单Agent│ │ 客服Agent│
   │ (MCP工具)│ │ (MCP工具)│ │ (RAG知识)│
   └────┬────┘ └────┬────┘ └────┬────┘
        │           │           │
        ▼           ▼           ▼
   ┌─────────────────────────────────────┐
   │         共享基础设施层               │
   │  (LLM推理 + RAG引擎 + 监控 + 日志)   │
   └─────────────────────────────────────┘
```

---

## Part 2：基础设施层

### LLM 推理客户端

```python
"""
infra/llm_client.py
LLM 推理客户端——统一所有模型调用
"""
from openai import OpenAI
from typing import List, Dict, Optional
import os
from dotenv import load_dotenv
import time

load_dotenv()


class LLMClient:
    """统一的 LLM 调用客户端"""

    def __init__(self, model: str = "glm-4-flash"):
        self.model = model
        self.client = OpenAI(
            api_key=os.getenv("ZHIPU_API_KEY"),
            base_url="https://open.bigmodel.cn/api/paas/v4"
        )

    def chat(
        self,
        messages: List[Dict],
        temperature: float = 0.7,
        max_tokens: int = 2000,
    ) -> str:
        """调用 LLM"""
        start = time.time()
        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                temperature=temperature,
                max_tokens=max_tokens,
            )
            result = response.choices[0].message.content
            tokens = response.usage.total_tokens if response.usage else 0
            return {
                "content": result,
                "tokens": tokens,
                "latency": time.time() - start,
                "model": self.model
            }
        except Exception as e:
            return {
                "content": f"LLM 调用失败: {e}",
                "tokens": 0,
                "latency": time.time() - start,
                "model": self.model,
                "error": str(e)
            }
```

### RAG 引擎

```python
"""
infra/rag_engine.py
RAG 引擎——基于向量检索的知识库
"""
from sentence_transformers import SentenceTransformer
import numpy as np
from typing import List, Dict
import json
import os


class RAGEngine:
    """RAG 检索引擎"""

    def __init__(self, model_name: str = "paraphrase-multilingual-MiniLM-L12-v2"):
        # 中文友好的 embedding 模型
        self.encoder = SentenceTransformer(model_name)
        self.documents = []
        self.embeddings = None

    def load_knowledge_base(self, kb_path: str):
        """加载知识库文档"""
        with open(kb_path, "r", encoding="utf-8") as f:
            for line in f:
                doc = json.loads(line)
                self.documents.append(doc)

        # 预计算所有文档的 embedding
        texts = [d["content"] for d in self.documents]
        self.embeddings = self.encoder.encode(texts, show_progress_bar=True)
        print(f"加载 {len(self.documents)} 条知识，向量维度 {self.embeddings.shape[1]}")

    def search(self, query: str, top_k: int = 3) -> List[Dict]:
        """检索相关文档"""
        query_vec = self.encoder.encode([query])
        scores = np.dot(self.embeddings, query_vec.T).flatten()
        top_indices = scores.argsort()[-top_k:][::-1]

        results = []
        for idx in top_indices:
            results.append({
                "content": self.documents[idx]["content"],
                "score": float(scores[idx]),
                "source": self.documents[idx].get("source", ""),
                "category": self.documents[idx].get("category", "")
            })
        return results

    def search_with_context(self, query: str, top_k: int = 3) -> str:
        """检索并格式化为上下文"""
        results = self.search(query, top_k)
        context_parts = []
        for i, r in enumerate(results):
            context_parts.append(
                f"[参考{i+1}] (相关性:{r['score']:.2f})\n{r['content']}"
            )
        return "\n\n".join(context_parts)
```

### 知识库数据准备

```python
"""
data/prepare_kb.py
准备知识库数据（退换货政策、商品信息等）
"""

KB_ENTRIES = [
    {
        "content": "退换货政策：自签收之日起7天内，商品完好且不影响二次销售，可申请无理由退货。部分商品除外（如内衣、食品等）。",
        "source": "return_policy.md",
        "category": "policy"
    },
    {
        "content": "退款时效：退货申请审核通过后，退款将在3-5个工作日内原路返回。",
        "source": "refund_policy.md",
        "category": "policy"
    },
    {
        "content": "物流查询：订单发货后可在「我的订单」页面查看物流信息，或联系在线客服获取物流单号。",
        "source": "shipping.md",
        "category": "shipping"
    },
    {
        "content": "商品保修：电子产品享受1年质保。质保期内非人为损坏可免费维修或更换。",
        "source": "warranty.md",
        "category": "policy"
    },
    {
        "content": "价格保护：购买后7天内商品降价，可申请价格保护，退还差价。",
        "source": "price_protection.md",
        "category": "policy"
    },
    {
        "content": "客服工作时间：在线客服工作时间为每天 9:00-22:00。非工作时间可留言，次日优先处理。",
        "source": "service_hours.md",
        "category": "service"
    },
    {
        "content": "发票开具：订单完成后可申请电子发票，在「我的订单」中找到对应订单点击「申请发票」。",
        "source": "invoice.md",
        "category": "service"
    }
]


def save_knowledge_base(output_path: str = "data/knowledge_base.jsonl"):
    """保存知识库"""
    import json
    os.makedirs(os.path.dirname(output_path), exist_ok=True)
    with open(output_path, "w", encoding="utf-8") as f:
        for entry in KB_ENTRIES:
            f.write(json.dumps(entry, ensure_ascii=False) + "\n")
    print(f"知识库已保存到 {output_path}")
```

---

## Part 3：Agent 层

### Router Agent——入口路由

```python
"""
agents/router_agent.py
路由 Agent——负责意图识别和分发
"""


class RouterAgent:
    """路由 Agent：理解用户意图，分发到对应 Agent"""

    def __init__(self, llm: LLMClient):
        self.llm = llm

    def route(self, user_input: str) -> dict:
        """分析意图并返回路由决策"""
        messages = [
            {"role": "system", "content": """你是一个意图识别专家。分析用户问题，归类到以下类型：

1. product_query: 商品相关（价格、规格、库存、推荐）
2. order_query: 订单相关（物流、退款、售后）
3. policy_query: 政策咨询（退换货规则、保修、发票）
4. complaint: 投诉
5. chitchat: 闲聊/问候
6. transfer_human: 需要转人工

只返回 JSON：
{"intent": "类型", "confidence": 0.0-1.0, "entities": {}, "brief": "一句话摘要"}
"""},
            {"role": "user", "content": user_input}
        ]

        result = self.llm.chat(messages, temperature=0.1)
        import json
        try:
            decision = json.loads(result["content"])
            return decision
        except:
            return {"intent": "transfer_human", "confidence": 0.0,
                    "entities": {}, "brief": "意图识别失败"}

    def dispatch(self, user_input: str, context: dict) -> str:
        """路由并分发"""
        decision = self.route(user_input)
        intent = decision["intent"]

        # 路由到对应 Agent
        if intent == "product_query":
            return "product_agent"
        elif intent == "order_query":
            return "order_agent"
        elif intent == "policy_query":
            return "service_agent"
        elif intent == "complaint":
            return "service_agent"
        elif intent == "transfer_human":
            return "human"
        else:
            return "service_agent"
```

### Product Agent——商品助手

```python
"""
agents/product_agent.py
商品 Agent——查询商品信息、推荐
"""
from typing import List, Dict


class MockProductDB:
    """模拟商品数据库"""

    def __init__(self):
        self.products = {
            "P001": {"name": "无线蓝牙耳机 Pro", "price": 299, "stock": 150,
                     "spec": "蓝牙5.3, 续航30小时, IPX5防水",
                     "category": "电子产品"},
            "P002": {"name": "智能手环 S2", "price": 199, "stock": 80,
                     "spec": "心率监测, 血氧检测, 7天续航",
                     "category": "电子产品"},
            "P003": {"name": "便携充电宝 20000mAh", "price": 129, "stock": 200,
                     "spec": "20000mAh, 快充, 可带上飞机",
                     "category": "配件"},
            "P004": {"name": "手机壳 硅胶防摔", "price": 39, "stock": 500,
                     "spec": "液态硅胶, 防摔, 多种颜色",
                     "category": "配件"},
            "P005": {"name": "机械键盘 K1", "price": 399, "stock": 60,
                     "spec": "青轴, RGB背光, 87键紧凑布局",
                     "category": "电子产品"},
        }

    def search_by_name(self, keyword: str) -> List[Dict]:
        """按名称搜索"""
        results = []
        for pid, info in self.products.items():
            if keyword.lower() in info["name"].lower():
                results.append({"id": pid, **info})
        return results

    def get_by_id(self, pid: str) -> Dict:
        """按 ID 查询"""
        return self.products.get(pid)

    def get_by_category(self, category: str) -> List[Dict]:
        """按分类查询"""
        return [
            {"id": pid, **info}
            for pid, info in self.products.items()
            if info["category"] == category
        ]

    def get_all(self) -> List[Dict]:
        """获取所有商品"""
        return [{"id": pid, **info} for pid, info in self.products.items()]


class ProductAgent:
    """商品 Agent"""

    def __init__(self, llm: LLMClient, db: MockProductDB):
        self.llm = llm
        self.db = db

    def handle(self, user_input: str) -> str:
        """处理商品查询"""
        # 1. 用 LLM 提取查询意图
        intent_messages = [
            {"role": "system", "content": """从用户问题中提取商品查询意图。
返回 JSON：
{"action": "search/list/detail", "keyword": "", "category": ""}
"""},
            {"role": "user", "content": user_input}
        ]
        intent_result = self.llm.chat(intent_messages, temperature=0.1)

        import json
        try:
            intent = json.loads(intent_result["content"])
        except:
            intent = {"action": "search", "keyword": ""}

        # 2. 查询数据库
        if intent["action"] == "search" and intent.get("keyword"):
            products = self.db.search_by_name(intent["keyword"])
        elif intent["action"] == "list" and intent.get("category"):
            products = self.db.get_by_category(intent["category"])
        else:
            products = self.db.get_all()

        # 3. 生成回复
        if not products:
            context = "没有找到相关商品。"
        else:
            context = "找到以下商品：\n" + "\n".join(
                [f"- {p['name']} ¥{p['price']} (库存:{p['stock']})" for p in products]
            )

        reply_messages = [
            {"role": "system", "content": "你是电商商品助手。基于查询结果回答用户问题，语气亲切专业。"},
            {"role": "user", "content": f"用户问题：{user_input}\n\n查询结果：{context}"}
        ]

        result = self.llm.chat(reply_messages)
        return result["content"]
```

### Order Agent——订单助手

```python
"""
agents/order_agent.py
订单 Agent——查询订单、处理退款
"""


class MockOrderDB:
    """模拟订单数据库"""

    def __init__(self):
        self.orders = {
            "ORD001": {
                "user": "张三",
                "product": "无线蓝牙耳机 Pro",
                "price": 299,
                "status": "已签收",
                "date": "2026-04-28",
                "tracking": "SF1234567890",
                "logistics": [
                    {"time": "2026-04-28 10:00", "event": "包裹已签收"},
                    {"time": "2026-04-27 15:30", "event": "到达派送点"},
                    {"time": "2026-04-26 20:00", "event": "已发货"},
                ]
            },
            "ORD002": {
                "user": "张三",
                "product": "智能手环 S2",
                "price": 199,
                "status": "配送中",
                "date": "2026-04-30",
                "tracking": "SF1234567891",
                "logistics": [
                    {"time": "2026-05-01 08:00", "event": "到达中转站"},
                    {"time": "2026-04-30 18:00", "event": "已发货"},
                ]
            },
            "ORD003": {
                "user": "李四",
                "product": "便携充电宝",
                "price": 129,
                "status": "退款中",
                "date": "2026-04-25",
                "tracking": "",
                "logistics": []
            }
        }

    def get_by_user(self, user: str) -> List[Dict]:
        return [{"id": oid, **info} for oid, info in self.orders.items()
                if info["user"] == user]

    def get_by_id(self, order_id: str) -> Dict:
        order = self.orders.get(order_id)
        return {"id": order_id, **order} if order else None


class OrderAgent:
    """订单 Agent"""

    def __init__(self, llm: LLMClient, db: MockOrderDB):
        self.llm = llm
        self.db = db

    def handle(self, user_input: str, user_name: str = "张三") -> str:
        """处理订单查询"""
        # 提取用户可能需要查的订单
        orders = self.db.get_by_user(user_name)
        order_context = "你的订单：\n" + "\n".join(
            [f"- {o['id']}: {o['product']} ({o['status']})" for o in orders]
        )

        messages = [
            {"role": "system", "content": "你是订单助手。基于订单数据回答用户问题。"},
            {"role": "user", "content": f"用户：{user_name}\n用户问题：{user_input}\n\n{order_context}"}
        ]

        result = self.llm.chat(messages)
        return result["content"]
```

### Service Agent——客服助手（带 RAG）

```python
"""
agents/service_agent.py
客服 Agent——处理退货、投诉、政策咨询等
"""


class ServiceAgent:
    """客服 Agent（带 RAG）"""

    def __init__(self, llm: LLMClient, rag: RAGEngine):
        self.llm = llm
        self.rag = rag

    def handle(self, user_input: str) -> str:
        """处理客服请求"""
        # 1. 从知识库检索相关信息
        kb_context = self.rag.search_with_context(user_input, top_k=3)

        # 2. 判断是否需要转人工
        judgment_messages = [
            {"role": "system", "content": """判断用户问题是否需要转人工客服。
需要转人工的情况：辱骂、威胁、复杂投诉、要求转人工、涉及法律问题。

返回 JSON：
{"transfer": true/false, "reason": ""}
"""},
            {"role": "user", "content": user_input}
        ]
        judgment = self.llm.chat(judgment_messages, temperature=0.1)

        import json
        try:
            decision = json.loads(judgment["content"])
            if decision.get("transfer"):
                return f"已为您转接人工客服。原因：{decision.get('reason', '')}"
        except:
            pass

        # 3. 基于知识库回答
        messages = [
            {"role": "system", "content": f"""你是电商客服助手。基于以下知识库信息回答用户问题。
如果知识库中没有相关信息，请如实告知，不要编造。

知识库：
{kb_context}"""},
            {"role": "user", "content": user_input}
        ]

        result = self.llm.chat(messages)
        return result["content"]
```

---

## Part 4：编排层

### 主 Agent 编排器

```python
"""
orchestrator/main_agent.py
主 Agent——统一调度所有子 Agent
"""


class CustomerServiceOrchestrator:
    """客服系统主编排器"""

    def __init__(self):
        # 基础设施
        self.llm = LLMClient()
        self.rag = RAGEngine()

        # 加载知识库
        kb_path = "data/knowledge_base.jsonl"
        if os.path.exists(kb_path):
            self.rag.load_knowledge_base(kb_path)
        else:
            print(f"知识库文件 {kb_path} 不存在，请先运行 prepare_kb.py")

        # 子 Agent
        self.product_db = MockProductDB()
        self.order_db = MockOrderDB()

        self.router = RouterAgent(self.llm)
        self.product_agent = ProductAgent(self.llm, self.product_db)
        self.order_agent = OrderAgent(self.llm, self.order_db)
        self.service_agent = ServiceAgent(self.llm, self.rag)

        # 监控
        self.monitor = AgentMonitor()

    def process(self, user_input: str, user_name: str = "张三",
                 session_id: str = None) -> dict:
        """处理用户请求"""
        start = time.time()
        trace_id = f"trace-{int(start)}"

        try:
            # 1. 路由
            route_start = time.time()
            target_agent = self.router.dispatch(user_input, {})
            route_latency = time.time() - route_start

            # 2. 分发到对应 Agent
            agent_start = time.time()
            if target_agent == "product_agent":
                reply = self.product_agent.handle(user_input)
            elif target_agent == "order_agent":
                reply = self.order_agent.handle(user_input, user_name)
            elif target_agent == "human":
                reply = "已为您转接人工客服，请稍候。"
            else:
                reply = self.service_agent.handle(user_input)
            agent_latency = time.time() - agent_start

            total_latency = time.time() - start

            # 记录监控数据
            self.monitor.record_request(
                duration=total_latency,
                tokens=0,  # 实际应汇总所有 LLM 调用的 tokens
                success=True
            )

            return {
                "reply": reply,
                "trace_id": trace_id,
                "metrics": {
                    "total_latency": round(total_latency, 2),
                    "route_latency": round(route_latency, 2),
                    "agent_latency": round(agent_latency, 2),
                    "target_agent": target_agent
                }
            }

        except Exception as e:
            self.monitor.record_error("internal_error", str(e))
            return {
                "reply": f"系统处理出错，请稍后重试。错误：{str(e)}",
                "trace_id": trace_id,
                "error": str(e),
                "metrics": {"total_latency": round(time.time() - start, 2)}
            }
```

### 对话管理

```python
"""
orchestrator/conversation.py
对话管理——维护多轮对话上下文
"""


class ConversationManager:
    """对话管理器"""

    def __init__(self, max_history: int = 10):
        self.sessions = {}  # session_id -> messages
        self.max_history = max_history

    def get_history(self, session_id: str) -> list:
        """获取对话历史"""
        return self.sessions.get(session_id, [])

    def add_message(self, session_id: str, role: str, content: str):
        """添加消息"""
        if session_id not in self.sessions:
            self.sessions[session_id] = []
        self.sessions[session_id].append({
            "role": role,
            "content": content,
            "timestamp": time.time()
        })
        # 限制历史长度
        if len(self.sessions[session_id]) > self.max_history * 2:
            # 保留系统消息和最近的消息
            self.sessions[session_id] = self.sessions[session_id][-self.max_history * 2:]

    def clear(self, session_id: str):
        """清空对话历史"""
        self.sessions.pop(session_id, None)


class SessionManager:
    """会话管理——隔离不同用户上下文"""

    def __init__(self):
        self.conversations = ConversationManager()
        self.user_info = {}  # session_id -> user info

    def get_or_create(self, session_id: str) -> dict:
        """获取或创建会话"""
        if session_id not in self.user_info:
            self.user_info[session_id] = {
                "user_name": f"用户{len(self.user_info) + 1}",
                "created_at": time.time()
            }
        return {
            "session_id": session_id,
            "user": self.user_info[session_id],
            "history": self.conversations.get_history(session_id)
        }
```

---

## Part 5：API 层

```python
"""
api/server.py
FastAPI 服务器——提供 REST API
"""
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import uvicorn

# 导入项目模块
import sys
sys.path.append("..")

from orchestrator.main_agent import CustomerServiceOrchestrator
from orchestrator.conversation import SessionManager

app = FastAPI(title="智能客服系统", version="2.0.0")

# 全局实例
orchestrator = CustomerServiceOrchestrator()
session_manager = SessionManager()


class ChatRequest(BaseModel):
    """聊天请求"""
    message: str
    session_id: Optional[str] = None
    user_name: Optional[str] = None


class ChatResponse(BaseModel):
    """聊天响应"""
    reply: str
    session_id: str
    trace_id: str
    metrics: dict


@app.post("/api/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    """聊天接口"""
    # 会话管理
    session_id = request.session_id or f"session-{id(request)}"
    session = session_manager.get_or_create(session_id)

    if request.user_name:
        session["user"]["user_name"] = request.user_name

    user_name = session["user"]["user_name"]

    # 处理请求
    result = orchestrator.process(
        user_input=request.message,
        user_name=user_name,
        session_id=session_id
    )

    # 记录对话历史
    session_manager.conversations.add_message(session_id, "user", request.message)
    session_manager.conversations.add_message(session_id, "assistant", result["reply"])

    return ChatResponse(
        reply=result["reply"],
        session_id=session_id,
        trace_id=result["trace_id"],
        metrics=result["metrics"]
    )


@app.get("/api/sessions")
async def list_sessions():
    """列出所有活跃会话"""
    return {
        "sessions": list(session_manager.user_info.keys()),
        "count": len(session_manager.user_info)
    }


@app.delete("/api/sessions/{session_id}")
async def clear_session(session_id: str):
    """清除会话"""
    session_manager.conversations.clear(session_id)
    session_manager.user_info.pop(session_id, None)
    return {"status": "cleared", "session_id": session_id}


@app.get("/health")
async def health():
    """健康检查"""
    return {
        "status": "healthy",
        "agents": ["product", "order", "service"],
        "rag_loaded": orchestrator.rag.embeddings is not None
    }


@app.get("/metrics")
async def metrics():
    """监控指标"""
    health_data = orchestrator.monitor.health_check()
    return health_data


if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## Part 6：测试与运行

### 单元测试

```python
"""
tests/test_agents.py
单元测试
"""
import pytest
from agents.router_agent import RouterAgent
from agents.product_agent import ProductAgent, MockProductDB
from infra.llm_client import LLMClient


class TestRouterAgent:
    """测试路由 Agent"""

    def setup_method(self):
        self.llm = LLMClient()
        self.router = RouterAgent(self.llm)

    def test_product_intent(self):
        """测试商品查询意图识别"""
        result = self.router.route("这个耳机多少钱？")
        assert result["intent"] == "product_query"
        assert result["confidence"] > 0.5

    def test_order_intent(self):
        """测试订单查询意图识别"""
        result = self.router.route("我的订单到哪了？")
        assert result["intent"] == "order_query"

    def test_greeting(self):
        """测试问候"""
        result = self.router.route("你好")
        assert result["intent"] == "chitchat"


class TestProductAgent:
    """测试商品 Agent"""

    def setup_method(self):
        self.db = MockProductDB()

    def test_search_by_name(self):
        results = self.db.search_by_name("耳机")
        assert len(results) > 0
        assert "蓝牙" in results[0]["name"]

    def test_search_not_found(self):
        results = self.db.search_by_name("不存在商品")
        assert len(results) == 0
```

### 集成测试

```python
"""
tests/test_integration.py
集成测试——模拟完整用户会话
"""


def simulate_conversation():
    """模拟完整对话流程"""
    from api.server import orchestrator, session_manager

    test_cases = [
        ("你好", "service_agent"),
        ("无线耳机多少钱？", "product_agent"),
        ("我的订单到哪里了？", "order_agent"),
        ("怎么退货？", "service_agent"),
        ("我想投诉", "service_agent"),
    ]

    session_id = "test-session-001"

    for question, expected_agent in test_cases:
        result = orchestrator.process(question, "张三", session_id)
        print(f"\nQ: {question}")
        print(f"Agent: {result['metrics']['target_agent']} (期望: {expected_agent})")
        print(f"A: {result['reply'][:50]}...")

        # 记录历史
        session_manager.conversations.add_message(session_id, "user", question)
        session_manager.conversations.add_message(session_id, "assistant", result["reply"])

    print(f"\n对话轮次: {len(test_cases)}")
    print(f"历史长度: {len(session_manager.conversations.get_history(session_id))}")


def run_stress_test():
    """简单压力测试"""
    import time
    from orchestrator.main_agent import CustomerServiceOrchestrator

    orchestrator = CustomerServiceOrchestrator()
    questions = [
        "耳机多少钱？", "订单到哪了？", "怎么退货？",
        "充电宝有货吗？", "我要投诉"
    ] * 10  # 50 个请求

    print(f"开始压力测试 ({len(questions)} 请求)...")
    start = time.time()

    for q in questions:
        orchestrator.process(q)

    total = time.time() - start
    print(f"完成! 耗时: {total:.2f}s")
    print(f"平均: {total/len(questions):.3f}s/请求")
    print(f"QPS: {len(questions)/total:.1f}")


if __name__ == "__main__":
    print("=== 集成测试 ===")
    simulate_conversation()
    print("\n=== 压力测试 ===")
    run_stress_test()
```

### 启动脚本

```python
"""
run.py
项目启动入口
"""
import os
import sys


def setup():
    """初始化项目"""
    print("=" * 50)
    print("智能客服系统 v2.0")
    print("=" * 50)

    # 1. 准备知识库
    print("\n[1/3] 准备知识库...")
    from data.prepare_kb import save_knowledge_base
    save_knowledge_base()

    # 2. 检查环境
    print("\n[2/3] 检查环境配置...")
    from dotenv import load_dotenv
    load_dotenv()
    assert os.getenv("ZHIPU_API_KEY"), "请设置 ZHIPU_API_KEY 环境变量"
    print("✅ 环境配置完成")

    # 3. 启动服务
    print("\n[3/3] 启动 API 服务...")
    print("服务地址: http://localhost:8000")
    print("API 文档: http://localhost:8000/docs")
    print("健康检查: http://localhost:8000/health")
    print("\n按 Ctrl+C 停止服务\n")

    import uvicorn
    uvicorn.run("api.server:app", host="0.0.0.0", port=8000, reload=True)


if __name__ == "__main__":
    setup()
```

---

## Part 7：项目总结构

### 目录结构

```plaintext
customer-service-agent/
├── run.py                    # 项目入口
├── requirements.txt          # 依赖
├── .env                      # 环境变量
│
├── infra/                    # 基础设施层
│   ├── __init__.py
│   ├── llm_client.py         # LLM 调用客户端
│   └── rag_engine.py         # RAG 检索引擎
│
├── agents/                   # Agent 层
│   ├── __init__.py
│   ├── router_agent.py       # 路由 Agent
│   ├── product_agent.py      # 商品 Agent
│   ├── order_agent.py        # 订单 Agent
│   └── service_agent.py      # 客服 Agent
│
├── orchestrator/             # 编排层
│   ├── __init__.py
│   ├── main_agent.py         # 主编排器
│   └── conversation.py       # 对话管理
│
├── api/                      # API 层
│   ├── __init__.py
│   └── server.py             # FastAPI 服务
│
├── data/                     # 数据
│   ├── prepare_kb.py         # 知识库准备
│   └── knowledge_base.jsonl  # 知识库文件
│
├── tests/                    # 测试
│   ├── test_agents.py
│   └── test_integration.py
│
└── deploy/                   # 部署配置
    ├── Dockerfile
    ├── docker-compose.yml
    └── k8s-deployment.yaml
```

### 扩展方向

```plaintext
项目已经是一个完整的多 Agent 系统，可以从以下方向继续扩展：

1. 更多 Agent
   └── 支付 Agent、物流 Agent、推荐 Agent、售后 Agent

2. 更智能的编排
   └── 动态任务规划、多 Agent 并行处理、A2A 协议互连

3. 记忆系统
   └── Day11 的四层记忆架构（短期/长期/工作/ episodic）

4. 可观测性
   └── Langfuse 追踪、A/B 测试、自动评估流水线

5. 安全加固
   └── Prompt 注入防护、输出审核、权限控制

6. 部署
   └── Docker 容器化、K8s 编排、自动扩缩容
```

---

## 课程总结

### 40 天学习之旅

```plaintext
第 1 周：AI 基础
  你学会了：Prompt 怎么写、多轮对话怎么维护、Function Calling 怎么用

第 2 周：数据与上下文
  你学会了：MCP 协议、RAG、知识图谱、Context Engineering

第 3 周：框架与架构
  你学会了：LangChain、Agent 设计模式、A2A 协议

第 4 周：推理与规划
  你学会了：ReAct 推理、任务规划、长上下文管理

第 5 周：多 Agent 与工具
  你学会了：多 Agent 协作、代码 Agent、工作流编排

第 6 周：生产化
  你学会了：模型部署、量化、微调、可观测性、安全

第 7 周：前沿
  你学会了：事件驱动、推理优化、具身智能、Agent 自我改进

第 8 周：实战
  你构建了一个完整的智能客服系统！
```

### 核心能力清单

完成本课程后，你应该具备以下能力：

```
□ 能用 Prompt Engineering 解决 80% 的日常问题
□ 能设计和实现单 Agent 系统（RAG + 工具调用）
□ 能理解 MCP 和 A2A 协议的设计思想
□ 能构建多 Agent 协作系统
□ 能部署和优化 LLM 推理服务
□ 能准备数据并进行 LoRA 微调
□ 能设计 Agent 安全防护体系
□ 能搭建生产级可观测性
□ 能持续跟进 AI Agent 前沿技术
```

### 持续学习建议

| 方向 | 推荐资源 |
|------|---------|
| **跟踪前沿** | 关注 MCP/A2A 协议更新、Linux Foundation Agentic AI Foundation |
| **动手实践** | 用学到的技术解决实际问题，从简单到复杂 |
| **深入原理** | 阅读 LLM 论文（Transformer、RLHF、稀疏注意力） |
| **参与社区** | GitHub、知乎、技术博客，分享你的 Agent 项目 |
| **国产生态** | 关注智谱 GLM、DeepSeek、Qwen 的更新和社区 |

---

## 今日要点

- 构建了完整的多 Agent 客服系统（架构设计 + 代码实现）
- 实现了 Router Agent → Product/Order/Service Agent 的路由分发
- 集成了 RAG 知识库（退换货政策、商品信息）
- 实现了生产级 API 服务（FastAPI + 监控 + 健康检查）
- 编写了单元测试和集成测试
- 理解了从开发到部署的完整流程

---

## 自测题

1. 多 Agent 系统中的 Router Agent 起什么作用？
2. RAG 在客服系统中解决的问题是什么？
3. 为什么需要对话管理？Session 隔离的重要性是什么？
4. 生产级 API 服务需要包含哪些关键要素？
5. 如果要在本系统基础上增加"物流查询"Agent，需要修改哪些部分？

---

## 动手作业

1. 运行本项目的完整代码，测试各个 Agent 的功能
2. 添加一个新的"推荐 Agent"，根据用户历史推荐商品
3. 为系统接入 Langfuse 可观测性
4. （进阶）用 Docker Compose 将整个系统容器化部署