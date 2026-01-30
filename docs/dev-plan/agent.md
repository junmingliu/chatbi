# Agent Plan (agent/)

范围：LangGraph 编排“澄清 + 灵活分析”，并通过 Tool 层调用后端 Query Service（不直连 StarRocks）。

核心原则：
- Node 必须有类型（TypedDict/Pydantic）；职责单一。
- Tool 必须参数校验 + 权限控制 + 可观测（耗时/命中率/错误类型）。

## M0 目录骨架 + 规范落地 (2-4 天)

- 建立目录
  - `agent/graphs/`：图定义
  - `agent/tools/`：query/kb/security/audit 等工具
  - `agent/prompts/`：system policy 与业务提示分离、版本化
  - `agent/eval/`：评测数据集结构占位
- 运行方式（MVP）
  - 提供一个本地 runner（CLI/脚本）可调用图（先不要求接 UI）

验收：
- 能在本地跑通一个最小图（mock 工具返回）。

## M1 最小分析链路（无澄清/KB，先通工具调用）(3-5 天)

- 定义输出协议（面向前端）
  - `summary`：结论文本
  - `table`：columns + rows
  - `chart`：chart spec（统一选 ECharts 或 Recharts 之一，MVP 建议 ECharts spec）
  - `explain`：逻辑说明（占位字段）
- Tool: Query
  - `query.execute`：调用后端 `POST /api/chat/message`（MVP 统一入口；后续可拆 `/api/query/execute`）
  - 参数校验：只允许 select；强制 limit；禁止敏感字段（由后端最终裁决）

验收：
- Agent 能把结构化 query 交给后端并返回标准 payload。

## M2 Clarify Graph（澄清图）(5-7 天)

- Nodes
  - `parse_intent`：从自然语言抽取候选（指标/维度/时间/过滤条件）
  - `detect_missing_slots`：识别缺失项（时间/地区/口径/维度范围）
  - `ask_clarify`：生成澄清问题（简洁明确，避免引导）
  - `finalize_context`：合并上下文（支持用户修改已澄清信息）
- 状态
  - session state schema：保存已确认的 slots
  - 兼容“继续追问/追问后执行”两种分支

验收：
- 缺失项识别命中率可在小样本 eval 上度量；澄清问句稳定。

## M3 Ad-hoc Analysis Graph（灵活分析图）+ KB 检索 (7-10 天)

- Nodes
  - `retrieve_kb`：调用 KB Tool 获取指标/维度/表关系/口径候选
  - `plan_query`：选择事实表/维表/join 路径与聚合粒度
  - `generate_sql`：生成 StarRocks 兼容 SQL（select-only，limit）
  - `sql_guardrail`：二次校验（成本/权限/脱敏提示）
  - `summarize`：生成结论（避免冗余）
  - `pick_chart`：按目标选择图表（趋势/对比/占比/相关性）
  - `render_payload`：组装给前端的标准输出
- Tool: KB
  - `kb.search_metrics` / `kb.search_dimensions` / `kb.get_relations`
- StarRocks 方言注意（在 generate_sql/guardrail 体现）
  - 强制 limit/topN
  - 时间函数/日期截断统一策略（封装为可替换的 dialect helper）
  - 禁止多语句；禁止写入

验收：
- 在 KB 已配置的代表性问题上能生成正确 SQL 并出结果。

## M4 安全/审计/可解释性完善 (5-7 天)

- Tool: Security/Audit
  - 调用后端安全策略：字段级可见性、脱敏策略、审计写入
  - Agent 输出必须带 `trace_id`（透传）
- Explainability（逻辑说明）
  - 意图解析（用户问题 -> 指标/维度/过滤）
  - 数据来源（表/字段）
  - SQL（脱敏，可复制）
  - 计算逻辑（口径/公式）

验收：
- 每次结果都可解释、可追溯（trace_id + 审计）。

## M5 Eval/回归门禁 (5-7 天)

- eval 数据集（先覆盖场景一相关）
  - `single_dim/`、`multi_dim/`、`need_clarify/` 至少各 100（可先 20 起步再扩）
  - 每条包含：query、expected_clarify、expected_sql_shape、expected_metrics、tolerance
- 自动评测
  - 统计：SQL 成功率、准确率、澄清识别率
  - 失败样例落盘（方便回归）

验收：
- eval 可在 CI 或本地跑出报告，作为回归门禁基础。
