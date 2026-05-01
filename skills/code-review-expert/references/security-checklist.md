# 安全与可靠性检查清单

## 输入/输出安全

- **XSS**：不安全的 HTML 注入、`dangerouslySetInnerHTML`、未转义模板、innerHTML 赋值
- **注入**：通过字符串拼接或模板字面量的 SQL/NoSQL/命令/GraphQL 注入
- **SSRF**：用户控制的 URL 访问内部服务，无白名单验证
- **路径遍历**：文件路径中的用户输入未清理（`../` 攻击）
- **文件上传安全**：未限制文件类型/大小、未验证 MIME 类型、文件名注入、存储路径可由用户控制、解压炸弹（ZIP bomb）、恶意文件（伪装成图片的脚本）
- **原型污染**：JavaScript 中不安全的对象合并（`Object.assign`、用户输入的展开运算符）

## 认证/授权

- 读写操作缺少租户或所有权检查
- 新端点缺少认证守卫或 RBAC 强制
- 信任客户端提供的角色/标志/ID
- 访问控制失效（IDOR - 不安全的直接对象引用）
- 会话固定或弱会话管理

## JWT 与 Token 安全

- 算法混淆攻击（期望 `RS256` 时接受 `none` 或 `HS256`）
- 弱密钥或硬编码密钥
- 缺少过期时间（`exp`）或未验证
- JWT 载荷中的敏感数据（Token 是 base64 编码，非加密）
- 未验证 `iss`（签发者）或 `aud`（受众）

## 密钥与 PII

- 代码/配置/日志中的 API Key、Token 或凭证
- Git 历史或暴露给客户端的环境变量中的密钥
- PII 或敏感载荷的过度日志记录
- 错误消息中缺少数据脱敏

## 供应链与依赖

- 未固定的依赖允许恶意更新
- 依赖混淆（私有包名冲突）
- 从不受信任的源或 CDN 导入，无完整性检查
- 存在已知 CVE 的过时依赖

## CORS 与安全响应头

- 过于宽松的 CORS（带凭证时 `Access-Control-Allow-Origin: *`）
- 缺少安全头：
  - `Content-Security-Policy`：防止 XSS 和数据注入
  - `X-Frame-Options: DENY` 或 `SAMEORIGIN`：防止点击劫持
  - `X-Content-Type-Options: nosniff`：防止 MIME 类型嗅探
  - `Referrer-Policy`：控制 Referer 信息泄露
  - `Strict-Transport-Security`（HSTS）：强制 HTTPS
  - `Permissions-Policy`：限制 API 使用
- 暴露的内部头或堆栈跟踪

### CSP 重点检查

- 是否使用了 `unsafe-inline`？应优先使用 nonce 或 hash
- 是否使用了 `unsafe-eval`？可能阻断合法动态代码
- `script-src` 是否包含不受信任的域或 `*`？
- `connect-src` 是否允许任意外部连接？
- 是否缺少 `object-src 'none'`？（防止 Flash/插件注入）
- 是否缺少 `base-uri` 限制？（防止 base tag 注入劫持相对 URL）
- `frame-ancestors` 是否等同于 X-Frame-Options？

## CSRF（跨站请求伪造）

- 状态变更操作（POST/PUT/DELETE）缺少 CSRF Token 或 SameSite Cookie
- 依赖仅前端 CSRF 保护（如 Double Submit Cookie 但未验证）
- 使用 GET 请求执行状态变更操作
- SPA 中通过 Authorization Header 传递 Token 时未额外防护（需确认 Token 非自动附加）

## 开放重定向

- 登录/登出回调 URL 未进行白名单校验
- 用户可控的重定向参数（`redirect_uri`、`next`、`returnUrl`）直接使用
- OAuth 流程中 `redirect_uri` 未与注册回调 URL 严格匹配
- 服务端 302 重定向目标由用户输入控制

## 运行时风险

- 无界循环、递归调用或大型内存缓冲区
- 外部调用缺少超时、重试或速率限制
- 请求路径上的阻塞操作（异步上下文中的同步 I/O）
- 资源耗尽（文件句柄、连接、内存）
- ReDoS（正则表达式拒绝服务）

### ReDoS 危险模式

ReDoS 攻击利用正则表达式的回溯机制，通过精心构造的输入使匹配时间呈指数级增长。

