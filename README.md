# user-center（用户中心）

一个“前后端分离”的用户中心小项目：后端提供注册 / 登录 / JWT 登录态（HttpOnly Cookie + 双 token）/ 管理员接口；前端提供登录、个人资料、后台用户管理等页面。

## 明确说明（项目演进：Session → JWT）

本项目是在原有 **Session（JSESSIONID）登录态方案**基础上改造而来，核心目标是：

- ✅ 将传统“有状态认证”升级为“无状态 JWT 认证”，并补齐安全机制


---

### 🧱 改造前（Session 模式）

原项目使用典型的：

- `HttpSession + JSESSIONID Cookie`

工作方式：

1. 登录成功 → 服务端保存用户到 Session
2. 浏览器保存 `JSESSIONID`
3. 每次请求通过 Session 获取用户

```
前端 → Cookie(JSESSIONID) → 后端 → Session → 用户信息
```

---

### ⚙️ 改造后（JWT + HttpOnly Cookie）

本项目完成了完整的 JWT 化改造：

```
前端 → Cookie(JWT) → 后端 → 解析 Token → 用户信息
```

核心变化：

- ❌ 移除 Session
- ✅ 引入 JWT（Json Web Token）
- ✅ 登录态改为 HttpOnly Cookie
- ✅ 通过拦截器统一鉴权

---

## 🔑 关键改造点（核心价值）

### 1）JWT 替代 Session

- 登录成功后：
    - 不再写入 Session
    - 改为签发 JWT
- JWT 存储：
    - 使用 **HttpOnly Cookie**
    - 前端无法读取，浏览器自动携带

👉 实现“无状态认证”

---

### 2）token_version（解决 JWT 无法失效的问题）

JWT 最大问题：**一旦签发，默认无法主动失效**

本项目通过数据库字段解决：

```sql
token_version INT NOT NULL DEFAULT 0
```

机制：

- JWT 中写入：`ver`
- 每次请求校验：
    
    ```
    token.ver == user.token_version
    ```
    
- 以下操作会使 token 立即失效：
    - 登出
    - 修改密码
    - 封禁用户

👉 实现：**“伪无状态 + 可控失效”**

---

### 3）JwtAuthInterceptor（统一认证入口）

新增拦截器：

- 拦截所有请求 `/**`
- 白名单放行：
    - `/user/login`
    - `/user/register`
    - `/user/logout`
    - `/user/refresh`
    - `/user/session/ttl`
    - `/public/**`
    - `/swagger-ui/**`
    - `/swagger-ui.html`
    - `/v3/api-docs/**`
    - `/error`
- 核心逻辑：
    1. 从 Cookie 读取 JWT
    2. 验签 + 校验过期
    3. 校验 token_version
    4. 注入当前用户到 request

👉 替代 Session 获取用户

---

### 4）CSRF 防护（关键安全点）

由于 JWT 放在 Cookie 中，引入 CSRF 风险。

本项目实现：

### ✅ Double Submit Cookie

- Cookie：`csrf_token`
- Header：`X-CSRF-Token`

校验规则：

```
Cookie.csrf_token == Header.X-CSRF-Token
```

👉 防止跨站请求伪造（很多 JWT 教程会忽略）

---

### 5）登录 / 登出行为变化

### 登录

- 返回：用户信息
- 设置：
    - `token`（HttpOnly）
    - `refresh_token`（HttpOnly）
    - `csrf_token`

### 登出（重点）

- 不只是清 Cookie
- 还会：
    
    ```sql
    token_version + 1
    ```
    

👉 确保旧 token **立即失效**

---

### 6）前端改造点

- 不再存 token（localStorage ❌）
- 全部依赖 Cookie 自动携带
- Axios：

```tsx
withCredentials: true
```

- 写请求自动加：

```tsx
X-CSRF-Token
```

---

## ⚠️ 当前实现的取舍

为了突出学习重点，当前方案有意简化：

- 未引入 Redis 黑名单
- 未做 Refresh Token Rotation
- 未使用 Spring Security

👉 但核心认证链路已经完整


## 功能清单 / Feature List

### 用户侧

- 用户注册 / 登录（JWT 登录态：Access Token + Refresh Token 存 HttpOnly Cookie）
- 获取当前登录用户信息
- 修改个人资料
- 退出登录（服务端 token_version 自增，旧 token 立即失效）
- 查看 JWT 有效期（秒）
- Access 过期时前端自动用 Refresh Token 换发（双 token 续期，前端内置 401 自动刷新）

