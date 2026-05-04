# Day 23：Computer Use 与浏览器自动化 Agent

> 让 Agent 像人一样"看"屏幕、"点"按钮、"输入"文字。本课学习 browser_use 框架、Playwright 驱动的浏览器自动化、以及视觉驱动的 UI 交互技术。

## Part 1：什么是 Computer Use？

### 传统 Agent 的局限

之前的 Agent 都是通过 API 与系统交互：

```plaintext
Agent → API 调用 → 获取 JSON → 解析 → 执行操作
```

但很多系统没有 API，或者 API 不完整：

```
场景：你的 Agent 需要操作一个内部 OA 系统
✗ 没有 API
✗ 没有 MCP 服务器
✗ 只有网页界面
✓ 只能像人一样用浏览器操作
```

### Computer Use 的定义

**Computer Use** = Agent 像人类一样使用计算机界面，通过视觉识别和 GUI 操作完成任务。

| 能力 | 说明 |
|------|------|
| **屏幕理解** | "看"屏幕上的内容，识别按钮、输入框、文字 |
| **鼠标操作** | 点击、双击、右键、拖拽 |
| **键盘输入** | 打字、快捷键 |
| **导航** | 滚动、切换标签页、前进/后退 |

### Computer Use 的技术层级

```
第 3 层：框架层（browser_use、Playwright）
    框架封装底层细节，提供高级 API
第 2 层：协议层（CDP、WebDriver）
    浏览器提供的标准控制协议
第 1 层：系统层（操作系统接口）
    鼠标、键盘、屏幕截图
```

### 2025-2026 年重大突破：模型原生 Computer Use

Computer Use 在 2025-2026 年发生了质的飞跃——从"框架驱动"进化为"模型原生"：

**Google Gemini 2.5 Computer Use（2025年10月）**
- 首个专为界面控制设计的模型
- 在多项网页和移动端控制基准测试中超越竞争对手
- 延迟更低，支持多模态感知（屏幕截图 + DOM 树）
- 可构建自动填表、网页导航、UI 测试等 Agent 应用

**OpenAI CUA（Computer-Using Agent，2025年）**
- 2025年发布的里程碑式能力
- 将 AI 从"被动响应者"转变为"主动执行者"
- 核心能力：屏幕截图感知 → 推理 → 操作（点击/输入）
- 通过 Playwright 脚本 + 屏幕截图识别实现闭环

```python
# OpenAI CUA 的核心工作流
"""
1. 感知：AI "看"屏幕截图，理解当前界面状态
2. 推理：判断"下一步该做什么"（点击哪个按钮/输入什么）
3. 执行：通过 Playwright/CDP 执行操作
4. 评估：再次截图，验证操作结果
5. 循环：重复直到任务完成
"""

# 这代表着 AI 系统从"被动响应"到"主动执行"的根本性转变
```

**国产动态：** 各大厂商也在跟进原生 GUI Agent 能力，部分国产模型已具备基础的多模态屏幕理解与操作能力。

---

## Part 2：技术方案对比

### 主流方案

| 方案 | 原理 | 优势 | 劣势 | 适用场景 |
|------|------|------|------|----------|
| **browser_use** | AI驱动的浏览器代理框架 | 开箱即用，LLM集成好 | 依赖大模型API | 通用网页自动化 |
| **Playwright** | 微软的浏览器自动化 | 稳定可靠，跨浏览器 | 需要手写选择器 | 测试、数据采集 |
| **CDP（Chrome DevTools Protocol）** | Chrome 原生调试协议 | 最底层，控制力最强 | 使用复杂 | 定制化需求 |
| **Selenium** | 老牌浏览器自动化 | 生态成熟 | 速度慢，维护成本高 | 兼容性要求高 |

### browser_use vs Playwright

| 对比维度 | browser_use | Playwright |
|---------|-------------|------------|
| **AI 集成** | 原生支持，LLM 驱动 | 无，需要自己实现 |
| **元素定位** | 视觉识别 + LLM 推理 | CSS 选择器 / XPath |
| **使用难度** | ⭐⭐（高级封装） | ⭐⭐⭐（需编程） |
| **灵活性** | 中 | 高 |
| **稳定性** | 依赖 LLM 质量 | 稳定可靠 |
| **学习成本** | 低 | 中 |

```
选择建议：
- 需要快速实现 AI 网页操作 → browser_use
- 需要稳定可靠的数据采集 → Playwright
- 需要深度定制 → CDP
- 新手入门 → browser_use，再学 Playwright
```

---

## Part 3：browser_use 框架实战

