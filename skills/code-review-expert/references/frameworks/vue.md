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
