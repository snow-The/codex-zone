# Hono 4.13.3 研究报告：评估「把自研 DSH 插件改用 Hono 组织 TS 代码」

> 研究来源：C:\Users\snow\codex-zone\hono（4.13.3，486 文件，源码 + 测试 + 基准齐全）。所有证据均为 file:line 引用。

## 速览

- Hono 是一个基于 Web Standards（`Request`/`Response`/`FetchEvent`）的 TS 优先框架，`Hono` 类内部路由默认 `SmartRouter([RegExpRouter, TrieRouter])`，入口是对象方法 `app.fetch(request, env?, executionCtx?)`，不是裸函数（src/hono.ts:16-33、src/hono-base.ts:480-486）。
- 核心卖点：零运行时依赖（package.json 无 `dependencies` 字段）、多 runtime（README.md:22,43-47）、`hono/tiny` 预设 <12kB（README.md:44，src/preset/tiny.ts 用 PatternRouter）、一等 TS 类型（类型安全的路径参数、validator 输入输出推断、RPC 式 client）。
- 中间件是 koa-compose 风格洋葱模型（src/compose.ts:15-73），单 handler 有快路径（src/hono-base.ts:431-449）；`c.req`/`c.env`/`c.var`（set/get）全部类型化（src/context.ts:366,315,550-597）。
- Node 上运行需要额外包 `@hono/node-server`（本仓库只是 devDependency，src/adapter 里没有 node adapter；hono 官方把 node 适配器放在独立仓库）；用法是 `serve({ fetch: app.fetch, port, hostname })` 或 `createAdaptorServer(app)`（runtime-tests/node/index.test.ts:1,220-229,276）。
- `app.fetch(req)` 是纯 `(Request) => Response|Promise<Response>`，可以**不启动任何 server**：直接把它挂到宿主进程已有的 HTTP 入口（http.createServer / Bun.serve / 自定义 dispatcher），这正是 DSH 插件「嵌入宿主进程」场景的理想形状。
- zod 校验：核心包只提供通用 `validator(target, fn)`（src/validator/validator.ts:46-88），zod 官方绑定是独立包 `@hono/zod-validator`；本仓库的 validator.test.ts:36-61 有一个参考实现，证明两行代码即可自接 zod。
- 路由组织：`.route(path, subApp)` 挂子应用、`.basePath()` 设前缀、`.mount()` 挂其他框架 handler（src/hono-base.ts:209-233,248-254,329-384），适合把 `/api/tools/:name` 这类插件路由拆成独立模块。
- 迁移成本：CJS 插件无 TS 链 → 需要 tsconfig + tsc/构建到 dist + package.json exports 双格式（ESM/CJS），hono 自己的 package.json:38-43 就是标准模板；cordis 插件若宿主用 require 加载，需保留 CJS 入口。
- 结论：Hono 适合作为 DSH 插件内部 HTTP 代码的「路由器 + 类型层」，替代手写 `if (url.startsWith(...))` 或裸 http 回调；不建议为此引入独立 server 进程，推荐 `createHonoApp(ctx)` 工厂 + `app.fetch` 嵌入现有入口。

---

## 1. Hono 核心模型

### 1.1 Hono 实例是什么

`Hono` 是一个泛型类（不是函数），继承自 `HonoBase`：

```ts
// src/hono.ts:16-33
export class Hono<E extends Env = BlankEnv, S extends Schema = BlankSchema, BasePath extends string = '/'>
  extends HonoBase<E, S, BasePath> {
  constructor(options: HonoOptions<E> = {}) {
    super(options)
    this.router =
      options.router ??
      new SmartRouter({ routers: [new RegExpRouter(), new TrieRouter()] })
  }
}
```

