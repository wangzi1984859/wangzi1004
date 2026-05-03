# SN ERP 启动方案 V1.2 实施合并版 - 复审修订说明

> 基于：Codex 第二轮复审报告
> 日期：2026-04-26
> 状态：创建骨架前已确认，业务表迁移前必须落实

---

## 一、复审通过结论

1. V1.2 可作为唯一施工依据。
2. 后续开发只引用 V1.2，不再引用 V1.1。
3. 允许进入项目骨架创建阶段。
4. 正式业务表迁移落地前，必须落实以下 5 个优化项。

---

## 二、5 个优化项

### 优化项 1：制单人与确认人分离规则强制化

**标记：创建骨架前已确认，业务表迁移前必须落实**

规则：
- 操作员不得确认自己创建的采购单。
- 管理员可以确认自己或他人创建的采购单，不受此限制。
- 后端确认入库接口必须校验：若当前用户角色为操作员且 `confirmed_by == created_by`，拒绝执行。
- 前端确认入库弹窗中应提示确认人规则。
- 操作日志必须记录 `created_by` 与 `confirmed_by`。

涉及变更：
- 后端 `POST /api/purchase-orders/:id/confirm` 接口增加校验逻辑
- 前端确认入库弹窗增加规则提示文案
- 操作日志 detail 字段记录 created_by 和 confirmed_by

---

### 优化项 2：默认经销商数据库唯一约束

**标记：创建骨架前已确认，业务表迁移前必须落实**

规则：
- `biz_dealer.is_default` 表示默认经销商。
- 同一系统第一阶段只允许一个默认经销商。
- 数据库层增加部分唯一索引：`CREATE UNIQUE INDEX uk_dealer_default ON biz_dealer (is_default) WHERE is_default = true;`
- 修改默认经销商时必须事务处理（先将旧默认改为 false，再将新默认改为 true）。
- 修改默认经销商必须记录操作日志。

涉及变更：
- Prisma schema `biz_dealer` 表增加部分唯一索引
- 后端经销商更新接口增加事务处理
- 种子数据确保只有一条 `is_default = true`

---

### 优化项 3：biz_inventory_summary 增加 pending_sn_quantity

**标记：创建骨架前已确认，业务表迁移前必须落实**

规则：
- `biz_inventory_summary` 增加字段：`pending_sn_quantity INT NOT NULL DEFAULT 0`
- 含义：待补录 SN 数量 = 采购数量 - 已录入 SN 数量（商品级汇总）
- 与 `total_quantity` / `available_quantity` / `sn_recorded_quantity` 一起事务更新。
- 库存页商品级快速查询可直接展示该字段，无需实时计算。
- 批次级明细仍通过采购明细 + SN 联查计算。

涉及变更：
- Prisma schema `biz_inventory_summary` 增加 `pending_sn_quantity` 字段
- 确认入库、SN 上传/补录时同步更新该字段

---

### 优化项 4：SN 导入错误增加 error_code

**标记：创建骨架前已确认，业务表迁移前必须落实**

规则：
- `biz_sn_import_error` 增加字段：`error_code VARCHAR(50) NOT NULL`
- 用于错误分类、筛选和统计。
- `error_reason` 保留为给用户看的详细原因。
- 枚举白名单：

| error_code | 含义 |
|-----------|------|
| DUPLICATE_SN | SN 码重复 |
| INVALID_FORMAT | SN 格式不合法 |
| PRODUCT_MISMATCH | SN 归属商品不匹配 |
| OVER_LIMIT | 超出导入数量限制 |
| EMPTY_SN | SN 码为空 |
| UNKNOWN_ERROR | 未知错误 |

涉及变更：
- Prisma schema `biz_sn_import_error` 增加 `error_code` 字段
- 后端 SN 批量导入逻辑使用 error_code 分类

---

### 优化项 5：Docker 生产环境非 root 强制化

**标记：创建骨架前已确认，业务表迁移前必须落实**

规则：
- Docker 生产环境容器必须使用非 root 用户运行。
- Dockerfile 中使用 `USER node` 或自定义非 root 用户。
- docker-compose.prod.yml 中明确指定 `user: "1000:1000"` 或等效配置。
- 如确有例外，必须在部署文档中记录原因。
- dev 环境可放宽，但 prod 必须强制。

涉及变更：
- 后端 Dockerfile 增加 `USER node`
- docker-compose.prod.yml 增加用户指定
- 前端 Dockerfile（Nginx）使用默认非 root 配置

---

## 三、下一步

完成本修订说明后，允许进入项目骨架创建阶段。

骨架创建要求：
1. 只创建项目骨架。
2. 不直接实现完整业务模块。
3. 不直接写复杂业务逻辑。
4. 不一次性生成所有代码。
5. 先初始化 monorepo、前端、后端、基础配置、README、.env.example、Docker Compose 初稿。
6. 骨架创建完成后先汇报目录结构和运行方式。
7. 不提交 Git，等用户确认。

---

**以上为《SN-ERP-启动方案-V1.2-实施合并版-复审修订说明》。等待用户确认是否开始创建骨架。**
