# Day 6：MCP 协议与工具生态

> 本课程深入理解 MCP（Model Context Protocol）协议，以及 Skills 作为 MCP 的用户友好封装。

## Part 1：为什么需要 MCP？

### 当前 Agent 接入工具的困境

**传统方式的问题：**

```
每个 AI 应用都要写一套工具连接代码

┌─────────────────────────────────────────────────┐
│ Claude Desktop                                 │
│   → 需要单独集成天气 API                        │
│   → 需要单独集成 GitHub API                     │
│   → 需要单独集成 Slack API                     │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ Cursor AI                                      │
│   → 需要单独集成天气 API                        │
│   → 需要单独集成 GitHub API                     │
│   → 需要单独集成 Figma API                     │
└─────────────────────────────────────────────────┘

每个应用 × 每个工具 = N×M 套集成代码 ❌
```

**Function Calling 的局限：**

| 限制 | 说明 |
|------|------|
| 模型绑定 | GLM 的 Function Calling 代码，DeepSeek 不能直接用 |
| 协议碎片化 | 每个模型有自己的 tool_call 格式 |
| 难以扩展 | 需要为每个新工具写专用代码 |
| 状态管理 | 无法保持跨会话的连接状态 |

### MCP 的解决思路

**MCP = 统一协议，让 AI 应用和工具解耦**

```
┌────────────────────────────────────────────────────┐
│                 MCP Protocol                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Claude   │  │ Cursor   │  │ Any AI   │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │              │              │            │
│       └──────────────┴──────────────┘            │
│                    │                             │
│            统一的 MCP Client                     │
│                    │                             │
│       ┌────────────┴────────────┐               │
│       ▼                        ▼               │
│  ┌──────────┐           ┌──────────┐           │
│  │ Weather  │           │  GitHub  │           │
│  │   MCP     │           │   MCP     │           │
│  └──────────┘           └──────────┘           │
└────────────────────────────────────────────────────┘

一个 MCP Client + N 个 MCP Server = 所有工具可用 ✅
```

---

## Part 2：MCP 架构详解

### MCP 核心组件

```
┌─────────────────────────────────────────────────────────────┐
│                        MCP 架构                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────────┐         ┌─────────────────┐          │
│   │                 │         │                 │          │
│   │   AI 应用       │         │   MCP Server    │          │
│   │   (Host App)    │◄───────►│   (Data Source) │          │
│   │                 │         │                 │          │
│   │  ┌───────────┐  │         │  ┌───────────┐  │          │
│   │  │ MCP Client│  │         │  │  Tools    │  │          │
│   │  └───────────┘  │         │  └───────────┘  │          │
│   │  ┌───────────┐  │         │  ┌───────────┐  │          │
│   │  │ AI Model  │  │         │  │ Resources │  │          │
│   │  └───────────┘  │         │  └───────────┘  │          │
│   └─────────────────┘         └─────────────────┘          │
│                                                              │
│   连接方式：                                                │
│   - stdio（本地进程通信）                                  │
│   - HTTP/SSE（远程服务）                                   │
└─────────────────────────────────────────────────────────────┘
```

### MCP Server 的三类能力

| 类型 | 说明 | 示例 |
|------|------|------|
| **Tools** | AI 可以调用的操作 | `search_web`、`send_email`、`query_database` |
| **Resources** | AI 可以读取的数据 | `file://config.json`、`database://users` |
| **Prompts** | 预定义的提示模板 | `/analyze_sales`、`/code_review` |

### MCP 协议消息格式

```json
// 1. 初始化请求
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {},
      "resources": {}
    },
    "clientInfo": {
      "name": "my-ai-app",
      "version": "1.0.0"
    }
  }
}

// 2. 工具调用请求
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "北京"
    }
  }
}

// 3. 工具调用响应
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "北京今天晴，25°C"
      }
    ]
  }
}
```

---

## Part 3：Function Calling 与 MCP 的关系

### 对比分析

| 维度 | Function Calling | MCP |
|------|------------------|-----|
| **定位** | 模型 API 特性 | 生态协议标准 |
| **层级** | 模型 → 工具 | 应用 → 工具 |
| **标准化** | 各厂商私有 | 社区开放标准 |
| **工具数量** | 单次 5-10 个 | 可动态加载 |
| **状态保持** | 无状态 | 可保持连接状态 |
| **适用场景** | 单模型单应用 | 多模型多应用共享 |

