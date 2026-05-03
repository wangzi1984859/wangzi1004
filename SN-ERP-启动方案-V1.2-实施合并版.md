# SN ERP 启动方案 V1.2 实施合并版

> 基于：V1.1 已裁决版 + V1.1 修正收口版 + V1.1 最终确认版 + Codex 审计意见
> 版本：V1.2 实施合并版
> 日期：2026-04-26
> 状态：可直接施工依据，后续开发只引用本文件
> 说明：本文件已合并所有修正结果，不再保留 V1.1 旧表述

---

## 一、15 个已裁决问题（含 R-01）

### Q-01 生产部署方式
私有化部署第一阶段：Docker Compose 单机部署（Nginx + Node.js + PostgreSQL 15+）。不做 SaaS 多租户。dev/staging/prod 三套环境配置隔离，通过 `.env` 切换。Redis 预留配置项但不实现。

### Q-02 拼音码范围
- 供应商编码默认由供应商名称自动生成拼音首字母，允许管理员手工修正。
- 商品编码不使用拼音自动生成，商品编码按 Q-06 手工录入 + 格式校验。
- 商品名称可保留 pinyin_code 作为搜索辅助，pinyin_code 不设 UNIQUE 约束。
- 品类、品牌的 pinyin_code 也仅做搜索辅助，不设 UNIQUE。
- 经销商编码采用手工录入 + 唯一校验（R-01 裁决），不使用拼音自动生成。

### Q-03 文件存储位置
第一阶段本地文件存储，路径通过配置项管理。目录结构：`uploads/`、`imports/`、`exports/`、`templates/`、`ocr/`、`backups/`。数据库 `files` 表只存元信息+相对路径。预留 `IStorageAdapter` 接口。

### Q-04 导入文件大小上限
单文件 ≤ 10MB，单次 SN 导入 ≤ 5000 条。后端使用流式解析。

### Q-05 采购单号生成规则
格式：`CG + YYYYMMDD + NNNN`，每日 00:00 起从 0001 开始。系统自动生成，不可手动编辑。

### Q-06 商品编码规则
手工录入，1-50 位，允许字母、数字、中划线(`-`)、下划线(`_`)，禁止空格和中文。唯一约束。被业务单据引用前管理员可修改，引用后不可修改。

### Q-07 历史快照粒度
完整业务快照策略。每条采购明细/出库明细/上报明细记录创建时，同时生成快照字段存储商品快照、供应商快照、采购明细快照、上报/经销商快照。

### Q-08 库存汇总计算策略
采用三层组合策略（Codex 审计修正）：
1. **商品级库存汇总表** `biz_inventory_summary`：用于首页统计和商品级快速查询。
2. **采购明细/批次级库存查询**：通过采购明细表 + SN 表联合查询，支持供应商维度、采购批次维度、价格类型维度。
3. **库存流水表** `biz_inventory_log`：保留完整追溯依据。

库存金额必须追溯到采购明细批次，不使用 product_id 单维度汇总金额。待补录 SN 数量必须可计算、可筛选、可展示。可出库数量基于当前有效可提取 SN 数量，并受 `allow_sn_extract` 影响。

### Q-09 OCR 服务商选择
第一阶段仅预留 `IOCRService` 适配器接口，默认关闭。

### Q-10 密码复杂度规则
初始密码 `yytx0401@`，首次登录强制改密。至少 8 位，必须含字母和数字。不允许与初始密码相同，不允许与登录账号相同。

### Q-11 前端是否支持移动端
第一阶段仅桌面端，1440px+。

### Q-12 第一阶段测试策略
后端必测（单元+集成），前端手工验收。

### Q-13 操作员库存权限
操作员可查看库存列表、导出库存数据、补录 SN。不可直接修改库存数量/金额字段。

### Q-14 初始化种子数据
种子脚本覆盖：三类角色 + 权限组 + 管理员账号 + 示例品类（≥3）+ 示例品牌（≥3）+ 采购价格类型（原价/特价/一口价）+ 自定义列枚举（≥3）+ 默认经销商（≥1）。

### R-01 经销商编码生成规则
经销商编码采用手工录入 + 唯一校验。不使用拼音自动生成。必填且唯一。用于上报、导出、外部模板和历史快照。被业务引用后不允许修改。历史单据保留快照。经销商名称可保留 pinyin_code 作为搜索辅助，不做 UNIQUE。

---

## 二、采购确认入库权限裁决（V1.2 补充裁决）

本裁决作为对 V6 的正式补充裁决，不直接修改 V6 冻结正文，在 V1.2 中高优先级执行。

### 规则

