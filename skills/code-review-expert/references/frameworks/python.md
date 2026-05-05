# Python 后端检查清单

本清单覆盖 Python（Django/Flask/FastAPI）应用的安全、并发、性能和代码质量问题。

## 快速索引

| 问题类型 | 级别 | 参考章节 |
|---------|------|---------|
| SQL 注入（raw SQL + % 格式化） | P0 | 安全检查 |
| Pickle 反序列化不可信数据 | P0 | 安全检查 |
| eval/exec 执行用户输入 | P0 | 安全检查 |
| Jinja2 模板注入（autoescape 关闭） | P0 | 安全检查 |
| DEBUG=True 生产环境 | P0 | Django 专项 |
| SECRET_KEY 硬编码 | P0 | Django 专项 |
| Django N+1 查询 | P1 | Django 专项 |
| asyncio.create_task 未 await | P1 | 并发 |
| 异常过宽（except Exception pass） | P1 | 代码质量 |
| 可变默认参数 | P2 | 代码质量 |
| import * 污染命名空间 | P3 | 代码质量 |
| request 无超时 | P1 | Flask/FastAPI |
| 路径遍历（os.path.join 用户输入） | P0 | 安全检查 |

---

## 安全检查

### SQL 注入

- 使用 `%` 格式化或 f-string 拼接 SQL（Django raw、SQLAlchemy text）→ **P0**
- Django `Model.objects.raw()` 使用 `%` 传入用户输入 → **P0**
- `cursor.execute()` 字符串拼接 → **P0**，必须参数化

### 反序列化

- `pickle.loads()` / `cPickle.loads()` 反序列化不可信数据 → **P0**
- `yaml.load()` / `yaml.full_load()` 替代 `yaml.safe_load()` → **P1**
- `eval()` / `exec()` / `compile()` 执行不可信字符串 → **P0**

### 模板注入（SSTI）

- Jinja2 渲染用户可控的模板字符串 → **P0**
- Flask `render_template_string()` 使用用户输入 → **P0**
- Django 模板的 `|safe` 过滤器用于用户输入 → **P0**

### 路径遍历

- `os.path.join` 拼接用户输入 → **P0**，需校验规范化路径仍在允许目录内
- 文件上传路径由用户文件名控制 → **P0**

---

## Django 专项

### 配置安全

- `DEBUG = True` 在生产环境 → **P0**（泄露配置/代码路径/SQL）
- `SECRET_KEY` 硬编码在 settings.py → **P0**
- `ALLOWED_HOSTS = ['*']` → **P1**

### ORM 查询

- N+1 查询：循环中访问外键/关联对象未用 `select_related` / `prefetch_related` → **P1**
- `Model.objects.raw()` 字符串拼接用户输入 → **P0**
- `QuerySet.extra()` 注入 → **P0**

### 中间件

- CSRF 中间件被禁用 → **P1**
- 安全响应头中间件缺失（`SECURE_HSTS_SECONDS`、`SECURE_SSL_REDIRECT`）→ **P1**
- Session Cookie 未设 `SESSION_COOKIE_SECURE=True` → **P1**

---

## Flask / FastAPI 专项

### Flask

- `app.debug = True` 生产环境 → **P0**
- Jinja2 `autoescape` 关闭 → **P0**
- `request.args.get()` 直接用于数据库查询 → **P0**
- Session secret key 使用默认值 → **P0**

### FastAPI

- 依赖注入函数中的全局可变状态（多次请求共享）→ **P1**
- 异步路径中调用同步阻塞函数（阻塞事件循环）→ **P2**
- Pydantic model 未做输入校验（`str` 代替 `constr(min_length=...)`）→ **P2**

### 通用

- HTTP 请求无超时（`requests.get` 无 timeout 参数）→ **P1**
- 生产环境使用开发服务器（Flask dev server、`uvicorn --reload`）→ **P1**

---

## 并发

### asyncio

- `asyncio.create_task()` 创建的 task 未 await 也未存储引用（可能被 GC 取消）→ **P1**
- 异步函数内调用同步阻塞代码（阻塞事件循环）→ **P2**
- `asyncio.gather` 一个协程异常导致其他协程被取消未处理 → **P2**

### 多线程

- `ThreadPoolExecutor` 未调用 `shutdown()` → **P2**
- 共享可变状态无 `threading.Lock` 保护 → **P1**
- GIL 限制下 CPU 密集型用多线程（应用 multiprocessing）→ **P3**

### 多进程

- `multiprocessing.Pool` 未关闭 → **P2**
- 子进程异常未捕获导致僵尸进程 → **P1**

---

## 代码质量

### 错误处理

- `except Exception: pass` 或 `except: pass` 吞没异常 → **P1**
- `except` 捕获范围过宽 → **P2**
- 异常信息未记录上下文 → **P2**

### 函数设计

- 可变对象作为默认参数（`def fn(items=[])`）→ **P2**
- 函数参数超过 5 个 → **P2**
- 函数超过 50 行 → **P2**

### 模块与导入

- `from module import *` 污染命名空间 → **P3**
- 循环导入 → **P2**
- 在函数/循环内部 import（隐藏依赖）→ **P3**

### 类型与安全

- `is` 用于数值/字符串比较（应 `==`）→ **P2**
- 可变对象作为类属性（所有实例共享）→ **P1**
- 未使用 `dataclass`/`NamedTuple` 的纯数据类 → **P3**

---

## 常见反模式

| 反模式 | 说明 | 修复方向 |
|--------|------|----------|
| **全局变量滥用** | 模块级可变状态影响测试隔离 | 依赖注入 / 上下文对象 |
| **`__del__` 依赖** | 析构函数调用时机不确定 | 显式 close/context manager |
| **list comprehension 过深** | 3+ 层嵌套难以阅读 | 拆分为循环或生成器函数 |
| **裸 except** | 捕获所有异常包括 KeyboardInterrupt | 指定具体异常类型 |
| **环境变量硬编码默认值** | `os.getenv('KEY', 'dev-secret')` | 缺失时报错而非用默认值 |
