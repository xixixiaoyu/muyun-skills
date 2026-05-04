# 安全检测信号

## 输入/输出
- innerHTML / dangerouslySetInnerHTML / v-html 无 DOMPurify → **P0 XSS**
- URL/路径拼接用户输入 → **P0 注入**
- 用户可控的 redirect_uri / next / returnUrl 未白名单校验 → **P0 开放重定向**
- 文件上传无类型/大小限制、MIME 未验证、文件名可由用户控制 → **P1**
- 用户输入参与文件路径拼接（`../` 遍历）→ **P0 路径遍历**

## 认证/授权
- 新端点/路由缺少认证守卫 → **P0**
- 读写操作未检查租户/所有权（IDOR）→ **P0**
- 信任客户端传入的角色/权限标志/ID → **P0**
- 仅前端路由守卫保护（后端 API 无独立认证）→ **P0**

## 原型污染（JS 特定）
- Object.assign / 展开运算符合并用户输入到配置对象 → **P0**

## JWT
- 弱密钥、硬编码密钥、未验证 exp → **P0**
- 算法混淆（期望 RS256 时接受 none 或 HS256）→ **P0**
- JWT payload 含敏感数据（base64 非加密）→ **P0**
- 未验证 iss/aud → **P2**

## 密钥管理
- 代码/配置/日志中含 API Key / Token / 凭证 → **P0**
- .env / 密钥文件被提交到 Git → **P0**
- 敏感数据通过 RSC/SSR props 从服务端传到客户端 → **P0**
- 错误消息中泄露内部细节/堆栈 → **P1**

## 供应链
- 依赖版本未固定（`^` 而非精确版本）→ **P2**
- 从非可信 CDN 加载资源无 integrity hash → **P1**
- 存在已知 CVE 的过时依赖 → **P1**

## CSP & 安全响应头
- CSP 含 `unsafe-inline` / `unsafe-eval` → **P1**（优先 nonce/hash）
- `script-src` 含 `*` 或不可信域 → **P1**
- 缺少 `object-src 'none'` / `base-uri` 限制 → **P2**
- 缺少 `X-Frame-Options` / `frame-ancestors` → **P1**
- 缺少 `X-Content-Type-Options: nosniff` / HSTS → **P2**

## CSRF
- 状态变更操作（POST/PUT/DELETE）无 CSRF Token 或 SameSite Cookie → **P1**
- GET 请求执行状态变更 → **P1**
- SPA 中 Token 通过 Cookie 自动附带（应使用 Authorization Header + 非 Cookie 存储）→ **P1**

## ReDoS
- 正则含嵌套量词 `(a+)+` 或重叠分支 `(a|a)+` → **P1**
- 用户输入直接传入正则且无长度限制 → **P1**

## GraphQL
- 生产环境未禁用 `__schema` / `__type`（内省泄露）→ **P1**
- 无查询深度/复杂度限制 → **P1**（防嵌套 DoS / 批量别名攻击）
- Resolver 未做字段级权限检查 → **P1**

## WebSocket
- 连接建立时无 Origin 检查（跨站 WebSocket 劫持）→ **P1**
- 使用 `ws://` 而非 `wss://` → **P1**
- 无心跳/超时机制 → **P2**

## 加密
- 使用 MD5/SHA1 于安全目的 → **P1**
- 硬编码 IV/盐值 → **P0**
- ECB 模式 / 无 HMAC 的加密 → **P1**

## 竞态条件
- 并发修改共享状态无同步机制 → **P0**
- TOCTOU：`if (exists) then use` 无原子操作（尤其金融/库存）→ **P0**
- 数据库读-改-写无乐观锁（version 列）或悲观锁（SELECT FOR UPDATE）→ **P1**
- 缓存失效竞态（写后读到过期数据）→ **P1**
- 计数器增量非原子（`count = count + 1` 而非 `SET count = count + 1`）→ **P1**
- 前端异步请求返回顺序不确定导致过期数据覆盖 → **P1**

## 数据完整性
- 部分写入无事务保护 → **P1**
- 可重试操作缺少幂等性 → **P1**
- 并发修改导致更新丢失 → **P1**

---

## Auto-Fix 模板