### 安装和基本使用

```bash
pip install browser-use
playwright install  # 安装浏览器
```

### 最简单的例子

```python
from browser_use import Agent
import asyncio


async def main():
    agent = Agent(
        task="打开百度，搜索'AI Agent 教程'，返回搜索结果标题",
        llm="glm-4-flash",  # 或使用其他模型
    )

    result = await agent.run()
    print(result)


asyncio.run(main())
```

### 带配置的 Agent

```python
from browser_use import Agent, BrowserConfig, Controller
from langchain_openai import ChatOpenAI
import asyncio


async def advanced_example():
    # 浏览器配置
    browser_config = BrowserConfig(
        headless=False,          # 显示浏览器窗口
        viewport_width=1280,     # 窗口宽度
        viewport_height=720,     # 窗口高度
        disable_security=True,   # 关闭安全限制
    )

    # LLM 配置
    llm = ChatOpenAI(
        model="glm-4-flash",
        temperature=0.0,  # 降低随机性，提高稳定性
    )

    # 创建 Agent
    agent = Agent(
        task="""
        完成以下步骤：
        1. 打开 GitHub 登录页面
        2. 输入用户名和密码
        3. 检查是否出现错误提示
        4. 截取当前屏幕
        """,
        llm=llm,
        browser_config=browser_config,
        use_vision=True,  # 启用视觉识别
    )

    result = await agent.run()
    print(f"结果: {result}")


asyncio.run(advanced_example())
```

### browser_use 的核心原理

```python
"""
browser_use 的工作流程：

1. 截图 → LLM 分析当前页面
2. LLM 决定下一步操作（点击/输入/滚动等）
3. 执行操作
4. 再次截图 → 评估结果
5. 循环直到任务完成
"""

# 简化的实现逻辑
class SimpleBrowserAgent:
    """简化的 browser_use 核心逻辑"""

    def __init__(self, llm):
        self.llm = llm
        self.history = []

    async def step(self, screenshot, page_source) -> str:
        """每一步：看屏幕 → 决定操作 → 执行"""
        # 1. 构建 prompt（含截图和页面信息）
        prompt = f"""
        当前屏幕内容（通过截图可见）。
        页面 HTML: {page_source[:1000]}

        历史操作：{self.history[-3:]}

        请决定下一步操作，格式：
        ACTION: click/type/scroll/wait
        TARGET: 元素描述或坐标
        VALUE: 输入的值（如果是 type 操作）
        """

        # 2. LLM 决定
        decision = self.llm.invoke(prompt)
        self.history.append(decision)

        # 3. 解析并执行操作
        action = self._parse_action(decision)
        await self._execute(action)

        return action

    def _parse_action(self, decision: str) -> dict:
        """解析 LLM 的决策"""
        lines = decision.strip().split("\n")
        action = {}
        for line in lines:
            if line.startswith("ACTION:"):
                action["type"] = line.replace("ACTION:", "").strip()
            elif line.startswith("TARGET:"):
                action["target"] = line.replace("TARGET:", "").strip()
            elif line.startswith("VALUE:"):
                action["value"] = line.replace("VALUE:", "").strip()
        return action

    async def _execute(self, action: dict):
        """执行操作"""
        print(f"执行: {action['type']} → {action.get('target', '')}")
        # 实际会调用 Playwright 执行
```

---

## Part 4：Playwright 基础

### 安装

```bash
pip install playwright
playwright install chromium  # 安装 Chromium 浏览器
playwright install firefox   # 安装 Firefox
playwright install webkit    # 安装 WebKit
```

### 基本操作

```python
from playwright.async_api import async_playwright
import asyncio


async def basic_playwright():
    async with async_playwright() as p:
        # 启动浏览器
        browser = await p.chromium.launch(headless=False)
        context = await browser.new_context(
            viewport={"width": 1280, "height": 720}
        )
        page = await context.new_page()

        # 导航
        await page.goto("https://www.baidu.com")

        # 截图
        await page.screenshot(path="baidu.png")

        # 输入文字
        await page.fill("#kw", "AI Agent")

        # 点击按钮
        await page.click("#su")

        # 等待加载
        await page.wait_for_load_state("networkidle")

        # 提取文本
        results = await page.text_content_all("h3")
        print(f"搜索结果: {results[:3]}")

        # 关闭
        await browser.close()


asyncio.run(basic_playwright())
```

### 元素定位策略

