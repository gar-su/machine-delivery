# smart_ad_put × machine-delivery 对齐需求

> 状态：暂存 · 待 smart_ad_put 侧落地
> 目的：让 smart_ad_put 能产出 machine-delivery 所需的「跟投决策」信号（`FOLLOW_UP`）。当前 smart_ad_put 的决策是「生命周期管理」语义，与 machine-delivery 的「跟投」语义错位，存在 4 项 gap。

## 背景（两侧定位）

| 系统 | 定位 | 决策产物 |
|---|---|---|
| smart_ad_put（上游） | 决策端：消费指标 → 判定 → 产出决策 | 现为生命周期管理动作（关停/加预算/渠道扩张/复制广告）|
| machine-delivery（下游）| 触发编排层：接收信号 → 新建自动化任务 | 需要「跟投」信号（某目标值得继续投 → 新建放量任务）|

## Gap 1：决策语义错位（最关键）

**现状**：smart_ad_put 的 `ActionType` = `GROWTH_BURST / CHANNEL_EXPAND / CLONE_AD / INCREASE_BUDGET / GRACEFUL_SHUTDOWN / REBUILD / BUDGET_SMOOTH …`，全是「调整已投广告」。

**machine-delivery 需要**：跟投 = 「判定某目标（剧/Campaign/素材）值得继续投放 → 新建一个自动化任务」。动作统一为 `FOLLOW_UP`，不需要这些细粒度动作枚举。

**要补**：smart_ad_put 新增一类决策输出「跟投」，即当判定目标进入增长/稳定等值得放量阶段时，产出 `FOLLOW_UP` 信号，而非 `CLONE_AD` 等生命周期动作。

## Gap 2：素材级决策缺失

**现状**：`MaterialLifecycleDetector` 注释明写「当前数据中没有素材ID，此检测器基于行业经验设计」，且 `StrategyEngine.get_dimension` 把 `material_*` 阶段归并入 `campaign`。**素材维度无真实数据、无真实判定能力。**

**machine-delivery 需要**：`target_dimension = material`（素材级跟投）。

**要补**：smart_ad_put 需接入素材级数据源（打分系统 short-drama-scoring 的素材分数 / 素材日级指标），并能产出素材级跟投决策。或明确「素材级跟投」的决策由哪一方出（此前定过"决策统一由 smart_ad_put，素材级由其补齐"）。

## Gap 3：信号字段契约不对齐

smart_ad_put 需产出符合 machine-delivery 定义的 `FOLLOW_UP` 信号：

| machine-delivery 字段 | 类型 | smart_ad_put 现状 | 要补 |
|---|---|---|---|
| `signal_id` | string | 有 `decision_id`（语义同幂等键）| 改名/映射即可 |
| `signal_type` | enum | 无，用 `Decision.type`（动作枚举）| 新增 `FOLLOW_UP` 信号类型 |
| `target_dimension` | enum | `Decision.dimension`(product/campaign) | 补 material；对齐取值 |
| `target_id` | string | `Decision.target_id` | ✅ 直接对齐 |
| `script_no` | string | 无 | 目标为短剧时补齐剧本编号 |
| `reason` | string | `Decision.reason` | ✅ 直接对齐 |

## Gap 4：信号通道未定

machine-delivery 需要一个信号入口接收 smart_ad_put 决策（文件/API/MQ，实现不锁）。smart_ad_put 现有输出是写本地 JSON 文件（`DecisionCommander.output`）。

**要定**：决策以什么方式、什么格式送达 machine-delivery。建议后续在 smart_ad_put 侧对齐一个稳定的信号出口（REST API 或消息队列或约定文件路径）。

## 待决策开放项

1. **素材级跟投的来源**：smart_ad_put 接入打分系统的分数作为素材级判定依据，还是素材级跟投直接由打分系统出、不经 smart_ad_put（推翻之前"统一到 smart_ad_put"的结论）？
2. **跟投决策的触发粒度**：smart_ad_put 现有的 lifecycle 判定（product/campaign）里，哪些阶段映射到"跟投"（如 `growth`/`sustained`/`entry` → 跟投）？需要定一张「阶段 → 跟投」的映射表。
3. **信号通道**：是否要现在定？（此前定过"不锁实现"）
