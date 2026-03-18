# TUI 聊天发送执行路径（以「读取 test.md」为例）

本文梳理：在 TUI 里输入「读取一下 test.md」并收到「文件内容是：123」的**完整执行路径**（从按键到工具调用再到回显）。

---

## 一、整体流程概览

```text
用户输入 "读取一下 test.md"
    → TUI 发送 chat.send RPC
    → Gateway 处理 chat.send
    → dispatchInboundMessage → getReplyFromConfig → runReplyAgent
    → runAgentTurnWithFallback → runEmbeddedPiAgent
    → runEmbeddedAttempt：createOpenClawCodingTools() 得到工具列表（含 read）
    → Pi 模型决定调用 read 工具，传入 path: "test.md"
    → read 工具在 workspace 下读 test.md，返回内容
    → 模型把内容格式化成「文件内容是：123」等
    → 流式/最终 payload 经 dispatcher 发回
    → Gateway 通过 WebSocket 推给 TUI
    → TUI 在聊天里展示
```

---

## 二、各层调用链（按执行顺序）

### 1. TUI 侧：用户输入 → 发 RPC

| 步骤     | 位置                      | 说明                                                                                                      |
| -------- | ------------------------- | --------------------------------------------------------------------------------------------------------- |
| 用户输入 | TUI 输入框                | 例如「读取一下 test.md」                                                                                  |
| 发送     | `src/tui/gateway-chat.ts` | `GatewayChat.sendChat()`                                                                                  |
| RPC      | 同上                      | `this.client.request("chat.send", { sessionKey, message, thinking, deliver, timeoutMs, idempotencyKey })` |

TUI 通过 Gateway 的 WebSocket/RPC 客户端发 `chat.send`，不直接调 `createOpenClawTools`（那是 HTTP `/tools/invoke` 或 Pi agent 侧用的）。

---

### 2. Gateway 侧：收到 chat.send

| 步骤     | 位置                                 | 说明                                                                         |
| -------- | ------------------------------------ | ---------------------------------------------------------------------------- |
| RPC 入口 | `src/gateway/server-methods/chat.ts` | handler `"chat.send"`                                                        |
| 校验     | 同上                                 | `validateChatSendParams`、`sanitizeChatSendMessageInput`、`loadSessionEntry` |
| 派发     | 同上                                 | `dispatchInboundMessage({ ctx, cfg, dispatcher, replyOptions })`             |

`ctx` 里包含：`BodyForAgent`（带时间戳的用户消息）、`SessionKey`、`OriginatingChannel` 等。

---

### 3. Auto-reply：派发到回复流水线

| 步骤         | 位置                                           | 说明                                                                            |
| ------------ | ---------------------------------------------- | ------------------------------------------------------------------------------- |
| 派发         | `src/auto-reply/dispatch.ts`                   | `dispatchInboundMessage` → `finalizeInboundContext` + `dispatchReplyFromConfig` |
| 从配置取回复 | `src/auto-reply/reply/dispatch-from-config.ts` | `dispatchReplyFromConfig` 里根据 session/command 等决定是否跑 agent、发 BTW 等  |
| 取回复实现   | `src/auto-reply/reply.js`（或 get-reply）      | `getReplyFromConfig(ctx, opts, configOverride)`                                 |

---

### 4. 回复实现：排队与执行 Agent 轮次

| 步骤            | 位置                                             | 说明                                                                                        |
| --------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| 会话/队列准备   | `src/auto-reply/reply/get-reply.ts`              | `getReplyFromConfig`：解析 session、workspace、model、typing 等                             |
| 入队/执行       | `src/auto-reply/reply/get-reply-run.ts`          | 构建 `FollowupRun`（含 `run.config`、`run.workspaceDir` 等），`enqueueFollowupRun` 或直接跑 |
| 执行单轮        | `src/auto-reply/reply/agent-runner.ts`           | `runReplyAgent`：准备 prompt、block streaming、typing、abort 等                             |
| 真正跑模型+工具 | `src/auto-reply/reply/agent-runner-execution.ts` | `runAgentTurnWithFallback` → **`runEmbeddedPiAgent`**                                       |

到这里，控制权交给 **Pi 嵌入式 Agent**（同一进程内，Gateway 进程）。

---

### 5. Pi 嵌入式 Agent：模型 + 工具

