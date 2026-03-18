# OpenClaw 本地浏览器架构与代码逻辑

本文档梳理 OpenClaw 如何使用本地浏览器：从 Gateway 启动 Browser Control 服务，到按需启动 Chrome、经 CDP/Playwright 执行操作，再到 Agent 工具 `browser` 的调用链。

---

## 一、整体架构

```mermaid
flowchart TB
  subgraph Agent["Agent 层"]
    Tool["browser 工具\n(agents/tools/browser-tool.ts)"]
  end

  subgraph Invoke["调用路径"]
    Direct["本机 HTTP\n(client.ts / client-fetch)"]
    GatewayRPC["Gateway RPC\n(gateway/server-methods/browser)"]
    NodeProxy["Node 代理\n(node.invoke browser.proxy)"]
    SandboxBridge["沙箱 Bridge\n(sandboxBridgeUrl)"]
  end

  subgraph Control["Browser Control 服务"]
    HTTP["HTTP 服务\n127.0.0.1:18791\n(browser/server.ts)"]
    Routes["路由\nbasic / tabs / agent"]
    Context["BrowserRouteContext\n(server-context.ts)"]
  end

  subgraph Runtime["浏览器运行时"]
    Chrome["本地 Chrome 进程\n(chrome.ts launchOpenClawChrome)"]
    CDP["CDP / Playwright\n(pw-session, pw-ai)"]
  end

  Tool --> Direct
  Tool --> GatewayRPC
  Tool --> NodeProxy
  Tool --> SandboxBridge

  Direct --> HTTP
  GatewayRPC --> Context
  NodeProxy --> HTTP
  SandboxBridge --> HTTP

  HTTP --> Routes
  Routes --> Context
  Context --> Chrome
  Context --> CDP
  Chrome --> CDP
```

---

## 二、启动链路

### 2.1 Gateway 是否启动 Browser Control

```mermaid
sequenceDiagram
  participant GW as Gateway 启动
  participant SB as server-browser.ts
  participant Server as browser/server.ts
  participant Lifecycle as runtime-lifecycle.ts

  GW->>SB: startBrowserControlServerIfEnabled()
  alt OPENCLAW_SKIP_BROWSER_CONTROL_SERVER=1
    SB-->>GW: null（不启动）
  else 正常
    SB->>SB: 懒加载 ../browser/server.js
    SB->>Server: startBrowserControlServerFromConfig()
    Server->>Server: resolveBrowserConfig(), ensureBrowserControlAuth()
    Server->>Server: Express.listen(127.0.0.1, controlPort)
    Server->>Server: registerBrowserRoutes(app, ctx)
    Server->>Lifecycle: createBrowserRuntimeState()
    Lifecycle-->>Server: state
    Server-->>SB: BrowserServerState
    SB-->>GW: { stop }
  end
```

### 2.2 Browser Control 服务与路由

```mermaid
flowchart LR
  subgraph server_ts["server.ts"]
    A[loadConfig] --> B[resolveBrowserConfig]
    B --> C{enabled?}
    C -->|否| D[return null]
    C -->|是| E[ensureBrowserControlAuth]
    E --> F[Express + 中间件]
    F --> G[registerBrowserRoutes]
    G --> H[createBrowserRuntimeState]
  end

  subgraph routes["routes/index.ts"]
    G --> R1[registerBrowserBasicRoutes]
    G --> R2[registerBrowserTabRoutes]
    G --> R3[registerBrowserAgentRoutes]
  end

  subgraph agent_routes["Agent 子路由"]
    R3 --> A1[agent.snapshot]
    R3 --> A2[agent.act]
    R3 --> A3[agent.debug]
    R3 --> A4[agent.storage]
  end
```

---

## 三、本地 Chrome 启动与 CDP

### 3.1 按需启动 Chrome

```mermaid
sequenceDiagram
  participant Req as 某次请求
  participant Ctx as BrowserRouteContext
  participant Avail as server-context.availability
  participant Chrome as chrome.ts
  participant Proc as Chrome 进程

  Req->>Ctx: forProfile(name).ensureBrowserAvailable()
  Ctx->>Avail: ensureBrowserAvailable()
  Avail->>Avail: getProfileState().running?
  alt 已有 running
    Avail-->>Ctx: 直接返回
  else 需要启动
    Avail->>Avail: 若 attachOnly/remote 则附加或报错
    Avail->>Chrome: launchOpenClawChrome(resolved, profile)
    Chrome->>Chrome: resolveBrowserExecutable, userDataDir
    Chrome->>Chrome: spawn(exe, --remote-debugging-port=cdpPort, ...)
    Chrome->>Proc: 启动浏览器
    Chrome-->>Avail: RunningChrome
    Avail->>Avail: setProfileRunning(running)
    Avail->>Avail: 轮询 isChromeCdpReady 直至就绪
    Avail-->>Ctx: 完成
  end
```

### 3.2 Chrome 启动内部流程（chrome.ts）

```mermaid
flowchart TB
  subgraph launch["launchOpenClawChrome"]
    L1[ensurePortAvailable]
    L2[resolveBrowserExecutableForPlatform]
    L3[resolveOpenClawUserDataDir]
    L4[userDataDir 存在?]
    L5[spawn --remote-debugging-port=cdpPort]
    L6[--user-data-dir=... about:blank]
    L7[needsBootstrap? 首次创建 prefs]
    L8[decorateOpenClawProfile 染色]
    L9[再次 spawn 正式运行]
  end

  L1 --> L2 --> L3 --> L4
  L4 -->|否| L7 --> L5 --> L8 --> L9
  L4 -->|是| L8 --> L9
```

---

## 四、Agent 工具 browser 调用链

### 4.1 工具执行与目标解析

