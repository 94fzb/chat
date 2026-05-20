# AIOS 应用观测后端技术概要设计（V0.3，Coze Loop 对齐版）

## 1. 文档目标与范围

本文在既有 `docs/observability.md` 的基础上，补充 **面向 Coze Loop Observation 能力对齐** 的 AIOS 后端落地方案。

目标：
- 对齐 Coze Loop 公开能力中的 Observation 主线：`Trace reporting` + `Trace data observation`。
- 建立 AIOS 从 SDK/API 上报、链路处理、明细检索、指标分析到告警处置的工程闭环。
- 明确“**Coze Loop 对齐项**”与“**AIOS 扩展项**”，避免概念混淆。

> 对齐依据：Coze Loop 公开文档强调从用户输入到 AI 输出的全链路可视化，核心阶段包含 prompt parsing、model invocation、tool execution，并通过 SDK 进行 Trace 上报与查询。

---

## 2. Coze Loop 对齐矩阵（关键）

| Coze Loop 公共能力 | AIOS 对齐实现 | 状态 |
|---|---|---|
| Trace reporting（SDK/API） | `POST /api/v1/observe/traces`、`/spans/batch`、`/events/batch` | 已定义 |
| Trace data observation（查询） | Trace 列表、Trace 详情、调用树下钻 | 已定义 |
| Prompt Parsing 可观测 | `span_type=prompt.parse`，记录模板版本/变量渲染耗时 | 本版新增 |
| Model Invocation 可观测 | `span_type=model.call`，记录 model/token/cost/latency | 已定义 |
| Tool Execution 可观测 | `span_type=tool.call`，记录工具参数摘要/失败码/重试 | 已定义 |
| 中间结果与异常捕获 | Event/Log + `exception` 事件模型 | 已定义 |
| 多环境与多工作区隔离 | `workspace_id/app_id/env` 租户与环境维度 | 已定义 |

---


## 2.1 与现有 Coze 上报 SDK 的对接约束（补充）

> 说明：当前仓库运行环境无法直接访问你提到的本地路径 `/Users/xiaochun-work/work/app`，因此这里先给出可执行的对接契约；拿到 SDK 代码后可按本节做逐字段映射核对。

建议将 AIOS 接入层与现有 Coze SDK 做“无损映射”：

- **身份上下文**：`workspace_id`、`app_id`、`env` 必填并作为查询主过滤维度。
- **链路主键**：保留 SDK 侧 `trace_id`/`span_id`/`parent_span_id`，避免网关重写。
- **阶段语义**：SDK 中 prompt、model、tool 三类阶段统一映射到 `prompt.parse`、`model.call`、`tool.call`。
- **成本口径**：`token_in/token_out/cost` 保持 SDK 原始单位，Processor 只做汇总不做口径变换。
- **异常口径**：SDK 错误统一落到 `error_code` + `error_message`，并补充 `event_type=exception`。

落地顺序建议：
1. 先打通 SDK -> `/api/v1/observe/spans/batch` 的最小链路（仅主字段）。
2. 再补齐事件与日志上报（`/events/batch`）。
3. 最后补齐成本、标签、重试等扩展字段并做回归校验。

---

## 3. 总体架构（对齐 Trace 主线）

```text
[AIOS App / Agent Runtime]
        |
   (CozeLoop-like SDK / OTel / HTTP)
        v
[Ingestion Gateway]
  - 鉴权 / 限流 / 协议转换 / 幂等
        v
[Message Queue]
  - Kafka/Pulsar
        v
[Trace Processor]
  - 语义标准化 / 脱敏 / 采样 / 纠偏 / 聚合
        |
   +----+------------------------------+
   |                                   |
   v                                   v
[OLTP: PostgreSQL/MySQL]         [OLAP: ClickHouse]
(元数据/规则/索引摘要)            (Trace/Span/Event 明细与分析)
   |
   v
[Query & Metrics API]
   |
   v
[AIOS Console / Alerting / Evaluation]
```

说明：
- 对齐 Coze Loop 的“Trace 上报 + Trace 查询”主路径，采用接入层与处理层解耦。
- 查询优先聚合、明细下钻，确保控制台可用性与成本平衡。

---

## 4. 统一语义规范（与 Coze Loop 观测阶段一致）

### 4.1 主实体

- `trace_id`：一次完整请求链路。
- `span_id`：链路阶段节点。
- `event_id`：阶段内关键事件（包括异常与重试）。

