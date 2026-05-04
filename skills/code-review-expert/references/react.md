# React 特定检测信号

## 竞态条件
- useEffect 异步无 cleanup（cancelled flag / AbortController）→ **P1**
- setState(count + 1) 而非 setState(prev => prev + 1) → **P1** 快速点击丢失更新
- 多个独立 useEffect 之间存在隐式依赖序 → **P2**

## 性能
- JSX 中每次渲染创建新对象/函数传给 memo 子组件 → **P2**
- useEffect 依赖含对象/数组（每次渲染都是新引用导致循环触发）→ **P2**
- Context value 高频变化导致全局重渲染 → **P2**
- 大列表无虚拟化且用 index 作 key（排序/增删时 Bug）→ **P2**

## Hooks 陷阱
- Hook 在条件/循环中调用 → **P1**
- useEffect 缺少依赖或依赖过多 → **P1**
- 能用事件处理器却用了 useEffect（如 onSubmit 中 fetch 而非 useEffect 监听 submit）→ **P2**
- 派生状态：state 可从 props 直接计算出来 → **P3**

## RSC / Next.js
- `'use server'` 组件中使用 Hooks / 事件处理器 → **P1**
- RSC props 传递不可序列化对象（函数/Class 实例）→ **P1**
- Server Action / getServerSideProps 中敏感数据返回给客户端 → **P0**
- Next.js middleware 中使用 Node.js API（Edge 不可用）→ **P1**
- `runtimeConfig.public` 中暴露密钥 → **P0**
- ISR `revalidate` 配置不当（过期数据 / 过于频繁重新生成）→ **P2**

## Suspense / 并发
- 异步组件无 Suspense 包裹 → **P2**
- Suspense 无 ErrorBoundary 搭配 → **P2**
- 多个独立数据源用单个 Suspense 导致瀑布式等待 → **P2**
- 昂贵计算阻塞输入 → 未使用 useDeferredValue / useTransition → **P2**

## 常见反模式信号
- 单组件 >300 行
- Props 传递 >3 层
- useEffect 中修改全局状态（无隔离）
- 未清理 timer/subscription/worker（内存泄漏）
- 索引作 key（排序/增删场景）