```javascript
// 危险：嵌套量词 - 指数级回溯
const evil1 = /^(a+)+$/           // 输入 "aaaaaaaaaaaaaaaaX" 会卡死
const evil2 = /(a|a)+$/           // 重叠分支 + 量词
const evil3 = /^([a-zA-Z0-9]+)*$/ // 字符类 + 嵌套量词

// 危险：重叠字符类
const evil4 = /^(\w+\s?)*$/       // \w 和 \s 在某些输入下重叠
const evil5 = /^(.*a){10}$/       // .* 贪婪匹配 + 重复

// 危险：常见业务场景中的 ReDoS
const emailRegex = /^([a-zA-Z0-9_\.\-])+\@(([a-zA-Z0-9\-])+\.)+([a-zA-Z0-9]{2,4})+$/
// 输入 "aaaaaaaaaaaaaaaaaaaaaaaa@" 会导致严重回溯

// 推荐：使用原子组或占有量词（如果引擎支持）
// 推荐：限制输入长度后再匹配
// 推荐：使用 safe-regex 或 rxxr2 等工具检测
function safeMatch(pattern: RegExp, input: string, maxLength = 1000): boolean {
  if (input.length > maxLength) {
    return false // 拒绝过长输入
  }
  return pattern.test(input)
}

// 推荐：使用线性时间复杂度的替代方案
// 例如：用 String.prototype.includes() 替代简单的正则匹配
```

**检测工具**：

- `https://github.com/substack/safe-regex` - 检测危险正则
- `https://github.com/superhuman/rxxr2` - 更精确的 ReDoS 检测
- `https://makenowjust-labs.github.io/recheck/` - 在线检测工具

---

## GraphQL 安全

- **深度查询 DoS**：无查询深度限制，攻击者可构造深层嵌套查询消耗服务器资源
- **内省泄露**：生产环境 `__schema` 和 `__type` 未禁用，暴露完整 API 结构
- **批量查询绕过**：GraphQL 的别名机制可绕过速率限制（一条请求内多次查询同一字段）
- **字段级权限缺失**：resolver 中未检查用户对特定字段的访问权限
- **查询复杂度限制**：无基于 cost/复杂度的查询分析

```graphql
# 危险：无深度/复杂度限制的批量别名攻击
query {
  a1: users(page: 1) { ...UserFields }
  a2: users(page: 2) { ...UserFields }
  # ...重复 100 次，单次 HTTP 请求执行大量操作
}

# 危险：深层嵌套
query {
  users {
    posts {
      comments {
        author {
          posts {
            comments {
              # ... 无限嵌套
            }
          }
        }
      }
    }
  }
}
```

---

## WebSocket 安全

- **缺少 Origin 检查**：任意来源可建立 WebSocket 连接（跨站 WebSocket 劫持）
- **缺少认证**：WebSocket 连接建立时未验证 Token 或 Session
- **消息注入**：未验证服务端/客户端消息的结构和内容
- **缺少加密**：使用 `ws://` 而非 `wss://`（应强制 TLS）
- **缺少心跳**：无 ping/pong 机制检测死连接，可能导致资源泄露
- **缺少速率限制**：连接数或消息数无限，可能耗尽服务器资源

---

## 加密

- 弱算法（MD5、SHA1 用于安全目的）
- 硬编码的 IV 或盐值
- 使用无认证的加密（ECB 模式，无 HMAC）
- 密钥长度不足

---

## 竞态条件

竞态条件是导致间歇性故障和安全漏洞的微妙 Bug。特别注意以下情况：

### 共享状态访问

- 多个线程/协程/异步任务无同步访问共享变量
- 并发修改的全局状态或单例
- 无适当锁定的延迟初始化（双重检查锁定问题）
- 并发上下文中使用非线程安全的集合

### 先检查后执行（TOCTOU）

- `if (exists) then use` 模式无原子操作
- `if (authorized) then perform` 其中授权可能变化
- 文件存在检查后跟文件操作
- 余额检查后跟扣款（金融操作）
- 库存检查后跟下单

### 数据库并发

- 缺少乐观锁（`version` 列、`updated_at` 检查）
- 缺少悲观锁（`SELECT FOR UPDATE`）
- 无事务隔离的读-改-写
- 计数器增量无原子操作（`UPDATE SET count = count + 1`）
- 并发插入的唯一约束冲突

### 分布式系统

- 共享资源缺少分布式锁
- 领导选举竞态条件
- 缓存失效竞态（写后读到过期数据）
- 无适当排序的事件顺序依赖
- 集群操作中的脑裂场景

### 需要问的问题

- "如果两个请求同时命中这段代码会发生什么？"
- "这个操作是原子的还是可被中断的？"
- "这段代码访问了什么共享状态？"
- "在高并发下这会如何表现？"

---

## 数据完整性

- 缺少事务、部分写入或不一致的状态更新
- 持久化前的弱验证（类型强制问题）
- 可重试操作缺少幂等性
- 并发修改导致的更新丢失

---

## 框架特定安全检查

根据项目技术栈，加载对应的框架特定检查清单：

| 技术栈 | 参考文件 |
| ------ | -------- |
| React  | `frameworks/react.md` |
| Vue    | `frameworks/vue.md` |