1. 管理员默认拥有采购确认入库权限。
2. 操作员默认不拥有采购确认入库权限。
3. 管理员可以单独授予某个操作员 `purchase:confirm` 权限。
4. 只有具备 `purchase:confirm` 权限的账号，才能执行采购确认入库。
5. 商务永远不可确认入库。
6. 后端必须校验 `purchase:confirm` 权限，不能只靠前端按钮控制。
7. 前端必须二次确认。
8. 确认入库必须记录操作日志。
9. 确认入库后采购单状态从草稿变为已入库，库存正式生效。
10. 确认入库后不允许继续按草稿逻辑编辑。

### 制单人与确认人分离（建议启用）

- 如果采购单由该操作员本人创建，则本人不得确认自己的采购单。
- 管理员不受此限制。
- 第一阶段如实现成本过高，至少预留该规则，标注为建议启用。
- `biz_purchase_order` 表需增加 `created_by` 字段（已有），确认入库时后端校验 `confirmed_by != created_by`（仅对操作员生效）。

### 协作规则冲突标注

《SN-ERP-opencode-Codex开发协作规则-V1.0.md》中"确认入库：管理员"的表述，已由本 V1.2 补充裁决覆盖。施工时以本文件为准。

---

## 三、V6 冻结业务规则（10 条核心红线）

1. **三层角色不可变更**：商务、操作员、管理员，角色权限矩阵不可增删改。
2. **SN 全局唯一**：归属到采购明细行，可追溯到商品/供应商/采购单/明细/价格。
3. **库存三列分离**：库存数量 ≠ 可出库数量 ≠ 已录入 SN 数量。
4. **保存 ≠ 确认入库**：采购保存=草稿不影响库存；具备权限的账号确认入库后库存生效；出库保存=草稿/待确认，确认出库后库存出库生效；转入上报则按 V6 上报链路执行。采购确认入库和出库确认是两个独立动作，不混写。
5. **历史快照不跟随主数据变化**。
6. **上报单号 = 发货日期范围最后一天**。
7. **红冲单号自成一派**：`HC + YYYYMMDD + NNNN`。
8. **上报确认 ≠ 生成文件**。
9. **价格修改权限默认关闭**。
10. **正式导出严格按 FCS 模板**。

---

## 四、UI 设计基线（合并 V1.0 + V1.1）

参见《SN-ERP-UI设计基线-V1.1-补强版.md》（优先于 V1.0）。

核心要点：
- 侧边栏 #0F172A，主色 #2563EB
- 30+ 状态标签统一枚举映射（V1.1 补充至 67 个状态）
- 三级确认弹窗 L1/L2/L3
- 确认入库按钮与保存草稿按钮视觉区分（V1.1 新增）
- 轻量 Dashboard（V1.1 修正：复杂图表暂缓）
- 权限感知按钮规范（V1.1 新增）
- 34 条 UI 验收标准

---

## 五、第一阶段开发范围

### 必须实现的模块（7 个模块 + 登录）

| 序号 | 模块 | 范围说明 |
|------|------|---------|
| 1 | 登录认证 | 账号密码登录、JWT、首次登录强制改密 |
| 2 | 权限账号管理 | 用户 CRUD、角色分配、`purchase:confirm` 独立权限点、密码重置 |
| 3 | 商品管理 | 商品 CRUD、编码唯一校验、pinyin_code 搜索辅助（无 UNIQUE） |
| 4 | 基础资料配置 | 品类、品牌、采购自定义字段枚举、经销商管理 |
| 5 | 供应商管理 | 供应商 CRUD、`allow_sn_extract` 字段、编码拼音生成+可修正 |
| 6 | 采购入库 | 新建采购单、草稿保存、确认入库（`purchase:confirm` 权限校验）、快照、库存更新 |
| 7 | 库存查询 + SN 上传补录 | 库存列表（三列分离+批次维度）、SN 上传/补录、SN 全局唯一校验 |

### 轻量 Dashboard
欢迎区 + 快捷入口 + 简单统计卡片 + 待处理事项占位。复杂图表暂缓。

### 暂缓范围
出库、上报、SN 全局查询、全局进销查询、红冲、复杂统计图表、OCR 接入、Redis。

---

## 六、技术栈

### 前端
React 18+ / TypeScript 5+ / Vite 5+ / Ant Design 5+ / Zustand 4+ / React Query 5+ / Tailwind CSS 3+ / Lucide React / React Router 6+

### 后端
Node.js 20 LTS / NestJS 10+ / TypeScript 5+ / Prisma 5+ / PostgreSQL 15+ / JWT + Refresh Token / class-validator / ExcelJS / pinyin / multer / @nestjs/swagger / nestjs-pino

### 部署
Docker + Docker Compose / Nginx / 本地文件系统 + IStorageAdapter

---

