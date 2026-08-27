# 数据管道契约设计（序2 · 素材表现数据管道）

> 日期：2026-08-11 · 状态：草稿待审 · 关联：《项目整体规划》（M1 线②数据管道）
> 本文档是六个模块设计的第二个（序2），定义素材表现数据管道的源表、聚合口径与结果表契约。

## 1. 定位与范围

数据管道是**独立管道服务**：从 Hive 数仓拉取原始投放数据，清洗聚合后写入 **Hive 结果表**，同时服务两个消费方：

| 消费方 | 消费内容 | 目的 |
|---|---|---|
| smart_ad_put（决策端） | Campaign 维度分段指标 + Product 维度聚合指标 | 生命周期判定 → 策略匹配 → 决策输出 |
| short-drama-scoring（打分模型） | 素材日粒度指标序列 | 预测素材未来 ROI → 0-100 分 |

**架构位置**：
```
Hive 数仓（原始表：campaign 小时级 / product 日级 / material 日级）
        │  拉取
        ▼
数据管道服务（独立模块：拉取 → 清洗 → 聚合）
        │  写入
        ▼
Hive 结果表（campaign_metrics / product_metrics / material_daily）
        │  消费
        ├──→ smart_ad_put（决策端）
        └──→ short-drama-scoring（打分模型）
```

**本期做**：源表拉取、清洗聚合、结果表写入、字段口径定义、更新调度。
**本期不做**：打分模型训练/推理（打分消费方只约定数据输入）、账户池、决策输出。

## 2. 源表（Hive 数仓原始层）

### 2.1 campaign 小时级表（`campaign_hour_raw`）

来源：投放系统。每条广告在每个小时一条记录。定义源自 smart_ad_put PRD §6.4。

| 字段 | 类型 | 说明 |
|---|---|---|
| campaign_id | string | 广告系列 ID |
| hour | datetime | 小时级时间戳（分区键） |
| cost_h | float | 小时消耗（元） |
| show_cnt | float | 展示量 |
| click_cnt | float | 点击量 |
| belong_h_cnt | float | 归因用户数 |
| belong_pay_cnt | float | 付费用户数 |
| vip_pay_cnt | float | 订阅用户数 |
| d0_order_amt | float | 当日收入-订单（元） |
| d0_ad_amt | float | 当日收入-广告（元） |
| business_type | int | 业务类型（1=付费, 2=免费） |
| link_language | string | 广告语言 |

### 2.2 product 日级表（`product_daily_raw`）

来源：投放系统。每个商品每天一条。定义源自 smart_ad_put PRD §6.5。

| 字段 | 类型 | 说明 |
|---|---|---|
| product_id | string | 商品（短剧）ID |
| date | date | 日期（分区键） |
| cost | float | 当日总消耗（元） |
| order_amt | float | 当日订单收入（元） |
| ad_amt | float | 当日广告收入（元） |
| campaign_count | int | 当日关联 Campaign 数 |
| first_campaign_hour | datetime | 首个 Campaign 投放时间 |

### 2.3 material 日级表（`material_daily_raw`）

来源：Hive 数仓（素材日级原始表）。每个素材每天一条。列定义对齐打分模型现有 `material_daily_v2.csv`。上游装载链路见 §7 待对齐。

| 字段 | 类型 | 说明 |
|---|---|---|
| filename | string | 素材 ID（`{…}_{语言}_{剧}_{日期}_{…}` 命名） |
| date | date | 日期 |
| cost | float | 当日消耗 |
| show_cnt | float | 展示量 |
| click_cnt | float | 点击量 |
| d0_recharge_amt | float | 当日订单充值 |
| active_cnt | float | 活跃用户数 |
| new_user_recharge_user_cnt | float | 新用户付费数 |
| d0_iaa | float | 当日广告分成收入 |
| ad_nu_cost | float | 广告新用户成本 |
| video_id | string | 短剧 Library ID |
| video_name | string | 短剧名 |
| nickname | string | 投放者 |
| language_name | string | 语言 |
| ctr / cpm / cpi | float | 派生指标 |
| d0_roi | float | 当日 ROI（= 收入/cost） |
| d0_ad_roi | float | 当日广告 ROI |

## 3. 结果表契约（Hive 结果层）

三张结果表，字段口径对齐两个消费方的实际调用（smart_ad_put 的 `detector.py` 签名 + 打分模型特征）。

### 3.1 `campaign_metrics` — 供 smart_ad_put 生命周期判定

对齐 `CampaignLifecycleDetector.detect()` 参数。**必须支持分段（0-24h / 24-72h / 72h+）**。

| 字段 | 说明 |
|---|---|
| campaign_id | 广告系列 ID |
| duration_hours | 投放时长（`MAX(hour)-MIN(hour)`） |
| revenue | 总收入（`SUM(d0_order_amt)+SUM(d0_ad_amt)`） |
| cost | 总成本（`SUM(cost_h)`） |
| revenue_0_24h / cost_0_24h | 前24h 分段收入/成本 |
| revenue_24_72h / cost_24_72h | 24-72h 分段收入/成本 |
| revenue_72plus / cost_72plus | 72h+ 分段收入/成本 |
| order_amt / ad_amt | 收入构成（判定盈利质量） |
| data_date | 计算日期（分区键） |

