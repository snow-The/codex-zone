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

- `01-codex-core.md` — Codex 主仓库架构(agent loop、工具系统、模型接口、沙箱、可嵌入性)
- `02-codex-security.md` — Codex Security 能力清单与可复用性
- `03-codex-action.md` — GitHub Action 封装逻辑与可复用步骤
- `04-codex-plugin-cc.md` — Claude Code 插件结构 → DSH 插件映射
- `05-hono.md` — Hono 在 DSH 插件中的用法与迁移方案
- `06-skill-dedup.md` — dsh-skill-pack 技能去重审查
- `07-own-plugins-review.md` — 自研插件按最新 DSH 版本检视

## 产物

- `dsh-codex` 插件(仓库:snow-The/dsh-codex)— 让 DSH 能调用 Codex CLI/SDK 执行任务

## 本机环境事实

- 无 `codex` 命令、无 `~/.codex`、`OPENAI_API_KEY` 未设置 —— 插件需做环境检测与安装引导
- Node v26.7.0 / npm 11.19.0(引擎警告:npx codex 下载时 EBADENGINE)
- DSH 宿主 rc.8(web profile)
