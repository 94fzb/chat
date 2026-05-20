# AIOS 可观测性概要设计（Observability）

## 1. 目标与范围

可观测性模块用于回答三类核心问题：
- **发生了什么**：任务、步骤、Agent、Tool 在何时以何状态执行。
- **为什么发生**：失败原因、延迟瓶颈、策略命中、上下文质量。
- **影响有多大**：对成功率、成本、时延、用户体验的影响范围。

范围覆盖：
- 在线执行链路（同步/异步任务）
- 多 Agent 协作过程
- Tool 调用与外部依赖
- Memory 检索与写入效果
- 策略引擎决策结果

---

## 2. 观测模型（Telemetry Model）

采用 **Logs + Metrics + Traces + Events** 四元模型。

### 2.1 Logs（结构化日志）
- 统一 JSON 日志格式，关键字段：
  - `timestamp`
  - `tenant_id`
  - `task_id`
  - `step_id`
  - `agent_id`
  - `tool_name`
  - `level`
  - `error_code`
  - `latency_ms`
  - `token_in/token_out`
  - `cost_usd`
- 日志等级：DEBUG / INFO / WARN / ERROR。
- 禁止输出原始敏感信息（PII、密钥、原始凭证）。

### 2.2 Metrics（指标）
按 RED + USE 思路抽象：

- **请求级**
  - `task_qps`
  - `task_success_rate`
  - `task_p95_latency_ms`
  - `task_timeout_rate`

- **步骤级**
  - `step_retry_count`
  - `step_failure_rate`
  - `step_p95_latency_ms`

- **模型级**
  - `llm_token_input`
  - `llm_token_output`
  - `llm_cost_usd`
  - `llm_first_token_latency_ms`

- **工具级**
  - `tool_call_qps`
  - `tool_error_rate`
  - `tool_p95_latency_ms`

- **记忆级**
  - `memory_hit_rate`
  - `memory_retrieval_latency_ms`
  - `memory_write_accept_rate`

### 2.3 Traces（链路追踪）
采用分布式 Trace，最小 Span 规范：
- `task.root`
- `orchestrator.plan`
- `agent.step`
- `tool.call`
- `memory.retrieve`
- `memory.write`
- `policy.evaluate`

关联字段：
- `trace_id`
- `span_id`
- `parent_span_id`
- `task_id`
- `tenant_id`

### 2.4 Events（领域事件）
关键事件：
- `task.created`
- `task.started`
- `task.succeeded`
- `task.failed`
- `step.retried`
- `policy.blocked`
- `tool.degraded`

事件用于：
- 异步告警触发
- 审计与回放
- 质量评估样本抽取

---

## 3. 指标看板设计

### 3.1 SLO 看板（面向平台）
- 可用性：任务成功率（按租户/场景拆分）
- 性能：P50/P95/P99 延迟
- 成本：每任务平均成本、每租户日成本
- 稳定性：超时率、重试率、降级率

### 3.2 质量看板（面向算法/产品）
- Agent 任务完成率
- 人工复核通过率
- 策略拦截命中率（误杀/漏杀）
- RAG 命中率与引用有效率

### 3.3 运维看板（面向 SRE）
- 队列堆积深度
- Worker 并发利用率
- 外部依赖 SLA（搜索、数据库、第三方 API）
- 异常 TopN（error_code + 触发频次）

---

## 4. 告警策略（Alerting）

分级告警：
- **P1**：核心链路不可用（成功率骤降、任务大面积超时）
- **P2**：功能退化（部分 Tool 失败率升高、延迟明显上升）
- **P3**：趋势异常（成本偏离、命中率下滑）

建议规则示例：
- `task_success_rate < 99%` 持续 5 分钟（P1）
- `task_p95_latency_ms > 4000` 持续 10 分钟（P2）
- `llm_cost_usd` 日环比上升 > 30%（P3）

抑制与去重：
- 基于 `tenant_id + scene + error_code` 聚合。
- 告警冷却时间与风暴保护（避免重复通知）。

---

## 5. 审计与回放

- 所有关键任务保存事件流与关键输入输出摘要。
- 回放模式支持：
  - 仅回放编排逻辑（不触发真实 Tool）
  - 半实物回放（部分 Tool 使用桩）
  - 全链路回放（灰度环境）
- 用途：
  - 事故复盘
  - 策略变更前后效果对比
  - 模型升级回归验证

---

## 6. 数据治理与合规

- 采集前脱敏：手机号、邮箱、身份证号等字段做掩码。
- 存储分级：热数据（7~30 天）、温数据（90 天）、冷归档（合规周期）。
- 访问控制：按租户 + 角色最小权限读取观测数据。
- 合规支持：导出审计报告、关键操作留痕。

---

## 7. 技术选型建议（可替换）

- Metrics：Prometheus / VictoriaMetrics
- Logs：Loki / Elasticsearch
- Traces：OpenTelemetry + Tempo / Jaeger
- Dashboards：Grafana
- Alerting：Alertmanager + OnCall

说明：
- 采用 OpenTelemetry 作为统一埋点协议，避免厂商锁定。
- 先统一语义模型，再落地具体存储后端。

---

## 8. 落地里程碑（Observability）

### O1（最小可用）
- 打通 task/step 基础日志与核心指标。
- 建立任务成功率 + P95 延迟看板。
- 上线 P1/P2 基础告警。

### O2（链路增强）
- 接入分布式 Trace。
- Tool / Memory / Policy 全链路埋点。
- 增加租户维度成本分析。

### O3（质量治理）
- 建立质量评估与可观测数据联动。
- 支持回放驱动的回归检查。
- 建立成本-质量-时延三维优化闭环。

---

## 9. 与主架构文档关系

本文件是对 `README.md` 中 “治理与观测层” 的展开设计。
后续建议在 README 中增加 docs 导航索引，并拆分：
- `docs/orchestration.md`
- `docs/memory.md`
- `docs/policy.md`
- `docs/evaluation.md`