- 泛型参数即「类型层」的载体：`E`（Env：Bindings + Variables）、`S`（Schema：路由表，RPC 式类型）、`BasePath`（路径前缀）——见 src/types.ts:26-33 与 src/hono-base.ts:98-113。
- 默认路由 `SmartRouter` 内部先试 RegExpRouter 再 TrieRouter（src/hono.ts:28-32）；可用 `hono/tiny`（PatternRouter，<12kB，src/preset/tiny.ts:11-19）或自定义 router。
- **入口是 `.fetch()`**：`fetch(request, env?, executionCtx?) => Response | Promise<Response>`（src/hono-base.ts:480-486），内部走 `#dispatch`：`getPath(request)` → `router.match(method, path)` → 建 `Context` → compose/快路径执行（src/hono-base.ts:407-467）。
- 额外入口：`.request()` 测试辅助（传 string/URL 即可发 GET，src/hono-base.ts:500-518）、`fire()`（Service Worker 模式，已废弃，hono-base.ts:537-543）。

### 1.2 路由、中间件、context

**路由注册**（src/hono-base.ts:104-169）：`get/post/put/delete/options/patch/query/all` 是类型化的 `HandlerInterface`，实现上收集 handler 并 `router.add(method, path, [handler, route])`。`app.use(path?, ...mw)` 注册全方法中间件（`METHOD_NAME_ALL` 即 `*`）。

**类型安全的路径参数**：路径是字符串字面量类型，`c.req.param('name')` 的类型由 `ParamKeys<P>`/`ParamKeyToRecord<P>` 推断：

```ts
// src/types.ts:2698-2712
type ParamKey<Component> = Component extends `:${infer NameWithPattern}` ? ... : never
export type ParamKeys<Path> = Path extends `${infer Component}/${infer Rest}`
  ? ParamKey<Component> | ParamKeys<Rest>
  : ParamKey<Path>
// src/request.ts:91-101 — param() 的 key 参数按路径字面量收窄
```

`app.get('/users/:id', (c) => c.req.param('id'))` → `id: string`，拼错键名编译期报错；`c.req.query()`/`queries()`/`header()` 同样类型化（src/request.ts:145-195）。

**Handler / MiddlewareHandler 签名**（src/types.ts:76-95）：

```ts
export type Handler<E, P, I, R> = (c: Context<E, P, I>, next: Next) => R
export type MiddlewareHandler<E, P, I, R> = (c: Context<E, P, I>, next: Next) => Promise<R | void>
```

**Context 对象**（src/context.ts）核心成员：

| 成员 | 含义 | 证据 |
|---|---|---|
| `c.req` | HonoRequest（raw/param/query/header/json/parseBody） | context.ts:366-368; request.ts:34-250 |
| `c.env` | Bindings（注入的宿主环境对象） | context.ts:315 |
| `c.set(k,v)` / `c.get(k)` / `c.var` | 类型化的请求级变量（Variables） | context.ts:550-597 |
| `c.text/json/body/html/redirect/notFound/newResponse` | 响应构造，返回 `TypedResponse`（可被 Schema 收集） | context.ts:658-793 |
| `c.res` / `c.finalized` / `c.error` | 响应结果 / 是否已终结 / 中间件错误 | context.ts:317,333,403-429 |

**中间件组合**是 koa-compose 洋葱模型（src/compose.ts:15-73）：`compose(middleware, onError, onNotFound)`，`next()` 继续下层，`next()` 重复调用抛错（compose.ts:33-35），末尾未 finalize 则走 notFound（compose.ts:62-65）。单 handler 时跳过 compose 走同步快路径（hono-base.ts:431-449）。

### 1.3 最小可运行示例（Node）

Node 没有内置于 hono 的 adapter，用官方独立包 `@hono/node-server`（本仓库 runtime-tests/node/index.test.ts:1 的用法即官方样例）：

```ts
// app.ts —— 纯框架层，与 runtime 无关
import { Hono } from 'hono'
const app = new Hono()

app.get('/', (c) => c.text('Hono!'))                       // README.md:26-33 同款
app.get('/api/tools/:name', (c) => c.json({ name: c.req.param('name') }))

export default app
```

```ts
// server.ts —— 仅 Node 启动层
import { serve } from '@hono/node-server'
import app from './app'

serve({ fetch: app.fetch, port: 3000, hostname: '127.0.0.1' })  // runtime-tests/node/index.test.ts:220-229
```

