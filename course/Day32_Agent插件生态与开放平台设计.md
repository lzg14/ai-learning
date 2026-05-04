# Day 32：Agent 插件生态与开放平台设计

> 一个人的能力有限，一群人的能力无限。本课深入插件系统设计、权限控制、沙箱隔离、第三方接入、插件市场运营。

## Part 1：为什么需要插件生态？系统的三个阶段

### 为什么需要插件生态？

#### 阶段一：全自研（0 到 1）
- 所有功能都是你自己团队写的
- 迭代快，质量可控
- 问题：人手有限，能做的功能也有限

#### 阶段二：开放 API（1 到 10）
- 你把核心能力封装成 API 开放出去
- 第三方开发者可以基于你的 API 做二次开发
- 问题：他们只能"调用"你的能力，不能"扩展"你的能力

#### 阶段三：插件生态（10 到 100）
- 第三方开发者可以写插件，直接扩展你的系统本身的能力
- 你的系统变成一个平台，所有人都可以往上面加功能
- 结果：一万个开发者帮你做功能

**例子：**
- Chrome：本身只是个浏览器，因为有插件生态，变成了无所不能的平台
- VS Code：本身只是个编辑器，因为有插件生态，变成了全栈开发工具

> 💡 你的平台能长多大，不取决于你自己能写多少功能，而取决于你能让多少第三方开发者在你的平台上写功能。

---

### 插件生态的价值

| 对平台方的价值 | 对开发者的价值 | 对用户的价值 |
| --- | --- | --- |
| 不用自己写所有功能 | 开发成本低，比做一个完整产品容易 100 倍 | 想要什么功能都有 |
| 专注于核心能力 | 可以利用平台的流量获客 | 可以自定义自己的专属体验 |
| 形成网络效应 | 可以收费卖插件赚钱 | 永远有新功能用 |

---

### 插件生态的风险

| 风险 | 说明 | 怎么解决 |
| --- | --- | --- |
| **安全风险** | 恶意插件偷用户数据、删用户东西 | 沙箱隔离、权限控制、审核机制 |
| **质量风险** | 垃圾插件、bug 插件把用户体验搞坏 | 评分系统、审核机制 |
| **生态混乱** | 一百个做同一件事的插件 | 分类、搜索、推荐、官方精选 |

---

## Part 2：插件系统核心设计

### 插件接口设计原则

**最少够用原则：** 给插件的 API 越少越好，只给它必须的。
- 给的权限越多，安全风险越大
- 给的 API 越多，后面的兼容性包袱越重

**向后兼容原则：** 插件 API 一旦发布了，就永远不能改。
- 你改了 API，几千个第三方插件就都崩了
- 宁肯加新 API，也不要改旧 API

---

### 插件 Manifest 文件设计

每个插件必须有一个 manifest.json：

```json
{
  "id": "com.example.todo-plugin",
  "name": "待办事项插件",
  "version": "1.0.0",
  "author": "张三",
  "description": "帮你管理待办事项",
  "permissions": [
    "agent.message.send",
    "agent.user_data.read",
    "agent.user_data.write"
  ],
  "entrypoint": "https://example.com/plugin/todo",
  "commands": [
    {
      "name": "add_todo",
      "description": "添加一个待办事项",
      "parameters": {
        "type": "object",
        "properties": {
          "content": {"type": "string", "description": "待办内容"}
        },
        "required": ["content"]
      }
    }
  ]
}
```

---

### 权限系统设计

**最小权限原则：** 插件要什么权限就给什么权限，不给多余的。

| 权限等级 | 说明 | 例子 | 审核要求 |
| --- | --- | --- | --- |
| **只读权限** | 只能读数据，不能改 | 读用户消息 | 宽松 |
| **读写权限** | 可以读也可以写 | 写用户数据、发消息 | 严格 |
| **敏感权限** | 涉及隐私、钱、删除 | 删数据、调用支付 | 极严格 |
| **高危权限** | 影响整个系统 | 执行任意代码 | 基本不给 |

**用户授权流程：**
1. 用户安装插件时，明确弹出提示告诉用户权限申请
2. 把每个权限是什么意思，用大白话写清楚
3. 用户任何时候都可以取消权限

---

### 沙箱隔离设计

**核心思想：插件是不可信的，是恶意的，默认它会干所有能想到的坏事。**

| 隔离层级 | 说明 | 安全程度 |
| --- | --- | --- |
| **API 层隔离** | 所有对平台的调用都经过权限校验 | ⭐⭐ |
| **进程级隔离** | 每个插件跑在独立进程里 | ⭐⭐⭐ |
| **容器级隔离** | 每个插件跑在独立 Docker 容器里 | ⭐⭐⭐⭐ |
| **虚拟机级隔离** | 每个插件跑在独立 VM 里 | ⭐⭐⭐⭐⭐ |

**最佳实践：**
- 插件的网络访问默认禁止，要申请才能访问外网
- 插件的执行时间限制：最多跑 30 秒，超时直接杀掉
- 插件的内存限制：最多用 128MB 内存

