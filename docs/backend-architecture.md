# MailNex 后端架构

## 一、总体结构

后端为 Go 编写的单二进制服务，前端 Vue 产物通过 `go:embed` 打入可执行文件，数据库为 SQLite 单文件。启动时自动建表与写入初始数据，运行时不需要外部 MySQL、Redis、Java 或 Nginx。

| 层 | 目录 | 职责 |
| --- | --- | --- |
| 启动 | `server/cmd/mailnex/main.go` | 读取配置、打开数据库、迁移、装配路由、优雅退出 |
| 配置 | `server/internal/config` | 监听地址、数据目录、数据库路径、JWT 密钥与有效期 |
| 数据库 | `server/internal/database` | SQLite 连接、表结构、初始数据、迁移 |
| 认证 | `server/internal/auth` | JWT 签发校验、内存会话、验证码 |
| 接口 | `server/internal/httpapi` | 路由、中间件、各业务 handler |
| 前端托管 | `server/internal/webui` | 静态资源与 SPA history 回退 |

## 二、启动流程

1. `config.Load()` 读取环境变量，填充默认值；`JWT_SECRET` 缺失或不足 32 字节时使用内置开发密钥。
2. `database.Open()` 创建数据目录并以 WAL 模式打开 SQLite，设置 `busy_timeout` 与 `foreign_keys`。
3. `database.Migrate()` 执行 `schema.sql` 建表，再执行 `seed.sql` 写入初始数据；两者全部幂等。
4. 创建附件与正文存储目录。
5. 装配会话表、验证码表、路由与中间件，启动 HTTP 服务。
6. 后台每 10 分钟清理过期令牌；收到中断信号后优雅关闭。

## 三、数据库

### 连接参数

| 参数 | 取值 | 说明 |
| --- | --- | --- |
| `journal_mode` | `WAL` | 提升并发读性能，写入不阻塞读 |
| `busy_timeout` | `5000` | 写锁等待 5 秒，避免瞬时锁冲突直接失败 |
| `foreign_keys` | `1` | 开启外键约束 |
| `synchronous` | `NORMAL` | 兼顾安全与写入性能 |
| 最大连接数 | `1` | SQLite 单写者模型下避免写冲突 |

### 表结构

`schema.sql` 创建两类表：

- 系统表 `sys_*`：用户、角色、菜单、部门、岗位、字典、参数、操作日志、登录日志及关联表。
- 业务表 `mail_*`：域名、DNS、证书、邮箱账号、邮件、附件、验证码、收件人组、联系人、模板、变量、发信任务、任务收件人、投递日志、SMTP 凭据、API 密钥、内容过滤、安全策略、卡密、卡密使用日志、系统事件。

`seed.sql` 写入管理员、示例组织与角色、菜单与按钮权限、业务字典、示例域名、邮箱账号、收件人组、联系人、模板、变量、卡密、过滤规则与安全策略。所有语句使用 `INSERT OR IGNORE`，重复启动不会覆盖已有数据。

### 存储策略

邮件正文、原始 EML 与附件落磁盘，数据库只保存相对路径、SHA-256 摘要与列表预览。这样邮件量增长时数据库体积保持可控，备份与迁移只需拷贝数据目录。

## 四、认证与会话

| 机制 | 实现 | 说明 |
| --- | --- | --- |
| 登录令牌 | HS256 JWT | 载荷含 `sub` 用户名、`jti` 令牌标识、`iat` 签发时间、`exp` 过期时间 |
| 会话校验 | `auth.SessionStore` | 内存表保存 `jti` 与用户名，校验令牌是否仍有效；退出即撤销 |
| 验证码 | `auth.CaptchaStore` | 4 位字符，5 分钟有效，验证后立即消费 |
| 密码 | bcrypt | 初始密码 `admin123` 对应固定密文 |

由于会话保存在内存，进程重启后旧令牌失效，需要重新登录。单机部署下这是可接受的取舍，也省去了外部 Redis。

## 五、接口层

### 统一响应

所有接口返回统一结构：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `code` | number | 业务码，200 表示成功 |
| `message` | string | 提示信息 |
| `data` | any | 业务数据 |
| `traceId` | string | 链路追踪标识，同时写入 `X-Trace-Id` 响应头 |

### 中间件

| 中间件 | 位置 | 作用 |
| --- | --- | --- |
| `withAuth` | `middleware.go` | 校验 `Authorization: Bearer`，解析 JWT 并确认会话有效，注入用户名与令牌标识 |
| `operLog` | `middleware.go` | 记录写操作到 `sys_oper_log`；跳过 GET 与认证接口 |

### 分页与导出

分页统一使用 `pageNum` / `pageSize` 参数，返回 `pageNum` / `pageSize` / `total` / `rows`，单页最大 100 条。导出统一返回带 BOM 的 UTF-8 CSV，字段按 RFC 4180 转义。

## 六、前端托管

`internal/webui/webui.go` 通过 `go:embed all:dist` 嵌入前端产物：

1. 请求路径命中静态文件时直接返回，带正确 MIME 类型。
2. 未命中的非 `/api` 路径返回 `index.html`，交给 Vue Router 处理深链刷新。
3. `/api/**` 由后端路由处理，未注册的接口返回 404 JSON，不会落到前端页面。

## 七、构建与发布

```powershell
# 构建前端
cd scaffold-vue
npm ci
npm run build

# 同步前端产物到 embed 目录
Remove-Item ..\server\internal\webui\dist -Recurse -Force -ErrorAction SilentlyContinue
Copy-Item dist ..\server\internal\webui\dist -Recurse

# 构建单文件（关闭 CGO 保证纯静态）
cd ..\server
$env:CGO_ENABLED = '0'
go build -o ..\dist\mailnex.exe .\cmd\mailnex
```

产物 `mailnex.exe` 约 16 MB，包含前端页面、后端接口与数据库表结构。发布时只需该文件，运行时在同级目录自动创建 `data` 目录。

## 八、与旧架构的差异

| 维度 | 旧实现 | 当前实现 |
| --- | --- | --- |
| 后端语言 | Java 17 + Spring Boot | Go 1.21+ |
| 构建方式 | Maven 多模块 + jar | `go build` 单文件 |
| 数据库 | MySQL 5.7+ | SQLite 单文件 |
| 会话存储 | Redis | 内存令牌表 |
| 前端托管 | 独立 Vite 服务 | `go:embed` 嵌入二进制 |
| 部署依赖 | JDK、MySQL、Redis、可选 Nginx | 单个可执行文件 |
