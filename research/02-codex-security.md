# OpenAI Codex Security 深度研究报告

> 研究对象:`C:\Users\snow\codex-zone\codex-security`(299 文件,OpenAI 官方开源,Apache-2.0)

## 速览

- **定位**:`@openai/codex-security` v0.1.16 —— CLI + TypeScript SDK,是「Codex 安全插件」的薄封装,本质是**LLM 驱动的智能体安全审计**,不是规则扫描器(`sdk/typescript/package.json:2-4`、`README.md:3`)。
- **核心能力**:找漏洞(finding discovery)、验证(validation)、修复(patch)三步闭环,覆盖整仓扫描、diff/PR 扫描、96 小时深扫、批量扫描、SARIF/CSV/JSON 导出、Linear 发布、pre-commit 钩子。
- **检测引擎**:无 semgrep/gitleaks/codeql,引擎是 `@openai/codex` 智能体 + 捆绑的 `codex-security` 插件(14 个 skills + Python workbench 脚本);底层搜索靠 ripgrep/git grep(`core-scan.md:18-20`)。
- **SDK 形态**:`CodexSecurity` 类(`run/preflight/login/close`)+ `ScanResult`(manifest/findings/coverage 三件套),CLI 是同一 API 的 incur 参数壳(`cli.ts:42-56`、`bin/codex-security.mjs:13-15`)。
- **扫描流程**:威胁建模 → 独立基线审计 subagent → 并行调查员 subagent → 源码实证验证 → 攻击路径 → 诚实覆盖率,最终 seal 出 `scan-manifest.json/findings.json/coverage.json/report.md`。
- **运行方式**:本地 CLI/CI、SDK 嵌入、Docker 容器化批量扫描(compose.yaml 驱动 CSV 仓库清单,不可变 commit 定位)。
- **关键设计**:隔离 CODEX_HOME 运行时、`codex_security_scan` 沙箱文件系统 profile、成本追踪+预算熔断、SQLite workbench 持久化扫描历史、假阳性反馈回路。
- **DSH 可复用性:高** —— skills/提示词/JSON Schema 全部 Apache-2.0 且是纯文本资产,可直接改造成 DSH「代码安全审查」skill;但执行引擎依赖 `@openai/codex`,DSH 需用自己的 LLM+工具重实现 agent 编排。
- **版本矩阵**:npm 包 0.1.16,捆绑插件 0.1.22(`version.ts:9-12`),依赖 `@openai/codex` 与 `@openai/codex-sdk` 0.148.0-alpha.8(`package.json:61-62`)。
- **环境要求**:Node ≥22.13(22.x/24.x/26.x)+ Python ≥3.10(`package.json:15-17`、`sdk/typescript/README.md:18-23`);支持多推理提供商(OpenAI/OpenRouter/Fireworks/Bedrock)。

---

## 1. 这是什么

**一句话**:一个把 OpenAI 的 Codex 编程智能体改造成"安全审计员"的开源 CLI+SDK。

- 官方自述:`README.md:3` —— "a CLI and TypeScript SDK for finding, validating, and fixing security vulnerabilities in your code";`sdk/typescript/README.md:3-5` —— "ESM-only package includes TypeScript declarations, the `codex-security` executable, and the matching Codex runtime"。
- 架构自述(AGENTS.md):"Codex Security is a thin wrapper around Codex and its security plugin"(`AGENTS.md` 顶层指令),代码里对应 `api.ts:344-346` `createCodex: (options) => new Codex(options)` —— 所有扫描最终都是 `Codex.startThread().runStreamed(prompt)`(`api.ts:979-1002`)。
- 包与版本:`sdk/typescript/package.json:2-3`(`name: @openai/codex-security`,`version: 0.1.16`),`main: ./dist/index.js`,`bin: codex-security`(`package.json:19-30`);捆绑插件版本 0.1.22 硬编码于 `version.ts:12`。
- 形态:CLI(`npx @openai/codex-security scan .`)、TypeScript SDK(`new CodexSecurity(); security.run(".")`)、Docker 容器批量扫描三合一,README 定位为同一产品。
- 分发:私有 npm 包 + GitHub 仓库 + ghcr.io 容器镜像(`sdk/typescript/README.md:939`);本地登录走 Codex 凭据后端(文件 keyring / 系统 keyring,`sdk/typescript/README.md:152-165`)。
- 治理:公开 CLI 面视为 public API(`sdk/typescript/AGENTS.md` "Public CLI changes"),有严格的仓库安全策略(`SECURITY.md`)。

## 2. 能力清单(安全审计能做什么)

