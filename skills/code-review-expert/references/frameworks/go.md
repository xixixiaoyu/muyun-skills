# Go 后端检查清单

本清单覆盖 Go 应用的安全、并发、性能和代码质量问题。

## 快速索引

| 问题类型 | 级别 | 参考章节 |
|---------|------|---------|
| SQL 注入（fmt.Sprintf 拼接） | P0 | 安全检查 |
| 模板注入（text/template 渲染用户输入） | P0 | 安全检查 |
| 路径遍历（path.Clean 绕过） | P0 | 安全检查 |
| math/rand 用于安全目的 | P1 | 安全检查 |
| goroutine 泄漏（无退出机制） | P1 | 并发 |
| channel 死锁/阻塞 | P1 | 并发 |
| defer 在循环中积累 | P2 | 并发/性能 |
| sync.Mutex 按值拷贝 | P1 | 并发 |
| context 未传递/未检查取消 | P1 | 并发 |
| error 被忽略（`_` 丢弃） | P1 | 代码质量 |
| panic 用于常规错误处理 | P2 | 代码质量 |
| HTTP client 无超时 | P1 | 常见反模式 |
| response.Body 未关闭 | P1 | 常见反模式 |
| string 拼接用 + 而非 Builder | P2 | 性能 |
| slice append 未预分配容量 | P2 | 性能 |
| nil pointer dereference 未检查 | P1 | 代码质量 |
| init() 滥用 | P2 | 代码质量 |
| 全局变量用于测试状态 | P3 | 常见反模式 |
| time.Tick 泄漏 | P1 | 常见反模式 |

---

## 安全检查

### SQL 注入

- 使用 `fmt.Sprintf` 拼接用户输入构造 SQL → **P0**，必须用参数化查询（`$1, $2` 占位符）
- ORM 原生查询（如 GORM `db.Raw`）未使用参数绑定 → **P0**
- 动态表名/列名/ORDER BY 来自用户输入且未白名单校验 → **P1**

### 模板注入

- `text/template` 渲染用户输入可能产生注入 → **P0**，应优先用 `html/template` 自动转义
- 模板内容从用户输入动态加载 → **P0**

### 路径遍历

- 文件路径拼接用户输入，`path.Clean` 不足以防 `../` 遍历 → **P0**，需用 `filepath.Abs` + 前缀校验
- 静态文件服务目录未限制在指定根目录内 → **P1**

### 加密安全

- 使用 `math/rand` 生成 token/session ID 等安全敏感值（应用 `crypto/rand`）→ **P1**
- 使用 MD5/SHA1 用于密码哈希 → **P1**
- 硬编码密钥/盐值 → **P0**

---

## 并发

### goroutine 泄漏

- **问题**：goroutine 启动后无退出机制，channel 永远阻塞或无限循环
- **检测信号**：`go func()` 内无 context 取消检查、无 done channel、无 timeout
- **修复方向**：传递 context，select 监听 `ctx.Done()`；设置超时退出

### channel 死锁/阻塞

- **问题**：无缓冲 channel 的发送和接收在同一个 goroutine 导致死锁；向已关闭 channel 发送导致 panic
- **检测信号**：无缓冲 `chan` + 同步 send/receive；缺少 `close` 保护
- **修复方向**：send 和 receive 分在不同 goroutine；用 select + default 避免阻塞

### sync 原语误用

- `sync.Mutex` 按值拷贝（应指针传递）→ **P1**
- `sync.WaitGroup.Add` 在 goroutine 内部而非外部调用 → **P1**
- `defer mu.Unlock()` 后还有 return，锁范围过大 → **P2**

### context 使用

- HTTP handler 未将 `r.Context()` 传递给下游调用 → **P1**
- 创建 context 未设置超时（`context.Background()` 直接用于外部调用）→ **P1**
- goroutine 内忽略 context 取消信号 → **P1**

---

## 性能

### 内存分配

- slice 循环 append 未预分配容量 → **P2**（频繁扩容导致内存拷贝）
- map 无界增长无上限 → **P1**（内存泄漏）
- `defer` 在热路径循环中使用 → **P3**（每次迭代有开销）

### 字符串操作

- 大量字符串拼接使用 `+` 而非 `strings.Builder` → **P2**
- `fmt.Sprintf` 用于简单字符串拼接 → **P3**
- `string([]byte)` 转换在热路径 → **P2**

### I/O 操作

- 同步 I/O 阻塞 goroutine（应用异步或 worker pool）→ **P2**
- 大文件用 `ioutil.ReadAll`/`os.ReadFile` 一次性读入内存 → **P1**，应用流式处理

---

## 代码质量

### 错误处理

- error 返回值被忽略（`_ = fn()` 或 `fn()` 无接收）→ **P1**
- `if err != nil` 后仅 log 然后继续执行（应 return error）→ **P1**
- 错误信息未提供足够上下文（`errors.New("failed")` 而非 `fmt.Errorf("xxx: %w", err)`）→ **P3**
- panic 用于常规业务流程控制（应返回 error）→ **P2**
- recover 后未充分记录堆栈信息 → **P2**

### 接口设计

- interface 过大（>5 个方法），违反接口隔离 → **P2**
- 函数接受具体类型而非 interface（降低可测试性）→ **P3**
- 导出不必要的内部类型/函数 → **P2**

### nil 安全

- 未检查指针/map/slice/interface 的 nil 就直接使用 → **P1**
- 对 nil map 写入会 panic → **P1**

---

## 常见反模式

| 反模式 | 说明 | 修复方向 |
|--------|------|----------|
| **init() 滥用** | 多个 init 函数隐式依赖顺序，难以测试 | 显式初始化函数替代 |
| **全局变量** | 包级变量用于测试状态共享 | 依赖注入替代全局状态 |
| **HTTP client 无超时** | `http.DefaultClient` 无超时设置 | 自定义 Client 设置 Timeout |
| **response.Body 未关闭** | HTTP 响应 body 未 close 导致连接泄漏 | defer resp.Body.Close() |
| **time.Tick 泄漏** | Tick 返回的 channel 无法被 GC | 用 time.NewTicker + Stop |
| **goroutine 无 recover** | panic 导致整个程序崩溃 | goroutine 内加 defer recover |
| **interface{} 过度使用** | 丢失类型安全（Go 1.18+ 应优先泛型） | 泛型或具体类型替代 |