### 管理员侧

- 用户列表查询（支持 `username` 模糊搜索）
- 用户信息修改 / 逻辑删除
- 角色管理（角色列表 / 新建角色）
- 用户角色分配（当前实现：一个用户最多一个角色）

### 其他

- 参数校验（`@Valid`）
- 统一返回体（`ApiResponse` + `GlobalResponseAdvice`）
- 统一异常处理（`GlobalExceptionHandler`）
- JWT 鉴权拦截（黑名单排除：`/**` 排除登录/注册/登出/refresh/TTL/文档等）
- CSRF 防护（Double Submit Cookie，写操作校验 `X-CSRF-Token`）
- CORS 配置（可选，前后端分离部署时配置 `cors.allowed-origins`）
- 管理员鉴权拦截（拦截 `/admin/**`）
- Swagger / OpenAPI 接口文档

## 整体架构说明

本项目采用典型的前后端分离架构：前端负责页面展示与交互，后端负责业务与数据，二者通过 HTTP 接口通信。

- 前端（`user-center-web`）
  - 负责页面展示与用户交互（登录、个人资料、后台管理页面等）
  - 通过 Axios 调用后端接口（比如 `/user/login`、`/admin/user/search`）
  - 开发环境通过 Vite 代理把 `/user`、`/admin` 转发到后端，避免跨域

- 后端（`user-center`）
  - 提供 RESTful API（`/user/**`、`/admin/**`）
  - 使用 JWT（HttpOnly Cookie）维护登录态，双 token：Access Token（鉴权）+ Refresh Token（仅换发）
  - 通过拦截器统一做管理员鉴权（拦截 `/admin/**`，未登录 401、无权限 403）
  - 使用 MyBatis-Plus 操作数据库，`isDelete` 字段实现逻辑删除
  - 统一响应与异常：正常返回用 `ApiResponse` 包装，异常由全局异常处理器统一输出

- 数据库（MySQL）
  - 三张核心表：`user`（用户）、`role`（角色字典）、`user_role`（用户-角色关系）
  - 当前设计：`user_role.user_id` 唯一，因此一个用户最多绑定一个角色

- 通信方式
  - 前端 <-> 后端：HTTP（JSON）
  - 后端 <-> 数据库：MyBatis-Plus（CRUD）

### 认证与安全说明（JWT + Cookie）

- **Token 类型（type claim）**：claim 名为 `type`（也可用标准 `typ`）。取值**固定枚举**：`access` / `refresh`（全小写）。校验时**不区分大小写、trim 首尾空格**，仅接受上述两值，避免大小写/空格绕过。
- **校验顺序**：**先验签、验 exp，再根据 claims 判定 type**。type 判定一律基于验签后的 payload，伪造的 type 无法影响逻辑分支。
- **CSRF**：写操作（含 POST `/user/refresh`）均走 Double Submit Cookie：Cookie `csrf_token` 与 header `X-CSRF-Token` 一致才放行。**/user/refresh 不在 CSRF 排除列表**，与登录一致受保护。
- **Refresh Token 风险与可选增强**：当前 Refresh Token 为**纯 JWT、自校验、不落库、不 rotation**。
  - **风险**：若 Refresh 泄露，在有效期内可被持续用于换发新 Access，从而间接调用业务接口；“影响更小”的前提是 Refresh 更难被获取（HttpOnly + 同源/可信域）且建议配合 rotation。
  - **可选增强（下一期）**：**Refresh Rotation** — Refresh 单次使用、换发后旧 Refresh 失效；DB 存 Refresh 哈希或版本，复用检测；可与 `token_version` 联动实现全端失效。文档中将其列为可选增强，并明确不 rotation 时的上述风险。

## 技术栈

- 后端：Spring Boot 3.5.8、MyBatis-Plus 3.5.5、MySQL、Springdoc OpenAPI（Swagger UI）
- 前端：Vue 3、Vite、Pinia、Vue Router、Axios

## 主要依赖



### 后端（Maven / pom.xml）

- `spring-boot-starter-web`：提供 HTTP 接口（Controller / Tomcat）
- `spring-boot-starter-validation`：参数校验（`@Valid` 等）
- `mybatis-plus-spring-boot3-starter`：数据库 CRUD（少写很多 SQL）
- `mysql-connector-j`：连接 MySQL 的驱动
- `lombok`：自动生成 getter/setter（少写样板代码）
- `springdoc-openapi-starter-webmvc-ui`：Swagger UI + OpenAPI 文档
- `jjwt-api` / `jjwt-impl` / `jjwt-jackson`（0.12.x）：JWT 签发与解析
- `spring-boot-devtools`（可选）：开发热重载