`serve()` 返回 Node `http.Server`（可 close/挂事件），回调拿到 `AddressInfo`（index.test.ts:226-229）；还有 `createAdaptorServer(app)` 直接得到一个可 `listen()` 的 http server（index.test.ts:276-277）。`c.body(Buffer.from(...))` 等 Node 特性可直接用（index.test.ts:251-273）。

---

## 2. 适合什么场景：嵌入式 vs 独立 server

### 2.1 `app.fetch` 就是嵌入点

`app.fetch(request, env?, executionCtx?)` 是纯函数签名（src/hono-base.ts:480-486），Hono 的设计哲学是「应用 = fetch 处理器」，运行时（server）与框架解耦。因此：

- **宿主进程内嵌**：DSH 插件已有 HTTP 入口（或愿意自己起一个 `http.createServer`）时，只需把 node 的 `IncomingMessage`/`ServerResponse` 翻译成 `Request`/`Response`，然后 `app.fetch(req)`。`@hono/node-server` 的 `serve({ fetch: app.fetch })` 就是干这个的（index.test.ts:220-229）；`createAdaptorServer(app)` 同理。
- **不启动 server 也能跑**：`app.request('/path')` 直接在当前进程内测试/调用（hono-base.ts:500-518）；`app.fetch(new Request(...))` 也可以被任何 Web 标准 fetch 兼容的入口（Bun.serve、Workers fetch handler、Fastly、Lambda 包装器）调用——README.md:45 明言「同一份代码跑所有平台」。
- **env 注入**：`fetch` 第二个参数就是 `c.env`（Bindings），DSH 插件可以把 `ctx`（cordis 上下文）作为 env 塞进去，handler 里 `c.env.ctx` 取用——这是与 cordis 组合的关键接口（hono-base.ts:480-486; context.ts:315）。

### 2.2 嵌入式与独立 server 的区别

| 维度 | 独立 server（serve 启动监听端口） | 嵌入式（app.fetch 挂到现有入口） |
|---|---|---|
| 生命周期 | 自己 listen/close，随插件停止而销毁 | 随宿主入口生命周期，无需管理端口 |
| 端口/暴露面 | 需要选端口、绑 hostname（127.0.0.1 才安全） | 完全由宿主决定，天然 loopback |
| 适用场景 | 插件要对外提供独立 HTTP 服务 | DSH 内已有 web 入口/隧道/SSE，只想规范路由层 |
| 组合 | `@hono/node-server` 的 serve | 任何 `(req) => res` 包装 |

### 2.3 能不能挂到 cordis 插件的某个 http 入口

可以，两条路：

1. **若 DSH/宿主已暴露 fetch 风格入口**：直接把 `app.fetch` 绑定为入口 handler（和 service-worker 的 `fire(app)` 同构，hono-base.ts:537-543 弃用但模式如此）。
2. **若宿主只有 node http 入口**：用 `@hono/node-server` 的 `createAdaptorServer(app)`（index.test.ts:276）拿到 `http.Server`，再挂到宿主（例如作为现有 server 的 request listener 链的一部分），或自己写 20 行转换层。
3. **完全不依赖 node-server**：cordis 插件自己 `http.createServer((req,res)=>{...app.fetch(webRequest)})`，Web 标准 `Request` 在 Node 16.9+ 全局可用（hono engines: node >=16.9.0，package.json:704）。

结论：**嵌入式是 DSH 插件的正确姿势**——Hono 只承担「路由 + 中间件 + 类型」，端口与进程边界仍归宿主。

---

## 3. DSH 插件用法建议（具体方案）

### 3.1 总体结构：工厂函数 + typed Env 注入 cordis ctx

Hono 的 `createFactory`（src/helper/factory/index.ts:332-375）就是为「带共享 Env 的应用工厂」设计的：`createFactory<{ Bindings: { ctx } }>()` 后 `factory.createApp()`、`factory.createMiddleware()`、`factory.createHandlers()` 全部继承 Env 类型。

