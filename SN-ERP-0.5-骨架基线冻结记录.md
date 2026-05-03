# SN ERP 0.5 骨架基线冻结记录

## 一、文档信息

| 项目 | 内容 |
|------|------|
| 文档名称 | SN ERP 0.5 骨架基线冻结记录 |
| 版本号 | 0.5-H-1 |
| 更新时间 | 2026-04-27 |
| 适用范围 | SN ERP 骨架阶段（0.5-A ~ 0.5-H-1） |
| 状态 | 冻结，已验证 |
| 冻结结论 | 通过 |
| 冻结日期 | 2026-04-27 |

---

## 二、背景说明

本记录用于归档 SN ERP 项目骨架阶段（0.5 阶段）的全部已完成工作、审计结论、冻结内容和保留项。骨架阶段的目标是建立项目基础设施、Docker 化部署、前后端构建闭环，确保后续业务模块开发有一个稳定、可复现的基线。

---

## 三、已完成内容（0.5-A ~ 0.5-H-1）

### 3.1 0.5-A：项目初始化与骨架搭建

| 编号 | 内容 | 状态 |
|------|------|------|
| A-1 | 创建项目根目录结构（monorepo） | 已完成 |
| A-2 | 初始化 `package.json`（npm workspaces） | 已完成 |
| A-3 | 创建 `packages/server`（NestJS 后端） | 已完成 |
| A-4 | 创建 `packages/web`（React + Vite 前端） | 已完成 |
| A-5 | 配置 TypeScript、Prettier；ESLint 脚本占位，后续完善 | 已完成 |
| A-6 | 初始化 Prisma ORM（schema 占位） | 已完成 |
| A-7 | 配置 Ant Design 前端 UI 框架 | 已完成 |

### 3.2 0.5-B：Docker 化部署骨架

| 编号 | 内容 | 状态 |
|------|------|------|
| B-1 | 编写 `Dockerfile`（多 stage：deps / dev / dev-web / builder / server-prod / web-prod） | 已完成 |
| B-2 | 编写 `docker-compose.dev.yml`（开发环境） | 已完成 |
| B-3 | 编写 `docker-compose.prod.yml`（生产环境） | 已完成 |
| B-4 | 编写 `nginx.conf`（反向代理 + SPA fallback） | 已完成 |
| B-5 | 配置 `.env.example`（环境变量模板） | 已完成 |

### 3.3 0.5-C：构建脚本与 CI 骨架

| 编号 | 内容 | 状态 |
|------|------|------|
| C-1 | 根 `package.json` 添加 workspace scripts（build:server / build:web / dev:server / dev:web） | 已完成 |
| C-2 | 配置 `tsconfig.json` 项目引用（后续在 F 中简化） | 已完成 |
| C-3 | 验证 `npm run build:server` 通过 | 已完成 |
| C-4 | 验证 `npm run build:web` 通过 | 已完成 |

### 3.4 0.5-D：Prisma 基础配置

| 编号 | 内容 | 状态 |
|------|------|------|
| D-1 | 配置 `packages/server/prisma/schema.prisma`（datasource + generator，仅此二块，无 model） | 已完成 |
| D-2 | 添加 `prisma/seed.ts` | 未实现，计划在第 1 步处理 |
| D-3 | 后端集成 Prisma Client（`PrismaService`） | 未实现，计划在第 1 步处理 |
| D-4 | 配置 `DATABASE_URL` 环境变量读取 | 已完成（`.env.example` 中定义） |

> 注：0.5 骨架阶段 Prisma schema 只有 `generator` + `datasource`，无任何业务 model，无任何业务表。

### 3.5 0.5-E：骨架阻断问题修复

