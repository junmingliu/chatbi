# Dev Plan (MVP: 数据洞察归因)

目标：优先交付“数据洞察归因/灵活分析”主链路。

本目录把同一组里程碑 (M0-M5) 按 **前端 / 后端 / Agent** 拆成可执行任务清单，便于并行推进。

文件：
- `docs/dev-plan/frontend.md`：Web UI + 交互 + 质量门禁（Vitest/Playwright）
- `docs/dev-plan/backend.md`：FastAPI + StarRocks Query Service + 安全审计
- `docs/dev-plan/agent.md`：LangGraph 澄清/分析图 + SQL 生成链路 + 可解释性

MVP 约定（建议作为前后端/Agent 的第一版接口契约）：
- `POST /api/chat/message`
  - 输入：`session_id?` + `message`（用户自然语言）
  - 输出：两种之一
    - `type=clarify`：返回澄清问题 + 需要补齐的 slots
    - `type=result`：返回结论/表格/图表/逻辑说明
- `GET /api/chat/session/{session_id}`（可选）：拉取会话历史（用于刷新恢复）
- `GET /api/health`：健康检查

响应字段最小集合（前端渲染依赖）：
- `trace_id`：每次请求都返回（用于定位/审计）
- `summary`：结论文本
- `table`：`columns[]` + `rows[]`
- `charts[]`：统一 chart spec（MVP 先支持 1-2 种图）
- `explain`：意图解析/数据来源/脱敏 SQL/计算逻辑

约束（来自现有文档）：
- UI 不直接拼 SQL；Agent 不直连 DB（必须通过 Query Service）。
- 后端：Python 3.10 + FastAPI；数据访问：SQLAlchemy 2.x + Alembic。
- 前端：React + TS (strict) + Vite + React Router + TanStack Query。
- 测试：前端 Vitest；后端 pytest；E2E Playwright。