---

## Part 3：开发者接入与审核流程

### 开发者接入的 5 步流程

```plaintext
1. 注册开发者账号
   ↓
2. 创建应用，获取 AppID 和 AppSecret
   ↓
3. 开发插件，按照平台规范写代码
   ↓
4. 提交审核
   ↓
5. 审核通过，发布到插件市场
```

> 💡 开发者体验不好的开放平台，一定做不起来。文档写得越烂，开发者越少。

---

### 审核流程设计

#### 审核内容：
1. **功能审核**：插件有没有用，是不是垃圾
2. **安全审核**：有没有恶意代码，有没有偷数据
3. **合规审核**：有没有违法内容，有没有侵权
4. **性能审核**：会不会特别慢，会不会占太多资源

#### 审核分级：

| 插件类型 | 审核方式 | 审核时间 |
| --- | --- | --- |
| 个人开发者，只读权限 | 自动审核 | 几分钟 |
| 个人开发者，读写权限 | 半人工审核 | 1-2 个工作日 |
| 企业开发者，敏感权限 | 全人工审核 | 3-5 个工作日 |

---

### 插件市场设计

**核心功能：**
1. **分类和搜索**：用户能快速找到自己想要的插件
2. **评分和评价**：用过的用户打分和写评价
3. **官方精选**：官方推荐的高质量插件
4. **安装量和更新时间**：大家都在装的插件一般不会太差

---

## Part 4：实战——实现一个简单的 Agent 插件系统

### 完整代码