| 编号 | 内容 | 状态 | 修复的阻断项 |
|------|------|------|-------------|
| E-1 | 执行 `npm install` 生成 `package-lock.json` | 已完成 | 阻断 #1：Docker `npm ci` 需要 lockfile |
| E-2 | 新建 `.dockerignore` | 已完成 | 阻断 #3：排除 node_modules / dist / .env / uploads / docs |
| E-3 | 重写 `Dockerfile` | 已完成 | 阻断 #2（前端 dist）、#4（uploads 权限）、#5（node_modules 路径） |
| E-4 | 重写 `nginx.conf` | 已完成 | 阻断 #2：`/` 改为 SPA fallback |
| E-5 | 修改 `docker-compose.prod.yml` | 已完成 | 阻断 #2：nginx 改为 build 模式 |
| E-6 | 微调 `docker-compose.dev.yml` | 已完成 | 密码改用环境变量 + 默认值 |

### 3.6 0.5-F：生产与构建闭环加固

| 编号 | 内容 | 状态 | 修复的风险 |
|------|------|------|-----------|
| F-1 | `Dockerfile` 移除 `prisma generate \|\| true` | 已完成 | 高风险 #2：Prisma generate 失败被掩盖 |
| F-2 | `Dockerfile` builder / dev stage 加 `ENV DATABASE_URL=dummy` | 已完成 | 支撑 F-1 |
| F-3 | `Dockerfile` web-prod stage 改为非 root（`USER nginx`） | 已完成 | 高风险 #1：Nginx 以 root 运行 |
| F-4 | `Dockerfile` web-prod stage 预设目录权限 | 已完成 | 支撑 F-3 |
| F-5 | `nginx.conf` `listen 80` → `listen 8080` | 已完成 | 支撑 F-3（非 root 不能绑定 80） |
| F-6 | `nginx.conf` 新增 `location = /api { return 308 /api/; }` | 已完成 | /api 无尾斜杠处理 |
| F-7 | `docker-compose.prod.yml` nginx ports `80:80` → `80:8080` | 已完成 | 支撑 F-5 |
| F-8 | `docker-compose.prod.yml` 关键变量强制校验（`${VAR:?message}`） | 已完成 | 空值继续启动 |
| F-9 | 简化 `tsconfig.json`（移除项目 references，修复 TS 编译） | 已完成 | 支撑构建闭环 |
| F-10 | 修复 `MainLayout.tsx` 中 `type: 'divider'` TS 类型错误 | 已完成 | 支撑构建闭环 |
| F-11 | 删除冗余 `tsconfig.app.json` | 已完成 | 清理 |
| F-12 | `.gitignore` 补充 `dist-node/` | 已完成 | 清理 |

### 3.7 0.5-H：Docker 真实验证

| 编号 | 内容 | 状态 | 说明 |
|------|------|------|------|
| H-1 | 修复 `Dockerfile` server-prod 用户创建（`-S` 自动分配 UID/GID） | 已完成 | 避免 gid 1000 冲突 |
| H-2 | 删除 `docker-compose.prod.yml` server `user: '1000:1000'` | 已完成 | Dockerfile 中已设定 `USER sn-erp` |
| H-3 | `docker-compose.prod.yml` nginx 端口映射改为 `18080:8080` | 已完成 | 避免本机 80 端口冲突 |
| H-4 | 修复 `Dockerfile` web-prod nginx pid 路径 `/tmp/nginx.pid` | 已完成 | 非 root 用户无 `/run` 写权限 |
| H-5 | 本地 Docker build/up 真实验证 | 已完成 | 镜像构建成功，whoami 输出 `nginx`，`curl` 验证通过 |

### 3.8 0.5-H-1：骨架收口修复

| 编号 | 内容 | 状态 | 说明 |
|------|------|------|------|
| H1-1 | 删除 `prisma/schema.prisma` 中的 Placeholder model | 已完成 | 避免污染 0.5 骨架基线，不生成任何业务表 |
| H1-2 | `Dockerfile` 所有 `prisma generate` 加 `--allow-no-models` | 已完成 | Prisma 5.22.0 原生支持 |
| H1-3 | 本地 + Docker 构建验证 | 已完成 | 全部通过 |

---

## 四、审计结论

### 4.1 审计来源

