# Node.js 后端检查清单

本清单覆盖 Node.js（Express/Koa/Fastify/NestJS）应用的安全、并发、性能和代码质量问题。

## 快速索引

| 问题类型 | 级别 | 参考章节 |
|---------|------|---------|
| SQL 注入（模板字面量拼接） | P0 | 安全检查 |
| NoSQL 注入（MongoDB $where） | P0 | 安全检查 |
| child_process exec 拼接用户输入 | P0 | 安全检查 |
| eval/Function 执行用户输入 | P0 | 安全检查 |
| EJS/Pug 模板注入 | P0 | 安全检查 |
| Express 中间件顺序错误 | P1 | Express 专项 |
| 错误处理中间件缺失（4 参数） | P1 | Express 专项 |
| helmet 缺失 | P1 | Express 专项 |
| Promise 无 catch | P1 | 并发 |
| EventEmitter 监听器泄漏 | P1 | 并发 |
| unhandledRejection 未处理 | P1 | 并发 |
| Event Loop 阻塞（同步大 JSON/crypto） | P1 | 性能 |
| Stream 未处理背压 | P2 | 性能 |
| callback hell | P2 | 代码质量 |
| process.exit() 用于错误处理 | P2 | 代码质量 |
| 同步 I/O 在请求路径 | P1 | 常见反模式 |
| HTTP 请求无超时 | P1 | 常见反模式 |

---

## 安全检查

### SQL 注入

- 使用模板字面量拼接用户输入构造 SQL → **P0**，必须用参数化查询或 ORM 参数绑定
- Sequelize/Knex/TypeORM 的 `raw` / `query` 方法未使用参数化 → **P0**
- 动态列名/表名/ORDER BY 来自用户输入且未白名单 → **P1**

### NoSQL 注入

- MongoDB `$where` / `$expr` 操作符使用用户输入 → **P0**
- Mongoose 查询中直接展开用户输入对象（操作符注入）→ **P0**
- Redis 命令拼接用户输入 → **P1**

### 命令注入

- `child_process.exec()` / `execSync()` 字符串拼接用户输入 → **P0**（应用 `execFile`/`spawn`）
- `eval()` / `new Function()` 执行不可信字符串 → **P0**
- `require()` 使用动态路径含用户输入 → **P1**

### 模板注入（SSTI）

- EJS `render()` 使用用户可控的模板字符串 → **P0**
- Pug 模板中 `!=`（未转义输出）用于用户输入 → **P0**
- Handlebars `{{{ triple-stash }}}` 原样输出用户内容 → **P0**

### 路径遍历

- `path.join()` / `path.resolve()` 拼接用户输入未校验 → **P0**
- 静态文件服务（`express.static`）目录未限制 → **P1**

---

## Express 专项

### 中间件

- 中间件顺序错误：认证中间件在路由之后注册 → **P1**
- 错误处理中间件未使用 4 参数签名 `(err, req, res, next)` → **P1**
- `req.body` 未校验直接使用 → **P1**
- `helmet` 缺失（安全响应头）→ **P1**
- CORS 配置 `origin: true` 或 `origin: '*'` + credentials → **P0**
- `express.urlencoded` 未设置 `limit` → **P2**

### 会话与认证

- session secret 使用默认值/硬编码 → **P0**
- cookie-session 未设 `httpOnly`/`secure`/`sameSite` → **P1**
- JWT secret 弱密钥 → **P0**

---

## 并发

### Promise / async-await

- Promise 链缺少 `.catch()` → **P1**
- `async` 函数调用未 `await` 且不处理错误 → **P1**
- `Promise.all` 一个 reject 导致其他成功结果被丢弃未处理 → **P2**
- 循环中使用 `await` 导致串行（应用 `Promise.all`）→ **P2**

### 事件循环

- `process.on('unhandledRejection')` 未注册 → **P1**
- `EventEmitter` 监听器超过 10 个未设 `setMaxListeners` 产生内存泄漏警告 → **P1**
- `process.on('uncaughtException')` 内未执行优雅关闭 → **P1**

---

## 性能

### Event Loop 阻塞

- 同步 `JSON.parse` 解析大文件 → **P1**（应流式解析或用 worker_threads）
- `crypto.randomBytes` / `crypto.pbkdf2` 等使用同步版本 → **P1**（应用异步版本）
- 长循环/递归在请求处理路径中 → **P2**

### Stream 与 I/O

- 可读流未处理 `backpressure`（`pipe` 自动处理，手动 `.on('data')` 需注意）→ **P2**
- `fs.readFile` 一次性读取大文件 → **P1**（应用 `fs.createReadStream`）
- HTTP 响应未设置 `Content-Length` 或 `Transfer-Encoding: chunked` → **P3**

### 内存

- `require` 缓存误用：期望重新加载却依赖缓存不生效 → **P2**
- 闭包持有大对象引用导致内存泄漏 → **P1**
- 全局变量持续追加（如日志数组无上限）→ **P1**

---

## 代码质量

### 异步模式

- callback hell：3+ 层嵌套回调 → **P2**（应用 async/await 或 Promise 链）
- 混用 callback 和 Promise 风格 → **P2**

### 错误处理

- `process.exit()` 在请求处理中直接终止进程 → **P2**（应用错误传播 + 优雅关闭）
- `try-catch` 中仅 `console.error` 吞没异常 → **P1**
- 未区分操作错误（预期）和程序错误（bug）→ **P2**

### 模块与结构

- 全局变量（`global.x = ...`）污染 → **P2**
- 循环依赖 → **P2**
- 单个文件包含路由 + 业务逻辑 + 数据访问 → **P2**

---

## 常见反模式

| 反模式 | 说明 | 修复方向 |
|--------|------|----------|
| **同步 I/O 在服务路径** | `fs.readFileSync` 等阻塞事件循环 | 全部使用异步版本 |
| **HTTP 请求无超时** | `fetch`/`axios` 无 timeout/AbortSignal | 设置超时 + 重试策略 |
| **环境变量硬编码默认值** | `process.env.KEY \|\| 'dev-secret'` | 缺失时报错退出 |
| **console 直接用于生产日志** | 无日志级别、无结构化 | 使用 pino/winston 等日志库 |
| **npm install 无 --production** | 生产环境安装 devDependencies | 使用 `--production` 或 CI 分离 |
