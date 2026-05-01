# Vue 特定检查清单

本清单覆盖 Vue 2/3 应用的安全、性能和代码质量问题。

---

## 安全检查

### XSS 风险

```vue
<!-- 危险：v-html 未过滤 -->
<div v-html="userInput"></div>

<!-- 推荐：使用 DOMPurify 过滤 -->
<script setup>
import DOMPurify from 'dompurify'
const safeHtml = computed(() => DOMPurify.sanitize(userInput.value))
</script>
<template>
  <div v-html="safeHtml"></div>
</template>

<!-- 危险：动态属性绑定可能注入恶意值 -->
<a :href="userProvidedUrl">Link</a>

<!-- 推荐：验证 URL -->
<script setup>
const safeUrl = computed(() =>
  userProvidedUrl.value?.startsWith('http') ? userProvidedUrl.value : '#'
)
</script>
```

### 敏感数据处理

- **响应式数据暴露**：Vue DevTools 可查看所有响应式数据
- **Pinia/Vuex 状态**：敏感数据不应存储在全局状态
- **localStorage 存储**：Token 等敏感信息易被 XSS 访问

### 认证/授权

```vue
<!-- 危险：仅前端路由守卫 -->
<script>
router.beforeEach((to, from, next) => {
  if (to.meta.requiresAuth && !isLoggedIn()) {
    next('/login')
  } else {
    next()
  }
})
// 后端 API 必须有独立的认证检查
</script>
```

- 前端路由守卫不能替代后端认证
- 敏感操作必须由后端验证权限
- 不使用 URL 参数传递敏感数据

---

## 竞态条件

### 异步请求竞态

```javascript
// 危险：未处理请求顺序
let currentId = ref(1)
watch(currentId, async (id) => {
  const data = await fetchData(id) // 旧请求可能在新请求之后返回
  result.value = data
})

// 推荐：使用 AbortController 或请求序号
let requestId = 0
watch(currentId, async (id) => {
  const thisRequest = ++requestId
  try {
    const data = await fetchData(id)
    if (thisRequest === requestId) {
      result.value = data
    }
  } catch (e) {
    if (thisRequest === requestId) {
      error.value = e
    }
  }
})

// Vue 3 Composition API 推荐：使用 onWatcherCleanup
import { watch, onWatcherCleanup } from 'vue'
watch(currentId, async (id) => {
  const controller = new AbortController()
  onWatcherCleanup(() => controller.abort())
  const data = await fetchData(id, { signal: controller.signal })
  result.value = data
})
```

### watch vs watchEffect 竞态

```javascript
// 危险：多个 watch 相互触发
const a = ref(0)
const b = ref(0)
watch(a, (val) => { b.value = val * 2 })
watch(b, (val) => { a.value = val / 2 }) // 无限循环风险

// 推荐：明确依赖方向，避免循环依赖
const a = ref(0)
const b = computed(() => a.value * 2)
```

---

## 性能

### 响应式系统

```javascript
// 危险：深层响应式大对象
const largeData = reactive(hugeObject) // 整个对象树被代理

// 推荐：使用 shallowRef/shallowReactive 处理大对象
const largeData = shallowRef(hugeObject)
```

```vue
<!-- 危险：模板中复杂计算 -->
<template>
  <div>{{ items.filter(i => i.active).map(i => i.name).join(', ') }}</div>
</template>
```

```vue
<!-- 推荐：使用 computed -->
<script setup>
const activeNames = computed(() =>
  items.value.filter(i => i.active).map(i => i.name).join(', ')
)
</script>
<template>
  <div>{{ activeNames }}</div>
</template>
```

### 组件渲染优化

```vue
<!-- 危险：v-for 无 key -->
<li v-for="item in items">{{ item.name }}</li>

<!-- 推荐：提供唯一 key -->
<li v-for="item in items" :key="item.id">{{ item.name }}</li>

<!-- 危险：v-if 和 v-for 同级 -->
<li v-for="item in items" v-if="item.active" :key="item.id">

<!-- 推荐：先过滤再循环 -->
<script setup>
const activeItems = computed(() => items.value.filter(i => i.active))
</script>
<template>
  <li v-for="item in activeItems" :key="item.id">
</template>

<!-- 推荐：大列表使用虚拟滚动 -->
<!-- 使用 vue-virtual-scroller 等库 -->
```

