# 深度研究:OpenAI Codex GitHub Action(`openai/codex-action`)

> 研究对象:`C:\Users\snow\codex-zone\codex-action`(30 文件:src/ 8 个 TS + dist/main.js 单文件 bundle + action.yml + test/ 6 个 .mjs)
> 结论先行:这是一个 **"受控执行 `codex exec` 的复合 GitHub Action"** —— 不是 API 编排器,不读 issue/PR 上下文,不贴评论;它只负责"安全地安装 CLI → 注入认证 → 跑一次 `codex exec` → 把最终消息写回输出文件"。GitHub 上下文拼装与结果回贴全部由工作流作者自己做。

---

## 速览

- **定位**:composite Action,运行 `codex exec` 单次任务;20 个 inputs + 1 个 output(`final-message`),默认 `safety-strategy: drop-sudo`(action.yml:1-3, 123-126)。
- **调用方式**:**spawn Codex CLI 二进制**(`npm i -g @openai/codex` 后 `spawn("codex", ["exec", ...])`),**不是** `@openai/codex-sdk`,也**不是** cloud/cloud_sdk;API 走本地 `codex-responses-api-proxy` 代理(runCodexExec.ts:313)。
- **认证**:API key 只通过 stdin 喂给后台代理进程,Codex 进程内**没有 key 环境变量**;代理把 `http://127.0.0.1:<port>/v1`(wire_api=responses)转发到 OpenAI/Azure Responses API(action.yml:222-243, writeProxyConfig.ts:23-35)。
- **GitHub 上下文**:Action 自身不拉 issue/PR/diff;README 示例由工作流用 `${{ github.event.pull_request.title }}` 拼 prompt,用 `actions/github-script` 把结果贴回 PR(README.md:56-96)。
- **结果通道**:`codex exec --output-last-message <file>` 写文件,Action 读文件后 `setOutput("final-message")`,可选再写用户指定的 `output-file`(runCodexExec.ts:262-263, 341-362)。
- **安全核心**:两套正交防护——进程级 `drop-sudo`(no_new_privs + 空 capability,不可逆)+ 参数级 `codex-args` 受保护校验(拒绝 30+ 个可提权配置根,runCodexExec.ts:540-569)。
- **测试**:6 个测试文件全部走 `dist/main.js` 真实 bundle + **fake `codex` 可执行文件放 PATH**,无 API key/网络即可验证命令构造与输出处理(runCodexExec.test.mjs:23-112)。
- **对 DSH**:「Codex 任务执行器」可直接借用其"stdin 喂 prompt + `--output-last-message` 文件读回"的流程;DSH 额外补 `git diff` 捕获即可满足"拿回 diff/总结"。
- **平台**:Windows 只能用 `unsafe`(无沙箱);Linux/macOS 支持全部 safety-strategy(action.yml:130-140)。

---

## 1. 这是什么

