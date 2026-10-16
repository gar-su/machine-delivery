# smart_ad_put × machine-delivery 对齐需求
> 状态：2026-10-16 复核 · 上游信号已产出并落盘 · 权威口径见 `docs/PRD-机器投放系统-支持跟投信号接入和绑定任务模板多选.md`（§2 信号契约、§8 开投前置）。本文档保留双方对接的事实、待办与开放项。
## 上游现状（2026-10-16 实测）
| 项 | 事实 |
|---|---|
| 信号模型 | `src/core/signal/models.py::BuildSignal`，注释标明「与 machine-delivery 对齐的信号字段」|
| 信号生成 | `BuildSignalGenerator`：阶段判定 → 信号类型映射 → 冷却控制（默认 24h，可按阶段覆写）|
| 阶段 → 信号映射 | 已定并生效：`config/signal_rules.json` → `campaign_growth` / `campaign_sustained` / `product_entry` / `product_sustained` / `product_growth` |
| 落盘 | `logs/decisions/<UTC日>/decisions.jsonl`（08-31：42 条 / 09-01：126 条 / 09-09：42 条）|
| 读取出口 | REST `GET /signals`（读当日文件）、`GET /signals/stats`（按类型统计）|
| 前端 | 已有「决策日志（建造信号）」页，列信号本体 |
| 素材维度 | 挂起。`StageSignalMapper` 注释：`material_*` 维度挂起；`MaterialLifecycleDetector` 基于行业经验设计、无真实素材数据 |
## 契约（已对齐，无需上游改动）
machine-delivery 直接用上游字段，不要求其增删。
| 字段 | 类型 | 必填 | machine-delivery 用途 |
|---|---|---|---|
| `signal_id` | string | ✅ | 幂等键，格式 `YYYYMMDDHHMMSS_目标ID` |
| `signal_type` | enum | ✅ | 路由到跟投路径（现仅 `FOLLOW_UP` 在用）|
| `target_dimension` | enum | ✅ | 参与路由键 `目标维度 × 语种` |
| `target_id` | string | ✅ | 绑定进任务的目标标识 |
| `language_code` | string | ✅ | 经对照表转本系统语种，参与路由键 |
| `script_no` | string | 否 | 命中本系统短剧库，补中文短剧名与剧本编号 |
| `shortplay_name` | string | 否 | 本系统短剧库未命中时的展示名（英文）|
| `reason` | string | 否 | 信号列表与执行记录展示 |
| `confidence` | number | ✅ | 信号列表展示（阶段判定可信度，非盈利概率）|
| `timestamp` | datetime | ✅ | 信号产出时间 |
## 唯一待拉齐项：语种取值
上游 `language_code` 用中文名，且取值集合与 machine-delivery 的语种表不重合。
| 上游取值 | machine-delivery 语种 | 处理 |
|---|---|---|
| 中文 / 英语 / 韩语 / 泰语 | 同名 | 直接使用 |
| 马来西亚语 | 马来语 | 对照表转换 |
| 印尼语 / 越南语 / 西班牙语 等 | — | 本系统不支持，信号跳过并留痕 |
待定：本系统是否要扩语种表以承接印尼语等（见「待决策开放项」）。
## 开投前置
| # | 前置 | 现状 | 归属 |
|---|---|---|---|
| 1 | 素材级决策能力 | 未闭环。上游素材维度挂起、无真实数据与判定能力 | 上游接入素材级数据源（打分系统素材分数 / 素材日级指标）|
| 2 | 语种取值对齐 | 未闭环。见上表 | 双方 |
| 3 | 信号通道（本系统接收端）| 未定。上游侧形态已有（落盘 JSONL + `GET /signals`）；本系统采用文件 / API / MQ 待定 | 双方 |
首项未闭环则素材级跟投不开投；后两项未闭环则连短剧级、单语种的试跑都起不来。
## 待决策开放项
1. **素材级跟投的来源**：上游接入打分系统（short-drama-scoring）的分数作为素材级判定依据，还是素材级跟投直接由打分系统出、不经上游？
2. **信号通道形态**：本系统接收端用文件（约定路径轮询）、REST（上游主动推送或本系统拉 `GET /signals`）、还是 MQ？
3. **名单外语种**：上游已产出印尼语 / 越南语 / 西班牙语信号，本系统当前只有中 / 英 / 韩 / 泰 / 马来五种。是扩语种表承接，还是这类信号一律跳过？
4. **冷却口径**：上游按「目标 + 信号类型」24h 冷却（可按阶段覆写）。本系统幂等只认 `signal_id`——同一目标冷却期后再次达标会产出新 `signal_id`，本系统会再绑一次。这是否是期望行为？
