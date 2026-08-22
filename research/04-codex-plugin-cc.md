# 深度研究:OpenAI Codex Plugin for Claude Code(`codex-plugin-cc` v1.0.6)

## 速览

- 仓库:`C:\Users\snow\codex-zone\codex-plugin-cc`,63 文件,`package.json` name=`@openai/codex-plugin-cc` v1.0.6,Apache-2.0,纯 Node ESM(devDeps 仅 `typescript`/`@types/node`,零运行时依赖)。
- 用途:让 Claude Code 用户通过 `/codex:*` 斜杠命令调用本机 Codex CLI 做代码审查、任务委派、后台作业与会话转移——"从现有工作流直接用上 Codex"。
- 入口:`plugins/codex/commands/*.md`(8 个命令)+ `agents/codex-rescue.md`(1 个子代理)+ `hooks/hooks.json`(SessionStart/SessionEnd/Stop)+ 单一 CLI `scripts/codex-companion.mjs`。
- 核心机制:不 spawn Codex 交互式 CLI,而是实现 **JSON-RPC over stdio 客户端**连接 `codex app-server`(每行一个 JSON 消息),并可选经 **unix socket/命名管道 broker** 共享单实例。
- 流式:发起 `turn/start` 后服务端推送 `thread/started`、`item/*`、`turn/completed` 等通知,插件把 agent 消息/reasoning/文件变更翻译成进度日志与最终结果。
- 后台:detached 自 spawn `task-worker` 子进程 + `state.json`/`jobs/*.json` 持久化,`/codex:status`、`/codex:result`、`/codex:cancel` 三命令闭环。
- 门控:可选的 Stop hook 用 Codex 审查上一轮 Claude 输出,`BLOCK:` 即拦截会话结束(默认关闭)。
- 转移:`externalAgentConfig/import` 把 Claude transcript(JSONL)导入 Codex,输出 `codex resume <thread-id>` 以便在 Codex 侧续跑。
- 测试:自制 `fake-codex-fixture.mjs`(一个模拟 app-server 的假 `codex` 二进制,注入 PATH)+ 8 个测试文件约 90 用例;CI 在 ubuntu 装真 Codex CLI 跑 `npm test` + `npm run build`。
- **无 MCP server、无网络服务**:全部是本机子进程 + 文件状态,与 DSH 插件体系(cordis 插件 + 工具注册 + 子进程 + 后台任务)映射非常直接。

---

## 1. 这是什么

Claude Code **插件(plugin marketplace 格式)**:一个由 OpenAI 官方维护、随 npm 包 `@openai/codex-plugin-cc` 分发(仓库根即 marketplace 仓库)的插件,让 Claude Code 用户无需离开现有工作流即可调用本机 Codex CLI:

- 代码审查:普通 review(`/codex:review`,read-only,等价于 Codex 内置 `/review`)与对抗式审查(`/codex:adversarial-review`,可带 focus 文本,challenge 设计取舍)。
- 任务委派:`/codex:rescue` 把调查/修复/续跑类任务交给 Codex(可 `--background`/`--wait`/`--resume`/`--fresh`/`--model`/`--effort`),甚至直接说"Ask Codex to ..."由模型自动路由到 `codex:codex-rescue` 子代理。
- 后台作业管理:`/codex:status`(查看运行/最近作业)、`/codex:result`(取最终输出)、`/codex:cancel`(取消)。
- 会话转移:`/codex:transfer` 把当前 Claude 会话历史导入 Codex 并打印 `codex resume <session-id>`。
- 环境检查:`/codex:setup`(检测 Codex 是否安装/认证,可代装,可开关 review gate)。
- 可选 Stop 门控:开启后每次 Claude 回合结束前由 Codex 审查,发现问题即 `block` 该回合。

证据:`README.md:10-15`(What You Get 与命令清单)、`README.md:22-48`(安装:`/plugin marketplace add openai/codex-plugin-cc` → `/plugin install codex@openai-codex` → `/reload-plugins` → `/codex:setup`)、`README.md:267-269`("wraps the Codex app server","uses the global `codex` binary"——不内置独立 runtime)、`README.md:302-311`(复用同一 Codex 安装/认证/配置)。

