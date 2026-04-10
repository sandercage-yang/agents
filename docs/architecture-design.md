# 企业级 AI 自动化研发平台 — 架构设计文档

> 版本：1.0
> 日期：2026-04-10
> 状态：已评审

---

## 一、平台概述

### 1.1 目标

构建一个 **AI 驱动的端到端自动化研发平台**（以下简称"平台"），将传统软件工程中"需求→设计→编码→测试→交付"的人力密集型流程，转变为 AI 自主执行、人工关键审批的智能化流水线。

### 1.2 业务流程

```
需求输入 → 需求分析 → PRD 产出 → 技术方案 → [人工 Review]
                                                    │
           部署完成 ← CI/CD ← [人工 Approval] ← 测试审查 ← 测试执行 ← 编码实现
```

### 1.3 核心理念

**确定性工作流 + 关键节点嵌入 Agent**。

平台不采用"多 Agent 协作对话"架构。全部 9 个流程环节中，只有"编码实现"和"测试审查"需要 Agent（LLM 自主决策），其余环节用确定性工作流编排 + LLM 单次调用即可。Agent 节点之间不互相通信，由工作流引擎串联，避免多 Agent 协作的级联幻觉、调试困难和协调开销。

---

## 二、设计原则

| # | 原则 | 说明 |
|---|------|------|
| 1 | **能用工作流就不用 Agent** | 只有当任务路径无法预定义、需要 LLM 自主循环迭代时才使用 Agent。LLM 单次调用和确定性逻辑优先。|
| 2 | **节点可插拔** | 任意节点可在 LLM Call / Workflow / Agent SubGraph 三种形态间切换，增减节点不影响全局架构。 |
| 3 | **状态归工作流管** | 全局状态由工作流引擎（LangGraph StateGraph）统一持有和持久化，Agent 节点只读取输入、写回产出。 |
| 4 | **人控关键节点** | 技术方案和最终交付前必须经过人工审批（LangGraph Interrupt），任何自主执行不可绕过审批门。 |
| 5 | **可观测优先** | 全链路 Tracing（LangSmith + OpenTelemetry）从 Day 1 就接入，不做"黑盒"运行。 |
| 6 | **沙箱隔离** | 所有代码执行（Coding Agent）在隔离容器中运行，与宿主环境和生产数据物理隔离。 |
| 7 | **渐进式复杂度** | 先以最简方案跑通全流程，有证据表明不够用时再升级（如节点升级为 Agent、引入并行编码）。 |

---

## 三、技术栈

| 层面 | 选型 | 说明 |
|------|------|------|
| **编程语言** | Python 3.12+ | LangGraph/LangChain 原生语言，AI/ML 生态最强 |
| **工作流引擎** | LangGraph | LLM 调用一等公民、Interrupt 审批、SubGraph 模块化、LangSmith 可观测 |
| **Web 框架** | FastAPI | 异步原生、自动 OpenAPI 文档 |
| **数据验证** | Pydantic v2 | LangChain 生态标准 |
| **Coding Agent** | Open SWE（首选）/ OpenHands（备选） | LangGraph 原生集成 / 最强编码能力 + 企业治理 |
| **Review Agent** | 自建 LangGraph SubGraph | 可按企业规范定制审查规则 |
| **状态持久化** | PostgreSQL | LangGraph Checkpointer 原生支持 |
| **向量存储** | pgvector（PostgreSQL 扩展） | RAG 检索，复用 PostgreSQL 基础设施 |
| **缓存/消息** | Redis | 异步任务通知、缓存 |
| **容器化** | Docker + Kubernetes | Agent 沙箱隔离、生产部署 |
| **可观测性** | LangSmith + OpenTelemetry | LLM 链路 + 基础设施双覆盖 |
| **LLM** | Claude Sonnet/Opus（主力） + DeepSeek（降本） | 支持模型路由、本地模型备选 |

---

## 四、核心架构

### 4.1 系统分层