**扫描类**:

| 能力 | 证据 |
|---|---|
| 标准整仓/路径扫描 | `README.md:20` `scan .`;`security-scan/SKILL.md:3`(默认仓库扫描) |
| diff/PR/commit/分支扫描 | `README.md:216` `--diff origin/main`;`security-diff-scan/SKILL.md:3` "Review a pull request, commit, branch diff, or working-tree patch" |
| 工作区未提交改动扫描 | `sdk/typescript/README.md:279-287` `--working-tree`;`targets.ts:34,87-92` `DiffTarget.workingTree()` |
| Deep 深扫(多 worker 并行,最高 96h) | `README.md:26` `--mode deep --workers 2 --subagents 0 ...`;`deep-security-scan/SKILL.md:8` |
| 批量扫描多仓库(CSV/交互发现) | `README.md:196-199`;`sdk/typescript/README.md:224-229,556-587`(90 天内 GitHub 仓库发现) |
| 知识库注入(架构文档/威胁模型/PDF/docx/md) | `README.md:202-203` `--knowledge-base PATH`;`knowledge-base.ts` |

**审查流程能力**(由插件 skills 承载,证据见 `_bundled_plugin/skills/*/SKILL.md` front-matter):

- 威胁建模 `threat-model`:独立架构审查,生成资产/信任边界/攻击者能力模型
- 发现候选 `finding-discovery`:SQL/NoSQL 注入、XSS、越权/IDOR、路径穿越、命令注入、SSRF、反序列化、硬编码凭据、XXE、内存安全等 20+ 漏洞类别(`core-scan.md:56`)
- 验证 `validation`:对每个候选做源码实证、构造反证、判断 disposition
- 攻击路径 `attack-path-analysis`:source→sink 可达性 + 严重度校准
- 修复 `fix-finding` + `verify-fix`:打补丁并只读验证"是否真的修复"
- 漏洞报告 `vulnerability-writeup`、加固方案 `propose-security-hardening`
- 安全策略 `define-security-policy`:维护 SECURITY.md
- 外部接入 `triage-finding`(Jira/Linear/GitHub 工单 triage)、`track-findings`(Linear/Jira/GitHub issue/安全公告,批量 ≤25,带重复检查与读回)

**生命周期/集成类**(CLI 命令,`cli.ts:1530-3381`):

- `scan` 附 `--patch --patch-severity high --create-pr`:修复后自动提交并开 draft PR(`README.md:38-45`)
- `findings list / false-positive`:跨扫描的开放漏洞台账与假阳性反馈(`cli.ts:1530-1607`;假阳性示例回灌进后续扫描提示词,`api.ts:884-932`)
- `scans list/show/logs/rerun/match/compare`:扫描历史、日志(含 worker 会话)、根因匹配、新旧扫描对比(new/persisting/reopened/resolved,`README.md:106-109`)
- `export` SARIF/CSV/JSON(`cli.ts:2744`,`README.md:246-249`)
- `publish scan --to linear`、`patch --linear-issue`、`verify-fix --linear-project`(`README.md:101-104,111-119`)
- `install-hook` pre-commit 钩子,阻塞高危发现(`README.md:274-276`,`cli.ts:2477`)
- `validate`(对候选跑验证 skill)、`info --json`、`mcp add` 把 CLI 注册为只读元数据 MCP 服务(`sdk/typescript/README.md:820-829`)
- CI 退出码策略:0 通过 / 1 策略违规 / 2 不完整覆盖或错误 / 130/143 中断(`README.md:904-906`)
- **自定义验证** `--validation-prompt-file`:用自己的测试/脚本替换最终验证轮(`README.md:31-34`,demo 见 examples/)
- 成本上限 `--max-cost` 与超时熔断(`README.md:545-551`)

**不是**它做的:没有规则库、签名、semgrep/gitleaks/codeql 等第三方检测器;不做运行时动态分析(除非自定义验证脚本);审计依赖 LLM 判断,靠"覆盖率诚实上报"兜底。

## 3. SDK 形态

**导出面**(`src/index.ts` 全部 export,95 行):类 `CodexSecurity`、工厂 `createSecurity`、`estimateScanCost`、`publishScan`、`loadContract/requireScanFile`、运行时工具集(`bootstrapPlugin/createIsolatedHome/extractPluginZip/resolvePluginPath` 等)、目标抽象(`DiffTarget/normalizeTarget`)、14 个错误类、`ScanResult`。

**核心类 `CodexSecurity`**(`api.ts:359-394`),公开方法(函数签名级):

