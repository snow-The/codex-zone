# 07 — 自研插件生态检视报告

> 范围:snow-The 名下 6 个自研 DSH 插件(dsh-gitkit / dsh-lib-analyzer / dsh-plugin-doctor / dsh-skill-pack / dsh-snapshot / dsh-session-handoff)+ 新建 dsh-codex。
> 方法:直接读取插件源码与 package.json(检视 agent 曾卡住,由主 agent 亲自完成)。

## 1. API 使用检视(全部通过 ✅)

| 插件 | 注册模式 | 结论 |
|---|---|---|
| dsh-gitkit | `ctx.tools.register(tool)` | 最新 API |
| dsh-lib-analyzer | `ctx.tools.register(tool)` | 最新 API |
| dsh-plugin-doctor | `ctx.tools.register(scanTool(name))` | 最新 API |
| dsh-skill-pack | 技能目录挂载(代码层薄) | — |
| dsh-snapshot | `ctx.tools.register(tool)` | 最新 API |
| dsh-session-handoff | `defineTool()` + `ctx.tools.register(...)` | 最新 API + 类型化辅助 |

**无任何过时 API**(无 `ctx.command` / 旧 provide 模式)。dsh-session-handoff 是
API 使用范式标杆(defineTool 携带 JSON schema,类型安全)。

## 2. 发现的问题与已修复

### 双份代码(已修复 ✅)
- **dsh-gitkit / dsh-plugin-doctor / dsh-snapshot**:同时存在
  `src/index.ts`(TS 源码)→ `dist/`(编译产物,main 指向)+ `lib/index.js`(遗留手写版)。
  三者 mtime 相同,dist 头注释带 "(TypeScript)" 标识为编译产物,lib 为旧手写版——
  **运行时走 dist,lib 是死代码**。
- 处理:`git rm lib/` 并推送:
  - dsh-gitkit `6879415`
  - dsh-plugin-doctor `d56d566`
  - dsh-snapshot `e44efe2`
- 处理后三个插件均为 `src → dist` 单一链条,与 dsh-codex 工程化范式一致。

### 结构现状(检视后)

| 插件 | main | 结构 | Hono |
|---|---|---|---|
| dsh-gitkit | ./dist/index.js | src + dist(已清 lib) | ✅ createHonoApp + mount |
| dsh-plugin-doctor | ./dist/index.js | src + dist(已清 lib) | ✅ createHonoApp + mount |
| dsh-snapshot | ./dist/index.js | src + dist(已清 lib) | ✅ createHonoApp + mount |
| dsh-skill-pack | ./dist/index.js | src + dist(技能为文本资源) | — |
| dsh-lib-analyzer | ./lib/index.js | 纯 JS,无 src | — |
| dsh-session-handoff | ./lib/index.js | 纯 JS,无 src | — |
| dsh-codex | ./dist/index.js | src + dist + Hono | ✅(范式源头) |

### Hono 化已落地(本轮完成 ✅)

dsh-gitkit / dsh-plugin-doctor / dsh-snapshot 已按 dsh-codex 同款模式加
`createHonoApp` 工厂(health/version 端点)+ `ctx.http?.mount` 尝试挂载,hono
4.13.3 为运行时依赖,tsc 编译通过,冒烟测试全部 200。commit:
gitkit `8581187`、doctor `5a13d0c`、snapshot `962a976`。

## 3. 后续建议

1. **dsh-lib-analyzer / dsh-session-handoff Hono 化**:这两个仍为纯 JS lib。
   迁移 src/index.ts + Hono 工程量大且 session-handoff 是宿主关键插件,
   建议后续轮次单独处理(先 TS 化,再逐步加 Hono)。
2. **统一 main 路径**:长期目标所有插件 `main=./dist/index.js` + prepack 构建脚本,
   发布 npm 时自动编译。
3. **CI**:仓库均已 public,可加 GitHub Actions(tsc --noEmit + 冒烟测试)防止
   dist 与 src 脱同步(本次双份问题的根源)。
