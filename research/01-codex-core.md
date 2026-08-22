# OpenAI Codex 主仓库深度研究报告(codex-rs 核心)

> 研究对象:`C:\Users\snow\codex-zone\codex`(monorepo,~6492 文件,含 codex-rs / codex-cli / sdk / docs)
> 代码位置证据格式:`相对路径:行号`(相对仓库根)

## 速览(10 行)

1. **几乎全部智能在 Rust**:`codex-rs/` 是约 140 个 crate 的 Rust workspace(edition 2024),承载 agent 循环、工具、模型客户端、沙箱;`codex-cli/` 只是发布 `@openai/codex` npm 包的一个 `bin/codex.js` 壳。
2. **Agent 主循环**:`run_turn()`(`codex-rs/core/src/session/turn.rs:153`)内 `loop`(:301)反复「采样模型 → 流式处理响应 → 执行工具 → 结果回填」,直到 response completed;没有显式 plan/act/observe 三阶段,plan 只是一个工具(`update plan`)。
3. **核心公开类型**:`CodexThread`(`core/src/codex_thread.rs:202`,`submit(Op)` :247)、`ThreadManager`(`core/src/thread_manager.rs`,start_thread :905)、`Session`/`TurnContext`/`ToolRouter`,`ToolName`(`protocol/src/tool_name.rs:12`)。
4. **工具系统**:`ToolRegistry` + `ToolExecutor` trait(tool_name/spec/handle 三方法),每个内置工具一个 handler 文件;每轮 turn 按 feature/模型能力动态装配(`core/src/tools/spec_plan.rs:889`)。
5. **MCP 是二等公民之外的深度集成**:`codex-mcp` crate 用 `rmcp`(Rust MCP SDK)聚合任意 MCP server 的工具/资源;另有 `mcp-server` crate 把 Codex 自身暴露为 MCP server(反向嵌入)。
6. **模型协议 = OpenAI Responses API**:`core/src/client.rs:162` 硬编码 `/responses` 与 `/responses/compact`,HTTP SSE + WebSocket 双通道;模型/提供商经 `model-provider` crate 抽象(openai/chatgpt/bedrock/ollama/lmstudio 等)。
7. **沙箱三件套**:macOS Seatbelt(`sandboxing/src/seatbelt.rs`)、Linux Landlock+seccomp+bubblewrap(`linux-sandbox/`、`sandboxing/src/landlock.rs`)、Windows 提权/受限令牌后端(`sandboxing/src/windows.rs`、`windows-sandbox-rs/`);命令执行在独立的 `exec-server` 进程(可与 app-server 异机)。
8. **可嵌入性是显式设计**:`codex-core` 就是 library,公开 `ThreadManager`/`CodexThread`/`ModelClient`/`Prompt`/`ResponseStream`;外层还有 JSON-RPC 2.0 `app-server`(stdio/ws/unix socket)与官方 TS/Python SDK。
9. **配置体系**:`config.toml`/`config.toml` 分层(用户/项目/requirements/云托管),`AGENTS.md` 从项目根到 cwd 逐级发现并作为「用户指令」注入(`core/src/agents_md.rs`)。
10. **对宿主集成**:最有价值的是「agent 即 library(ThreadManager/CodexThread)+ 每轮工具装配(registry/spec_plan)+ 审批→沙箱→重试编排(orchestrator)+ app-server 协议 + 扩展 API(ext/)」这条完整可插拔链路。

---

## 1. 整体架构:Rust 核心与 TS/CLI 的分工

### 1.1 目录分工

| 顶层 | 内容 |
|---|---|
| `codex-rs/` | 全部核心逻辑,Rust workspace(`codex-rs/Cargo.toml:1-139` 列出全部 ~140 成员 crate) |
| `codex-cli/` | npm 发布壳:`"bin": { "codex": "bin/codex.js" }`(`codex-cli/package.json:6-8`),`files` 只含 `bin/codex.js`,无业务代码 |
| `sdk/` | 官方嵌入层:`sdk/typescript`(Codex/Thread/runStreamed)、`sdk/python`(openai_codex 包,含 generated/v2_all.py) |
| `docs/` | 用户文档(config.md、sandbox.md、exec.md、agents_md.md、skills.md 等) |
| `codex-rs/docs/` | 内部协议文档(`codex_mcp_interface.md`、`protocol_v1.md`) |

