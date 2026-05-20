# AIOS 应用观测后端技术概要设计（V0.1）

## 1. 文档目标与范围

本文面向 **AIOS 前期“应用观测”能力建设**，给出后端技术概要设计。设计目标：

- 建立从 SDK 上报到查询分析的完整 Trace 数据链路。
- 支撑应用运行过程的可观测（请求、模型调用、工具调用、异常、耗时、成本）。
- 为后续评测、告警、诊断与优化留出扩展接口。

> 说明：你提供的本地参考工程路径 `/Users/xiaochun-work/work/coze-loop` 在当前执行环境不可访问，因此本文结合 Coze Loop 公开资料与通用可观测实践输出可落地的 V0.1 方案。

---

## 2. 需求分层（前期必须能力）

### 2.1 核心功能（MVP）

1. **Trace 数据接入**
   - 支持服务端 SDK/HTTP API 上报 Trace、Span、Event、Log。
   - 支持同步（短请求）与异步（批量/流式）两种写入模式。

2. **Trace 检索与查看**
   - 按应用、环境、时间范围、状态（成功/失败）、耗时范围、标签过滤。
   - 支持单条 Trace 详情：调用树、输入输出摘要、错误栈、关键属性。

3. **基础聚合指标**
   - QPS、成功率、P95/P99 延迟、错误码分布。
   - Token 用量、模型调用次数、工具调用失败率。

4. **数据治理**
   - 多租户隔离（workspace/project/app）。
   - 字段脱敏与 PII 保护。
   - 可配置数据保留策略（冷热分层）。

### 2.2 下一阶段能力（MVP+）

- 规则告警（错误率突增、延迟恶化、成本异常）。
- Trace 与评测结果关联（同一会话/请求主键）。
- 离线质量诊断任务（TopN 异常路径、失败模式聚类）。

---

## 3. 总体架构

```text
[AIOS App / Agent Runtime]
        |
   (SDK / OpenTelemetry / HTTP)
        v
[Ingestion Gateway]
  - 鉴权 / 限流 / 协议转换
        v
[Message Queue]
  - Kafka/Pulsar
        v
[Trace Processor]
  - 校验、采样、脱敏、补全、聚合
        |
   +----+------------------------------+
   |                                   |
   v                                   v
[OLTP: MySQL/PostgreSQL]         [OLAP: ClickHouse]
(元数据/配置/索引)               (明细检索/聚合分析)
   |
   v
[Query API / Aggregation API]
   |
   v
[AIOS Console / BI / 告警系统]
```

### 3.1 分层说明

- **接入层（Gateway）**：统一接入协议、鉴权、流控、幂等校验。
- **处理层（Processor）**：规范化数据模型、富化上下文、脱敏与采样。
- **存储层（冷热分离）**：
  - OLTP 保存应用配置、索引元数据、告警规则。
  - OLAP 保存高吞吐 Trace/Span 明细与聚合查询。
- **服务层（Query API）**：面向前端与内部系统提供检索、聚合、导出接口。

---

## 4. 数据模型设计（核心实体）

### 4.1 Trace

- `trace_id`（全局唯一）
- `workspace_id / app_id / env`
- `start_time / end_time / duration_ms`
- `status`（ok/error/timeout）
- `root_input_digest / root_output_digest`
- `cost_total`（tokens、金额估算）
- `tags`（json）

### 4.2 Span

- `span_id / parent_span_id / trace_id`
- `span_type`（agent_step/model_call/tool_call/retrieval/http/db/custom）
- `name`
- `start_time / end_time / duration_ms`
- `status`、`error_type`、`error_message`
- `model_name`、`tool_name`、`token_in/out`
- `attributes`（json）

### 4.3 Event / Log

- 与 `span_id` 关联
- `event_type`（prompt_render、tool_result、exception、retry 等）
- `payload`（支持结构化）

### 4.4 指标宽表（可选）

- `time_bucket`（分钟/5分钟）
- `app_id/env`
- `qps/success_rate/p95/p99/error_rate`
- `token_in/out/cost`

