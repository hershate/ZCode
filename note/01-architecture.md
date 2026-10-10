# 01 · 全局架构

> 证据均以当前检出源码为准（commit `872ad96`）。粒度：包 / 模块 / 关键类与数据结构。

## 1. 技术栈全景

| 层 | 技术 | 证据 |
| --- | --- | --- |
| 语言/运行时 | TypeScript 6、Node ≥24（mise 锁 24.14.0）、ESM 优先（root `"type":"module"`） | `package.json:2,79-80`、`mise.toml:1-3` |
| 桌面 | Electron（main/preload/host/scheduler 四 bundle，tsup） | `packages/desktop/tsup.config.ts:139-239` |
| UI | React 19.2.7（pnpm overrides 锁定）、Zustand（无 persist 中间件，手写持久化）、tailwind 系 DESIGN token | `package.json:83-86`、`DESIGN.md` |
| Web | Vite + React；后端 Hono + `@hono/node-server` + `@hono/node-ws` | `packages/server/package.json:19-21`、`src/http.ts:6-8` |
| 终端 | TUI：OpenTUI + React 19（非 ink） | `apps/zcode-cli/packages/tui/package.json` |
| RPC | 自研 VS Code 风格 IPC（二进制序列化 + 13 字节帧 + 懒事件） | `packages/rpc/src/index.ts:1-29` |
| 协议校验 | zod 4.6.5（shared/services/ui）+ zod 3.24（cli contracts），双实例隔离 | `packages/shared/package.json`、`apps/zcode-cli/packages/contracts/package.json:100` |
| 持久化 | node:sqlite（`DatabaseSync`）+ JSON 文件（原子写 + 文件锁） | `services/src/session/tasksDatabase/startup.ts:6-8` |
| 终端模拟 | node-pty（Host 侧懒加载）+ @xterm/xterm（UI） | `services/src/terminal/terminalService.ts:28-43` |
| 模型接入 | @ai-sdk（anthropic / openai-compatible，带 video 输入 patch） | root `package.json:87-91`、`patches/` |
| 质量 | oxlint（max-lines 400）、oxfmt、knip、tsc -b 12 项目引用 | `.oxlintrc.json`、`package.json:29` |

## 2. 包依赖图

依赖方向（导入方 → 被导入方），由各包 `package.json` 的 `workspace:*` 声明归纳：

```mermaid
graph TD
    subgraph 终端形态
        CLI["apps/zcode-cli<br/>17 个子包独立 workspace"]
    end
    subgraph 应用层
        DESKTOP["@zcode/desktop<br/>Electron 壳"]
        WEB["@zcode/web<br/>浏览器壳"]
        SERVER["@zcode/server<br/>Hono HTTP/WS"]
        SRVCLI["@zcode/server-cli<br/>守护进程"]
    end
    subgraph 共享层
        UI["@zcode/ui<br/>全部业务 UI"]
        SERVICES["@zcode/services<br/>Host 业务服务"]
        CLIENT["@zcode/client<br/>Agent 客户端 SDK"]
        RPC["@zcode/rpc<br/>IPC 框架"]
    end
    subgraph 契约层
        SHARED["@zcode/shared<br/>协议+类型+zod"]
        PROVIDER["@zcode/provider<br/>模型配置领域"]
        PNODE["@zcode/provider-node"]
        MOM["@zcode/model-option-map"]
        CUA["@zcode/zcode-cua<br/>fail-closed 占位"]
        FPROOF["@zcode/formal-proof"]
    end

    DESKTOP --> UI
    DESKTOP --> SERVICES
    DESKTOP --> SERVER
    DESKTOP --> SHARED
    DESKTOP --> CLIENT
    WEB --> UI
    WEB --> CLIENT
    WEB --> SHARED
    SERVER --> SERVICES
    SERVER --> SHARED
    SRVCLI --> SERVER
    UI --> SERVICES
    UI --> SHARED
    SERVICES --> RPC
    SERVICES --> SHARED
    SERVICES --> PROVIDER
    CLIENT --> RPC
    CLIENT --> SERVICES
    RPC --> SHARED
    PROVIDER --> SHARED
    PROVIDER --> MOM
    PNODE --> PROVIDER
    CLI --> SHARED
    CLI --> PROVIDER
    CLI --> FPROOF
    CLI --> CUA
```

