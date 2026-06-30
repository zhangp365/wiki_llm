---
title: "Claude 高级工具使用"
created: 2026-05-05
updated: 2026-05-05
type: concept
tags: [agent, tool-use, anthropic]
sources: [raw/articles/anthropic-3-advanced-tool-use.md]
original: https://www.anthropic.com/engineering/advanced-tool-use
---

# Claude 高级工具使用：Tool Search Tool、Programmatic Tool Calling 和 Tool Use Examples

> 原文链接: [English Original](https://www.anthropic.com/engineering/advanced-tool-use)

- Anthropic 工程博客

## Claude 开发者平台上的高级工具使用

发布于 2025年11月24日

我们添加了三个新的 beta 功能，让 Claude 能够动态地发现、学习和执行工具。以下是它们的工作原理。

AI Agent 的未来是模型无缝地跨数百或数千个工具工作的场景。一个集成 git 操作、文件操作、包管理器、测试框架和部署流水线的 IDE 助手。一个同时连接 Slack、GitHub、Google Drive、Jira、公司数据库和数十个 MCP server 的运营协调器。

要构建有效的 Agent，它们需要与无限的工具库协作，而无需预先将每个定义都塞入上下文。我们关于将代码执行与 MCP 结合使用的博客文章讨论了工具结果和定义有时在 Agent 读取请求之前就消耗了 50,000+ token 的情况。Agent 应该按需发现和加载工具，只保留与当前任务相关的内容。

Agent 还需要从代码中调用工具的能力。使用自然语言工具调用时，每次调用都需要完整的推理过程，中间结果无论是否有用都会在上下文中堆积。代码是编排逻辑（如循环、条件语句和数据转换）的自然选择。Agent 需要根据任务灵活选择代码执行和推理。

Agent 还需要从示例中学习正确的工具使用方法，而不仅仅是 schema 定义。JSON schema 定义了结构上有效的配置，但无法表达使用模式：何时包含可选参数、哪些组合是合理的，或者你的 API 期望什么约定。

今天，我们发布三个使这一切成为可能的功能：

- **Tool Search Tool**，允许 Claude 使用搜索工具访问数千个工具而不消耗其 context window
- **Programmatic Tool Calling**，允许 Claude 在代码执行环境中调用工具，减少对模型 context window 的影响
- **Tool Use Examples**，提供了用于演示工具使用的通用标准

在内部测试中，我们发现这些功能帮助我们构建了使用传统工具使用模式不可能实现的东西。例如，Claude for Excel 使用 Programmatic Tool Calling 读取和修改数千行的电子表格，而不会使模型的 context window 过载。

基于我们的经验，我们相信这些功能为使用 Claude 构建的功能开辟了新的可能性。

## Tool Search Tool

### 挑战

MCP 工具定义提供了重要的上下文，但随着更多 server 连接，这些 token 会累积。考虑一个五个 server 的设置：
- GitHub：35 个工具（~26K token）
- Slack：11 个工具（~21K token）
- Sentry：5 个工具（~3K token）
- Grafana：5 个工具（~3K token）
- Splunk：2 个工具（~2K token）

这在对话开始前就有 58 个工具消耗约 55K token。添加更多 server 如 Jira（仅它就使用 ~17K token），你很快就接近 100K+ token 的开销。在 Anthropic，我们曾看到工具定义在优化前消耗了 134K token。

但 token 成本不是唯一的问题。最常见的失败是错误的工具选择和不正确的参数，特别是当工具有类似名称时，如 `notification-send-user` vs. `notification-send-channel`。

### 我们的解决方案

与其预先加载所有工具定义，Tool Search Tool 按需发现工具。Claude 只看到当前任务实际需要的工具。Tool Search Tool 相比 Claude 的传统方法保留了 191,300 token 的上下文（传统方法为 122,800）。

**传统方法：**
- 预先加载所有工具定义（50+ MCP 工具约 ~72K token）
- 对话历史和系统 prompt 竞争剩余空间
- 在任何工作开始前总上下文消耗：~77K token

**使用 Tool Search Tool：**
- 仅预先加载 Tool Search Tool（~500 token）
- 按需发现工具（3-5 个相关工具，~3K token）
- 总上下文消耗：~8.7K token，保留 95% 的 context window

这代表了 token 使用量减少 85%，同时保持对完整工具库的访问。内部测试显示，在使用大型工具库时，MCP 评估的准确性显著提高。Opus 4 从 49% 提升到 74%，Opus 4.5 从 79.5% 提升到 88.1%。

### Tool Search Tool 的工作原理

Tool Search Tool 让 Claude 动态发现工具，而不是预先加载所有定义。你向 API 提供所有工具定义，但用 `defer_loading: true` 标记工具使其可按需发现。延迟加载的工具最初不会加载到 Claude 的上下文中。Claude 只看到 Tool Search Tool 本身加上任何 `defer_loading: false` 的工具（你最重要、最常用的工具）。

当 Claude 需要特定功能时，它搜索相关工具。Tool Search Tool 返回匹配工具的引用，这些引用会在 Claude 的上下文中扩展为完整定义。

例如，如果 Claude 需要与 GitHub 交互，它搜索"github"，只有 `github.createPullRequest` 和 `github.listIssues` 会被加载——而不是你来自 Slack、Jira 和 Google Drive 的其他 50+ 工具。

这样，Claude 可以访问你的完整工具库，同时只为它实际需要的工具支付 token 成本。

**Prompt caching 说明：** Tool Search Tool 不会破坏 prompt caching，因为延迟加载的工具完全从初始 prompt 中排除。它们只在 Claude 搜索它们之后才被添加到上下文中，所以你的系统 prompt 和核心工具定义保持可缓存。

**实现：**

```json
{
  "tools": [
    // 包含一个工具搜索工具（regex、BM25 或自定义）
    {"type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex"},
    
    // 标记工具以按需发现
    {
      "name": "github.createPullRequest",
      "description": "Create a pull request",
      "input_schema": {...},
      "defer_loading": true
    }
    // ... 数百个更多延迟加载的工具，设置 defer_loading: true
  ]
}
```

对于 MCP server，你可以延迟加载整个 server，同时保持特定高使用率工具的加载：

```json
{
  "type": "mcp_toolset",
  "mcp_server_name": "google-drive",
  "default_config": {"defer_loading": true},
  "configs": {
    "search_files": {
      "defer_loading": false
    }
  }
}
```

Claude 开发者平台提供了开箱即用的基于 regex 和 BM25 的搜索工具，但你也可以使用 embeddings 或其他策略实现自定义搜索工具。

### 何时使用 Tool Search Tool

像任何架构决策一样，启用 Tool Search Tool 涉及权衡。该功能在工具调用之前添加了一个搜索步骤，所以它在上下文节省和准确性改进超过额外延迟时提供最佳 ROI。

**适合使用时：**
- 工具定义消耗 >10K token
- 遇到工具选择准确性问题
- 构建由多个 server 驱动的 MCP 系统
- 有 10+ 个可用工具

**不太适合时：**
- 小型工具库

## Programmatic Tool Calling

### 挑战

传统工具调用在 workflow 变得复杂时产生两个根本问题：

- **中间结果导致的上下文污染：** 当 Claude 分析 10MB 日志文件中的错误模式时，整个文件进入其 context window，尽管 Claude 只需要错误频率的摘要。当跨多个表获取客户数据时，每条记录无论是否相关都会在上下文中累积。这些中间结果消耗大量 token 预算，可能将重要信息完全推出 context window。
- **推理开销和手动综合：** 每次工具调用都需要完整的模型推理过程。收到结果后，Claude 必须"目测"数据以提取相关信息，推理各部分如何关联，并决定下一步做什么——全部通过自然语言处理。一个五个工具的 workflow 意味着五次推理过程加上 Claude 解析每个结果、比较值和综合结论。这既慢又容易出错。

### 我们的解决方案

Programmatic Tool Calling 使 Claude 能够通过代码而非单独的 API 往返来编排工具。Claude 不再逐个请求工具并将每个结果返回到其上下文，而是编写调用多个工具、处理其输出并控制哪些信息实际进入其 context window 的代码。

Claude 擅长编写代码，通过让它用 Python 而非自然语言工具调用表达编排逻辑，你获得了更可靠、精确的控制流。循环、条件语句、数据转换和错误处理在代码中都是显式的，而非隐含在 Claude 的推理中。

### 示例：预算合规检查

考虑一个常见的业务任务："哪些团队成员超过了他们的 Q3 差旅预算？"

你有三个可用工具：
- `get_team_members(department)` - 返回团队成员列表（包含 ID 和级别）
- `get_expenses(user_id, quarter)` - 返回用户的费用明细
- `get_budget_by_level(level)` - 返回员工级别的预算限制

**传统方法：**
- 获取团队成员 → 20 人
- 对每个人获取 Q3 费用 → 20 次工具调用，每次返回 50-100 条明细（机票、酒店、餐饮、收据）
- 按员工级别获取预算限制
- 所有这些进入 Claude 的上下文：2,000+ 条费用明细（50KB+）
- Claude 手动汇总每个人的费用，查找其预算，比较费用与预算限制
- 更多的模型往返，显著的上下文消耗

**使用 Programmatic Tool Calling：**

Claude 编写一个 Python 脚本来编排整个 workflow。脚本在 Code Execution 工具（沙盒环境）中运行，在需要来自工具的结果时暂停。当你通过 API 返回工具结果时，它们由脚本处理而非被模型消费。脚本继续执行，Claude 只看到最终输出。

以下是 Claude 在预算合规任务中的编排代码：

```python
team = await get_team_members("engineering")

# 获取每个唯一级别的预算
levels = list(set(m["level"] for m in team))
budget_results = await asyncio.gather(*[
    get_budget_by_level(level) for level in levels
])

# 创建查找字典：{"junior": budget1, "senior": budget2, ...}
budgets = {level: budget for level, budget in zip(levels, budget_results)}

# 并行获取所有费用
expenses = await asyncio.gather(*[
    get_expenses(m["id"], "Q3") for m in team
])

# 查找超过差旅预算的员工
exceeded = []
for member, exp in zip(team, expenses):
    budget = budgets[member["level"]]
    total = sum(e["amount"] for e in exp)
    if total > budget["travel_limit"]:
        exceeded.append({
            "name": member["name"],
            "spent": total,
            "limit": budget["travel_limit"]
        })

print(json.dumps(exceeded))
```

Claude 的上下文只收到最终结果：超过预算的两三个人。2,000+ 条明细、中间汇总和预算查找不影响 Claude 的上下文，将消耗从 200KB 的原始费用数据减少到仅 1KB 的结果。

效率提升是显著的：
- **Token 节省：** 通过将中间结果排除在 Claude 的上下文之外，PTC 显著减少了 token 消耗。平均使用量从 43,588 降至 27,297 token，在复杂研究任务上减少了 37%。
- **降低延迟：** 每次 API 往返都需要模型推理（数百毫秒到数秒）。当 Claude 在单个代码块中编排 20+ 次工具调用时，你消除了 19+ 次推理过程。API 处理工具执行而无需每次返回模型。
- **提高准确性：** 通过编写显式的编排逻辑，Claude 比在自然语言中处理多个工具结果时犯更少的错误。内部知识检索从 25.6% 提高到 28.5%；GIA 基准从 46.5% 提高到 51.2%。

生产 workflow 涉及杂乱的数据、条件逻辑和需要扩展的操作。Programmatic Tool Calling 让 Claude 以编程方式处理这种复杂性，同时保持其专注于可操作的结果而非原始数据处理。

### Programmatic Tool Calling 的工作原理

#### 1. 将工具标记为可从代码调用

在工具中添加 `code_execution`，并设置 `allowed_callers` 以启用工具的程序化执行：

```json
{
  "tools": [
    {
      "type": "code_execution_20250825",
      "name": "code_execution"
    },
    {
      "name": "get_team_members",
      "description": "Get all members of a department...",
      "input_schema": {...},
      "allowed_callers": ["code_execution_20250825"]
    },
    {
      "name": "get_expenses",
      "..."
    },
    {
      "name": "get_budget_by_level",
      "..."
    }
  ]
}
```

API 将这些工具定义转换为 Claude 可以调用的 Python 函数。

#### 2. Claude 编写编排代码

Claude 不是逐个请求工具，而是生成 Python 代码：

```json
{
  "type": "server_tool_use",
  "id": "srvtoolu_abc",
  "name": "code_execution",
  "input": {
    "code": "team = get_team_members('engineering')\n..."
  }
}
```

#### 3. 工具执行而不影响 Claude 的上下文

当代码调用 `get_expenses()` 时，你会收到一个带有 `caller` 字段的工具请求：

```json
{
  "type": "tool_use",
  "id": "toolu_xyz",
  "name": "get_expenses",
  "input": {"user_id": "emp_123", "quarter": "Q3"},
  "caller": {
    "type": "code_execution_20250825",
    "tool_id": "srvtoolu_abc"
  }
}
```

你提供结果，结果在 Code Execution 环境中处理而非 Claude 的上下文。这个请求-响应循环对代码中的每个工具调用重复。

#### 4. 只有最终输出进入上下文

当代码完成运行时，只有代码的结果返回给 Claude：

```json
{
  "type": "code_execution_tool_result",
  "tool_use_id": "srvtoolu_abc",
  "content": {
    "stdout": "[{\"name\": \"Alice\", \"spent\": 12500, \"limit\": 10000}...]"
  }
}
```

这就是 Claude 看到的全部，而不是沿途处理的 2000+ 条费用明细。

### 何时使用 Programmatic Tool Calling

Programmatic Tool Calling 在你的 workflow 中添加了一个代码执行步骤。当 token 节省、延迟改进和准确性提升显著时，这个额外开销是值得的。

**最有利时：**
- 处理大型数据集且只需要聚合或摘要
- 运行具有三个或更多依赖工具调用的多步骤 workflow
- 在 Claude 看到之前过滤、排序或转换工具结果
- 处理中间数据不应影响 Claude 推理的任务
- 跨多个项目运行并行操作（例如检查 50 个端点）

**不太有利时：**
- 进行简单的单工具调用
- Claude 应该看到并推理所有中间结果的任务
- 运行响应量小的快速查找

## Tool Use Examples

### 挑战

JSON Schema 擅长定义结构——类型、必填字段、允许的枚举——但它无法表达使用模式：何时包含可选参数、哪些组合是合理的，或者你的 API 期望什么约定。

考虑一个支持工单 API：

```json
{
  "name": "create_ticket",
  "input_schema": {
    "properties": {
      "title": {"type": "string"},
      "priority": {"enum": ["low", "medium", "high", "critical"]},
      "labels": {"type": "array", "items": {"type": "string"}},
      "reporter": {
        "type": "object",
        "properties": {
          "id": {"type": "string"},
          "name": {"type": "string"},
          "contact": {
            "type": "object",
            "properties": {
              "email": {"type": "string"},
              "phone": {"type": "string"}
            }
          }
        }
      },
      "due_date": {"type": "string"},
      "escalation": {
        "type": "object",
        "properties": {
          "level": {"type": "integer"},
          "notify_manager": {"type": "boolean"},
          "sla_hours": {"type": "integer"}
        }
      }
    },
    "required": ["title"]
  }
}
```

Schema 定义了什么是有效的，但关键问题仍未回答：
- **格式歧义：** `due_date` 应使用 "2024-11-06"、"Nov 6, 2024" 还是 "2024-11-06T00:00:00Z"？
- **ID 约定：** `reporter.id` 是 UUID、"USR-12345" 还是只是 "12345"？
- **嵌套结构使用：** 何时应该 Claude 填充 `reporter.contact`？
- **参数关联：** `escalation.level` 和 `escalation.sla_hours` 如何与 priority 关联？

这些歧义可能导致格式错误的工具调用和不一致的参数使用。

### 我们的解决方案

Tool Use Examples 让你直接在工具定义中提供示例工具调用。不只依赖 schema，你向 Claude 展示具体的使用模式：

```json
{
  "name": "create_ticket",
  "input_schema": { /* 与上相同的 schema */ },
  "input_examples": [
    {
      "title": "Login page returns 500 error",
      "priority": "critical",
      "labels": ["bug", "authentication", "production"],
      "reporter": {
        "id": "USR-12345",
        "name": "Jane Smith",
        "contact": {
          "email": "jane@acme.com",
          "phone": "+1-555-0123"
        }
      },
      "due_date": "2024-11-06",
      "escalation": {
        "level": 2,
        "notify_manager": true,
        "sla_hours": 4
      }
    },
    {
      "title": "Add dark mode support",
      "labels": ["feature-request", "ui"],
      "reporter": {
        "id": "USR-67890",
        "name": "Alex Chen"
      }
    },
    {
      "title": "Update API documentation"
    }
  ]
}
```

从这三个示例中，Claude 学到：
- **格式约定：** 日期使用 YYYY-MM-DD，用户 ID 遵循 USR-XXXXX，labels 使用 kebab-case
- **嵌套结构模式：** 如何构建包含嵌套 contact 对象的 reporter 对象
- **可选参数关联：** 关键 bug 有完整联系信息 + 紧密 SLA 的升级；功能请求有 reporter 但无 contact/escalation；内部任务只有 title

在我们自己的内部测试中，Tool Use Examples 将复杂参数处理的准确性从 72% 提高到 90%。

### 何时使用 Tool Use Examples

Tool Use Examples 为你的工具定义增加了 token，所以当准确性改进超过额外成本时它们最有价值。

**最有利时：**
- 复杂嵌套结构，其中有效 JSON 不意味着正确使用
- 有许多可选参数且包含模式重要的工具
- 具有 schema 中未捕获的领域特定约定的 API
- 类似工具，其中示例澄清使用哪个（例如 `create_ticket` vs `create_incident`）

**不太有利时：**
- 使用明显的简单单参数工具
- Claude 已经理解的标准格式如 URL 或 email
- 最好由 JSON Schema 约束处理的验证问题

## 最佳实践

构建执行真实世界操作的 Agent 意味着同时处理规模、复杂性和精确性。这三个功能协同工作以解决工具使用 workflow 中的不同瓶颈。以下是如何有效地组合它们。

### 战略性地分层功能

并非每个 Agent 都需要针对给定任务使用所有三个功能。从你最大的瓶颈开始：
- 工具定义导致的上下文膨胀 → Tool Search Tool
- 大型中间结果污染上下文 → Programmatic Tool Calling
- 参数错误和格式错误的调用 → Tool Use Examples

这种专注的方法让你能够解决限制 Agent 性能的特定约束，而不是预先增加复杂性。

然后根据需要分层额外功能。它们是互补的：Tool Search Tool 确保找到正确的工具，Programmatic Tool Calling 确保高效执行，Tool Use Examples 确保正确调用。

### 设置 Tool Search Tool 以实现更好的发现

工具搜索匹配名称和描述，所以清晰、描述性的定义能提高发现准确性。

```json
// 好
{
  "name": "search_customer_orders",
  "description": "Search for customer orders by date range, status, or total amount. Returns order details including items, shipping, and payment info."
}

// 差
{
  "name": "query_db_orders",
  "description": "Execute order query"
}
```

添加系统 prompt 指导，让 Claude 知道什么可用：

```
You have access to tools for Slack messaging, Google Drive file management,
Jira ticket tracking, and GitHub repository operations. Use the tool search
to find specific capabilities.
```

保持你最常用的三到五个工具始终加载，其余延迟加载。这在常见操作的即时访问和所有其他操作的按需发现之间取得平衡。

### 设置 Programmatic Tool Calling 以实现正确执行

由于 Claude 编写代码来解析工具输出，请清楚地记录返回格式。这有助于 Claude 编写正确的解析逻辑：

```json
{
  "name": "get_orders",
  "description": "Retrieve orders for a customer.\nReturns:\nList of order objects, each containing:\n- id (str): Order identifier\n- total (float): Order total in USD\n- status (str): One of 'pending', 'shipped', 'delivered'\n- items (list): Array of {sku, quantity, price}\n- created_at (str): ISO 8601 timestamp"
}
```

以下是从程序化编排中受益的 opt-in 工具：
- 可以并行运行的工具（独立操作）
- 可以安全重试的操作（幂等操作）

### 设置 Tool Use Examples 以实现参数准确性

为行为清晰度精心设计示例：
- 使用真实数据（真实城市名称、合理价格，而非 "string" 或 "value"）
- 通过最小、部分和完整规范模式展示多样性
- 保持简洁：每个工具 1-5 个示例
- 专注于歧义（只在正确使用从 schema 不明显的地方添加示例）

## 开始使用

这些功能在 beta 中可用。要启用它们，添加 beta header 并包含你需要的工具：

```python
client.beta.messages.create(
    betas=["advanced-tool-use-2025-11-20"],
    model="claude-sonnet-4-5-20250929",
    max_tokens=4096,
    tools=[
        {"type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex"},
        {"type": "code_execution_20250825", "name": "code_execution"},
        # 你的带有 defer_loading、allowed_callers 和 input_examples 的工具
    ]
)
```

有关详细的 API 文档和 SDK 示例，请参阅我们的：
- Tool Search Tool 的文档和 cookbook
- Programmatic Tool Calling 的文档和 cookbook
- Tool Use Examples 的文档

这些功能将工具使用从简单的函数调用推向智能编排。随着 Agent 处理跨越数十个工具和大型数据集的更复杂 workflow，动态发现、高效执行和可靠调用变得至关重要。

我们期待看到你构建什么。

## 致谢

由 Bin Wu 撰写，Adam Jones、Artur Renault、Henry Tay、Jake Noble、Noah Picard、Sam Jiang 和 Claude 开发者平台团队做出了贡献。本文建立在 Chris Gorgolewski、Daniel Jiang、Jeremy Fox 和 Mike Lambert 的基础研究之上。我们还从整个 AI 生态系统中获得灵感，包括 Joel Pobar 的 LLMVM、Cloudflare 的 Code Mode 和 Code Execution as MCP。特别感谢 Andy Schumeister、Hamish Kerr、Keir Bradwell、Matt Bleifer 和 Molly Vorwerck 的支持。

## 相关链接
- [[anthropic-writing-tools-for-agents]] — 为 Agent 编写高效工具的原则与实践
- [[anthropic-think-tool]] — 复杂工具调用前的专用思考步骤
- [[anthropic-agent-skills]] — Agent Skills 机制与渐进式披露
- [[anthropic-code-execution-mcp]] — MCP 代码执行与工具发现