```python
"""
简单的 Agent 插件系统实现
包含：插件注册、权限校验、沙箱执行、调用插件
"""
import asyncio
import json
import re
from typing import Dict, List, Any, Optional
from dataclasses import dataclass
from enum import Enum

class Permission(Enum):
    """权限枚举"""
    MESSAGE_READ = "agent.message.read"
    MESSAGE_SEND = "agent.message.send"
    USER_DATA_READ = "agent.user_data.read"
    USER_DATA_WRITE = "agent.user_data.write"
    NETWORK_ACCESS = "system.network_access"
    EXECUTE_CODE = "system.execute_code"

@dataclass
class Plugin:
    """插件信息"""
    id: str
    name: str
    version: str
    author: str
    description: str
    permissions: List[Permission]
    entrypoint: str
    commands: Dict[str, Any]
    enabled: bool = True

class PermissionDeniedError(Exception):
    """权限不足异常"""
    pass

class PluginManager:
    """插件管理器"""

    def __init__(self):
        self.plugins: Dict[str, Plugin] = {}
        self.installed_plugins: Dict[str, List[str]] = {}

    def register_plugin(self, manifest: Dict) -> Plugin:
        """注册一个插件"""
        plugin = Plugin(
            id=manifest["id"],
            name=manifest["name"],
            version=manifest["version"],
            author=manifest["author"],
            description=manifest["description"],
            permissions=[Permission(p) for p in manifest["permissions"]],
            entrypoint=manifest["entrypoint"],
            commands={cmd["name"]: cmd for cmd in manifest["commands"]}
        )
        self.plugins[plugin.id] = plugin
        print(f"✅ 插件 {plugin.name} ({plugin.id}) 注册成功")
        return plugin

    def install_plugin_for_user(self, user_id: str, plugin_id: str) -> bool:
        """用户安装一个插件"""
        if plugin_id not in self.plugins:
            raise ValueError(f"插件 {plugin_id} 不存在")
        if user_id not in self.installed_plugins:
            self.installed_plugins[user_id] = []
        if plugin_id not in self.installed_plugins[user_id]:
            self.installed_plugins[user_id].append(plugin_id)
            print(f"✅ 用户 {user_id} 安装了插件 {plugin_id}")
        return True

    def get_user_plugins(self, user_id: str) -> List[Plugin]:
        """获取用户安装的所有插件"""
        if user_id not in self.installed_plugins:
            return []
        return [self.plugins[pid] for pid in self.installed_plugins[user_id] if pid in self.plugins]

    def check_permission(self, plugin_id: str, permission: Permission) -> bool:
        """检查插件有没有某个权限"""
        if plugin_id not in self.plugins:
            return False
        return permission in self.plugins[plugin_id].permissions

class PluginExecutor:
    """插件执行器"""

    def __init__(self, plugin_manager: PluginManager):
        self.plugin_manager = plugin_manager

    async def execute_plugin(
        self,
        plugin_id: str,
        command: str,
        args: Dict,
        current_user_id: str
    ) -> Any:
        """执行一个插件的命令，带权限校验"""

        # 1. 检查插件是否存在
        if plugin_id not in self.plugin_manager.plugins:
            raise ValueError(f"插件 {plugin_id} 不存在")

        plugin = self.plugin_manager.plugins[plugin_id]

        # 2. 检查命令是否存在
        if command not in plugin.commands:
            raise ValueError(f"插件 {plugin.name} 没有命令 {command}")

        # 3. 检查参数是否合法
        command_info = plugin.commands[command]
        required_params = command_info["parameters"].get("required", [])
        for param in required_params:
            if param not in args:
                raise ValueError(f"缺少必填参数：{param}")

        # 4. 权限校验
        if command == "send_message" and not self.plugin_manager.check_permission(
            plugin_id, Permission.MESSAGE_SEND
        ):
            raise PermissionDeniedError(f"插件 {plugin.name} 没有发消息的权限")

        # 5. 沙箱执行
        print(f"🚀 执行插件：{plugin.name}，命令：{command}，参数：{args}")
        print(f"🔒 权限校验通过，插件权限：{[p.value for p in plugin.permissions]}")

        await asyncio.sleep(0.5)

        result = {
            "success": True,
            "plugin_id": plugin_id,
            "command": command,
            "result": f"执行成功：{command} 完成",
            "args_echo": args
        }

        print(f"✅ 插件执行完成：{result}")
        return result

class AgentWithPlugin:
    """支持插件的 Agent"""

    def __init__(self):
        self.plugin_manager = PluginManager()
        self.executor = PluginExecutor(self.plugin_manager)

    def parse_user_command(self, user_input: str) -> tuple:
        """解析用户输入，格式：@插件名 命令名 参数1=值1"""
        pattern = r"@(\w+)\s+(\w+)\s+(.*)"
        match = re.match(pattern, user_input.strip())
        if not match:
            return None, None, None

        plugin_name, command, args_str = match.group(1), match.group(2), match.group(3)

        args = {}
        for part in args_str.split(" "):
            if "=" in part:
                k, v = part.split("=", 1)
                args[k.strip()] = v.strip()

        plugin_id = None
        for pid, plugin in self.plugin_manager.plugins.items():
            if plugin.name.lower() == plugin_name.lower():
                plugin_id = pid
                break

        return plugin_id, command, args

    async def process_user_message(self, user_id: str, user_input: str) -> str:
        """处理用户消息"""
        print(f"\n👤 用户 {user_id} 说：{user_input}")

        plugin_id, command, args = self.parse_user_command(user_input)

        if plugin_id and command:
            user_plugins = self.plugin_manager.get_user_plugins(user_id)
            if not any(p.id == plugin_id for p in user_plugins):
                return f"❌ 你还没有安装插件 {plugin_id}，请先安装再使用"

            try:
                result = await self.executor.execute_plugin(plugin_id, command, args, user_id)
                return f"✅ 插件执行成功：\n{json.dumps(result, ensure_ascii=False, indent=2)}"
            except PermissionDeniedError as e:
                return f"❌ 权限不足：{e}"
            except Exception as e:
                return f"❌ 插件执行失败：{e}"

        return "我是支持插件的 Agent，你可以用 @插件名 来调用已安装的插件~"

async def main():
    print("🚀 初始化 Agent 插件系统")
    print("=" * 80)

    agent = AgentWithPlugin()

    # 注册待办事项插件
    todo_manifest = {
        "id": "com.example.todo-plugin",
        "name": "todo",
        "version": "1.0.0",
        "author": "张三",
        "description": "帮你管理待办事项",
        "permissions": ["agent.user_data.read", "agent.user_data.write"],
        "entrypoint": "https://example.com/plugin/todo",
        "commands": [
            {
                "name": "add_todo",
                "description": "添加一个待办事项",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "content": {"type": "string", "description": "待办内容"},
                        "due_date": {"type": "string", "description": "截止日期"}
                    },
                    "required": ["content"]
                }
            }
        ]
    }
    agent.plugin_manager.register_plugin(todo_manifest)

    # 用户安装插件
    user_id = "user_123"
    agent.plugin_manager.install_plugin_for_user(user_id, "com.example.todo-plugin")

    print(f"\n📋 用户 {user_id} 已安装的插件：")
    for p in agent.plugin_manager.get_user_plugins(user_id):
        print(f"  - {p.name} v{p.version}：{p.description}")

    # 调用插件
    print("\n" + "=" * 80)
    user_input = "@todo add_todo content=明天下午3点开会 due_date=2026-05-03"
    result = await agent.process_user_message(user_id, user_input)
    print(f"\n{result}")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 📌 今日要点

- ✅ 理解了系统发展的三个阶段：全自研→开放 API→插件生态
- ✅ 掌握了插件系统设计的三大核心：接口设计、权限设计、沙箱设计
- ✅ 学会了开发者接入流程、审核流程、插件市场设计
- ✅ 实现了一个完整可用的插件系统

---

## 自测题

1. 系统发展的三个阶段分别是什么？为什么插件生态能让你的系统能力增长 100 倍？
2. 插件接口设计的三个原则是什么？
3. 插件权限设计的核心原则是什么？四个权限等级分别是什么？
4. 沙箱隔离的四个层级分别是什么？

## 动手作业

1. 把上面的插件系统跑起来
2. 加一个新的插件：天气插件
3. 给天气插件加 NETWORK 权限校验
