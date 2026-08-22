# 08 — DSH 生态调研报告(GitHub star 排序 + awesome 清单)

> 调研方式:gh CLI 直接查询 GitHub(按 star 排序)+ awesome 精选列表
> 日期:2026-08-22 | 目的:为 busyloop 家族与自研插件寻找可吸收的强化方向

## 一、DSH 生态全景(按 star 排序)

| stars | 项目 | 价值点 |
|---|---|---|
| 17.8k | anywhere-labs/deepseek-harness-desktop | 桌面端壳;"万物皆插件,桌面本身也是插件" |
| 11.2k | awesome-dsh-plugin/awesome-dsh-plugin | **官方精选插件清单**(UI 增强为主,含 AI 类) |
| 5.4k | zhu1090093659/dsh-web-ui | 全家桶:task board / git graph / 右侧面板 / 皮肤中心 |
| 973 | Anil-matcha/awesome-dsh-plugin | 另一个 curated 清单(英文) |
| 803 | 0xsline/awesome-deepseek-harness | 生态汇总(dsh-external/hub + dsh-plugin topic) |
| 617 | Electricitysheep/dsh-handbook | **从 0 到 1 深度手册**:安装/插件开发/性能调优/实测 |
| 538 | myYangyunfan/dsh_desktop | Windows 桌面客户端(内置 Node + dsh CLI) |
| 404 | MeteorNOX/DeepSeek-Balance-Whale-Widget | 余额监控小部件 |
| 253 | Zhiyuan-Fan/Awesome-DeepSeek-Harness-Plugins | 双语 curated 清单 |
| 193 | anysearch-team/anysearch-dsh | **Web 搜索 provider + 高级搜索工具**(能力型) |
| 190 | ZSeven-W/dsh-ios | iOS 模拟器插件(22 个 agent 工具:引导/构建/无障碍驱动 UI) |
| 185 | d-dev0101/open-sea-skin | WebGPU 海洋皮肤 |

## 二、awesome-dsh-plugin 中值得借鉴的 AI/能力型插件

- **Aik358/dsh-anchored-monitor**:监控 V4 Pro 推理指纹,把模型从 "let me" 拉回 "We will / I will" 专注模式 —— 与我们的梁神模式(锚定)同思路,可互相印证
- **2nd1st/dsh-plugin-open-app**:把 open-mcp-apps 跑进 DSH(每个 MCP app 独立 sidebar 容器/workspace/session)
- **a735624258/dsh-skill-picker**:WorkBuddy 风格技能选择器(composer 旁按钮 → 搜索技能 → 插入 /skill 手势)
- **a903067276-rgb/dsh-hud**:悬浮面板显示 Git 状态 / MCP servers / skills / model & token usage
- **bitxeno/dsh-github-picker**:composer 内直接搜 workspace 仓库的 issue/PR(gh CLI),插入引用 —— 与我们的 gitkit 互补

## 三、可吸收方向(按优先级)

1. **busyloop 工具集**(我们自己造):给 busyloop 注入宿主工具表 —— 已确认宿主 ctx.tools 不可枚举,由调用方注入;调研确认生态里也没有人做"通用宿主工具桥",这是空白
2. **搜索能力**(anysearch-dsh 模式):宿主有 `dsh-web-search-deepseek`(DEEPSEEK_BASE_URL),搜索 provider 可挂到 ctx;busyloop 可注入 `web_search` 工具
3. **技能选择器**(skill-picker 模式):我们的 dsh-skill-pack 有 29 技能,可加一个 `busyloop_skill_picker` 交互工具
4. **状态 HUD**(hud 模式):复用 ctx.llm.listProviders + token meter,做成 `/api/busyloop/status` 端点增强
5. **锚定监控**(anchored-monitor 模式):与梁神模式同源,未来 codexstyle 的 persona 层可集成

## 四、TS 编写方法结论(调研 + 实测)

| 方向 | 结论 |
|---|---|
| 运行时 | Node 22+ 原生 TS type-stripping 可跑 .ts;但我们保持 tsc 类型检查 + esbuild bundle(发布物自包含,已验证) |
| 官方类型 | **@deepseek-ai/dsh-llm 等官方包直接提供完整类型**(LlmRuntime/GenerateOptions/StreamChunk/BlockAssembler),devDep 拉类型、运行时 external 由宿主提供 —— 这是当前最佳实践 |
| Bundle | esbuild `--external:@deepseek-ai/*` + hono 打进 dist:发布物在"纯 dsh+官方包"环境可跑(已 E2E 验证) |
| 测试 | node:test + fake chunks(不碰真模型)+ 宿主解析链验证 + 真机 E2E,四层测试金字塔 |
| 生态借鉴 | awesome-llm-agents(1.5k)的 agent 架构可参考;但我们已确认 busyloop 应复用宿主 ctx 而非独立运行时 |

## 五、落地建议(已排入 10-commit 门禁)

- [ ] busyloop 增加 `providers()` 工具(busyloop_providers 端点/工具)
- [ ] busyloop README(用法/架构/测试金字塔)
- [ ] busyloop-codex 薄壳化挂接 busyloop 家族描述
- [ ] codex-zone 更新(本报告 + 家族架构图)
- [ ] 后续:web_search 工具注入示例 / skill picker 交互

## 风险与教训

- GitHub Packages 不支持 npm deprecate(E400 version.ID cannot be empty):旧包只能保留不更新,改名时靠 README/公告引导
- 生态 UI 插件占 90%,能力型插件(搜索/模拟器/锚定)稀缺 —— 我们的能力型路线差异化明显