要点：

- `@zcode/shared` 是零 IO 契约包：运行时依赖仅 `zod` + `@zcode/model-option-map`；例外是 workspace-hook 子族依赖 `node:fs/node:crypto`，经独立 subpath 暴露（`packages/shared/src/zcode-protocol-v4/core.ts:2` 纪律注释）。
- 「contracts → shared、永不反向」是 CLI 侧铁律（`apps/zcode-cli/packages/contracts/.../workflow-runs-reducer.ts:11` 注释）。
- `apps/zcode-cli` 是嵌套 pnpm workspace（自带 lockfile + turbo），但其构建 inputs 显式包含外层仓库的 `packages/shared|provider` 源码（`apps/zcode-cli/turbo.json:57-80`）——跨仓源码消费。

## 3. 分层与进程模型（桌面形态）

```mermaid
graph TD
    subgraph Electron主进程["Electron Main（1 个，不承载业务状态）"]
        M1["窗口/菜单/托盘/更新"]
        M2["TaskRealtimeBus / BroadcastHub"]
        M3["BrowserGuestManager（CDP on WebContentsView）"]
        M4["消息转发：ServicePort 转移、Host↔scheduler"]
    end
    subgraph Renderer["Renderer（每主窗 1 + 更新窗等）"]
        R1["@zcode/ui Root（全部业务 UI）"]
        R2["ChannelClient over MessagePort"]
    end
    subgraph LocalHost["Local Host（utilityProcess，每主窗 1）"]
        H1["createLocalServices（services/node.ts:1281）"]
        H2["attachment 注册表 + 远程连接注册表"]
        H3["ZCodeAgentProcessManager"]
    end
    subgraph Scheduler["Cron Scheduler（utilityProcess，单例）"]
        S1["20s 轮询认领 automations / off-peak"]
    end
    subgraph Agent["Agent 子进程（zcode-cli，每 workspaceKey 1）"]
        A1["AgentRuntime（事件溯源）"]
        A2["V4Gateway + CommandInbox"]
        A3["MCP 子进程 / subagent / 工具"]
    end

    R2 -- "MessagePortMain 直连（main 只转移端口）" --> H1
    M4 -- "InitLocal / postMessage" --> H1
    H3 -- "spawn + stdio JSON-RPC" --> A1
    M4 -- "cron/offpeak dispatch" --> S1
    H2 -- "SSH / WSL / Docker" --> REMOTE["远端 zcode-server（stdio）"]
```

- 进程类型共 5 类：Main、Renderer、Local Host、Cron Scheduler、Agent（及其派生 MCP/PTY/CUA Helper 等）——清单与通道见 [packages/05-desktop.md](packages/05-desktop.md) §10。
- 通信通道四族：`PlatformChannels`（renderer↔main，约 110 个，`packages/shared/src/channels.ts:158-424`）、`HostMessageTypes`/`HostResponseTypes`（main↔host，30/47 个，`channels.ts:505/559`）、`ServiceChannels`（RPC 服务频道 40+，`channels.ts:74`）、scheduler 私有协议（`schedulerProtocol.ts`）。

## 4. 协议栈总览

```mermaid
graph TD
    subgraph 传输层
        T1["MessagePort（renderer↔host，免分帧+流控帧）"]
        T2["WebSocket（web/手机 ↔ server；/ws 与 /ws/host）"]
        T3["stdio（host ↔ Agent 子进程，LF 帧 NDJSON）"]
        T4["SSH/WSL/Docker stdio（远端 zcode-server）"]
    end
    subgraph RPC框架["@zcode/rpc（自定义二进制）"]
        C1["ChannelClient / ChannelServer"]
        C2["ProxyChannel：服务自动代理"]
        C3["PersistentProtocol：ACK/重放/饱和"]
    end
    subgraph Agent协议["@zcode/shared zcode-protocol"]
        P1["v3 legacy：JSON-RPC 帧 zcodeProtocolMessageSchema（index.ts:326）"]
        P2["v4：hello/subscribe/frame/command（transport.ts/command.ts）"]
    end

    T1 --> C1
    T2 --> C1
    C1 --> C2
    C1 --> C3
    T3 --> P1
    T3 --> P2
    T4 --> P1
```