| 方法 | 签名 | 行 |
|---|---|---|
| `run` | `run(repository: string, options?: ScanOptions): Promise<ScanResult>` | `api.ts:396-401` |
| `preflight` | `preflight(repository, options?): Promise<ScanPreflight>`(不启动运行时/不加载凭据) | `api.ts:403-414` |
| `loginApiKey` | `loginApiKey(apiKey: string): Promise<void>` | `api.ts:1458` |
| `loginChatGPT` / `loginChatGPTDeviceCode` | `(): Promise<CodexLoginHandle>` | `api.ts:1478-1484` |
| `account` / `logout` / `close` | — | `api.ts:1502/1521/1542` |

**`ScanOptions`**(`api.ts:205-237`):`auth`(auto/chatgpt/api-key)、`target`(`"repository" | DiffTarget | string[]`,`targets.ts:94`)、`mode`(`"standard"|"deep"`,`targets.ts:33`)、`knowledgeBasePaths`、`scanPrompt/validationPrompt/postScanPrompt`、`outputDir/archiveExisting`、`maxCostUsd`、`maxTimeHours`、`failureSeverity`、`parentScanId`、`signal`;13 个回调 `onAuthentication/onCost/onProgress/onWorkerStatus/onSessionEvent/onReconnect/onWarning/...`(`api.ts:219-237`)。

**`ScanResult`**(`result.ts:47-128`):`manifest`(`ScanManifest`)、`findings`(`FindingsDocument`)、`coverage`、`scanDir`、`threadId`、`cost`、`sarifPath`、`repositoryFindings`(跨扫描台账),getter `reportPath/findingsPath/manifestPath/coveragePath/artifactsDir`,`toJSON()`。

**CLI 入口与共用逻辑**:`bin/codex-security.mjs:13-15` → 加载 `dist/cli.js` 的 `main()`;`cli.ts:42-56` 用 `incur` 声明命令并直接 import 同一个 `CodexSecurity` 类 —— **CLI 与 SDK 共用同一核心类**,CLI 只是参数解析 + 进度渲染壳(TUI 用 `ink/react`,`patch-tui.tsx`、`scan-dashboard.ts`)。

**"安全智能"住在哪**:`_bundled_plugin/`(npm 包 `files` 字段包含它,`package.json:31-37`)。SDK 运行时会把它 bootstrap 进一个**隔离的 CODEX_HOME**(`runtime.ts:2136-2239` `bootstrapPlugin`:`plugin marketplace add` → `plugin add` 安装),然后以 `startThread({workingDirectory: scanDir, skipGitRepoCheck: true, approvalPolicy})` 启动 Codex 线程(`api.ts:979-983`),提示词包含 skill 名 + 扫描上下文(`api.ts:872-882`)。Python 侧是"workbench"记账服务(`workbench_cli.py`、`deep_scan_workbench.py`:argparse 子命令 + sqlite3,如 `register-cli-scan/complete-scan/fail-scan`,见 `api.ts:778-796,1135-1142`)。

## 4. 工作模式(docker/ + compose.yaml + examples/)

**容器 = 非交互、可恢复的批量扫描器**,不是常驻服务、不是定时任务:

- `compose.yaml:36-40`:`command: ["bulk-scan", "/input/repositories.csv", "--output-dir", "/output"]` —— 每次 `docker compose run` 跑一轮;输入 `repositories.csv`(id,repository,revision 全 hash)只读挂载,`/output` 结果、`/state` 凭据可恢复(`sdk/typescript/README.md:924-948`)。
- 安全硬化:Dockerfile 里非 root 用户 10001、`cap_drop: ALL`、`no-new-privileges`、seccomp 配置(`docker/codex-security-seccomp.json`,`compose.yaml:8-12`)、可选 AppArmor profile(`compose.apparmor.yaml`、`docker/codex-security.apparmor`),CI 里对 `ghcr.io/openai/codex-security` 镜像做版本校验(`docker/verify-container-release-version.sh`)。
- `docker/entrypoint.sh`:参数白名单重写;检测受限 Ubuntu userns 环境并自动补 `--codex features.use_legacy_landlock=true`(`entrypoint.sh:57-111`);有 GH_TOKEN 时注入 git credential helper(`entrypoint.sh:113-140`,`docker/git-credential.sh`),支持私有仓库。
- 镜像内嵌完整 npm 包 + Codex 运行时 + Python 3(`Dockerfile:25-40`)。

**examples/ = 用法演示**(两个):