| 策略 | 方法 | 示例 | 适用场景 |
|------|------|------|----------|
| **CSS 选择器** | `page.fill("#id")` | `#username` | 有稳定的 ID/Class |
| **XPath** | `page.locator("xpath=...")` | `//button[text()="登录"]` | 复杂匹配条件 |
| **文本匹配** | `page.get_by_text("登录")` | 按可见文本定位 | 按钮、链接 |
| **角色定位** | `page.get_by_role("button")` | ARIA 角色 | 无障碍友好的页面 |
| **Placeholder** | `page.get_by_placeholder("请输入")` | input 的 placeholder | 输入框 |

```python
# 多种定位方式
async def locate_elements(page):
    # CSS 选择器
    await page.click("#submit-btn")
    await page.fill("input[name='username']", "admin")

    # 文本匹配
    await page.get_by_text("确认删除").click()

    # Placeholder
    await page.get_by_placeholder("请输入搜索关键词").fill("AI")

    # 多个匹配（取第一个）
    buttons = page.get_by_role("button")
    count = await buttons.count()
    print(f"页面有 {count} 个按钮")

    # 链式定位
    modal = page.locator(".modal")
    await modal.locator(".confirm-btn").click()
```

---

## Part 5：CDP（Chrome DevTools Protocol）

### CDP 是什么？

CDP 是 Chrome 浏览器提供的原生调试协议，Playwright 和 browser_use 底层都基于它。

```python
"""
CDP 的核心能力：

1. DOM 操作：获取/修改页面元素
2. 网络拦截：监控/修改网络请求
3. 屏幕截图：截取页面或元素
4. JavaScript 执行：在页面中运行任意 JS
5. 性能监控：CPU、内存、网络性能
"""
```

### 直接使用 CDP

```python
import asyncio
from playwright.async_api import async_playwright


async def cdp_example():
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=False)

        # 创建 CDP 会话
        page = await browser.new_page()
        cdp_session = await context.new_cdp_session(page)

        # 通过 CDP 执行高级操作
        # 1. 捕获网络请求
        await cdp_session.send("Network.enable")
        cdp_session.on("Network.requestWillBeSent", lambda params:
            print(f"请求: {params['request']['url']}")
        )

        # 2. 执行 JavaScript
        result = await cdp_session.send("Runtime.evaluate", {
            "expression": "document.title"
        })
        print(f"页面标题: {result['result']['value']}")

        # 3. 拦截网络请求
        await cdp_session.send("Network.setRequestInterception", {
            "patterns": [{"urlPattern": "*"}]
        })

        await page.goto("https://example.com")
        await browser.close()


# Playwright 封装的高级 CDP 功能
async def advanced_cdp_operations(page):
    """通过 CDP 实现的高级功能"""

    # 1. 获取完整的可访问性树
    accessibility_tree = await page.accessibility.snapshot()
    print(f"可访问性树节点数: {len(accessibility_tree)}")

    # 2. 模拟地理定位
    await page.context.grant_permissions(["geolocation"])
    await page.set_geolocation({"latitude": 39.9, "longitude": 116.4})

    # 3. 模拟网络条件
    await page.context.set_offline(True)  # 离线模式
    await page.context.set_offline(False)

    # 4. 拦截和修改响应
    await page.route("**/api/**", lambda route: route.fulfill(
        status=200,
        content_type="application/json",
        body='{"mock": true}'
    ))
```

---

## Part 6：Computer Use 实战场景

### 场景 1：自动化表单填写

```python
from playwright.async_api import async_playwright
import asyncio


class FormFiller:
    """智能表单填写器"""

    def __init__(self):
        self.browser = None
        self.page = None

    async def start(self):
        p = await async_playwright().start()
        self.browser = await p.chromium.launch(headless=False)
        self.page = await self.browser.new_page()

    async def fill_form(self, url: str, data: dict):
        """自动识别并填写表单"""
        await self.page.goto(url)

        # 1. 获取所有输入框
        inputs = await self.page.query_selector_all(
            "input, textarea, select"
        )

        for input_el in inputs:
            # 2. 获取输入框的标识信息
            name = await input_el.get_attribute("name") or ""
            placeholder = await input_el.get_attribute("placeholder") or ""
            input_type = await input_el.get_attribute("type") or "text"

            # 3. 智能匹配数据
            value = self._match_field(name, placeholder, data)
            if value:
                await input_el.fill(str(value))
                print(f"填写 {name or placeholder}: {value}")

        # 4. 点击提交按钮
        submit = await self.page.query_selector(
            "button[type='submit'], input[type='submit']"
        )
        if submit:
            await submit.click()
            print("已提交表单")

    def _match_field(self, name: str, placeholder: str, data: dict) -> str:
        """智能匹配字段名到数据"""
        name_lower = (name + " " + placeholder).lower()

        mapping = {
            "name": ["name", "姓名", "username"],
            "phone": ["phone", "tel", "mobile", "手机", "电话"],
            "email": ["email", "mail", "邮箱"],
            "address": ["address", "addr", "地址"],
            "note": ["note", "remark", "备注", "memo"],
            "company": ["company", "corp", "公司", "企业"],
        }

        for data_key, keywords in mapping.items():
            if any(k in name_lower for k in keywords):
                return data.get(data_key, "")

        return ""

    async def stop(self):
        await self.browser.close()


# 使用
async def demo():
    filler = FormFiller()
    await filler.start()
    await filler.fill_form("https://example.com/form", {
        "name": "张三",
        "phone": "13800138000",
        "email": "zhangsan@example.com",
        "address": "北京市海淀区"
    })
    await filler.stop()


asyncio.run(demo())
```

