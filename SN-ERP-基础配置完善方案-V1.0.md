# SN ERP 基础配置完善方案 V1.0（Part 1/5）

> 基线提交：b64b9e1 | 输出日期：2026-04-26 | 状态：方案待确认

---

## 一、包管理器统一方案

### 1.1 现状评估

| 项目 | 当前状态 |
|---|---|
| 包管理器 | npm，已生成 package-lock.json |
| workspaces | 根 package.json 已配置 "workspaces": ["packages/*"] |
| pnpm-workspace.yaml | 不存在（无需移除） |
| monorepo 脚本 | 根已配置 npm-run-all --parallel |

### 1.2 决策

- 继续使用 npm，理由：
  - 项目已锁定 npm + package-lock.json，无迁移成本
  - npm workspaces 已正常工作，两个子包已识别
  - 无 pnpm 特性依赖
  - 团队无额外安装 pnpm 的心智负担

- pnpm-workspace.yaml 不存在，无需处理

### 1.3 Monorepo 命令统一规范

根 package.json scripts 命名规则：

- dev / dev:server / dev:web → 开发
- build / build:server / build:web → 构建
- lint → 全量检查
- test / test:server → 测试
- db:generate → prisma generate
- db:migrate → prisma migrate dev
- db:seed → prisma seed
- db:reset → prisma migrate reset（仅开发）

新增命令建议：
- lint:fix — 自动修复
- typecheck — 仅类型检查，不构建
- db:push — 开发阶段 schema 推送（不生成 migration 文件）
# SN ERP 基础配置完善方案 V1.0（Part 2/5）

## 二、代码规范方案

### 2.1 ESLint

| 项目 | 配置 |
|---|---|
| Server | @typescript-eslint/recommended + NestJS 插件 |
| Web | @typescript-eslint/recommended + React hooks 插件 + React refresh 插件 |
| 共享 | 根目录 .eslintrc.js 基础规则，子包可 override |

需要安装的包（devDependencies）：
- eslint、@typescript-eslint/parser、@typescript-eslint/eslint-plugin
- Server 额外：@nestjs/eslint-plugin
- Web 额外：eslint-plugin-react-hooks、eslint-plugin-react-refresh

关键规则：
- 禁止 any（warn 级别）
- 禁止未使用的变量（error）
- import 排序（eslint-plugin-import 或 simple-import-sort）

### 2.2 Prettier

根目录 .prettierrc，统一：
- singleQuote: true
- trailingComma: 'all'
- printWidth: 100
- tabWidth: 2
- semi: true

添加 .prettierignore 排除 dist/、node_modules/、package-lock.json。

### 2.3 TypeScript strict

| 子包 | 当前状态 | 目标 |
|---|---|---|
| Server | strictNullChecks + noImplicitAny（部分 strict） | 补全 strict: true |
| Web | 已有 strict: true | 保持，补充 forceConsistentCasingInFileNames |

Server 需要在 tsconfig.json 中将分散的 strict 选项替换为 "strict": true。

### 2.4 Import 规范

