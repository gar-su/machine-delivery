# 账户池设计（序3 · 跨账户复制执行底座）

> 日期：2026-08-11 · 状态：草稿待审 · 关联：《项目整体规划》（M1 线②账户池）
> 本文档是六个模块设计的第三个（序3），定义账户池的模型、存储、状态维护与选取策略。

## 1. 定位与范围

账户池是机器投放**执行端**的投放账户底座。跨账户复制（决策契约 `strategy.copy_count` + `strategy.account_pool`）从账户池选取执行账户。

**架构位置**：
```
决策契约（序1）strategy.account_pool / copy_count
        │  引用
        ▼
账户池（本模块，独立存储）
  accounts（投放账户 + 状态 + 标签）
  pools（账户池定义）
        │  选取
        ▼
执行引擎（序1）：跨账户复制 → 每个副本分配一个账户
```

**本期做**：账户模型、账户池定义、状态维护流程、随机选取策略、执行端接入接口。
**本期不做**：标签过滤动态选取（决策定为纯随机）、自动检测账户状态（决策定为人工维护）、平台侧账户创建/授权（业务侧 OAuth 修复）。

## 2. 决策记录（业务拍板）

| 决策项 | 结论 | 说明 |
|---|---|---|
| 存储位置 | **独立存储** | 独立轻量库/服务，与执行记录、数据管道解耦 |
| 状态维护 | **人工维护** | 账户封禁/限额由开发人员手动改配置/库 |
| 选取策略 | **纯随机** | 从池子可用账户随机选 N 个，不考虑标签 |

> 注：后续如需标签过滤，扩展池子定义为「过滤型池子」（见 §6 预留）。

## 3. 账户模型

### 3.1 `accounts` 表

| 字段 | 类型 | 说明 |
|---|---|---|
| account_id | string | 投放账户 ID（对应平台广告账户 / `created_by` 投放者 ID） |
| platform | enum | facebook / tiktok / 其他（预留） |
| status | enum | active / paused / banned / daily_limit |
| labels | json | 标签（country / language / tier），**本期仅展示用，不参与选取** |
| credentials_ref | string | 凭证引用（序4 对接面） |
| pool_id | string | 所属账户池 |
| remark | string | 备注 |
| updated_at | datetime | 最后更新时间 |

### 3.2 账户状态

| status | 含义 | 可否选取 |
|---|---|---|
| active | 可用 | ✅ 可选取 |
| paused | 暂停（人工停用） | ❌ |
| banned | 封禁 | ❌ |
| daily_limit | 当日达限额 | ❌ 当日不可选，次日恢复候选 |

**人工维护流程**：开发人员发现封禁/限额（平台报错、投放失败告警）后，手动改 `status`。`daily_limit` 次日由调度重置回 `active`（或人工确认后改）。

## 4. 账户池定义

### 4.1 `pools` 表

| 字段 | 类型 | 说明 |
|---|---|---|
| pool_id | string | 池子 ID（决策契约 `account_pool` 引用此值） |
| name | string | 池子名（如 `default`、`kr_high_tier`） |
| description | string | 说明 |
| created_at | datetime | 创建时间 |

### 4.2 池子与账户关系

- 一个账户属于一个池子（`accounts.pool_id`）。
- 池子 = 账户的显式列表（**无标签过滤**，纯随机从池内 `active` 账户选）。
- 池子须满足可选出数量：`copy_count ≤ 池内 active 账户数`，否则记录 skipped（对齐序1 §6 告警）。

## 5. 选取逻辑（纯随机）

```
select_accounts(pool_id, copy_count, exclude_account_ids):
  候选 = accounts WHERE pool_id=? AND status='active' AND account_id NOT IN exclude
  if len(候选) < copy_count:
      return skipped(原因=可用账户不足，需要N个，池内仅剩M个)
  随机选 copy_count 个 → 返回

exclude_account_ids = 已在跑该素材的账户（序1 执行引擎传入，避免重复复制到同账户）
```

- **排除重复**：同素材不复制到已在跑它的账户上（execution 记录查 `copied_accounts`）。
- 选取结果为纯随机；标签不参与过滤，仅展示。

## 6. 预留扩展（本期不做）

- **标签过滤型池子**：`pools.filter` 定义标签条件（country/language/tier），动态计算候选账户。本期决策为纯随机，此能力预留，扩展时不改选取接口签名，仅改池子定义类型。
- **自动状态检测**：序4 对接后，平台返回封禁/限额错误时可选自动更新状态，本期人工维护。

## 7. 独立存储方案

- 账户池**独立存储**，与执行记录、数据管道解耦，职责单一、互不耦合。
- 账户/池子的增删改走配置或管理脚本（人工维护），无管理界面（看板只读展示在序5）。

## 8. 接口（供执行引擎）

| 接口 | 说明 |
|---|---|
| `select_accounts(pool_id, copy_count, exclude) -> (accounts | skipped_reason)` | 纯随机选取（§5） |
| `list_pools() -> list[pool]` | 列出池子 |
| `list_accounts(pool_id, status?) -> list[account]` | 列账户（看板用） |
| `update_status(account_id, status)` | 人工改状态 |

## 9. 验收口径

- 选取满足 `copy_count` 且账户均为 `active`，不选入 `paused/banned/daily_limit`
- 同素材不重复复制到已跑账户（排除重复生效）
- 池内 active 账户不足时返回明确 `skipped` 原因，不静默降级
- 状态变更可追溯（`updated_at` + 变更来源）

## 10. 依赖与待对齐

- **依赖**：序1 决策契约 `account_pool` / `copy_count` 字段、序4 系统对接的凭证引用（`credentials_ref`）、序5 看板只读展示
- **待对齐**：
  - 现有 `created_by` 与账户池 `account_id` 的映射关系（`2030896251477688321` 是否为池内首个账户）
  - 池子初始清单（哪些账户、哪些池、状态如何）
  - `daily_limit` 次日重置的归属（调度任务 vs 人工）
  - 独立库与执行记录库同机部署还是独立部署
- **假设**：账户池账户数与平台侧账户一一对应；凭证由序4 统一管理