## 七、数据库核心表（18 张 + 2 张预留 + 2 张 SN 导入辅助）

### 表 1：sys_user（系统用户表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK, DEFAULT gen_random_uuid() | 主键 |
| username | VARCHAR(50) | NOT NULL, UNIQUE | 登录账号 |
| password_hash | VARCHAR(255) | NOT NULL | bcrypt 哈希 |
| real_name | VARCHAR(50) | NOT NULL | 真实姓名 |
| phone | VARCHAR(20) | NULLABLE | 手机号 |
| email | VARCHAR(100) | NULLABLE | 邮箱 |
| is_active | BOOLEAN | NOT NULL, DEFAULT TRUE | 是否启用 |
| is_first_login | BOOLEAN | NOT NULL, DEFAULT TRUE | 是否首次登录 |
| last_login_at | TIMESTAMPTZ | NULLABLE | 最后登录时间 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | - |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | - |
| created_by | UUID | NULLABLE, FK → sys_user.id | 创建人 |
| updated_by | UUID | NULLABLE, FK → sys_user.id | 修改人 |

**索引**：`uk_username` UNIQUE(username)；`idx_user_is_active`(is_active)

### 表 2：sys_role（系统角色表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| code | VARCHAR(30) | NOT NULL, UNIQUE | ADMIN / OPERATOR / VIEWER |
| name | VARCHAR(50) | NOT NULL | 角色名称 |
| description | VARCHAR(200) | NULLABLE | - |
| is_system | BOOLEAN | NOT NULL, DEFAULT FALSE | 系统内置不可删 |
| created_at | TIMESTAMPTZ | NOT NULL | - |
| updated_at | TIMESTAMPTZ | NOT NULL | - |

### 表 3：sys_permission（系统权限表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| code | VARCHAR(50) | NOT NULL, UNIQUE | 如 purchase:confirm |
| name | VARCHAR(100) | NOT NULL | 权限名称 |
| module | VARCHAR(30) | NOT NULL | 所属模块 |
| description | VARCHAR(200) | NULLABLE | - |

### 表 4：sys_role_permission（角色权限关联表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| role_id | UUID | NOT NULL, FK → sys_role.id | 角色 |
| permission_id | UUID | NOT NULL, FK → sys_permission.id | 权限 |

**索引**：`uk_role_perm` UNIQUE(role_id, permission_id)

### 表 5：biz_product（商品表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| code | VARCHAR(50) | NOT NULL, UNIQUE | 商品编码（手工录入） |
| name | VARCHAR(200) | NOT NULL | 商品名称 |
| pinyin_code | VARCHAR(50) | NULLABLE | 搜索辅助，无 UNIQUE |
| color | VARCHAR(50) | NULLABLE | 颜色 |
| category_id | UUID | NOT NULL, FK → biz_category.id | 品类 |
| brand_id | UUID | NULLABLE, FK → biz_brand.id | 品牌 |
| custom_attr_1 | VARCHAR(200) | NULLABLE | 自定义属性1 |
| custom_attr_2 | VARCHAR(200) | NULLABLE | 自定义属性2 |
| custom_attr_3 | VARCHAR(200) | NULLABLE | 自定义属性3 |
| is_active | BOOLEAN | NOT NULL, DEFAULT TRUE | 启用/停用 |
| is_referenced | BOOLEAN | NOT NULL, DEFAULT FALSE | 引用后编码不可改 |
| remark | TEXT | NULLABLE | 备注 |
| created_at | TIMESTAMPTZ | NOT NULL | - |
| updated_at | TIMESTAMPTZ | NOT NULL | - |
| created_by | UUID | NULLABLE | - |
| updated_by | UUID | NULLABLE | - |

**索引**：`uk_product_code` UNIQUE(code)；`idx_product_category`(category_id)；`idx_product_brand`(brand_id)

### 表 6：biz_category（品类表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| name | VARCHAR(100) | NOT NULL, UNIQUE | 品类名称 |
| pinyin_code | VARCHAR(50) | NULLABLE | 搜索辅助，无 UNIQUE |
| sort_order | INT | NOT NULL, DEFAULT 0 | 排序 |
| is_active | BOOLEAN | NOT NULL, DEFAULT TRUE | - |
| created_at | TIMESTAMPTZ | NOT NULL | - |
| updated_at | TIMESTAMPTZ | NOT NULL | - |

### 表 7：biz_brand（品牌表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| name | VARCHAR(100) | NOT NULL, UNIQUE | 品牌名称 |
| pinyin_code | VARCHAR(50) | NULLABLE | 搜索辅助，无 UNIQUE |
| sort_order | INT | NOT NULL, DEFAULT 0 | - |
| is_active | BOOLEAN | NOT NULL, DEFAULT TRUE | - |
| created_at | TIMESTAMPTZ | NOT NULL | - |
| updated_at | TIMESTAMPTZ | NOT NULL | - |