```ts
// src/plugin-app.ts —— 纯框架层，不 import cordis 运行时
import { createFactory } from 'hono/factory'
import type { Context } from 'cordis'            // 仅类型

export type PluginEnv = { Bindings: { ctx: Context } }

export const createHonoApp = () => {
  const { createApp, createMiddleware, createHandlers } =
    createFactory<PluginEnv>()
  const app = createApp()

  // 全局中间件：loopback-only 鉴权（挂在 app.use('*')）
  const loopbackOnly = createMiddleware(async (c, next) => {
    const info = getConnInfo(c)                  // hono/conninfo 类型; node-server 提供实现
    if (info.remote.address !== '127.0.0.1' && info.remote.address !== '::1') {
      return c.text('Forbidden', 403)
    }
    await next()
  })
  app.use('*', loopbackOnly)

  // 路由分组：子应用独立文件
  const tools = new Hono<PluginEnv>()
  tools.get('/:name', (c) => {
    const { ctx } = c.env                       // ← cordis ctx 由此进入
    return c.json({ name: c.req.param('name'), status: ctx.registry?.status ?? 'ok' })
  })
  app.route('/api/tools', tools)                // → GET /api/tools/:name

  return app
}
```

要点（全部有源码证据）：
- **分组**：`.route(path, subApp)` 把子应用挂到前缀（hono-base.ts:209-233）；子应用可再 `.basePath()`（248-254）。`/api/tools/:name` 正好用 `tools.get('/:name')` 表达，类型自动收窄。
- **中间件**：`app.use('*', mw)` 全路径挂载（hono-base.ts:157-169）；loopback 判断用 `getConnInfo(c)`（helper/conninfo/index.ts 导出类型；node-server 提供真实实现，见 hono 文档 conninfo 页）。若宿主信息在 request header 或 env 里，也可直接 `c.req.header('x-forwarded-for')` 或 `c.env.ctx` 查。
- **鉴权模式**：middleware 里 `return c.text('Forbidden', 403)` 短路（MiddlewareHandler 可返回 Response，types.ts:83-88），后续 handler 不执行。
- **类型校验（zod）**：核心 `validator(target, fn)` 已内置（src/validator/validator.ts:46-88），支持 `json/form/query/param/header/cookie` 六种 target（ValidationTargets，src/types.ts:2683-2690），校验结果自动并入 `c.req` 的类型（validator.ts 的 `V.in/V.out` 泛型）。zod 绑定在独立包 `@hono/zod-validator`，但参考实现仅 26 行（validator.test.ts:36-61）：`schema.safeParse(value)` + `validator(target, validationFunc)` 即可，输出类型 `z.output<T>` 自动注入后续 handler 的 `c.req`。
- **与 cordis 组合**：三种注入点——(a) `fetch` 的 env 参数（推荐：`app.fetch(req, { ctx })`）；(b) `createFactory<{ Bindings: { ctx } }>()` 类型层绑定；(c) handler 闭包捕获插件实例字段。推荐 (a)+(b) 组合，框架层保持纯函数可测试。

### 3.2 最小落地骨架（DSH 插件内）

```
plugin/
  lib/            ← 编译产物（CJS，cordis require 加载）
  src/
    index.ts      ← cordis 插件入口：register 时 createHonoApp() 并挂载
    app.ts        ← createHonoApp()：中间件 + 路由分组
    routes/
      tools.ts    ← 子应用（/api/tools）
      health.ts
    middleware/
      loopback.ts ← loopback-only 鉴权
    types.ts      ← PluginEnv 等共享类型
```

入口示例：

```ts
// src/index.ts（cordis 插件入口）
import { Context } from 'cordis'
import { createHonoApp } from './app'
import { serve } from '@hono/node-server'   // 或挂宿主已有入口

export const inject = ['server']             // 按需声明依赖

export function apply(ctx: Context) {
  const app = createHonoApp()
  const server = serve({ fetch: app.fetch, port: ctx.config.port, hostname: '127.0.0.1' })
  ctx.on('dispose', () => server.close())
}
```

