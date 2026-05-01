# 代码质量检查清单

## 错误处理

### 需要标记的反模式

- **吞掉异常**：空 catch 块或仅日志记录的 catch
  ```javascript
  try { ... } catch (e) { }  // 静默失败
  try { ... } catch (e) { console.log(e) }  // 记录后遗忘
  ```
- **过宽 catch**：捕获 `Exception`/`Error` 基类而非具体类型
- **错误信息泄露**：向用户暴露堆栈跟踪或内部细节
- **缺失错误处理**：可失败操作（I/O、网络、解析）周围无 try-catch
- **异步错误处理**：未处理的 Promise 拒绝、缺失 `.catch()`、无错误边界

### 需要检查的最佳实践

- [ ] 错误在适当边界被捕获
- [ ] 错误消息对用户友好（不暴露内部细节）
- [ ] 错误日志包含足够的调试上下文
- [ ] 异步错误被正确传播或处理
- [ ] 可恢复错误定义了回退行为
- [ ] 关键错误触发告警/监控

### 需要问的问题

- "这个操作失败时会发生什么？"
- "调用者会知道出了问题吗？"
- "有足够的上下文来调试这个错误吗？"

---

## 性能与缓存

### CPU 密集操作

- **热路径中的昂贵操作**：循环中的正则编译、JSON 解析、加密
- **阻塞主线程**：同步 I/O、无 worker/async 的重计算
- **不必要的重复计算**：相同计算执行多次
- **缺失记忆化**：纯函数用相同输入重复调用

### 数据库与 I/O

- **N+1 查询**：循环中每项发起一次查询而非批量
  ```javascript
  // 差：N+1
  for (const id of ids) {
    const user = await db.query(`SELECT * FROM users WHERE id = ?`, id)
  }
  // 好：批量
  const users = await db.query(`SELECT * FROM users WHERE id IN (?)`, ids)
  ```
- **缺失索引**：查询未索引的列
- **过度获取**：只需几列时使用 SELECT *
- **无分页**：将整个数据集加载到内存

### 网络请求优化

```javascript
// 危险：串行请求
const user = await fetchUser()
const orders = await fetchOrders()
const products = await fetchProducts()

// 推荐：并行请求（全部成功才继续）
const [user, orders, products] = await Promise.all([
  fetchUser(),
  fetchOrders(),
  fetchProducts()
])

// 推荐：并行请求（部分失败也能继续）
const results = await Promise.allSettled([
  fetchUser(),
  fetchOrders(),
  fetchProducts()
])

// 处理 allSettled 结果
const [userResult, ordersResult, productsResult] = results
const user = userResult.status === 'fulfilled' ? userResult.value : null
const orders = ordersResult.status === 'fulfilled' ? ordersResult.value : []
const products = productsResult.status === 'fulfilled' ? productsResult.value : []

// 记录失败的请求
results.forEach((result, index) => {
  if (result.status === 'rejected') {
    console.error(`Request ${index} failed:`, result.reason)
  }
})
```

**选择指南**：

- `Promise.all`：所有请求必须成功，任一失败则整体失败
- `Promise.allSettled`：允许部分失败，需要单独处理每个结果
- `Promise.race`：只需要最快的结果（如超时控制）
- `Promise.any`：只需要第一个成功的结果（如多源获取）

### 缓存问题

- **昂贵操作缺失缓存**：重复的 API 调用、DB 查询、计算
- **无 TTL 的缓存**：过期数据无限期提供
- **无失效策略的缓存**：数据更新但缓存未清除
- **缓存键冲突**：键唯一性不足
- **全局缓存用户特定数据**：安全/隐私问题

### 内存

- **无界集合**：无限增长的数组/Map
- **大对象保留**：持有引用阻止 GC
- **循环中的字符串拼接**：应使用 StringBuilder/join
- **完整加载大文件**：应使用流式处理

### 需要问的问题

- "这个操作的时间复杂度是多少？"
- "数据量增加 10 倍/100 倍时会如何表现？"
- "这个结果可缓存吗？应该缓存吗？"
- "能否批量处理而非逐个处理？"

---

## 边界条件

### Null/Undefined 处理

- **缺失 null 检查**：访问可能为 null 对象的属性
- **Truthy/Falsy 混淆**：`if (value)` 当 `0` 或 `""` 是有效值时
- **可选链过度使用**：`a?.b?.c?.d` 隐藏结构问题
- **Null vs Undefined 不一致**：混合使用无明确约定

```javascript
// 危险：直接访问可能不存在的数据
const userName = res.data.user.profile.name

// 推荐：安全访问 + 默认值
const userName = res?.data?.user?.profile?.name ?? '未知用户'

// 危险：数组方法在可能为空时调用
const firstItem = list[0]
const mapped = list.map((item) => item.name)

// 推荐：空数组保护
const firstItem = list?.[0]
const mapped = (list ?? []).map((item) => item?.name ?? '')
```

### 空集合

- **未处理空数组**：代码假设数组有元素
- **空对象边界情况**：空对象上的 `for...in` 或 `Object.keys`
- **首/末元素访问**：`arr[0]` 或 `arr[arr.length-1]` 无长度检查

### 数值边界

- **除零**：除法前缺失检查
- **整数溢出**：大数超过安全整数范围
- **浮点比较**：使用 `===` 而非 epsilon 比较
- **负值**：不应为负的索引或计数
- **Off-by-one 错误**：循环边界、数组切片、分页

```javascript
// 危险：金额计算精度问题
const total = price * quantity // 浮点精度丢失

// 推荐：使用整数计算（分为单位）或专用库
const totalCents = Math.round(priceCents * quantity)
const total = (totalCents / 100).toFixed(2)

// 危险：分页边界
const page = currentPage - 1 // 当 currentPage = 0 时为 -1

// 推荐：边界保护
const page = Math.max(0, currentPage - 1)
```

### 字符串边界

- **空字符串**：未作为边界情况处理
- **仅空白字符串**：通过 truthy 检查但实际为空
- **超长字符串**：无长度限制导致内存/显示问题
- **Unicode 边界情况**：Emoji、RTL 文本、组合字符

### 常见危险模式

```javascript
// 危险：无 null 检查
const name = user.profile.name

// 危险：数组访问无检查
const first = items[0]

// 危险：除法无检查
const avg = total / count

// 危险：truthy 检查排除有效值
if (value) { ... }  // 0、""、false 会失败

// 危险：类型假设
const id = params.id  // 可能是 string 而非 number
```

### 需要问的问题

- "如果这是 null/undefined 会怎样？"
- "如果这个集合为空会怎样？"
- "这个数字的有效范围是什么？"
- "在边界值（0、-1、MAX_INT）时会发生什么？"

---

## 类型安全

- [ ] Props/参数有完整的 TypeScript 类型定义
- [ ] API 响应有类型定义
- [ ] 避免使用 any 类型
- [ ] 使用类型守卫处理联合类型
- [ ] 泛型使用得当，避免过度泛化

---

## 框架特定质量检查

根据项目技术栈，加载对应的框架特定检查清单：

| 技术栈 | 参考文件 |
|--------|----------|
| React | `frameworks/react.md` |
| Vue | `frameworks/vue.md` |