Lombok 注意点：

- 如果你在 IDE 里看到 `@Data` 等注解报红，通常是 IDE 没装 Lombok 支持
- IntelliJ IDEA：安装 Lombok 插件，并开启 Annotation Processing

### 前端（npm / package.json）

- 运行依赖：Vue 3、Vue Router、Pinia、Axios
- 开发依赖：Vite、TypeScript、vue-tsc、ESLint、Prettier

## 目录结构

- `user-center/`：后端（Spring Boot）
- `user-center-web/`：前端（Vue + Vite）

## 快速开始（Windows）

### 1）准备环境

- JDK 21
- MySQL（建议 8.x）
- Node.js（建议 18+）

### 2）初始化数据库

1. 在 MySQL 里创建数据库（名字和后端配置一致，默认是 `yupi`）：

   ```sql
   CREATE DATABASE IF NOT EXISTS yupi DEFAULT CHARACTER SET utf8mb4;
   ```

2. 创建 `user` 表（下面这份结构对齐你现在库里的 `user` 表字段；若已有表，需执行 `user-center/src/main/resources/migration_token_version.sql` 增加 `token_version`）：

   ```sql
   USE yupi;

   CREATE TABLE IF NOT EXISTS `user` (
     `id` BIGINT NOT NULL AUTO_INCREMENT,
     `username` VARCHAR(256) DEFAULT NULL,
     `userAccount` VARCHAR(256) NOT NULL,
     `avatarUrl` VARCHAR(1024) DEFAULT NULL,
     `gender` TINYINT DEFAULT NULL,
     `userPassword` VARCHAR(512) NOT NULL,
     `phone` VARCHAR(128) DEFAULT NULL,
     `email` VARCHAR(512) DEFAULT NULL,
     `userStatus` INT NOT NULL DEFAULT 0,
     `createTime` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
     `token_version` INT NOT NULL DEFAULT 0,
     `updateTime` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
     `isDelete` TINYINT(1) NOT NULL DEFAULT 0,
     `userRole` TINYINT(1) NOT NULL DEFAULT 0,
     PRIMARY KEY (`id`),
     UNIQUE KEY `uk_user_account` (`userAccount`)
   ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
   ```

3. 导入角色与用户-角色关联表：

   - SQL 文件位置：`user-center/src/main/resources/schema_role.sql`
   - 这份脚本会创建 `role`、`user_role`，并插入一些初始角色（包含 `ADMIN`）

### 3）启动后端（Spring Boot）

1. 进入后端目录：

   ```powershell
   cd .\user-center\
   ```

2. 配置数据库连接（推荐用环境变量）：

   当前后端读取的环境变量是：

   - 必填：`DB_PASSWORD`（数据库密码）
   - 可选：`DB_URL`（数据库连接串，不配则使用本地默认值）
   - 可选：`DB_USERNAME`（数据库账号，不配则默认 `root`）
   - 可选：`JWT_SECRET`（JWT 签名密钥，生产环境务必配置）
   - 可选：`CORS_ALLOWED_ORIGINS`（前后端分离部署时，逗号分隔的允许源，如 `http://localhost:3000`）

   临时生效（只对当前 PowerShell 窗口有效，关闭窗口就没了）：

   ```powershell
   $env:DB_PASSWORD="你的真实数据库密码"
   # 可选：
   # $env:DB_USERNAME="root"
   # $env:DB_URL="jdbc:mysql://localhost:3306/yupi?useSSL=false&serverTimezone=UTC&useUnicode=true&characterEncoding=utf8&allowPublicKeyRetrieval=true"
   ```

   永久生效（写入系统环境变量，新开 PowerShell 才会生效）：

   ```powershell
   setx DB_PASSWORD "你的真实数据库密码"
   ```

3. 启动：

   ```powershell
   .\mvnw.cmd spring-boot:run
   ```

4. 验证是否启动成功：

   - 后端默认端口：`http://localhost:8090`
   - Swagger UI：`http://localhost:8090/swagger-ui/index.html`
   - OpenAPI JSON：`http://localhost:8090/v3/api-docs`

### 4）启动前端（Vue + Vite）

1. 进入前端目录：

   ```powershell
   cd ..\user-center-web\
   ```

