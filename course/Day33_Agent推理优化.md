# Day 33：Agent 推理优化

> 让 Agent 思考得更快、更好、更准。本课深入推理加速、批量处理、成本优化、国产优化方案。

## Part 1：为什么需要推理优化？

### 推理优化的核心问题

**"做饭"类比：**
- 慢的厨师：洗菜切菜要 1 小时，做一道菜要 3 小时
- 快的厨师：同样的菜，30 分钟搞定

**推理优化的目标：**
- 同样的输出质量，时间更短
- 同样的时间，质量更高
- 同样的效果，成本更低

---

### 推理优化的核心指标

| 指标 | 说明 | 目标 |
| --- | --- | --- |
| **延迟（Latency）** | 单个请求的响应时间 | 越短越好 |
| **吞吐量（Throughput）** | 每秒能处理的请求数 | 越高越好 |
| **成本（Cost）** | 每 1000 token 的费用 | 越低越好 |
| **准确率（Accuracy）** | 回答的正确率 | 越高越好 |

**优化三角：**

```plaintext
         速度
        /    \
       /      \
      /   质量  \
     /__________\
       成本
```

不可能三者同时最优，只能权衡取舍。

---

### 常见瓶颈分析

| 瓶颈 | 表现 | 原因 |
| --- | --- | --- |
| **计算瓶颈** | GPU 利用率低 | 模型太大、Batch Size 太小 |
| **内存瓶颈** | 显存不够 | 模型加载太多、Context 太长 |
| **IO 瓶颈** | 等待时间长 | 网络延迟、数据库查询慢 |
| **API 瓶颈** | 排队等待 | 上游服务响应慢 |

---

## Part 2：推理加速技术

### 技术一：批量处理（Batching）

**核心思想：** 把多个请求打包在一起处理，共享计算。

```python
# 串行处理：10 个请求 = 10 × 100ms = 1 秒
for request in requests:
    result = model.predict(request)

# 批量处理：10 个请求 = 1 × 100ms + 开销 = 200ms
batch = [r.text for r in requests]
results = model.batch_predict(batch)
```

**动态批处理：**

```python
class DynamicBatcher:
    def __init__(self, max_batch_size=32, max_wait_ms=50):
        self.queue = []
        self.max_batch_size = max_batch_size
        self.max_wait_ms = max_wait_ms

    async def add(self, request):
        self.queue.append(request)
        if len(self.queue) >= self.max_batch_size:
            return await self._process_batch()
        elif self._waited_too_long():
            return await self._process_batch()
        return None

    async def _process_batch(self):
        if not self.queue:
            return None
        batch = self.queue[:self.max_batch_size]
        self.queue = self.queue[self.max_batch_size:]
        return await model.batch_predict([r.text for r in batch])
```

---

### 技术二：缓存（Caching）

**缓存类型：**

| 类型 | 命中率 | 适用场景 |
| --- | --- | --- |
| **精确匹配缓存** | 高 | 完全相同的问题 |
| **语义相似缓存** | 中 | 相似的问题 |
| **Token 缓存** | 高 | 相同的 Context 前缀 |

**多级缓存架构：**

```plaintext
用户请求
    ↓
L1 缓存（精确匹配）→ 命中直接返回
    ↓ 未命中
L2 缓存（语义相似）→ 命中返回相似结果
    ↓ 未命中
L3 缓存（向量索引）→ 命中返回参考
    ↓ 未命中
模型推理
```

---

### 技术三：投机解码（Speculative Decoding）

**核心思想：** 用小模型猜，大模型验证。

```plaintext
1. 小模型（7B）快速生成 10 个 token
2. 大模型（70B）并行验证这 10 个 token
3. 正确的直接接受，错误的重新生成
4. 速度提升 2-3 倍，质量不变
```

**效果：**
- 延迟降低 50-70%
- 吞吐量提升 2-3 倍
- 成本降低 40-60%

---

### 技术四：上下文压缩

**核心思想：** 压缩 Context，减少计算量。

| 方法 | 压缩率 | 效果 |
| --- | --- | --- |
| **摘要压缩** | 30-50% | 把长对话压缩成摘要 |
| **关键词提取** | 40-60% | 只保留关键信息 |
| **重要性排序** | 50-70% | 按重要性筛选内容 |

---

## Part 3：成本优化策略

### 成本构成分析

| 成本项 | 占比 | 优化空间 |
| --- | --- | --- |
| **Token 费用** | 60-70% | 选更便宜的模型、减少 Token 数 |
| **API 调用费** | 20-30% | 批量处理、缓存 |
| **计算资源费** | 10-20% | 模型量化、GPU 优化 |

---

### 智能路由（Smart Routing）

**核心思想：** 把请求路由到最合适的模型。

