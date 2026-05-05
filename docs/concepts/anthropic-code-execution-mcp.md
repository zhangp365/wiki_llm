---
title: "使用 MCP 进行代码执行：构建更高效的智能体"
created: 2026-05-05
updated: 2026-05-05
type: concept
tags: [agent, mcp, code-execution, tool-use, anthropic]
sources: [raw/articles/anthropic-11-code-execution-mcp.md]
original: https://www.anthropic.com/engineering/code-execution-with-mcp
---

# 使用 MCP 进行代码执行：构建更高效的智能体

> 原文链接: [English Original](https://www.anthropic.com/engineering/code-execution-with-mcp)

直接的工具调用会为每个定义和结果消耗 context。智能体通过编写代码来调用工具可以更好地扩展。以下是与 MCP 配合使用的方法。

Model Context Protocol（MCP）是将 AI 智能体连接到外部系统的开放标准。将智能体连接到工具和数据传统上需要为每对组合进行自定义集成，造成碎片化和重复劳动，使得扩展真正连接的系统变得困难。MCP 提供了一个通用协议——开发者在智能体中实现一次 MCP，即可解锁整个集成生态系统。

自 2024 年 11 月推出 MCP 以来，采用速度很快：社区已构建了数千个 MCP server，所有主要编程语言都提供了 SDK，行业已将 MCP 采用为连接智能体与工具和数据的事实标准。

如今开发者通常会构建连接数十个 MCP server 上数百或数千个工具的智能体。然而，随着连接工具数量的增长，预先加载所有工具定义并通过 context window 传递中间结果会减慢智能体速度并增加成本。

在本文中，我们将探讨代码执行如何使智能体更高效地与 MCP server 交互，使用更少的 token 处理更多工具。

## 工具消耗过多 token 使智能体效率降低

随着 MCP 使用规模的扩大，有两种常见模式会增加智能体成本和延迟：
1. 工具定义使 context window 过载
2. 中间工具结果消耗额外的 token

### 1. 工具定义使 context window 过载

大多数 MCP client 预先将所有工具定义直接加载到 context 中，使用直接工具调用语法将其暴露给模型。这些工具定义可能如下所示：

```
gdrive.getDocument
Description: Retrieves a document from Google Drive
Parameters:
  documentId (required, string): The ID of the document to retrieve
  fields (optional, string): Specific fields to return
Returns: Document object with title, body content, metadata, permissions, etc.
```

```
salesforce.updateRecord
Description: Updates a record in Salesforce
Parameters:
  objectType (required, string): Type of Salesforce object (Lead, Contact, Account, etc.)
  recordId (required, string): The ID of the record to update
  data (required, object): Fields to update with their new values
Returns: Updated record object with confirmation
```

工具描述占据了更多的 context window 空间，增加了响应时间和成本。在智能体连接到数千个工具的情况下，它们需要在读取请求之前处理数十万个 token。

### 2. 中间工具结果消耗额外的 token

大多数 MCP client 允许模型直接调用 MCP 工具。例如，你可能要求智能体："从 Google Drive 下载我的会议记录并将其附加到 Salesforce lead。"

模型会进行如下调用：

```
TOOL CALL: gdrive.getDocument(documentId: "abc123")
→ returns "Discussed Q4 goals...\n[full transcript text]"
（加载到模型 context 中）

TOOL CALL: salesforce.updateRecord(
  objectType: "SalesMeeting",
  recordId: "00Q5f000001abcXYZ",
  data: { "Notes": "Discussed Q4 goals...\n[full transcript text written out]" }
)
（模型需要再次将整个记录写入 context）
```

每个中间结果都必须通过模型。在这个例子中，完整的调用记录流经两次。对于 2 小时的销售会议，这意味着处理额外的 50,000 个 token。更大的文档可能超过 context window 限制，中断工作流。

对于大型文档或复杂数据结构，模型在工具调用之间复制数据时可能更容易出错。MCP client 将工具定义加载到模型的 context window 中，并编排一个消息循环，其中每个工具调用和结果在操作之间通过模型传递。

## 使用 MCP 的代码执行提高了 context 效率

随着代码执行环境对智能体变得越来越普遍，一种解决方案是将 MCP server 呈现为代码 API 而非直接工具调用。智能体可以编写代码与 MCP server 交互。这种方法解决了两个挑战：智能体可以只加载它需要的工具，并在执行环境中处理数据后再将结果返回给模型。

有多种方法可以做到这一点。一种方法是从连接的 MCP server 生成所有可用工具的文件树。以下是使用 TypeScript 的实现：

```
servers
├── google-drive
│   ├── getDocument.ts
│   ├── ... (other tools)
│   └── index.ts
├── salesforce
│   ├── updateRecord.ts
│   ├── ... (other tools)
│   └── index.ts
└── ... (other servers)
```

然后每个工具对应一个文件，类似这样：

```typescript
// ./servers/google-drive/getDocument.ts
import { callMCPTool } from "../../../client.js";

interface GetDocumentInput {
  documentId: string;
}

interface GetDocumentResponse {
  content: string;
}

/* Read a document from Google Drive */
export async function getDocument(input: GetDocumentInput): Promise<GetDocumentResponse> {
  return callMCPTool('google_drive__get_document', input);
}
```

我们上面的 Google Drive 到 Salesforce 的例子变成了如下代码：

```typescript
// Read transcript from Google Docs and add to Salesforce prospect
import * as gdrive from './servers/google-drive';
import * as salesforce from './servers/salesforce';

const transcript = (await gdrive.getDocument({ documentId: 'abc123' })).content;
await salesforce.updateRecord({
  objectType: 'SalesMeeting',
  recordId: '00Q5f000001abcXYZ',
  data: { Notes: transcript }
});
```

智能体通过探索文件系统来发现工具：列出 `./servers/` 目录以查找可用的 server（如 google-drive 和 salesforce），然后读取它需要的特定工具文件（如 `getDocument.ts` 和 `updateRecord.ts`）以了解每个工具的接口。这让智能体只加载当前任务需要的定义。这将 token 使用量从 150,000 减少到 2,000——节省了 98.7% 的时间和成本。

Cloudflare 发布了类似的发现，将使用 MCP 的代码执行称为"Code Mode"。核心洞察是相同的：LLM 擅长编写代码，开发者应该利用这一优势构建更高效地与 MCP server 交互的智能体。

## 使用 MCP 进行代码执行的优势

使用 MCP 的代码执行使智能体能够更高效地使用 context，通过按需加载工具、在数据到达模型之前进行过滤，以及在单步中执行复杂逻辑。这种方法还具有安全性和状态管理方面的优势。

### 渐进式披露

模型非常擅长导航文件系统。将工具作为文件系统上的代码呈现，允许模型按需读取工具定义，而不是预先读取所有定义。

或者，可以向 server 添加 `search_tools` 工具来查找相关定义。例如，当使用上面假设的 Salesforce server 时，智能体搜索"salesforce"并只加载当前任务需要的那些工具。在 `search_tools` 工具中包含一个细节级别参数，允许智能体选择所需的细节程度（如仅名称、名称和描述、或包含 schema 的完整定义），也有助于智能体节省 context 并高效查找工具。

### Context 高效的工具结果

在处理大型数据集时，智能体可以在代码中过滤和转换结果，然后再返回。考虑获取一个 10,000 行的电子表格：

```javascript
// Without code execution - all rows flow through context
TOOL CALL: gdrive.getSheet(sheetId: 'abc123')
→ returns 10,000 rows in context to filter manually

// With code execution - filter in the execution environment
const allRows = await gdrive.getSheet({ sheetId: 'abc123' });
const pendingOrders = allRows.filter(row =>
  row["Status"] === 'pending'
);
console.log(`Found ${pendingOrders.length} pending orders`);
console.log(pendingOrders.slice(0, 5)); // Only log first 5 for review
```

智能体看到五行而不是 10,000 行。类似模式适用于聚合、跨多个数据源的连接，或提取特定字段——所有这些都不会膨胀 context window。

### 更强大且 context 高效的控制流

循环、条件判断和错误处理可以使用熟悉的代码模式完成，而不是链接单独的工具调用。例如，如果你需要在 Slack 中获取部署通知，智能体可以编写：

```javascript
let found = false;
while (!found) {
  const messages = await slack.getChannelHistory({ channel: 'C123456' });
  found = messages.some(m => m.text.includes('deployment complete'));
  if (!found) await new Promise(r => setTimeout(r, 5000));
}
console.log('Deployment notification received');
```

这种方法比通过智能体循环交替使用 MCP 工具调用和 sleep 命令更高效。

此外，能够写出被执行的条件树还节省了"time to first token"延迟：与其等待模型评估 if 语句，不如让代码执行环境来做这件事。

### 隐私保护操作

当智能体使用 MCP 的代码执行时，中间结果默认保留在执行环境中。这样，智能体只看到你显式记录或返回的内容，意味着你不希望与模型共享的数据可以在工作流中流动，而不会进入模型的 context。

对于更敏感的工作负载，智能体框架可以自动对敏感数据进行 token 化。例如，假设你需要将电子表格中的客户联系信息导入 Salesforce。智能体编写：

```javascript
const sheet = await gdrive.getSheet({ sheetId: 'abc123' });
for (const row of sheet.rows) {
  await salesforce.updateRecord({
    objectType: 'Lead',
    recordId: row.salesforceId,
    data: {
      Email: row.email,
      Phone: row.phone,
      Name: row.name
    }
  });
}
console.log(`Updated ${sheet.rows.length} leads`);
```

MCP client 在数据到达模型之前拦截并 token 化 PII：

```javascript
// What the agent would see, if it logged the sheet.rows:
[
  { salesforceId: '00Q...', email: '[EMAIL_1]', phone: '[PHONE_1]', name: '[NAME_1]' },
  { salesforceId: '00Q...', email: '[EMAIL_2]', phone: '[PHONE_2]', name: '[NAME_2]' },
  ...
]
```

然后，当数据在另一个 MCP 工具调用中共享时，通过 MCP client 中的查找进行反 token 化。真实的电子邮件、电话号码和姓名从 Google Sheets 流向 Salesforce，但从不通过模型。这防止了智能体意外记录或处理敏感数据。你还可以使用此方法定义确定性的安全规则，选择数据可以从哪里流向哪里。

### 状态持久化和 Skills

具有文件系统访问权限的代码执行允许智能体在操作之间维护状态。智能体可以将中间结果写入文件，使它们能够恢复工作和跟踪进度：

```javascript
const leads = await salesforce.query({
  query: 'SELECT Id, Email FROM Lead LIMIT 1000'
});
const csvData = leads.map(l => `${l.Id},${l.Email}`).join('\n');
await fs.writeFile('./workspace/leads.csv', csvData);

// Later execution picks up where it left off
const saved = await fs.readFile('./workspace/leads.csv', 'utf-8');
```

智能体还可以将自己的代码持久化为可重用函数。一旦智能体为任务开发了可工作的代码，它可以保存该实现以备将来使用：

```typescript
// In ./skills/save-sheet-as-csv.ts
import * as gdrive from './servers/google-drive';
export async function saveSheetAsCsv(sheetId: string) {
  const data = await gdrive.getSheet({ sheetId });
  const csv = data.map(row => row.join(',')).join('\n');
  await fs.writeFile(`./workspace/sheet-${sheetId}.csv`, csv);
  return `./workspace/sheet-${sheetId}.csv`;
}

// Later, in any agent execution:
import { saveSheetAsCsv } from './skills/save-sheet-as-csv';
const csvPath = await saveSheetAsCsv('abc123');
```

这与 Skills 的概念密切相关——Skills 是可重用指令、脚本和资源的文件夹，用于提高模型在专门任务上的性能。向这些保存的函数添加 `SKILL.md` 文件会创建一个结构化的 skill，模型可以引用和使用。随着时间的推移，这允许你的智能体构建一个高级能力的工具箱，进化它最有效工作所需的脚手架。

请注意，代码执行引入了自身的复杂性。运行智能体生成的代码需要一个安全的执行环境，配备适当的 sandboxing、资源限制和监控。这些基础设施需求增加了运营开销和安全考虑，而直接工具调用则避免了这些。代码执行的好处——降低 token 成本、更低延迟和改进的工具组合——应该与这些实现成本进行权衡。

## 总结

MCP 为智能体连接许多工具和系统提供了基础协议。然而，一旦连接了太多 server，工具定义和结果可能消耗过多 token，降低智能体效率。

尽管这里的许多问题感觉很新颖——context 管理、工具组合、状态持久化——但它们在软件工程中有已知的解决方案。代码执行将这些成熟的模式应用于智能体，让它们使用熟悉的编程构造更高效地与 MCP server 交互。如果你实现了这种方法，我们鼓励你与 [anthropic-mcp-intro](concepts/MCP 社区.md) 分享你的发现。

## 致谢

本文由 Adam Jones 和 Conor Kelly 撰写。感谢 Jeremy Fox、Jerome Swannack、Stuart Ritchie、Molly Vorwerck、Matt Samuels 和 Maggie Vo 对本文草稿的反馈。