2. 安装依赖并启动：

   ```powershell
   npm install
   npm run dev
   ```

3. 打开页面：

   - 前端默认端口：`http://localhost:3000`
   - 已配置代理：`/user`、`/admin` 会自动转发到后端 `http://localhost:8090`

## 数据库表设计（三张表）

数据库里这三张表：`user`、`role`、`user_role`。

### 1）user（用户表）

存“用户本体信息”，比如账号、密码、头像、状态等。

- `id`：主键，自增
- `username`：昵称/用户名
- `userAccount`：登录账号（唯一）
- `userPassword`：登录密码（项目里是加盐 MD5 后存库）
- `avatarUrl`：头像链接
- `gender`：性别
- `phone` / `email`：联系方式
- `userStatus`：用户状态（例如 0 正常、1 封禁）
- `userRole`：老的“单字段角色”（0 普通用户、1 管理员）
- `createTime` / `updateTime`：创建时间 / 更新时间
- `isDelete`：逻辑删除标记（0 未删、1 已删）
- `token_version`：JWT 版本号（INT NOT NULL DEFAULT 0），登出/改密/封禁时自增，使旧 token 立即失效

### 2）role（角色表）

这是“角色字典”，只负责记录系统里有哪些角色（ADMIN、DRIVER 等）。

- `id`：主键，自增
- `role_key`：角色标识（唯一，比如 `ADMIN`）
- `role_name`：角色中文名（比如 管理员）
- `description`：描述
- `status`：状态（0 启用、1 禁用）
- `createTime` / `updateTime`：创建时间 / 更新时间
- `isDelete`：逻辑删除标记（0 未删、1 已删）

### 3）user_role（用户-角色关系表）

这张表是“绑定关系”，用来说明“哪个用户拥有哪些角色”。

- `id`：主键，自增
- `user_id`：用户 id（外键关联 `user.id`）
- `role_id`：角色 id（外键关联 `role.id`）
- `createTime`：创建时间

注意：当前 `schema_role.sql` 里对 `user_role.user_id` 加了唯一索引（`UNIQUE KEY uk_user_role_user_id (user_id)`），这代表“一个用户最多绑定一个角色”。这也和后端接口 `/admin/user/roles/assign` 的校验逻辑一致：一次最多只能给用户分配 1 个角色。

## 接口速览

### 普通用户接口

- `POST /user/register`：注册
- `POST /user/login`：登录（成功后 Set-Cookie：Access Token、Refresh Token、csrf_token；body 返回脱敏用户信息）
- `POST /user/refresh`：用 Refresh Token 换发新的 Access + Refresh Token（需带 CSRF header；Access 过期时前端可自动调用）
- `GET /user/current`：获取当前登录用户（依赖 Access Token Cookie，未登录或 token 无效会 401）
- `POST /user/logout`：退出登录（服务端 token_version 自增并清除 Cookie；幂等）
- `POST /user/update`：更新自己的资料（必须登录；写操作需带 `X-CSRF-Token`）
- `GET /user/session/ttl`：返回 JWT 有效期（秒），无需登录

### 管理员接口（统一走 `/admin/**`）

- `GET /admin/user/search`：按 username 模糊搜索用户
- `POST /admin/user/delete`：按 id 删除用户（逻辑删除）
- `POST /admin/user/update`：更新用户信息
- `GET /admin/user/roles`：查询用户拥有的角色 id 列表
- `POST /admin/user/roles/assign`：给用户分配角色（当前实现：一个用户最多一个角色）
- `GET /admin/role/list`：查询角色列表
- `POST /admin/role/create`：创建角色

你也可以直接用后端自带的请求示例：

- `user-center/src/main/resources/test.http`（为旧版 Session 示例，当前 JWT 版请以 Cookie `token` / `refresh_token` / `csrf_token` 为准）

## 权限与登录态

- **登录态**：JWT 存于 HttpOnly Cookie（`token` = Access Token，`refresh_token` = Refresh Token）。登录成功后后端 Set-Cookie，浏览器自动携带，前端无需读写 token。
- **鉴权**：需登录的接口由 JwtAuthInterceptor 从 Cookie 读取 Access Token，验签、校验 `token_version` 与 DB 一致后注入当前用户。
- **写操作**：POST/PUT/DELETE/PATCH 需在请求头带 `X-CSRF-Token`（与 Cookie 中 `csrf_token` 一致），否则 403。
- **续期**：Access Token 过期后，前端收到 401 时可自动调用 `POST /user/refresh`（携带 Refresh Token Cookie + CSRF header）换发新 token 并重试原请求。
- **管理员接口**（`/admin/**`）：在 JWT 鉴权通过后，再校验是否为管理员；未登录 401，非管理员 403。