### 表 8：biz_product_snapshot（商品快照表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| product_id | UUID | NOT NULL, FK → biz_product.id | 商品ID |
| code | VARCHAR(50) | NOT NULL | 快照编码 |
| name | VARCHAR(200) | NOT NULL | 快照名称 |
| color | VARCHAR(50) | NULLABLE | - |
| category_id | UUID | NOT NULL | - |
| category_name | VARCHAR(100) | NOT NULL | - |
| brand_id | UUID | NULLABLE | - |
| brand_name | VARCHAR(100) | NULLABLE | - |
| custom_attr_1 | VARCHAR(200) | NULLABLE | - |
| custom_attr_2 | VARCHAR(200) | NULLABLE | - |
| custom_attr_3 | VARCHAR(200) | NULLABLE | - |
| snapshot_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 快照时间 |

**索引**：`idx_ps_product`(product_id)；`idx_ps_time`(snapshot_at DESC)

### 表 9：biz_purchase_field_enum（采购自定义字段枚举表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| field_key | VARCHAR(30) | NOT NULL | price_type / custom_col_1 / custom_col_2 / custom_col_3 |
| field_name | VARCHAR(50) | NOT NULL | 字段显示名 |
| enum_value | VARCHAR(100) | NOT NULL | 枚举值 |
| sort_order | INT | NOT NULL, DEFAULT 0 | - |
| is_active | BOOLEAN | NOT NULL, DEFAULT TRUE | - |
| created_at | TIMESTAMPTZ | NOT NULL | - |

**索引**：`uk_field_value` UNIQUE(field_key, enum_value)

### 表 10：biz_dealer（经销商表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| code | VARCHAR(50) | NOT NULL, UNIQUE | 经销商代码（手工录入） |
| name | VARCHAR(200) | NOT NULL | 经销商名称 |
| pinyin_code | VARCHAR(50) | NULLABLE | 搜索辅助，无 UNIQUE |
| contact_person | VARCHAR(50) | NULLABLE | 联系人 |
| contact_phone | VARCHAR(20) | NULLABLE | 联系电话 |
| address | VARCHAR(300) | NULLABLE | 地址 |
| is_active | BOOLEAN | NOT NULL, DEFAULT TRUE | - |
| is_referenced | BOOLEAN | NOT NULL, DEFAULT FALSE | 引用后 code 不可改 |
| is_default | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否默认经销商（全局唯一） |
| created_at | TIMESTAMPTZ | NOT NULL | - |
| updated_at | TIMESTAMPTZ | NOT NULL | - |

**索引**：`uk_dealer_code` UNIQUE(code)

### 表 11：biz_supplier（供应商表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| code | VARCHAR(50) | NOT NULL, UNIQUE | 供应商编码（拼音首字母生成+可修正） |
| name | VARCHAR(200) | NOT NULL | 供应商名称 |
| pinyin_code | VARCHAR(50) | NULLABLE | 搜索辅助，无 UNIQUE |
| contact_person | VARCHAR(50) | NULLABLE | 联系人 |
| contact_phone | VARCHAR(20) | NULLABLE | 联系电话 |
| address | VARCHAR(300) | NULLABLE | 地址 |
| allow_sn_extract | BOOLEAN | NOT NULL, DEFAULT TRUE | 是否允许提取该供应商名下 SN |
| is_active | BOOLEAN | NOT NULL, DEFAULT TRUE | 供应商启用/停用 |
| is_referenced | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否已被引用 |
| created_at | TIMESTAMPTZ | NOT NULL | - |
| updated_at | TIMESTAMPTZ | NOT NULL | - |
| created_by | UUID | NULLABLE | - |
| updated_by | UUID | NULLABLE | - |

**索引**：`uk_supplier_code` UNIQUE(code)

### 表 12：biz_purchase_order（采购主表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| order_no | VARCHAR(20) | NOT NULL, UNIQUE | CG+YYYYMMDD+NNNN |
| supplier_id | UUID | NOT NULL, FK → biz_supplier.id | 供应商 |
| supplier_snapshot | JSONB | NOT NULL | 供应商快照 |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'DRAFT' | DRAFT / CONFIRMED / RED_CANCELED |
| purchase_date | DATE | NOT NULL | 采购日期 |
| total_amount | DECIMAL(15,2) | NOT NULL, DEFAULT 0 | 总金额 |
| remark | TEXT | NULLABLE | 备注 |
| is_red_canceled | BOOLEAN | NOT NULL, DEFAULT FALSE | - |
| confirmed_at | TIMESTAMPTZ | NULLABLE | 确认入库时间 |
| confirmed_by | UUID | NULLABLE, FK → sys_user.id | 确认人 |
| created_at | TIMESTAMPTZ | NOT NULL | - |
| updated_at | TIMESTAMPTZ | NOT NULL | - |
| created_by | UUID | NOT NULL, FK → sys_user.id | 创建人 |