### 异步组件与懒加载

```javascript
// 推荐：路由懒加载
const routes = [
  {
    path: '/dashboard',
    component: () => import('@/views/Dashboard.vue')
  }
]

// 推荐：异步组件
import { defineAsyncComponent } from 'vue'
const HeavyComponent = defineAsyncComponent(() =>
  import('@/components/HeavyComponent.vue')
)
```

---

## 代码质量

### Composition API 最佳实践

```javascript
// 危险：过长 setup / <script setup>
// 应将逻辑提取为自定义 composable

// 推荐：提取可复用逻辑
// composables/useUser.ts
export function useUser() {
  const user = ref(null)
  const loading = ref(false)

  async function fetchUser(id) {
    loading.value = true
    try {
      user.value = await api.getUser(id)
    } finally {
      loading.value = false
    }
  }

  return { user, loading, fetchUser }
}

// 组件中
<script setup>
const { user, loading, fetchUser } = useUser()
</script>
```

### Props 定义

```typescript
// 危险：无类型定义
const props = defineProps(['title', 'count'])

// 推荐：完整类型定义
interface Props {
  title: string
  count: number
  items: Array<{ id: number; name: string }>
  loading?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  loading: false
})
```

### Emits 定义

```typescript
// 危险：无 emits 声明
// （在 Vue 2 中会被视为原生事件，Vue 3 中会警告）

// 推荐：完整类型化 emits
const emit = defineEmits<{
  update: [value: string]
  delete: [id: number]
  submit: []
}>()
```

### 生命周期

```javascript
// 危险：在错误的生命周期执行操作
// - 不要在 onBeforeMount 中进行 DOM 操作
// - 不要在 onMounted 中进行异步数据获取的竞态处理

// 推荐：正确使用生命周期
onMounted(() => {
  // DOM 操作安全
  // 注意：fetchData 可能产生竞态，需要处理
  const controller = new AbortController()
  fetchData({ signal: controller.signal })

  // 清理函数（Vue 3.2+）
  onBeforeUnmount(() => {
    controller.abort()
  })
})
```

### Watch 清理

```javascript
// 危险：未清理副作用
watch(source, async (newVal) => {
  const result = await fetch(newVal)
  data.value = result // 可能设置过期数据
})

// 推荐：使用 onWatcherCleanup（Vue 3.5+）
watch(source, async (newVal, oldVal, onCleanup) => {
  let cancelled = false
  onCleanup(() => { cancelled = true })

  const result = await fetch(newVal)
  if (!cancelled) {
    data.value = result
  }
})
```

---

## Vue 2 特定问题

### Options API

```javascript
// 危险：大型组件 Options API 代码组织混乱
// data、methods、computed、watch 交错难以跟踪

// 推荐：
// 1. 使用 mixins 提取可复用逻辑（但注意命名冲突）
// 2. 或升级到 Vue 3 Composition API
// 3. 或使用 @vue/composition-api 插件
```

### 响应式陷阱

```javascript
// Vue 2 危险：新增属性不是响应式的
this.newProp = 'value' // 不响应！

// 推荐：使用 Vue.set
this.$set(this.obj, 'newProp', 'value')

// Vue 2 危险：数组索引赋值不是响应式的
this.items[0] = newValue // 不响应！

// 推荐：
this.$set(this.items, 0, newValue)
// 或
this.items.splice(0, 1, newValue)
```

---

## 常见反模式

| 反模式 | 说明 | 修复 |
|--------|------|------|
| **巨型组件** | 单组件 > 500 行 | 拆分为子组件 + composable |
| **v-if/v-for 同级** | 渲染性能差 | 先过滤再循环 |
| **直接修改 props** | 破坏单向数据流 | 使用 emit 通知父组件 |
| **无 key 的 v-for** | 渲染错误和性能差 | 提供唯一 key |
| **过度使用 provide/inject** | 耦合难跟踪 | 优先 props/emit 或状态管理 |
| **在 watch 中修改被监听值** | 可能无限循环 | 使用 computed 替代 |
| **未清理的定时器/订阅** | 内存泄漏 | onBeforeUnmount 中清理 |
| **$refs 直接操作 DOM** | 绕过 Vue 响应式 | 使用响应式数据驱动 |

