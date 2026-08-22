# dsh-skill-pack 技能去重审查报告

审查对象:`C:\Users\snow\.dsh-starter\plugins\dsh-skill-pack\skills`(39 个技能目录)
对比源:
- **aegis 插件**(`C:\Users\snow\.dsh\profiles\web\node_modules\aegis\skills`,22 个)
- **宿主/其他插件内置技能**(catalog 中已存在,无 SKILL.md 文件,内嵌于宿主 bundle)

结论速览:**建议从 skill-pack 删除 10 个技能(3 个是纯包装壳,7 个与其他源或包内技能重复),39 − 10 = 29 个保留**。

---

## 1. 去重矩阵表

| # | 重复组 | skill-pack | aegis | 宿主内置 | 建议保留 | 建议删除/合并 | 理由 |
|---|---|---|---|---|---|---|---|
| 1 | **grilling 访谈** | `grill-me`、`grill-with-docs`、`batch-grill-me` | `brainstorming`(含 grilling 模式) | `grilling` | 宿主 `grilling`(原语);aegis `brainstorming`(设计优先流程) | 删 skill-pack 全部 3 个 | `grill-me`/`grill-with-docs` 正文仅一行 "Run a `/grilling` session",是宿主 `grilling` 的纯包装;`batch-grill-me` 有真实方法内容(frontier 设计树)但职责与 `grilling`/`brainstorming` 完全重叠,可把"批量问 frontier"技巧并进宿主技能 |
| 2 | **调试** | `debugging-and-error-recovery` | `systematic-debugging` | `diagnosing-bugs`、`diagnose` | 宿主 `diagnosing-bugs`(或 aegis `systematic-debugging`) | 删 skill-pack 的 | 四个技能同一职责(系统化根因调试)。宿主已有两个,描述完全相同("Reproduce → minimise → hypothesise → instrument → fix → regression-test");skill-pack 版无差异化方法 |
| 3 | **TDD** | `test-driven-development` | `test-driven-development`(strict 路由门控) | `tdd` | **skill-pack 的**(最全,398 行,模型调用) | aegis 版仅 strict 路由时可用(其自身描述即"use when...explicit `TDD Route: strict`");宿主 `tdd` 为薄提示 | 三方同名/同职。skill-pack 版是唯一通用、模型调用的完整 TDD 循环 |
| 4 | **领域语言/术语表** | `ubiquitous-language` | `establishing-project-context`(CONTEXT.md 治理) | `domain-modeling` | 宿主 `domain-modeling` | 删 skill-pack 的 | `domain-modeling` 描述即涵盖 "pin down domain terminology or a ubiquitous language";且 ask-matt 主流程已通过 `/domain-modeling` 接线,`ubiquitous-language` 另立 UBIQUITOUS_LANGUAGE.md 约定,形成第三套输出 |
| 5 | **写技能** | `writing-great-skills` | `writing-skills`(670 行,压力场景验证) | `write-a-skill` | 宿主 `write-a-skill` + aegis `writing-skills` | 删 skill-pack 的 | 三方重叠。skill-pack 版自我定位只是 "Reference for writing and editing skills well"(词汇/原则参考),另外两方都是完整创作+验证流程 |
| 6 | **交接** | `handoff`、`claude-handoff` | — | — | `handoff`(通用) | 删 `claude-handoff` | 包内重复:`claude-handoff` 正文 = `handoff` 正文 + 一行 `claude --bg --name ...` 启动命令(Claude Code 专用)。通用版更可移植,启动机制在本环境(DSH)无 `claude` CLI 场景 |
| 7 | **切片成票** | `to-issues`、`to-tickets` | — | `request-refactor-plan`(重构专项) | `to-tickets` | 删 `to-issues` | 包内重复:`to-tickets` 是 `to-issues` 的超集(同 vertical-slice 规则 + blocking edges + 本地 markdown tracker + 宽重构 expand–contract 例外)。`to-tickets` 也是 ask-matt 主流程接线的那一个 |
| 8 | **对话转文档** | `to-prd`、`to-spec` | — | — | `to-spec` | 删 `to-prd` | 包内重复:`to-spec` 正文第 6 行自述 "produces a spec (you may know this document as a PRD)",两者模板/流程逐字相同;ask-matt 接线的是 `/to-spec` |
| 9 | **写作 exploit 变体** | `writing-beats`、`writing-shape` | — | — | `writing-shape` | 合并 `writing-beats` | 包内重复:两者同为 "Writing, exploit"(输入都是 markdown 原材料堆),共用同一套 Grounding 机制(概念需先 grounding 才能被后文依赖),仅输出单元不同(beat 旅程 vs 逐段成型)。建议并为一个技能,保留两种节奏 |
| 10 | **代码审查** | `code-review-and-quality`(5 轴) | `requesting-code-review`/`receiving-code-review`(收发协议) | `code-review`/`review`(Standards+Spec 双轴,并行子代理) | 宿主 `code-review` 作为流程节点(implement 收尾调它);skill-pack 版保留为深度参考 | 无删除(轻度重叠,标记) | 职责交叉但视角不同:宿主是"diff 双轴评审",skill-pack 是"5 轴质量门",aegis 是"评审收发协议"。建议保留 skill-pack 版但明确其定位为深度参考,避免与宿主双轴版同时模型调用 |
| 11 | **ADR/决策记录** | `documentation-and-adrs` | `recording-architecture-decisions`(绑 Aegis docs 基建) | `domain-modeling` 附带 ADR 记录 | skill-pack 版(通用,含"match existing convention first") | 无删除(标记;Aegis 治理项目可只用 aegis 版) | 重叠真实但 aegis 版依赖其 Required Read Set(docs/adr/ADR-CREATION-GATE.md 等),仅 Aegis 流程内自洽;通用项目保留 skill-pack 版 |
| 12 | **退役/删除旧码** | `deprecation-and-migration` | `anti-entropy-governance`(窄域治理) | — | 两者都留(互补) | 无删除(标记) | aegis 版自述 "a narrow governance owner for retirement, fallback collapse, duplicate-owner cleanup, and deletion safety";skill-pack 版是通用决策框架(是否退役 5 问 + compulsory/advisory)。重叠在"退役"一个子域 |
| 13 | **按计划执行** | `implement`(薄,14 行) | `executing-plans`、`subagent-driven-development` | — | 都留(标记) | 无删除 | implement 是 ask-matt 流程节点(调 /tdd + /code-review),aegis 版是计划执行协议+子代理编排,深度不同、定位不同 |
| 14 | **对抗式复核** | `doubt-driven-development` | `first-principles-review` | — | 都留(标记) | 无删除 | 精神相似(站对立面复核)但触发与方法不同(fresh-context 复审核 vs 第一性原理/Occam 决策复核) |
| 15 | **环境级冗余(不涉 skill-pack)** | — | `communicating-concisely` | `caveman` | 留宿主 `caveman` | — | 提示:宿主已有 `caveman`(同为极简输出模式),aegis 的 `communicating-concisely` 与它重复,与本次 skill-pack 无关但值得后续处理 |