```
┌──────────────────────────────────────────────────────────────────────┐
│  接入层 (Entry Layer)                                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐          │
│  │ Web 管理台│  │Slack/钉钉│  │ API 调用  │  │ Git 事件   │          │
│  │  (审批UI) │  │  Bot     │  │(REST/gRPC)│  │(Webhook)   │          │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └─────┬──────┘          │
├───────┼──────────────┼─────────────┼──────────────┼─────────────────┤
│  编排层 (Orchestration Layer)                                        │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │               LangGraph StateGraph (MainAgent)                │   │
│  │                                                               │   │
│  │  ┌──────┐ ┌─────┐ ┌──────┐ ┌────────┐ ┌──────┐ ┌─────────┐  │   │
│  │  │需求   │→│PRD  │→│技术  │→│人工     │→│编码  │→│测试执行 │  │   │
│  │  │分析   │ │产出 │ │方案  │ │Review  │ │实现  │ │        │  │   │
│  │  │(LLM) │ │(LLM)│ │(RAG) │ │(中断)  │ │(Agent│ │(CI触发)│  │   │
│  │  └──────┘ └─────┘ └──────┘ └────────┘ │SubG) │ └───┬────┘  │   │
│  │                                        └──────┘     │       │   │
│  │  ┌─────────┐  ┌────────────┐  ┌──────────┐         │       │   │
│  │  │CI/CD    │←─│ 人工       │←─│测试审查  │←────────┘       │   │
│  │  │集成     │  │ Approval   │  │(Agent    │                 │   │
│  │  │(API调用)│  │ (中断)     │  │ SubG)    │                 │   │
│  │  └─────────┘  └────────────┘  └──────────┘                 │   │
│  └───────────────────────────────────────────────────────────────┘   │
├──────────────────────────────────────────────────────────────────────┤
│  Agent 层 (Agent Layer)                                              │
│  ┌─────────────────────────┐  ┌─────────────────────────────────┐   │
│  │  Coding Agent           │  │  Review Agent                    │   │
│  │  (Open SWE SubGraph)    │  │  (自建 LangGraph SubGraph)       │   │
│  │  ┌───────────────────┐  │  │  ┌───────────────────────────┐   │   │
│  │  │ Deep Agents 脚手架 │  │  │  │ diff 分析 → 上下文检索    │   │   │
│  │  │ (对外透明)         │  │  │  │ → LLM 评估 → 审查报告     │   │   │
│  │  └───────────────────┘  │  │  └───────────────────────────┘   │   │
│  └─────────────────────────┘  └─────────────────────────────────┘   │
├──────────────────────────────────────────────────────────────────────┤
│  数据层 (Data Layer)                                                 │
│  ┌──────────┐  ┌───────────┐  ┌───────────┐  ┌────────────────┐    │
│  │PostgreSQL│  │pgvector   │  │Redis      │  │对象存储        │    │
│  │(状态/     │  │(向量索引/ │  │(缓存/     │  │(制品/日志      │    │
│  │ 审计日志) │  │ RAG)      │  │ 消息队列) │  │ 存档)          │    │
│  └──────────┘  └───────────┘  └───────────┘  └────────────────┘    │
├──────────────────────────────────────────────────────────────────────┤
│  基础设施层 (Infrastructure Layer)                                    │
│  ┌──────────┐  ┌───────────┐  ┌───────────┐  ┌────────────────┐    │
│  │Docker/K8s│  │LangSmith  │  │OpenTele-  │  │企业内部服务     │    │
│  │(沙箱)    │  │(LLM链路)  │  │metry(基础)│  │(CI/CD/SSO等)   │    │
│  └──────────┘  └───────────┘  └───────────┘  └────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 节点类型体系

架构中每个流程节点属于三种类型之一，可按需切换：

| 类型 | 控制方 | 执行方式 | 适合场景 | 当前使用的节点 |
|------|--------|---------|---------|--------------|
| **LLM Call** | 开发者（预定义 Prompt） | 单次 LLM 调用，无循环 | 格式化文档生成 | 需求分析、PRD 产出、技术方案 |
| **Workflow** | 开发者（预定义逻辑） | 确定性代码/API 调用 | 审批等待、CI 触发 | 人工 Review、测试执行、人工 Approval、CI/CD |
| **Agent SubGraph** | LLM（自主决策） | 循环迭代，工具调用 | 开放性任务 | 编码实现、测试审查 |

**节点升级路径**：任何 LLM Call 节点都可以升级为 Agent SubGraph（替换节点实现函数即可），工作流拓扑无需改动。

---

## 五、工作流详细设计

### 5.1 全局状态定义

```python
from typing import TypedDict, Optional
from datetime import datetime