| 审计轮次 | 审计人 | 日期 | 范围 |
|---------|--------|------|------|
| 0.5-E 审计 | Codex | 2026-04-27 | Dockerfile、nginx.conf、docker-compose.prod.yml、docker-compose.dev.yml、.dockerignore |
| 0.5-F 审计 | Codex | 2026-04-27 | Dockerfile、nginx.conf、docker-compose.prod.yml、tsconfig、构建闭环 |
| 0.5-H 审计 | Codex | 2026-04-27 | Docker 真实验证修复 |
| 0.5-H-1 审计 | Codex | 2026-04-27 | 骨架收口修复（Placeholder model 移除） |

### 4.2 审计结论

| 审计轮次 | 阻断级 | 高风险 | 中风险 | 低风险 | 结论 |
|---------|--------|--------|--------|--------|------|
| 0.5-E 审计 | 0 | 2 | 3 | — | 通过，需加固 |
| 0.5-F 审计 | 0 | 0 | 2 | 3 | 通过，带保留项 |
| 0.5-H 审计 | 0 | 0 | 0 | 1 | 通过 |
| 0.5-H-1 审计 | 0 | 0 | 0 | 0 | 通过 |

### 4.3 当前结论

**阻断级、高风险、中风险已全部清零。** 保留项 R-1 ~ R-3 已在 0.5-H 中关闭或降级。

---

## 五、冻结结论

### 5.1 冻结版本号

**0.5-H-1**

### 5.2 冻结日期

**2026-04-27**

### 5.3 冻结结论

**通过。**

骨架阶段基础设施已建立完毕，Docker 生产/构建闭环已在真实 Docker 环境中验证通过。Prisma schema 仅含 generator + datasource，无任何业务 model，无任何业务表，基线干净。

### 5.4 已确认功能

- 项目 monorepo 结构（npm workspaces）
- NestJS 后端骨架：Prisma schema 基础占位已创建（仅 generator + datasource）；JWT 相关依赖已安装，但 Auth / JWT 模块未创建；AppModule 当前为空根模块
- React + Vite 前端骨架（Ant Design、路由、布局）
- Docker 多 stage 构建（deps / dev / dev-web / builder / server-prod / web-prod）
- Docker Compose 开发环境（postgres + server + web）
- Docker Compose 生产环境（postgres + server + nginx），本地验证端口 `18080:8080`（生产正式端口可按部署环境调整为 80/443）
- Nginx 反向代理（非 root 运行，listen 8080，/api/ → server:3000，/uploads/ → 静态文件，SPA fallback，/api 308 重定向）
- 关键变量强制校验（`${VAR:?message}`）
- 环境变量模板（.env.example）
- 前后端构建脚本（npm run build:server / build:web）
- `prisma generate --allow-no-models` 通过（无 model 也可生成 Client 代码）

### 5.5 已确认业务规则

骨架阶段不涉及业务规则，以下规则为长期保留规则（已在 0.5 阶段建立的基础设施中预留支撑）：

1. SN 必须全局唯一。
2. 采购单保存后先进入草稿，不直接影响库存。
3. 只有执行确认入库后，才正式影响库存。
4. 涉及 SN 的入库动作，必须先做系统校验。
5. 图片识别后的 SN 结果必须先通过系统校验。
6. 识别失败或格式异常时，允许人工修正后再保存。
7. 入库动作需要二次确认。
8. 商品编码手工录入，一个编码只能对应一个商品。
9. 商品编码未被业务单据引用前，可以由管理员修改。
10. 商品编码一旦被业务单据引用，不允许修改。
11. 历史单据中的商品编码、商品名称、颜色等展示字段应保留当时快照。
12. 价格修改权限需要由管理员控制开启或关闭。
13. 导出模板第 1 到第 6 行不允许修改，只允许补齐第 6 行以后的内容。
14. 预览表可以展示非促销价总金额、促销价总金额、合计总金额。
15. 如果汇总位于全局查询中，则这些汇总可用于导出。
16. 失败、红冲、回退等异常操作必须保留原因、时间线和操作人。
17. 回退数量必须一致。
18. 回退涉及 SN 时，SN 必须完整。
19. 失败处理应提示原因，并允许创建工单人工处理。
20. 异常红冲记录应支持时间线视图和仅看异常过滤。
21. 库存查询、SN 全局查询、全局进销查询必须明确时间口径、金额口径、汇总口径、明细口径。
22. 不允许把草稿单据计入正式库存。
23. 不允许无痕删除业务记录。
24. 不允许只靠前端做权限控制。
25. 不允许没有事务保护地修改库存、金额、SN 状态。
26. 不允许只改库存数量、不写库存流水。
27. 不允许只改 SN 状态、不写 SN 时间线。
28. 不允许导出逻辑破坏模板第 1 到第 6 行。
29. 不允许历史单据展示字段跟随主数据变化而改变。

