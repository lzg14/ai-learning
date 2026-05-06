# Day 23：代码 Agent 开发实战

> 从 Copilot 到完整的代码助手，本课深入代码 Agent 的原理、架构、开发与优化，打造你自己的专属 AI 程序员。

## Part 1：代码 Agent 能力边界

### 代码 Agent vs 普通代码补全

| 维度 | 普通代码补全（Copilot） | 代码 Agent |
|------|------------------------|------------|
| **交互方式** | 用户写一部分，AI 补全 | 用户说需求，AI 完成整个任务 |
| **粒度** | 几行代码 | 整个文件、整个功能 |
| **自主性** | 被动响应 | 主动规划、执行、验证 |
| **工具能力** | 无 | 读写文件、运行命令、搜索 |
| **反馈循环** | 单次 | 多轮，Self-Correction |

### 能力边界

**✅ 能做的：**
- 重复性代码生成
- 代码重构和优化
- Bug 修复和调试
- 测试用例生成
- 文档撰写

**❌ 做不到/做不好的：**
- 完全理解复杂业务逻辑
- 处理大型代码库的全部上下文
- 保证代码 100% 正确
- 替代资深开发者的架构设计能力

---

## Part 2：代码 Agent 核心架构

### 五大核心组件

| 组件 | 作用 | 类比 |
|------|------|------|
| **Planner** | 理解需求，规划任务 | 项目经理 |
| **Coder** | 生成和修改代码 | 程序员 |
| **Executor** | 运行代码，获取结果 | 运维 |
| **Reviewer** | 检查代码质量 | 代码审查员 |
| **Memory** | 记住上下文和历史 | 文档 |

### 架构图

```
用户需求
    ↓
┌─────────┐
│ Planner │ → 拆解任务
└────┬────┘
     ↓
┌─────────┐
│  Coder  │ → 生成代码
└────┬────┘
     ↓
┌──────────┐
│ Executor │ → 运行验证
└────┬─────┘
     ↓
┌─────────┐
│ Reviewer │ → 检查质量
└────┬────┘
     ↓
  通过？→ 否 → 回到 Coder 重新生成
     ↓
     是
     ↓
   完成
```

---

## Part 3：实战——代码 Agent 实现

### 完整代码