### 场景 2：监控面板数据采集

```python
class DashboardMonitor:
    """监控面板数据采集 Agent"""

    def __init__(self, dashboard_url: str):
        self.url = dashboard_url
        self.page = None

    async def collect_metrics(self) -> dict:
        """采集监控面板数据"""
        async with async_playwright() as p:
            browser = await p.chromium.launch(headless=True)
            page = await browser.new_page()
            await page.goto(self.url, wait_until="networkidle")

            # 截图存档
            await page.screenshot(path="dashboard_snapshot.png")

            # 提取数值型指标
            metrics = await page.evaluate("""
                () => {
                    const items = document.querySelectorAll('.metric-value');
                    return Array.from(items).map(el => ({
                        name: el.dataset.metric || el.textContent.trim(),
                        value: parseFloat(el.textContent.replace(/[,%]/g, ''))
                    }));
                }
            """)

            # 提取表格数据
            table_data = await page.evaluate("""
                () => {
                    const rows = document.querySelectorAll('table tbody tr');
                    return Array.from(rows).map(row =>
                        Array.from(row.querySelectorAll('td'))
                            .map(td => td.textContent.trim())
                    );
                }
            """)

            await browser.close()

            return {
                "metrics": metrics,
                "table": table_data,
                "timestamp": asyncio.get_event_loop().time()
            }
```

### 场景 3：视觉驱动的 UI 测试

```python
from PIL import Image
import io
import numpy as np


class VisualTestAgent:
    """视觉驱动的 UI 测试 Agent"""

    def __init__(self, llm):
        self.llm = llm
        self.page = None

    async def assert_visual(self, description: str) -> bool:
        """通过视觉检查页面状态"""
        screenshot = await self.page.screenshot(full_page=True)

        prompt = f"""
        检查当前页面截图，验证以下断言是否为真：
        "{description}"

        请回答：TRUE 或 FALSE，并说明理由。
        """

        # 将截图传给 LLM 分析
        result = self.llm.invoke([
            {"role": "user", "content": [
                {"type": "text", "text": prompt},
                {"type": "image", "image": screenshot}
            ]}
        ])

        return "TRUE" in result.content.upper()

    async def find_and_click(self, target_description: str) -> bool:
        """根据描述找到并点击元素"""
        screenshot = await self.page.screenshot()

        prompt = f"""
        在截图中的网页上，找到 "{target_description}"。
        返回它的坐标 (x, y) 或描述它的位置。
        """

        result = self.llm.invoke([
            {"role": "user", "content": [
                {"type": "text", "text": prompt},
                {"type": "image", "image": screenshot}
            ]}
        ])

        # 解析坐标并点击
        # 实际实现需要从 LLM 回复中提取坐标
        return True

    async def verify_layout(self, expected_layout: str) -> dict:
        """验证页面布局"""
        screenshot = await self.page.screenshot()

        prompt = f"""
        分析截图中的页面布局，检查是否符合以下要求：
        {expected_layout}

        返回 JSON：
        {{
            "passed": true/false,
            "issues": ["问题1", "问题2"]
        }}
        """

        result = self.llm.invoke([
            {"role": "user", "content": [
                {"type": "text", "text": prompt},
                {"type": "image", "image": screenshot}
            ]}
        ])

        return json.loads(result.content)
```

---

## Part 7：最佳实践与注意事项

### 稳定性策略

