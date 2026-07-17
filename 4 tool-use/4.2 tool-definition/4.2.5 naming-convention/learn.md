# 工具命名规范与语义

## 简单介绍

工具命名看起来是个表面问题，但它对 LLM 的工具有显著影响。LLM 在训练数据中见过大量代码和 API 命名，形成了对命名模式的隐式理解。好的命名让 LLM "凭直觉"就能选对工具。

## 基本原理

LLM 本质上是一个巨大的模式匹配器。它见过的 API 命名越多，对命名模式的理解就越深：

| 命名风格 | LLM 理解模式 | 效果 |
|----------|-------------|------|
| `get_weather` | "这是一个获取数据的函数" | 准确 |
| `weather_query` | "和天气相关的查询" | 较明确 |
| `performWeatherDataRetrieval` | 不太自然的命名 | 一般 |
| `wd` | 缩写 | 很差 |

## 命名原则

### 1. 动词 + 名词 结构

最清晰且 LLM 最容易识别的命名模式：

```
get_user           → 获取用户
send_email         → 发送邮件
create_order       → 创建订单
delete_product     → 删除产品
search_articles    → 搜索文章
```

### 2. 使用高频动词

LLM 训练数据中常见的高频 API 动词：

| 动词 | 含义 | 适用场景 |
|------|------|----------|
| get/fetch/retrieve | 查询获取 | 读操作、查询 |
| search/find/lookup | 搜索查找 | 模糊搜索 |
| create/add/new | 创建 | 新增资源 |
| update/edit/modify | 更新 | 修改资源 |
| delete/remove | 删除 | 移除资源 |
| send/post/notify | 发送 | 通知、推送 |
| validate/check/verify | 校验 | 验证操作 |
| calculate/compute | 计算 | 数据处理 |

### 3. 避免的命名模式

```
❌ 缩写: gU → get_user（LLM 不懂）
❌ 无意义命名: doSomething → search_database（语义不清）
❌ 过长的命名: getUserInformationFromDatabaseByQuery → search_users（简洁明确）
❌ 模棱两可: processData → (太宽泛，不知道做什么)
```

### 4. 一致性 > 完美性

一旦选择了某种命名约定，在整个工具集中保持一致：

```
✅ 一致性示例:
  get_user / get_order / get_product  ← 同一模式

❌ 不一致示例:
  get_user / fetch_order_data / retrieveProductInfo  ← 三种模式混用
```

## 参考：常见命名模式

| 领域 | 推荐命名 | 避免 |
|------|---------|------|
| 用户管理 | get_user, create_user, update_user, delete_user | userInfo, manageUser |
| 订单系统 | get_order, create_order, cancel_order, list_orders | ord, orderProcess |
| 内容检索 | search_content, get_article, list_categories | cntSearch, content |
| 通知 | send_notification, list_notifications, mark_read | notif, pushMsg |

## 命名与描述的关系

命名和描述是互补的：

```
名称: get_weather       ← 快速匹配（LLM 的第一印象）
描述: 获取指定城市的当前天气数据  ← 精细匹配（LLM 的二次确认）

名称: send_email        ← 一看就知道是发邮件
描述: 发送电子邮件给指定收件人，支持 HTML 模板和附件  ← 说明具体能力
```

## 最佳实践

1. **命名用英文**：大多数 LLM 的编程语料以英文为主，英文命名表现更好
2. **使用下划线分隔**：`snake_case` 在 LLM 训练数据中比 `camelCase` 更常见于函数名
3. **不要使用数字后缀**：`get_user_2` 这种命名会让 LLM 困惑
4. **避免否定命名**：`do_not_send_notification` 不如用 `disable_notification` 参数
