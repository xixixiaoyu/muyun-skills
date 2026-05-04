# 代码质量检测信号

## 错误处理
- 空 catch 块或仅 console.log → **P1** 静默失败
- 异步操作无 .catch() / try-catch → **P1**
- 错误信息向用户暴露堆栈/内部细节 → **P1**
- catch 块过宽（catch Error 基类而非具体类型）→ **P3**

## 性能
- 循环中 await（N+1 查询）→ **P1**
- 独立请求串行 await 而非 Promise.all/allSettled → **P2**
- 无界数组/Map 持续增长无上限 → **P1** 内存泄漏
- 热路径中重复计算无缓存/memoization → **P2**
- 组件卸载后未清理 timer/listener/subscription/worker → **P1**
- 同步 I/O 或重计算阻塞主线程 → **P2**
- 大文件一次性加载到内存而非流式处理 → **P2**

## 边界条件
- 不安全属性访问 `a.b.c` 无 null 保护 → **P1**
- 除法无除零检查 → **P2**
- 数组索引访问无长度校验 → **P2**
- 空数组/空对象未处理 → **P2**
- 0、""、false 被 truthy 检查误排除 → **P2**
- 金额浮点直接运算（应用整数/专用库）→ **P1**
- Off-by-one：分页、切片、循环边界 → **P2**

## 类型安全（TS 项目）
- `as any` 绕过类型检查 → **P2**
- API 响应无类型定义（返回 any）→ **P2**
- 类型守卫逻辑与实际类型不一致 → **P2**
- `as` 断言无运行时验证（如 DOM 元素 / API 响应）→ **P2**
- `unknown` 场景用了 `any` → **P3**

## 测试（如项目含测试文件）
- 新增功能/Bug 修复无对应测试 → **P2**
- 测试依赖 setTimeout 等时序（flaky）→ **P2**
- 测试间共享可变状态 → **P2**
- Mock 过度：测试实现而非行为 → **P3**
- 测试名称不描述行为和条件 → **P3**

## 可访问性 a11y（如含 UI 组件变更）
- div/span 作为可交互元素（应使用 button/a）→ **P2**
- input 无关联 label（for/id 或 wrapping）→ **P2**
- 图标按钮无 aria-label → **P2**
- 颜色作为唯一信息传达方式 → **P2**
- 标题层级跳级（h1→h3 跳过 h2）→ **P3**
- 动态内容更新无 aria-live → **P3**
- 模态框无焦点陷阱 → **P2**

## 国际化 i18n（如含用户可见文案变更）
- 硬编码用户可见文案未用 i18n 函数 → **P2**
- 伪复数 `n > 1 ? 's' : ''` 而非 i18n 库复数支持 → **P2**
- 手写日期/数字格式而非 `Intl` 或 locale 感知库 → **P2**
- 拼接翻译 key 如 `t('error.' + code)` → **P2**
- UI 没有为文本膨胀留空间（英文比中文长 30-50%）→ **P3**

---

## Auto-Fix 模板

以下标准修复模式在自动修复时直接套用，结合具体代码上下文调整。

### 错误处理修复

```typescript
// 修复空 catch：至少记录错误并重新抛出或降级处理
try {
  await riskyOperation()
} catch (error) {
  // 修复：记录错误上下文，避免静默失败
  console.error('[module] operation failed:', error)
  // 根据场景选择：throw error（上层处理）或返回 fallback 值
  return fallbackValue
}

// 修复缺失异步错误处理：为 Promise 添加 .catch()
fetchData(id)
  .then(setData)
  .catch((err) => {
    // 修复：处理异步错误
    console.error('[fetchData] failed for id:', id, err)
    setError(err.message)
  })

// 修复过宽 catch：区分错误类型
try {
  await apiCall()
} catch (error) {
  if (error instanceof ValidationError) {
    // 修复：具体类型处理
    handleValidationError(error)
  } else if (error instanceof NetworkError) {
    handleNetworkError(error)
  } else {
    throw error // 未知错误向上抛出
  }
}
```

### 性能修复

```typescript
// 修复 N+1 查询：使用批量查询
// 之前：for (const id of ids) { await db.find(id) }
// 修复后：
const results = await db.findMany({ where: { id: { in: ids } } })

// 修复串行 await：并行独立请求
const [users, posts, settings] = await Promise.all([
  fetchUsers(),
  fetchPosts(),
  fetchSettings(),
])

// 修复内存泄漏：组件卸载时清理
useEffect(() => {
  const timer = setInterval(() => tick(), 1000)
  const controller = new AbortController()
  fetchData({ signal: controller.signal })
  // 修复：返回 cleanup 函数
  return () => {
    clearInterval(timer)
    controller.abort()
  }
}, [])
```

### 边界条件修复

```typescript
// 修复不安全属性访问：可选链 + 默认值
// 之前：const name = user.profile.name
// 修复后：
const name = user?.profile?.name ?? 'Unknown'

// 修复除零：添加守卫
const average = items.length > 0 ? total / items.length : 0

// 修复数组越界：索引前校验
function getItem(arr: Item[], index: number): Item | undefined {
  if (index < 0 || index >= arr.length) return undefined
  return arr[index]
}

// 修复浮点金额：使用整数（分）或 decimal 库
// 之前：const total = price * quantity
// 修复后：
const totalCents = Math.round(priceCents * quantity) // 整数运算
```

### 类型安全修复（TS 项目）

```typescript
// 修复 as any：使用具体类型或 unknown + 类型守卫
// 之前：const data = response as any
// 修复后：
interface ApiResponse { id: number; name: string }
function isApiResponse(obj: unknown): obj is ApiResponse {
  return typeof obj === 'object' && obj !== null && 'id' in obj && 'name' in obj
}
if (!isApiResponse(data)) throw new Error('Invalid response')

// 修复 { [key: string]: any } 用 Record 替代
// 之前：const config: { [key: string]: any } = {}
// 修复后：
const config: Record<string, unknown> = {}
```