1. `examples/custom-validation/`:一个**故意带 IDOR 漏洞的本地发票 API**(`app.py:34-35` "BUG: authentication does not establish ownership")。`run.mjs` 把 app.py+validate.py 拷到临时目录,`codex-security scan <target> --path app.py --scan-prompt-file scan.md --validation-prompt-file validation.md --headless`(`run.mjs:16-36`);`validation.md` 指示 agent 跑 `validate.py` 起真实 loopback 服务、验证匿名 401/本人 200/跨账号 200,产出 `artifacts/custom-validation/{candidates,http-proof,results}.json`(`README.md:20-33`)。**证明了"静态扫描 + 用户自定义动态验证"的组合用法**。
2. `_bundled_plugin/examples/completed-scan/`:一次真实扫描的四件套样例(coverage.json / findings.json / report.md / scan-manifest.json),即产物形状参考。

**其余运行模式**:本地交互 CLI(全屏 TUI + finding browser)、CI 无头模式(`--headless --json --fail-on-severity`)、SDK 嵌入、pre-commit 钩子。

## 5. 依赖

**npm 运行时依赖**(`package.json:57-74`,全部锁定版本):

- 引擎:`@openai/codex` 0.148.0-alpha.8(Codex 可执行运行时)、`@openai/codex-sdk` 0.148.0-alpha.8(agent 线程 API)
- 集成:`@linear/sdk` 89.0.0(Linear 发布/工单)、`@octokit/core` 7.0.6(GitHub PR 创建)
- CLI/TUI:`incur` 0.4.13(声明式 CLI + `--llms` agent 清单 + MCP 注册)、`ink` 6.8.0 + `react` 19.2.4(TUI)
- 数据/格式:`ajv`(JSON Schema 校验)、`papaparse`(CSV)、`smol-toml`(Codex config.toml)、`pdfjs-dist`(PDF 知识库文本抽取)、`extract-zip`/`fflate`(插件 zip)、`fast-uri`、`semver`、`@inquirer/prompts`

**Python 侧**:标准库为主(argparse/json/sqlite3/dataclasses,`workbench_cli.py:1-24`、`deep_scan_workbench.py:1-29`),仅要求 Python ≥3.10(3.10 需 `tomli`)。**没有任何第三方安全检测器** —— 这是与 Semgrep/GitLab SAST/CodeQL 类工具的**根本差异**。

**底层"检测"手段**:LLM 代码审计 + `resolve_security_md.py` 继承仓库 SECURITY.md 策略(`core-scan.md:22-24`)+ 离线搜索命令(优先 ripgrep,拒绝下载型 wrapper,`core-scan.md:18-20`)+ 目录/工作树内容指纹(`workbench_target.py` 的 `directory_content_digest/git_revision`)。

## 6. 对 DSH 的可复用性

**可直接抽取(低成本,纯资产,Apache-2.0)**:

1. **审计方法论提示词** —— `core-scan.md`(基线审计员 prompt + 4 种调查员视角 + 严重度校准规则 + CWE 映射)和 `threat-model.md` 是**完整可移植的 agent 安全审计手册**,DSH 可做成 skill 直接绑定现有 read/grep/glob 工具。
2. **JSON Schema 数据契约** —— `schemas/findings.schema.json`、`coverage.schema.json`、`scan-manifest.schema.json`(及生成的 `models.ts`):finding 含 ruleId/severity/confidence/taxonomy.cwe/locations/codeEvidence/rootCause/attackPath/remediation/validation 的完整模型,直接可作为 DSH 审查工具的输出格式。
3. **14 个 skill 的流程设计** —— threat-model→finding-discovery→validation→attack-path-analysis→fix/verify 的分相编排,以及 `finding-detail-fields.md`、`final-report.md` 等参考文档。
4. **假阳性反馈回路**(`api.ts:884-932`):把历史 false-positive 及其原因作为"审查者反馈"注入下一轮,DSH 可照搬实现误报学习。
5. **成本追踪模型**(`cost.ts`/`cost-model.ts`,`api.ts:696-748`):token→美元估算 + `maxCostUsd` 熔断,对 DSH 长任务同样适用。

**不能直接搬(需重实现)**:

- **执行引擎**:`@openai/codex-sdk` 是闭源依赖,扫描本质是"驱动 Codex agent"(`api.ts:979-1002`)。DSH 复用时只能二选一:(a) 若本机装了 codex CLI,走 `CODEX_CLI_PATH` 子进程桥接(DSH 已有 pwsh/持久 bash 能力);(b) 用 DSH 自己的 LLM + 工具重新实现 core-scan 工作流 —— **可行**,因为提示词全部开源,且 DSH 的 subagent 模型天然支持"基线审计员 + 并行调查员"扇出。
- Python workbench(扫描注册/历史 SQLite):DSH 若要同样的扫描历史/对比,需用插件侧 SQLite 或复用 workbench 脚本(纯 Python 标准库,可整包拷贝)。

