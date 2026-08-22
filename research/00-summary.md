# Codex 系列全方位研究汇总对比 + dsh-codex 落地蓝图

> 汇总 01–07 七份研究报告(详见各分报告,均附 file:line 证据)。研究日期:2026-08。

## 一、四仓库全景对比

| 维度 | openai/codex | codex-security | codex-action | codex-plugin-cc |
|---|---|---|---|---|
| **本质** | Rust 核心 monorepo(~140 crates)+ TS CLI + SDK | LLM 驱动安全审计 CLI/SDK | GitHub Action 封装(`codex exec` 单次执行) | Claude Code 插件(JSON-RPC 客户端) |
| **规模** | 6492 文件 / ~110 万行 | 299 文件 | 30 文件 / ~2100 行 | 63 文件 / v1.0.6 |
| **与宿主交互方式** | app-server v2 JSON-RPC(stdio/ws/unix)+ 官方 TS/Python SDK + library API(ThreadManager→CodexThread.submit) | CLI + SDK 共用 CodexSecurity 类;docker 批量扫描器 | spawn `codex exec` + stdin prompt + `--output-last-message` 文件通道 | 自实现 JSONRPC(JSONL over stdio)连 `codex app-server`,broker 单例共享 |
| **工具系统** | ~25 内置 handler,每轮动态装配,深度 MCP 双向集成 | 14 个安全插件 skills(威胁模型/调查员扇出/假阳性反馈) | 只传 prompt,不编排 GitHub 上下文 | 8 个 /codex:* 命令 + 后台任务状态机 |
| **沙箱** | macOS Seatbelt / Linux Landlock+seccomp+bubblewrap / Windows 双令牌 | cap_drop ALL + seccomp + AppArmor 容器 | drop-sudo 两阶段降权 | 无(继承 Codex) |
| **模型** | Responses API(SSE+WS),model-provider 抽象(openai/bedrock/ollama/lmstudio) | 同 codex | 同 codex | 同 codex |
| **配置** | ConfigToml 分层 + AGENTS.md 逐级发现注入 | ScanOptions 13 回调 + 预算熔断 | 20 inputs,受保护参数校验 | 命令 md + companion 脚本分离 |
| **对 DSH 最有价值** | ① app-server 协议 + 官方 SDK ② library API ③ ToolOrchestrator 审批→沙箱→重试 ④ 每轮工具装配 ⑤ MCP 双向 | ① 威胁模型+独立审计员+调查员扇出 ② 成本熔断 ③ 假阳性反馈回路 ④ 密封产物四件套 ⑤ 可恢复扫描契约 | ① 文件结果通道 ② 认证/执行分离 ③ spawn 前参数校验 ④ fake-CLI 可测试性 | ① app-server 客户端可照搬 ② 命令/脚本分离 ③ 作业状态机+文件持久化 ④ 流式事件 captureTurn ⑤ 会话转移闭环 |

## 二、关键机制裁决(做成插件怎么选)

1. **执行引擎**:优先 **spawn `codex` + app-server JSON-RPC 客户端**(plugin-cc 已验证的路径,支持后台任务/流式事件/持久线程),降级为 `codex exec` 文件通道(codex-action 路径,已实现于 dsh-codex v0.1.0)。不引入 @openai/codex-sdk 依赖(官方 action/plugin 都不用它,保持零第三方依赖)。
2. **安全审计**:不重实现引擎;把 codex-security 的**方法论提示词**(core-scan.md、threat-model.md、4 个 JSON Schema)吸收为 DSH skill,执行走 codex CLI。
3. **代码审查**:复用 plugin-cc 的 review 命令设计(prompt 拼 git diff + 结果回传)。
4. **配置面**:支持 AGENTS.md 注入(codex 核心能力),ConfigToml 分层加载思想用于 dsh-codex 自己的配置。

## 三、dsh-codex 落地蓝图(已完成/待办)

### ✅ 已完成(v0.1.0,commit d16775b)
- TS + Hono 工程化:`src/index.ts` → tsc → `dist/index.js`(Node16 模块解析,TS 7.0)
- Hono 内嵌 API:`createHonoApp(ctx)` 工厂,`/api/codex/status` + `/api/codex/health`,`app.fetch` 零端口嵌入
- 工具:`codex_status`(环境检测:binary/CODEX_HOME/key)+ `codex_exec`(spawn + stdin + 文件通道)
- 安全:受保护参数校验(PROTECTED_ARGS)、认证/执行分离
- 本地类型 shim(`types/dsh-tools.d.ts`),零宿主类型依赖
- GitHub:[snow-The/dsh-codex](https://github.com/snow-The/dsh-codex)

### ✅ 插件生态 Hono 化(07 报告,本轮完成)
- **dsh-gitkit / dsh-plugin-doctor / dsh-snapshot** 已按 dsh-codex 同款模式加
  `createHonoApp` 工厂(health/version 端点)+ `ctx.http?.mount` 尝试挂载;
  hono 4.13.3 运行时依赖;冒烟测试全部 200。commit:gitkit `8581187`、
  doctor `5a13d0c`、snapshot `962a976`。
- 三个插件遗留手写 `lib/` 已清除(双份代码问题),统一 src → dist 单一链条
  (gitkit `6879415`、doctor `d56d566`、snapshot `e44efe2`)。
- 7 个自研插件全部使用最新 `ctx.tools.register` API,无过时调用。

### 🔄 待办(Roadmap)
1. app-server JSON-RPC 客户端(codex_task 后台任务 / codex_review / codex_result / codex_cancel)
2. AGENTS.md 支持 + git diff 收集
3. 安全审计 skill(codex-security 方法论移植)
4. 作业状态机 + 文件持久化

## 四、本机环境事实(影响插件设计)

- **无 codex 二进制、无 ~/.codex、无 OPENAI_API_KEY** → 插件必须先检测 + 引导安装(`npm i -g @openai/codex`)
- Node v26.7.0 / npm 11.19.0;npm 被 DSH 注入的 `npm_config_allow_scripts` 环境变量锁死项目级安装 → **插件开发/装依赖用 pnpm**
- TS 7.0 移除 `moduleResolution: node10`,需 Node16/nodenext

## 五、技能去重结论(06)

dsh-skill-pack 39 → 29(删 10:4 宿主包装壳 + 2 三方重复 + 4 包内重复;TDD 三方中保留 skill-pack 通用版)。已落地 commit 7ffab1a,已装副本同步。