### 4.2 Span 类型（V0.3）

- `prompt.parse`（新增，强对齐 Coze Loop）
- `orchestrator.plan`
- `agent.step`
- `model.call`
- `tool.call`
- `memory.retrieve`
- `memory.write`
- `policy.evaluate`
- `custom`

### 4.3 状态与错误码

- `status`：`ok/error/timeout/cancelled`
- 错误码前缀：
  - `PLAT_XXXX`（平台侧）
  - `LLM_XXXX`（模型侧）
  - `TOOL_XXXX`（工具侧）
  - `BIZ_XXXX`（业务侧）

---

## 5. 数据模型（MVP）

### 5.1 Trace
- `trace_id`
- `workspace_id/app_id/env`
- `start_time/end_time/duration_ms`
- `status`
- `root_input_digest/root_output_digest`
- `token_total/cost_total`
- `tags`

### 5.2 Span
- `trace_id/span_id/parent_span_id`
- `span_type/name/status`
- `start_time/end_time/duration_ms`
- `error_code/error_message`
- `model_name/tool_name/token_in/token_out`
- `attributes`

### 5.3 Event / Log
- `trace_id/span_id/event_type`
- `payload`
- `occurred_at`

---

## 6. 关键流程

### 6.1 上报流程（Reporting）
1. SDK 生成 Trace/Span/Event（含 `workspace_id/app_id/env`）。
2. Gateway 鉴权与 schema 校验。
3. 数据写入 MQ，Processor 异步消费处理。
4. 明细写入 ClickHouse，摘要与规则元数据入 OLTP。
5. 控制台在分钟级窗口内可见聚合，秒级可查明细。

### 6.2 查询流程（Observation）
- 列表查询：按时间、状态、应用、错误码、耗时过滤。
- 详情查询：展示调用树，重点查看 `prompt.parse -> model.call -> tool.call`。
- 聚合查询：QPS、成功率、P95/P99、错误码 TopN、Token/成本趋势。

---

## 7. 存储与索引（ClickHouse）

### 7.1 表划分
- `observe_trace`
- `observe_span`
- `observe_event`
- `observe_metric_1m`

### 7.2 分区与索引
- 分区：`toDate(start_time)`
- 排序键：`(workspace_id, app_id, start_time, trace_id)`
- 高频筛选列（`status/span_type/tool_name/error_code`）建立 skipping index

### 7.3 生命周期
- 明细 30 天
- 聚合 180 天
- 冷数据归档至对象存储（Parquet）

---

## 8. API 概要

### 8.1 写入
- `POST /api/v1/observe/traces`
- `POST /api/v1/observe/spans/batch`
- `POST /api/v1/observe/events/batch`

### 8.2 查询
- `GET /api/v1/observe/traces`
- `GET /api/v1/observe/traces/{trace_id}`
- `GET /api/v1/observe/metrics/overview`
- `GET /api/v1/observe/metrics/trend`

### 8.3 鉴权
- `Authorization: Bearer <token>`
- 按应用/工作区配额限流（超限 429）

---

## 9. SLO 与告警闭环

### 9.1 SLO
- `trace_write_success_rate >= 99.9%`（5m）
- `trace_query_p95 <= 1500ms`（15m）
- `ingest_to_query_delay_p95 <= 120s`

### 9.2 告警分级
- P1：写入成功率 < 99%（5m）
- P2：查询 P95 > 2s（10m）
- P3：日成本较 7 日均值偏离 > 30%

### 9.3 处置
- 自动附带受影响应用、错误码 TopN、最近回归版本。
- 30 分钟内初判，24 小时内复盘。

---

## 10. 里程碑（8 周）

- W1-W2：语义与协议冻结，SDK 最小上报。
- W3-W4：Gateway/Processor/ClickHouse 打通；Trace 列表与详情。
- W5-W6：聚合指标、查询优化、权限脱敏、告警联动。
- W7：压测、容灾、灰度接入。
- W8：上线复盘与优化清单。

---

## 11. 验收标准（MVP）

- 至少 2 类应用成功接入并稳定上报。
- 10 分钟内可检索到新增 Trace。
- 控制台可下钻 `prompt.parse/model.call/tool.call` 三阶段。
- 可输出日级成功率、P95、成本趋势。
- P1/P2 告警演练各通过至少 1 次。

---

## 12. 与现有文档关系

- `docs/observability.md`：保留治理与观测框架。
- 本文：后端工程化与 Coze Loop 对齐实现细化。