### 3.2 `product_metrics` — 供 smart_ad_put 商品生命周期判定

对齐 `ProductLifecycleDetector.detect()` 参数。

| 字段 | 说明 |
|---|---|
| product_id | 短剧 ID |
| total_revenue | 总收入 |
| total_cost | 总成本 |
| campaign_count | 关联 Campaign 数 |
| duration_hours | 最大投放时长 |
| order_amt / ad_amt | 收入构成 |
| recent_roi_history | 近 N 天每日 ROI 序列（oldest-first，≥14 天） |
| data_date | 计算日期（分区键） |

### 3.3 `material_daily` — 供打分模型

对齐打分模型特征工程输入。**必须保留 cost=0 行**（打分模型 `retrain_models.py` 明确不做 cost>0 过滤）。

| 字段 | 说明 |
|---|---|
| filename | 素材 ID |
| date | 日期 |
| cost / show_cnt / click_cnt / d0_recharge_amt / active_cnt / new_user_recharge_user_cnt / d0_iaa / ad_nu_cost | 原始指标（2.3 同口径） |
| video_id / video_name / nickname / language_name | 素材归属 |
| ctr / cpm / cpi / d0_roi / d0_ad_roi | 派生指标 |
| data_date | 计算日期（分区键） |

## 4. 聚合与清洗规则

- **分段聚合**（campaign_metrics）：按 `campaign_id` 分组，`hour` 相对首次投放时间偏移，落入 0-24h / 24-72h / 72h+ 三段累加。
- **日 ROI 序列**（product_metrics）：按 `product_id` + `date` 逐日计算 `ROI_d = (order_amt+ad_amt)/cost`，组成 oldest-first 数组，回填 ≥14 天。
- **ROI 定义**：`ROI = 收入 / 成本`，收入 = 订单收入 + 广告分成。此口径 smart_ad_put 与打分模型必须**共用同一定义**（两系统当前 ROI 口径需核对，见 §7 待对齐）。
- **零值保留**：`material_daily` 不按 cost>0 过滤；`campaign_metrics` / `product_metrics` 中 cost=0 的行保留（ROI 记为 0 或 NULL，由消费方处理）。

## 5. 更新频率与 SLA

| 项 | 目标 | 说明 |
|---|---|---|
| 调度频率 | 每 30min 增量拉取 | 对齐机器投放「决策→执行 ≤30min」链路 |
| 数据延迟 | 原始数据就绪延迟 < 1h | Hive 上游表就绪时间 |
| 历史回填 | ≥14 天 | 支撑 product_metrics 日 ROI 序列 + 打分 3d/5d 模型 |
| 结果表更新 | 增量更新，按 data_date / hour 分区 | 上游分区表按 `hour`/`date` 分区 |

**⚠️ 张力点**：机器投放验收目标「决策产生→执行 ≤30min」，但 Hive 数仓通常小时级就绪（PRD §6.4 数据延迟 <1h）。这意味着**决策端看到的指标天然滞后约 1-2h**。需确认 30min SLA 是「从决策产生到执行」而非「从真实发生到执行」，后者需要更实时的源。

## 6. 异常与兜底

| 场景 | 行为 |
|---|---|
| 源表某分区数据未就绪（缺小时） | 跳过该分区，记录告警，不阻断整批 |
| 数据全空 / 全零 | 整批跳过，记录原因（对齐机器投放告警规则） |
| 必填字段缺失 / 类型错 | 该行丢弃并计数，不静默 |
| 管道执行连续失败 N 次 | 日志 + 告警（N 默认 3） |
| cost 为负 / ROI 异常大 | 标记可疑行，写入异常清单，消费方可选忽略 |

## 7. 待对齐与依赖

- **依赖**：Hive 数仓可用性（三张源表就绪，含素材日级数据入 Hive 的装载链路）
- **待对齐**：
  - **素材级数据入 Hive 装载链路**：`material_daily_raw` 由谁 ETL 进 Hive、何时就绪（需确认 Hive 数仓当前是否已有素材日级表）
  - **ROI 口径统一**：smart_ad_put（`d0_order_amt + d0_ad_amt`）vs 打分模型（`d0_roi` 定义）是否一致，需拉一份数据核对
  - **30min SLA 边界**：确认是「决策产生→执行」还是「数据发生→执行」
  - 源表命名/分区约定（2.x 表名与现有 Hive 实际表名的映射）
  - 结果表由谁建（管道服务建表 vs 数仓侧建表）
- **假设**：数据管道服务独立部署（与 smart_ad_put、机器投放分离）；两个消费方通过 Hive 结果表只读消费，不直接依赖管道内部实现。
