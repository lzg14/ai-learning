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

## 📌 今日要点

- ✅ 理解了代码 Agent 和普通代码补全的区别
- ✅ 知道了代码 Agent 的能力边界
- ✅ 掌握了代码 Agent 的 5 大核心组件
- ✅ 实现了完整的代码 Agent
- ✅ 学会了反思机制、人类介入点、TDD 模式

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
