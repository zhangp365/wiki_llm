# ACP（Agent Client Protocol）接口总结文档

本文档整理了项目中 UI 侧（TensorChat）与 Agent 侧（OpenCode）之间使用 ACP 协议通信的所有核心接口。这些接口涵盖了生命周期管理、会话控制与状态反向同步等功能。

## 一、生命周期与连接初始化

### 1. `initialize`
- **作用**：握手与初始化连接。UI向Agent宣告自身支持的特性（clientCapabilities）、协议版本及环境信息；Agent响应服务端支持的特性及工作区初始状态。
- **触发时机**：当 Agent 进程启动或发生重启，建立通信连接的第一步（进程预热阶段 `warmupProcess` 或进程正式挂载前）。
- **代码位置**：
  - **UI侧**：`src/main/presenter/llmProviderPresenter/acp/acpProcessManager.ts`（在 `spawnProcess` 中被调用）
  - **Agent侧**：`opencode/packages/opencode/src/server/routes/global.ts` 或核心初始化路由中。

---

## 二、会话控制 (Session Management)

### 2. `session/new` (`newSession`)
- **作用**：通知 Agent 创建一个全新的交互会话环境，可以携带初始的工作目录（`cwd`）及挂载的服务器配置。
- **触发时机**：用户在 UI 侧发起全新对话（首次发送消息），且该会话尚未在 Agent 端建立上下文时。
- **代码位置**：
  - **UI侧**：`src/main/presenter/llmProviderPresenter/acp/acpSessionManager.ts`（`getOrCreateSession` 实现逻辑里）
  - **Agent侧**：`opencode/packages/opencode/src/server/routes/session.ts`

### 3. `session/load` (`loadSession`)
- **作用**：让 Agent 恢复/加载一个已有历史上下文的对话会话。
- **触发时机**：用户在历史对话列表中切换回过去的对话，并继续与 Agent 交互时。
- **代码位置**：
  - **UI侧**：`src/main/presenter/llmProviderPresenter/acp/acpSessionManager.ts` 及其配置重载
  - **Agent侧**：`opencode/packages/opencode/src/server/routes/session.ts`

### 4. `session/prompt` (`prompt`)
- **作用**：发送用户输入指令（包含聊天记录上下文或指令块）给 Agent，并启动 Agent 的执行循环（返回事件流）。
- **触发时机**：用户在输入框点击“发送”，提交任务或者代码生成需求给 Agent。
- **代码位置**：
  - **UI侧**：`src/main/presenter/llmProviderPresenter/providers/acpProvider.ts` （`runPrompt` 方法，通过 `connection.prompt` 调用并监听执行流）
  - **Agent侧**：`opencode/packages/opencode/src/server/routes/session.ts`

### 5. `session/cancel` (`cancel`)
- **作用**：请求终止当前执行的 Prompt 或长耗时任务。
- **触发时机**：用户点击界面上的“停止/中止”生成按钮，或主动关闭相关的会话窗口导致组件被销毁。
- **代码位置**：
  - **UI侧**：`src/main/presenter/llmProviderPresenter/providers/acpProvider.ts` （生成结束或者异常时的 `finally` 块中调用 `session.connection.cancel`）
  - **Agent侧**：`opencode/packages/opencode/src/server/routes/session.ts`

### 6. `session/setMode` (`setSessionMode`)
- **作用**：切换 Agent 的当前工作模式（例如从 Chat 对话模式切换为具备强执行能力的 Agent 模式）。
- **触发时机**：当用户在 UI 下拉框或设置面板中更改 ACP 会话模式。
- **代码位置**：
  - **UI侧**：`src/main/presenter/llmProviderPresenter/acp/acpSessionManager.ts` 与 `acpProvider.ts` 中有关 `setPreferredMode` 和 `setSessionMode` 处理逻辑
  - **Agent侧**：`opencode/packages/opencode/src/server/routes/session.ts` 

### 7. `session/setModel` (`setSessionModel`)
- **作用**：设置或切换当前会话所绑定的底层 LLM 模型参数。
- **触发时机**：用户在 UI 模型切换下拉框中主动更改该工作区底层挂载的模型时触发同步。
- **代码位置**：
  - **UI侧**：`src/main/presenter/llmProviderPresenter/providers/acpProvider.ts` （`setSessionModelCompat` 和调试动作处）
  - **Agent侧**：`opencode/packages/opencode/src/server/routes/session.ts`（处理模型切换逻辑）

---

## 三、扩展方法与通知 (Extension Mechanisms)

### 8. `ext/method` (`extMethod`)
- **作用**：允许发送特定的、可双向通信的扩展 RPC 命令，适应未来的功能扩展及特殊调试指令（如在 Debug 调试接口或插件系统中发挥作用）。
- **触发时机**：因具体的插件或特定功能模块（如调试指令 `runDebugAction` 时）调用触发。
- **代码位置**：
  - **UI侧**：`src/main/presenter/llmProviderPresenter/providers/acpProvider.ts`（在 `runDebugAction` 中进行了映射）
  - **Agent侧**：通常在 `opencode/packages/opencode/src/server/routes/experimental.ts` 或动态注册路由扩展。

### 9. `ext/notification` (`extNotification`)
- **作用**：向对方发送无需阻塞等待返回值的扩展事件通知，适用于轻量级同步信息。
- **触发时机**：当需要广播内部非关键链路层事件（如状态重置、特定的调试通知等）时。
- **代码位置**：
  - **UI侧**：`src/main/presenter/llmProviderPresenter/providers/acpProvider.ts` （在 `runDebugAction` 配置映射）
  - **Agent侧**：对应的自定义或扩展路由接口。

---

## 四、Agent 反向通知 UI (Server to Client Messages)

由于 ACP 是双向协议，Agent 在产生增量信息及遇到工具权限校验时，会反过来向 UI 客户端发送事件：

### 10. `session/update` (Notification)
- **作用**：Agent 执行过程中的状态迭代（可用 mode 变更、配置同步、终端日志输出产生等），或会话状态变化会主动推送给 UI。
- **触发时机**：在 `session/prompt` 运行中或后台触发状态发生变化时（异步并发）。
- **代码位置**：
  - **UI侧**：`src/main/presenter/llmProviderPresenter/providers/acpProvider.ts`（`handleSessionUpdate` 方法监听反向同步并更新界面状态）
  - **Agent侧**：任何在 Agent 运行周期导致上下文变化的地方主动 emit。

### 11. `requestPermission` / `permission`
- **作用**：请求权限（Permission）。当 Agent 需要执行某些“不安全”或受限制的工具调用时（如修改本地关键文件或执行高危命令），将请求挂起并通知 UI。UI获取请求后弹窗询问用户并回传授权结果（Approve / Reject）。
- **触发时机**：Agent 脚本内部拦截到某个高风险 action（如 bash / file-write 操作）触发。
- **代码位置**：
  - **UI侧**：`src/main/presenter/llmProviderPresenter/providers/acpProvider.ts` （`handlePermissionRequest` 方法弹出对话框并执行挂起和接续）及 `acpProcessManager.ts` 中的 `registerPermissionResolver` 方法。
  - **Agent侧**：通常由 `opencode/packages/opencode/src/tool/` 相关工具内部发起权限确认流程；对应的上层入口为 `opencode/packages/opencode/src/server/routes/permission.ts`（权限控制网关）。