```mermaid
flowchart TB
  subgraph Execute["browser 工具 execute"]
    E1[读 action / profile / target / node]
    E2[resolveBrowserNodeTarget]
    E3{nodeTarget?}
    E4[resolveBrowserBaseUrl]
    E5[proxyRequest = callBrowserProxy]
    E6[baseUrl = control 本机]
  end

  E1 --> E2 --> E3
  E3 -->|有 node| E5
  E3 -->|无 node| E4 --> E6

  E5 --> NodeInvoke["Gateway node.invoke\nbrowser.proxy"]
  E6 --> ClientFetch["client-fetch\n→ 127.0.0.1:18791"]
```

### 4.2 本机 Host 请求完整链路

```mermaid
sequenceDiagram
  participant Agent as Agent
  participant Tool as browser-tool.ts
  participant Actions as browser-tool.actions
  participant Client as client.ts / client-actions
  participant Fetch as client-fetch.ts
  participant HTTP as Browser Control HTTP
  participant Dispatcher as routes/dispatcher
  participant Context as server-context
  participant Avail as ensureBrowserAvailable
  participant CDP as CDP / Playwright

  Agent->>Tool: execute("browser", { action: "snapshot", ... })
  Tool->>Tool: resolveBrowserNodeTarget / resolveBrowserBaseUrl
  Tool->>Actions: executeSnapshotAction(baseUrl, profile, proxyRequest=null)
  Actions->>Client: browserSnapshot(baseUrl, ...)
  Client->>Fetch: fetchBrowserJson(url, init)
  Fetch->>Fetch: startBrowserControlServiceFromConfig?（如需）
  Fetch->>HTTP: GET/POST http://127.0.0.1:18791/agent/snapshot...
  HTTP->>Dispatcher: dispatch(req)
  Dispatcher->>Context: ctx.forProfile(profile)
  Context->>Avail: ensureBrowserAvailable()
  Avail->>Avail: launchOpenClawChrome 若未运行
  Avail-->>Context: 就绪
  Context->>CDP: 执行 snapshot（Playwright/CDP）
  CDP-->>Context: 结果
  Context-->>Dispatcher: 响应
  Dispatcher-->>HTTP: JSON
  HTTP-->>Fetch: body
  Fetch-->>Client: 解析结果
  Client-->>Actions: SnapshotResult
  Actions-->>Tool: AgentToolResult
  Tool-->>Agent: 工具返回
```

### 4.3 经 Node 代理的请求

```mermaid
sequenceDiagram
  participant Agent as Agent
  participant Tool as browser-tool
  participant Gateway as Gateway
  participant Node as Node Host
  participant Invoke as invoke-browser
  participant Dispatcher as createBrowserRouteDispatcher
  participant Context as Browser Control Context

  Agent->>Tool: execute(..., nodeTarget 有值)
  Tool->>Tool: callBrowserProxy({ nodeId, method, path, ... })
  Tool->>Gateway: callGatewayTool("node.invoke", { command: "browser.proxy", ... })
  Gateway->>Node: 转发 node.invoke
  Node->>Invoke: runBrowserProxyCommand(params)
  Invoke->>Invoke: ensureBrowserControlService()
  Invoke->>Invoke: createBrowserRouteDispatcher(createBrowserControlContext())
  Invoke->>Dispatcher: dispatch({ method, path, query, body })
  Dispatcher->>Context: 同本机路由逻辑
  Context-->>Dispatcher: 结果
  Dispatcher-->>Invoke: { result, files? }
  Invoke-->>Node: 返回
  Node-->>Gateway: payload
  Gateway-->>Tool: BrowserProxyResult
  Tool-->>Agent: 工具返回
```

---

## 五、关键文件索引

| 角色                          | 路径                                                               |
| ----------------------------- | ------------------------------------------------------------------ |
| Gateway 是否启动 browser 控制 | `src/gateway/server-browser.ts`                                    |
| Browser HTTP 服务（完整）     | `src/browser/server.ts`                                            |
| Browser 仅服务状态（无 HTTP） | `src/browser/control-service.ts`                                   |
| 路由注册与 Context            | `src/browser/routes/index.ts`、`src/browser/server-context.ts`     |
| 按 profile 保证浏览器可用     | `src/browser/server-context.availability.ts`                       |
| 停止所有 profile              | `src/browser/server-lifecycle.ts`                                  |
| 启动本地 Chrome 进程          | `src/browser/chrome.ts`（`launchOpenClawChrome`）                  |
| Tab / 打开页面（CDP）         | `src/browser/server-context.tab-ops.ts`                            |
| Agent 用 snapshot/act 等 API  | `src/browser/routes/agent.ts`、`agent.snapshot.ts`、`agent.act.ts` |
| 工具 "browser" 定义与执行     | `src/agents/tools/browser-tool.ts`、`browser-tool.actions.ts`      |
| 本机发 HTTP 到 control        | `src/browser/client.ts`、`src/browser/client-fetch.ts`             |
| Node 上执行 browser.proxy     | `src/node-host/invoke-browser.ts`                                  |
| Gateway 侧 RPC 转 browser     | `src/gateway/server-methods/browser.ts`                            |

---

## 六、配置与端口

- **Browser Control 端口**：由 `gateway.port` 推导（默认 `18791`，即 gateway + 2），见 `src/config/port-defaults.ts`。
- **CDP 端口**：每 profile 可配 `cdpPort`（如 openclaw 默认 `18800`），或远程 `cdpUrl`。
- **启用/禁用**：`browser.enabled` 为 `false` 时，不启动 Control 服务，工具会报 "browser control disabled"。

---

_文档基于当前 main 分支代码整理，用于理解 OpenClaw 本地浏览器的代码逻辑与数据流。_