---

## 2. skill-pack 内部冗余清单(建议删除/合并,共 10)

| 删除/合并 | 技能 | 类型 |
|---|---|---|
| 删除 | `grill-me` | 宿主 `grilling` 的一行包装壳 |
| 删除 | `grill-with-docs` | 宿主 `grilling` + `/domain-modeling` 的一行包装壳 |
| 删除 | `batch-grill-me` | 与宿主 `grilling` / aegis `brainstorming` 职责重复(frontier 树技巧可并入) |
| 删除 | `claude-handoff` | `handoff` 的 Claude 专属超集,包内重复 |
| 删除 | `to-issues` | `to-tickets` 的旧版子集 |
| 删除 | `to-prd` | `to-spec` 的同模板副本 |
| 删除 | `ubiquitous-language` | 宿主 `domain-modeling` 已覆盖 |
| 删除 | `writing-great-skills` | 宿主 `write-a-skill` + aegis `writing-skills` 已覆盖 |
| 删除 | `debugging-and-error-recovery` | 宿主 `diagnosing-bugs`/`diagnose` + aegis `systematic-debugging` 已覆盖 |
| 合并 | `writing-beats` | 并入 `writing-shape`(同为 exploit,共用 Grounding 机制) |

## 3. 最终建议保留列表(39 − 10 = **29**)