---

## 4. 对比：不用 Hono（原生 http / express / fastify）的 TS 规整度

| 维度 | 原生 node:http | express | fastify | **Hono** |
|---|---|---|---|---|
| 类型安全路由/路径参数 | 无（手写解析 `req.url`） | 弱（`req.params` 是 any/string 索引） | 中（fastify 有泛型 schema 但仪式感重） | **强**（路径字面量 → `ParamKeys` 推导，types.ts:2698-2712） |
| 中间件类型 | 无 | 手写类型 | 插件体系较复杂 | 一等 `MiddlewareHandler`（types.ts:83-88），compose 洋葱（compose.ts:15-73） |
| 请求/响应体解析 | 手动拼 buffer | body-parser 外部包 | 内置 schema 校验 | `c.req.json()/parseBody()` + 缓存（request.ts:209-250） |
| validator 类型流 | — | 无 | schema 是 JSON Schema 风格 | `Input.in/out` 双向类型，zod safeParse 输出直接进 handler 类型（validator.ts:46-88, validator.test.ts:36-61） |
| 响应类型 | — | 无 | 有 | `c.json<T>()` 返回 `TypedResponse<T>`，可进 Schema（context.ts:181-208, types.ts:70-74） |
| 测试 | 手动 | supertest | inject | `app.request()`（hono-base.ts:500-518），零 server |
| 依赖体积 | 0 | 庞大（express 全家桶） | 中（fastify 自带多） | **零依赖**，tiny <12kB（README.md:44, package.json 无 dependencies） |
| 多 runtime | Node only | Node only | Node only（web 需额外） | Cloudflare/Deno/Bun/Node/Lambda/Edge 同代码（README.md:22,45） |
| 异步风格 | 回调/事件 | 回调链 | async 好 | 全 async/await + Promise 响应（HandlerResponse，types.ts:70-74） |

**结论**：在「TS 规整度」上，Hono 相对 express/原生 http 是代差级的（类型推导内建于框架而非靠装饰器/代码生成）；相对 fastify 的优势是**零依赖 + 类型机制更轻 + runtime 无关**，代价是生态（插件/文档规模）不如 express/fastify 大。

### tree-shaking / 零依赖证据

- package.json **没有 `dependencies` 字段**（只有 devDependencies，package.json:676-701）——运行时零第三方依赖，只依赖 Web 标准全局对象（README.md:44 明言 "zero dependencies and uses only the Web Standard API"）。
- exports map 把每个中间件/helper/adapter 独立成子路径（package.json:38-418，如 `./basic-auth`、`./validator`、`./router/reg-exp-router`），按需 import 即按需打包（tree-shaking 友好）；发布包 `files: ["dist"]`（package.json:9-12）。
- `hono/tiny` 预设（PatternRouter）< 12kB（README.md:44；src/preset/tiny.ts:11-19）。
- 构建出 ESM + CJS 双格式 + `.d.ts`（package.json:5-8: `main: dist/cjs/index.js` / `module: dist/index.js` / `types: dist/types/index.d.ts`），`exports` 里 `import`/`require` 各给一份（package.json:38-43）。

---

## 5. 对现有插件迁移成本（CJS lib/index.js → TS + Hono）

### 5.1 现状假设与目标

- 现状：CommonJS `lib/index.js`，无 TS 构建链，cordis 直接 require。
- 目标：`src/**/*.ts` 源码 + tsc 编译到 `lib/`（或 `dist/`），运行时仍是 CJS（保 cordis 兼容），类型/路由由 Hono 承担。

### 5.2 迁移步骤（按序）