**索引**：`uk_po_no` UNIQUE(order_no)；`idx_po_status`(status)；`idx_po_supplier`(supplier_id)；`idx_po_date`(purchase_date DESC)

### 表 13：biz_purchase_detail（采购明细表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| purchase_order_id | UUID | NOT NULL, FK → biz_purchase_order.id | 采购单 |
| line_number | INT | NOT NULL | 行号 |
| product_id | UUID | NOT NULL, FK → biz_product.id | 商品 |
| product_snapshot | JSONB | NOT NULL | 商品快照 |
| quantity | INT | NOT NULL | 采购数量 |
| unit_price | DECIMAL(15,2) | NOT NULL | 单价 |
| total_price | DECIMAL(15,2) | NOT NULL | 小计 |
| price_type | VARCHAR(30) | NULLABLE | 原价/特价/一口价 |
| custom_col_1 | VARCHAR(200) | NULLABLE | 自定义列1 |
| custom_col_2 | VARCHAR(200) | NULLABLE | 自定义列2 |
| custom_col_3 | VARCHAR(200) | NULLABLE | 自定义列3 |
| sn_count | INT | NOT NULL, DEFAULT 0 | 已录入SN数量 |
| remark | VARCHAR(500) | NULLABLE | 备注 |

**索引**：`idx_pd_order`(purchase_order_id)；`idx_pd_product`(product_id)；`uk_pd_line` UNIQUE(purchase_order_id, line_number)

### 表 14：biz_sn（序列号表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| sn_code | VARCHAR(100) | NOT NULL, UNIQUE | SN 码（全局唯一） |
| purchase_detail_id | UUID | NOT NULL, FK → biz_purchase_detail.id | 归属采购明细 |
| product_id | UUID | NOT NULL, FK → biz_product.id | 商品 |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'IN_STOCK' | IN_STOCK / OUT_STOCK / REPORTED / RED_CANCELED / RETURNED |
| abnormal_flags | JSONB | NULLABLE | 异常标签（枚举白名单，GIN 索引） |
| is_supplement | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否补录 |
| supplement_by | UUID | NULLABLE, FK → sys_user.id | 补录人 |
| supplement_at | TIMESTAMPTZ | NULLABLE | 补录时间 |
| upload_batch_id | UUID | NULLABLE, FK → biz_sn_import_batch.id | 上传批次 |
| remark | VARCHAR(500) | NULLABLE | 备注 |
| created_at | TIMESTAMPTZ | NOT NULL | - |
| updated_at | TIMESTAMPTZ | NOT NULL | - |

**索引**：`uk_sn_code` UNIQUE(sn_code)；`idx_sn_detail`(purchase_detail_id)；`idx_sn_product`(product_id)；`idx_sn_status`(status)；`idx_sn_batch`(upload_batch_id)；`idx_sn_abnormal` GIN(abnormal_flags)

### 表 15：biz_sn_import_batch（SN 导入批次表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| batch_no | VARCHAR(20) | NOT NULL, UNIQUE | 导入批次号 |
| file_id | UUID | NULLABLE, FK → sys_file.id | 上传文件 |
| purchase_order_id | UUID | NULLABLE, FK → biz_purchase_order.id | 采购单 |
| purchase_detail_id | UUID | NULLABLE, FK → biz_purchase_detail.id | 采购明细 |
| total_count | INT | NOT NULL, DEFAULT 0 | 导入总数 |
| success_count | INT | NOT NULL, DEFAULT 0 | 成功数 |
| fail_count | INT | NOT NULL, DEFAULT 0 | 失败数 |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'PENDING' | PENDING / PROCESSING / COMPLETED / FAILED |
| error_summary | TEXT | NULLABLE | 错误摘要 |
| created_by | UUID | NOT NULL, FK → sys_user.id | 操作人 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | - |
| completed_at | TIMESTAMPTZ | NULLABLE | 完成时间 |

**索引**：`uk_batch_no` UNIQUE(batch_no)；`idx_bib_order`(purchase_order_id)；`idx_bib_detail`(purchase_detail_id)

### 表 16：biz_sn_import_error（SN 导入错误明细表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| batch_id | UUID | NOT NULL, FK → biz_sn_import_batch.id | 批次 |
| row_number | INT | NOT NULL | 行号 |
| sn_code | VARCHAR(100) | NULLABLE | SN 码 |
| error_reason | VARCHAR(500) | NOT NULL | 错误原因 |
| raw_data | TEXT | NULLABLE | 原始行数据 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | - |

**索引**：`idx_sie_batch`(batch_id)

### 表 17：biz_inventory_summary（库存汇总表 - 商品维度）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| product_id | UUID | NOT NULL, UNIQUE, FK → biz_product.id | 商品 |
| total_quantity | INT | NOT NULL, DEFAULT 0 | 库存总数量 |
| available_quantity | INT | NOT NULL, DEFAULT 0 | 可出库数量（有效可提取 SN，受 allow_sn_extract 影响） |
| sn_recorded_quantity | INT | NOT NULL, DEFAULT 0 | 已录入 SN 数量 |
| total_amount | DECIMAL(15,2) | NOT NULL, DEFAULT 0 | 库存总金额（批次口径） |
| last_in_at | TIMESTAMPTZ | NULLABLE | 最后入库时间 |
| last_out_at | TIMESTAMPTZ | NULLABLE | 最后出库时间 |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | - |

**索引**：`uk_inv_product` UNIQUE(product_id)

**说明**：商品级汇总用于首页统计和快速查询。库存金额、待补录 SN、供应商维度、价格类型维度等详细数据，通过采购明细表 + SN 表联合查询获取，不依赖本表单一维度。

### 表 18：biz_inventory_log（库存流水表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| product_id | UUID | NOT NULL, FK → biz_product.id | 商品 |
| biz_type | VARCHAR(30) | NOT NULL | PURCHASE_IN / STOCK_OUT / RED_CANCEL / SN_UPLOAD / SN_SUPPLEMENT |
| biz_id | UUID | NOT NULL | 业务单据ID |
| change_quantity | INT | NOT NULL | 正=入库，负=出库 |
| change_amount | DECIMAL(15,2) | NOT NULL, DEFAULT 0 | 变更金额 |
| before_quantity | INT | NOT NULL | 变更前 |
| after_quantity | INT | NOT NULL | 变更后 |
| before_sn_count | INT | NOT NULL, DEFAULT 0 | - |
| after_sn_count | INT | NOT NULL, DEFAULT 0 | - |
| remark | VARCHAR(500) | NULLABLE | - |
| operator_id | UUID | NOT NULL, FK → sys_user.id | 操作人 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | - |

**索引**：`idx_il_product`(product_id)；`idx_il_biz`(biz_type, biz_id)；`idx_il_time`(created_at DESC)

### 表 19：sys_operation_log（操作日志表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| user_id | UUID | NOT NULL, FK → sys_user.id | 操作用户 |
| username | VARCHAR(50) | NOT NULL | 冗余 |
| module | VARCHAR(30) | NOT NULL | 操作模块 |
| action | VARCHAR(50) | NOT NULL | CREATE / UPDATE / DELETE / CONFIRM / EXPORT / UPLOAD / SUPPLEMENT / FIX_ABNORMAL |
| target_type | VARCHAR(30) | NOT NULL | 目标类型 |
| target_id | UUID | NULLABLE | 目标ID |
| detail | JSONB | NULLABLE | 变更前后对比 |
| ip_address | VARCHAR(45) | NULLABLE | - |
| user_agent | VARCHAR(500) | NULLABLE | - |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | - |

**索引**：`idx_ol_user`(user_id)；`idx_ol_module`(module)；`idx_ol_action`(action)；`idx_ol_time`(created_at DESC)；`idx_ol_target`(target_type, target_id)

### 预留表 20：sys_file（文件管理表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| original_name | VARCHAR(255) | NOT NULL | 原始文件名 |
| stored_name | VARCHAR(255) | NOT NULL | 随机存储文件名 |
| stored_path | VARCHAR(500) | NOT NULL | 存储相对路径 |
| file_size | BIGINT | NOT NULL | 字节 |
| mime_type | VARCHAR(100) | NOT NULL | MIME 类型 |
| biz_type | VARCHAR(20) | NOT NULL | IMPORT / EXPORT / UPLOAD / OCR / TEMPLATE |
| biz_id | UUID | NULLABLE | 关联业务ID |
| created_at | TIMESTAMPTZ | NOT NULL | - |
| created_by | UUID | NULLABLE | - |

### 预留表 21：sys_config（系统配置表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 主键 |
| config_key | VARCHAR(50) | NOT NULL, UNIQUE | 配置键 |
| config_value | TEXT | NOT NULL | 配置值 |
| description | VARCHAR(200) | NULLABLE | - |
| updated_at | TIMESTAMPTZ | NOT NULL | - |
| updated_by | UUID | NULLABLE | - |

