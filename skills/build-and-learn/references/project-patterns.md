# 项目模式参考

## 常见项目类型与步骤模板

### 1. 前端 Web 应用（React）

**起点假设**：用户已安装 Node.js，但可能没有。

| 步骤 | 做什么 | 核心概念 | 关键文件 |
|------|--------|----------|----------|
| 环境检查 | 确认 Node.js、npm 版本 | 包管理器是什么 | 无 |
| 项目脚手架 | `npm create vite@latest` | 项目结构、package.json | 项目根目录 |
| 第一个组件 | 创建 App 组件，渲染「Hello」 | 组件、JSX | `src/App.jsx` |
| 添加状态 | 计数器功能 | `useState`、事件处理 | `src/App.jsx` |
| 组件拆分 | 把计数器抽成独立组件 | Props 传递、组件通信 | `src/components/Counter.jsx` |
| 列表渲染 | 渲染一组数据 | `map()`、key 属性 | `src/App.jsx` |
| 用户输入 | 表单、添加条目 | 受控组件、onChange | `src/components/AddForm.jsx` |
| 样式 | 加 CSS | className、CSS Modules | `src/App.module.css` |
| 部署 | 构建 + 预览 | 构建过程、静态文件 | `dist/` |

**React 常见教学要点**：
- JSX 不是 HTML，是 JavaScript 表达式
- 组件名必须大写
- `useState` 的不可变性（不要直接改 state）
- 为什么需要 key
- 受控组件 vs 非受控组件

### 2. 前端 Web 应用（Vue）

| 步骤 | 做什么 | 核心概念 | 关键文件 |
|------|--------|----------|----------|
| 环境检查 | 确认 Node.js 版本 | 包管理器 | 无 |
| 项目脚手架 | `npm create vue@latest` | Vite、项目结构 | 项目根目录 |
| 第一个组件 | App.vue 改成 Hello | SFC 单文件组件、template | `src/App.vue` |
| 响应式数据 | 计数器 | `ref()`、`{{ }}` 插值 | `src/App.vue` |
| 组件拆分 | 抽离 Counter | Props、defineProps | `src/components/Counter.vue` |
| 列表渲染 | `v-for` 渲染数组 | `v-for`、`:key` | `src/App.vue` |
| 用户输入 | 表单、`v-model` | 双向绑定 | `src/App.vue` |

**Vue 常见教学要点**：
- SFC 的三个块：`<template>` `<script>` `<style>`
- `ref()` vs `reactive()`
- `v-model` 是语法糖
- 模板中访问 ref 不用 `.value`

### 3. 后端 API（Node.js + Express）

| 步骤 | 做什么 | 核心概念 | 关键文件 |
|------|--------|----------|----------|
| 环境检查 | Node.js、npm | 运行时环境 | 无 |
| 初始化 | `npm init -y`、装 Express | package.json、依赖 | `package.json` |
| Hello World | 一个返回文本的接口 | 路由、请求/响应 | `index.js` |
| GET 接口 | 返回 JSON 数据 | JSON、状态码 | `index.js` |
| 路由拆分 | 用 Express Router | 模块化、Router | `routes/items.js` |
| POST 接口 | 接收用户数据 | 请求体、express.json() | `routes/items.js` |
| 数据持久化 | 用文件存数据 | fs 模块、JSON 读写 | `data/items.json` |

**后端常见教学要点**：
- 什么是 API、什么是 REST
- GET vs POST vs PUT vs DELETE
- 状态码 200/201/400/404/500 的含义
- `req.params` vs `req.query` vs `req.body`

### 4. CLI 命令行工具（Node.js）

| 步骤 | 做什么 | 核心概念 | 关键文件 |
|------|--------|----------|----------|
| 初始化 | 创建项目、配置 package.json | bin 字段 | `package.json` |
| Hello CLI | 输出一行文字 | shebang、chmod | `cli.js` |
| 接收参数 | 拿到命令行参数 | `process.argv` | `cli.js` |
| 交互式输入 | 问用户问题 | readline 或 inquirer | `cli.js` |
| 文件操作 | 读/写文件 | fs 模块 | `cli.js` |
| 美化输出 | 彩色文字、表格 | chalk 或原生 | `cli.js` |

### 5. 全栈应用（React + Express）

**前置条件**：用户至少对一个端有基础了解。