### 1.2 进程模型(重要)

- `codex-rs/cli/` 是 TUI 入口(`cli/src/main.rs`);`codex-rs/exec/` 是 `codex-exec` 非交互入口(`exec/src/main.rs`)。
- **app-server 是中枢**:`codex app-server` 用 JSON-RPC 2.0 双向协议驱动 Codex 的富界面(VS Code 扩展、SDK 都连它),支持 stdio / websocket / unix socket 传输(`codex-rs/app-server/README.md:1-33`)。
- **exec-server 是执行器**:独立进程负责 shell/文件操作,`exec-server/src/lib.rs` 有 `local_process`、`process_sandbox`、`sandboxed_file_system`、`remote`/`forward`(远程执行器);AGENTS.md 明确「app-server 与 exec-server 可运行在不同操作系统」。
- `exec/src/main.rs:1-14` 还有个巧妙的 arg0 派发:同一个 `codex-exec` 二进制,以 `codex-linux-sandbox` 名字被调用时就扮演 Linux 沙箱助手(Landlock + seccomp)。

### 1.3 Agent 主循环(plan→act→observe 的真相)

- 没有独立的 plan 阶段。主循环在 `core/src/session/turn.rs`:`run_turn()`(:153)建立 `TurnContext`、注入 AGENTS.md/skills/plugins 上下文,然后 `loop`(:301)内反复:
  1. 取 pending input(:305)
  2. `run_sampling_request()`(:1340)发起 `/responses` 流式请求,流内 `handle_non_tool_response_item` / `handle_output_item_done`(`stream_events_utils.rs`)消费文本/推理/工具调用
  3. 工具调用经 `ToolRouter`(`core/src/tools/router.rs`)、`ToolCallRuntime`(`core/src/tools/parallel.rs`)并发执行,结果以 `function_call_output` 回填
  4. 直到 `response.completed`,turn 结束
- 「plan」只是模型可选调用的一个工具:`PlanHandler`(`core/src/tools/handlers/plan.rs:25`),参数 `UpdatePlanArgs`(`protocol/src/plan_tool.rs`),作用仅「Plan updated」+ 更新模型可见的计划项。
- 核心类型:`CodexThread`(`core/src/codex_thread.rs:202`)— 「conduit for the bidirectional stream of messages」,`submit(Op)` :247;`Session`(`core/src/session/session.rs`);`TurnContext`(`core/src/session/turn_context.rs:144`);`Op`/`EventMsg`(`protocol/src/protocol.rs`);多 agent 由 `AgentControl`(`core/src/agent/control.rs`)管理(spawn/send_message/wait/interrupt/list,有 v1 与 v2 两套)。

---

## 2. 工具系统

### 2.1 内置工具清单(全部 handler 在 `codex-rs/core/src/tools/handlers/`)

| 工具 | 文件 | 说明 |
|---|---|---|
| exec_command / write_stdin(新 shell,即旧 `shell`) | `handlers/unified_exec/exec_command.rs`、`write_stdin.rs` | 统一执行器,managed sandbox 下运行命令 |
| shell(旧)/ zsh_fork | `handlers/shell_spec.rs`、`runtimes/zsh_fork/` | macOS 本地 zsh 直启路径 |
| apply_patch | `handlers/apply_patch.rs` + `apply_patch.lark` | 语法解析(git 风格 patch,grammar 用 Lark) |
| plan | `handlers/plan.rs` | 更新计划 |
| view_image / test_sync / current_time / sleep | `handlers/view_image.rs`、`test_sync.rs`、`current_time.rs`、`sleep.rs` | 工具类小件 |
| tool_search | `handlers/tool_search.rs`(`tools/src/tool_discovery.rs:6` 定义 `TOOL_SEARCH_TOOL_NAME`) | 工具检索,供延迟暴露 |
| mcp / mcp_resource | `handlers/mcp.rs`、`handlers/mcp_resource/`(list/read/templates) | MCP 工具与资源 |
| request_permissions / request_user_input | `handlers/request_permissions.rs`、`request_user_input.rs` | 权限/提问 |
| multi_agents v1/v2 | `handlers/multi_agents/`(spawn/send_input/wait/close/resume)、`handlers/multi_agents_v2/`(spawn/send_message/followup_task/interrupt/list/wait) | 子 agent |
| new_context_window / get_context_remaining | `handlers/new_context_window.rs`、`get_context_remaining.rs` | token 预算 |
| request_plugin_install / list_available_plugins_to_install | `handlers/request_plugin_install.rs` 等 | 插件市场 |
| send_user_message_async / wait_for_environment / dynamic / extension_tools | `handlers/send_user_message_async.rs`、`wait_for_environment.rs`、`dynamic.rs`、`extension_tools.rs` | 扩展面 |

