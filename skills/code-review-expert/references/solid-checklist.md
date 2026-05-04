# SOLID & 架构异味检测信号

## 前端 SRP（单一职责）
- 页面文件同时包含 UI + 数据请求 + 状态管理 + 工具函数
- 组件既处理展示又处理业务逻辑（容器/展示未分离）
- 单个 Hook/Composable 承担 3+ 种职责

## 前端 OCP（开闭原则）
- 添加新行为需修改路由/核心逻辑中的 if-else/switch
- 表单验证规则硬编码在组件中而非可配置
- 主题/样式切换需修改多处代码

## 前端 LSP（里氏替换）
- 子组件忽略或不实现父组件的关键 Props
- 扩展基础 Service 但改变了返回数据结构
- 重写生命周期/方法但不调用 super

## 前端 ISP（接口隔离）
- 组件 Props 定义庞大（10+ 属性），大部分可选且极少使用
- Context/Store 暴露过多状态，消费者仅需其中一小部分
- 工具模块导出 20+ 函数，单文件仅用 2-3 个

## 前端 DIP（依赖倒置）
- 页面/组件直接调用 fetch/axios 绕过 Service 层
- 组件直接依赖具体的存储实现（localStorage/sessionStorage）
- 业务逻辑直接依赖特定 UI 框架 API

## 通用代码异味
- 函数 >30 行，多层嵌套
- 方法使用其他类的数据多于自身（特性嫉妒）
- 相同参数组重复传递（数据泥团）
- 一个变更需编辑多个文件（霰弹式修改）
- 一个文件因多个不相关原因变更（发散式变化）
- 死代码：不可达、从未调用、已关闭的 Feature Flag
- 为假设的未来需求创建抽象（投机性泛化）
- 硬编码值无命名常量（魔法数字/字符串）

## 前端特定异味
- Props 钻取超过 3 层
- 巨型组件 >500 行
- 样式泄露影响全局/父组件
- 未处理组件卸载后的异步回调（竞态）
- 高频状态更新未使用批量/路径更新

## 重构启发式
- 等待第二个用例再抽象（Rule of Three）
- 组合优于继承
- 按职责拆分，非按文件大小
- 先隔离行为再移动（增量重构）
- 用类型系统使非法状态不可表示

---

## 重构修复模式

自动修复时套用以下标准化重构模式。

### 组件拆分（SRP）

```typescript
// 修复巨型组件：分离容器/展示组件 + 自定义 Hook
// 之前：单文件 500+ 行混合 UI/逻辑/数据
function UserDashboard({ userId }) {
  const [user, setUser] = useState(null)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState(null)

  useEffect(() => {
    const controller = new AbortController()
    setLoading(true)
    fetchUser(userId, { signal: controller.signal })
      .then(setUser)
      .catch((err) => {
        if (err.name !== 'AbortError') setError(err)
      })
      .finally(() => setLoading(false))
    return () => controller.abort()
  }, [userId])
  // ... 大量 JSX + 其他逻辑
}

// 修复后：提取数据逻辑到 Hook
// hooks/useUser.ts
function useUser(userId: string) {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<Error | null>(null)

  useEffect(() => {
    const controller = new AbortController()
    setLoading(true)
    fetchUser(userId, { signal: controller.signal })
      .then(setUser)
      .catch((err) => {
        if (err.name !== 'AbortError') setError(err)
      })
      .finally(() => setLoading(false))
    return () => controller.abort()
  }, [userId])

  return { user, loading, error }
}

// UserDashboard.tsx — 纯展示
function UserDashboard({ userId }: { userId: string }) {
  const { user, loading, error } = useUser(userId)
  if (loading) return <Skeleton />
  if (error) return <ErrorDisplay error={error} />
  return <UserProfile user={user} />
}
```

### Props 钻取修复（ISP）

```typescript
// 修复 Props 钻取 >3 层：使用 Context 或组合
// 之前：A → B → C → D 层层传递，中间组件不关心但被迫接收
function A() {
  const [theme, setTheme] = useState('light')
  return <B theme={theme} onThemeChange={setTheme} />
}
function B({ theme, onThemeChange }) {
  return <C theme={theme} onThemeChange={onThemeChange} />
}
function C({ theme, onThemeChange }) {
  return <D theme={theme} onThemeChange={onThemeChange} />
}

// 修复后：使用 Context 直接注入，跳过中间层
const ThemeContext = createContext<ThemeValue>({ theme: 'light', setTheme: () => {} })
function A() {
  const [theme, setTheme] = useState('light')
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <B />
    </ThemeContext.Provider>
  )
}
function D() {
  const { theme, setTheme } = useContext(ThemeContext)
  // 直接使用，无需中间层传递
}
```

### 依赖倒置修复（DIP）

```typescript
// 修复直接依赖具体实现：引入 Service 抽象层
// 之前：组件直接调用 fetch
// 修复后：
// services/productService.ts
export const productService = {
  async list(): Promise<Product[]> {
    const res = await fetch('/api/products')
    if (!res.ok) throw new Error('Failed to fetch products')
    return res.json()
  }
}
```

### 魔法数字/字符串修复

```typescript
// 修复硬编码值：提取为命名常量
// 之前：if (user.age > 18 && status === 'active') { ... }
// 修复后：
const MIN_ADULT_AGE = 18
const ACTIVE_STATUS = 'active' as const
if (user.age > MIN_ADULT_AGE && status === ACTIVE_STATUS) { ... }
```

### 重复代码消除

```typescript
// 修复霰弹式修改：提取公共逻辑
// 修复后：
// utils/validators.ts
export const validators = {
  email: (v: string) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v),
  required: (v: unknown) => v !== null && v !== undefined && v !== '',
}
// 各处导入使用，单一变更点
```
