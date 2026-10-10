# ZCode 代码分析报告（总览索引）

> 生成方式：基于 `codebase-analyzer` 流程的静态代码分析（只读），粒度为功能 / 函数 / 数据结构 / 类级别。
> 分析时间：2026-10-10；基线 commit：`872ad96`（main，与 origin/main 同步）。
> 按要求，本报告不包含 AI 自动化建议与 Skill 蓝图，仅代码分析。

## 项目快照

| 项 | 值 |
| --- | --- |
| 项目 | ZCode v3.14.0 — AI 编程工作台（桌面 / Web / 终端 Agent 三形态） |
| 技术栈 | TypeScript 6、React 19.2、Electron、Vite、tsup、Hono、Zustand、zod（v3+vod4 双版本共存）、node:sqlite、node-pty、pnpm 10.33.2 / Node 24.14.0（mise 锁定） |
| 仓库形态 | pnpm monorepo：`packages/*` 14 包 + `apps/zcode-cli`（内嵌独立 workspace，17 个子包） |
| 代码规模 | 约 3880 个 TS/JS 源文件；最大两包：`packages/ui`（1477）、`apps/zcode-cli`（1354） |
| 构建 | turbo（zcode-cli 内部）、tsup（desktop/server）、Vite（web/ui renderer）、esbuild+SEA（CLI 单文件可执行） |
| 测试 | 仅 4 个 `node:test` 文件（services/test×3、ui/test×1）；E2E（WDIO）基建存在但配置未随开源发布 |
| CI/CD | 无 `.github/workflows`；CI 语义（计时日志、镜像回退）保留在脚本中 |
| Git | 仅 2 个提交（开源快照），无法从历史推断演进 |

## 三种产品形态与入口

| 形态 | 入口命令 | 进程组成 |
| --- | --- | --- |
| 桌面版 | `pnpm dev:desktop` | Electron main + renderer + 每窗口一个 Local Host（utilityProcess）+ cron scheduler（utilityProcess）+ Agent 子进程（zcode-cli，stdio） |
| Web / 命令行版 | `pnpm dev:web` 或发行包 `zcode --web` | Hono HTTP+WS server + 浏览器 + Agent 子进程（spawn，stdio） |
| 终端 Agent | `zcode`（TUI）/ `zcode app-server --stdio` | TUI（OpenTUI+React 19）直连 AgentRuntime；app-server 供 Host JSON-RPC 驱动 |

## 报告目录

| 报告 | 内容概要 |
| --- | --- |
| [01-architecture.md](01-architecture.md) | 全局架构：包依赖图、分层与进程模型、RPC/协议栈总览、核心设计模式 |
| [02-operation-principles.md](02-operation-principles.md) | 运行原理：三条入口的启动时序、消息全链路数据流、核心状态机汇总、错误处理链 |
| [03-workflow.md](03-workflow.md) | 开发工作流：bootstrap / dev / build / 打包发布 / 架构治理 / 质量工具链 |
| [packages/01-shared-protocol.md](packages/01-shared-protocol.md) | `@zcode/shared`：协议数据结构全景（v3/v4）、zod 校验体系、workspace hook 信任链 |
| [packages/02-rpc.md](packages/02-rpc.md) | `@zcode/rpc` + `@zcode/client`：VS Code 风格 IPC 框架、二进制序列化、重连 |
| [packages/03-provider.md](packages/03-provider.md) | `@zcode/provider`(-node) 等：模型配置三层合成、Registry、model-option-map CEL |
| [packages/04-services.md](packages/04-services.md) | `@zcode/services`：业务服务清单、SQLite 任务索引、设置/凭据/插件/技能 |
| [packages/05-desktop.md](packages/05-desktop.md) | `@zcode/desktop`：Main 启动序列、Host/attachment、scheduler、更新 |
| [packages/06-ui.md](packages/06-ui.md) | `@zcode/ui`：Zustand store 全景、hooks 服务层、v4 会话投影、组件体系 |
| [packages/07-zcode-cli.md](packages/07-zcode-cli.md) | Agent CLI 与运行时：入口分发、AgentRuntime turn 循环、权限、CommandInbox、TUI、动态工作流 |
| [packages/08-web-server.md](packages/08-web-server.md) | `@zcode/web` / `@zcode/server` / `@zcode/server-cli`：HTTP/WS、远程部署、守护进程 |