### 2.2 注册与调度

- 统一 trait:`CoreToolRuntime`(或 `ToolExecutor`)三方法 `tool_name()` / `spec()` / `handle()`(例如 `handlers/current_time.rs:50-83`),返回 `ToolExecutorFuture`。
- `ToolRegistry`(`core/src/tools/registry.rs:271`):`register_trusted()` :302 / `register_external()` :336(带 exposure 的变体控制模型可见性:`ToolExposure::{Direct, Deferred, DirectModelOnly}`)。
- **每轮装配,不是全局注册**:`build_core_tool_registry`(`spec_plan.rs:249`)→ `add_core_tool_sources`(:889)→ `add_shell_tools`(:958)/`add_mcp_resource_tools`(:1001)/`add_core_utility_tools`(:1010)/`add_collaboration_tools`(:1118),全部按 feature flag、模型能力(`model_info.experimental_supported_tools`)、环境(environments)条件注册,再 `build_tool_router`(:117)产出模型可见 specs。
- 名称带命名空间:`ToolName { name, namespace }`(`protocol/src/tool_name.rs:12`),默认 `functions` 命名空间,`Display` 用 `/` 拼接(:58)——这就是模型看到的 `functions/exec_command` 之类名字的由来。

### 2.3 MCP 集成(重点)

- 客户端聚合:`codex-mcp` crate —— `McpRuntime`/`McpBinding`/`McpConnectionSet`(`codex-mcp/src/connection_manager.rs:3-12` 注释:聚合所有 RMCP 客户端、汇总工具与资源、处理启动状态);基于 `rmcp = "=3.1.3"`(`codex-rs/Cargo.toml:401`,即 Rust MCP SDK)。
- 模型可见性:`tool_is_model_visible`(`codex-mcp/src/lib.rs:12`)按配置决定 MCP 工具是否进模型上下文;`core/src/mcp_tool_exposure.rs` 做暴露策略。
- Codex 自身也可作为 MCP server:`mcp-server/` crate(`mcp-server/src/main.rs`、`codex_tool_runner.rs`、`exec_approval.rs`、`patch_approval.rs`)—— 其他 agent framework 可以通过 MCP 协议调用 Codex 的工具与 turn(反向嵌入,见 §5.4)。
- 还有 `ext/mcp`(agent 扩展)与 `codex-mcp` 的 `tool_catalog_cache.rs`(工具目录缓存)。

### 2.4 执行编排

`ToolOrchestrator`(`core/src/tools/orchestrator.rs:1-8` 注释自述):「approval → select sandbox → attempt → retry with escalated sandbox strategy on denial(no re-approval thanks to caching)」——审批缓存 + 沙箱升级重试是执行路径的核心语义。

---

## 3. 模型接口

### 3.1 协议:OpenAI Responses API

- `core/src/client.rs:162-163`:`RESPONSES_ENDPOINT = "/responses"`、`/responses/compact`(压缩用 unary)。
- 传输:HTTP SSE(`stream_responses_api` :1462,`codex-client/src/sse.rs`)与 WebSocket(`stream_responses_websocket` :1603;`client.rs:1060 connect_websocket`)双通道,`ModelClientSession`(:300)turn 级缓存 WS + sticky routing。
- 请求携带 session/thread 元数据、originator、attestation 头(`build_responses_options` :1209);`codex-api` crate 定义 `ResponsesApiRequest`/`ApiResponsesOptions` 等 wire 类型。

### 3.2 模型/提供商抽象