**接入成本评估**:做一个 DSH「代码安全审查」工具,中等成本(约 1 个工作日级):拷贝 2 个核心参考文档 + 4 个 JSON Schema + 编写 skill 主文件(约 300 行),复用 DSH 既有文件工具做 scope 枚举,subagent 做并行调查,输出 findings.json 形状的报告。若再要扫描历史/对比/假阳性反馈,加一个 SQLite 插件服务(workbench 思路)。**不建议**直接依赖 npm 包本身(它需要 OpenAI/Codex 生态账号与 TAC 授权,`README.md:7-9`)。

## 7. 对宿主集成最有价值的 5 个设计(file:line)

1. **威胁模型 + 独立基线审计员 + 聚焦调查员的多智能体扇出**(`_bundled_plugin/references/core-scan.md:5-12`):一个无偏基线 subagent 全仓独立审计,同时 parent 建威胁模型、分组调查包、并行派调查员(前向/后向/授权逻辑/开放四视角,`core-scan.md:30-37`),最后以"并集 fully_reviewed_files ∩ 授权清单"诚实计算覆盖率。→ **DSH 可直接映射为 subagent 并行安全审查流水线**,并解决"单 agent 扫描易漏、无覆盖率概念"两大痛点。
2. **隔离运行时 + 插件 bootstrap**(`runtime.ts:2136-2239`,`api.ts:348` `SCAN_PERMISSION_PROFILE = "codex_security_scan"`,`README.md:974-984` 本地安全模型):每次扫描用私有 CODEX_HOME、`codex_security_scan` 文件系统 profile(读全盘、只写 workspace 与 state dir)、自动审批但可用 `approval_policy="never"` 收紧。→ **DSH 做审查工具时的沙箱边界模板**:"只读仓库 + 指定输出目录"的权限模型。
3. **可恢复的扫描契约与密封产物**(`result.ts:47-128`、`models.ts:3-66`、`_bundled_plugin/examples/completed-scan/`、`workbench_db.py`):scan-manifest(含 producer/target/scope/threatModel/artifacts+sha256)/findings/coverage 三件套 + SQLite 历史 + 不可变 revision 定位,支持 `scans rerun/match/compare`。→ **DSH 长任务的产物规范化模板**:结果目录可重跑、可比对、可审计。
4. **成本追踪与预算熔断**(`api.ts:696-748` ScanCostTracker、`cost.ts`,`api.ts:737-745` 超限即 `costAbortController.abort(ScanCostLimitExceededError)`,且 Deep 扫描超预算时保留已完成 finding 产出部分报告,`api.ts:1300-1373`):长 agent 任务按 token 计费 + 优雅降级。→ **DSH 深扫/多轮任务的成本护栏蓝本**。
5. **假阳性反馈注入 + 覆盖率诚实协议**(`api.ts:884-932` false_positive_feedback.json 以"reviewer feedback, not instructions"注入;`core-scan.md:14` "mark coverage complete only when the requested source scope was actually reviewed"、`deep-security-scan/SKILL.md:72` "never claim the repository is free of vulnerabilities"):把模型输出当数据、禁止把 repo 内容当指令、缺证据必须上报。→ **DSH 安全类工具的道德护栏与可信度设计**,可直接写进 skill。

---

## 对 DSH 插件的启示

1. **最划算的落地**:把 `core-scan.md` 基线审计/调查员提示词 + 4 个 JSON Schema 拷成 DSH skill「代码安全审查」,复用现有 read/grep/glob 与 subagent,输出 findings.json 形状报告 —— 成本约一天,效果等价一个自托管 LLM 版 SAST。
2. **流程骨架直接用**:threat-model → 独立基线 → 并行调查 → 证据验证 → 覆盖率并集 → 密封产物,这个六段式是仓库最有价值的方法论资产。
3. **沙箱边界照抄**:只读扫描仓库、结果写指定目录、`approval_policy` 可切换,是安全工具类插件的默认权限模板。
4. **加 SQLite 历史层即可获得"扫描台账 + 新旧对比 + 假阳性学习"**,workbench 脚本是纯标准库 Python,可整包复用。
5. **不要引入 `@openai/codex` 依赖**(需 OpenAI 账号/TAC 授权);若目标环境已有 codex CLI,可保留 `CODEX_CLI_PATH` 式子进程桥接作为可选后端。