## 2. 插件结构

两级清单,均为纯声明 + 约定目录,`plugin.json` 本身极简:

**marketplace 级** `.claude-plugin/marketplace.json`:`name: openai-codex`,`metadata.version: 1.0.6`,`plugins: [{name: "codex", source: "./plugins/codex", version: 1.0.6}]`(`.claude-plugin/marketplace.json:1-21`)。

**plugin 级** `plugins/codex/.claude-plugin/plugin.json` 只有 `name/version/description/author`(`plugins/codex/.claude-plugin/plugin.json:1-8`)——**没有显式声明 commands/hooks/agents/skills**。Claude Code 插件规范中这些由目录约定自动注册:命令 = `commands/*.md`(frontmatter + 正文即注入给模型的指令),agent = `agents/*.md`,skill = `skills/*/SKILL.md`,hook = `hooks/hooks.json`。各声明与实现文件的对应:

| 声明 | 文件 | 内容 / 实现 |
|---|---|---|
| 8 个斜杠命令 | `commands/{review,adversarial-review,rescue,transfer,status,result,cancel,setup}.md` | frontmatter:`description`/`argument-hint`/`allowed-tools`/`disable-model-invocation`;正文 = 给 Claude 的操作指令(何时前台/后台、如何调用 companion 脚本) |
| 1 个子代理 | `agents/codex-rescue.md` | `model: sonnet`、`tools: Bash`、`skills: [codex-cli-runtime, gpt-5-4-prompting]`;正文 = "thin forwarding wrapper"(只转发一次 `task` 调用) |
| 3 个 hook | `hooks/hooks.json` → `scripts/session-lifecycle-hook.mjs`(SessionStart/End,timeout 5s)、`scripts/stop-review-gate-hook.mjs`(Stop,timeout 900s) | SessionStart 把 session_id/transcript_path 写入 `CLAUDE_ENV_FILE`;SessionEnd 关停 broker、清理作业;Stop 条件触发 Codex 审查 |
| 3 个 skill | `skills/codex-cli-runtime/SKILL.md`、`skills/codex-result-handling/SKILL.md`(均 `user-invocable: false` 内部契约)、`skills/gpt-5-4-prompting/SKILL.md`(带 3 个 references) | 约束子代理只准转发、结果展示规则、prompt 改写规则 |
| 唯一 CLI 入口 | `scripts/codex-companion.mjs`(1073 行) | 子命令分发:`setup/review/adversarial-review/task/task-worker/transfer/status/result/task-resume-candidate/cancel`(`codex-companion.mjs:1024-1067`) |
| app-server 客户端/协议 | `scripts/lib/app-server.mjs`(JSON-RPC 客户端)、`scripts/lib/codex.mjs`(业务编排)、`scripts/lib/app-server-protocol.d.ts`(协议类型,prebuild 生成) | 见第 3 节 |
| 状态/作业 | `scripts/lib/state.mjs`、`tracked-jobs.mjs`、`job-control.mjs` | 见第 3/4 节 |
| broker | `scripts/app-server-broker.mjs`、`scripts/lib/broker-lifecycle.mjs`、`broker-endpoint.mjs` | 共享 app-server 的单例转发器 |
| prompt 模板 | `prompts/adversarial-review.md`、`prompts/stop-review-gate.md` | 渲染后作为 turn 的 prompt 文本 |
| 输出 schema | `schemas/review-output.schema.json` | 对抗式审查的结构化输出契约(verdict/summary/findings/next_steps) |
| 发布 | `scripts/bump-version.mjs` | 同步 4 处版本号 |
| 构建 | `package.json:14-16` | `prebuild` = `codex app-server generate-ts --out plugins/codex/.generated/app-server-types`(从真 Codex 生成协议类型),`build` = tsc |

## 3. 实现机制:如何让 Claude Code 调起 Codex

**不是** spawn `codex exec`/`codex -` 之类交互式 CLI,也不是 MCP server,而是自实现的 **JSON-RPC(JSONL,每行一个 JSON)客户端连 `codex app-server` 子进程**:

- 直连模式:`spawn("codex", ["app-server"], {cwd, stdio: ["pipe","pipe","pipe"], shell: win32 ? (SHELL||true) : false})`(`app-server.mjs:190-196`);stdout 用 `readline` 逐行 `JSON.parse`(`app-server.mjs:220-223`),发请求写 `JSON.stringify(msg)+"\n"` 到 stdin(`app-server.mjs:268-275`);握手 `initialize` → `initialized`(`app-server.mjs:225-229`)。`request(method, params)` 用递增 id + pending map 实现请求/响应配对(`app-server.mjs:86-98`),服务端主动推送的 notification 走 `notificationHandler`(`app-server.mjs:151-153`)。
- **没有 --json/--format json 之类 CLI 标志**;CLI 级只有 `codex --version` 与 `codex app-server --help` 探活(`codex.mjs:886-904`)。
- 共享 broker 模式(默认开启):`ensureBrokerSession` 先起一个 detached 的 `app-server-broker.mjs serve --endpoint <unix-socket|命名管道> --cwd ... --pid-file ...`(`broker-lifecycle.mjs:59-70,113-171`),broker 内部持有一个 `disableBroker: true` 的直连客户端(`app-server-broker.mjs:68`),把每个连入 socket 的 JSON-RPC 消息转发给它;同一时刻只允许一个请求(其余返回 `BROKER_BUSY_RPC_CODE=-32001` "Shared Codex broker is busy."),但允许 `turn/interrupt` 穿插(`app-server-broker.mjs:170-195`);streaming 方法(`turn/start`/`review/start`/`thread/compact/start`)期间的 notification 只路由回发起者,`turn/completed` 后释放(`app-server-broker.mjs:12,84-100,197-221`)。这样一次 Claude 会话内多个 `/codex:*` 调用共享同一个 Codex runtime。broker 端点由 `CODEX_COMPANION_APP_SERVER_ENDPOINT` env 或 `broker.json` 状态文件记录(`app-server.mjs:338-347`;`broker-lifecycle.mjs:72-93`);broker 忙时客户端自动降级直连(`codex.mjs:613-642 withAppServer` 的 retry 逻辑)。
- 流式输出:发起 `turn/start`(参数:`input: [{type:"text", text: prompt, text_elements:[]}]`、`model`、`effort`、`outputSchema`、threadId——`codex.mjs:86-88,1132-1144`)后,`captureTurn` 注册 notification handler(`codex.mjs:559-611`),处理 `thread/started`、`turn/started`、`item/started`、`item/completed`(agentMessage/reasoning/fileChange/commandExecution/collabAgentToolCall/webSearch/enteredReviewMode/exitedReviewMode)、`turn/completed`、`error`(`codex.mjs:490-557`),把每条翻译成进度事件:stderr `[codex] ...` + 追加 job 日志(`tracked-jobs.mjs:117-132`),最终汇总 `lastAgentMessage`(final_answer)、`reasoningSummary`、`fileChanges`/`touchedFiles`、`commandExecutions`(`codex.mjs:1146-1159`)。
- 沙箱与审批:`thread/start` 参数 `approvalPolicy: "never"`、`sandbox: "read-only"`(review)或 `"workspace-write"`(task `--write`)(`codex.mjs:63-72,491,1013,1116`;`codex-companion.mjs:414,491`)。
- 审查路径还可用 Codex 内置 reviewer:`review/start {threadId, delivery:"inline", target:{type:"uncommittedChanges"|"baseBranch", branch}}`(`codex.mjs:1022-1030`),结果取 `exitedReviewMode` 项的 `review` 文本(`codex.mjs:449-460`)。
- 中断:`turn/interrupt {threadId, turnId}`(`codex.mjs:960-1000`),随后 `terminateProcessTree`(Windows 用 `taskkill /PID <pid> /T /F`,非 Windows 用进程组 SIGTERM——`process.mjs:57-118`)。
- 会话转移:`externalAgentConfig/import {migrationItems:[{itemType:"SESSIONS", details:{sessions:[{path, cwd}]}}]}`(`codex.mjs:681-699,1058-1093`),等待 `externalAgentConfig/import/completed` 通知,再从 `~/.codex/external_agent_session_imports.json` ledger 里按 `source_path + content_sha256` 找 `imported_thread_id`(`codex.mjs:657-679`);`transfer` 前强制 source 必须位于 `~/.claude/projects`(`claude-session-transfer.mjs:20-43`)。
- 后台作业:`codex-companion.mjs` 用 `spawn(process.execPath, [scriptPath, "task-worker", "--cwd", cwd, "--job-id", jobId], {detached:true, stdio:"ignore"})` 重新拉起自己(`codex-companion.mjs:671-682`),worker 从存储的 job 文件读回 request 再执行(`codex-companion.mjs:838-881`);作业记录(状态/阶段/pid/threadId/turnId/result/rendered)双写 `state.json` 与 `jobs/<id>.json`,输出落 `jobs/<id>.log`(`tracked-jobs.mjs:142-204`;`state.mjs:166-191`),`MAX_JOBS=50` 自动裁剪(`state.mjs:13,80-116`)。作业状态机:`queued → running → completed|failed|cancelled`,phase 细化 `starting/running/verifying/editing/investigating/finalizing/done/failed`(`tracked-jobs.mjs:117-132`;`codex.mjs:241-300`)。
- 会话关联:SessionStart hook 把 `CODEX_COMPANION_SESSION_ID`(及 transcript path、`CLAUDE_PLUGIN_DATA`)追加进 `CLAUDE_ENV_FILE`(`session-lifecycle-hook.mjs:77-81`),companion 据此给作业打 `sessionId` 标签、status/result/cancel 默认只显示当前会话的作业(`codex-companion.mjs:294-316`);SessionEnd hook 发送 `broker/shutdown`、kill 该会话的存活作业、删除 broker 会话文件(`session-lifecycle-hook.mjs:83-114`)。