---

## 八、SN 异常标签枚举白名单

abnormal_flags 不允许自由文本，必须使用以下枚举：

| 枚举值 | 中文含义 | 说明 |
|--------|---------|------|
| DUPLICATE_RISK | 疑似重复 | SN 码与其他记录疑似重复 |
| OWNER_MISMATCH | 归属不一致 | SN 归属商品与实际记录不一致 |
| CHAIN_BROKEN | 链路断裂 | SN 无法追溯到采购单 |
| BATCH_UNKNOWN | 批次无法识别 | SN 无法唯一判定所属批次 |
| MANUAL_REVIEW | 需人工复核 | 需人工介入处理 |

异常修复必须记录操作日志，包括：修复前标签、修复后标签、修复人、修复时间、修复原因。

---

## 九、文件上传 / 导入 / 导出安全设计

| 序号 | 安全项 | 要求 |
|------|--------|------|
| 1 | 文件大小限制 | ≤ 10MB |
| 2 | SN 单次导入限制 | ≤ 5000 条 |
| 3 | 文件扩展名白名单 | .xlsx / .xls / .csv / .txt / .png / .jpg / .jpeg |
| 4 | MIME 类型白名单 | application/vnd.openxmlformats-officedocument.spreadsheetml.sheet 等 |
| 5 | 真实文件头校验 | 校验文件头部 magic bytes |
| 6 | 随机存储文件名 | 禁止使用用户原始文件名作为存储名 |
| 7 | 路径穿越防护 | 文件名中不得包含 .. / \ 等路径字符 |
| 8 | 上传目录不可执行 | Nginx 配置 uploads/ 目录禁止执行 |
| 9 | 导入模板格式校验 | 校验表头、列数、必填列 |
| 10 | 导出公式注入防护 | Excel/CSV 导出时对 = + - @ 等前缀转义 |
| 11 | FCS 模板保护 | 第 1-6 行只读，代码层面不可修改 |
| 12 | 操作日志 | 上传、导入、导出全部记录操作日志 |

---

## 十、Docker Compose 部署安全设计

| 序号 | 安全项 | 要求 |
|------|--------|------|
| 1 | PostgreSQL 端口 | 默认不对公网暴露，仅容器内部网络可达 |
| 2 | 生产密钥 | 不得进入 Git，通过 .env 注入 |
| 3 | .env.example | 只放示例变量，不放真实密钥 |
| 4 | 容器用户 | 尽量非 root 运行 |
| 5 | PostgreSQL 数据卷 | 必须持久化 |
| 6 | 文件上传目录 | 必须持久化 |
| 7 | healthcheck | PostgreSQL / Node.js / Nginx 增加 healthcheck |
| 8 | 日志轮转 | 增加日志轮转策略 |
| 9 | HTTPS | Nginx 负责 HTTPS 终端 |
| 10 | 环境隔离 | dev / staging / prod 配置隔离 |
| 11 | 目录分离 | 备份目录和上传目录分离 |

---

## 十一、页面清单（15 个页面）

| 序号 | 页面 | 路由 | 访问角色 | 核心功能 |
|------|------|------|---------|---------|
| 1 | 登录页 | /login | 所有人 | 登录、JWT、首次改密 |
| 2 | 首页看板 | /dashboard | 全部 | 轻量：欢迎+快捷入口+统计卡片+待处理 |
| 3 | 商品管理 | /products | 全部（商务只读） | CRUD、编码校验 |
| 4 | 基础资料 | /basic-data | 管理员 | 品类/品牌/枚举/经销商 |
| 5 | 供应商管理 | /suppliers | 全部（商务只读） | CRUD、allow_sn_extract |
| 6 | 采购列表 | /purchase | 全部 | 列表、状态筛选 |
| 7 | 采购新建 | /purchase/new | 操作员/管理员 | 新建草稿 |
| 8 | 采购详情 | /purchase/:id | 全部 | 详情、确认入库（purchase:confirm） |
| 9 | SN 上传 | /sn/upload | 操作员/管理员 | 单个+批量导入 |
| 10 | 库存查询 | /inventory | 全部 | 三列分离+批次维度+导出+补录 |
| 11 | 权限管理 | /permissions | 管理员 | 用户CRUD、purchase:confirm 授权 |
| 12 | 角色管理 | /roles | 管理员 | 角色权限矩阵 |
| 13 | 操作日志 | /logs | 管理员 | 日志列表、详情查看 |
| 14 | 个人中心 | /profile | 全部 | 个人信息 |
| 15 | 修改密码 | /change-password | 全部 | 改密+重新登录 |

---

## 十二、接口清单（约 60 个接口）

