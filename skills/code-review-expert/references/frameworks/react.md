# React 特定检查清单

本清单覆盖 React 应用的安全、性能和代码质量问题。

## 快速索引（Auto-Fix 模板定位）

| 问题类型 | 级别 | 参考章节 |
|---------|------|---------|
| dangerouslySetInnerHTML 无过滤 | P0 | 安全检查 → XSS 风险 |
| 用户输入作为 URL | P1 | 安全检查 → XSS 风险 |
| useEffect 异步无 cleanup | P1 | 竞态条件 → useEffect 异步竞态 |
| setState 非函数式更新 | P1 | 竞态条件 → 状态更新竞态 |
| 每次渲染创建新对象/函数 | P2 | 性能 → 不必要的重渲染 |
| useEffect 依赖缺失/过多 | P1 | 性能 → useEffect 依赖 |
| 列表无 key 或用 index | P2 | 性能 → 列表渲染优化 |
| 组件内混合关注点 | P2 | Hooks 最佳实践 → 自定义 Hook 提取 |
| RSC 中使用 Hooks/事件 | P1 | React Server Components |
| 敏感数据返回客户端 | P0 | RSC → 敏感数据泄露 |
| 异步组件无 Suspense | P2 | Suspense 数据获取 |
| Suspense 无 ErrorBoundary | P2 | 错误边界 |
| 昂贵计算阻塞 UI | P2 | React 18+ 并发特性 |

---

---

## 安全检查

### XSS 风险

```jsx
// 危险：dangerouslySetInnerHTML 未过滤
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// 推荐：使用 DOMPurify 过滤
import DOMPurify from 'dompurify'
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userInput) }} />

// 危险：直接使用用户输入作为 URL
<a href={userInput}>Link</a>

// 推荐：验证 URL
const safeUrl = userInput?.startsWith('http') ? userInput : '#'
<a href={safeUrl}>Link</a>
```

### 敏感数据处理

- **环境变量暴露**：`REACT_APP_*` 前缀的变量会被打包到客户端代码
- **状态中的敏感数据**：React DevTools 可查看所有 state
- **localStorage 存储**：Token 等敏感信息易被 XSS 访问

### 认证/授权

- 前端路由守卫不能替代后端认证
- 敏感操作必须由后端验证权限
- 不使用 URL 参数传递敏感数据
- JWT 不应存储在 localStorage（优先 httpOnly cookie）

---

## 竞态条件

### useEffect 异步竞态

```jsx
// 危险：未处理组件卸载后的状态更新
useEffect(() => {
  fetchUser(id).then((data) => setUser(data))
}, [id])

// 推荐：使用 cleanup flag
useEffect(() => {
  let cancelled = false
  fetchUser(id).then((data) => {
    if (!cancelled) setUser(data)
  })
  return () => {
    cancelled = true
  }
}, [id])

// React 19 推荐：使用 AbortController
useEffect(() => {
  const controller = new AbortController()
  fetchUser(id, { signal: controller.signal })
    .then((data) => setUser(data))
    .catch((err) => {
      if (err.name !== 'AbortError') setError(err)
    })
  return () => controller.abort()
}, [id])
```

### 状态更新竞态

```jsx
// 危险：基于过期 state 更新
const [count, setCount] = useState(0)
const handleClick = () => {
  setCount(count + 1) // 快速点击可能丢失更新
}

// 推荐：使用函数式更新
const handleClick = () => {
  setCount((prev) => prev + 1)
}
```

---

## 性能

### 不必要的重渲染

```jsx
// 危险：每次父组件渲染都创建新对象/函数
<Child style={{ color: 'red' }} onClick={() => handleClick(id)} />

// 推荐：使用 useMemo/useCallback
const style = useMemo(() => ({ color: 'red' }), [])
const onClick = useCallback(() => {
  handleClick(id)
}, [id, handleClick])
<Child style={style} onClick={onClick} />

// 推荐：使用 React.memo 包裹子组件
const Child = React.memo(({ style, onClick }) => {
  return <div style={style} onClick={onClick}>Child</div>
})
```

### useEffect 依赖