管理员判断规则（满足任一即可）：

1. `user.userRole == 1`（user 表里的管理员标记）
2. 或者：在 `user_role` 里把这个用户绑定到 `role_key = 'ADMIN'` 的角色（通过 user_role -> role 这条关系判断）

如果你想把某个用户变成管理员，有两种常见做法（两选一即可）：

1）直接改 user 表的标记：

```sql
UPDATE `user` SET `userRole` = 1 WHERE id = 你的用户id;
```

2）通过角色表绑定（更“正规”，扩展性更好）：

```sql
-- 先确认 ADMIN 角色的 id
SELECT id FROM role WHERE role_key = 'ADMIN' AND isDelete = 0 AND status = 0;

-- 再把用户绑定到 ADMIN（注意：user_role 的 user_id 是唯一的，所以同一用户只能保留一条绑定）
INSERT INTO user_role(user_id, role_id) VALUES (你的用户id, ADMIN角色id)
ON DUPLICATE KEY UPDATE role_id = VALUES(role_id);
```

## 常见问题

- **调用管理员接口一直 401**：先调用 `POST /user/login` 登录，并确保请求能带上 Cookie（`withCredentials: true`；Postman/浏览器会自动带，手写 curl 需 `-b` 带 Cookie）。写操作还需带 header `X-CSRF-Token`（与 Cookie 中 `csrf_token` 一致）。
- **前端请求报跨域**：开发环境一般不会，因 Vite 已把 `/user`、`/admin` 代理到后端 8090。前后端分离部署时，需配置后端 `cors.allowed-origins`（或环境变量 `CORS_ALLOWED_ORIGINS`）为前端源，且须 `allowCredentials: true`。
- **导入 `schema_role.sql` 报错**：请先创建 `user` 表（因 `user_role` 外键引用 `user(id)`）。若已有 `user` 表但无 `token_version` 字段，请执行 `user-center/src/main/resources/migration_token_version.sql`。
## 项目演示截图
<img width="1635" height="734" alt="image" src="https://github.com/user-attachments/assets/6e00cb8d-6519-4385-a6de-73813c1e06ed" />
<img width="1909" height="761" alt="image" src="https://github.com/user-attachments/assets/81c48dde-90ba-4e9d-bb42-e6cd572860ef" />
<img width="1915" height="728" alt="image" src="https://github.com/user-attachments/assets/3881f577-d52b-4f91-9ee4-5965ee23f1e2" />
<img width="1873" height="795" alt="image" src="https://github.com/user-attachments/assets/d3619f4b-f895-42d2-aaa5-bee9fff3dcf3" />
<img width="1649" height="788" alt="image" src="https://github.com/user-attachments/assets/d055a92e-4047-4ffb-9442-2d6268047906" />
<img width="1846" height="746" alt="image" src="https://github.com/user-attachments/assets/eec3e402-31c5-4d1f-981e-5f3c62410a60" />

## 后续可扩展方向

当前已实现：JWT（HttpOnly Cookie）、双 token（Access + Refresh）、token_version 登出/改密即时失效、CSRF Double Submit、CORS 可选配置。

### 1）Refresh Rotation（Refresh Token 单次使用）

当前：Refresh Token 为纯 JWT、不落库、不 rotation，泄露后在有效期内可被持续用于换发。

可扩展为：换发时使旧 Refresh 失效（DB 存 Refresh 哈希或版本，单次使用 + 复用检测），可与 `token_version` 联动实现全端失效。

### 2）引入 Spring Security（统一认证与授权）

当前：自定义 JwtAuthInterceptor + AdminAuthInterceptor，路径级鉴权。

可扩展为：Spring Security 接管认证与授权，`UserDetailsService`、`SecurityContext`，注解式权限（如 `@PreAuthorize("hasRole('ADMIN')")`），接口级/方法级权限。

### 3）引入 Redis（缓存与鉴权增强）

当前：鉴权每次查库校验 `token_version`，角色判断查 user_role + role。

可扩展为：Redis 缓存用户信息/角色列表、热点数据，减少查库；或 Redis 存 token 黑名单/Refresh 白名单，增强登出与 rotation 能力。