### 认证（5）
POST /api/auth/login | POST /api/auth/refresh | POST /api/auth/logout | GET /api/auth/me | POST /api/auth/change-password

### 用户管理（7）
GET /api/users | GET /api/users/:id | POST /api/users | PUT /api/users/:id | POST /api/users/:id/reset-password | PATCH /api/users/:id/toggle-status | PUT /api/users/:id/profile

### 角色管理（3）
GET /api/roles | GET /api/roles/:id | PUT /api/roles/:id/permissions

### 权限管理（1）
GET /api/permissions

### 商品管理（7）
GET /api/products | GET /api/products/:id | POST /api/products | PUT /api/products/:id | PATCH /api/products/:id/toggle-status | PUT /api/products/:id/pinyin | POST /api/products/import

### 品类（4）
GET /api/categories | POST /api/categories | PUT /api/categories/:id | DELETE /api/categories/:id

### 品牌（4）
GET /api/brands | POST /api/brands | PUT /api/brands/:id | DELETE /api/brands/:id

### 供应商（6）
GET /api/suppliers | GET /api/suppliers/:id | POST /api/suppliers | PUT /api/suppliers/:id | PATCH /api/suppliers/:id/toggle-status | PUT /api/suppliers/:id/pinyin

### 经销商（3）
GET /api/dealers | POST /api/dealers | PUT /api/dealers/:id

### 采购入库（6）
GET /api/purchase-orders | GET /api/purchase-orders/:id | POST /api/purchase-orders | PUT /api/purchase-orders/:id | POST /api/purchase-orders/:id/confirm（需 purchase:confirm 权限） | DELETE /api/purchase-orders/:id

### SN 管理（5）
POST /api/sn/upload | POST /api/sn/batch-import | POST /api/sn/supplement | GET /api/sn | GET /api/sn/:id

### 库存（2）
GET /api/inventory | GET /api/inventory/:productId/logs

### 导出（1）
POST /api/export/inventory

### 操作日志（1）
GET /api/operation-logs

### 采购字段枚举 + 文件（5）
GET /api/purchase-field-enums | POST /api/purchase-field-enums | PUT /api/purchase-field-enums/:id | POST /api/files/upload | GET /api/files/templates/:type

---

## 十三、测试策略

### 后端必测

**单元测试**：SN 校验、采购单号生成、拼音码、密码校验、权限矩阵（含 purchase:confirm）、库存三列更新、待补录计算、文件导入校验。

**集成测试**：登录、改密、采购草稿创建、确认入库（权限校验+库存更新+快照+日志+事务）、SN 录入/批量导入/补录、库存查询（三列+批次维度）、导出。

### 前端验收
手工验收，对照 34 条 UI 验收标准逐条检查。

---

## 十四、验收标准

### 业务验收点

| 序号 | 验收项 | 通过标准 |
|------|--------|---------|
| 1 | 登录认证 | 登录成功；首次强制改密；Token 过期刷新 |
| 2 | 三层角色权限 | 商务只看/导出；操作员可录入；purchase:confirm 独立权限点；管理员全权限 |
| 3 | 商品编码 | 唯一；引用后不可改；pinyin_code 仅搜索辅助 |
| 4 | 采购单号 | CG+YYYYMMDD+NNNN；每日流水重置 |
| 5 | 草稿与确认 | 保存=草稿；确认入库需 purchase:confirm 权限；二次确认；日志记录 |
| 6 | 库存三列分离 | 库存数量/可出库数量/已录入SN 独立展示 |
| 7 | SN 全局唯一 | 录入校验；批量返回重复；归属可追溯 |
| 8 | 历史快照 | 确认入库时生成；主数据变更不影响历史 |
| 9 | allow_sn_extract | 独立于 is_active；影响可出库数量 |
| 10 | 经销商编码 | 手工录入；唯一；引用后不可改；is_referenced 控制 |
| 11 | 默认经销商 | 全局唯一；变更记录日志 |
| 12 | 文件安全 | 白名单、大小限制、随机文件名、路径穿越防护 |
| 13 | FCS 模板 | 第 1-6 行只读 |

---

## 十五、种子数据

| 数据项 | 内容 |
|--------|------|
| 角色 | ADMIN / OPERATOR / VIEWER |
| 权限 | 含 purchase:confirm 独立权限点 |
| 管理员 | admin / yytx0401@ / 首次改密 |
| 品类 | ≥3 条示例 |
| 品牌 | ≥3 条示例 |
| 采购价格类型 | 原价 / 特价 / 一口价（不含"不限"） |
| 自定义列枚举 | ≥3 条 |
| 默认经销商 | ≥1 条，is_default=true |

---

**以上为《SN ERP 启动方案 V1.2 实施合并版》。后续开发只引用本文件。**
