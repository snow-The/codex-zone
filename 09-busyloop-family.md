# 09 — dsh-busyloop 家族(架构与发布状态)

> 引擎/范式分离设计:能力等价,范式不同。全部活在 dsh 宿主内,非独立应用。

## 架构

```
dsh-busyloop ──────────── 引擎(能力:LLM 适配器 + loop + 宿主工具调度)
├── dsh-busyloop-codex ──── 范式①:桥接 codex CLI(外包)
└── dsh-busyloop-codexstyle ─ 范式②:自研 codex 风格(规划中)
```

## 关键决策

- **LLM 通道 = 官方 `ctx.llm`**(@deepseek-ai/dsh-llm 的 LlmRuntime):非手刻、非外接 SDK;类型来自官方包,运行时宿主提供
- **工具表注入**:宿主 ctx.tools 只读不可枚举,loop 工具由调用方提供(schema+execute)
- **发布物自包含**:hono 打进 dist,官方包 external 由宿主提供,"纯 dsh+官方包"可跑(真机 E2E 验证)
- **测试金字塔**:fake chunks 单测(14) → 宿主解析链验证 → 真机 E2E(e2e-host.mjs)

## 发布状态(2026-08-22)

| 包 | 版本 | 说明 |
|---|---|---|
| @snow-the/dsh-busyloop | 0.1.2 | 引擎:health/providers 端点、sessionId 透传、14 测试 |
| @snow-the/dsh-busyloop-codex | 0.1.1 | CLI 桥(原 dsh-codex 改名,旧名保留不更新) |
| @snow-the/dsh-codex | 0.1.0 | 已冻结(改名引导到新名) |

## 上市场门禁:10 次有意义 commit(已达成)

1. busyloop-codex 改名发布(de129c9)
2. 08-dsh-ecosystem-survey 调研报告(b75f59f)
3. providers 端点 + llm-aware apply(28980df)
4. bump 0.1.1(2edcd2a)
5. README:架构/原则/用法/测试金字塔(08d1c79)
6. sessionId 透传(4f27b5e)
7. bump 0.1.2(8a92803)
8. busyloop-codex README 家族改名(e75d0be)
9. bump codex 0.1.1(92e1736)
10. e2e-host.mjs 真机冒烟入库(8f62805)

## 下一步(规划)

- dsh-busyloop-codexstyle:自研 codex 风格范式(AGENTS.md/工具契约/会话格式),构建在引擎上
- busyloop 工具集:web_search 注入示例、skill picker 交互、anchored-monitor 式推理锚定
- 统一插件模板:Standard Schema + openapi(调研结论)