1. `api-and-interface-design` — 宿主无等价物(Hyrum's Law / One-Version / contract-first)
2. `ask-matt` — 路由技能,保留但需更新(删除的 grill 引用要改指向宿主 `/grilling`)
3. `code-review-and-quality` — 5 轴质量门,与宿主双轴版定位区分后保留
4. `code-simplification` — 无重复源
5. `deprecation-and-migration` — 通用退役决策框架(与 aegis anti-entropy 互补)
6. `design-md` — 自有格式(google-labs DESIGN.md),无重复
7. `documentation-and-adrs` — 通用 ADR/文档技能
8. `doubt-driven-development` — fresh-context 对抗复核,无等价物
9. `edit-article` — 文章编辑(改既有草稿),与 writing-* 职责不同
10. `handoff` — 通用交接文档
11. `implement` — ask-matt 流程节点
12. `improve-codebase-architecture` — 深化机会扫描+HTML 报告,无重复
13. `incremental-implementation` — 薄垂直切片执行纪律
14. `loop-me` — 有状态 workflow 规格访谈(产出 workflows/*.md),定位独特
15. `performance-optimization` — 无重复源
16. `security-and-hardening` — 无重复源
17. `setup-ts-deep-modules` — 检查过:**不过时**。检测包管理器、合并已有 `.dependency-cruiser.*` 配置,不绑定单一工具链版本;工具链特定是其设计(user-invoked),保留
18. `source-driven-development` — 官方文档背书实现,与宿主 `research` 交付物不同
19. `teach` — 教学工作区,无重复
20. `test-driven-development` — 三方 TDD 中保留此通用版
21. `to-questionnaire` — 独特(把未知决策转成给别人的问卷)
22. `to-spec` — 与 to-prd 合并后的保留方
23. `to-tickets` — 与 to-issues 合并后的保留方
24. `triage` — 问题/PR 状态机,与宿主 `qa` 不同(会话式报 bug vs 状态机治理)
25. `wayfinder` — 决策票地图,无等价物
26. `wizard` — 交互式 bash 向导,无重复
27. `writing-fragments` — 写作 explore 阶段,与 exploit 技能互补
28. `writing-shape` — 写作 exploit 保留方(吸收 writing-beats)
29. `zoom-out` — 轻量上下文拉升工具,无重复

## 4. 证据引用(SKILL.md 原文)

- **grill-me**:`description: A relentless interview to sharpen a plan or design.`;正文仅 6 行,全部内容 = "Run a `/grilling` session."
- **grill-with-docs**:正文仅 6 行 = "Run a `/grilling` session, using the `/domain-modeling` skill."
- **宿主 grilling**:catalog 描述 "Grill the user relentlessly about a plan, decision, or idea…uses any 'grill' trigger phrases." — 即上两个包装的目标原语。
- **batch-grill-me**:`A relentless interview that asks every frontier question at once, round by round.`(有方法:design tree + frontier 轮询,但职责同 grilling)。
- **aegis brainstorming**:`…or when grilling/pressure-testing a plan or design` + "Enter `Grilling Mode`" 分支 — aegis 侧已承担 grilling 职责。
- **claude-handoff vs handoff**:两者共享逐字相同的清单("Write a handoff…","Include a 'suggested skills' section","Do not duplicate content already captured in other artifacts…","Redact any sensitive information…"),claude-handoff 额外只有一行 `claude --bg --name "<descriptive name>" "<handoff summary>"` 启动命令。
- **to-issues vs to-tickets**:to-tickets 描述含 `each declaring its blocking edges, published to the configured tracker — edges as text in one file per ticket locally, or native blocking links on a real tracker`(超集能力),并额外有"宽重构 expand–contract"章节;两者共用 `<vertical-slice-rules>` 块与 "run `/setup-matt-pocock-skills` if not"。
- **to-prd vs to-spec**:to-spec 第 6 行自述 "produces a spec (you may know this document as a PRD)";两者模板逐字相同(Problem Statement / Solution / User Stories…),流程相同("Do NOT interview the user — just synthesize")。
- **ubiquitous-language**:`Extract a DDD-style ubiquitous language glossary…Saves to UBIQUITOUS_LANGUAGE.md`;宿主 domain-modeling 描述: `pin down domain terminology or a ubiquitous language, record an architectural decision…`;aegis establishing-project-context: `establish shared project language…needs active semantic modeling`(CONTEXT.md 写策略主)。
- **writing-great-skills**:`Reference for writing and editing skills well — the vocabulary and principles that make a skill predictable`(82 行,纯参考);宿主 write-a-skill: `Create new agent skills with proper structure, progressive disclosure, and bundled resources`;aegis writing-skills: `creating new skills, editing existing skills, or verifying skills work before deployment`(670 行,压力场景验证)。
- **writing-beats vs writing-shape**:两者 description 同前缀 `Writing, exploit —`;均要求"user has passed (or will pass) a markdown file of raw material";均有同名 "## Grounding" 章节(概念须 grounding 后才能被后续段落/beat 依赖,Prerequisite/Introduced 两种来源)。
- **debugging-and-error-recovery**:`Guides systematic root-cause debugging…`;宿主 diagnosing-bugs 与 diagnose 描述均为 `Reproduce → minimise → hypothesise → instrument → fix → regression-test`;aegis systematic-debugging: `Use when encountering a bug, test failure, or unexpected behavior, before proposing fixes`。
- **test-driven-development 三方**:skill-pack 版 `Drives development with tests. Use when implementing any logic, fixing any bug…`(398 行完整循环);aegis 版 `Use when the user explicitly requests strict or test-first TDD, or when the current conversation already contains an explicit TDD Route: strict decision`(默认 off,严格路由门控);宿主 tdd: `Test-driven development…red-green-refactor…`(薄)。
- **documentation-and-adrs**:`Records decisions and documentation…ADRs capture the reasoning behind significant technical decisions`;aegis recording-architecture-decisions: `Use when the user asks to create, write, update, amend, supersede, or evaluate an ADR…`(Required Read Set 绑定 Aegis docs 目录)。
- **deprecation-and-migration vs anti-entropy-governance**:skill-pack `Manages deprecation and migration. Use when removing old systems, APIs, or features…`;aegis `Use when touching retiring old logic, collapsing duplicate owners, removing fallbacks…` + 自述 "It is a narrow governance owner for retirement, fallback collapse, duplicate-owner cleanup, and deletion safety"。
- **setup-ts-deep-modules(过时性检查)**:检测包管理器(`pnpm-lock.yaml`→pnpm…)、"If one exists, do **not** overwrite it: merge the four rules…" — 依赖 dependency-cruiser(持续维护),不绑定过时工具链,结论:保留。

---

## 动手步骤(在 dsh-skill-pack 仓库执行)

1. `cd C:\Users\snow\.dsh-starter\plugins\dsh-skill-pack && git rm -r skills/grill-me skills/grill-with-docs skills/batch-grill-me skills/claude-handoff skills/to-issues skills/to-prd skills/ubiquitous-language skills/writing-great-skills skills/writing-beats skills/debugging-and-error-recovery`(skills 目录为纯 SKILL.md 文本,src/index.ts 不生成技能,删目录即生效)
2. 更新 `README.md`:"## Skills (27)" 改为 "## Skills (29)",从技能表删除上述 10 行;并编辑 `skills/ask-matt/SKILL.md`(第 16 行等),把 `/grill-with-docs`、`/grill-me` 引用改指向宿主内置 `/grilling`,`/domain-modeling`、`/tdd`、`/code-review` 本就是宿主技能无需保留在包内
3. `git commit -m "chore: dedup skill-pack vs host & aegis skills (-10)"`,然后 `dsh plugin` 重装/重启生效(改动前可用 undo 快照兜底;建议同时评估 aegis 侧 `test-driven-development`/`communicating-concisely` 与宿主的重复)