### 互补关系

```
MCP 是更高层的抽象，Function Calling 是其底层实现之一

┌─────────────────────────────────────────────────────┐
│                    你的 Agent                        │
│                                                      │
│  ┌───────────────────┐                              │
│  │  MCP Client       │                              │
│  │                   │                              │
│  │  ┌─────────────┐  │   1. MCP 定义工具接口        │
│  │  │ MCP Server  │──┼──► 2. Function Calling 传给模型│
│  │  └─────────────┘  │   3. 模型返回调用指令         │
│  │                   │   4. MCP Client 执行工具      │
│  └───────────────────┘                              │
└─────────────────────────────────────────────────────┘
```

---

## Part 4：Skills 是什么？

### Skills 定义

**Skills = MCP 的用户友好封装**

| 维度 | MCP | Skills |
|------|-----|--------|
| 定位 | 底层协议 | 上层应用 |
| 用户 | 开发者 | 普通用户 |
| 配置 | JSON 配置 | 可视化安装 |
| 生态 | 社区标准 | 官方认证 |

### Skills 核心能力

- **一键安装**：用户可直接添加各种技能
- **预设工具**：内置常用的 GitHub、Slack、文件系统等
- **可视化管理**：通过界面安装/卸载
- **国产类比**：类似钉钉小程序、微信插件

### Skills 2026 年最新进展：从 Claude 特性到行业标准

2025 年 10 月，Anthropic 在 Claude Code 中引入了 Agent Skills 概念——一种"文件夹加 Markdown"的能力封装方式。短短几个月后，这已经成为 AI Agent 能力组织的事实标准：

| 事件 | 时间 | 影响 |
|------|------|------|
| Anthropic 在 Claude Code 引入 Skills | 2025年10月 | 开创性的"文件夹+Markdown"设计 |
| agentskills.io 上线 | 2025年12月18日 | Agent Skills 规范正式开放 |
| 微软 VS Code / GitHub 集成 | 2025年12月 | 微软直接采用该规范 |
| OpenAI Codex CLI / ChatGPT 跟进 | 2026年初 | OpenAI 几乎复制了同一架构 |
| Cursor / Trae / Qoder 跟进 | 2026年 | 国内主流 IDE 全面支持 |

**Skills 的核心价值：**把专业能力封装为可发现、可安装、可组合的单元。AI 应用不需要内置所有能力，用户按需安装——就像手机应用商店，但能力精确到"会写 Python"、"懂财务分析"这个粒度。

**国产动态：** Skills 规范的开源特性使得国内厂商可以无缝对接。Trae IDE 已经完整实现了 Skills 的安装、管理和调用流程，与 agentskills.io 生态兼容。

### 主流 Skills 示例

| Skills | 功能 | 使用场景 |
|--------|------|----------|
| GitHub | 代码仓库管理 | 查看 PR、提交代码 |
| Slack | 团队协作 | 发送消息、查看频道 |
| Filesystem | 文件操作 | 读写本地文件 |
| Database | 数据库查询 | SQL 查询、数据分析 |
| Web Search | 网络搜索 | 实时信息查询 |

---

## Part 5：国产 MCP 生态

### MCP 生态现状（2025）

| 厂商 | MCP 支持 | SDK |
|------|----------|-----|
| **GLM** | ✅ 支持 | `mcp-zhipuai` |
| **DeepSeek** | 🔜 规划中 | - |
| **Qwen** | ✅ 支持 | `dashscope-mcp` |
| **Kimi** | 🔜 规划中 | - |

### MCP 生态 2026 年最新动态

MCP 自 2024 年 11 月发布以来，已成为 AI Agent 工具连接的事实标准：

| 里程碑 | 时间 | 说明 |
|--------|------|------|
| **Linux Foundation 托管** | 2025 年 12 月 | 捐赠给 Agentic AI Foundation，厂商中立治理 |
| **月下载量** | 2026 年 | 9700 万+ SDK 下载 |
| **公开 Server** | 2026 年 | 10,000+ 社区构建的 MCP Server |
| **Claude** | 原生支持 | Anthropic 全线产品 |
| **ChatGPT** | 2025 年 3 月 | OpenAI 原生支持 MCP |
| **Gemini** | 2025 年 4 月 | Google 原生支持 |
| **VS Code / Copilot** | 2025 年 7 月 | Microsoft 全面支持 |

