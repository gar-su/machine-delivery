# 系统对接设计（序4 · 外部系统与广告平台 API）

> 日期：2026-08-11 · 状态：草稿待审 · 关联：《项目整体规划》（§5 系统对接范围与复杂度）
> 本文档是六个模块设计的第四个（序4），定义机器投放链路执行侧与广告平台的对接需求。

## 1. 定位与范围

执行侧只对接一个外部系统：广告平台（Meta/TikTok，经 netshort `autoTask` 后台接口，非直连）。素材-短剧绑定已由上游全量完成（决策 payload 直接携带 `shortplay_id` + `material_ids`），执行端不调 binding 接口；投放数据经 Hive 数仓（序2），不经报表系统 API。大模型/TTS（素材生产）序外仅确认接口形态。

**对接面总览**：

| 对接面 | 接口形式 | 用途 |
|---|---|---|
| 广告平台（Meta/TikTok） | **经 netshort `autoTask` 后台接口**，非直连 | 创建投放任务 |
| 大模型/TTS（LLM/豆包） | — | 素材生产（序外） |

**本期做**：执行侧对接需求——任务创建（幂等）、跨账户复制、渠道包、账户选取（序3）。
**本期不做**：直连 Meta/TikTok Graph API（经后台接口，不另建直连）；素材绑定（上游已完成）；数据获取（经 Hive 数仓，序2）；接口内部实现（鉴权/重试/限流等属既有接口实现，不在需求设计范围）。

## 2. 对接需求

### 2.1 幂等

- **任务创建**：`taskName` 含决策 `decision_id`（或执行 `execution_id`）标识；重复执行前按 `decision_id` 幂等（唯一索引），平台侧查重避免重复任务。
- **跨账户复制**：每账户每次复制分配唯一 `execution_id`，失败重试不产生重复任务（平台侧 task 幂等键或业务查重）。

### 2.2 账户与凭证

- 跨账户复制按序3 账户池选取 `copy_count` 个 `active` 账户；每账户独立凭证引用（序3 `credentials_ref`），互不串扰。

## 3. 各对接面明细

### 3.1 广告平台（Meta/TikTok，经 autoTask 后台）

| 项 | 内容 |
|---|---|
| 接口 | 确认 `/batchput/meta/autoTask/selectVideoCount` → 创建 `/batchput/meta/autoTask/insertOrUpdate` |
| 输入 | 决策 `target`（`shortplay_id` / `material_ids` / `language_code`）+ 渠道包 + 执行账户（序3 选取）；素材-短剧绑定上游已保证 |
| 风险感知 | 封禁/限额错误（`daily_limit`/账户级错误）→ 反馈账户池（序3 状态维护） |
| 消费方 | 执行引擎（序1） |

### 3.2 大模型/TTS（素材生产，序外）

- 已具备（narration_pipeline），本设计仅确认接口形态：LLM 旁白 / 豆包 TTS / BGM 库，均可替换，不进入本期对接范围。

## 4. 验收口径

- 任务创建幂等：同 `decision_id`/`execution_id` 重复执行不产生重复任务
- 执行端不调 binding 接口：建任务直接使用决策 `shortplay_id` + `material_ids`，素材-短剧关联由上游保证
- 跨账户复制：账户池（序3）选取与状态反馈正确，每账户独立凭证

## 5. 依赖与待对齐

- **依赖**：序1 决策契约（`shortplay_id` + `material_ids` + `language_code`）、序3 账户池（凭证引用 + 状态反馈）
- **待对齐**：
  - FB OAuth 授权修复（M0-M1 前置，业务侧批量修复）
  - 渠道优先级（Meta/TikTok 并行 vs 先后）
- **假设**：广告平台任务继续经 netshort `autoTask` 后台接口创建，不直连平台 Graph API；素材-短剧绑定已由上游全量完成，执行端不绑定；OAuth 修复由业务侧完成，本系统只做感知与告警
