# Day 25：模型部署与推理优化

> 从训练到上线：vLLM / TGI 推理框架、量化技术（GPTQ / AWQ / GGUF）、批处理与并发优化。

## Part 1：推理框架对比

### 主流框架

| 框架 | 核心技术 | 性能 | 易用性 | 推荐场景 |
|------|----------|------|--------|----------|
| **vLLM** | PagedAttention | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 生产级高吞吐 |
| **TGI** | 动态批处理 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 企业级生产 |
| **SGLang** | RadixAttention | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 高并发场景 |
| **llama.cpp** | GGUF 量化 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 边缘/低资源 |

### 性能对比（7B 模型）

| 方案 | 吞吐量 |
|------|--------|
| 纯 HuggingFace | ~50 token/s |
| vLLM | ~300-800 token/s |
| SGLang | ~400-1000+ token/s |

---

## Part 2：vLLM 实战

### PagedAttention 原理

**核心思想：像操作系统管理内存一样管理 KV Cache**

| 对比 | 传统 KV Cache | PagedAttention |
|------|--------------|----------------|
| 内存利用率 | 40-60% | 95%+ |
| Batch Size | 小 | 大 |
| 吞吐量 | 低 | 高（20-100x） |

### 安装和使用

```bash
pip install vllm
```

```python
from vllm import LLM, SamplingParams

# 初始化模型
llm = LLM(
    model="THUDM/glm-4-9b-chat",
    max_model_len=8192,
    gpu_memory_utilization=0.9,
    quantization="awq",  # 量化加速
    trust_remote_code=True
)

# 采样参数
sampling_params = SamplingParams(
    temperature=0.7,
    top_p=0.9,
    max_tokens=512
)

# 批量推理
prompts = ["你好", "解释大模型"]
outputs = llm.generate(prompts, sampling_params)

for output in outputs:
    print(output.outputs[0].text)
```

---

## Part 3：量化技术

### 量化技术对比

| 技术 | 精度 | 压缩率 | 推荐场景 |
|------|------|--------|----------|
| **FP16** | 16bit | 1x | 追求最高质量 |
| **GPTQ** | 4bit | 4x | 生产部署 |
| **AWQ** | 4bit | 4x | **推荐** |
| **GGUF** | 2~8bit | 2~8x | llama.cpp |

### 显存节省（7B 模型）

| 精度 | 文件大小 | 节省比例 |
|------|----------|----------|
| FP16 | 14 GB | 0% |
| 8bit | 7 GB | 50% |
| 4bit | 3.5 GB | 75% |

---

## Part 4：国产模型部署

### 智谱 GLM 部署

```python
"""
智谱 GLM 本地部署示例
"""
from transformers import AutoModel, AutoTokenizer

model_path = "THUDM/glm-4-9b-chat"

# 加载模型
tokenizer = AutoTokenizer.from_pretrained(model_path, trust_remote_code=True)
model = AutoModel.from_pretrained(model_path, trust_remote_code=True).cuda()

# 推理
inputs = tokenizer("你好，请介绍一下你自己", return_tensors="pt").to("cuda")
outputs = model.generate(**inputs, max_new_tokens=512)
print(tokenizer.decode(outputs[0], skip_special_tokens=True))
```

### DeepSeek 部署

```python
"""
DeepSeek 本地部署
"""
from transformers import AutoModelForCausalLM, AutoTokenizer

model_path = "deepseek-ai/DeepSeek-V2.5"

tokenizer = AutoTokenizer.from_pretrained(model_path)
model = AutoModelForCausalLM.from_pretrained(
    model_path,
    device_map="auto",
    torch_dtype="auto"
)

# 推理
inputs = tokenizer("解释量子计算", return_tensors="pt")
outputs = model.generate(**inputs, max_new_tokens=512)
print(tokenizer.decode(outputs[0]))
```

---

## Part 5：API 服务部署

### FastAPI + vLLM

```python
"""
推理 API 服务
"""
from fastapi import FastAPI
from pydantic import BaseModel
from vllm import LLM, SamplingParams
from typing import List, Optional

app = FastAPI()

# 全局模型
llm = None

@app.on_event("startup")
async def load_model():
    global llm
    llm = LLM(
        model="THUDM/glm-4-9b-chat",
        max_model_len=8192,
        gpu_memory_utilization=0.9
    )

class ChatRequest(BaseModel):
    messages: List[dict]
    temperature: float = 0.7
    max_tokens: int = 512

class ChatResponse(BaseModel):
    content: str
    usage: dict

@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    # 构建 prompt
    prompt = build_prompt_from_messages(request.messages)

    # 采样参数
    sampling_params = SamplingParams(
        temperature=request.temperature,
        max_tokens=request.max_tokens
    )

    # 推理
    outputs = llm.generate([prompt], sampling_params)
    content = outputs[0].outputs[0].text

    return ChatResponse(
        content=content,
        usage={
            "prompt_tokens": outputs[0].prompt_token_ids,
            "completion_tokens": len(outputs[0].outputs[0].token_ids)
        }
    )

def build_prompt_from_messages(messages: List[dict]) -> str:
    """构建 prompt"""
    parts = []
    for msg in messages:
        if msg["role"] == "system":
            parts.append(f"System: {msg['content']}")
        elif msg["role"] == "user":
            parts.append(f"User: {msg['content']}")
        else:
            parts.append(f"Assistant: {msg['content']}")
    return "\n".join(parts) + "\nAssistant:"

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## Part 6：性能监控

### 关键指标

| 指标 | 说明 | 目标 |
|------|------|------|
| 吞吐量（TPS） | 每秒生成的 token 数 | 越高越好 |
| 延迟（P50/P99） | 每个请求的响应时间 | 越短越好 |
| GPU 利用率 | GPU 计算单元的利用率 | 85-95% |
| 显存利用率 | 显存占用比例 | 不超过 90% |

### 监控实现

```python
import prometheus_client as prom