```python
class SmartRouter:
    def __init__(self):
        self.models = {
            "glm-4-flash": {"cost": 0.1, "quality": 0.7, "speed": "fast"},
            "glm-4": {"cost": 0.5, "quality": 0.9, "speed": "slow"},
            "deepseek-v3": {"cost": 0.3, "quality": 0.85, "speed": "medium"},
        }

    async def route(self, request):
        # 简单问题 → 便宜快速的模型
        if self._is_simple(request):
            return self.models["glm-4-flash"]
        # 复杂问题 → 高质量模型
        elif self._is_complex(request):
            return self.models["glm-4"]
        # 中等 → 平衡模型
        else:
            return self.models["deepseek-v3"]

    def _is_simple(self, request):
        """判断是否简单问题"""
        simple_keywords = ["天气", "时间", "计算", "定义"]
        return any(kw in request for kw in simple_keywords)
```

---

### Prompt 优化

**减少 Token 的技巧：**

```plaintext
❌ 不好的写法：
"请帮我写一段 Python 代码，实现以下功能：
1. 接收用户输入
2. 进行数据处理
3. 返回结果
请确保代码是高质量的，包含错误处理和注释。"

✅ 好的写法：
"Python 接收输入 → 数据处理 → 返回结果，添加错误处理和注释"
```

**效果：** Token 减少 30-50%，费用同步降低。

---

## Part 4：实战——构建优化版 Agent

### 完整代码：推理优化版 Agent