## 4. 数据流:用户请求 → 插件命令 → Codex → 结果回流

关键架构决策:**命令 = Markdown 指令(给 Claude 的 prompt)+ Claude 用 Bash 工具调 companion 脚本(确定性执行)**。命令文件本身不是执行体,而是"让 Claude 学会怎么跑"的指令(例如 `commands/review.md:43-61` 的前台/后台两套 Bash 调用模板,`commands/setup.md:7-11` 的 `--json` 调用)。

典型流程(`/codex:review --base main`):

1. 用户输入 `/codex:review --base main` → Claude Code 命中 `commands/review.md`,把正文注入为对该命令的指令(frontmatter 还限制 `allowed-tools: Read, Glob, Grep, Bash(node:*), Bash(git:*), AskUserQuestion`)。
2. Claude 按指令先估算审查规模(`git status --short`/`git diff --shortstat`,见 `commands/review.md:21-29`),用 `AskUserQuestion` 问前台还是后台,然后执行 `node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" review --base main`(`commands/review.md:43-59`;后台则 `Bash({run_in_background: true})`)。
3. companion 解析参数(`lib/args.mjs` 支持值选项/布尔选项/`--` passthrough/引号与转义),`resolveReviewTarget` 算出 `working-tree` 或 `branch` 目标(`git.mjs:135-191`,脏工作树优先 working-tree,否则 `merge-base..HEAD`),`ensureCodexAvailable` + 连接 app-server(`codex-companion.mjs:358-407`)。
4. `review/start` 或 `turn/start`(adversarial 先 `collectReviewContext` 把 git status/diff/untracked 拼进 prompt 模板 `prompts/adversarial-review.md`,`codex-companion.mjs:409-417`)。
5. Codex 执行期间 notification 流 → 进度写到 stderr + `jobs/<id>.log`;结束后 companion 把 `reviewText`/`finalMessage`/`reasoningSummary` 渲染成人类可读报告(`lib/render.mjs`),stdout 输出。
6. Bash 工具把 stdout 作为工具结果返回给 Claude;Claude 按 `skills/codex-result-handling/SKILL.md` 原样呈现(**禁止自行修复、禁止改写**,`SKILL.md:10-20`),review 场景连摘要都不允许。
7. 后台场景:`task` 带 `--background` 时 companion 只打印 `started in the background as <job-id>`,之后用户(或 Claude)用 `/codex:status <id>`、`/codex:result <id>`、`/codex:cancel <id>` 轮询/取结果/取消。
8. 停止门控(可选):Claude 回合结束 → Stop hook(`stop-review-gate-hook.mjs`)把 `last_assistant_message` 拼进 `prompts/stop-review-gate.md`(要求首行 `ALLOW:`/`BLOCK:`),spawnSync 调 `codex-companion.mjs task --json`(`stop-review-gate-hook.mjs:98-140`),解析首行:`ALLOW:` → 放行;`BLOCK:` → 输出 `{decision:"block", reason}` 阻塞 stop(`stop-review-gate-hook.mjs:69-96,166-173`);配置开关存 `state.json` 的 `config.stopReviewGate`(`state.mjs:22-24`)。

