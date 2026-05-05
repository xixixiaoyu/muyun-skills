# Vue 特定检查清单

本清单覆盖 Vue 2/3 应用的安全、性能和代码质量问题。

## 快速索引

| 问题类型 | 级别 | 参考章节 |
|---------|------|---------|
| v-html 无 DOMPurify | P0 | 安全检查 → XSS 风险 |
| :href 绑定用户可控 URL | P1 | 安全检查 → XSS 风险 |
| watch 异步无 onWatcherCleanup | P1 | 竞态条件 → 异步请求竞态 |
| 多个 watch 循环依赖 | P1 | 竞态条件 → watch vs watchEffect |
| reactive 包裹巨型对象 | P2 | 性能 → 响应式系统 |
| 模板复杂计算未用 computed | P2 | 性能 → 响应式系统 |
| v-if 和 v-for 同级 | P1 | 性能 → 组件渲染优化 |
| v-for 无 key 或用 index | P2 | 性能 → 组件渲染优化 |
| 过长 script setup 未提取 composable | P2 | 代码质量 → Composition API |
| Vue 2 新增属性不响应 | P1 | Vue 2 特定问题 → 响应式陷阱 |
| Teleport 含 v-html | P0 | Teleport 安全 |
| Nuxt public runtimeConfig 暴露密钥 | P0 | Nuxt.js 特定检查 |
| useCookie 敏感数据无 httpOnly | P1 | Nuxt.js 特定检查 |
| defineProps/defineEmits 无类型声明 | P3 | 代码质量 → Props/Emits 定义 |
| 直接修改 defineModel 返回值 | P2 | defineModel |
| 直接修改 props | P1 | 常见反模式 |
| watch 中修改被监听值 | P1 | 常见反模式 |
| provide/inject 过度使用 | P2 | 常见反模式 |
| $refs 直接操作 DOM | P2 | 常见反模式 |
| 异步请求未用 AbortController | P1 | 竞态条件 |
| Teleport 未受控 DOM 目标 | P2 | Teleport 安全 |
| 非异步 setup 组件使用 Suspense | P3 | Suspense |

---

## 安全检查

### XSS 风险

- `v-html` 未经过滤直接使用用户输入 → **P0**，必须用 DOMPurify 等库净化后再绑定
- `:href` 等动态属性绑定用户可控 URL 未校验 → **P1**，需验证协议（只允许 http/https）
- 用户输入参与动态组件名或插槽名拼接 → **P1**

### 敏感数据处理

- Vue DevTools 可查看所有响应式数据（ref/reactive），敏感数据不应放入响应式系统
- Pinia/Vuex 全局状态不应存储 Token 等敏感信息
- Token 等敏感信息不应存储在 localStorage（易被 XSS 访问）

### 认证/授权

- 前端路由守卫（`router.beforeEach`）不能替代后端认证，后端 API 必须有独立认证
- 敏感操作必须由后端验证权限
- 不使用 URL 参数传递敏感数据

---

## 竞态条件

### 异步请求竞态

- **问题**：watch 中发起异步请求，旧请求可能在新请求之后返回，导致过期数据覆盖最新结果
- **检测信号**：watch 回调内使用 async/await 但无 AbortController 或请求序号机制
- **修复方向**：Vue 3.5+ 使用 `onWatcherCleanup` 取消旧请求；旧版本使用递增请求序号忽略过期响应

### watch vs watchEffect 竞态

- **问题**：多个 watch 相互修改对方监听的值，形成循环依赖甚至无限循环
- **检测信号**：watch A 修改 B，watch B 修改 A
- **修复方向**：明确依赖方向，能用 computed 的尽量用 computed 替代 watch

---

## 性能

### 响应式系统

- **问题**：`reactive()` 包裹深层巨型对象，整个对象树被代理，内存和性能开销大
- **检测信号**：reactive 包裹大对象（>100 个属性层级）
- **修复方向**：大对象使用 `shallowRef()` / `shallowReactive()` 避免深层代理

- **问题**：模板中进行复杂计算（filter + map + join 等链式操作）
- **检测信号**：模板表达式中含多步链式操作
- **修复方向**：提取到 `computed` 属性，利用缓存避免重复计算

### 组件渲染优化

- **问题**：`v-for` 无 key 或用 index 作为 key，增删排序时渲染错乱
- **检测信号**：`v-for` 缺少 `:key`，或 `:key="index"`
- **修复方向**：使用稳定唯一的业务 ID 作为 key

- **问题**：`v-if` 和 `v-for` 同级使用（Vue 3 中 v-if 优先级更高，可能导致预期外的行为）
- **检测信号**：同一元素上同时出现 v-if 和 v-for
- **修复方向**：先用 computed 过滤再 v-for 循环，或将 v-if 提升到外层容器

- 大列表考虑使用虚拟滚动（如 `vue-virtual-scroller`）而非全量渲染

### 异步组件与懒加载

- 路由级组件使用动态 import 实现代码分割
- 非首屏重型组件使用 `defineAsyncComponent` 延迟加载

---

## 代码质量

### Composition API 最佳实践

