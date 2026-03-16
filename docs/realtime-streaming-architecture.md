# 实时信息如何从 Claude Code / Codex / GitHub Copilot 传输到 Web 页面

## 概述

这三个 CLI 工具都**没有**提供专门的"Web 推送"接口。项目自己实现了一套完整的管道：

```
AI CLI 进程 stdout/stderr
        ↓
   解析 & 规范化层（Rust）
        ↓
     MsgStore（内存广播）
        ↓
   WebSocket 服务端（Axum）
        ↓
   streamJsonPatchEntries（前端 JS）
        ↓
       React 状态更新
```

---

## 第一层：AI CLI 各自的输出协议

### Claude Code CLI

Claude 官方提供了结构化输出模式，通过启动参数开启：

```
claude --output-format=stream-json --input-format=stream-json --verbose ...
```

Claude 会把每个事件（思考、消息、工具调用、工具结果）以 **JSON Lines** 的形式输出到 **stdout**，每行一个 JSON 对象，例如：

```json
{"type":"assistant","message":{"content":[{"type":"text","text":"I'll read the file..."}]}}
{"type":"tool_use","id":"toolu_01","name":"Read","input":{"file_path":"src/main.rs"}}
{"type":"tool_result","tool_use_id":"toolu_01","content":"fn main() {...}"}
```

这是 Claude 官方支持的机器可读格式，项目在 `crates/executors/src/executors/claude.rs` 里用 `ClaudeLogProcessor::process_logs()` 逐行解析这些 JSON。

### OpenAI Codex CLI

Codex 不支持 JSON 流格式，而是使用 **JSON-RPC over stdin/stdout**：

```
codex ... （无特殊参数，通过 stdin 发送 JSON-RPC 请求，从 stdout 读取响应）
```

项目把 Codex 当作一个 **app server** 来运行：通过 `JsonRpcPeer` 在 `stdin`/`stdout` 上进行 JSON-RPC 通信（`crates/executors/src/executors/codex.rs` 的 `spawn_app_server`）。每个 RPC 响应事件同样被解析并推送到 MsgStore。

### GitHub Copilot CLI

Copilot CLI 使用 **ACP（Agent Client Protocol）**，通过启动参数开启：

```
gh copilot ... --acp
```

加了 `--acp` 后，Copilot 会在 **stdout** 输出结构化的 ACP 事件（`SessionNotification` 消息），项目在 `crates/executors/src/executors/acp/normalize_logs.rs` 里监听并解析这些事件。

---

## 第二层：规范化层（Rust 后端）

所有三个工具的输出格式不同，但项目把它们统一转换成同一种数据结构 `NormalizedEntry`：

```rust
pub enum NormalizedEntryType {
    AssistantMessage,   // AI 文本回复
    UserMessage,        // 用户输入
    ToolUse { tool_name, action_type, status },  // 工具调用（读文件/写文件/执行命令等）
    Thinking,           // 思考过程
    ErrorMessage,       // 错误信息
    TokenUsageInfo,     // Token 用量
    Loading,            // 加载状态
    // ...
}
```

每个执行器的 `normalize_logs()` 方法负责把自家的原始输出解析为这些规范化条目，然后通过 `MsgStore::push_patch()` 以 **JSON Patch** 格式推送。

---

## 第三层：MsgStore（内存广播总线）

`MsgStore`（`crates/utils/src/msg_store.rs`）是核心枢纽：

- **写端**：解析协程调用 `push_patch()`、`push_stdout()`、`push_stderr()`
- **广播**：内部用 `tokio::sync::broadcast` channel，支持多订阅者同时消费
- **历史缓存**：保存最近 100 MB 的消息，新连接可以"回放"历史再接收实时更新

```
                  ┌─────────────┐
Claude stdout ──▶ │ LogProcessor│──push_patch──▶ MsgStore.broadcast_channel
Codex JSON-RPC ──▶│ normalize_  │                     │
Copilot ACP   ──▶ │ logs()      │               history (回放)
                  └─────────────┘
```

---

## 第四层：WebSocket 服务端（Axum）

后端暴露两个 WebSocket 端点（`crates/server/src/routes/execution_processes.rs`）：

| 端点 | 过滤内容 | 用途 |
|------|---------|------|
| `GET /api/execution-processes/{id}/raw-logs/ws` | `Stdout` + `Stderr` | 原始日志查看 |
| `GET /api/execution-processes/{id}/normalized-logs/ws` | `JsonPatch` | AI 活动结构化视图 |

`stream_normalized_logs` 的实现逻辑：
1. 如果进程还在运行：从 MsgStore 拿到 `history_plus_stream()`，只保留 `JsonPatch` 消息，通过 WebSocket 推送
2. 如果进程已结束：从数据库加载历史消息，创建临时 MsgStore 重放，再执行一次规范化，推送结果

每条消息的 Wire Format 是 JSON，例如：

```json
{"JsonPatch":[{"op":"add","path":"/entries/0","value":{"entry_type":{"type":"tool_use","tool_name":"Read","action_type":{"action":"file_read","path":"src/main.rs"},"status":{"status":"success"}},"content":"","timestamp":"2025-01-01T00:00:00Z"}}]}
```

完成后发送 `{"finished":true}`。

---

## 第五层：前端实时接收（React + TypeScript）

前端通过 `streamJsonPatchEntries` 工具函数（`packages/web-core/src/shared/lib/streamJsonPatchEntries.ts`）订阅 WebSocket：

1. 连接 WebSocket
2. 收到 `JsonPatch` 消息 → 用 `immer` + `rfc6902` 原地 patch 内存中的 `{ entries: [] }` 数组
3. 通过 `requestAnimationFrame` 批量更新，避免频繁 re-render
4. 收到 `{"finished":true}` → 调用 `onFinished` 回调

具体 Hook 是 `useConversationHistory`（`packages/web-core/src/shared/hooks/useConversationHistory/`），它把 `NormalizedEntry` 数组进一步聚合（合并连续的文件读取、折叠思考步骤等），再传给 `ConversationList` 组件渲染。

---

## 总结对比

| 维度 | Claude Code | Codex | GitHub Copilot |
|------|-------------|-------|----------------|
| 结构化输出方式 | `--output-format=stream-json` 参数，Claude 官方支持 | JSON-RPC over stdin/stdout | `--acp` 参数，ACP 协议 |
| 项目解析文件 | `executors/claude.rs` + `claude/protocol.rs` | `executors/codex/normalize_logs.rs` + `jsonrpc.rs` | `executors/acp/normalize_logs.rs` |
| 共同抽象层 | `NormalizedEntry` + `MsgStore` → WebSocket JsonPatch | 同左 | 同左 |
| Web 页面消费 | `streamJsonPatchEntries` + `useConversationHistory` | 同左 | 同左 |

**关键结论**：Claude/Codex/Copilot 各自只提供了结构化的 CLI 输出协议（JSON Lines / JSON-RPC / ACP），项目自己构建了整套"解析 → 规范化 → 广播 → WebSocket → 前端实时渲染"的管道，这才是实时信息到达 Web 页面的核心机制。
