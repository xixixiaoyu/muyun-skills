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

## SQL 注入
- 字符串拼接/模板字面量/格式化构造 SQL → **P0**
- ORM 原生查询/动态查询未使用参数绑定（如 JPA native query、Django raw、Go fmt.Sprintf）→ **P0**
- 存储过程参数拼接用户输入 → **P0**
- 动态表名/列名/ORDER BY/GROUP BY 未白名单校验 → **P1**
- LIKE 查询用户输入未转义 `%` `_` 通配符 → **P2**

## NoSQL 注入
- MongoDB $where/`$expr` 使用用户输入 → **P0**
- 聚合管道中直接插入用户输入 → **P0**
- 用户输入被解析为查询操作符（如 `{ "$gt": "" }` 绕过权限）→ **P0**
- Redis 命令拼接用户输入 → **P1**

## 命令注入
- exec/system/popen/subprocess/os.exec 等系统调用拼接用户输入 → **P0**
- shell 元字符未过滤（`, $, |, &, ;, \n, \r`）→ **P0**
- 动态命令名或参数来自用户输入 → **P0**

## SSRF（服务端请求伪造）
- 用户输入直接作为 HTTP 请求 URL → **P0**
- URL 解析后未校验 IP 指向内网地址（127.0.0.1/10.0.0.0/8/172.16.0.0/12/192.168.0.0/16）→ **P0**
- 未限制请求协议（允许 file:/// dict:/// gopher://）→ **P0**
- 未防御 DNS 重绑定攻击（每次 DNS 解析后未重新校验 IP）→ **P1**

## 反序列化安全
- Java ObjectInputStream 反序列化不可信数据 → **P0**
- Python pickle.loads/cPickle 反序列化不可信数据 → **P0**
- PHP unserialize 反序列化不可信数据 → **P0**
- JavaScript eval/Function 执行不可信字符串 → **P0**
- YAML.load/YAML.full_load 替代 YAML.safe_load → **P1**

## SSTI（服务端模板注入）
- 用户输入拼入模板字符串或传入模板引擎渲染（Jinja2/EJS/Thymeleaf/FreeMarker/Pug）→ **P0**

## XXE（XML 外部实体注入）
- XML 解析器未禁用外部实体和 DTD 处理 → **P0**

## Cookie 安全
- 敏感 Cookie 缺失 Secure 标志（仅 HTTPS 传输）→ **P1**
- 缺失 HttpOnly 标志（阻止 JavaScript 访问）→ **P1**
- 缺失 SameSite 或设为 None 但无 Secure → **P1**
- Cookie Domain 设置过宽（如 `.example.com` 影响所有子域）→ **P2**
- 敏感 Cookie 未设置 `__Host-` 前缀（防子域攻击）→ **P2**

## CORS 配置
- `Access-Control-Allow-Origin` 设为 `*` 且同时允许携带凭证（Credentials）→ **P0**
- 动态反射 Origin 请求头值未做白名单校验 → **P1**
- `Access-Control-Allow-Methods` 设置为 `*` 或过于宽松 → **P2**

---

## 修复指引

自动修复时根据问题类型套用标准化修复模式，注意以下不可自动修复的场景需标注 ⚠需手动处理：

- **代码级可自动修复**：XSS 过滤、URL 白名单校验、认证守卫添加、租户/所有权检查、竞态条件（AbortController / 乐观锁 / 原子操作）、密钥环境变量化、日志脱敏、JWT 算法指定、正则简化
- **配置/基础设施级需手动处理**：CSP/安全响应头配置、CSRF Token 机制、GraphQL 查询限制、WebSocket Origin 检查、加密算法替换、供应链依赖版本锁定、文件上传服务端校验

修复时遵循：安全第一、最小化变更、保留业务意图、每项修复独立可逆。
