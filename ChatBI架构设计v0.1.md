# ChatBI 架构设计（v0.1）

## 1. 技术栈约束（必须遵守）

### 1.1 前端（React）

* 框架：React **最新稳定版**（截至 2026-01，React 19.2.x 为最新版本之一）。([React][1])
* 工程化：TypeScript（强制）、Vite（推荐）、ESM
* UI：现代简约 + 浅蓝色主题（见 UI 规范）
* 图表：优先 ECharts 或 Recharts（两者择一统一）
* 数据请求：TanStack Query（React Query）
* 路由：React Router
* 状态：优先“服务端状态用 Query，本地 UI 状态用 Zustand/Context”，避免全局 Redux 过度设计

### 1.2 后端（Python）

* 版本：Python **3.10**（注意其上游 EOL 在 2026 年 10 月左右，需要在路线图里预留升级窗口）。([Python Developer's Guide][2])
* Web 框架：FastAPI（最新稳定版以 PyPI 为准；截至 2025-12 发布的 0.128.0）。([PyPI][3])
* 数据访问：SQLAlchemy 2.x + Alembic
* 鉴权：JWT / OIDC（企业内推荐对接 SSO）
* 异步：优先 async（FastAPI 原生），I/O 密集任务可用 Celery/RQ（可选）

### 1.3 Agent（LangGraph）

* Agent 编排：LangGraph **最新稳定版**（PyPI 显示 2026-01 仍在持续发布，且 v1 为稳定主线）。([PyPI][4])
* LLM 接入：抽象为 Provider Interface（OpenAI/本地/其他云），避免业务代码绑定单一模型
* 工具调用：SQL 生成/校验/执行、知识库检索、模板渲染、导出等全部通过 Tool 层统一管理

---

## 2. 代码仓库结构

```
chatbi/
  apps/
    web/                      # React 前端
    api/                      # FastAPI 后端
  packages/
    shared/                   # 前后端共享类型/协议（可选）
  agent/
    graphs/                   # LangGraph 图定义
    prompts/                  # 系统提示词、策略提示词
    tools/                    # 工具：sql、kb、template、export、policy
    eval/                     # agent 评测集与回归
  docs/
    prd/                      # 需求与口径
    adr/                      # 架构决策记录
    runbook/                  # 运维手册
  infra/
    docker/                   # 本地开发与测试环境
    k8s/                      # 生产部署（可选）
  .github/
    workflows/                # CI
```

## 6. 系统功能架构设计（Functional Architecture）

> 需求要求：多轮澄清、模板化报告、知识库（元数据/指标/表关系）、数据安全脱敏审计、准确性保障与可解释性。

### 6.1 分层架构（建议）

* **Presentation 层（Web）**

  * Chat UI、结果视图、模板管理、权限管理、审计管理
* **API 层（FastAPI）**

  * Auth、Chat API、Template API、Query API、Export API、Audit API、KB Admin API
* **Agent 层（LangGraph）**

  * Clarify Graph（澄清）
  * Ad-hoc Analysis Graph（灵活分析）
  * Template Report Graph（固定模板）
  * Guardrail（权限/脱敏/成本）
* **Data 层**

  * OLAP/DB（MySQL/ClickHouse/Hive…）
  * 元数据与语义库（指标/维度/关系/同义词）
  * 向量库（可选，用于口径/字段检索）
  * 审计库（查询与导出日志）

### 6.2 核心服务拆分

1. **Chat Orchestrator（对话编排）**
2. **Semantic/KB Service（语义与口径）**：元数据、指标定义、表关系维护与检索
3. **Query Service（SQL 工具链）**：生成、校验、执行、限流、超时
4. **Template Service（模板与版本）**：模板配置、版本管理、发布、回溯
5. **Export Service（导出）**：PDF/Excel
6. **Security & Audit（安全审计）**：字段脱敏、日志保留≥1年

---

## 7. 功能设计（按场景 + 模块拆解）

### 7.1 场景一：数据洞察归因（灵活分析）

**目标**：多维交叉分析 + 自动图表 + 结论输出 + 支持切换图表。

#### 7.1.1 交互流程（落到系统行为）

1. 用户提问（自然语言）
2. Agent 进行**模糊项识别**（时间/地区/指标口径/维度范围），缺失则澄清（准确率≥98%）
3. 检索知识库：指标定义、可拆分维度、表关系（为 SQL 生成提供依据）
4. 生成 SQL → 安全校验（权限/脱敏/成本/limit）→ 执行
5. 输出：结论 + 图表 + 表格
6. 支持“追问/切换图表/修改筛选条件”进入下一轮

#### 7.1.2 必备输出结构

* 摘要结论（简洁精准，避免冗余）
* 图表（默认推荐，支持切换）
* 明细/汇总表（默认分页或 TopN）
* **逻辑说明入口**（意图→来源→SQL→计算逻辑，SQL 脱敏且可复制）

---

### 7.2 场景二：数据总结（固定模板分析）

**目标**：按模板路径取数、出图、生成标准报告、支持导出 PDF/Excel、模板版本可回溯。

#### 7.2.1 模板模型（建议最小字段集）

* `template_id / name / description`
* `parameters[]`：必填/可选、类型、默认值、校验规则
* `sections[]`：

  * `title`
  * `queries[]`（每段可多个查询）
  * `charts[]`（图表类型、字段映射、排序、TopN）
  * `narrative_rules`（可选：自动结论规则）
* `version / status(draft|published) / created_by / created_at`

#### 7.2.2 模板执行流程

1. 用户选择模板 → 填参数
2. 系统按预设路径执行查询并做数据完整性校验
3. 生成图表与报告
4. 支持导出 PDF/Excel，允许补充人工备注

---

### 7.3 核心模块功能清单（MVP 必做）

#### A. 多轮对话澄清

* 缺失项检测：时间/地区/指标口径/维度范围（≥98%）
* 上下文延续：同一会话复用澄清结果
* 用户可修改已澄清信息并触发重算

#### B. 数据安全与权限 + 脱敏 + 审计

* 表/字段级权限按角色/部门配置
* 敏感字段脱敏（掩码/替换/权限可见）
* 查询与导出行为审计、日志保留≥1年

#### C. 准确性保障 + 数据校验

* 准确率 > 95%，数值误差 ≤ ±1%
* 四类测试集，每类≥100，并衡量澄清有效性与模板匹配
* 结果校验：缺失值/异常值/同比环比逻辑一致性，异常提示

#### D. 可解释性

* 逻辑说明：意图解析→数据来源→SQL（脱敏可复制）→计算逻辑

#### E. 本地知识库（语义层）

* 元数据管理（库/表/字段/类型/说明/标签）
* 指标定义（公式、口径、适用场景、可拆维度）
* 表关系管理（Join 依据 + 关系图谱）
* 更新可回溯，修改后同步到分析参考库

---

## 8. LangGraph 参考图谱

### 8.1 Clarify Graph（澄清图）

* Nodes：`parse_intent` → `detect_missing_slots` → `ask_clarify` / `fill_defaults` → `finalize_context`
* Output：结构化上下文（时间范围、主体、指标、维度、输出偏好）

### 8.2 Ad-hoc Analysis Graph（灵活分析图）

* Nodes：`retrieve_kb` → `plan_query` → `generate_sql` → `sql_guardrail` → `execute_sql` → `summarize` → `pick_chart` → `render_payload`

### 8.3 Template Report Graph（模板图）

* Nodes：`load_template(version)` → `validate_params` → `run_section_queries` → `compose_report` → `export_pdf_excel`

---

## 9. 性能与SLO（直接映射需求）

* 单维查询 ≤ 3s；多维交叉 ≤ 5s；模板报告 ≤ 10s
* 并发：在线 ≤ 500，并发查询 ≤ 100/s

---

## 10. 交付清单（你用 AI coding 从 0 开始时的“第一阶段 Done Definition”）

* 前端：聊天 + 结果 Tabs（表/图/逻辑说明）
* 后端：Auth、Chat API、Query API、Template API（最小）
* Agent：澄清图 + 灵活分析图（支持一条 DB）+ 逻辑说明输出
* 知识库：元数据/指标/表关系的最小管理与检索
* 安全：字段脱敏 + 审计日志
* 测试：四类 eval 集（每类≥100）+ CI 回归门禁

---