```jsx
// 危险：缺失依赖或依赖过多导致无限循环
useEffect(() => {
  fetchData(filters)
}, []) // filters 变化不会重新请求

// 推荐：正确声明依赖
useEffect(() => {
  fetchData(filters)
}, [filters])

// 危险：对象/数组依赖导致每次渲染都执行
useEffect(() => {
  fetchData({ page, size })
}, [{ page, size }]) // 每次都是新对象

// 推荐：使用基本类型依赖
useEffect(() => {
  fetchData({ page, size })
}, [page, size])
```

### 列表渲染优化

```jsx
// 危险：无 key 或使用 index 作为 key
{items.map((item, index) => <Item key={index} {...item} />)}

// 推荐：使用稳定的唯一 key
{items.map((item) => <Item key={item.id} {...item} />)}

// 推荐：大列表使用虚拟化
import { FixedSizeList } from 'react-window'
<FixedSizeList height={600} itemCount={items.length} itemSize={50}>
  {({ index, style }) => <Item style={style} {...items[index]} />}
</FixedSizeList>
```

### 懒加载

```jsx
// 推荐：路由级代码分割
const Dashboard = lazy(() => import('./Dashboard'))
const Settings = lazy(() => import('./Settings'))

<Suspense fallback={<Loading />}>
  <Routes>
    <Route path='/dashboard' element={<Dashboard />} />
    <Route path='/settings' element={<Settings />} />
  </Routes>
</Suspense>
```

---

## Hooks 最佳实践

### 自定义 Hook 提取

```jsx
// 差：组件内混合关注点
function UserProfile({ userId }) {
  const [user, setUser] = useState(null)
  const [loading, setLoading] = useState(false)

  useEffect(() => {
    setLoading(true)
    fetchUser(userId)
      .then(setUser)
      .finally(() => setLoading(false))
  }, [userId])
  // ... 还有很多其他逻辑
}

// 好：提取自定义 Hook
function useUser(userId) {
  const [user, setUser] = useState(null)
  const [loading, setLoading] = useState(false)

  useEffect(() => {
    let cancelled = false
    setLoading(true)
    fetchUser(userId)
      .then((data) => {
        if (!cancelled) setUser(data)
      })
      .finally(() => {
        if (!cancelled) setLoading(false)
      })
    return () => {
      cancelled = true
    }
  }, [userId])

  return { user, loading }
}

function UserProfile({ userId }) {
  const { user, loading } = useUser(userId)
  // 简洁的组件代码
}
```

### Hooks 规则

- 只在组件顶层调用 Hooks，不在条件/循环中
- 自定义 Hook 以 `use` 开头命名
- `useRef` 适合存储不触发渲染的值
- `useReducer` 优于多个 `useState`（当状态互相依赖时）

---

## 错误边界

```jsx
// 推荐：使用错误边界包裹关键区域
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null }

  static getDerivedStateFromError(error) {
    return { hasError: true, error }
  }

  componentDidCatch(error, errorInfo) {
    console.error('ErrorBoundary caught:', error, errorInfo)
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || <div>Something went wrong</div>
    }
    return this.props.children
  }
}

// 使用
<ErrorBoundary fallback={<ErrorUI />}>
  <CriticalSection />
</ErrorBoundary>
```

---

## 常见反模式

| 反模式 | 说明 | 修复 |
|--------|------|------|
| **巨型组件** | 单组件 > 300 行 | 拆分为子组件 + 自定义 Hook |
| **Props 钻取** | 属性传递超过 3 层 | 使用 Context 或组合 |
| **过度使用 Context** | 高频变化导致全局重渲染 | 拆分 Context 或使用状态管理 |
| **useEffect 滥用** | 能用事件处理器的用了 effect | 移到事件处理器中 |
| **未记忆化的回调** | 每次渲染创建新函数传给子组件 | 使用 useCallback |
| **内存泄漏** | 未清理的定时器/订阅 | useEffect cleanup 中清理 |
| **派生状态** | state 可以从 props 计算出来 | 直接在 render 中计算 |
| **索引作 key** | 列表排序/增删时出现问题 | 使用稳定唯一 ID |