---

## Vue 3 `<Suspense>`

```vue
<!-- 推荐：包裹异步组件 -->
<template>
  <Suspense>
    <template #default>
      <AsyncDashboard />
    </template>
    <template #fallback>
      <DashboardSkeleton />
    </template>
  </Suspense>
</template>

<script setup>
// AsyncDashboard.vue 中使用 await
// <script setup>
// const data = await fetchDashboard()
// </script>
</script>
```

### Suspense 检查点

- [ ] 异步组件外层是否包裹了 `<Suspense>` 提供加载状态？
- [ ] 是否搭配了 `onErrorCaptured` 或 error boundary 处理加载失败？
- [ ] 多个独立数据源是否使用嵌套 `<Suspense>` 避免瀑布式等待？
- [ ] `#fallback` 内容是否合理（不是空白）？
- [ ] 是否错误地在非异步 setup 组件中使用 `<Suspense>`？（无效果）

---

## `defineModel`（Vue 3.4+）

```vue
<!-- 传统方式：手动 emit -->
<script setup>
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()
</script>
<template>
  <input :value="props.modelValue" @input="emit('update:modelValue', $event.target.value)" />
</template>

<!-- Vue 3.4+ 推荐：defineModel 简化 v-model -->
<script setup>
const model = defineModel<string>({ required: true })
</script>
<template>
  <input v-model="model" />
</template>

<!-- 多个 v-model -->
<script setup>
const firstName = defineModel<string>('firstName', { required: true })
const lastName = defineModel<string>('lastName', { default: '' })
</script>
```

### defineModel 检查点

- [ ] 是否过度使用 `defineModel` 导致双向绑定链路复杂难以追踪？
- [ ] `defineModel` 返回值是否被直接修改（绕过了 setter 逻辑）？
- [ ] 对于需要转换/验证的场景，是否仍使用传统的 props + emit 模式？

---

## `<Teleport>` 安全

```vue
<!-- 危险：Teleport 到未受控的 DOM 位置 -->
<template>
  <Teleport to="body">
    <div v-html="userContent"></div>  <!-- XSS 风险扩展到全局 DOM -->
  </Teleport>
</template>

<!-- 推荐：Teleport 配合安全措施 -->
<template>
  <Teleport to="#modal-root">
    <div class="modal" role="dialog" aria-modal="true">
      <slot />
    </div>
  </Teleport>
</template>
```

### Teleport 检查点

- [ ] Teleport 目标元素是否存在且可控？（避免注入到不可信的 DOM）
- [ ] Teleported 内容的样式是否导致全局样式污染？
- [ ] Teleported 的焦点管理是否正确？（模态框的焦点陷阱）
- [ ] CSP 配置是否允许 Teleport 的目标操作？
- [ ] Teleported 内容的 Event 冒泡是否符合预期？（事件仍按组件树传播）

---

## Nuxt.js 特定检查

### 服务端渲染安全

- **`useFetch`/`useAsyncData` 中的密钥**：服务端执行的代码中使用的环境变量不会到客户端，但返回的 data 会
- **`server/` 目录的认证**：API routes 需要独立认证，不依赖前端路由守卫
- **`nuxt.config` 中的敏感配置**：`runtimeConfig.public` 中的所有值会暴露到客户端

```typescript
// nuxt.config.ts — 正确分离公私配置
export default defineNuxtConfig({
  runtimeConfig: {
    // 仅服务端可访问
    apiSecret: process.env.API_SECRET,
    // 公开给客户端
    public: {
      apiBase: process.env.API_BASE_URL  // 可公开的 URL
    }
  }
})
```

### Nuxt 模块安全

- [ ] 第三方 Nuxt 模块是否有足够的社区信任度和维护活跃度？
- [ ] 模块是否请求了过多权限（如修改路由、注入全局中间件）？
- [ ] 自定义模块中是否有 XSS 注入点（如 `addPlugin` 注入的代码）？

### 渲染模式

- [ ] `ssr: false` 的页面是否正确处理了客户端激活期间的 hydration 不匹配？
- [ ] 混合渲染（SSR + CSR + Static）的页面边界是否清晰？
- [ ] `useCookie` 的敏感数据是否设置了 `httpOnly` 和 `secure`？