GitHub 官方市场 Action(`openai/codex-action`,Apache-2.0),把 [Codex CLI](https://github.com/openai/codex) 的 `codex exec` 封装成一个**复合 Action**(`runs.using: composite`,action.yml:128),让你在 workflow 里跑一次一次性 Codex 任务(改代码、审查 PR、跑实验),同时把 Codex 进程的权限压到最低。

**设计哲学**(README.md:3、docs/security.md):Action 只解决两件事 —— (a) 让 Codex 能安全地跑起来(装 CLI、配代理、降权),(b) 把最终结果拿出来。**"喂什么"和"拿结果干什么"都是工作流作者的责任**:README 的 PR review 示例里,`actions/checkout` 检出 PR merge commit,工作流把 PR 标题/正文拼进 `prompt`,`post_feedback` job 用 `actions/github-script` 把 `final-message` 发成评论。因此它是"执行器 / runner",不是"编排器 / orchestrator"。

### action.yml 全表(action.yml:4-122,README.md:99-122 一致)

| input | 含义 | 默认 |
|---|---|---|
| `prompt` | 内联 prompt 文本(`prompt` 与 `prompt-file` 二选一必填) | `""` |
| `prompt-file` | 含 prompt 的文件路径 | `""` |
| `output-file` | Codex 最终消息额外写入的路径;留空不写 | `""` |
| `openai-api-key` | API key,用于启动 Responses API 代理 | `""` |
| `responses-api-endpoint` | Responses API 端点覆盖(如 Azure `.../openai/v1/responses`);空 = 代理内置端点 | `""` |
| `working-directory` | 传给 `codex exec --cd` 的目录;默认仓库根 | `""` |
| `sandbox` | 旧式沙箱:`workspace-write`/`read-only`/`danger-full-access`;与 `permission-profile` 互斥 | `""` |
| `permission-profile` | 权限档(`:read-only`/`:workspace`/命名档),经 `default_permissions` 选择 | `""` |
| `codex-version` | 要安装的 `@openai/codex` 版本 | `""` |
| `codex-args` | 透传给 `codex exec` 的额外参数,JSON 数组或 shell 风格字符串 | `""` |
| `output-schema` | 内联 JSON Schema,写临时文件后传 `--output-schema`(与 file 互斥) | `""` |
| `output-schema-file` | Schema 文件路径 | `""` |
| `model` | 模型 | `""` |
| `effort` | 推理强度 | `""` |
| `codex-home` | Codex 配置目录;空 = CLI 默认 `~/.codex` | `""` |
| `safety-strategy` | `drop-sudo`(默认)/`unprivileged-user`/`read-only`/`unsafe` | `drop-sudo` |
| `codex-user` | `unprivileged-user` 时指定运行 Codex 的 UNIX 用户 | `""` |
| `allow-users` | 可触发该 Action 的 GitHub 用户名白名单,`*` = 所有人 | `""` |
| `allow-bots` | 允许 `github-actions[bot]` 绕过写权限检查 | `false` |
| `allow-bot-users` | 可绕过的 bot 用户名列表(不支持 `*`) | `""` |

**outputs**(action.yml:123-126):`final-message` = `codex exec` 的原始最终输出。

## 2. 核心逻辑(主流程)

整体是一个 15 步的 composite 流水线(action.yml:130-382),`dist/main.js` 是一个多命令 CLI(commander 注册 6 个子命令,main.ts:29-320):

```
Check write access (octokit) → Install Codex CLI + Responses proxy (npm -g)
→ Resolve codex-home → Start proxy (key 走 stdin, 后台进程)
→ Read proxy port → Write config.toml (注入 model_provider)
→ [Linux] 开 userns / drop-sudo 降权 → run-codex-exec
```

### 如何调用 Codex(核心:runCodexExec.ts:123-339)

1. **解析 prompt 源**(runCodexExec.ts:150-158):inline 字符串直接当输入;file 则 `readFile` 读内容。
2. **解析输出文件**(runCodexExec.ts:163-168):用户指定 `output-file` 用显式路径,否则 `mkdtemp` 建临时目录 + `output.md`。
3. **组装 argv**(runCodexExec.ts:256-292):`codex exec --skip-git-repo-check --cd <dir> --output-last-message <file>` + 可选 `--output-schema <file>` / `--model` / `--config model_reasoning_effort="..."` / 用户 `extraArgs` / 权限选择(`--sandbox <mode>` 或 `--config default_permissions="<profile>"`)。
4. **降权包裹**(runCodexExec.ts:183-254):Linux drop-sudo 时,通过 `sudo /bin/sh -c LINUX_DROP_SUDO_SCRIPT` 先做不可逆清理,再用 `/usr/bin/setpriv --reuid --regid=nobody --clear-groups --no-new-privs --bounding-set=-all ...` 启动 Codex;`unprivileged-user` 时 `sudo -u <user> --` 启动。
5. **spawn + stdin 喂 prompt**(runCodexExec.ts:311-335):`spawn(program, command, { stdio: ["pipe","inherit","inherit"] })`,`child.stdin.write(input); child.stdin.end()`——prompt 从 stdin 进,stdout/stderr 透传。
6. **读回结果**(runCodexExec.ts:341-362):进程 0 退出后 `readFile` 输出文件 → `setOutput("final-message", lastMessage)`,并清理临时目录;若以别的用户运行则 `sudo -u <user> cat` 读。
7. **环境标记**(runCodexExec.ts:294-302):设 `CODEX_INTERNAL_ORIGINATOR_OVERRIDE=codex_github_action`、`CODEX_HOME=<codex-home>`,供 Codex 内部识别来源。

### 如何"喂"GitHub 上下文、如何贴回

**代码层面:Action 完全不接触 GitHub 上下文。** 唯一的 GitHub API 调用是 `check-write-access`(校验触发者写权限,checkActorPermissions.ts:139-144,`octokit.repos.getCollaboratorPermissionLevel`)。上下文拼接与结果回贴是 README 教的工作流写法:

- 喂上下文:工作流在 prompt 里拼 `${{ github.event.pull_request.title/body }}`、commit 消息等(README.md:62-74);`actions/checkout` + `git fetch` 准备 base/head ref(README.md:36-50)。
- 拿结果:`jobs.codex.outputs.final_message: ${{ steps.run_codex.outputs.final-message }}`(README.md:33-34),下游 job 用 `actions/github-script` 的 `github.rest.issues.createComment` 贴回 PR(README.md:84-96)。

### 认证注入(代理模式)

API key 不进 Codex 进程:`codex-responses-api-proxy` 后台运行,`writeProxyConfig` 往 `CODEX_HOME/config.toml` 写 `model_provider = "codex-action-responses-proxy"` + `[model_providers.codex-action-responses-proxy] base_url="http://127.0.0.1:<port>/v1" wire_api="responses"`(writeProxyConfig.ts:23-35)。代理从 stdin 收 key(`env -u PROXY_API_KEY ... <<< "$PROXY_API_KEY"`,action.yml:240-242),并写 `server-info.json`(含 port),Action 轮询读 port(action.yml:244-264, readServerInfo.ts:8-26)。`responses-api-endpoint` 可换 upstream(如 Azure,README.md:207-223)。

## 3. CLI 调用 vs SDK 调用:结论是 **CLI spawn**

- **调用 Codex = 纯 CLI spawn**:`spawn("codex", ["exec", ...])`(runCodexExec.ts:313);Linux/macOS 非默认路径时先 `which codex` 解析绝对路径(runCodexExec.ts:201, 248)。`@openai/codex` 以**全局 npm 包**安装(action.yml:167)。
- **没有** `@openai/codex-sdk`、没有 Responses API 直连、没有 cloud SDK。SDK 依赖只有四个基础设施包(package.json:11-17):
  - `@actions/core` —— `setOutput`(runCodexExec.ts:5, 358)
  - `@octokit/rest` + `@octokit/plugin-retry` —— 写权限检查(checkActorPermissions.ts:2-5)
  - `commander` —— 子命令 CLI(main.ts:1)
  - `string-argv` —— 解析 shell 风格的 `codex-args`(main.ts:17, 348)
- **构建**:esbuild 单文件 bundle 到 `dist/main.js`(package.json:7),CI 强制 dist 与源码一致(.github/workflows/ci.yml:39-45)。

## 4. 可复用模块(test/ 覆盖 + 签名级接口)

### test/ 覆盖矩阵

| 测试文件 | 覆盖对象 | 手法 |
|---|---|---|
| `runCodexExec.test.mjs`(628 行,~40 用例) | 命令组装、权限选择、受保护参数校验的**拒绝/放行矩阵**(sandbox 默认值、profile 与 sandbox 互斥、30+ 个被禁 config 根、bypass 旗标、safe flags 透传) | fake `codex` 脚本捕获 argv + 写假输出文件,spawnSync 整程跑 `dist/main.js run-codex-exec` |
| `actionHardening.test.mjs`(282 行) | action.yml 步骤级加固:代理启动不留 key 环境变量、SIGUSR1 防护、drop-sudo 步骤条件 | 解析 action.yml + 真实进程注入环境检查 /proc |
| `checkActorPermissions.test.mjs`(182 行) | bot 白名单语义、`allow-users` 通配、Octokit 重试、401/403/404 不重试 | 本地 http server 伪 GitHub API(`GITHUB_API_URL`) |
| `dropSudo.test.mjs`(620 行) | Linux 真实降权:no_new_privs、空 capability、/run 根 socket 收紧、sudoers 清理,14 个对抗场景 | 需 Linux + sudo,真实建用户/组/socket |
| `dropSudoRootPath.test.mjs`(72 行) | root 阶段 PATH 固定,防调用者 PATH 劫持 | 伪 `id/deluser/gpasswd` 验证不被执行 |
| `linuxCredentials.test.mjs`(107 行) | 凭据序列化/反序列化、uid/gid 校验、组并集 | esbuild 内存打包 + vm 注入 spawn |

### 可复用的导出函数(签名级)

```ts
// src/runCodexExec.ts —— 核心执行器(认证在外部,天然可测)
runCodexExec({
  prompt: PromptSource,            // {type:"inline",content} | {type:"file",path}
  codexHome: string | null,
  cd: string,                      // 工作目录
  extraArgs: Array<string>,
  explicitOutputFile: string | null,
  outputSchema: OutputSchemaSource | null,
  model: string | null, effort: string | null,
  safetyStrategy: SafetyStrategy,  // "drop-sudo"|"read-only"|"unprivileged-user"|"unsafe"
  codexUser: string | null,
  sandbox: SandboxMode | null,     // "read-only"|"workspace-write"|"danger-full-access"
  permissionProfile: string | null,
}): Promise<void>                  // 副作用:setOutput("final-message", ...)
// 导出类型:PromptSource / SafetyStrategy / SandboxMode / OutputSchemaSource (runCodexExec.ts:79-112)

// src/checkActorPermissions.ts —— 纯函数,Octokit 可注入
ensureActorHasWriteAccess(options?: {
  octokit?: Octokit; token?: string; actor?: string; repository?: string;
  allowBotActors?: boolean; allowUsers?: string; allowBotUsers?: string;
}): Promise<WriteAccessCheck>      // {status:"approved",actor} | {status:"rejected",actor,reason}

// src/checkOutput.ts —— 通用子进程捕获
checkOutput(command: Array<string>): Promise<string>   // stdout 文本;非零退出 reject

// src/writeProxyConfig.ts —— 注入模型提供商配置
writeProxyConfig(codexHome: string, port: number, safetyStrategy: SafetyStrategy): Promise<void>

// src/readServerInfo.ts —— 轮询代理就绪
readServerInfo(serverInfoFile: string): Promise<void>  // 读出 {"port":N} → setOutput("port")

// src/dropSudo.ts —— 平台降权(不可逆!)
dropSudo({user, group, rootPhase, runnerCredentials}: DropSudoOptions): Promise<void>

// src/linuxCredentials.ts —— 纯函数凭据工具
parseLinuxRunnerCredentials(value: string): LinuxRunnerCredentials
captureLinuxRunnerCredentials(): LinuxRunnerCredentials
includeAccountGroups(original, accountUserId, accountGroupIds): LinuxRunnerCredentials

// 内部但值得抄的安全校验(runCodexExec.ts:472-663)
determinePermissionSelection(...): PermissionSelection
validateProtectedExtraArgs(args, customPermissionProfile): void   // 白名单外拒绝
RESTRICTED_CONFIG_ROOTS: Set<string>   // 30+ 个不可覆盖的 config 根
```

## 5. 配置面(yaml 可调项)

见第 1 节全表。要点分层:

- **任务语义**:`prompt`/`prompt-file`、`working-directory`、`output-file`、`output-schema(-file)`(让 `codex exec` 输出 JSON Schema 校验后的结构化结果)。
- **模型**:`model`、`effort`(转成 `--config model_reasoning_effort=`,runCodexExec.ts:274-278)、`codex-version`。
- **权限**(三层):
  1. 进程级 `safety-strategy`(drop-sudo / unprivileged-user / read-only / unsafe)+ `codex-user`;
  2. 沙箱/权限档 `sandbox`(旧)或 `permission-profile`(新,`default_permissions`);
  3. 参数级 `codex-args` 透传(带受保护校验,`unsafe` 时全放行)。
- **认证与网络**:`openai-api-key`、`responses-api-endpoint`(Azure 等自定义端点)、`codex-home`(信任边界:其中的 `config.toml`/命名档是**受信任输入**,不做消毒,README.md:152-153)。
- **Gate 控制**:`allow-users`、`allow-bots`、`allow-bot-users`(谁有权触发这个烧钱的 Action)。

## 6. 对 DSH 的可复用性(做「Codex 任务执行器」)

目标形态:DSH 工具 —— 输入(任务描述,工作区路径)→ 异步执行 Codex → 返回(diff, 总结)。**可以直接借用的步骤序列**:

1. **prompt 源解析**(runCodexExec.ts:150-158):任务描述既可作 inline,也可先落盘再作 file——DSH 里长任务建议写文件避免 shell 转义。
2. **装/找 CLI**:DSH 侧一次 `npm i -g @openai/codex`(action.yml:167),后续 `which codex` 定位(runCodexExec.ts:201)。
3. **认证注入**:复用代理模式最干净 —— 起 `codex-responses-api-proxy`,往临时 `CODEX_HOME/config.toml` 写 `model_provider` + `base_url`(writeProxyConfig.ts:23-35);或 DSH 简化版直接设 `OPENAI_API_KEY` env(丢失"key 不进 Codex 内存"的保护,但 DSH 本就是本机工具,可接受)。
4. **构造 argv**(runCodexExec.ts:256-292):`codex exec --skip-git-repo-check --cd <workspace> --output-last-message <tmpfile>` + `--model`/`--effort`/`--sandbox`。
5. **spawn + stdin 喂 prompt**(runCodexExec.ts:311-318):`stdio:["pipe","inherit","inherit"]`,异步任务在 DSH 后台执行即满足"异步执行"。
6. **结果接缝**(runCodexExec.ts:341-362):**不要解析 stdout** —— 等子进程退出后读 `--output-last-message` 写出的文件,这是拿"总结"的稳定通道;DSH 补一步 `git -C <workspace> diff` 拿 diff。
7. **参数校验**:照抄 `validateProtectedExtraArgs` + `RESTRICTED_CONFIG_ROOTS`(runCodexExec.ts:540-663)拦截用户透传参数里的提权配置(DSH 里尤其要防 `--yolo`、`-c mcp_servers.*`、`model_provider` 覆盖)。
8. **临时目录清理**(runCodexExec.ts:385-413):输出/ schema 临时目录用完即删。

**可直接移植的纯函数**:`checkOutput`、`ensureActorHasWriteAccess`(DSH 若做 GitHub 门禁)、`parseLinuxRunnerCredentials` 组。**平台注意**:Windows 无沙箱,DSH 在 Windows 上要么 `--sandbox` 走 Codex 自身,要么接受 `unsafe` 语义。

## 7. 对宿主集成最有价值的 5 个设计(file:line)

1. **认证与执行分离 + 本地代理(密钥不进 Codex 进程)** —— action.yml:222-243 + writeProxyConfig.ts:23-35。API key 只经 stdin 进代理进程(`env -u PROXY_API_KEY ... <<<`),Codex 只连 `http://127.0.0.1:<port>/v1`;代理可换 upstream(自定义端点)。任何"让 Agent 用外部 API"的宿主都可抄:密钥面最小化 + 端点可替换。
2. **文件通道传结果(`--output-last-message`)** —— runCodexExec.ts:262-263(组装)+ 341-362(读回/清理)。异步执行器拿"最终产物"的最稳接缝:进程退出码 + 一个文件,天然绕过 stdout 混入日志/ANSI 的问题;临时文件自动清理。
3. **受保护参数静态校验(执行前拒绝)** —— runCodexExec.ts:472-663,RESTRICTED_CONFIG_ROOTS:540-569。在 spawn 前用词法扫描 `extraArgs`,拒绝会改权限/信任/提供商/执行命令的旗标与 `-c` 覆盖;`unsafe` 才全放行。这是防 prompt-injection 提权的第一道闸,测试矩阵覆盖 40+ 变体。
4. **进程级不可逆降权(drop-sudo:两阶段 + setpriv 空 capability)** —— runCodexExec.ts:9-77(LINUX_DROP_SUDO_SCRIPT)+ dropSudo.ts:52-246。先 root 阶段清理 sudoers/组/root socket,再以 `--no-new-privs --bounding-set=-all --clear-groups` 启动,并在 spawn 前先做参数校验(校验失败不清权,dropSudo.test.mjs:459-481)。给"不可信输入驱动特权 Agent"的宿主提供纵深防御范本。
5. **可测试性内建(认证外置 + fake CLI 整程测试)** —— runCodexExec.ts:118-122(注释声明认证在函数外)+ runCodexExec.test.mjs:23-112。fake `codex` 放 PATH 捕获 argv,无需 key/网络验证命令构造;checkActorPermissions 通过注入 Octokit(参数化依赖)用本地 http server 测重试语义(checkActorPermissions.test.mjs:136-167)。所有副作用都留了注入缝。

---

## 对 DSH 插件的启示

1. DSH「Codex 任务执行器」不必重新发明:直接移植 `runCodexExec` 的 argv 组装 + stdin 喂 prompt + `--output-last-message` 文件读回,DSH 后台任务天然满足异步执行。
2. 结果交付用"临时文件 + 读回 + 清理"而非解析 stdout;DSH 侧在 Codex 完成后补 `git diff` 即可得到"diff + 总结"双产物。
3. 用户透传参数必须先过 `validateProtectedExtraArgs` 式校验(禁用 `--yolo`、`--full-auto`、provider/hook/MCP 配置覆盖),拒绝在 spawn 前完成。
4. API key 走"stdin → 代理进程"或至少独立 env 管理,不要把 key 暴露在 agent 会话里;`responses-api-endpoint` 保留,方便换 Azure/第三方网关。
5. Windows 宿主无沙箱,DSH 应把 sandbox/permission-profile 设成可选且默认保守(workspace 限定),并在文档里明确平台差异。
