# NestJS 检查清单

本清单覆盖 NestJS 应用的模块架构、依赖注入、安全、性能和代码质量问题。

## 快速索引

| 问题类型 | 级别 | 参考章节 |
|---------|------|---------|
| 模块循环依赖（forwardRef 滥用） | P1 | 模块架构 |
| Provider 未声明在模块的 providers 中 | P1 | 依赖注入 |
| Controller 中写业务逻辑 | P2 | 代码质量 |
| 未使用 class-validator 校验 DTO | P1 | 校验 |
| 全局异常过滤器缺失 | P1 | 异常处理 |
| Guard 中直接调用数据库 | P2 | 守卫与拦截器 |
| TypeORM repository 无事务 | P1 | 数据库 |
| 同步阻塞代码在请求路径中 | P1 | 性能 |
| ConfigModule 未使用 Joi 校验 | P1 | 配置 |
| 循环中 await（串行数据库查询） | P2 | 数据库 |
| @Res() 直接操作 response 绕过拦截器 | P2 | 控制器 |
| JWT secret 硬编码 | P0 | 认证与安全 |
| CORS 配置 origin: true 且 credentials | P0 | 认证与安全 |
| request-scoped provider 过度使用 | P2 | 性能 |
| Bull/BullMQ 任务无重试/死信队列 | P1 | 任务队列 |
| GraphQL resolver 无权限检查 | P1 | GraphQL |
| 动态模块配置未做异步工厂 | P2 | 模块架构 |
| 日志打印请求体/响应体含敏感数据 | P0 | 日志 |
| Swagger/OpenAPI 暴露内部 DTO 结构 | P3 | 安全 |

---

## 模块架构

### 模块设计

- 循环依赖：`ModuleA` imports `ModuleB`，`ModuleB` imports `ModuleA` → **P1**，应重构为共享模块或拆分接口，`forwardRef` 仅为最后手段
- 全局模块（`@Global()`）过度使用 → **P2**，应仅在真正需要全局共享的场景（如 ConfigModule、DatabaseModule）使用
- 单个模块包含过多 Provider/Controller（>20 个）→ **P2**，应按领域拆分
- 动态模块配置未使用 `registerAsync` / `forRootAsync`（硬编码而非从 ConfigService 读取）→ **P2**

### 模块导出

- Provider 未在模块的 `exports` 中导出 → **P1**（其他模块导入时 DI 注入失败）
- 导出了不应对外暴露的内部 Provider → **P3**
- 导入整个模块仅为使用其中 1-2 个 Service（可考虑拆分子模块）→ **P2**

---

## 依赖注入

### Provider 注册

- Provider 使用了 `@Injectable()` 但未声明在所在模块的 `providers` 数组中 → **P1**
- `@Inject()` 装饰器使用字符串 token 但无对应的 `provide` 常量 → **P1**
- 自定义 Provider 使用 `useClass` 但指定的类未标记 `@Injectable()` → **P1**

### 作用域

- **request-scoped** Provider 过度使用 → **P2**（每条请求创建新实例，影响性能），优先使用 singleton（默认）scope
- request-scoped Provider 注入到 singleton Provider 中，未显式处理 scope 提升 → **P1**
- 在 singleton Service 中使用 `REQUEST` token 注入请求对象 → **P1**（应使用 request-scoped 或通过参数传递）

### 循环依赖

- Service 之间循环依赖（`AService → BService → AService`）→ **P1**，应提取公共逻辑到第三个 Service
- 使用 `forwardRef(() => Module)` 解决模块循环依赖 → **P1**（应视为技术债，优先重构消除循环）

---

## 控制器

### 路由设计

- Controller 中直接写业务逻辑而非委托给 Service → **P2**（Controller 应仅负责路由、参数提取、响应格式化）
- 使用 `@Res()` 装饰器直接操作 response 对象（绕过 NestJS 拦截器/异常过滤器/管道）→ **P2**
- 路由参数使用 `@Param('id')` 未做格式校验 → **P2**

### DTO 与校验

- 入参 DTO 未使用 `class-validator` 装饰器校验（`@IsString`/`@IsInt`/`@Min` 等）→ **P1**
- `ValidationPipe` 未全局注册或未配置 `whitelist: true`（允许未知属性通过）→ **P1**
- DTO 中敏感字段（如 `role`/`isAdmin`）在 `@IsOptional()` 但未防止客户端传入 → **P0**
- 使用 `@Transform()` 转换类型但未处理异常值（如 `parseInt` 遇到非数字返回 NaN）→ **P2**

---

## 守卫、拦截器、管道与过滤器

### 守卫（Guard）

- Guard 中直接调用数据库/外部 API（阻断请求链路，影响性能）→ **P2**，应通过 Service 间接调用并使用缓存
- 全局 Guard 未处理公开路由的豁免逻辑 → **P1**
- JWT Guard 未验证 `exp`、`iss`、`aud` 声明 → **P1**

### 拦截器（Interceptor）

- 响应拦截器未处理 `Observable` 和 `Promise` 两种返回类型 → **P2**
- 拦截器中抛出异常但未使用 `throw`（错误被静默吞掉）→ **P1**

### 管道（Pipe）

- 自定义 Pipe 未继承 `PipeTransform` 接口 → **P1**
- 管道中执行异步校验（如数据库唯一性检查）未处理超时 → **P2**

### 异常过滤器（ExceptionFilter）

- 全局异常过滤器缺失 → **P1**（生产环境默认返回标准错误格式）
- 异常过滤器中 `catch` 的异常未区分类型（如 `HttpException` vs 未知错误）→ **P2**
- 错误响应中泄露内部堆栈信息 → **P1**