- `model-provider` crate:`ModelProvider` trait(`model-provider/src/provider.rs`,lib.rs:21),`create_model_provider`(lib.rs:29)按配置构造;`runtime.rs` 是运行时(每个 provider 实例);`models_endpoint.rs` 拉取远程模型列表。
- 内置提供商(`model-provider-info/src/lib.rs`):`openai`(:39)、`amazon-bedrock`(:42)、`amazon-bedrock-runtime`(:44)、`ollama`(:499)、`lmstudio`(:498),`WireApi` 枚举(:64)决定走 Responses 协议还是 Chat 协议;`config_toml.rs:292-293` 的 `model_providers` map 支持自定义 provider(base_url/wire_api/认证)。
- `models-manager` crate:`SharedModelsManager`(`models-manager/src/manager.rs`,RefreshStrategy)缓存模型目录;`ThreadManager::list_models`(`thread_manager.rs:759`)。

### 3.3 `--model` 与 reasoning effort

- CLI:`utils/cli/src/shared_options.rs:23 pub model: Option<String>` 是 `--model` 旗标来源;也支持 `-c 'model="o3"'` 配置覆盖(`utils/cli/src/config_override.rs:26`)。
- 配置键:`config_toml.rs:354 model_reasoning_effort`、`:355 plan_mode_reasoning_effort`、`default_subagent_reasoning_effort`(:681);`ReasoningEffort` 枚举在 `protocol/src/openai_models.rs:50`。
- 模型信息:`ModelInfo`(`protocol/src/openai_models.rs:389`,含 `use_responses_lite` :458、`shell_type`、`apply_patch_tool_type` 等),驱动工具装配与行为开关;模型切换在 TUI 里有 `increase/decrease_reasoning_effort` 快捷键(`config/src/tui_keymap.rs:131-133`)。

---

## 4. 沙箱与安全

### 4.1 三平台实现(`codex-rs/sandboxing/src/`)

| 平台 | 机制 | 证据 |
|---|---|---|
| macOS | **Seatbelt**(`sandbox-exec`):内置 4 个 `.sbpl` 策略(base/network/preferences/restricted_read_only_platform_defaults)+ 运行时拼装;`seatbelt.rs:21-33` | `seatbelt.rs:18-33` |
| Linux | **bubblewrap + seccomp**(默认),旧路径 Landlock;`create_linux_sandbox_command_args_for_permission_profile`(`landlock.rs:37`)把 permission profile JSON 传给 `codex-linux-sandbox` 助手 | `landlock.rs:14`(arg0 常量)、`linux-sandbox/src/{bwrap,landlock,main}.rs` |
| Windows | **两种后端**:提权后端(Elevated)与受限令牌后端(RestrictedToken);`windows_sandbox_uses_elevated_backend`(`windows.rs:38`)、`permission_profile_supports_windows_restricted_token_sandbox`(:45) | `windows.rs:38-53`、`windows-sandbox-rs/` |

依赖:`landlock = "0.4.4"`(`Cargo.toml:358`)、`seccompiler = "0.5.0"`(:411);AGENTS.md 还提到 macOS Seatbelt 子进程会带 `CODEX_SANDBOX=seatbelt` 环境变量(测试据此跳过)。

### 4.2 执行链路

- shell 工具运行在 **exec-server 进程**(`exec-server/src/lib.rs`:`process.rs`/`process_sandbox.rs`/`local_process.rs`/`remote_process.rs`/`sandboxed_file_system.rs`/`fs_sandbox.rs`),与 app-server 解耦,可远程(`forward`/`remote`),这也是 AGENTS.md「app-server 与 exec-server 可异机」的由来。
- 每轮权限由 `PermissionProfile`(`protocol/src/models.rs`)驱动:Managed(受管,读写根/网络策略)/Disabled/External;`core/src/tools/sandboxing.rs` 提供 `SandboxManager`/`SandboxType`/`SandboxOverride`。
- 审批体系:`utils/approval-presets/`(预设)、`tools/approvals.rs`(审批上下文)、`tools/network_approval.rs`(网络审批)、`guardian/`(模型自审 reviewer 角色)、`execpolicy` crate(deny 规则,`core/src/exec_policy.rs`)。

---

## 5. 可嵌入性(SDK 层与分层边界)

### 5.1 Rust library 层:agent 即 library