```python
"""
代码 Agent 实现
支持：需求理解 → 代码生成 → 执行验证 → 质量审查
"""
import os
from dotenv import load_dotenv
from langchain_community.chat_models import ChatZhipuAI
from langchain.tools import tool
from typing import List, Dict
import subprocess

load_dotenv()

llm = ChatZhipuAI(
    api_key=os.getenv("ZHIPU_API_KEY"),
    model="glm-4-flash",
    temperature=0.0
)

# ========== 工具定义 ==========

@tool
def read_file(file_path: str) -> str:
    """读取文件内容"""
    with open(file_path, "r", encoding="utf-8") as f:
        return f.read()

@tool
def write_file(file_path: str, content: str) -> str:
    """写入文件"""
    with open(file_path, "w", encoding="utf-8") as f:
        f.write(content)
    return f"文件 {file_path} 已写入"

@tool
def run_command(command: str) -> str:
    """运行命令"""
    result = subprocess.run(
        command,
        shell=True,
        capture_output=True,
        text=True
    )
    return f"stdout: {result.stdout}\nstderr: {result.stderr}\nreturn_code: {result.returncode}"

# ========== Planner ==========

def plan_task(requirement: str) -> List[str]:
    """规划任务步骤"""
    prompt = f"""分析以下需求，拆解成具体的实现步骤：

需求：{requirement}

请返回 JSON 格式：
{{"steps": ["步骤1", "步骤2", "步骤3"]}}
"""
    response = llm.invoke(prompt).content
    # 简单解析
    if "{" in response:
        import json
        try:
            data = json.loads(response)
            return data.get("steps", [])
        except:
            pass
    return ["分析需求", "编写代码", "测试验证"]

# ========== Coder ==========

def generate_code(task: str, context: str = "") -> str:
    """生成代码"""
    prompt = f"""{context}

任务：{task}

请生成高质量的 Python 代码，代码需要：
1. 符合 PEP8 规范
2. 有适当的注释
3. 包含错误处理

代码：
"""
    return llm.invoke(prompt).content

# ========== Executor ==========

def execute_code(code: str) -> Dict:
    """执行代码"""
    try:
        exec(code)
        return {"success": True, "output": "代码执行成功"}
    except Exception as e:
        return {"success": False, "error": str(e)}

# ========== Reviewer ==========

def review_code(code: str) -> Dict:
    """审查代码"""
    prompt = f"""审查以下代码的问题：

代码：
{code}

请检查：
1. 逻辑错误
2. 安全隐患
3. 代码风格
4. 性能问题

返回 JSON 格式：
{{"issues": ["问题1", "问题2"], "rating": 1-10}}
"""
    response = llm.invoke(prompt).content
    # 简单解析
    import json
    try:
        if "{" in response:
            return json.loads(response)
    except:
        pass
    return {"issues": [], "rating": 8}

# ========== 主 Agent ==========

class CodeAgent:
    """代码 Agent"""

    def __init__(self):
        self.max_iterations = 3
        self.context = []

    def run(self, requirement: str) -> Dict:
        """运行代码 Agent"""
        print(f"🎯 需求：{requirement}\n")

        # 1. 规划
        print("📋 规划任务...")
        steps = plan_task(requirement)
        print(f"计划 {len(steps)} 个步骤\n")

        # 2. 生成代码
        print("💻 生成代码...")
        code = generate_code(requirement, "\n".join(self.context))

        # 3. 执行和审查循环
        for i in range(self.max_iterations):
            print(f"\n🔄 审查轮次 {i+1}...")

            # 审查
            review = review_code(code)
            print(f"代码评分：{review.get('rating', 'N/A')}/10")
            print(f"问题：{review.get('issues', [])}")

            if review.get("rating", 0) >= 8 and not review.get("issues"):
                break

            # 如果有问题，重新生成
            if review.get("issues"):
                print("🔧 修复问题...")
                code = generate_code(
                    requirement + f"\n\n请修复以下问题：{review['issues']}",
                    "\n".join(self.context)
                )

        return {
            "requirement": requirement,
            "code": code,
            "iterations": i + 1
        }

# ========== 使用示例 ==========

if __name__ == "__main__":
    agent = CodeAgent()

    result = agent.run("写一个函数来计算斐波那契数列第 n 项")
    print("\n" + "="*50)
    print("生成代码：")
    print(result["code"])
```

---

## Part 4：高级特性

### 1. 反思机制

```python
def self_reflect(code: str, execution_result: Dict) -> str:
    """反思执行结果，修正代码"""
    if not execution_result["success"]:
        prompt = f"""代码执行失败了：

错误：{execution_result['error']}

代码：
{code}

请分析失败原因，并修复代码：
"""
        return llm.invoke(prompt).content
    return code
```

### 2. 人类介入点

```python
def request_human_review(code: str, question: str) -> str:
    """请求人类审查"""
    print(f"\n🤔 需要人工确认：{question}")
    print("生成的代码：")
    print(code)
    print("\n请确认是否接受 (y/n)：", end=" ")
    response = input()
    if response.lower() == "n":
        print("请描述你想怎么修改：")
        modification = input()
        return modification
    return code
```

### 3. TDD 模式

```python
def tdd_mode(requirement: str) -> str:
    """测试驱动开发模式"""
    # 1. 先写测试
    test_code = generate_code(f"为以下功能写测试：{requirement}")
    write_file("test_temp.py", test_code)

    # 2. 运行测试（应该失败）
    run_command("pytest test_temp.py")

    # 3. 写实现代码
    impl_code = generate_code(f"实现以下功能（测试已准备好）：{requirement}")

    # 4. 运行测试验证
    write_file("impl_temp.py", impl_code)
    result = run_command("pytest test_temp.py")

    return impl_code if "passed" in result else "测试未通过"
```

---

## Part 5：生产级案例——OpenAI Symphony

### 什么是 Symphony？