class PlatformState(TypedDict):
    # ── 输入 ──
    task_id: str                        # 任务唯一标识
    requirement: str                    # 原始需求描述
    repo_url: str                       # 目标代码仓库
    branch_name: str                    # 工作分支名

    # ── 流程产物 ──
    requirement_analysis: str           # 需求分析结果
    prd: str                            # PRD 文档
    tech_design: str                    # 技术方案文档
    coding_result: dict                 # 编码结果 {pr_url, summary, files_changed, diff}
    test_execution_result: dict         # 测试执行结果 {passed, failed, report_url}
    review_report: dict                 # 审查报告 {verdict, issues, suggestions}

    # ── 审批 ──
    tech_review_decision: str           # 技术方案审批: approved / rejected / revision_needed
    tech_review_feedback: str           # 审批反馈意见
    final_approval_decision: str        # 最终审批: approved / rejected
    final_approval_feedback: str        # 审批反馈意见

    # ── 元信息 ──
    created_at: datetime
    updated_at: datetime
    current_stage: str                  # 当前所处阶段（用于 UI 展示）
    error: Optional[str]                # 错误信息（如有）
```

### 5.2 工作流状态机

```
                    ┌──────────────────────────────────┐
                    │                                  │
                    ▼                                  │
┌───────┐    ┌───────────┐    ┌──────┐    ┌────────┐  │
│ START │───→│ requirement│───→│ prd  │───→│tech_   │  │
└───────┘    │ _analysis  │    │      │    │design  │  │
             └───────────┘    └──────┘    └───┬────┘  │
                                              │       │
                                              ▼       │
                                        ┌──────────┐  │
                                        │tech_     │  │
                                        │review    │──┘  rejected / revision_needed
                                        │(interrupt)│
                                        └────┬─────┘
                                             │ approved
                                             ▼
                                        ┌──────────┐
                                        │ coding   │
                                        │(SubGraph)│◄────────────┐
                                        └────┬─────┘             │
                                             │                   │
                                             ▼                   │
                                        ┌──────────┐             │
                                        │test_     │             │
                                        │execute   │             │
                                        └────┬─────┘             │
                                             │                   │
                                             ▼                   │
                                        ┌──────────┐             │
                                        │test_     │             │
                                        │review    │             │
                                        │(SubGraph)│             │
                                        └────┬─────┘             │
                                             │                   │
                                             ▼                   │
                                        ┌──────────┐             │
                                        │final_    │             │
                                        │approval  │─────────────┘  rejected
                                        │(interrupt)│
                                        └────┬─────┘
                                             │ approved
                                             ▼
                                        ┌──────────┐
                                        │ cicd     │
                                        └────┬─────┘
                                             │
                                             ▼
                                        ┌───────┐
                                        │  END  │
                                        └───────┘