以下标准修复模式在自动修复时直接套用。

### XSS / 注入修复

```typescript
// 修复 dangerouslySetInnerHTML / innerHTML：使用 DOMPurify
import DOMPurify from 'dompurify'
// 之前：<div dangerouslySetInnerHTML={{ __html: userInput }} />
// 修复后：
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userInput) }} />

// 修复 v-html：使用 DOMPurify + computed
// 之前：<div v-html="userInput"></div>
// 修复后：<div v-html="sanitizedHtml"></div>
// script: const sanitizedHtml = computed(() => DOMPurify.sanitize(userInput.value))

// 修复 URL 注入：白名单校验
// 之前：window.location.href = userInput
// 修复后：
const ALLOWED_REDIRECTS = ['/dashboard', '/profile', '/settings']
const safeUrl = ALLOWED_REDIRECTS.includes(userInput) ? userInput : '/'
window.location.href = safeUrl

// 修复 :href 绑定用户输入：验证协议
// 之前：<a :href="userLink">
// 修复后：
const safeHref = computed(() => {
  if (!userLink.value) return '#'
  return /^https?:\/\//.test(userLink.value) ? userLink.value : '#'
})
```

### 认证/授权修复

```typescript
// 修复缺失认证守卫：添加中间件
// 之前：export default function handler(req, res) { ... }
// 修复后：
export default async function handler(req, res) {
  const session = await getSession(req)
  if (!session) return res.status(401).json({ error: 'Unauthorized' })
  // ... 原有逻辑
}

// 修复 IDOR：添加租户/所有权检查
// 之前：const item = await db.find({ id: req.params.id })
// 修复后：
const item = await db.find({ id: req.params.id, tenantId: session.tenantId })
if (!item) return res.status(404).json({ error: 'Not found' })
```

### 竞态条件修复

```typescript
// 修复请求竞态（React）：使用 AbortController
useEffect(() => {
  const controller = new AbortController()
  async function load() {
    try {
      const data = await fetchUser(id, { signal: controller.signal })
      setUser(data)
    } catch (err) {
      if (err.name !== 'AbortError') setError(err)
    }
  }
  load()
  return () => controller.abort()
}, [id])

// 修复请求竞态（Vue）：使用 onWatcherCleanup
watch(source, async (newVal) => {
  const controller = new AbortController()
  onWatcherCleanup(() => controller.abort())
  const data = await fetchData(newVal, { signal: controller.signal })
  result.value = data
})

// 修复数据库竞态：乐观锁
// 之前：await db.update({ id, data })
// 修复后：
const result = await db.update({
  where: { id, version: currentVersion },
  data: { ...data, version: currentVersion + 1 },
})
if (result.count === 0) throw new ConflictError('数据已被修改')

// 修复原子计数器：使用数据库原子操作
// 之前：const count = await db.get(); await db.set(count + 1)
// 修复后：
await db.increment({ where: { id }, data: { count: 1 } })
```

### 密钥泄露修复

```typescript
// 修复硬编码密钥：使用环境变量
// 之前：const API_KEY = 'sk-abc123xyz'
// 修复后：
const API_KEY = process.env.API_KEY
if (!API_KEY) throw new Error('API_KEY environment variable is required')

// 修复日志泄露敏感数据：过滤后再记录
function safeLog(data: Record<string, unknown>) {
  const { password, token, secret, ...safe } = data
  console.log(safe)
}
```

### JWT 修复

```typescript
// 修复算法混淆：指定算法白名单
import jwt from 'jsonwebtoken'
// 之前：jwt.verify(token, secret)
// 修复后：
jwt.verify(token, secret, { algorithms: ['RS256'] })

// 修复 payload 含敏感数据：只存最小必要信息
// 之前：{ userId, email, password, ssn, ... }
// 修复后：{ sub: userId, role: user.role }
```

### ReDoS 修复

```typescript
// 修复危险正则：消除嵌套量词 + 限制输入长度
// 之前：/^(a+)+$/.test(userInput)
// 修复后：
const MAX_INPUT = 1000
if (userInput.length > MAX_INPUT) return false
/^a+$/.test(userInput) // 简化正则，消除嵌套量词
```