---

## 5. 关键流程

### 5.1 Trace 上报流程

1. SDK 生成 Trace/Span 并携带上下文（app/env/version/user_session）。
2. Gateway 鉴权（AK/SK 或 PAT）、校验 schema、写入 MQ。
3. Processor 异步消费：
   - 数据规范化（字段映射、时间统一 UTC）
   - 敏感字段处理（mask/hash/drop）
   - 异常修正（缺失 parent_span 时挂载 root）
4. 明细写入 ClickHouse；索引摘要写入 OLTP/ES（按规模选型）。
5. Query API 对外提供 Trace 列表与详情查询。

### 5.2 聚合分析流程

- 通过物化视图或流式聚合任务生成分钟级指标；
- 控制台查询优先走聚合表，明细下钻再查 Trace/Span 表。

---

## 6. API 概要

### 6.1 写入接口

- `POST /api/v1/observe/traces`
- `POST /api/v1/observe/spans/batch`
- `POST /api/v1/observe/events/batch`

### 6.2 查询接口

- `GET /api/v1/observe/traces`
- `GET /api/v1/observe/traces/{trace_id}`
- `GET /api/v1/observe/metrics/overview`
- `GET /api/v1/observe/metrics/trend`

### 6.3 鉴权与配额

- Header：`Authorization: Bearer <token>`
- 每应用写入 TPS 限流，超限返回 429。

---

## 7. 非功能设计

### 7.1 性能目标（MVP 建议）

- 写入：峰值 5k spans/s（单集群）。
- 查询：95% 请求 < 1.5s（7 天窗口内）。
- 可用性：99.9%。

### 7.2 可靠性

- Gateway 无状态多副本；MQ 削峰。
- Processor 至少一次消费 + 幂等写入。
- 死信队列处理脏数据与 schema 不兼容数据。

### 7.3 安全与合规

- 传输 TLS；静态数据加密。
- PII 字段白名单上报；默认拒绝原文 Prompt/用户内容（可配置）。
- 审计日志记录查询与导出行为。

---

## 8. 部署拓扑（建议）

- `observe-gateway`（2~4 副本）
- `observe-processor`（按消费 lag 弹性扩缩）
- `mysql/postgres`（主从）
- `clickhouse`（2 shard * 2 replica 起步）
- `kafka`（3 broker）
- `redis`（查询缓存/会话上下文）

---

## 9. 与 Coze Loop“应用观测”能力对齐点

参考 Coze Loop 对外公开能力（全链路观测、SDK Trace 上报、Trace 查询）：

- 我们在 MVP 先实现 **Trace 上报 + Trace 查询 + 基础指标** 三件套。
- 数据模型以 Trace/Span 为主线，覆盖 Prompt 解析、模型调用、工具执行等关键阶段。
- 架构上采用“接入-队列-处理-分析”的解耦方案，便于后续接入评测和告警任务。

---

## 10. 里程碑计划（8 周）

- **W1-W2**：需求冻结、数据模型、接口契约、SDK 最小上报。
- **W3-W4**：Gateway + Processor + ClickHouse 落库，完成 Trace 列表/详情。
- **W5-W6**：指标聚合、查询优化、权限与脱敏。
- **W7**：压测、容灾演练、灰度接入 1~2 个真实应用。
- **W8**：上线与复盘，输出 MVP 运行报告。

---

## 11. 风险与对策

1. **高基数字段导致查询抖动**：限制动态 tag，建立字段治理与索引策略。
2. **上报数据质量参差**：SDK 侧 schema 校验 + 服务端兜底修复。
3. **成本不可控（存储/查询）**：冷热分层、采样策略、聚合优先。
4. **隐私合规风险**：默认最小化采集 + 可追溯审计。

---

## 12. 附录：MVP 验收标准

- 能接入至少 2 类应用（Agent 工作流、普通 LLM 应用）。
- 10 分钟内可看到新上报 Trace。
- 控制台支持按时间/状态/应用过滤并查看调用树。
- 可输出日级别成功率、P95、成本趋势。