```

### 5.3 各节点详细设计

#### 节点 1：需求分析 (requirement_analysis)

| 属性 | 值 |
|------|---|
| **类型** | LLM Call |
| **输入** | `requirement`（原始需求） |
| **输出** | `requirement_analysis`（结构化需求规格） |
| **LLM** | Claude Sonnet |
| **RAG** | 检索企业知识库（历史需求、产品文档、技术规范） |
| **Prompt 策略** | 结构化模板输出：功能点列表、约束条件、依赖关系、风险点 |
| **未来升级** | 如需多轮检索多个知识库，可升级为 Agent SubGraph |

#### 节点 2：PRD 产出 (prd)

| 属性 | 值 |
|------|---|
| **类型** | LLM Call |
| **输入** | `requirement_analysis` |
| **输出** | `prd`（结构化 PRD 文档） |
| **LLM** | Claude Sonnet |
| **Prompt 策略** | 参考 MetaGPT ProductManager 角色的 SOP Prompt 设计。输出含：产品概述、用户故事、功能/非功能需求、验收标准、优先级 |

#### 节点 3：技术方案 (tech_design)

| 属性 | 值 |
|------|---|
| **类型** | LLM Call + RAG |
| **输入** | `prd` + 代码库结构（通过 RAG 检索） |
| **输出** | `tech_design`（技术方案文档） |
| **LLM** | Claude Sonnet |
| **RAG** | 检索目标代码库的架构文档、已有模块结构、API 规范 |
| **Prompt 策略** | 参考 MetaGPT Architect 角色的 SOP Prompt。输出含：架构设计、模块划分、API 定义、数据模型、实现步骤 |
| **未来升级** | 如需自主探索代码库并迭代方案，可升级为 Agent SubGraph |

#### 节点 4：人工 Review (tech_review)

| 属性 | 值 |
|------|---|
| **类型** | Workflow（Interrupt） |
| **输入** | `tech_design` + `prd` |
| **输出** | `tech_review_decision` + `tech_review_feedback` |
| **机制** | `interrupt()` 暂停工作流 → 通知审批人（Slack/钉钉/Web） → 审批人决策 → `Command(resume=...)` 恢复 |
| **决策路由** | `approved` → coding / `rejected`或`revision_needed` → tech_design（带上反馈重新生成） |

#### 节点 5：编码实现 (coding) — Agent SubGraph

| 属性 | 值 |
|------|---|
| **类型** | Agent SubGraph（Open SWE） |
| **输入** | `tech_design` + `prd` + `repo_url` + `branch_name` |
| **输出** | `coding_result`（{pr_url, summary, files_changed, diff}） |
| **集成方式** | Open SWE 作为 LangGraph SubGraph 嵌入（共享状态模式） |
| **沙箱** | 每个编码任务在独立容器中执行（Modal / Daytona / Docker） |
| **内部流程** | Open SWE 内部执行 Planner → Programmer → Reviewer 三阶段（对外透明） |
| **超时** | 可配置最大执行时间（默认 30 分钟） |
| **备选** | 如 Open SWE 编码质量不达标，切换为 OpenHands（封装为 LangChain Tool） |

#### 节点 6：测试执行 (test_execute)

| 属性 | 值 |
|------|---|
| **类型** | Workflow（API 调用） |
| **输入** | `repo_url` + `branch_name` + `coding_result` |
| **输出** | `test_execution_result`（{passed, failed, coverage, report_url}） |
| **机制** | 调用企业 CI 系统 API（Jenkins / GitLab CI）触发测试流水线 → 轮询/回调等待结果 |

#### 节点 7：测试审查 (test_review) — Agent SubGraph

| 属性 | 值 |
|------|---|
| **类型** | Agent SubGraph（自建） |
| **输入** | `coding_result.diff` + `test_execution_result` + `tech_design` |
| **输出** | `review_report`（{verdict, issues[], suggestions[]}） |
| **内部流程** | 如下 |

```
┌─────────────────────────────────────────────────┐
│  Review Agent SubGraph (自建 LangGraph)           │
│                                                   │
│  ┌────────────┐    ┌───────────────┐              │
│  │ 解析 diff  │───→│ 检索代码库上下文│              │
│  │ 提取变更文件│    │ (pgvector RAG)│              │
│  └────────────┘    └───────┬───────┘              │
│                            │                      │
│                            ▼                      │
│  ┌──────────────────────────────────────────┐     │
│  │            LLM 多维度评估                  │     │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐  │     │
│  │  │代码质量  │ │架构合规  │ │安全检查  │  │     │
│  │  │检查      │ │检查      │ │          │  │     │
│  │  └──────────┘ └──────────┘ └──────────┘  │     │
│  └──────────────────┬───────────────────────┘     │
│                     │                             │
│                     ▼                             │
│  ┌────────────────────────────────────────┐       │
│  │  汇总审查报告                           │       │
│  │  verdict: pass / fail / warning         │       │
│  │  issues: [{severity, file, description}]│       │
│  │  suggestions: [...]                     │       │
│  └────────────────────────────────────────┘       │
└─────────────────────────────────────────────────┘
```

| 审查维度 | 检查内容 |
|---------|---------|
| **代码质量** | 可读性、命名规范、重复代码、复杂度 |
| **架构合规** | 是否符合技术方案的架构设计、模块划分、API 规范 |
| **安全检查** | SQL 注入、XSS、敏感信息泄露、权限校验遗漏 |
| **测试覆盖** | 是否有充分的测试用例、边界条件是否覆盖 |
| **变更影响** | 是否对已有功能产生回归风险 |

#### 节点 8：人工 Approval (final_approval)

| 属性 | 值 |
|------|---|
| **类型** | Workflow（Interrupt） |
| **输入** | `coding_result.pr_url` + `test_execution_result` + `review_report` |
| **输出** | `final_approval_decision` + `final_approval_feedback` |
| **机制** | 同节点 4。展示 PR 链接、测试报告、AI 审查报告供人工决策 |
| **决策路由** | `approved` → cicd / `rejected` → coding（带上反馈重新编码） |

#### 节点 9：CI/CD 集成 (cicd)

| 属性 | 值 |
|------|---|
| **类型** | Workflow（API 调用） |
| **输入** | `repo_url` + `branch_name` + `coding_result.pr_url` |
| **输出** | 部署结果 |
| **机制** | Merge PR → 触发部署流水线 → 等待部署完成 |

---

## 六、数据架构

### 6.1 数据库设计

```
┌────────────────────────────────────────────────────────┐
│  PostgreSQL                                            │
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ tasks         │  │ task_stages  │  │ approvals    │  │
│  │──────────────│  │──────────────│  │──────────────│  │
│  │ id (PK)      │  │ id (PK)      │  │ id (PK)      │  │
│  │ requirement  │  │ task_id (FK) │  │ task_id (FK) │  │
│  │ repo_url     │  │ stage_name   │  │ stage_name   │  │
│  │ branch_name  │  │ status       │  │ decision     │  │
│  │ status       │  │ input_data   │  │ feedback     │  │
│  │ created_by   │  │ output_data  │  │ reviewer     │  │
│  │ created_at   │  │ started_at   │  │ decided_at   │  │
│  │ updated_at   │  │ completed_at │  │              │  │
│  └──────────────┘  │ error        │  └──────────────┘  │
│                    └──────────────┘                     │
│  ┌──────────────┐  ┌──────────────┐                    │
│  │ audit_logs   │  │ langgraph_   │                    │
│  │──────────────│  │ checkpoints  │                    │
│  │ id (PK)      │  │──────────────│                    │
│  │ task_id (FK) │  │ (LangGraph   │                    │
│  │ action       │  │  内部表，     │                    │
│  │ actor        │  │  自动管理)    │                    │
│  │ details      │  └──────────────┘                    │
│  │ timestamp    │                                      │
│  └──────────────┘  ┌──────────────┐                    │
│                    │ embeddings   │                    │
│                    │ (pgvector)   │                    │
│                    │──────────────│                    │
│                    │ id (PK)      │                    │
│                    │ source_type  │                    │
│                    │ source_id    │                    │
│                    │ content      │                    │
│                    │ embedding    │                    │
│                    │ metadata     │                    │
│                    └──────────────┘                    │
└────────────────────────────────────────────────────────┘
```

### 6.2 状态持久化策略

| 数据 | 存储 | 用途 |
|------|------|------|
| **工作流状态** | LangGraph PostgresSaver（checkpoints 表） | 流程断点恢复、Interrupt 恢复、历史回溯 |
| **业务数据** | tasks / task_stages / approvals 表 | 管理后台查询、报表统计 |
| **审计日志** | audit_logs 表 | 合规追溯、操作记录 |
| **向量索引** | pgvector embeddings 表 | RAG 检索（企业知识库、代码库结构） |
| **临时缓存** | Redis | LLM 响应缓存、任务进度通知 |
| **制品存档** | 对象存储（MinIO/S3） | PRD 文档、技术方案、测试报告、审查报告的历史版本 |

---

## 七、集成架构

### 7.1 外部系统集成

```
┌─────────────────────────────────────────────────────────┐
│  平台核心                                                │
│  (LangGraph + FastAPI)                                  │
│                                                         │
│  ┌──────── 入站集成 ────────┐  ┌──── 出站集成 ─────────┐ │
│  │                          │  │                       │ │
│  │  Slack/钉钉/企微 Bot ←──→│  │──→ CI/CD (Jenkins/    │ │
│  │  (需求输入 + 审批通知)    │  │    GitLab CI API)    │ │
│  │                          │  │                       │ │
│  │  Git Webhook ←───────────│  │──→ Git Provider       │ │
│  │  (PR/Issue 事件)         │  │    (GitHub/GitLab API)│ │
│  │                          │  │                       │ │
│  │  Web API (REST) ←────────│  │──→ LLM Providers      │ │
│  │  (管理后台/第三方系统)    │  │    (Anthropic/OpenAI  │ │
│  │                          │  │     /DeepSeek/本地)    │ │
│  │                          │  │                       │ │
│  │                          │  │──→ 消息通知            │ │
│  │                          │  │    (Slack/钉钉/邮件)   │ │
│  └──────────────────────────┘  └───────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### 7.2 集成接口设计

