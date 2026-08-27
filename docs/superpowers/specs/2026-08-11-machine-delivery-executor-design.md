# 机器投放系统 — 执行端设计（序1 · 决策契约 + 执行端）

> 日期：2026-08-11 · 状态：草稿待审 · 关联：《项目整体规划》（M1 线③机器投放核心）
> 本文档是「短剧投放全自动化链路」六个模块设计中的第一个（序1），定义决策契约与执行端需求。

## 1. 定位与范围

机器投放系统是**执行端（executor）**：消费上游 `smart_ad_put`（决策端/指挥官）输出的决策，执行广告平台操作，记录执行结果，供看板查询。

**决策端与执行端的职责边界：**

| 角色 | 系统 | 职责 |
|---|---|---|
| 决策端 | smart_ad_put（已存在） | 消费指标数据 → 生命周期判定 → 策略匹配 → 输出决策 |
| 执行端 | 本仓库（待建） | 读决策 → 执行（建任务/跨账户复制）→ 记录执行 → 看板 |

**关键前提**：决策契约由执行端定义（从执行端能力反推），smart_ad_put 按此产出。「决策带全」——决策 payload 已含执行所需全部信息，执行端不回查任何数据源。

**本期做**：决策消费、建任务/跨账户复制执行、执行记录、执行相关看板接口。
**本期不做**：生命周期判定/策略匹配（决策端职责）、预算调整/关停/素材预热执行（§3 动作边界）、登录鉴权、界面改配置。

## 2. 决策契约（执行端反推）

执行端完成一次操作所需的全部信息即决策对象。**字段由执行端定义，smart_ad_put 按此产出。**

### 2.1 决策对象

```yaml
decision:
  decision_id: str          # 必填 · 幂等键（全局唯一，去重依据）
  created_at: datetime      # 必填 · 决策生成时间（UTC）
  action: CREATE_DELIVERY   # 必填 · 动作类型（§2.2 枚举）
  target:                   # 必填 · 投什么
    shortplay_id: str       #   短剧 ID
    shortplay_name: str     #   短剧名
    language_code: str      #   素材语言（如 en_US）
    material_ids: list[str] #   素材 ID 列表
  strategy:                 # 必填 · 怎么投
    copy_count: int         #   跨账户复制份数（≥1）
    account_pool: str       #   账户池名（依赖序3 账户池）
  rule_id: str              # 必填 · 触发规则 ID（看板规则列表命中统计）
  reason: str               # 必填 · 决策理由（可读）
  confidence: float         # 选填 · 默认 1.0
  source: str               # 必填 · "smart_ad_put"
```

### 2.2 动作类型（MVP）

| action | 含义 | 本期 |
|---|---|---|
| `CREATE_DELIVERY` | 建任务/跨账户复制 | ✅ 实现 |

**设计决策**：MVP 只定义 `CREATE_DELIVERY` 一种动作。`INCREASE_BUDGET` / `GRACEFUL_SHUTDOWN` / `MATERIAL_PREPARE` 不进入契约，后续按执行端能力扩展枚举，不预先定义空动作。

### 2.3 契约示例

```json
{
  "decision_id": "20260811103000_sp_10001",
  "created_at": "2026-08-11T10:30:00Z",
  "action": "CREATE_DELIVERY",
  "target": {
    "shortplay_id": "2044271853970653185",
    "shortplay_name": "The Lion's Captive",
    "language_code": "en_US",
    "material_ids": ["922860", "922861"]
  },
  "strategy": {
    "copy_count": 3,
    "account_pool": "default"
  },
  "rule_id": "growth_roi_gt_40",
  "reason": "成长期ROI>40%触发跨账户复制",
  "confidence": 0.85,
  "source": "smart_ad_put"
}
```

## 3. 执行需求

- **送达与幂等**：决策须幂等送达执行端——执行端按 `decision_id` 去重消费，重复投递不重复执行（消费通道/机制为实现细节）。
- **CREATE_DELIVERY 执行**：建任务 × `copy_count`，直接使用决策 `shortplay_id` + `material_ids`（素材-短剧绑定由上游保证，执行端不绑定）；每个副本从账户池（序3）选取执行账户。
- **动作边界**：预算调整 / 关停 / 素材预热本期不实现；收到未支持动作 → 记录 `skipped: unsupported_action` 并告警，不静默丢弃。

## 4. 执行记录

执行端须为每次执行留下可审计记录：

| 字段 | 类型 | 说明 |
|---|---|---|
| execution_id | str | 执行记录 ID |
| decision_id | str | 决策 ID（幂等键，唯一索引） |
| rule_id | str | 触发规则 ID（规则列表命中统计） |
| action | str | 动作类型 |
| target | json | 目标快照（shortplay/material/language） |
| metrics_snapshot | json | 触发时表现数据（roi/cost/conversions，若决策端提供） |
| status | enum | success / failed / skipped |
| error_reason | str | 失败/跳过原因 |
| platform_task_ids | json | 平台返回的 task_id 列表 |
| copied_accounts | json | 实际复制到的账户列表 |
| cooldown_until | datetime | 素材/短剧冷却到期时间 |
| created_at | datetime | 执行时间 |

**告警规则**：连续执行失败 N 次 → 日志告警（N 可配，默认 3）；账户池可用账户不足 → 记录 skipped 不强行执行；决策数据异常（必填字段缺失/类型错）→ 整条跳过并记录原因。

## 5. 验收口径

- 决策产生 → 执行完成 ≤ 30min
- 同 `decision_id` 重复投递不重复执行（幂等，以唯一索引为准）
- 成功率 = 执行成功数 / 应执行数（排除账户池不足等客观原因），可观测
- 规则命中 / 跟投任务数可查（看板）

## 6. 依赖与待对齐

- **依赖**：序3 账户池（账户选取）、序4 系统对接（广告平台接口）、序5 看板
- **待对齐**：smart_ad_put 决策输出适配（按 §2 契约产出）、决策消费通道约定（文件/API/MQ）、账户池命名规范（`account_pool` 与序3 对齐）
- **假设**：执行端复用现有渠道包/投放能力，经序4 对接面接入广告平台；广告平台任务经 netshort `autoTask` 后台接口创建（序4）