> 注：多账套（多租户）相关规则不在本阶段落地。如后续进入多账套版本，再另行裁决。

### 5.6 已确认字段

骨架阶段无业务字段，以下为基础设施相关字段：

| 字段 | 含义 | 位置 |
|------|------|------|
| DATABASE_URL | 数据库连接字符串 | 环境变量 |
| JWT_SECRET | JWT 签名密钥 | 环境变量 |
| JWT_REFRESH_SECRET | JWT Refresh Token 签名密钥 | 环境变量 |
| POSTGRES_PASSWORD | PostgreSQL 密码 | 环境变量 |
| POSTGRES_DB | PostgreSQL 数据库名 | 环境变量 |
| NODE_ENV | 运行环境 | 环境变量 |
| PORT | 后端服务端口 | 环境变量 |
| LOG_LEVEL | 日志级别 | 环境变量 |
| CORS_ORIGIN | 前端跨域来源 | 环境变量 |
| VITE_API_BASE_URL | 前端 API 基础地址 | 环境变量 |

### 5.7 已确认接口

骨架阶段无业务接口。API 前缀 `/api` 已在 Nginx 中配置，但后端暂无业务接口或健康检查接口。

### 5.8 已确认页面

以下为前端已配置的路由占位，仅包含基础布局和空页面，无业务逻辑：

| 页面 | 路径 | 说明 |
|------|------|------|
| 首页 / Dashboard | / | 系统首页骨架 |
| 采购管理 | /purchase | 采购入库占位 |
| 库存查询 | /inventory | 库存查询占位 |
| SN 管理 | /sn | SN 管理占位 |
| 商品管理 | /product | 商品管理占位 |
| 供应商管理 | /supplier | 供应商管理占位 |
| 系统设置 | /system | 系统设置占位 |
| 404 页 | * | 未匹配路由占位 |

### 5.9 已确认权限

权限系统未实现。后续必须按 V6 / V1.2 的三角色落地：

| 角色 | 说明 |
|------|------|
| 管理员 | 系统管理员 |
| 操作员 | 日常业务操作 |
| 商务 | 销售 / 采购 / 价格相关 |

---

## 六、保留项

| 编号 | 保留项 | 等级 | 说明 | 状态 |
|------|--------|------|------|------|
| R-1 | Docker 生产 build / up 未在真实 Docker 环境验证 | — | 已在 0.5-H 完成验证，验证端口 `18080:8080` | ✅ 已关闭 |
| R-2 | Nginx `/var/run` chown 可能存在启动问题 | — | 已改为 pid `/tmp/nginx.pid`，非 root 启动验证通过 | ✅ 已关闭 |
| R-3 | `npm run build:web` TypeScript 配置兼容问题 | 低 | 已通过简化 tsconfig 修复，后续需观察是否复现 | 持续观察 |

### 6.1 已验证的 Docker 命令

