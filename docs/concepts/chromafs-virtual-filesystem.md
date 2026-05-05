     1|---
     2|title: ChromaFs — Agent 虚拟文件系统
     3|created: 2026-05-05
     4|updated: 2026-05-05
     5|type: concept
     6|tags: [agent, tool-use, open-source]
     7|sources: [raw/articles/mintlify-chromafs-virtual-filesystem.md]
     8|---
     9|
    10|# ChromaFs — Agent 虚拟文件系统
    11|
    12|> **原文链接**: [How we built a virtual filesystem for our Assistant](https://www.mintlify.com/blog/how-we-built-a-virtual-filesystem-for-our-assistant) — Dens Sumesh, Mintlify Engineering, 2026-03-24
    13|
    14|## 核心思想
    15|
    16|Agent 正在趋同于以文件系统作为主要交互界面（参考 [arXiv:2601.11672](https://arxiv.org/abs/2601.11672)）。`grep`、`cat`、`ls`、`find` 就是一个 Agent 需要的全部工具。ChromaFs 的核心洞察是：Agent 不需要一个**真实的**文件系统，只需要**幻觉**。通过拦截 UNIX 命令并将其翻译为数据库查询，实现了轻量级的虚拟文件系统。
    17|
    18|## 背景：为什么不用沙箱
    19|
    20|传统方案是给 Agent 一个真实文件系统（sandbox + git clone），但问题明显：
    21|
    22|| 指标 | Sandbox 方案 | ChromaFs |
    23||------|-------------|----------|
    24|| P90 启动时间 | ~46 秒 | ~100 毫秒 |
    25|| 边际计算成本 | ~$0.0137/会话 | ~$0（复用现有 DB） |
    26|| 搜索机制 | 线性磁盘扫描 | DB 元数据查询 |
    27|| 基础设施 | Daytona 等 sandbox 提供商 | 已有 Chroma DB |
    28|
    29|85 万次/月的对话量下，sandbox 方案年成本超过 $70,000。
    30|
    31|## 技术架构
    32|
    33|### 基础：just-bash
    34|
    35|ChromaFs 基于 [just-bash](https://github.com/vercel-labs/just-bash)（Vercel Labs 出品的 TypeScript bash 重实现），支持 `grep`/`cat`/`ls`/`find`/`cd`。just-bash 暴露可插拔的 `IFileSystem` 接口，负责解析、管道和 flag 逻辑，ChromaFs 则将每个底层文件系统调用翻译为 Chroma 查询。
    36|
    37|### 目录树引导
    38|
    39|整个文件树以 gzip 压缩的 JSON（`__path_tree__`）存储在 Chroma collection 中，包含路径和访问控制元数据：
    40|
    41|```json
    42|{
    43|  "auth/oauth": { "isPublic": true, "groups": [] },
    44|  "internal/billing": { "isPublic": false, "groups": ["admin", "billing"] }
    45|}
    46|```
    47|
    48|初始化时解压为内存中的 `Set<string>` + `Map<string, string[]>`，`ls`/`cd`/`find` 全部本地内存解析，零网络调用。
    49|
    50|### 访问控制（RBAC）
    51|
    52|根据用户 session token 在构建文件树前裁剪 slug。用户无权访问的文件从树中完全移除，Agent 既不能访问也不能引用。在真实 sandbox 中需要管理 Linux 用户组/chmod/独立容器镜像，ChromaFs 中只需几行过滤代码。
    53|
    54|### 页面重组
    55|
    56|Chroma 中的文档按 chunk 存储（用于 embedding），`cat` 时按 `chunk_index` 排序并拼接为完整页面。结果带缓存，grep 流程中重复读取不二次查询数据库。
    57|
    58|还支持**惰性文件指针**：大型 OpenAPI spec 存在 S3 中，`ls` 时可见但 `cat` 时才拉取。
    59|
    60|### 只读保证
    61|
    62|所有写操作抛出 `EROFS` 错误。Agent 可自由探索但不能修改文档，系统无状态、无需 session 清理、无 Agent 间污染风险。
    63|
    64|### Grep 优化
    65|
    66|`grep -r` 的核心优化是**两阶段过滤**：
    67|1. **粗筛（Chroma）**：将 grep 模式翻译为 Chroma 查询（`$contains` 或 `$regex`），识别可能命中的文件
    68|2. **精筛（内存）**：将匹配的 chunks 批量预取到 Redis 缓存，重写 grep 命令后交还 just-bash 在内存中执行
    69|
    70|大规模递归查询毫秒级完成。
    71|
    72|## 关键数据
    73|
    74|- 日均 30,000+ 会话
    75|- 启动时间从 46 秒降至 100 毫秒（460x 提升）
    76|- 零新增基础设施成本
    77|- 内置 RBAC
    78|
    79|## 相关概念
    80|
    81|- [A2A 协议](a2a-protocol.md) — Agent 间通信协议，文件系统接口是另一种 Agent 交互范式
    82|- [Hermes Agent 记忆架构](hermes-agent-memory-architecture.md) — Hermes Agent 使用真实文件系统的 Agent 工具实现
    83|