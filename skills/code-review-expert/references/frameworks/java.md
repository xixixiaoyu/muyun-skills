# Java / Spring Boot 检查清单

本清单覆盖 Java（Spring Boot/JPA/Hibernate）应用的安全、并发、性能和代码质量问题。

## 快速索引

| 问题类型 | 级别 | 参考章节 |
|---------|------|---------|
| SQL 注入（JDBC Statement 拼接） | P0 | 安全检查 |
| JPA native query 字符串拼接 | P0 | 安全检查 |
| ObjectInputStream 反序列化不可信数据 | P0 | 安全检查 |
| XXE（XML 解析器未禁用外部实体） | P0 | 安全检查 |
| Log4Shell 模式（JNDI 注入） | P0 | 安全检查 |
| @Transactional 自调用失效 | P1 | Spring Boot |
| 缺少 @Valid/@Validated | P2 | Spring Boot |
| actuator 端点暴露 | P1 | Spring Boot |
| JPA N+1 查询 | P1 | JPA/Hibernate |
| Open Session In View 性能问题 | P1 | JPA/Hibernate |
| SimpleDateFormat 非线程安全 | P1 | 并发 |
| HashMap 多线程环境 | P1 | 并发 |
| 资源未 try-with-resources | P1 | 代码质量 |
| Optional 用作字段/getter 返回 null | P2 | 代码质量 |
| BigDecimal equals 而非 compareTo | P2 | 代码质量 |
| 连接池未配置超时 | P1 | 常见反模式 |
| 静态集合作缓存无界增长 | P1 | 常见反模式 |

---

## 安全检查

### SQL 注入

- JDBC `Statement`（非 `PreparedStatement`）拼接用户输入 → **P0**
- JPA `createNativeQuery` / `@Query(nativeQuery=true)` 字符串拼接 → **P0**
- `CriteriaBuilder` 动态查询未安全处理用户输入 → **P0**
- MyBatis `${}` 字符串替换（应使用 `#{}` 参数绑定）→ **P0**
- 动态 ORDER BY / GROUP BY 未白名单 → **P1**

### 反序列化

- `ObjectInputStream.readObject()` 反序列化不可信数据 → **P0**
- Fastjson/Jackson 反序列化启用了 defaultTyping / enableDefaultTyping → **P0**
- 反序列化白名单未配置或不完整 → **P1**

### XXE

- `DocumentBuilderFactory` / `SAXParserFactory` 未禁用外部实体 → **P0**
- `XMLInputFactory` 未设置 `IS_SUPPORTING_EXTERNAL_ENTITIES=false` → **P0**
- JAXB Unmarshaller 反序列化 XML 未配置安全 → **P1**

### 其他注入

- XPath 查询拼接用户输入 → **P0**
- 日志框架 JNDI 注入（Log4Shell 模式）→ **P0**，Log4j >=2.17.1 并禁用 lookup
- EL 表达式注入（Spring EL/OGNL 执行用户输入）→ **P0**

---

## Spring Boot 专项

### 事务

- `@Transactional` 方法自调用（同类内调用）事务不生效 → **P1**
- `@Transactional` 仅捕获 RuntimeException 默认，受检异常不回滚 → **P2**
- 事务内调用外部 API/发送消息（长事务持有锁）→ **P1**

### 安全配置

- Controller 方法缺少 `@Valid` / `@Validated` 校验 → **P2**
- Spring Security 配置中使用 `permitAll()` 但路径未正确匹配 → **P1**
- actuator 端点公开暴露（未限制访问）→ **P1**
- `application.properties` 中包含硬编码密钥 → **P0**

### 依赖注入

- 循环依赖（`@Autowired A → B → A`）→ **P2**
- Prototype scope bean 注入到 Singleton 中未用 `@Lookup` → **P2**
- `@Autowired` 字段注入而非构造器注入（降低可测试性）→ **P3**

---

## JPA / Hibernate 专项

### 查询

- N+1 查询：`FetchType.EAGER` 默认导致多余查询 → **P1**
- 关联查询未使用 `JOIN FETCH` / `@EntityGraph` → **P1**
- `Open Session In View` 开启导致数据库连接持有时间过长 → **P1**
- 批量操作使用循环 `save()` 而非 `saveAll()` → **P2**

### 实体设计

- Entity 中使用 Lombok `@Data`（生成的 equals/hashCode 包含所有字段）→ **P2**
- `@OneToMany` 使用 `FetchType.EAGER`（默认 Lazy 更安全）→ **P2**
- Entity 间双向关联未管理 `mappedBy` 导致循环 → **P2**

### 缓存

- 一级缓存（Persistence Context）脏数据导致更新丢失 → **P1**
- 二级缓存配置不当导致多个节点数据不一致 → **P1**

---

## 并发

### 线程安全

- `SimpleDateFormat` 作为静态字段多线程使用（非线程安全）→ **P1**
- `HashMap` 在多线程环境使用（应用 `ConcurrentHashMap`）→ **P1**
- `synchronized` 范围过大导致性能瓶颈 → **P2**
- 双重检查锁定实现错误（缺少 `volatile`）→ **P1**

### 资源管理

- `ThreadLocal` 在线程池环境中未 `remove()` 导致内存泄漏 → **P1**
- `CountDownLatch.await()` 无超时 → **P2**
- 锁未在 finally 中释放 → **P1**

---

## 代码质量

### 异常处理

- `catch (Exception e) { }` 空块吞没异常 → **P1**
- 异常抛出后资源未关闭 → **P1**（应用 try-with-resources）
- `e.printStackTrace()` 直接打印而非日志框架 → **P2**
- 自定义异常未保留原始异常（`new MyException(msg)` 而非 `new MyException(msg, cause)`）→ **P3**

### 类型与 API

- `BigDecimal` 用 `equals()` 而非 `compareTo()` 比较数值（`equals` 区分 2.0 和 2.00）→ **P2**
- `Optional` 用作类字段或方法参数 → **P2**（Optional 设计用于返回值）
- `Optional.get()` 未检查 `isPresent()` → **P1**
- `String` 用 `==` 比较而非 `.equals()` → **P1**

### Lombok

- `@Data` 在 JPA Entity 上（equals/hashCode 包含所有字段导致代理/延迟加载问题）→ **P2**
- `@Builder` 无 `@AllArgsConstructor` 配合 JPA → **P2**
- `@EqualsAndHashCode` 未指定 `callSuper = true`（继承时）→ **P3**

---

## 常见反模式

| 反模式 | 说明 | 修复方向 |
|--------|------|----------|
| **连接池未配置超时** | HikariCP 默认无限等待 | 设置 connectionTimeout / idleTimeout |
| **静态集合作缓存** | `static Map` 无限增长无上限 | 用 Caffeine/Guava Cache 设置 TTL |
| **日志打印敏感数据** | 用户手机号/身份证/密码出现在日志 | 脱敏后在 log |
| **异常用于流程控制** | try-catch 做业务分支判断 | 用条件判断替代异常 |
| **Utils/Helper 膨胀** | 单个工具类 20+ 方法各不相同 | 按职责拆分 |"