```python
"""
推理优化版 Agent
包含：批量处理、多级缓存、智能路由、Token 压缩
"""
import asyncio
import hashlib
import time
from typing import Dict, List, Any, Optional, Tuple
from dataclasses import dataclass
from collections import defaultdict
import random

@dataclass
class Request:
    """用户请求"""
    id: str
    text: str
    timestamp: float
    metadata: Dict = None

@dataclass
class Response:
    """响应"""
    request_id: str
    text: str
    model_used: str
    latency_ms: float
    tokens_used: int
    cached: bool = False

class SemanticCache:
    """语义缓存"""

    def __init__(self):
        self.exact_cache: Dict[str, str] = {}
        self.semantic_cache: Dict[str, Tuple[str, float]] = {}

    def get_exact(self, text: str) -> Optional[str]:
        """精确匹配"""
        key = self._hash(text)
        return self.exact_cache.get(key)

    def set_exact(self, text: str, response: str):
        """设置精确缓存"""
        key = self._hash(text)
        self.exact_cache[key] = response

    def get_semantic(self, text: str, threshold: float = 0.95) -> Optional[str]:
        """语义相似匹配"""
        for key, (response, score) in self.semantic_cache.items():
            if score >= threshold:
                return response
        return None

    def set_semantic(self, text: str, response: str, similarity: float):
        """设置语义缓存"""
        key = self._hash(text)
        self.semantic_cache[key] = (response, similarity)

    def _hash(self, text: str) -> str:
        return hashlib.md5(text.encode()).hexdigest()

class DynamicBatcher:
    """动态批处理器"""

    def __init__(self, max_batch_size: int = 8, max_wait_ms: int = 100):
        self.queue: List[Request] = []
        self.max_batch_size = max_batch_size
        self.max_wait_ms = max_wait_ms
        self.last_batch_time = time.time()
        self.processing = False

    async def add(self, request: Request) -> Optional[List[Request]]:
        """添加请求到批次"""
        self.queue.append(request)

        if len(self.queue) >= self.max_batch_size:
            return await self._process_batch()

        if time.time() - self.last_batch_time > self.max_wait_ms / 1000:
            return await self._process_batch()

        return None

    async def _process_batch(self) -> List[Request]:
        """处理批次"""
        if not self.queue:
            return []

        batch = self.queue[:self.max_batch_size]
        self.queue = self.queue[self.max_batch_size:]
        self.last_batch_time = time.time()
        return batch

class SmartRouter:
    """智能路由"""

    def __init__(self):
        self.models = {
            "glm-4-flash": {"cost": 0.1, "quality": 0.7, "speed": "fast"},
            "glm-4": {"cost": 0.5, "quality": 0.9, "speed": "slow"},
            "deepseek-v3": {"cost": 0.3, "quality": 0.85, "speed": "medium"},
        }

    async def route(self, request: Request) -> str:
        """路由到合适的模型"""
        text = request.text.lower()

        # 简单问题 → 便宜快速的模型
        simple_patterns = ["天气", "时间", "计算", "定义", "是什么", "多少"]
        if any(p in text for p in simple_patterns):
            if len(text) < 50:
                return "glm-4-flash"

        # 复杂问题 → 高质量模型
        complex_patterns = ["分析", "比较", "解释", "代码", "实现"]
        if any(p in text for p in complex_patterns):
            return "deepseek-v3"

        # 默认 → 平衡模型
        return "glm-4-flash"

    def estimate_cost(self, model: str, tokens: int) -> float:
        """估算成本"""
        return self.models[model]["cost"] * tokens / 1000

class ContextCompressor:
    """上下文压缩器"""

    def compress(self, messages: List[Dict]) -> List[Dict]:
        """压缩对话历史"""
        if len(messages) <= 5:
            return messages

        # 保留最新的 3 条和系统提示
        system = [m for m in messages if m.get("role") == "system"]
        recent = messages[-3:] if messages else []
        middle = messages[1:-3] if len(messages) > 3 else []

        # 压缩中间部分
        compressed_middle = self._compress_history(middle)

        return system + compressed_middle + recent

    def _compress_history(self, messages: List[Dict]) -> List[Dict]:
        """压缩历史消息"""
        if not messages:
            return []

        # 简单策略：保留第一条和最后一条，中间用摘要代替
        if len(messages) > 2:
            summary = {
                "role": "system",
                "content": f"[{len(messages)} 条对话已压缩为摘要]"
            }
            return [messages[0], summary, messages[-1]]

        return messages

class OptimizedAgent:
    """优化版 Agent"""

    def __init__(self):
        self.cache = SemanticCache()
        self.batcher = DynamicBatcher(max_batch_size=8, max_wait_ms=100)
        self.router = SmartRouter()
        self.compressor = ContextCompressor()

        # 统计
        self.stats = {
            "total_requests": 0,
            "cache_hits": 0,
            "tokens_saved": 0,
        }

    async def chat(self, request: Request) -> Response:
        """处理聊天请求"""
        start_time = time.time()
        self.stats["total_requests"] += 1

        # 1. 缓存检查
        cached_response = self.cache.get_exact(request.text)
        if cached_response:
            self.stats["cache_hits"] += 1
            return Response(
                request_id=request.id,
                text=cached_response,
                model_used="cache",
                latency_ms=(time.time() - start_time) * 1000,
                tokens_used=0,
                cached=True
            )

        # 2. 智能路由
        model = await self.router.route(request)

        # 3. 模拟模型调用
        await asyncio.sleep(0.1)  # 模拟延迟

        response_text = f"[{model}] 这是针对 '{request.text}' 的回答"

        # 4. 缓存结果
        self.cache.set_exact(request.text, response_text)

        latency = (time.time() - start_time) * 1000
        tokens = len(request.text) + len(response_text)

        return Response(
            request_id=request.id,
            text=response_text,
            model_used=model,
            latency_ms=latency,
            tokens_used=tokens,
            cached=False
        )

    async def batch_chat(self, requests: List[Request]) -> List[Response]:
        """批量处理请求"""
        batch = await self.batcher.add(requests[0])
        if batch:
            return await asyncio.gather(*[self.chat(r) for r in batch])
        return []

    def print_stats(self):
        """打印统计"""
        cache_hit_rate = self.stats["cache_hits"] / max(1, self.stats["total_requests"]) * 100
        print(f"\n📊 优化统计：")
        print(f"总请求数：{self.stats['total_requests']}")
        print(f"缓存命中：{self.stats['cache_hits']} ({cache_hit_rate:.1f}%)")
        print(f"Token 节省：{self.stats['tokens_saved']}")

async def main():
    print("=" * 70)
    print("🚀 推理优化版 Agent 测试")
    print("=" * 70)

    agent = OptimizedAgent()

    # 测试请求
    test_requests = [
        Request("1", "北京今天天气怎么样？", time.time()),
        Request("2", "北京今天天气怎么样？", time.time()),  # 重复请求
        Request("3", "解释一下机器学习", time.time()),
        Request("4", "什么是 Python？", time.time()),
        Request("5", "帮我写一段快速排序代码", time.time()),
    ]

    print("\n📨 处理请求：")
    responses = []
    for req in test_requests:
        resp = await agent.chat(req)
        responses.append(resp)
        cache_tag = " [缓存]" if resp.cached else ""
        print(f"  {req.id}. {req.text[:30]}... → {resp.model_used}{cache_tag} ({resp.latency_ms:.0f}ms)")

    agent.print_stats()

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 📌 今日要点

- ✅ 理解了推理优化的核心指标和常见瓶颈
- ✅ 掌握了四大推理加速技术：批量处理、缓存、投机解码、上下文压缩
- ✅ 学会了成本优化策略：智能路由、Prompt 优化
- ✅ 实现了完整的推理优化版 Agent

---

## 自测题

1. 推理优化的核心指标有哪些？为什么说三者不可兼得？
2. 批量处理和动态批处理有什么区别？
3. 投机解码的原理是什么？能让速度提升多少？
4. 智能路由是怎么工作的？

## 动手作业

1. 把上面的优化版 Agent 跑起来
2. 添加真实的大模型调用
3. 实现更复杂的智能路由策略
