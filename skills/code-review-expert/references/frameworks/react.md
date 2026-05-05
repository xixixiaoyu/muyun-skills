# React 特定检查清单

本清单覆盖 React 应用的安全、性能和代码质量问题。

## 快速索引

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
| Hook 在条件/循环中调用 | P1 | Hooks 最佳实践 → Hooks 规则 |
| 能用事件处理器却用了 useEffect | P2 | 常见反模式 |
| 派生状态（state 可从 props 计算） | P3 | 常见反模式 |
| 多个独立 useEffect 隐式依赖序 | P2 | 竞态条件 |
| Context value 高频变化重渲染 | P2 | 常见反模式 |
| RSC props 传递不可序列化对象 | P1 | React Server Components |
| Next.js middleware 使用 Node.js API | P1 | RSC → Next.js 特定检查 |
| ISR revalidate 配置不当 | P2 | RSC → Next.js 特定检查 |
| 多个独立数据源用单个 Suspense | P2 | Suspense 数据获取 |

---

## 安全检查

### XSS 风险

- `dangerouslySetInnerHTML` 未经过滤直接使用用户输入 → **P0**，必须用 DOMPurify 等库净化
- 用户输入直接作为 `href`/`src` 等属性值 → **P1**，需验证 URL 协议和来源
- `a` 标签 `href` 绑定用户可控数据未校验 → **P1**，需白名单或协议校验

### 敏感数据处理

- `REACT_APP_*` 前缀的变量会被打包到客户端代码，不能存放密钥
- React DevTools 可查看所有 state，敏感数据不应放入组件状态
- Token 等敏感信息不应存储在 localStorage（易被 XSS 访问），优先 httpOnly cookie

### 认证/授权

- 前端路由守卫不能替代后端认证，后端 API 必须有独立认证
- 敏感操作必须由后端验证权限，不能信任前端传入的角色/权限标志
- 不使用 URL 参数传递敏感数据
- JWT 不应存储在 localStorage，优先 httpOnly cookie

---

## 竞态条件

### useEffect 异步竞态

- **问题**：useEffect 中发起异步请求，组件卸载后回调仍可能执行 setState，导致内存泄漏和控制台警告
- **检测信号**：useEffect 内有 fetch/异步调用但无 cleanup 或 AbortController
- **修复方向**：使用 AbortController 在 cleanup 中取消请求，或使用 cancelled flag 忽略卸载后的更新

### 状态更新竞态

- **问题**：基于当前 state 值计算新 state（如 `setCount(count + 1)`），快速连续调用时可能丢失更新
- **检测信号**：setState 调用中使用 state 变量而非函数式更新
- **修复方向**：使用函数式更新 `setCount(prev => prev + 1)` 确保基于最新值

---

## 性能

### 不必要的重渲染

- **问题**：每次渲染创建新对象/函数作为 props 传给子组件，导致 React.memo 失效
- **检测信号**：JSX 中内联对象 `style={{...}}` 或内联箭头函数 `onClick={() => ...}`
- **修复方向**：使用 useMemo/useCallback 稳定引用，或用 React.memo 包裹子组件

### useEffect 依赖

- **问题**：依赖数组缺失导致闭包过期；对象/数组作为依赖导致每次渲染都执行
- **检测信号**：useEffect 内部使用了外部变量但依赖数组为空或不完整；依赖数组含对象字面量
- **修复方向**：正确声明所有依赖；用基本类型值作为依赖而非对象引用

### 列表渲染优化

- **问题**：列表项无 key 或使用 index 作为 key，增删排序时渲染错乱
- **检测信号**：`.map()` 返回的 JSX 无 key 属性或 key={index}
- **修复方向**：使用稳定唯一的业务 ID 作为 key；大列表（>100 项）考虑虚拟滚动

### 懒加载

- 路由级组件使用 `React.lazy()` + `Suspense` 实现代码分割
- 非首屏重型组件延迟加载，减少初始 bundle 大小

---

## Hooks 最佳实践

### 自定义 Hook 提取