---

## 认证与安全

### JWT / Passport

- JWT secret 硬编码在代码中 → **P0**，必须从环境变量/ConfigService 读取
- JWT 签发时 payload 包含敏感数据（密码/手机号/身份证）→ **P0**（JWT base64 可解码）
- `@nestjs/passport` 的 Strategy 中 `validate` 方法返回 null 未正确处理 → **P1**
- 未限制 JWT 有效期（token 永不过期）→ **P1**

### 通用安全

- CORS 配置 `origin: true`（反射所有 Origin）且 `credentials: true` → **P0**
- `helmet` 未集成 → **P1**
- CSRF 保护缺失（如使用 Cookie session 时未启用 csurf）→ **P1**
- `@nestjs/throttler` 未在认证/敏感接口上配置 → **P1**

---

## 数据库

### TypeORM

- Repository 操作未包裹在事务中（多表写入部分成功）→ **P1**
- 循环中 `await repository.save()`（应使用 `repository.save(entities[])` 批量操作）→ **P2**
- `find` 查询使用 `relations` 加载所有关联（应懒加载或按需 JOIN）→ **P2**
- Entity 列定义缺少 `@Column` 选项（如未设置 `nullable`/`type`/`length`）→ **P3**

### Prisma

- Prisma Client 实例在多个 Service 中重复实例化 → **P2**（应封装为全局 Module）
- `prisma.$transaction` 未使用交互式 API（导致长事务）→ **P2**
- 查询未使用 `select` 只返回需要字段（`SELECT *` 全量返回）→ **P3**

### 通用

- 数据库连接配置未使用连接池（或使用默认连接数）→ **P1**
- 数据库凭据硬编码在 `app.module.ts` 中 → **P0**

---

## 任务队列（Bull/BullMQ）

- 任务处理器（Processor）未处理异常（任务失败后无重试/死信队列）→ **P1**
- 任务没有设置超时或重试次数上限 → **P1**
- 延迟任务数量巨大会导致 Redis 内存压力 → **P2**
- 敏感数据（用户手机号/邮箱）放在任务 payload 中明文化 → **P2**

---

## GraphQL（@nestjs/graphql）

- Resolver 查询未使用 `@ResolveField` 优化关联数据（导致 N+1 查询）→ **P1**
- 使用 `DataLoader` 前的 N+1 问题 → **P1**
- Mutation 输入未使用 `@ArgsType`/`@InputType` 校验 → **P2**
- Resolver 中无权限检查（`@Roles`/Guard 缺失）→ **P1**
- GraphQL 复杂度/深度限制缺失（防嵌套查询 DoS）→ **P1**
- 生产环境未禁用 playground / introspection → **P1**

---

## 性能

### 启动与运行时

- 同步阻塞代码在 Controller/Service 请求路径中（如 `fs.readFileSync`）→ **P1**
- 异步方法中执行 CPU 密集计算阻塞事件循环 → **P2**（应用 worker_threads 或队列异步处理）
- request-scoped Provider 过多导致每次请求创建大量实例 → **P2**
- 未使用 `@nestjs/serve-static` 处理静态资源（通过 Node.js 管道传输）→ **P3**

### 缓存

- 频繁查询数据未使用 `@nestjs/cache-manager` 缓存 → **P2**
- 缓存 TTL 设置不当（热点数据过期时间相同导致雪崩）→ **P1**

---

## 配置

- `@nestjs/config` 的 `ConfigModule.forRoot()` 未使用 `Joi` / `class-validator` 校验环境变量 → **P1**
- 环境变量命名冲突（未使用命名空间 `registerAs` 隔离）→ **P3**
- `.env` 文件被提交到 Git → **P0**
- 配置值在 Service 中直接通过 `process.env` 读取（绕过 ConfigService）→ **P2**

---

## 日志

- 使用 `console.log` 替代 NestJS Logger → **P3**（无法通过 Logger 级别控制输出）
- 日志中打印完整请求体/响应体含用户密码/Token → **P0**
- 异常日志无请求 ID 或用户 ID 上下文 → **P2**
- 生产环境日志级别设置为 `debug`/`verbose` 输出大量敏感调试信息 → **P1**

---

## 测试

- Controller 测试中 Service 未 Mock（依赖真实数据库）→ **P2**
- `Test.createTestingModule` 未使用 `overrideProvider` 替换外部依赖 → **P2**
- E2E 测试中数据库未使用独立测试库/事务回滚 → **P1**
- 测试使用的 `app.init()` 后未 `app.close()` → **P2**

---

## 常见反模式

| 反模式 | 说明 | 修复方向 |
|--------|------|----------|
| **Controller 包含业务逻辑** | Controller 中直接查询数据库/调用外部 API | 委托给 Service 层 |
| **forwardRef 滥用** | 模块/Service 循环依赖用 forwardRef 掩盖 | 提取共享模块消除循环 |
| **@Res() 绕过管道** | 直接操作 response 导致拦截器/过滤器失效 | 用 `@HttpCode`/自定义拦截器替代 |
| **全局 Guard 无豁免** | 所有路由强制认证但登录接口也被拦截 | 使用 `@Public()` 装饰器 + Reflector |
| **单个 Module 过大** | 单模块含 20+ Controller/Service | 按领域拆分（UserModule/OrderModule） |
| **同步阻塞代码** | `fs.readFileSync`/`execSync` 在请求路径 | 全部使用异步版本 |