| 集成对象 | 协议 | 关键接口 |
|---------|------|---------|
| **CI/CD** | REST API | 触发流水线、查询状态、获取报告 |
| **Git Provider** | REST API + Webhook | 创建分支、创建 PR、Merge PR、监听事件 |
| **LLM Provider** | LangChain Adapter | 统一调用接口，支持模型路由和回退 |
| **消息通知** | REST API / SDK | 发送审批通知、进度更新、完成通知 |
| **企业知识库** | RAG Pipeline | 文档索引、向量检索 |

---

## 八、安全与治理

### 8.1 安全架构

| 层面 | 措施 |
|------|------|
| **沙箱隔离** | Coding Agent 在独立 Docker 容器中运行，无宿主机访问权限，网络受限 |
| **LLM 数据安全** | 自托管部署，代码不出企业内网；敏感信息（密钥、凭证）在传入 LLM 前脱敏 |
| **API 安全** | JWT 认证 + RBAC 授权，所有接口 HTTPS |
| **审计追溯** | 全部操作写入 audit_logs，含操作人、时间、内容、结果 |
| **审批不可绕过** | Interrupt 审批门由工作流引擎强制执行，代码层面无法跳过 |

### 8.2 RBAC 设计

| 角色 | 权限 |
|------|------|
| **Admin** | 全部权限：系统配置、用户管理、任务管理 |
| **Tech Lead** | 技术方案审批、最终部署审批、任务查看 |
| **Developer** | 提交需求、查看任务进度、查看产物 |
| **Viewer** | 只读：查看任务和产物 |