- **问题**：组件内同时包含 UI 渲染、数据请求、状态管理、业务逻辑
- **检测信号**：组件函数超过 150 行，useEffect 超过 2 个，useState 超过 3 个
- **修复方向**：将数据获取和状态逻辑提取为自定义 Hook（`useXxx`），组件只负责渲染

### Hooks 规则

- 只在组件顶层调用 Hooks，不在条件/循环/return 之后调用
- 自定义 Hook 以 `use` 开头命名
- `useRef` 适合存储不触发渲染的值（DOM 引用、interval ID 等）
- `useReducer` 优于多个 `useState`（当状态互相依赖或有复杂更新逻辑时）

---

## 错误边界

- 关键业务区域（支付、数据提交、第三方集成）应包裹 ErrorBoundary
- ErrorBoundary 需要实现 `getDerivedStateFromError` 和/或 `componentDidCatch`
- fallback UI 应提供有意义的错误提示，不是空白页
- ErrorBoundary 无法捕获：事件处理器中的错误、异步代码（setTimeout/promise）、SSR 中的错误、ErrorBoundary 自身的错误

---

## 常见反模式

| 反模式 | 说明 | 修复方向 |
|--------|------|----------|
| **巨型组件** | 单组件 > 300 行 | 拆分为子组件 + 自定义 Hook |
| **Props 钻取** | 属性传递超过 3 层 | 使用 Context 或组合模式 |
| **过度使用 Context** | 高频变化导致全局重渲染 | 拆分 Context 或使用状态管理库 |
| **useEffect 滥用** | 能用事件处理器的用了 effect | 移到事件处理器中 |
| **未记忆化的回调** | 每次渲染创建新函数传给子组件 | 使用 useCallback |
| **内存泄漏** | 未清理的定时器/订阅/请求 | useEffect cleanup 中清理 |
| **派生状态** | state 可以从 props 计算出来 | 直接在 render 中计算，不另设 state |
| **索引作 key** | 列表排序/增删时出现问题 | 使用稳定唯一 ID |

---

## React Server Components（RSC）

### 服务端/客户端边界

- **问题**：在 RSC（服务端组件）中使用 useState/useEffect/事件处理器等客户端特性会报错
- **检测信号**：`'use server'` 文件中出现 Hooks 调用或事件绑定
- **修复方向**：明确分离服务端组件（纯数据获取和渲染）与客户端组件（`'use client'`），前者将数据通过 props 传给后者

### 敏感数据泄露

- RSC 中直接 `import` 的环境变量不会到客户端，但作为 props 传递的数据会序列化到客户端
- 标记为 `'use client'` 的组件，其子树默认也变为客户端（除非被服务端组件再次包裹）
- RSC props 必须可序列化（JSON-safe），函数和 Class 实例无法传递

### Next.js 特定检查

- `getServerSideProps` / Server Actions 中的密钥不能在返回的 props 中暴露给客户端
- `route.ts` 中的 API 路由需要独立认证，不依赖 middleware 的前端守卫
- Next.js middleware 在 Edge 环境运行，不能使用 Node.js 专用 API
- ISR/SSG 的 `revalidate` 配置不当可能导致过期数据或过于频繁的重新生成

---

## React 18+ 并发特性

### useTransition / useDeferredValue

- **问题**：昂贵的计算（大列表过滤、搜索）阻塞用户输入导致 UI 卡顿
- **检测信号**：input onChange 中直接执行重计算或大量 setState
- **修复方向**：将非紧急更新包裹在 `startTransition` 中，或使用 `useDeferredValue` 延迟非紧急值

### 并发特性检查点

- 复杂列表过滤时是否使用 `useDeferredValue` 避免输入卡顿？
- Tab/路由切换是否使用 `useTransition` 保持 UI 响应？
- 是否错误地在 transition 中使用了 `useEffect` 等待状态变化？
- `Suspense` 的 fallback 是否合理（不是空白页）？

---

## Suspense 数据获取

- 异步组件外层必须包裹 `<Suspense>` 提供加载状态
- 搭配 `<ErrorBoundary>` 处理加载失败的场景
- 多个独立数据源应使用独立 Suspense 边界，避免瀑布式等待
- Suspense 嵌套层级应合理：不过浅导致全屏 loading，不过深导致多处 UI 跳动