1. **加构建工具**：`typescript`（+ 可选 esbuild/bun 加速，hono 自己用 bun + build.ts，package.json:30）。devDependencies 增加 `typescript`、`@types/node`；运行时 dependencies 增加 `hono`、`@hono/node-server`。
2. **tsconfig.json**（参考 hono 自己的 tsconfig.base.json:2-14）：
   ```jsonc
   {
     "compilerOptions": {
       "target": "ES2022",              // hono 同款
       "module": "CommonJS",            // cordis 宿主 require 加载
       "moduleResolution": "Bundler" 或 "Node",
       "strict": true,
       "declaration": true,             // 产出 .d.ts
       "outDir": "lib",
       "esModuleInterop": true,
       "skipLibCheck": true
     },
     "include": ["src"]
   }
   ```
3. **package.json 调整**：`"main": "lib/index.js"`（CJS 入口保持不变，cordis 无感知）；`"types": "lib/index.d.ts"`；`"scripts": { "build": "tsc -p tsconfig.json", "prepublishOnly": "npm run build" }`。若想同时给 ESM 消费方，可学 hono 的 exports 双格式（package.json:38-43）——但 cordis 场景先保 CJS 即可。
4. **迁移代码组织**：把 lib/index.js 里散落的 HTTP 分支重写为 `src/app.ts` 的 `createHonoApp()`（第 3 节骨架）；`ctx` 通过 `fetch` 的 env 参数注入（不反向依赖 Hono 到 cordis）。
5. **验证**：`npm run build` + `node -e "require('./lib')"` 确认 CJS 可加载；用 `app.request()` 写 smoke test（hono-base.ts:500-518，无需起 server）。

### 5.3 推荐目录结构（最终形态）

```
plugin/
├── package.json          # main: lib/index.js; types: lib/index.d.ts; deps: hono, @hono/node-server
├── tsconfig.json         # 上述配置
├── src/
│   ├── index.ts          # cordis 入口：apply(ctx) → createHonoApp() + 挂载/关闭
│   ├── app.ts            # createHonoApp()：use 全局中间件 + route 分组
│   ├── types.ts          # PluginEnv = { Bindings: { ctx: CordisContext } }
│   ├── routes/
│   │   ├── tools.ts      # 子应用：/api/tools/:name
│   │   └── health.ts
│   └── middleware/
│       └── loopback.ts   # loopback-only 鉴权中间件
└── lib/                  # tsc 输出（构建产物，gitignore 可选）
```

### 5.4 风险与注意

- **ESM 陷阱**：`@hono/node-server` 与 `hono` 的 exports 都带 `require` 分支（hono package.json:38-43），CJS require 可用；但若宿主是 ESM 加载 cordis 插件，需确认 `module` 产物路径一致。
- **Node 版本**：hono 要求 Node >= 16.9.0（package.json:704），Web 标准 `Request`/`Response` 需该版本以上。
- **迁移收益边界**：纯逻辑型插件（无 HTTP）不值得迁移；有 HTTP/SSE/定时回调 Webhook 的插件收益最大。
- **不做的事**：不要为了 Hono 引入独立进程/端口；保持「app.fetch 嵌入」模式。

---

## 对 DSH 插件迁移的启示

- Hono 是「类型安全的路由层 + 中间件层」，`app.fetch` 纯函数签名让它可以零端口嵌入 cordis 插件的既有 HTTP 入口——先选一个含 HTTP 代码的插件试点，把 `if (url.startsWith('/api/tools/'))` 分支重写为 `tools.get('/:name')`，立即获得路径参数与响应类型推导。
- cordis `ctx` 通过 `fetch(req, { ctx })` 的 env 参数注入（`c.env.ctx`），框架层保持与 cordis 解耦、可单测；`createFactory` 在类型层绑定 `Bindings`，无需全局类型体操。
- 迁移只需 tsc + 把 package.json main 指向编译产物，CJS 入口不动、宿主无感知；零运行时依赖与 tiny 预设让插件体积几乎不变。
- loopback-only 鉴权、body 解析、zod 校验都是开箱即用的 middleware/validator，比手写省掉大量样板；测试用 `app.request()` 无需起 server。
- 若 DSH 生态未来要跨 runtime（如 Bun/边缘部署）或组件化复用 HTTP 层，Hono 的「同代码多 runtime」是当前选项里成本最低的一条路径。
