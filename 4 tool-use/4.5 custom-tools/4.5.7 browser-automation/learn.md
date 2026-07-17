# 浏览器自动化与视觉操作（Playwright、截图理解）

## 简单介绍

浏览器自动化工具赋予 Agent 操控网页浏览器能力——导航、点击、填写表单、截图、读取页面内容。这使得 Agent 能够使用那些没有 API 的 Web 应用，或者模拟用户操作流程。

## 主要工具集

```python
browser_tools = [
    {
        "name": "browser_navigate",
        "description": "导航到指定 URL",
        "parameters": {"url": {"type": "string"}}
    },
    {
        "name": "browser_click",
        "description": "点击页面上的元素",
        "parameters": {"selector": {"type": "string", "description": "CSS 选择器"}}
    },
    {
        "name": "browser_fill",
        "description": "填写输入框",
        "parameters": {
            "selector": {"type": "string"},
            "value": {"type": "string"}
        }
    },
    {
        "name": "browser_screenshot",
        "description": "截取当前页面截图，用于视觉理解",
        "parameters": {"full_page": {"type": "boolean"}}
    },
    {
        "name": "browser_get_text",
        "description": "获取页面上的文本内容"
    },
    {
        "name": "browser_evaluate",
        "description": "在浏览器中执行 JavaScript"
    }
]
```

## 基本原理

浏览器自动化的核心循环是"观察-决策-行动"：

```
Agent 思考当前任务
  ↓
截图或获取页面 HTML → 理解当前页面状态
  ↓
决策下一步操作（点击哪个按钮、填什么内容）
  ↓
执行操作（click / fill / navigate）
  ↓
等待页面加载完成
  ↓
截图验证操作结果
  ↓
继续循环直到任务完成
```

## 视觉理解集成

现代浏览器自动化结合了**截图理解**（Vision-Language Model）：

```python
async def browser_act(agent, task: str):
    """基于视觉理解的浏览器操作"""
    # 1. 截取当前页面截图
    screenshot = await page.screenshot()
    
    # 2. 让多模态 LLM 理解页面并决定下一步
    response = await llm.understand_screenshot(
        screenshot=screenshot,
        task=task,
        available_actions=["click", "type", "scroll", "navigate"]
    )
    
    # 3. 执行 LLM 建议的操作
    action = response.decided_action
    await execute_action(page, action)
```

## 等待策略

浏览器操作最大的挑战是**时序**——操作了之后页面可能还没加载完：

```python
async def click_and_wait(page, selector: str):
    """点击并智能等待页面加载"""
    await page.click(selector)
    # 等待策略组合
    try:
        await page.wait_for_load_state("networkidle", timeout=5000)
    except:
        pass
    try:
        await page.wait_for_load_state("domcontentloaded", timeout=3000)
    except:
        pass
    await asyncio.sleep(0.5)  # 额外保险等待
```

## 与 Playwright 集成

```python
from playwright.async_api import async_playwright

class PlaywrightTool:
    async def __init__(self):
        self.playwright = await async_playwright().start()
        self.browser = await self.playwright.chromium.launch(
            headless=True,
            args=["--no-sandbox"]
        )
        self.context = await self.browser.new_context(
            viewport={"width": 1280, "height": 720},
            user_agent="Mozilla/5.0 (Agent)"
        )
        self.page = await self.context.new_page()
    
    async def navigate(self, url: str):
        await self.page.goto(url, wait_until="networkidle")
        return {"title": await self.page.title(), "url": self.page.url}
    
    async def screenshot(self):
        # 返回 Base64 编码的截图
        return base64.b64encode(await self.page.screenshot()).decode()
    
    async def get_text(self):
        return await self.page.evaluate("document.body.innerText")
```

## 核心挑战

| 挑战 | 说明 | 缓解方案 |
|------|------|----------|
| 页面加载 | 操作后页面可能未完全渲染 | 智能等待 + 重试 |
| 动态内容 | SPA、AJAX 异步加载 | 等待特定元素出现 |
| 登录状态 | 需要登录才能操作的站点 | Session 持久化 |
| 反爬机制 | 网站检测到自动化操作 | 模拟人类操作模式 |
| 故障恢复 | 操作失败后如何继续 | 截图检查 + 重试 |
| 选择器稳定性 | CSS 选择器可能因页面更新失效 | 多种选择器策略 |

## 安全考量

1. **不应该访问的页面**：内网地址、本地文件
2. **不应该执行的操作**：自动提交表单、自动支付
3. **隐私保护**：截图内容可能包含敏感信息
4. **操作审计**：浏览器的每一次操作都要记录

## 最佳实践

- 优先使用 API（如果有），浏览器自动化是"最后手段"
- 每次操作后检查页面状态——确认操作生效了再继续
- 设置页面超时（30s），防止无限等待
- 对需要登录的站点，提前配置好 Session 或 Cookie，不要让 Agent 操作登录表单
- 监控自动化操作的失败率，对频繁失败的页面进行人工审查