---

## React Server Components（RSC）

### 服务端/客户端边界

```jsx
// 危险：服务端组件中使用客户端特性
// server-component.tsx
'use server'
export default function ServerComp() {
  const [state, setState] = useState(null)  // ❌ RSC 中不能使用 Hooks
  useEffect(() => {}, [])                     // ❌ RSC 中不能使用 Effect
  return <div onClick={handler}>...</div>    // ❌ RSC 中不能使用事件处理器
}

// 推荐：明确分离服务端和客户端组件
// server-component.tsx — 纯数据获取和渲染
export default async function ServerComp() {
  const data = await fetchData()
  return <ClientComp initialData={data} />
}

// client-component.tsx
'use client'
export default function ClientComp({ initialData }) {
  const [state, setState] = useState(initialData)
  // ... 交互逻辑
}
```

### 敏感数据泄露

- **服务端组件中暴露密钥**：RSC 中直接 `import` 的 `process.env.X` 不会到客户端，但作为 props 传递的数据会
- **"use client" 边界**：标记为客户端组件后，其子树默认也变为客户端（除非被服务端组件再次包裹）
- **Serialization 陷阱**：RSC props 必须可序列化，函数和 Class 实例无法传递

### Next.js 特定检查

- **`getServerSideProps` / Server Actions 中的密钥**：这些在服务端执行，确保不将敏感数据返回给客户端
- **Route Handlers 的认证**：`route.ts` 中的 API 路由独立于 middleware 配置
- **Middleware 的作用域**：Next.js middleware 在 Edge 运行，不能使用 Node.js API
- **ISR/SSG 缓存**：`revalidate` 配置不当可能导致过期数据或过于频繁的重新生成

---

## React 18+ 并发特性

### useTransition / useDeferredValue

```jsx
// 危险：昂贵更新阻塞用户交互
const [query, setQuery] = useState('')
const filteredList = list.filter(item => item.name.includes(query))
// 用户输入时列表过滤卡顿

// 推荐：将非紧急更新标记为 transition
const [query, setQuery] = useState('')
const [isPending, startTransition] = useTransition()

const handleChange = (e) => {
  setInputValue(e.target.value)  // 紧急：更新输入框
  startTransition(() => {
    setQuery(e.target.value)     // 非紧急：可中断的列表过滤
  })
}

// 或使用 useDeferredValue
const deferredQuery = useDeferredValue(query)
const filteredList = useMemo(
  () => list.filter(item => item.name.includes(deferredQuery)),
  [deferredQuery]
)
```

### 并发特性检查点

- [ ] 复杂列表过滤时是否使用 `useDeferredValue` 避免输入卡顿？
- [ ] Tab/路由切换是否使用 `useTransition` 保持 UI 响应？
- [ ] 是否错误地在 transition 中使用了 `useEffect` 等待状态变化？（应直接使用 `isPending`）
- [ ] `Suspense` 的 fallback 是否合理（不是空白页）？

---

## Suspense 数据获取

```jsx
// 推荐：Suspense + React 19 use() hook
function UserProfile({ userId }) {
  return (
    <Suspense fallback={<ProfileSkeleton />}>
      <UserData userId={userId} />
    </Suspense>
  )
}

async function UserData({ userId }) {
  const user = await fetchUser(userId)  // RSC 中直接 await
  return <div>{user.name}</div>
}

// 推荐：ErrorBoundary + Suspense 组合
<Suspense fallback={<Loading />}>
  <ErrorBoundary fallback={<ErrorUI />}>
    <AsyncComponent />
  </ErrorBoundary>
</Suspense>
```

### Suspense 数据获取检查点

- [ ] 异步组件外层是否包裹了 `<Suspense>` 提供加载状态？
- [ ] 是否搭配了 `<ErrorBoundary>` 处理加载失败？
- [ ] 多个独立数据源是否使用独立 Suspense 边界（避免瀑布式等待）？
- [ ] Suspense 嵌套层级是否合理（不过浅导致全屏 loading，不过深导致 UI 跳动）？