- `codex-core` 是 library(非 binary),`core/src/lib.rs` 公开:`CodexThread`、`ThreadManager`、`NewThread`、`StartThreadOptions`、`ModelClient`、`ModelClientSession`、`Prompt`、`ResponseStream`、`McpManager`、`RolloutRecorder`、`TurnInput` 等(见 lib.rs 各 `pub use`,如 :100-126);并有 `#[deprecated]` 别名 `ConversationManager = ThreadManager`(:110)。
- **宿主调用路径**:`ThreadManager::new(Config)`(:241)→ `start_thread(StartThreadOptions)`(:905)拿 `NewThread` → `CodexThread::submit(Op)`(`codex_thread.rs:247`)→ 订阅事件流;还支持 `resume_thread_from_rollout`(:996)、`fork_thread`(:1176)、`spawn_subagent`(:961)、`shutdown_all_threads_bounded`(:1124)。
- 这正是「CodexAgent struct + run 方法」的等价物(名字是 ThreadManager/CodexThread,不是 CodexAgent)。

### 5.2 扩展点:Extension API

- `ext/extension-api` crate:`turn_lifecycle.rs`(TurnStart/TurnStop/TurnAbort/TurnError 贡献者)、`turn_input.rs`、`context.rs`(TurnContextContributionInput);`ext/` 下有 goal、mcp、skills、web-search、memories、queue、connectors、items 等一整套插件式 crate,`ExtensionRegistry` 注入 Session(`thread_manager.rs` 依赖)。
- hooks 体系:`core/src/hook_runtime.rs`(run_pending_session_start_hooks / run_turn_stop_hooks / run_legacy_after_agent_hook)——生命周期事件外挂。

### 5.3 进程协议层:app-server v2(官方 SDK 的底座)

- JSON-RPC 2.0 双向(`app-server/README.md:15-33`),传输 stdio / ws / unix socket;schema 可 `codex app-server generate-ts` 导出。
- `app-server-protocol/src/protocol/v2.rs`(thread/turn/model 资源,`thread/start`、`turn/start`、`turn/steer`、`turn/interrupt` 等 `*Params/*Response` 命名规范);SDK 的 TS `src/exec.ts`/`thread.ts` 与 Python `openai_codex/client.py` 都走这条协议。
- TS SDK 形态(`sdk/typescript/src/codex.ts:13-47`):`Codex` 类 → `startThread()` / `resumeThread(id)` → `Thread.runStreamed(input)`(`thread.ts:66`)返回 `AsyncGenerator<ThreadEvent>`;Python SDK 同构(sync/async 双客户端,`sdk/python/src/openai_codex/`)。

### 5.4 反向嵌入:MCP server

`mcp-server/` crate 让任意 MCP client 驱动 Codex(turn 注册表 `active_turn_registry.rs`、工具 runner `codex_tool_runner.rs`、执行/补丁审批 `exec_approval.rs`/`patch_approval.rs`)——对「宿主集成」而言这是零代码接入点。

---

## 6. 配置体系

- 核心类型 `ConfigToml`(`config/src/config_toml.rs:155`):顶层键含 `model`、`model_provider`(:162)、`model_providers`(:292)、`model_reasoning_effort`(:354)、`plan_mode_reasoning_effort`(:355)、`approval_policy`、`sandbox_mode`、`agents`(:667 `AgentsToml`/`AgentRoleToml`)等;`config.schema.json` 由 `just write-config-schema` 生成(AGENTS.md 约定)。
- 分层加载:`config/src/loader/mod.rs`(用户 `~/.codex/config.toml`、项目、session 覆盖)、`profile_toml.rs`(profiles)、`requirements_layers/`(requirements.toml + 托管层)、`cloud_config`(云端下发 bundle);`config.md` 文档指向 developers.openai.com 的完整参考。
- **AGENTS.md 体系**(`core/src/agents_md.rs`):
  - `DEFAULT_AGENTS_MD_FILENAME = "AGENTS.md"`(:42),`AGENTS_OVERRIDE_FILENAME`(:43-46,即 `AGENTS.override.md` 优先);
  - 发现规则:从项目根(按 git 边界)向下到 cwd 收集每级 AGENTS.md(:201),全局 `~/.codex/AGENTS.md` 由 `codex-home/src/instructions/mod.rs:9` 读取;
  - 注入格式:`# AGENTS.md instructions for {cwd}` + `<INSTRUCTIONS>…</INSTRUCTIONS>`(测试快照可见 `core/tests/suite/agents_md.rs:132`),且有大小上限与预算管理(:492 附近「max bytes to embed」)、cwd 变更时替换/失效语义(`context/world_state/agents_md.rs`)。