```bash
cd /Users/zh/Documents/trae_projects/TSN

# 1. 构建全部镜像
POSTGRES_PASSWORD=test JWT_SECRET=test JWT_REFRESH_SECRET=test \
  docker compose -f docker-compose.prod.yml build --no-cache

# 2. 启动服务
POSTGRES_PASSWORD=test JWT_SECRET=test JWT_REFRESH_SECRET=test \
  docker compose -f docker-compose.prod.yml up -d

# 3. 验证 Nginx 非 root
POSTGRES_PASSWORD=test JWT_SECRET=test JWT_REFRESH_SECRET=test \
  docker compose -f docker-compose.prod.yml exec nginx whoami
# 期望输出: nginx

# 4. 验证前端页面
curl -I http://localhost:18080/
# 期望: HTTP/1.1 200 OK

# 5. 验证 /api 308 重定向
curl -I http://localhost:18080/api
# 期望: HTTP/1.1 308 Permanent Redirect

# 6. 停止并清理
POSTGRES_PASSWORD=test JWT_SECRET=test JWT_REFRESH_SECRET=test \
  docker compose -f docker-compose.prod.yml down
```

---

## 七、不进入本版本的内容

| 内容 | 说明 |
|------|------|
| 业务模块实现 | 出库、上报、红冲、销售/退货等暂缓模块；第一阶段业务范围以 V1.2 为准，详见第 8 节 |
| Prisma schema 业务表 / 业务 model | `schema.prisma` 仅含 generator + datasource，无任何 model |
| 前端业务页面 | 采购管理、入库管理、库存查询等。当前仅路由占位，无业务逻辑 |
| 图片识别（OCR） | SN 图片识别功能 |
| 导出模板 | Excel 导出功能 |
| 电商运营监控 | 运营数据监控面板 |

---

## 八、后续版本计划

| 版本 | 内容 | 状态 |
|------|------|------|
| 0.6 | 第 1 步：V1.2 复审 5 个优化项（Prisma schema 变更 + 后端校验逻辑） | 待开始 |
| 0.7 | 第 2 步：V1.2 第一阶段 7 个模块开发 | 待规划 |
| 0.8 | 第 3 步：测试、验收、文档完善 | 待规划 |
| 0.9 | 第 4 步：性能优化与部署上线 | 待规划 |
| 1.0 | 正式发布 | 待规划 |

### 8.1 V1.2 第一阶段 7 个模块 + 轻量 Dashboard

1. 登录认证
2. 权限账号管理
3. 商品管理
4. 基础资料配置
5. 供应商管理
6. 采购入库
7. 库存查询 + SN 上传补录
8. 轻量 Dashboard

---

## 九、变更记录

| 版本 | 日期 | 变更内容 | 负责人 |
|------|------|---------|--------|
| 0.5-A | 2026-04-27 | 项目初始化与骨架搭建 | backend |
| 0.5-B | 2026-04-27 | Docker 化部署骨架 | backend |
| 0.5-C | 2026-04-27 | 构建脚本与 CI 骨架 | backend |
| 0.5-D | 2026-04-27 | Prisma 基础配置 | backend |
| 0.5-E | 2026-04-27 | 骨架阻断问题修复 | backend |
| 0.5-F | 2026-04-27 | 生产与构建闭环加固 | backend |
| 0.5-F-1 | 2026-04-27 | 文档修正：对齐真实骨架状态（Codex 审计） | doc |
| 0.5-H | 2026-04-27 | Docker 真实验证修复（UID 冲突、80 端口冲突、nginx pid 非 root） | backend |
| 0.5-H-1 | 2026-04-27 | 骨架收口：移除 Placeholder model，改用 `--allow-no-models` | backend |

---

## 十、下一步建议

1. **直接进入第 1 步**：开始 V1.2 复审 5 个优化项（Prisma schema 变更 + 后端校验逻辑）。Docker 真实验证已通过，无需再补跑。
2. **生产部署时注意**：将 `docker-compose.prod.yml` 中 nginx 端口从 `18080:8080` 调整为 `80:8080` 或 `443:8080`（按部署环境）。
3. **更新 .env 文件**：根据 `.env.example` 配置真实环境变量，确保生产部署时变量校验通过。

---

*本文档由 doc（文档整理员）整理，基于 chief（总控架构师）、backend（后端工程师）、Codex（代码审计员）的输出。*