# 定义指标
REQUEST_LATENCY = prom.Histogram(
    "request_latency_seconds",
    "Request latency",
    ["endpoint"]
)

THROUGHPUT = prom.Counter(
    "tokens_generated_total",
    "Total tokens generated"
)

@REQUEST_LATENCY.time()
async def handle_request(request):
    # 处理请求
    result = await generate(request)
    THROUGHPUT.inc(len(result["tokens"]))
    return result

# 启动监控
prom.start_http_server(8001)
```

---

## Part 7：Kubernetes 部署

生产环境不可能单机运行，需要用 K8s 做编排。

### K8s + vLLM 部署架构

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-server
spec:
  replicas: 3  # 水平扩展
  selector:
    matchLabels:
      app: vllm-server
  template:
    metadata:
      labels:
        app: vllm-server
    spec:
      containers:
      - name: vllm
        image: vllm/vllm-openai:latest
        command: ["python3", "-m", "vllm.entrypoints.openai.api_server"]
        args:
          - "--model"
          - "THUDM/glm-4-9b-chat"
          - "--tensor-parallel-size"
          - "2"
          - "--gpu-memory-utilization"
          - "0.9"
        ports:
        - containerPort: 8000
        resources:
          limits:
            nvidia.com/gpu: 2  # 每 Pod 2 张 GPU
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 60
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: vllm-service
spec:
  selector:
    app: vllm-server
  ports:
  - port: 8000
    targetPort: 8000
  type: ClusterIP
```

### 自动缩放

```yaml
# hpa.yaml - 根据 GPU 利用率自动扩缩
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: vllm-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    name: vllm-server
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: nvidia.com/gpu
      target:
        type: Utilization
        averageUtilization: 80
```

---

## Part 8：AI 网关

AI 网关统一管理多个模型的路由、限流和负载均衡。

### 网关的核心功能

| 功能 | 说明 | 实现方式 |
|------|------|----------|
| **请求路由** | 将请求分发到合适的模型 | 基于模型名称/负载 |
| **限流保护** | 防止突发流量打垮服务 | 令牌桶/漏桶算法 |
| **负载均衡** | 均匀分配请求到多个副本 | 轮询/最小连接数 |
| **熔断降级** | 后端故障时快速失败 | 断路器模式 |
| **日志审计** | 记录所有请求和响应 | 异步日志 |

### 基于 Envoy 的 AI 网关

```yaml
# envoy-ai-gateway.yaml
static_resources:
  listeners:
  - name: ai_listener
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 8080
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          route_config:
            virtual_hosts:
            - name: ai_backend
              domains: ["*"]
              routes:
              - match:
                  prefix: "/v1/chat/completions"
                route:
                  cluster: vllm_cluster
                  retry_policy:
                    num_retries: 3
                    retry_on: "5xx"
          http_filters:
          - name: envoy.filters.http.router

  clusters:
  - name: vllm_cluster
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: vllm-service
              port_value: 8000
```

### 国产推理部署方案

| 厂商 | 产品 | 特点 |
|------|------|------|
| **华为** | ModelArts + 昇腾 | 国产芯片适配，性能优秀 |
| **阿里云** | PAI-EAS | Qwen 系列原生支持 |
| **腾讯云** | TI-ONE | 混元模型生态 |
| **火山引擎** | 方舟平台 | DeepSeek 原生支持 |

### 生产部署最佳实践

```plaintext
□ 健康检查：/health 端点 + readinessProbe
□ 优雅关闭：捕获 SIGTERM，处理完正在处理的请求
□ 日志收集：结构化日志（JSON格式）+ 集中式日志系统
□ 监控告警：Prometheus + Grafana + 告警规则
□ 备份策略：模型权重定期备份，分布在不同节点
□ 回滚方案：保留上一版本模型，一键回滚
□ 压测验证：上线前做压力测试，确定最大QPS
```

---

## 📌 今日要点

- ✅ 掌握了主流推理框架对比（vLLM / TGI / SGLang / llama.cpp）
- ✅ 理解了 PagedAttention 核心技术
- ✅ 学会了 vLLM 的离线推理和 API 服务部署
- ✅ 掌握了量化技术（GPTQ / AWQ / GGUF）
- ✅ 实现了高吞吐推理服务

---

## 自测题

1. vLLM 的核心技术 PagedAttention 解决了什么问题？
2. 4bit 量化相比 FP16 能节省多少显存？
3. 如何使用 FastAPI 部署推理服务？
4. 性能监控的四大指标是什么？
5. K8s 部署推理服务时，HPA 的作用是什么？

---

## 动手作业

1. 用 vLLM 部署一个 GLM-4-9B 的 API 服务
2. 将一个 7B 模型用 AWQ 量化为 4bit
3. 设计一个推理服务的监控面板
4. （进阶）写一个 K8s Deployment + Service 的 YAML 配置文件