- 其他:execpolicy(execpolicy crate)、skills(`skills.md`、`codex-rs/skills/`)、plugins(`utils/plugins/`、`core-plugins`)。

---

## 7. 对宿主集成最有价值的 5 个设计(file:line 证据)

1. **每轮工具装配的 ToolRegistry/`CoreToolRuntime` trait** —— 工具不是全局静态注册,而是按 turn 的 config/feature/模型能力动态 `registry.add(handler)`;`spec_plan.rs:889 add_core_tool_sources`、`registry.rs:271 ToolRegistry`、任意 handler 的 `tool_name/spec/handle`(如 `handlers/current_time.rs:50-83`)。→ DSH 插件可以精确控制「这个会话模型能看到哪些工具」。
2. **Agent 即 library:`ThreadManager + CodexThread::submit(Op)`** —— `thread_manager.rs:905 start_thread`、`:782 get_thread`、`:1176 fork_thread`;`codex_thread.rs:247 submit`。宿主进程内嵌一个完整 Codex 会话,事件经 `SessionIo` 双向流。→ 这是「把 Codex 变成 DSH 的 agent 后端」的直接 API。
3. **审批→沙箱→重试的执行编排(ToolOrchestrator)** —— `tools/orchestrator.rs:1-8` 注释即设计说明:approval → sandbox select → attempt → 升级沙箱重试、审批结果缓存不重复询问。→ DSH 的安全执行管线可以照搬这个顺序与缓存语义。
4. **app-server v2 JSON-RPC 协议 + 官方 SDK** —— `app-server/README.md:15-33`(stdio/ws/unix)、`app-server-protocol/src/protocol/v2.rs`(thread/turn 资源与 `*Params/*Response` 命名)、`sdk/typescript/src/codex.ts:13` 与 `sdk/python/src/openai_codex/`。→ 任何语言宿主都能复用官方 SDK 的协议形状,DSH 可直接把「Codex 会话」包装成可编程服务。
5. **Extension API + hooks + MCP 双向** —— `ext/extension-api/src/contributors/turn_lifecycle.rs`(TurnStart/TurnStop/TurnAbort)、`core/src/hook_runtime.rs`(session start / turn stop / after-agent 钩子)、`codex-mcp/src/connection_manager.rs`(聚合外部 MCP server)、`mcp-server/`(把 Codex 暴露成 MCP server)。→ DSH 既可以通过 MCP 把自身工具喂给 Codex,也可以作为 MCP client 消费 Codex 工具,双向零胶水。

---

## 对 DSH 插件的启示(5 行)

1. 若要「用 Codex 干活」:优先走 `codex app-server` 的 JSON-RPC v2(stdio 传输)+ 官方 TS/Python SDK,而不是自己解析 TUI 输出;`thread/start`、`turn/start`、`turn/steer`、`turn/interrupt` 就是全部控制面。
2. 若要「深度嵌入」:直接依赖 `codex-core`,用 `ThreadManager::start_thread` + `CodexThread::submit(Op)` + 事件流,宿主只需提供 `SessionIo` 与 config;这是官方为 VS Code 扩展/SDK 铺的路。
3. 安全执行可直接借鉴 `ToolOrchestrator` 的「审批→沙箱→升级重试、审批缓存」管线与 `PermissionProfile` 模型,DSH 不必重新发明。
4. MCP 双向集成是零成本入口:DSH 插件做成 MCP server → 任何 Codex 会话可调用;反过来 `codex-mcp` 客户端也能让 Codex 会话消费 DSH 的 MCP 工具。
5. 配置与上下文注入参照 AGENTS.md 机制(项目根→cwd 逐级发现、`# AGENTS.md instructions` 包裹格式、字节上限与预算)做 DSH 的 workspace 指令注入;每轮工具按 feature 装配的「所见即所得」思路对 DSH 的预设/角色系统很有参考价值。