| 步骤         | 位置                                           | 说明                                                                                                                                  |
| ------------ | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| 入口         | `src/agents/pi-embedded-runner/run.ts`         | `runEmbeddedPiAgent`                                                                                                                  |
| 单次尝试     | `src/agents/pi-embedded-runner/run/attempt.ts` | `runEmbeddedAttempt`                                                                                                                  |
| **工具列表** | 同上                                           | `createOpenClawCodingTools({ ... })`（在 attempt 里调用），得到 **read / write / edit / exec / browser** 等；其中 **read** 对应读文件 |
| 调用模型     | 同上                                           | 把消息历史 + 工具定义发给 LLM；模型决定调用 `read`，参数如 `path: "test.md"`                                                          |
| 执行 read    | `src/agents/pi-tools.read.js` 等               | `createOpenClawReadTool` / sandbox read：在 **agent workspace** 下解析 `test.md`，读内容返回                                          |
| 结果回填     | attempt.ts                                     | 工具结果写回 transcript，再继续模型生成，得到最终回复文本（如「文件内容是：123」）                                                    |

「读取 test.md」对应的就是：**模型选择 read 工具 → read 在 workspace 下读 test.md → 返回 "123" → 模型组织成你看到的那段话**。

---

### 6. 回复返回 TUI

| 步骤       | 位置                                                 | 说明                                                                                                  |
| ---------- | ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| 流式/块    | `agent-runner-execution.ts` / `followup-runner.ts`   | `runEmbeddedPiAgent` 通过 callback 吐出 payload（text delta、tool 事件等）                            |
| 派发       | `createReplyDispatcher` 等                           | 把 payload 发给 chat 的 `dispatcher`（在 chat.send 里传入的）                                         |
| 推给客户端 | `src/gateway/server-methods/chat.ts` 及 WebSocket 层 | 通过 `broadcastChatFinal`、`registerToolEventRecipient` 等把最终回复和 tool 事件推给连接的 TUI 客户端 |
| 展示       | TUI                                                  | 在聊天界面渲染「文件内容是：123」等                                                                   |

---

## 三、关键点小结

1. **TUI 不直接调 `createOpenClawTools`**  
   你看到的那条「createOpenClawTools 没被调用」的日志，是因为 TUI 聊天走的是 **chat.send → dispatch → Pi agent**；**工具列表是在 Pi 的 `runEmbeddedAttempt` 里用 `createOpenClawCodingTools` 建的**（在 Gateway 进程内，但属于 agent 执行路径）。

2. **读文件用的是 Pi 的 read 工具**  
   名字是 **`read`**（不是 `read_file`），来自 `createOpenClawCodingTools` 里的 `createOpenClawReadTool` / sandbox read；路径相对于 **agent workspace**（如 `~/.openclaw/agents/<agentId>/workspace` 或配置的 workspace）。

3. **同一进程**  
   整个链路在 **同一个 Gateway 进程** 里：chat.send → dispatch → getReplyFromConfig → runReplyAgent → runEmbeddedPiAgent → runEmbeddedAttempt → createOpenClawCodingTools + read 工具执行。

4. **若要看 createOpenClawTools 的日志**
   - **TUI 聊天路径**：不会调用 `createOpenClawTools`，会调用 **`createOpenClawCodingTools`**（在 `pi-embedded-runner/run/attempt.ts`）；若在 `createOpenClawCodingTools` 里加日志，会在每次 TUI 发消息并跑 agent 时打到 Gateway 日志。
   - **HTTP `/tools/invoke` 路径**：会调 **`createOpenClawTools`**（在 `gateway/tools-invoke-http.ts`），只有外部用 `POST /tools/invoke` 时才会触发。

---

## 四、和「createOpenClawTools 没被调用」的关系

- **createOpenClawTools**：用在 **HTTP `/tools/invoke`** 和 **auto-reply inline actions** 等，**不在 TUI 普通聊天的 agent 路径里**。
- **createOpenClawCodingTools**：用在 **Pi 嵌入式 agent**（TUI 聊天、cron、channel 回复等），会包含 **read / write / edit / exec** 等，**TUI 里「读取 test.md」走的就是这条路径**。

因此：**TUI 里输入「读取一下 test.md」的完整执行路径 = 上面第二节的 1→2→3→4→5→6**，其中「读文件」发生在第 5 步的 **read 工具**，工具列表来自 **createOpenClawCodingTools**，不是 **createOpenClawTools**。