---

## 九、可观测性

### 9.1 监控体系

```
┌─────────────────────────────────────────────────────┐
│  可观测性三支柱                                       │
│                                                     │
│  ┌───────────────┐  ┌────────────┐  ┌────────────┐  │
│  │  Tracing       │  │  Metrics   │  │  Logging   │  │
│  │                │  │            │  │            │  │
│  │  LangSmith:    │  │ OpenTele-  │  │ 结构化日志 │  │
│  │  - LLM 调用链  │  │ metry:     │  │ (JSON)     │  │
│  │  - Token 用量  │  │ - 延迟 P99 │  │            │  │
│  │  - 推理耗时    │  │ - 成功率   │  │ 写入:      │  │
│  │  - Prompt/回复 │  │ - 队列深度 │  │ - stdout   │  │
│  │                │  │ - 资源利用 │  │ - 日志收集 │  │
│  │  OpenTelemetry:│  │            │  │   系统     │  │
│  │  - API 调用链  │  │ 写入:      │  │            │  │
│  │  - 跨服务追踪  │  │ Prometheus │  │            │  │
│  └───────────────┘  └────────────┘  └────────────┘  │
└─────────────────────────────────────────────────────┘
```

### 9.2 关键指标

| 类别 | 指标 | 告警阈值 |
|------|------|---------|
| **流程** | 端到端完成时间 | > 2 小时 |
| **流程** | 各阶段通过率 | < 60% |
| **Agent** | Coding Agent 执行时间 | > 30 分钟 |
| **Agent** | Review Agent 执行时间 | > 5 分钟 |
| **LLM** | Token 单次消耗 | > 100k tokens |
| **LLM** | API 调用失败率 | > 5% |
| **基础设施** | API 延迟 P99 | > 3s |
| **基础设施** | 沙箱容器 OOM | 任何 |

---

## 十、部署架构

### 10.1 PoC 阶段（单机部署）