## 5. 脚本与测试

**scripts/**(仓库根,2 个):
- `scripts/bump-version.mjs`:一次同步 4 处版本元数据(`package.json`、`package-lock.json` 两处、`plugins/codex/.claude-plugin/plugin.json`、`.claude-plugin/marketplace.json` 两处);`--check` 模式校验一致性(`bump-version.mjs:8-73,175-219`)。
- `scripts/bump-version.test.mjs` 覆盖:同步全部 manifest、check 模式报不一致(`tests/bump-version.test.mjs:58,74`)。

**plugins/codex/scripts/** 即插件实现(见第 2/3 节)。

**tests/**(8 个文件,`node --test`,约 90 用例;CI = ubuntu/node 22/`npm ci`/装真 Codex/`npm test`+`npm run build`,`pull-request-ci.yml:15-35`):

- 核心思路:`tests/fake-codex-fixture.mjs` 生成一个假的 `codex` 可执行文件(内含完整 JSON-RPC app-server 模拟器:initialize/account-read/config-read/thread-start/thread-resume/thread-list/review-start/turn-start/turn-interrupt/externalAgentConfig-import,并模拟 `turn/started`、`item/*`、`turn/completed` 通知流),`buildEnv` 注入 PATH(Windows 还造 `codex.cmd` wrapper)(`fake-codex-fixture.mjs:250-658`)。多种 behavior:review-ok / logged-out / refreshable-auth / auth-run-fails / provider-no-auth / api-key-account-only / invalid-json / with-reasoning / with-subagent / with-late-subagent-message / with-subagent-no-main-turn-completed / slow-task / interruptible-slow-task / external-import-unsupported / external-import-fails / adversarial-clean / config-read-fails(`fake-codex-fixture.mjs:7,68-113,193-248,303-366,407-437,588-608`)。**无网络、无真实 API**,全部端到端跑 companion 子进程。
- `tests/runtime.test.mjs`(最重):setup 就绪判定矩阵(61 行起)、review 渲染(review/start 路径,`139`)、task 转发与模型/effort 透传(`766`)、`--resume-last` 续最近 task thread(`482`)且**不跨 Claude 会话恢复**(`573,617`)、transfer 导入/升级提示/失败(`196-327`)、background 入队 detached worker(`923`)、cancel 先 turn/interrupt 再杀进程(`1542,1740`)、stop hook 门控/放行/降级(`1926-2091`)、broker 惰性启动与复用(`2119-2238`)、subagent(Codex 内部 collab agent)消息与 reasoning 日志前缀(`811`)、主线程完成后才收尾(`839-893`)。
- `tests/git.test.mjs`:target 解析(脏树优先/干净树回退 branch/显式 base/特殊字符默认分支名)、diff 上下文收集的边界(超大 diff 降级为轻量摘要、跳过目录/断链/超限/二进制 untracked)(`git.test.mjs:9-192`)。
- `tests/commands.test.mjs`:命令文档的约定校验(禁止用户可见 `continue` 命令、rescue 吸收 continue 语义、hooks 保持启用的门控等)(`commands.test.mjs:14-213`)。
- `tests/process.test.mjs`:Windows taskkill 与"进程已消失"处理;`render.test.mjs`:渲染降级;`state.test.mjs`:状态目录选址(含 `CLAUDE_PLUGIN_DATA` 覆盖)与作业裁剪;`broker-endpoint.test.mjs`:unix socket vs Windows 命名管道。

## 6. 对比价值:映射到 DSH(DeepSeek Harness)插件体系

cc 的"Claude Code 插件"与 DSH 的"cordis 插件 + 工具注册"都是宿主程序加载的外部扩展,但 cc 是 **prompt/约定驱动**(命令是 Markdown,逻辑靠宿主模型用 Bash 跑脚本),DSH 是 **工具函数驱动**(宿主把工具注册给模型直接调用)。逐项映射:

| codex-plugin-cc | DSH(cordis 插件 + 工具注册) | 说明 |
|---|---|---|
| `.claude-plugin/marketplace.json`(多插件仓库清单) | `cordis.patch.yml` 的插件安装/启用配置 + 插件清单 | cc 一个 repo 打包一个 plugin;DSH 一个插件一个包,入口注册 |
| `plugins/codex/.claude-plugin/plugin.json`(name/version/author) | 插件 package.json/入口(plugin.ts/js) | 身份元数据 |
| `commands/*.md`(命令 = 指令 prompt + 允许工具 + 参数 hint) | **tool 注册**(`ctx.set('tools.xxx')` 之类) | cc 靠"Claude 读指令后自己调 Bash";DSH 直接把 `codex:review` 编译为宿主可直接调用的工具,命令参数即工具参数——**DSH 更直接** |
| `hooks/hooks.json` SessionStart/SessionEnd | 生命周期事件(session start/end 钩子) | cc 借 SessionStart 注入 env、SessionEnd 清理;DSH 同样有会话生命周期 |
| `hooks/hooks.json` Stop(review gate) | 每轮后的 gate hook(如 DSH 的 goal 轮/回合钩子) | 同构:外部模型审查宿主回合输出,可阻断 |
| `agents/codex-rescue.md`(子代理,model/tools/skills) | DSH 子代理/预设(如 dsh-liangshen 预设) | 子代理能力声明 |
| `skills/*/SKILL.md`(user-invocable:false 内部契约) | DSH skill 目录(同名机制) | 直接复用模式:内部技能约束行为 |
| `scripts/codex-companion.mjs`(单一 CLI + 子命令) | 插件内的 lib/实现模块(工具 handler 内部逻辑) | cc 需要 CLI 是因为 Bash 工具只认进程;DSH 可直接 import 函数 |
| `scripts/lib/codex.mjs` + `app-server.mjs`(JSON-RPC 客户端 + captureTurn) | **DSH 插件内同样实现一个 `codexAppServerClient`** | 代码可几乎照搬(纯 Node ESM,零依赖):spawn `codex app-server` + JSONL RPC + notification 流 |
| `app-server-broker.mjs`(共享单实例 + 忙信号 -32001) | 可选的 DSH 侧 codex 守护/连接池 | 多会话/多工具共享一个 Codex runtime,省进程 |
| `state.mjs` + `tracked-jobs.mjs`(jobs/*.json + 状态机) | DSH 的任务/后台任务体系(如 dsh-task-board 的 Host 账本) | 概念同构:作业持久化、status/result/cancel 查询 |
| `scripts/lib/git.mjs`(target 解析 + diff 收集 + 大小边界) | DSH 工具实现(可直接 import 复用) | 纯 git 命令封装,平台无关 |
| `schemas/review-output.schema.json` + `outputSchema` 传参 | DSH 工具的 JSON Schema 输出约束 | 宿主让外部模型按 schema 产出结构化结果 |
| `scripts/bump-version.mjs` | 版本/发布脚本 | 琐碎 |
| `tests/fake-codex-fixture.mjs`(假 app-server) | DSH 测试可照搬:假 codex 二进制注入 PATH | 端到端测试无真实 API 依赖 |

结论:做一个 `dsh-codex` 插件,**cc 的整个 `lib/`(app-server 客户端、captureTurn、git 收集、tracked-jobs)可以直接以 ESM 模块形式搬进 DSH 插件**,外层把 `codex-companion.mjs` 的子命令封装成若干 DSH 工具(`codex_review`/`codex_task`/`codex_status`/`codex_result`/`codex_cancel`/`codex_transfer`),生命周期清理挂到 DSH 会话结束事件。

## 7. 对宿主集成最有价值的 5 个设计(file:line 证据)

1. **把外部 agent 包成"app-server 服务"而非裸 CLI**:JSON-RPC(JSONL over stdio)客户端 + `thread/turn` 抽象 + notification 流式事件(`app-server.mjs:183-276` spawn 与 readline 协议;`codex.mjs:490-611` captureTurn 把 `item/*` 通知翻译成阶段化进度),外加 **broker 单例共享**——一进程服务多次调用、忙时 `-32001` 退避、`turn/interrupt` 可穿插(`app-server-broker.mjs:12,170-221`)。这是"宿主把另一个 agent 当协作者"的最干净形态:起一个常驻服务、按回合驱动、按事件消费进度、可中断可复用。

2. **命令 = 指令 prompt + 确定性 companion 脚本的分离架构**:`commands/*.md` 只描述"何时前台/后台、怎么调脚本、怎么呈现结果",执行细节全部收敛到 `codex-companion.mjs` 一个 CLI(`codex-companion.mjs:1024-1067` 子命令分发;`commands/review.md:43-61` 前台/后台两套 Bash 模板)。决策(是否后台、是否问用户)交给宿主模型,执行确定性强、可测试——DSH 直接做成工具参数即可获得同样收益。

3. **作业状态机 + 文件持久化 + detached 自 spawn worker**:`queued→running→completed|failed|cancelled` 双写 `state.json`+`jobs/<id>.json`、日志落盘(`tracked-jobs.mjs:142-204`;`state.mjs:124-191`),后台 = 重新 spawn 自己(`codex-companion.mjs:671-682`),再以 `sessionId` 隔离多会话(`codex-companion.mjs:294-316`;`tests/runtime.test.mjs:573` 专门测跨会话不串)。这让"跑了半小时的 Codex 任务"与宿主进程生命周期解耦,随时可查可续可取消——DSH 后台任务体系直接对标。

4. **结构化输出契约驱动门控**:JSON Schema(`schemas/review-output.schema.json`)随 `turn/start` 的 `outputSchema` 传参让 Codex 产出 `verdict/summary/findings[]/next_steps`(`codex.mjs:1136-1142`;`codex-companion.mjs:415-441` 解析),Stop gate 则用更轻的 `ALLOW:`/`BLOCK:` 首行协议(`stop-review-gate-hook.mjs:79-96`)决定是否阻断宿主回合。外部模型输出一旦可被机器判定,就能做自动化质量闸门。

5. **可恢复性闭环:命名持久线程 + `codex resume` + 会话导入**:task 线程用 `Codex Companion Task: <摘要>` 命名并持久化(`codex.mjs:48-50,107-110,1184-1186`),`--resume-last` 自动续最近线程(`codex-companion.mjs:336-356`);`/codex:transfer` 把 Claude transcript 导入 Codex 并返回 `codex resume <id>`(`codex.mjs:681-699,1058-1093`;`claude-session-transfer.mjs:20-43` 强制 `~/.claude/projects` 边界 + sha256 ledger 去重)。委托出去的工作永不丢失,可回到 Codex 原生界面继续——对 DSH 而言就是"把 session 变成可转移资产"。

---

## 对 DSH 插件的启示

1. 照搬 `app-server.mjs`/`codex.mjs` 的 JSON-RPC 客户端与 captureTurn 即可获得完整 Codex 集成(进度、最终答案、reasoning、touched files),零外部依赖。
2. 用工具注册替代 cc 的"命令 = 指令",`codex_review(codex_task(...))` 一个工具即可覆盖全部命令,参数校验留在 DSH 层。
3. 后台任务直接映射 DSH 的 Host 侧账本/后台作业:job 状态机、日志落盘、`sessionId` 隔离、detached worker 模式照搬。
4. 把 `review-output.schema.json` + `ALLOW/BLOCK` 门控接进 DSH 的回合钩子,可做"提交前 Codex 审查"质量闸门。
5. broker 共享与 `codex resume` 会话转移是最值得抄的两个能力:省进程、且让 DSH 用户的 Codex 工作在两侧 UI 无缝续跑。