**核心变化：** MCP 从 Anthropic 的"私有标准"变成了真正的行业标准，所有主流 AI 平台都原生支持。这意味着：

- 开发一个 MCP Server = 所有 AI 平台都能用
- 不再需要为 Claude/ChatGPT/Gemini 各自写一套集成
- MCP 就是 AI 工具连接的 "USB-C 接口"

### 国产 MCP Server 开发

```python
# 使用国产 MCP SDK 开发自己的 MCP Server
from mcp.server import MCPServer
from mcp.types import Resource, Tool

class 国产工具MCPServer(MCPServer):
    """国产工具 MCP Server 示例"""

    def __init__(self):
        super().__init__(name="国产工具集", version="1.0.0")

    def get_tools(self) -> list[Tool]:
        """暴露可调用的工具"""
        return [
            Tool(
                name="query_china_weather",
                description="查询中国城市天气",
                inputSchema={
                    "type": "object",
                    "properties": {
                        "city": {
                            "type": "string",
                            "description": "城市名称，如：北京、上海"
                        }
                    },
                    "required": ["city"]
                }
            ),
            Tool(
                name="search_baidu",
                description="百度搜索",
                inputSchema={
                    "type": "object",
                    "properties": {
                        "query": {"type": "string", "description": "搜索关键词"}
                    },
                    "required": ["query"]
                }
            )
        ]

# 启动服务
if __name__ == "__main__":
    server = 国产工具MCPServer()
    server.run(transport="stdio")
```

---

## 今日要点

- ✅ 理解了为什么需要 MCP——解决 AI 应用与工具集成的碎片化问题
- ✅ 掌握了 MCP 的核心架构：Client、Server、Tools/Resources/Prompts
- ✅ 理解了 MCP 与 Function Calling 的关系——互补而非替代
- ✅ 了解了 Skills 的概念——MCP 的用户友好封装
- ✅ 了解了国产模型的 MCP 支持情况

---

## 自测题

1. MCP 解决了 AI 工具集成的什么问题？
2. MCP Server 可以暴露哪三类能力？
3. MCP 与 Function Calling 的关系是什么？
4. Skills 和 MCP 的区别是什么？
5. 国产模型中，哪两个已经支持 MCP？

---

## Part 6：预告——A2A 协议

MCP 解决了"Agent 连接工具"的问题，但还有一个关键问题没解决：**Agent 之间怎么通信？**

这就是 **A2A（Agent-to-Agent Protocol）** —— Google 在 2025 年 4 月发布的 Agent 间通信协议，现已捐赠给 Linux Foundation。

```
MCP → Agent 连接工具（本课主题）
A2A → Agent 连接 Agent（下一课：Day15）
```

简单理解：

| 协议 | 类比 | 解决什么问题 |
|------|------|-------------|
| MCP | USB-C 接口 | Agent 怎么用工具 |
| A2A | 电子邮件/HTTP | Agent 之间怎么协作 |

> 在 Day15（A2A 协议与 Agent 间通信标准）中，我们将深入学习 A2A 的完整原理和实战。

---

## 今日要点

- ✅ 理解了为什么需要 MCP——解决 AI 应用与工具集成的碎片化问题
- ✅ 掌握了 MCP 的核心架构：Client、Server、Tools/Resources/Prompts
- ✅ 理解了 MCP 与 Function Calling 的关系——互补而非替代
- ✅ 了解了 Skills 的概念——MCP 的用户友好封装
- ✅ 了解了 MCP 2026 年生态：Linux Foundation 托管、9700 万+月下载量
- ✅ 了解了 A2A 协议与 MCP 的互补关系

---

## 自测题

1. MCP 解决了 AI 工具集成的什么问题？
2. MCP Server 可以暴露哪三类能力？
3. MCP 与 Function Calling 的关系是什么？
4. 2026 年 MCP 生态有哪些重要里程碑？
5. MCP 和 A2A 分别解决什么问题？两者是什么关系？

---

## 动手作业

1. 了解你使用的 AI 工具是否支持 MCP
2. 思考：你的业务场景中，哪些工具适合用 MCP 封装？
3. 对比 MCP 和 A2A 的适用场景，思考两者的互补关系