- 路径别名：Server 已有 @/* → src/*，Web 已有 @/* → ./src/* — 保持
- 禁止相对路径跨层引用：如 ../../common/xxx，应使用 @/common/xxx
- import 排序：第三方库 → 框架 → 本地模块，空行分隔
- 命名规范：
  - Service 类：xxx.service.ts
  - Controller：xxx.controller.ts
  - Module：xxx.module.ts
  - DTO：xxx.dto.ts、xxx-response.dto.ts
  - 前端页面组件：PascalCase，pages/XxxPage.tsx
  - 前端业务组件：components/XxxDialog.tsx

### 2.5 Commit 前检查

需要，使用 husky + lint-staged：

- pre-commit 钩子：运行 lint-staged
- lint-staged 配置：
  - packages/server/src/**/*.ts → eslint --fix, prettier --write
  - packages/web/src/**/*.{ts,tsx} → eslint --fix, prettier --write
  - **/*.{json,md} → prettier --write
- 暂不添加 commit-msg 钩子（commitlint 可后续引入）
- 提供 --no-verify 跳过说明（紧急修复时允许）
# SN ERP 基础配置完善方案 V1.0（Part 3/5）

## 三、后端基础规范

### 3.1 统一响应格式

所有 Controller 返回值由全局拦截器包装：

成功响应：
{ code: 0, message: "success", data: T }

分页响应：
{ code: 0, message: "success", data: { items: T[], total: number, page: number, pageSize: number } }

错误响应（由异常过滤器生成）：
{ code: number, message: string, details?: any }

code 规则：
- 0 = 成功
- 10000-19999 = 通用错误（参数校验、未登录等）
- 20000-29999 = 业务错误（采购单状态不符、SN重复等）
- 30000-39999 = 系统错误

### 3.2 统一异常格式

建立 BusinessException 类（继承 HttpException）：
BusinessException(moduleCode, businessCode, message, statusCode?)

全局异常过滤器（AllExceptionsFilter）捕获：
- BusinessException → 提取业务错误码和消息
- HttpException → 提取 HTTP 状态码
- 其他未知异常 → 500 + 隐藏详情（prod），详情日志（dev）

### 3.3 全局 ValidationPipe

在 main.ts 启用：
- whitelist: true — 自动剥离非 DTO 定义的字段
- forbidNonWhitelisted: true — 拒绝多余字段
- transform: true — 自动类型转换
- exceptionFactory — 自定义校验错误消息格式为数组

### 3.4 JWT 鉴权基础结构

模块设计：
modules/auth/
├── auth.module.ts          # 注册 JwtModule、PassportModule
├── auth.service.ts         # login、refreshToken
├── auth.controller.ts      # POST /api/auth/login、POST /api/auth/refresh
├── strategies/
│   └── jwt.strategy.ts     # Passport JWT 策略，从 token 解析 userId
└── dto/
    ├── login.dto.ts
    └── token-response.dto.ts

鉴权流程：
1. POST /api/auth/login → 验证用户名密码 → 返回 accessToken + refreshToken
2. 请求头 Authorization: Bearer <token> → JwtAuthGuard 校验
3. POST /api/auth/refresh → 用 refreshToken 换 accessToken

全局默认：
- 所有接口默认需鉴权（APP_GUARD 注册 JwtAuthGuard）
- 公开接口用 @Public() 装饰器豁免

### 3.5 权限枚举定义方式

双层架构：
1. 数据库层：SysPermission 表存储所有权限码（已存在），seed 时初始化
2. 应用层：TypeScript 枚举 + 装饰器

- common/constants/permission.enum.ts — 与 seed.ts 中的 code 一一对应
- common/decorators/permissions.decorator.ts — @Permissions('purchase:create')
- common/guards/permissions.guard.ts — 检查当前用户角色是否拥有该权限

权限码格式：{module}:{action}（如 purchase:create、sn:upload），与 seed.ts 现有格式一致。

### 3.6 操作日志拦截器

预留，但本期不实现完整逻辑。当前 SysOperationLog model 已在 schema 中定义。

方案：
- 创建 @LogAction('purchase', 'confirm') 装饰器
- 创建 OperationLogInterceptor，在 after 事件中记录操作
- 通过 Request 对象获取 userId、IP、UserAgent
- 异步写入数据库（不阻塞响应）
- 本期仅建立装饰器 + 拦截器骨架，具体记录在业务模块开发时填充
# SN ERP 基础配置完善方案 V1.0（Part 4/5）

## 四、前端基础规范

### 4.1 路由规范

目录结构：
src/router/
├── index.tsx           # 路由入口，配置全局 layout
├── routes.tsx          # 路由表定义（扁平化）
└── guard.tsx           # 鉴权守卫组件（AuthRedirect、RequireAuth）

路由规则：
- 一级路由对应侧边栏菜单：/purchase、/inventory、/sn、/system
- 二级路由对应页面：/purchase/list、/purchase/create、/purchase/:id
- 页面组件统一放 src/pages/{Module}/XxxPage.tsx
- 路由守卫：未登录 → /login，无权限 → 403 页面

### 4.2 API 请求封装

基于 axios（已安装）：
src/services/
├── request.ts          # axios 实例（baseURL、拦截器、token 注入、401 刷新）
├── api.ts              # 统一导出所有 API 函数
└── modules/            # 按模块拆分的 API 文件（如 auth.ts、purchase.ts）

关键点：
- 请求拦截器：自动注入 Authorization header
- 响应拦截器：解包 response.data，统一错误处理
- 401 时自动尝试 refresh token，失败则跳转登录
- 取消重复请求（可选，按需）

### 4.3 React Query 使用规范

使用场景：所有服务端数据获取

规范：
- 每个 API 函数对应一个 useXxxQuery / useXxxMutation
- Query Key 层级化：['purchase', 'list', params]、['purchase', 'detail', id]
- useMutation 成功后 invalidateQueries 刷新列表
- 全局配置 QueryClient：staleTime: 30s、retry: 1、refetchOnWindowFocus: false
- 集中存放：src/hooks/queries/ 或就近放在 page 文件内（简单页面）

### 4.4 Zustand 使用边界

使用场景：仅客户端状态

允许：
- 当前用户信息（useAuthStore）
- 侧边栏折叠状态（useLayoutStore）
- 全局通知/消息

禁止：
- 服务端数据缓存（用 React Query）
- 跨模块复杂状态（避免 store 膨胀）

Store 命名：src/stores/useXxxStore.ts

### 4.5 StatusTag 枚举映射

当前 src/constants/status.ts 已定义完整枚举，保持。

补充映射层：
src/constants/
├── status.ts           # 已有枚举定义
└── status-map.ts       # 新增：枚举 → { label, color, tagColor } 映射

映射规则：
- 每个枚举值对应一个中文标签 + Ant Design Tag 颜色
- 统一组件 <StatusTag status={PurchaseOrderStatus.CONFIRMED} /> 自动渲染
- 映射表集中维护，避免分散在页面组件中

颜色约定：
- 草稿/待处理 → default / orange
- 已确认/完成 → green / blue
- 红冲 → red
- 异常 → volcano

### 4.6 UI Token 使用规范

当前 tailwind.config.js 已定义部分颜色，保持并扩展。

- 不引入 Ant Design ConfigProvider theme token 自定义（增加复杂度，初期无必要）
- 直接使用 Ant Design 默认主题 + Tailwind 自定义颜色
- Tailwind 自定义颜色扩展需遵循 Ant Design 色板体系
- 禁止硬编码颜色值（如 text-red-500），统一用语义化 Token：
  - text-primary / text-error / text-success
  - bg-primary / bg-sidebar
- 如未来需要暗色主题，再引入 ConfigProvider theme
# SN ERP 基础配置完善方案 V1.0（Part 5/5）

## 五、数据库与 Prisma 规范

### 5.1 Migration 生成规则

| 场景 | 命令 |
|---|---|
| 开发阶段修改 schema | npx prisma migrate dev --name xxx |
| 生产部署 | npx prisma migrate deploy |
| 仅修改 SQL（自定义索引） | 手动编辑 migration SQL 文件 |

命名规则：
- migration 名称使用 snake_case：add_outbound_tables、create_partial_unique_index
- 不可删除已 applied 的 migration 文件
- 自定义 SQL 索引单独一个 migration

### 5.2 Partial Unique Index 落地

BizDealer.is_default 的 partial unique index：
1. 先运行 prisma migrate dev --name add_dealer_default_unique_index --create-only
2. 编辑生成的 migration SQL 文件，替换为：
   CREATE UNIQUE INDEX uk_dealer_default ON biz_dealer (is_default) WHERE is_default = true;
3. 运行 prisma migrate dev 应用

在 schema.prisma 中通过注释标记（当前已有），不依赖 Prisma 原生支持。

### 5.3 GIN Index 落地

BizSn.abnormal_flags 的 GIN 索引：
1. 同上流程，--create-only 生成空 migration
2. 编辑 SQL：
   CREATE INDEX idx_sn_abnormal ON biz_sn USING GIN (abnormal_flags);
3. prisma migrate dev 应用

### 5.4 Seed 数据执行规则

- seed 文件必须幂等（当前 seed.ts 使用 upsert，已满足）
- 执行命令：npm run db:seed（根级脚本代理）
- 首次部署流程：migrate deploy → seed
- 后续更新：seed 新增数据用 upsert，不删除旧数据
- 环境区分：seed 数据全环境一致，不按 env 区分

### 5.5 schema.prisma 命名规范

| 规则 | 说明 | 示例 |
|---|---|---|
| Model 命名 | PascalCase，前缀区分模块 | SysUser、BizPurchaseOrder |
| 表名映射 | snake_case，@@map("sys_user") | 全部 model 必须有 @@map |
| 字段命名 | camelCase | purchaseDate |
| 列名映射 | snake_case，@map("purchase_date") | 全部字段必须显式 @map |
| 主键 | UUID，@id @default(uuid()) @db.Uuid | 统一 UUID |
| 时间字段 | @db.Timestamptz，带 created_at/updated_at | 必须有时区 |
| 枚举字段 | @db.VarChar(n) 存储枚举值字符串 | 不使用 Prisma enum 类型 |
| JSON 字段 | Json 类型 | 快照、配置等 |
| 索引命名 | @@index([field]) 放在 model 底部 | 查询热字段必须加索引 |

---

## 六、Docker / 环境规范

### 6.1 Dev / Prod 命令

| 环境 | 命令 | 说明 |
|---|---|---|
| 开发（Docker） | docker compose -f docker-compose.dev.yml up | PG + Server(hot-reload) + Web(hot-reload) |
| 开发（本地） | npm run dev | 不依赖 Docker，直连本地 PG |
| 生产构建 | docker compose -f docker-compose.prod.yml build | 多阶段构建 |
| 生产启动 | docker compose -f docker-compose.prod.yml up -d | PG + Server + Nginx |

### 6.2 .env.example 使用方式

当前已存在两份：
- 根目录 .env.example：生产环境模板（DATABASE_URL 指向 postgres:5432）
- server packages/server/.env.example：开发环境模板（DATABASE_URL 指向 localhost:5432）

规范：
- 开发者 clone 后 cp packages/server/.env.example packages/server/.env 即可启动
- 生产部署 cp .env.example .env.prod 后修改所有值
- .env 和 .env.prod 已在 .gitignore 中，不会提交
- 新增环境变量时，必须同步更新 .env.example 和注释

### 6.3 数据库初始化方式

首次搭建开发环境：
1. docker compose -f docker-compose.dev.yml up postgres -d
2. npm run db:migrate（prisma migrate dev）
3. npm run db:seed

生产部署：
1. docker compose -f docker-compose.prod.yml up -d
2. Server 容器启动脚本中加入 prisma migrate deploy
3. 首次部署后手动执行 seed

### 6.4 上传目录挂载策略

| 环境 | 策略 |
|---|---|
| 开发 | 本地目录挂载：./packages/server/uploads:/app/uploads |
| 生产 | Docker Named Volume：uploads_prod:/app/uploads |
| Nginx | 只读挂载：uploads_nginx:/usr/share/nginx/uploads:ro（prod） |

建议补充：
- Nginx 配置 /uploads/ location 直接返回静态文件，不经过 Server
- 上传文件通过 Server API 写入，Nginx 代理读取
- 文件大小限制由 UPLOAD_MAX_SIZE 环境变量控制

---

## 七、下一步建议执行顺序

### 7.1 第一批（优先，基础骨架补全）

| 序号 | 任务 | 原因 |
|---|---|---|
| 1 | TypeScript strict 补全（Server） | 后续所有代码的基础，越早补成本越低 |
| 2 | ESLint + Prettier 配置 | 统一代码风格，防止后续代码风格混乱 |
| 3 | Husky + lint-staged | 提交前自动检查，保证代码质量 |
| 4 | 统一响应格式（拦截器 + 异常过滤器） | 所有接口的基础包装 |
| 5 | 全局 ValidationPipe | 所有 DTO 校验的基础 |

### 7.2 第二批（核心，鉴权+数据库）

| 序号 | 任务 | 原因 |
|---|---|---|
| 6 | JWT 鉴权模块 | 后续所有业务接口需要鉴权 |
| 7 | 权限 Guard + 装饰器 | 依赖鉴权模块 |
| 8 | Migration 落地（partial unique index、GIN index） | 数据库约束完整性 |
| 9 | Prisma Service 模块 | 统一数据库连接管理 |
| 10 | 操作日志拦截器骨架 | 拦截器和装饰器结构预留 |

### 7.3 第三批（前端基础）

| 序号 | 任务 | 原因 |
|---|---|---|
| 11 | Axios 封装 + Token 管理 | 前端请求基础设施 |
| 12 | 路由守卫 + Layout 骨架 | 页面框架 |
| 13 | React Query 全局配置 | 数据请求规范 |
| 14 | StatusTag 映射组件 | 多页面复用 |
| 15 | Zustand Store 建立（auth + layout） | 仅客户端状态 |

### 7.4 暂缓

| 任务 | 原因 |
|---|---|
| commitlint（commit message 规范） | 当前阶段 commit 简单，暂无必要 |
| Ant Design 主题定制 | 无暗色主题需求，默认主题够用 |
| E2E 测试框架 | 业务逻辑未开发，无法编写 E2E |
| CI/CD 流水线 | 部署方式未最终确定 |
| 前端 i18n | 当前仅中文，无多语言需求 |

### 7.5 需要 Codex 审计的项

| 项 | 审计重点 |
|---|---|
| schema.prisma | 字段类型、索引合理性、partial unique index 方案可行性 |
| 权限模型 | RBAC 三角色设计是否满足业务需求，权限粒度是否合理 |
| seed.ts | 初始数据完整性、admin 默认密码安全性 |
| Docker Compose | 网络隔离、端口暴露、volume 权限 |
| 响应格式 + 错误码体系 | 错误码区间划分是否合理 |

---

> 以上为方案全文，共 7 大板块 5 部分，待确认后按执行顺序逐步落地。
