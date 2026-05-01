# React 特定检查清单

本清单覆盖 React 应用的安全、性能和代码质量问题。

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