| 步骤 | 做什么 | 核心概念 |
|------|--------|----------|
| 后端初始化 | Express + 基础 API | 同后端模板 |
| 前端初始化 | Vite + React | 同前端模板 |
| 连接前后端 | 前端 fetch 调后端 | CORS、fetch API、异步 |
| 状态同步 | 页面数据来自后端 | useEffect、loading 状态 |

**全栈特别注意**：
- 一定要讲 CORS 是什么、为什么需要
- 前端请求的 URL 在开发时和部署后不一样
- 错误处理：后端报错前端怎么展示

---

## 脚手架式教学策略

### 什么是脚手架式教学？

类比盖楼时的脚手架：先搭骨架，再填充细节。在编程教学中：

1. **先给一个能跑的简单版**（骨架）
2. **再逐步替换/扩展**（填充）
3. **每次只改一个部分**

### 示例：教 React 组件

❌ **错误方式**：直接写出最终版本的完整组件
```jsx
// 不要一次给这个
function TodoApp() {
  const [items, setItems] = useState([]);
  const [input, setInput] = useState('');
  const addItem = () => { ... };
  const removeItem = (id) => { ... };
  return ( /* 大段 JSX */ );
}
```

✅ **正确方式**：分步来
```jsx
// 第 1 步：最简单的组件，只显示静态内容
function TodoApp() {
  return <div>待办事项</div>;
}

// 第 2 步：加一个状态变量，显示在页面上
function TodoApp() {
  const [items] = useState(['学 React', '做项目']);
  return (
    <div>
      <h1>待办事项</h1>
      {items.map(item => <div>{item}</div>)}
    </div>
  );
}

// 第 3 步：加输入框
// ... 以此类推
```

### 教学的「三个不跳」

1. **不跳概念**：每个新概念都完整解释
2. **不跳代码**：每行代码都解释
3. **不跳验证**：每步都让用户跑一遍确认

---

## 常见错误预判与纠正

### React 新手常见错误

| 错误 | 原因 | 如何纠正 |
|------|------|----------|
| `'X' is not defined` | 忘记 import 或组件名小写 | 检查 import、组件名首字母大写 |
| 直接修改 state | `items.push(x)` 而不是 `setItems([...items, x])` | 讲不可变性原理 |
| 无限循环 | useEffect 依赖写错 | 讲 useEffect 的依赖数组 |
| `Each child should have a unique key` | map 没加 key | 讲 React 为什么需要 key |

### Vue 新手常见错误

| 错误 | 原因 | 如何纠正 |
|------|------|----------|
| 模板中 ref 不生效 | 没 return 或没 setup | 讲 script setup 语法 |
| `v-model` 不更新 | 绑定了 computed | 讲 computed 是只读的 |
| 样式不生效 | 没加 scoped 或选择器不对 | 讲 scoped 原理 |

### 后端新手常见错误

| 错误 | 原因 | 如何纠正 |
|------|------|----------|
| `Cannot GET /xxx` | 路由没定义或路径写错 | 检查路由注册 |
| `req.body` 是 undefined | 忘记 `express.json()` 中间件 | 讲中间件的顺序 |
| `ECONNREFUSED` | 前后端端口不一致 | 讲端口、CORS |
| 修改不生效 | 没重启服务器 | 介绍 nodemon |

---

## 概念映射表

什么概念在项目的哪个阶段讲最合适？

| 概念 | 最佳教学时机 | 项目类型 |
|------|-------------|----------|
| 变量、数据类型 | 第一步写代码时 | 所有 |
| 函数 | 第一次需要复用逻辑时 | 所有 |
| 数组、对象 | 第一次处理数据时 | 所有 |
| 异步/回调 | 第一次调 API 时 | 全栈、前端 |
| Promise/async-await | 调 API 且需要处理结果时 | 全栈、前端 |
| 模块化 | 文件超过 100 行时 | 所有 |
| 状态管理 | 状态需要在多组件共享时 | 前端 |
| 路由 | 需要多页面时 | 前端、后端 |
| 环境变量 | 需要区分开发/生产环境时 | 所有 |
| 错误处理 | 第一次写 try-catch 时 | 所有 |
| 中间件 | 需要鉴权、日志时 | 后端 |
| 数据库 | 需要持久化时 | 后端、全栈 |

**决策原则**：不要提前讲 —— 在用户「需要它」的那一刻再讲。