## 核心发现（Top 10）

1. **双协议并存**：旧 JSON-RPC over stdio（`zcode-protocol/index.ts`，3717 行）与 V4「快照 + 增量 + 命令 + 物理分片」协议（`zcode-protocol-v4/`，47 文件）共存，V4 用 deliveryProfile 区分桌面实时链路（`desktop-continuous`，30ms flush）与手机/Web 恢复链路（`web-remote-replayable`，150ms flush）。
2. **重业务全部下沉 Agent 子进程**：Host（services）只是装配层 —— 插件下载、模型执行、会话状态机的真身在 `zcode-cli` 进程内（事件溯源 + node:sqlite），Host 经 `ZCodeProtocolClient`（stdio JSON-RPC）驱动，且为插件管理与 MCP 状态维护独立控制面进程。
3. **RPC 框架为 VS Code IPC 移植**：自定义二进制序列化（类型标签 + VQL 变长整数）、13 字节帧头、懒事件订阅、PersistentProtocol ACK/重放、ProxyChannel 自动代理（`packages/rpc/src/index.ts:1-29` 分层图）。
4. **两级输入队列**：协议层 `CommandInbox`（bootstrap，幂等 + 每 session FIFO 串行 admission）与 core 层 `RuntimeCommandQueue`/`admitPrompt`（busy 权威，steer/queue 车道）—— 对应 AGENTS.md「已接受输入串行 admission」约束。
5. **状态所有者纪律严格**：Main 不承载 task/session 业务状态；每窗口一个 window-scoped Local Host；远程 workspace 由窗口内连接注册表管理；外部 relay 只做鉴权转发（代码均有对应实现与注释佐证）。
6. **workspaceIdentity 贯穿**：身份键 `workspaceIdentity?.trim() || workspacePath` 的规范实现在 `packages/shared/src/task-realtime-core.ts:78`，远程身份格式 `remote:ssh:<host>:<port>:<user>:<path>`（`remote-workspace-identity.ts:46`）。
7. **持久化双轨**：Host 侧 `~/.zcode/v2/setting.json` + `credentials.json`（原子写 + 文件锁）与 `tasks-index.sqlite`（node:sqlite，冻结账本迁移，Worker 预热）；Agent 侧事件溯源 SQLite（messages/parts/events 多仓）。
8. **权限模型五模式**：`build/edit/plan/yolo/auto`（`core/src/permission/service.ts:97-231`），工具执行经 PreToolUse hook 覆写 + broker/hook 竞速裁决。
9. **安全边界 Fail-closed**：host capability 一次性 TTL 票据防重放提权（`server/src/hostCapability.ts`）；workspace hook 信任 store 非法即整体拒绝；zcode-cua 为占位包全部不可用。
10. **zod3/zod4 双实例边界**：`@zcode/shared` 用 zod4、`apps/zcode-cli/packages/contracts` 用 zod3，禁止互相组合 schema，打包期 `createZodDedupePlugin` 强校验（`cli/scripts/build.mjs:74-104`）。

## 分析模式与可信度说明

- 大型项目（3880 文件）→ 采样分析：**全部子目录职责覆盖 + 代表文件深读**（每包报告末尾附深读/采样清单）。
- 所有论断附 `file:line` 证据；两处与文档（README）不符的事实已在报告中纠正：`zcode --web` 的入口实际在 `scripts/zcode-distribution/runner.mjs`（发行包 `bin/zcode.mjs`），而非 `packages/zcode-server-cli`；`zcode-server-cli` 本体是守护进程 CLI（serve/status/stop/restart/update/uninstall）。
- 静态分析未运行项目；运行时行为结论均以代码路径与注释为依据。
