# Seed 基线记录

> 基于：V1.2 实施合并版 Q-14 种子数据要求
> 创建日期：2026-04-28
> 版本：4.0（第 2-B 基础资料 Seed）

---

## 一、第 2-B 冻结基线

### 权限基线（第 2 步已冻结，保持不变）

| 项目 | 内容 | 状态 |
|------|------|------|
| 基础角色 | admin / operator / business（V1.2/V6 三层角色） | ✅ 已冻结 |
| 基础权限 | 21 项模块级权限 | ✅ 已冻结 |
| 角色-权限绑定 | admin(21) / operator(10) / business(8)，受管角色先清后建 | ✅ 已冻结 |
| 管理员账号 | admin 用户绑定 admin 角色，bcrypt 密码 hash，普通 seed 不重置密码 | ✅ 已冻结 |

### 基础资料 seed（第 2-B 完成）

| 编号 | 项目 | 表 | 数量 | 状态 |
|------|------|------|------|------|
| Q-14-1 | 品类 | `biz_category` | 3 条（手机 / 平板 / 配件） | ✅ 已完成 |
| Q-14-2 | 品牌 | `biz_brand` | 3 条（Apple / Samsung / Xiaomi） | ✅ 已完成 |
| Q-14-3 | 价格类型 | `biz_purchase_field_enum` (field_key=price_type) | 3 条（原价 / 特价 / 一口价） | ✅ 已完成 |
| Q-14-4 | 自定义列枚举 | `biz_purchase_field_enum` (field_key=custom_col_1/2/3) | 6 条（颜色/容量/成色/版本/运营商/网络制式） | ✅ 已完成 |
| Q-14-5 | 默认经销商 | `biz_dealer` | 1 条（DEFAULT / is_default=true） | ✅ 已完成 |

### 默认经销商处理规则

- upsert by `code = 'DEFAULT'`，幂等
- 如果存在其他 `is_default=true` 的经销商且 code 不是 'DEFAULT'，先取消旧默认再设置新默认
- 受 `uk_dealer_default` 部分唯一索引约束，全局仅允许一个默认经销商

### Q-14 全部完成状态

| 编号 | 项目 | 状态 |
|------|------|------|
| Q-14-1 | 品类 ≥3 条 | ✅ |
| Q-14-2 | 品牌 ≥3 条 | ✅ |
| Q-14-3 | 价格类型 | ✅ |
| Q-14-4 | 自定义列枚举 ≥3 条 | ✅ |
| Q-14-5 | 默认经销商 ≥1 条 | ✅ |
| — | 三类角色 + 权限组 | ✅（第 2 步） |
| — | 管理员账号 | ✅（第 2 步） |

> Q-14 种子数据全部完成，不再有待办项。

---

## 二、旧版角色清理说明

### 开发环境

如果曾执行过旧 seed（含 super_admin / viewer 角色），必须先执行：

```bash
prisma migrate reset --force
prisma db seed
```

确保只保留 admin / operator / business 三层角色。

### 生产环境

- 生产环境**不得直接删除**已有角色。
- 如需清理 super_admin / viewer，需要单独迁移方案和数据确认。
- 当前第 2 步冻结基线以 **reset 后 seed 结果** 为准。

---

## 三、价格类型和自定义列枚举落库说明

| 项目 | 落库方式 | 说明 |
|------|---------|------|
| 价格类型 | `biz_purchase_field_enum`，field_key = `price_type` | schema 已有此表，直接写入 seed |
| 自定义列枚举 | `biz_purchase_field_enum`，field_key = `custom_col_1` / `custom_col_2` / `custom_col_3` | schema 已有此表，直接写入 seed |

---

## 变更记录

| 版本 | 日期 | 变更内容 | 负责人 |
|------|------|---------|--------|
| 1.0 | 2026-04-28 | 初始创建：记录第 2 步范围及 Q-14 剩余项 | backend |
| 2.0 | 2026-04-28 | 第 2-R-1：权限矩阵收口，移除 operator purchase:confirm + business manage 权限，新增 4 项查询权限，旧角色清理说明 | backend |
| 3.0 | 2026-04-28 | 第 2-R-2：operator 移除 inventory:manage，新增 sn:supplement，对齐 V1.2 Q-13 操作员库存权限边界 | backend |
| 4.0 | 2026-04-28 | 第 2-B：基础资料 seed 完成（品类/品牌/价格类型/自定义列枚举/默认经销商），Q-14 全部完成 | backend |