- RPC 框架与 Agent 协议是**两套正交体系**：前者承载「UI ↔ Host 服务」（任意 IServiceAccessor 调用），后者承载「Host ↔ Agent 子进程」的会话/任务语义。
- V4 wire 层定义物理分片（1MiB 帧上限、crc32、base64 分片装配），逻辑层定义快照 + 五种封闭增量操作与 33 个命令（详见 [packages/01-shared-protocol.md](packages/01-shared-protocol.md)）。

## 5. 关键设计模式（附位置）

| 模式 | 实现 | 位置 |
| --- | --- | --- |
| 服务定位器 / DI | `ServiceCollection` + `ServiceDescriptor`（channel 名 + phantom 类型），`exposeOnChannelServer` 自动注册 | `services/src/collection.ts:12-41`、`descriptors.ts:14-16` |
| 自动代理（ES6 Proxy） | `ProxyChannel.fromService/toService` 把服务接口转 RPC 代理 | `packages/rpc/src/proxy-channel.ts` |
| 事件溯源 | Agent 会话 = 事件流（appendEvent → sqlite），投影由 `EventReducer` 重放 | `core/src/runtime/methods/events.ts:81-147`、`contracts/src/events/event-reducer.ts:74` |
| 端口-适配器 | core 依赖 contracts 端口接口，Node 实现在 adapters | `apps/zcode-cli/packages/contracts`、`adapters` |
| 租约/代际守卫 | TaskRunLease、restartGeneration、writeGeneration、invalidationVersion | `desktop/src/main/taskRealtimeBus.ts:142-183`、`zcodeAgentProcessManager.ts:988-999` |
| 单飞（single-flight） | Agent 进程启动并发收敛 `startingByWorkspaceKey` | `zcodeAgentProcessManager.ts:832-887` |
| 乐观更新 + 权威收敛 | UI optimisticCommands overlay → 2s 宽限自愈；taskUnreadOverlay | `ui/src/v4/conversationProjectionStore.ts:1202-1296`、`taskQueryCacheStore.ts:373-383` |
| 防回环广播 | `applyingBroadcast` 闭包标志 + `BROADCAST_FIELDS` 白名单 | `ui/src/store/index.ts:214,436-488` |
| 原子写 + 文件锁 | setting.json/credentials.json：tmp→rename + withFileLock | `services/src/setting/settingService.ts:184-218`、`credential/credentialService.ts:104-115` |
| 受控上下文 | `architecture:context` 输出模块契约阅读包 | `scripts/architecture/index.mjs:325-359` |

## 6. 架构治理（目标态 vs 现状）

`architecture-policy.yaml` 定义 14 个模块；唯一 `managed: true` 的是 `services/storage`（domain→app→adapters 三层，公开入口 `contract.ts`）。`session` 模块声明的 `session/contract.ts` 在磁盘上不存在（仍为 legacy）——策略是目标态而非现状。检查算法（行数/契约行数/公开方法数/domain-io/层向/深导入/环检测 + baseline 指纹 + `--changed` 反向依赖扩集）详见 [03-workflow.md](03-workflow.md) §4。

## 7. 三形态共用内核

桌面 renderer、Web 浏览器、TUI 共用同一套业务内核：

- **UI 共用**：`@zcode/ui` 的 `Root`（`packages/ui/src/Root.tsx:104`）；桌面与 Web 仅注入不同 `IPlatformService`（`desktopPlatform.ts` vs `createWebPlatform`，`web/src/main.tsx:190-352`）。
- **Agent 共用**：`zcode-cli` 的 `AgentRuntime` 同时服务 TUI（进程内直连）、Desktop Host（spawn + stdio）、Web server（spawn + stdio）。
- **投影共用**：V4 snapshot/delta 的装配与应用逻辑（shared 的 `apply.ts`）被桌面 store、Web 回放页（`ui/src/replay.ts` re-export 桶）与 TUI 侧共同复用。