[Symphony](https://github.com/openai/symphony) 是 OpenAI 于 2026 年 2 月悄悄开源、3 月正式宣布的工程预览项目。它的核心理念是：**团队管理"工作"（Work），而不是监督"编码 Agent"**。

Symphony 把整个软件开发流程自动化——监控 Linear 看板上的任务、自动为每个 Issue 创建隔离工作空间、派发 Codex Agent 去完成、Agent 完成后提交 PR 和"工作证明"（CI 状态、代码审查反馈、复杂度分析、演示视频），工程师只在最后审核通过即可。

> "Engineers do not need to supervise Codex; they can manage the work at a higher level."

### 架构解析

Symphony 由 5 个核心组件构成：

| 组件 | 职责 |
|------|------|
| **Workflow Loader** | 读取仓库里的 `WORKFLOW.md`，解析 YAML 配置和 Prompt 模板 |
| **Config Layer** | 配置层，从 `WORKFLOW.md` 读取参数，支持环境变量注入和默认值 |
| **Issue Tracker Client** | 对接 Linear，读取活跃 Issue、获取当前状态、处理状态流转 |
| **Orchestrator** | 调度器，拥有轮询心跳，管理内存中的运行时状态，决定派发/重试/停止 |
| **Workspace Manager** | 为每个 Issue 创建隔离工作空间，绑定生命周期钩子，清理完成的工作区 |

### 核心工作流

```
[Linear 看板]  ←→  [Orchestrator]  →  [Workspace Manager]  →  [隔离工作空间]
                    ↓
             [Workflow Loader] + [Config Layer]
                    ↓
             [Codex / 编程 Agent]  →  [工作证明：CI/PR反馈/演示视频]
                    ↓
             [工程师审核]  →  PR 合并 / 打回重做
```

关键设计点：
- **隔离工作空间**：每个 Issue 拥有独立目录，Agent 命令只在该目录下执行，防止跨任务污染
- **WORKFLOW.md 入库**：团队把 Agent 行为规范和运行时设置随代码一起版本管理
- **可观测性优先**：至少输出结构化日志，支持追踪多个并发 Agent 运行状态
- **无状态恢复**：重启后不恢复内存状态，但通过 Issue 状态和文件系统驱动恢复

### 对比课堂实现 vs 生产系统

| 维度 | 课堂版（Part 3） | Symphony（生产级） |
|------|-----------------|-------------------|
| 工作空间隔离 | ❌ 无，所有任务在同一目录 | ✅ 每个 Issue 独立工作空间 |
| Issue 对接 | ❌ 手动输入 | ✅ 自动监控 Linear 看板 |
| 工作流规范 | ❌ Hardcode | ✅ `WORKFLOW.md` 驱动 |
| 人类介入 | ❌ 手动确认 | ✅ PR 审核 + 自动合并 |
| 重试机制 | ❌ 无 | ✅ 指数退避 + 状态同步 |
| 并发控制 | ❌ 串行 | ✅ 有界并发调度 |

### 动手作业

1. 阅读 [Symphony SPEC.md](https://github.com/openai/symphony/blob/main/SPEC.md)，理解完整的规范设计
2. 参考 Symphony 的 Workspace Manager 思路，给课堂代码 Agent 添加"项目隔离"能力
3. 实现一个简化版的 `WORKFLOW.md` 驱动 Agent，根据配置文件切换不同的 Prompt 策略

---

## 📌 今日要点

- ✅ 理解了代码 Agent 和普通代码补全的区别
- ✅ 知道了代码 Agent 的能力边界
- ✅ 掌握了代码 Agent 的 5 大核心组件
- ✅ 实现了完整的代码 Agent
- ✅ 学会了反思机制、人类介入点、TDD 模式
- ✅ 了解了 OpenAI Symphony——生产级代码 Agent 系统的架构与设计思路

---

## 自测题

1. 代码 Agent 和普通的代码补全有什么本质区别？
2. 代码 Agent 的 5 大核心组件是什么？
3. 为什么不能让代码 Agent 完全自主运行？
4. 反思机制的作用是什么？

---

## 动手作业

1. 实现一个完整的代码 Agent
2. 给代码 Agent 添加文件搜索能力
3. 实现 TDD 模式的代码生成
