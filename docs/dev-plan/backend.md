# Backend Plan (apps/api)

范围：FastAPI + StarRocks 数据源 + Query Service + 安全/审计。

StarRocks 说明（MVP 约束）：
- StarRocks 兼容 MySQL 协议；后端连接实现要封装为可替换的 DB adapter。
- Query Service 是唯一允许访问数据库的通道；Agent 不能直连 DB。

## M0 工程化骨架 (2-4 天)

- 项目初始化
  - 创建 `apps/api/`（FastAPI，Python 3.10）
  - 目录落地：`routers/ services/ repositories/ schemas/ models/`
  - 配置：`Settings`（环境变量），`.env.example`
- 质量工具
  - Ruff/Black/isort/mypy 配置与脚本（按工程规范）
  - pytest + httpx 集成测试样例
- API 基础设施
  - 统一错误模型：`ErrorCode` + `trace_id`，错误文案必须可操作
  - Request tracing：中间件生成/透传 trace_id

验收：
- `uvicorn` 可启动；基础 healthcheck；lint/test 可跑。

## M1 端到端最小链路（先不做澄清/KB）(3-5 天)

- API 设计（MVP）
  - `POST /api/chat/message`：接收用户消息（MVP 可先支持固定结构化消息/或自然语言直通）
    - 返回 `type=clarify` 或 `type=result`
  - `GET /api/health`：依赖检查（包含 StarRocks 连通性可选）
- Query Service (StarRocks)
  - 建立 StarRocks 连接（封装 connector/engine）
  - 统一超时、最大行数、分页策略（MVP 至少强制 limit/topN）
  - 结果标准化：列 schema + rows + 简单统计（行数/耗时）

验收：
- 后端能把固定 SQL/结构化查询跑到 StarRocks 并返回标准响应。

## M2 会话 + 澄清支撑接口 (5-7 天)

- 会话模型（MVP）
  - `session_id` 生成与存储（先内存/Redis 可选；MVP 可 DB 表）
  - `GET /api/chat/session/{session_id}`（可选）：拉取会话历史，支持前端刷新恢复
  - `POST /api/chat/message`：接收用户消息，返回：需要澄清 or 最终结果
- Trace + 可观测
  - 日志：结构化日志包含 session_id、trace_id、user_id、耗时
  - 指标（可选）：请求耗时、DB 查询耗时、错误码统计

验收：
- 同一 session 下可复用澄清信息；可定位单次请求。

## M3 KB（语义层）+ SQL 生成支撑 (7-10 天)

- KB Service (MVP)
  - 数据模型：表/字段、指标定义、表关系
  - 管理接口（管理员）：最小 CRUD（可先只写入固定 seed 数据）
  - 检索接口：给 Agent 提供候选（指标/维度/关系）
- Query Service 增强
  - StarRocks 方言约束：
    - 强制 limit
    - 禁止写操作（DDL/DML）
    - 可选：`EXPLAIN` / 估算扫描量（若可行）
  - SQL 校验：语句白名单/黑名单、select-only、禁止多语句

验收：
- 给定 KB 配置，后端能辅助 Agent 生成并执行正确 SQL。

## M4 权限/脱敏/审计 (5-7 天)

- 鉴权
  - JWT（MVP），用户/角色模型（admin/user）
  - 表/字段级授权策略（最小可用）
- 脱敏
  - 敏感字段清单 + 规则（mask/replace/role-based reveal）
  - 返回前脱敏（在 Query Service 统一做）
- 审计
  - 记录：user_id、session_id、intent、tables/fields、sql_hash、耗时、是否导出
  - 查询审计检索接口（管理员）

验收：
- 无权限不可查；敏感字段按规则脱敏；审计可追溯（>= trace_id）。

## M5 稳定性 + 测试门禁 (5-7 天)

- 测试
  - 单元：SQL 校验、脱敏、权限策略
  - 集成：httpx 启动测试服务 + StarRocks 测试库（可用 docker/测试环境）
- 可靠性
  - DB 超时、重试策略（谨慎，避免放大压力）
  - 取消查询（如 StarRocks/driver 支持；否则在 API 层暴露“取消占位”）

验收：
- pytest 可跑；关键安全逻辑有覆盖；失败可定位。
