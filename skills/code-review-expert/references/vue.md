# Vue 特定检测信号

## 竞态条件
- watch 内异步无 onWatcherCleanup / onCleanup → **P1**
- 多个 watch 形成循环依赖 → **P1**
- 异步请求未使用 AbortController 取消过期请求 → **P1**

## 性能
- reactive() 包裹巨型对象（应使用 shallowRef/shallowReactive）→ **P2**
- 模板中复杂计算未用 computed → **P2**
- v-for 无 :key 或用 index 作 key → **P2**
- v-if 和 v-for 同级（v-if 优先级高于 v-for，应先过滤再循环）→ **P1**
- 大列表无虚拟滚动 → **P2**

## XSS / 安全
- v-html 无 DOMPurify → **P0**
- :href 绑定用户可控 URL → **P1**
- Teleport 内容含 v-html / 用户输入（XSS 扩展到全局 DOM）→ **P0**
- Nuxt `runtimeConfig.public` 中暴露密钥 → **P0**

## Composition API 陷阱
- defineModel 过度使用导致双向绑定链路不可追踪 → **P2**
- defineProps / defineEmits 无类型声明 → **P3**
- 过长 `<script setup>` 未提取 composable → **P2**
- 直接修改 defineModel 返回值（绕过 setter 逻辑）→ **P2**

## Vue 2 特定
- 直接给对象新增属性（不响应）→ 需 Vue.set / $set → **P1**
- 数组索引赋值（不响应）→ 需 $set 或 splice → **P1**

## Nuxt
- useFetch / useAsyncData 返回数据含敏感信息 → **P0**
- server/ API routes 依赖前端路由守卫认证（需独立认证）→ **P0**
- useCookie 敏感数据未设 httpOnly / secure → **P1**

## 常见反模式信号
- 单组件 >500 行
- 直接修改 props（破坏单向数据流）
- watch 中修改被监听的值（可能死循环）
- provide/inject 过度使用（耦合难跟踪）
- $refs 直接操作 DOM 绕过响应式 → 用响应式数据驱动
- 未清理 timer/subscription/worker（onBeforeUnmount）