```python
class RobustBrowserAgent:
    """健壮的浏览器 Agent"""

    def __init__(self):
        self.max_retries = 3
        self.timeout = 30000  # 30秒

    async def click_with_retry(self, page, selector: str):
        """带重试的点击"""
        for i in range(self.max_retries):
            try:
                await page.click(selector, timeout=self.timeout)
                return True
            except Exception as e:
                print(f"点击失败 (第{i+1}次): {e}")
                await page.wait_for_timeout(1000)  # 等1秒重试
        return False

    async def wait_and_fill(self, page, selector: str, value: str):
        """等待元素出现后再填写"""
        try:
            # 等待元素可见
            await page.wait_for_selector(
                selector,
                state="visible",
                timeout=self.timeout
            )
            # 清空并填写
            await page.fill(selector, "")
            await page.fill(selector, value)
            return True
        except Exception as e:
            print(f"填写失败: {e}")
            return False

    async def safe_navigate(self, page, url: str):
        """安全导航"""
        try:
            await page.goto(url, wait_until="domcontentloaded")
            # 额外等待页面完全加载
            await page.wait_for_load_state("networkidle", timeout=10000)
            return True
        except Exception:
            # 即使超时，页面可能已经可用
            return True
```

### 常见问题与解决

| 问题 | 原因 | 解决方法 |
|------|------|----------|
| **元素未找到** | 页面没加载完 / 动态渲染 | 增加等待时间，使用 `wait_for_selector` |
| **点击无效** | 元素被遮挡 / 不在视口 | 先滚动到元素，再用 JS 点击 |
| **截图为空** | headless 模式渲染问题 | 设置 viewport，等待 fonts 加载 |
| **LLM 误判** | 截图模糊 / 页面复杂 | 提高截图质量，拆分任务 |
| **反爬检测** | 网站检测到自动化工具 | 修改 User-Agent，模拟人类操作 |

```python
# 反检测配置
async def create_stealth_browser():
    """创建不易被检测的浏览器"""
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=False)

        context = await browser.new_context(
            user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                       "AppleWebKit/537.36 (KHTML, like Gecko) "
                       "Chrome/120.0.0.0 Safari/537.36",
            viewport={"width": 1280, "height": 720},
            locale="zh-CN",
            timezone_id="Asia/Shanghai",
            # 注入反检测脚本
        )

        # 覆盖 webdriver 属性
        await context.add_init_script("""
            Object.defineProperty(navigator, 'webdriver', {
                get: () => undefined
            });
        """)

        return browser, context
```

### 性能优化

| 策略 | 效果 | 实现 |
|------|------|------|
| **无头模式** | 节省 30% 资源 | `headless=True` |
| **禁用图片** | 加快加载 50% | 拦截图片请求 |
| **复用浏览器** | 避免重复启动 | 保持 browser 实例 |
| **并行操作** | 多页面同时处理 | 多个 context |
| **缓存** | 减少重复加载 | 持久化缓存目录 |

```python
# 性能优化配置
async def optimized_browser():
    """高性能浏览器配置"""
    async with async_playwright() as p:
        browser = await p.chromium.launch(
            headless=True,
            args=[
                "--disable-gpu",
                "--disable-dev-shm-usage",
                "--no-sandbox",
            ]
        )

        context = await browser.new_context(
            viewport={"width": 1280, "height": 720},
            # 禁用图片和字体加载
            extra_http_headers={
                "Accept-Language": "zh-CN,zh;q=0.9"
            }
        )

        # 拦截不必要的资源
        async def block_resources(route):
            if route.request.resource_type in ["image", "font", "media"]:
                await route.abort()
            else:
                await route.continue_()

        page = await context.new_page()
        await page.route("**/*", block_resources)

        return browser, context
```

---

## 今日要点

- 理解了 Computer Use 的概念——让 Agent 像人一样操作界面
- 掌握了 browser_use 框架的基本使用
- 学会了 Playwright 的核心操作
- 理解了 CDP 协议的能力和应用场景
- 实现了表单填写、数据采集、视觉测试等实战场景
- 掌握了稳定性、反检测、性能优化等最佳实践

---

## 自测题

1. Computer Use 和传统 API 调用方式的核心区别是什么？
2. browser_use 框架的工作原理是什么？
3. Playwright 的四种主要元素定位策略是什么？
4. CDP 的核心能力有哪些？
5. 浏览器自动化的三大稳定性策略是什么？

---

## 动手作业

1. 使用 browser_use 实现一个自动搜索并获取结果的 Agent
2. 用 Playwright 实现一个自动登录多个网站的工具
3. 实现一个视觉驱动的 UI 测试 Agent，验证页面元素是否存在
4. （进阶）实现一个能自动填写表单并提交的智能 Agent