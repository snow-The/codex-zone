# Codex 专区 (codex-zone)

OpenAI 开源 Codex 系列的全方位研究专区,目标是把 Codex 做成 DSH(DeepSeek Harness)生态插件。

## 仓库镜像(clone 于 2026,shallow)

| 目录 | 来源 | 说明 |
|---|---|---|
| `codex` | https://github.com/openai/codex | 主 monorepo:Rust 核心 + TS CLI + SDK,6492 文件 |
| `codex-security` | https://github.com/openai/codex-security | 安全审计 CLI/SDK(Docker 容器化) |
| `codex-action` | https://github.com/openai/codex-action | GitHub Action 封装(PR 审查/任务执行) |
| `codex-plugin-cc` | https://github.com/openai/codex-plugin-cc | Claude Code 插件(对 DSH 插件化最有借鉴意义) |
| `hono` | https://github.com/honojs/hono | Hono Web 框架 4.13.3(自研插件 TS 重构用) |

## 研究报告(`research/`)

- `00-summary.md` — 全系列总览
- `01-codex-core.md` — Codex 主仓库架构(agent loop、工具系统、模型接口、沙箱、可嵌入性)
- `02-codex-security.md` — Codex Security 能力清单与可复用性
- `03-codex-action.md` — GitHub Action 封装逻辑与可复用步骤
- `04-codex-plugin-cc.md` — Claude Code 插件结构 → DSH 插件映射
- `05-hono.md` — Hono 在 DSH 插件中的用法与迁移方案
- `06-skill-dedup.md` — dsh-skill-pack 技能去重审查
- `07-own-plugins-review.md` — 自研插件按最新 DSH 版本检视
- `08-dsh-ecosystem-survey.md` — **DSH 生态调研**(按 star 排序 + awesome 清单;主题与 Codex 系列不同,是"自研插件该吸收什么"的横向扫描)
- `09-busyloop-family.md` — busyloop 家族(架构、发布状态、10-commit 门禁)

> **关于目录布局**:`research/` 是本系列唯一的家,00-09 连续。第 8、9 篇曾被提交到仓库**根目录**,
> 而 00-07 在 `research/` 里,同一个文件还在 `~/.dsh-starter/codex-zone/` 留了一份无版本控制的副本
> —— 三处并存。现已全部归入 `research/`,根目录只剩 `.gitignore` 与 `README.md`。

## 产物

- `dsh-codex` 插件(仓库:`snow-The/dsh-busyloop-codex`,目录名 `dsh-codex.frozen`)— 让 DSH 能调用 Codex CLI/SDK 执行任务
  > 已冻结:`dsh-busyloop` 才是现在在用的 agent-loop 引擎(能力,不是 codex 的替代品);这个插件保留为 Codex 特定的历史产物。

## 本机环境事实

> 这些是**调研当时**的快照(2026-08)。宿主已从 rc.8 走到 0.2.0-rc.2 —— 下面几条里只有前两条仍按原样成立。

- 无 `codex` 命令、无 `~/.codex`、`OPENAI_API_KEY` 未设置 —— 插件需做环境检测与安装引导
- Node v26.7.0 / npm 11.19.0(引擎警告:npx codex 下载时 EBADENGINE)
- ~~DSH 宿主 rc.8(web profile)~~ → 现为 **0.2.0-rc.2**