```
┌─────────────────────────────────────────┐
│  单台服务器 / 开发机                      │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │ docker-compose                  │    │
│  │                                 │    │
│  │  ┌──────────┐  ┌─────────────┐  │    │
│  │  │ platform │  │ PostgreSQL  │  │    │
│  │  │ (FastAPI │  │ + pgvector  │  │    │
│  │  │  + Lang- │  └─────────────┘  │    │
│  │  │  Graph)  │  ┌─────────────┐  │    │
│  │  └──────────┘  │ Redis       │  │    │
│  │                └─────────────┘  │    │
│  │  ┌──────────┐                   │    │
│  │  │ sandbox  │  ← Agent 沙箱容器  │    │
│  │  │ (按需创建)│                   │    │
│  │  └──────────┘                   │    │
│  └─────────────────────────────────┘    │
│                                         │
│  外部依赖: LLM API, CI/CD API, Git API  │
└─────────────────────────────────────────┘
```

### 10.2 生产阶段（K8s 部署）

```
┌─────────────────────────────────────────────────────┐
│  Kubernetes Cluster                                  │
│                                                     │
│  ┌── Namespace: ai-dev-platform ──────────────────┐ │
│  │                                                │ │
│  │  ┌──────────────┐  ┌──────────────────────┐    │ │
│  │  │ Deployment:   │  │ StatefulSet:          │    │ │
│  │  │ platform-api  │  │ postgresql            │    │ │
│  │  │ (replicas: 2) │  │ (replicas: 1, PVC)   │    │ │
│  │  └──────────────┘  └──────────────────────┘    │ │
│  │                                                │ │
│  │  ┌──────────────┐  ┌──────────────────────┐    │ │
│  │  │ Deployment:   │  │ Job (动态创建):       │    │ │
│  │  │ redis         │  │ coding-sandbox-{id}  │    │ │
│  │  │ (replicas: 1) │  │ (每个编码任务一个容器) │    │ │
│  │  └──────────────┘  └──────────────────────┘    │ │
│  │                                                │ │
│  │  ┌──────────────┐  ┌──────────────────────┐    │ │
│  │  │ Ingress       │  │ NetworkPolicy        │    │ │
│  │  │ (HTTPS)      │  │ (沙箱网络隔离)        │    │ │
│  │  └──────────────┘  └──────────────────────┘    │ │
│  └────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

---

## 十一、API 设计

### 11.1 核心 API

| 方法 | 路径 | 说明 |
|------|------|------|
| `POST` | `/api/v1/tasks` | 创建新任务（输入需求，启动流水线） |
| `GET` | `/api/v1/tasks/{id}` | 获取任务详情和当前状态 |
| `GET` | `/api/v1/tasks/{id}/stages` | 获取任务各阶段的产物 |
| `POST` | `/api/v1/tasks/{id}/approve` | 提交审批决策（approved/rejected） |
| `GET` | `/api/v1/tasks` | 任务列表（分页、过滤、排序） |
| `POST` | `/api/v1/tasks/{id}/retry` | 从失败节点重试 |
| `POST` | `/api/v1/tasks/{id}/cancel` | 取消运行中的任务 |

### 11.2 Webhook 回调

| 事件 | 触发时机 | Payload |
|------|---------|---------|
| `task.stage_completed` | 任意阶段完成 | {task_id, stage, output} |
| `task.approval_needed` | 到达审批节点 | {task_id, stage, artifacts} |
| `task.completed` | 全流程完成 | {task_id, pr_url, deploy_result} |
| `task.failed` | 任意阶段失败 | {task_id, stage, error} |

---

## 十二、项目结构

```
ai-dev-platform/
├── pyproject.toml                    # 依赖管理 (uv/poetry)
├── Dockerfile
├── docker-compose.yml                # PoC 部署
├── k8s/                              # K8s 部署清单
│   ├── deployment.yaml
│   ├── statefulset-postgres.yaml
│   └── networkpolicy.yaml
│
├── src/
│   ├── __init__.py
│   ├── main.py                       # FastAPI 入口
│   ├── config.py                     # 配置管理
│   │
│   ├── api/                          # API 层
│   │   ├── __init__.py
│   │   ├── routes/
│   │   │   ├── tasks.py              # 任务 CRUD
│   │   │   ├── approvals.py          # 审批接口
│   │   │   └── webhooks.py           # Webhook 入站
│   │   ├── schemas.py                # 请求/响应模型
│   │   └── dependencies.py           # 认证/授权
│   │
│   ├── workflow/                     # 工作流层 (LangGraph)
│   │   ├── __init__.py
│   │   ├── state.py                  # PlatformState 定义
│   │   ├── graph.py                  # 主工作流图构建
│   │   ├── nodes/                    # 各节点实现
│   │   │   ├── requirement.py
│   │   │   ├── prd.py
│   │   │   ├── tech_design.py
│   │   │   ├── review_gates.py       # 人工审批节点
│   │   │   ├── test_execute.py
│   │   │   └── cicd.py
│   │   └── routing.py                # 条件路由逻辑
│   │
│   ├── agents/                       # Agent 层
│   │   ├── __init__.py
│   │   ├── coding/                   # Coding Agent 集成
│   │   │   ├── open_swe.py           # Open SWE SubGraph 集成
│   │   │   ├── openhands.py          # OpenHands 备选集成
│   │   │   └── sandbox.py            # 沙箱管理
│   │   └── review/                   # Review Agent
│   │       ├── graph.py              # Review SubGraph 定义
│   │       ├── analyzers.py          # 各维度分析器
│   │       └── prompts.py            # 审查 Prompt 模板
│   │
│   ├── integrations/                 # 外部集成
│   │   ├── __init__.py
│   │   ├── ci_cd.py                  # Jenkins/GitLab CI 适配
│   │   ├── git_provider.py           # GitHub/GitLab API
│   │   ├── notification.py           # Slack/钉钉通知
│   │   └── llm_router.py            # LLM 模型路由
│   │
│   ├── rag/                          # RAG 检索
│   │   ├── __init__.py
│   │   ├── indexer.py                # 文档索引
│   │   └── retriever.py              # 向量检索
│   │
│   ├── db/                           # 数据层
│   │   ├── __init__.py
│   │   ├── models.py                 # SQLAlchemy 模型
│   │   └── repositories.py           # 数据访问
│   │
│   └── common/                       # 公共模块
│       ├── __init__.py
│       ├── auth.py                   # 认证/RBAC
│       ├── audit.py                  # 审计日志
│       └── observability.py          # Tracing 配置
│
├── prompts/                          # Prompt 模板（版本化管理）
│   ├── requirement_analysis.md
│   ├── prd_generation.md
│   ├── tech_design.md
│   └── code_review.md
│
└── tests/
    ├── unit/
    ├── integration/
    └── e2e/
