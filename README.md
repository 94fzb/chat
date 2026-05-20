# AIOS 概要设计（持续完善）

## 1. 项目定位

AIOS（AI Operating System）用于统一管理 **多智能体协作、工具调用、上下文记忆与任务编排**，面向“可持续运行的 AI 业务系统”。

目标：
- 降低 AI 应用从 Demo 到生产的复杂度。
- 提供标准化的任务生命周期管理。
- 支持可观测、可回放、可审计的执行链路。

---

## 2. 总体架构

采用分层 + 事件驱动的设计：

1. **接入层（Gateway）**
   - 对外提供 API / Webhook / 消息队列接入。
   - 负责鉴权、限流、租户隔离与请求标准化。

2. **编排层（Orchestrator）**
   - 负责任务拆解、路由、重试策略、并行调度。
   - 支持 Workflow DSL 与状态机编排。

3. **智能体层（Agent Runtime）**
   - 承载不同角色 Agent（Planner/Worker/Reviewer 等）。
   - 管理会话上下文、工具权限与执行预算。

4. **能力层（Tools & Skills）**
   - 统一封装外部工具（搜索、代码执行、数据库、第三方 API）。
   - 提供技能模板（提示词、策略、输入输出协议）。

5. **记忆与知识层（Memory & Knowledge）**
   - 短期记忆：会话窗口与任务缓存。
   - 长期记忆：向量库 + 结构化知识库。
   - 支持检索增强（RAG）和知识版本化。

6. **治理与观测层（Governance & Observability）**
   - 日志、指标、链路追踪、成本统计。
   - 安全审计、策略引擎、敏感信息治理。

---

## 3. 核心模块设计

### 3.1 Task Engine（任务引擎）
- **任务模型**：Task / Step / Artifact / Event。
- **状态流转**：Created → Queued → Running → Succeeded/Failed/Cancelled。
- **容错机制**：幂等键、指数退避重试、死信队列。

### 3.2 Agent Kernel（智能体内核）
- 支持 ReAct / Plan-Execute / Reflection 等策略。
- 内置上下文压缩与摘要机制，控制 Token 成本。
- 对每次工具调用做权限校验与沙箱隔离。

### 3.3 Tool Bus（工具总线）
- 采用统一 ToolSpec 协议：`name / input_schema / output_schema / timeout / permission`。
- 支持同步调用、异步任务、流式结果。
- 标准化错误码与降级策略。

### 3.4 Memory Service（记忆服务）
- 支持语义检索、时间衰减、标签过滤。
- 记忆写入触发规则：高价值结论、用户偏好、长期任务上下文。
- 通过评分机制避免“噪声记忆”污染。

### 3.5 Policy Engine（策略引擎）
- 定义模型选择策略（质量/速度/成本优先）。
- 定义数据访问策略（按租户、角色、资源范围授权）。
- 定义响应合规策略（脱敏、拒答、人工复核）。

---

## 4. 关键流程

### 4.1 请求执行主流程
1. Gateway 接收请求并完成鉴权。
2. Orchestrator 创建 Task 并进行任务拆解。
3. Agent Runtime 拉取 Step，结合 Memory 生成执行计划。
4. 通过 Tool Bus 调用外部能力并回收结果。
5. Orchestrator 汇总产物并写入审计日志。
6. 将最终结果返回调用方，并触发异步评估。

### 4.2 失败恢复流程
- Step 失败后按策略重试。
- 达到重试上限后标记失败并触发补偿任务。
- 对可回放任务保留完整事件流用于复盘。

---

## 5. 数据模型（草案）

- `tasks(id, tenant_id, type, status, priority, created_at, updated_at)`
- `task_steps(id, task_id, agent_role, tool_name, status, input, output, started_at, ended_at)`
- `artifacts(id, task_id, kind, uri, metadata, version)`
- `memories(id, tenant_id, scope, content, embedding, score, tags, ttl)`
- `policies(id, tenant_id, policy_type, rule, enabled, version)`

---

## 6. 非功能性设计

- **可靠性**：核心链路高可用，任务可恢复。
- **可扩展性**：支持多租户横向扩展与模型/工具热插拔。
- **可观测性**：端到端 Trace + 成本/质量指标看板。
- **安全性**：最小权限、数据加密、操作审计。
- **性能目标（初稿）**：
  - P95 首 token 延迟 < 2s（轻量任务）
  - 异步任务吞吐可水平扩展

---

## 7. 里程碑规划（建议）

### M1（基础可用）
- 单 Agent 执行链路
- 工具总线最小集
- 基础任务状态机 + 日志

### M2（协作增强）
- 多 Agent 协作与任务拆解
- 记忆服务接入（RAG）
- 失败恢复与回放

### M3（生产治理）
- 策略引擎与权限系统
- 全链路观测 + 成本治理
- 灰度发布与 A/B 评估

---

## 8. 下一步待补充

- Workflow DSL 语法与示例。
- Agent 角色协议（输入输出契约）。
- 评估体系（质量、时延、成本三维度）。
- 运维手册（告警、扩缩容、应急预案）。

