# smart_ad_put × machine-delivery 对齐需求
> 状态：跟投自 2026-10-14 起本期接入 · 权威口径见 `docs/机器投放系统-跟投需求.md`（§2.2 信号契约、§8 开投前置）。本文档保留两侧 gap、契约映射与待办，供对接使用。
## 背景（两侧定位）
| 系统 | 定位 | 决策产物 |
|---|---|---|
| smart_ad_put（上游） | 决策端：消费指标 → 判定 → 产出决策 | 现为生命周期管理动作（关停 / 加预算 / 渠道扩张 / 复制广告）|
| machine-delivery（下游）| 触发编排层：接收信号 → 绑定进自动化任务 | 需要「跟投」信号（某目标值得继续投 → 绑定进跟投任务）|
## 契约映射（原 Gap 1 / Gap 3）
machine-delivery 只认一个动作 `FOLLOW_UP`，不需要上游的细粒度动作枚举。
| machine-delivery 字段 | 类型 | 必填 | smart_ad_put 现状 | 要做 |
|---|---|---|---|---|
| `signal_id` | string | ✅ | 有 `decision_id`（语义同幂等键）| 改名或映射 |
| `signal_type` | enum | ✅ | 无，用 `Decision.type`（动作枚举）| 新增 `FOLLOW_UP` 取值 |
| `target_dimension` | enum | ✅ | `Decision.dimension`(product/campaign) | 补 `material`，对齐取值 |
| `target_id` | string | ✅ | `Decision.target_id` | 已对齐 |
| `shortplay_id` | string | ✅ | 无 | 新增：目标所属短剧，`product` 级与 `target_id` 同值 |
| `reason` | string | 否 | `Decision.reason` | 已对齐 |
原 `script_no` 取消：剧本编号由 machine-delivery 从短剧库反查取得，上游不再补此字段。
语义补充：上游判定目标进入增长 / 稳定等值得放量阶段时产出 `FOLLOW_UP`，而非 `CLONE_AD` 等生命周期动作。生命周期动作（关停 / 加预算 / 渠道扩张）仍走上游自有链路，不进 machine-delivery。
## 开投前置（未闭环则不开投）
| # | 前置 | 现状 | 归属 |
|---|---|---|---|
| 1 | 信号通道 | 未定（文件 / API / MQ）；上游现输出本地 JSON 文件（`DecisionCommander.output`）| 上游提供稳定出口，双方定格式 |
| 2 | 素材级决策能力 | `MaterialLifecycleDetector` 注释明写「当前数据中没有素材 ID，此检测器基于行业经验设计」；`StrategyEngine.get_dimension` 把 `material_*` 阶段并入 `campaign` | 上游接入素材级数据源（打分系统素材分数 / 素材日级指标）|
| 3 | 阶段 → 跟投映射表 | 未定 | 上游定表：哪些 lifecycle 阶段映射为「跟投」|
为什么是前置：跟投全自动、无人工确认，信号到达即绑定即产生消耗。这三项不闭环，机器投放接到的信号要么送不到，要么是错的目标。
## 待决策开放项
1. **素材级跟投的来源**：smart_ad_put 接入打分系统（short-drama-scoring）的分数作为素材级判定依据，还是素材级跟投直接由打分系统出、不经 smart_ad_put（推翻此前「决策统一由 smart_ad_put，素材级由其补齐」的结论）？
2. **阶段映射表的粒度**：`growth` / `sustained` / `entry` 等阶段分别是否映射为跟投，还是只取其一？需定一张「阶段 → 跟投」映射表。
3. **信号通道是否现在定**：可否先约定一个文件路径跑通、后续再换 API / MQ？（此前定过「不锁实现」）
## 已闭环
- 信号契约字段与取值——见本文档「契约映射」
- 语种来源——machine-delivery 侧按 `shortplay_id` 反查短剧库，上游不传语种
- 跟投配置与权限、处理流程、异常与验收——见 `docs/机器投放系统-跟投需求.md` §3 / §4 / §5 / §10