```

---

## 十三、实施路线

| 阶段 | 目标 | 交付内容 | 技术重点 |
|------|------|---------|---------|
| **Phase 1: PoC** | 单条流水线跑通 | 需求→技术方案→编码→PR | LangGraph 工作流 + Open SWE 集成 + Interrupt 审批 + LangSmith |
| **Phase 2: 核心打通** | 全流程 + 企业集成 | 9 个节点全部上线 | CI/CD 对接 + Review Agent + 消息通知 + RAG 知识库 |
| **Phase 3: 生产加固** | 全团队推广 | 管理后台 + 监控 | RBAC + 审计 + K8s 部署 + 高可用 + 安全加固 |
| **Phase 4: 演进（按需）** | 规模化 | 并行编码 + 节点升级 | 多实例并发 + 节点升级为 Agent + 多仓协同 |

---

## 十四、参考文档

| 文档 | 位置 |
|------|------|
| 架构决策记录 | `docs/architecture-decision-multi-agent-or-not.md` |
| Coding Agent 技术选型 | `docs/enterprise-ai-dev-platform-tech-selection.md` |
| SubAgent vs SubGraph 概念辨析 | `docs/guide-subagent-vs-subgraph.md` |
| Anthropic — Building Effective Agents | https://www.anthropic.com/research/building-effective-agents |
| Google/MIT — Scaling Agent Systems | https://arxiv.org/abs/2512.08296 |
| LangGraph SubGraph 文档 | https://docs.langchain.com/oss/python/langgraph/use-subgraphs |
| Open SWE | https://github.com/langchain-ai/open-swe |
| OpenHands SDK | https://docs.openhands.dev/sdk |