- **问题**：`<script setup>` 过长，混合多种关注点（数据获取、状态管理、业务逻辑、UI 逻辑）
- **检测信号**：`<script setup>` 超过 150 行，包含 3+ 个不相关的 watch
- **修复方向**：将关联逻辑提取为可复用 composable（`useXxx`），保持组件简洁

### Props/Emits 定义

- **问题**：Props 无类型定义（仅数组形式 `defineProps(['a', 'b'])`）
- **检测信号**：`defineProps` 使用数组语法或泛型参数为 any
- **修复方向**：使用完整的 TypeScript 泛型定义 + `withDefaults` 设置默认值

- **问题**：Emits 未声明，Vue 3 会警告，Vue 2 可能被误认为原生事件
- **检测信号**：`$emit` 调用但无 `defineEmits` 声明
- **修复方向**：使用完整类型化的 `defineEmits`

### 生命周期

- 不要在 `onBeforeMount` 中进行 DOM 操作（DOM 尚未挂载）
- 在 `onMounted` 中进行异步数据获取时注意竞态处理
- 定时器、订阅、事件监听必须在 `onBeforeUnmount` 中清理

---

## Vue 2 特定问题

### 响应式陷阱

- **新增属性不响应**：直接 `this.obj.newProp = value` 不是响应式的，需要使用 `this.$set(this.obj, 'newProp', value)`
- **数组索引赋值不响应**：`this.items[0] = newValue` 不触发更新，需要使用 `this.$set(items, 0, newValue)` 或 `splice`
- **数组长度直接修改不响应**：`this.items.length = 0` 不触发，应使用 `this.items = []`

### Options API 组织

- 大型组件中 data、methods、computed、watch 分散在不同选项导致逻辑难以追踪
- 推荐提取 mixins（注意命名冲突）或升级到 Vue 3 Composition API

---

## 常见反模式

| 反模式 | 说明 | 修复方向 |
|--------|------|----------|
| **巨型组件** | 单组件 > 500 行 | 拆分为子组件 + composable |
| **v-if/v-for 同级** | 渲染性能差，逻辑不清晰 | 先 computed 过滤再循环 |
| **直接修改 props** | 破坏单向数据流 | 使用 emit 通知父组件 |
| **无 key 的 v-for** | 渲染错误和性能差 | 提供唯一 key |
| **过度使用 provide/inject** | 耦合难跟踪 | 优先 props/emit 或 Pinia |
| **在 watch 中修改被监听值** | 可能无限循环 | 使用 computed 替代 |
| **未清理的定时器/订阅** | 内存泄漏 | onBeforeUnmount 中清理 |
| **$refs 直接操作 DOM** | 绕过 Vue 响应式系统 | 使用响应式数据驱动 |

---

## Vue 3 `<Suspense>`

- 异步组件（`<script setup>` 中有顶层 await）外层必须包裹 `<Suspense>` 提供加载状态
- 搭配 `onErrorCaptured` 或 error boundary 处理异步加载失败
- 多个独立数据源使用嵌套 `<Suspense>` 避免瀑布式等待（外层快速数据，内层慢速数据）
- `#fallback` 插槽内容应提供有意义的骨架屏，不是空白

---

## `defineModel`（Vue 3.4+）

- 简化 v-model 双向绑定，替代传统的 props + emit 手动模式
- 注意不要过度使用导致双向绑定链路复杂难以追踪
- `defineModel` 返回值不应被直接修改绕过 setter 逻辑
- 对于需要转换/验证的场景，仍使用传统的 props + emit 模式

---

## `<Teleport>` 安全

- Teleport 目标元素必须存在且可控，避免注入到不可信的 DOM 位置
- Teleported 内容中如有 `v-html` 需特别注意 XSS 风险扩展到全局 DOM
- Teleported 组件的焦点管理需正确处理（模态框的焦点陷阱）
- 注意 Teleported 内容的事件冒泡仍按组件树传播，非 DOM 树

---

## Nuxt.js 特定检查

### 服务端渲染安全

- `useFetch`/`useAsyncData` 中服务端执行的代码使用的环境变量不会到客户端，但返回的 data 会序列化
- `server/` 目录中的 API routes 需要独立认证，不依赖前端路由守卫
- `runtimeConfig.public` 中的所有值会暴露到客户端，仅存放可公开的配置

### 配置分离

- 公私配置正确分离：`runtimeConfig.apiSecret` 仅服务端，`runtimeConfig.public.apiBase` 可公开
- `nuxt.config` 中的 `runtimeConfig` 区分 `public` 和非 `public`

### Nuxt 模块安全

- 第三方 Nuxt 模块需评估社区信任度和维护活跃度
- 模块是否请求了过多权限（修改路由、注入全局中间件）
- 自定义模块中避免 XSS 注入点（如 `addPlugin` 注入的代码）

### 渲染模式

- `ssr: false` 页面需正确处理客户端激活期间的 hydration 不匹配问题
- 混合渲染（SSR + CSR + Static）的页面边界应清晰分离
- `useCookie` 存储敏感数据时必须设置 `httpOnly: true` 和 `secure: true`