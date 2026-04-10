# 架构决策：我应该采用多 Agent 架构吗？

> 更新日期：2026-04-10
> 状态：架构和技术栈已确定

---

## 零、先回答你的问题

**不要从"多 Agent 架构"出发。你应该从"确定性工作流 + 关键节点嵌入 Agent"出发。**

这不是一个"单 Agent vs 多 Agent"的二选一问题，而是一个**更根本的架构判断**：你的系统中，哪些环节需要 Agent（自主决策），哪些环节只需要 Workflow（确定性流程）。过早地把一切都设计成 Agent，是当前业界踩得最多的坑。

---

## 一、先搞清三个概念

业界（包括 Anthropic、Google、LangChain）在 2026 年已经形成共识，需要严格区分三个层次：

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   LLM 调用            最简单。一次 Prompt → 一次回答          │
│   (单次推理)          适合：文本生成、分类、摘要等             │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   Workflow            确定性编排。人定义流程，LLM 在            │
│   (工作流)            各节点执行，流程本身是预定义的            │
│                       适合：步骤清晰的流水线                   │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   Agent               自主决策。LLM 自己决定下一步做什么，      │
│   (智能体)            用什么工具，循环多少次                    │
│                       适合：开放性任务，路径无法预定义          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

Anthropic 的原话：

> **"Workflows are systems where LLMs and tools are orchestrated through predefined code paths. Agents are systems where LLMs dynamically direct their own processes and tool usage."**

关键区别：**谁控制流程**。Workflow 里是你的代码控制流程，Agent 里是 LLM 控制流程。

---

## 二、逐一分析你的每个环节

现在让我们把你的平台流程拆开，逐个环节判断它到底需要"Workflow"还是"Agent"：

| 环节 | 输入 | 输出 | 路径可预定义？ | 需要自主决策？ | 判定 |
|------|------|------|---------------|---------------|------|
| **需求分析** | 原始需求文本 + 企业上下文 | 结构化需求规格 | ✅ 是 | ❌ 否（格式固定） | **LLM 调用**（模板化 Prompt） |
| **PRD 产出** | 需求规格 | PRD 文档 | ✅ 是 | ❌ 否（格式固定） | **LLM 调用**（模板化 Prompt） |
| **技术方案** | PRD + 代码库上下文 | 技术方案文档 | ✅ 是 | ⚠️ 部分（需检索代码库） | **LLM 调用 + RAG** |
| **人工 Review** | 技术方案 | 通过/驳回 | ✅ 是 | ❌ 否（人来决策） | **Workflow 节点**（纯等待） |
| **编码实现** | 技术方案 + 代码库 | 代码变更 + PR | ❌ 不可预定义 | ✅ 是（读文件、写代码、运行测试、修复错误、循环迭代） | **Agent** ← 唯一真正需要 Agent 的环节 |
| **自动化测试：执行** | 代码分支 | 测试报告 | ✅ 是 | ❌ 否（触发 CI） | **Workflow 节点**（API 调用） |
| **自动化测试：审查** | 代码变更 + 测试结果 + 代码库上下文 | 审查报告（质量/安全/架构合规） | ❌ 不可预定义 | ✅ 是（需理解代码意图、架构一致性、安全隐患） | **Agent**（Review Agent） |
| **人工 Approval** | PR + 测试报告 | 通过/驳回 | ✅ 是 | ❌ 否（人来决策） | **Workflow 节点**（纯等待） |
| **CI/CD 集成** | 代码分支 | 部署结果 | ✅ 是 | ❌ 否（调 API） | **Workflow 节点**（API 调用） |

### 结论

你的平台有 **9 个环节**，其中：
- **2 个**真正需要 Agent（编码实现 + 测试审查）
- **3 个**是 LLM 调用（需求分析、PRD、技术方案）
- **4 个**是确定性流程（Review 等待、测试执行、Approval 等待、CI/CD 触发）

**你需要的不是"全 Agent 对话架构"，而是一个确定性工作流引擎 + 在 2 个关键节点嵌入专业 Agent。**

> **关于"自动化测试"环节的修正说明**
>
> 你说得对。"自动化测试"实际上包含两个本质不同的子步骤：
>
> 1. **测试执行**（确定性）：触发 CI 跑测试 → 拿到通过/失败报告 → 这是纯 Workflow
> 2. **测试审查**（需要智能）：基于代码变更内容、测试结果、代码库整体架构，做出"这个变更是否安全、是否符合规范、是否有遗漏的测试场景、是否存在安全隐患"的综合判断 → 这需要 Agent
>
> 后者需要 Agent 的理由与"编码实现"类似——它需要**在代码库上下文中自主推理**：读取变更的文件、理解变更意图、对照架构规范检查一致性、识别边界条件和安全风险。这不是一次简单的 LLM 调用能搞定的，也不是 ESLint 或 SonarQube 这样的静态分析工具能覆盖的。

---

## 三、为什么不应该把每个环节都做成 Agent

### 3.1 Google + MIT 的研究结论

Google 和 MIT 2026 年的论文 *"Towards a Science of Scaling Agent Systems"* 通过 180 种 Agent 配置的对照实验得出：

> **在顺序性任务中，多 Agent 协作导致性能下降 39%～70%。**

你的平台流程（需求→PRD→方案→编码→测试→部署）本质上是**严格顺序的流水线**，每个环节依赖上一个环节的产出。这恰恰是多 Agent 架构表现最差的场景。

原因：
- **认知预算浪费**：Agent 之间的通信和协调会消耗大量 Token，挤占实际任务的推理资源
- **级联幻觉**：上游 Agent 的小错误被下游 Agent 当作事实，错误逐级放大（最高可放大 17.2 倍）
- **调试噩梦**：出了问题你不知道是哪个 Agent 的问题，也不知道是 Agent 间传递的什么信息导致了偏差

### 3.2 Anthropic 的忠告

Anthropic 在 *"Building Effective Agents"* 中明确建议：

> **"Start with the simplest solution possible. For many applications, optimizing a single LLM call with retrieval and in-context examples is sufficient."**
>
> **"Only increase complexity when simpler solutions fall short."**

你的"需求分析""PRD""技术方案"环节，本质上就是**一次结构化的 LLM 调用**（给一个精心设计的 Prompt + 上下文，输出一个格式化文档）。把它们包装成独立的"Agent"不会带来任何额外能力，反而增加了复杂性、延迟和成本。

### 3.3 实际的反面案例

如果你把需求分析做成 Agent：
```
需求分析 Agent 启动 → "我要先检索企业知识库" → 调用检索工具 → 
"结果不够，让我换个查询" → 再次调用 → "现在我要分析需求" → 
"我是否需要调用其他工具？" → 决定不需要 → 最终输出需求规格
```
这里 Agent 的"自主决策"步骤（选择工具、决定是否继续）完全是多余的。同样的效果，一个固定流程就能做到：
```
检索企业知识库(需求文本) → LLM(需求文本 + 检索结果, PRD模板) → 输出需求规格
```
后者更快、更便宜、更可预测、更容易调试。

---

## 四、你真正需要的架构

### 4.1 正确的架构：确定性流水线 + 双 Agent 节点

```
┌──────────────────────────────────────────────────────────────────────┐
│                     确定性工作流引擎 (Workflow)                        │
│                 （控制流程、状态管理、错误恢复、审批）                    │
│                                                                      │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐     │
│   │ 需求分析  │──→│ PRD 产出  │──→│ 技术方案  │──→│ 人工 Review   │     │
│   │ LLM Call │   │ LLM Call │   │ LLM+RAG  │   │  (等待审批)   │     │
│   └──────────┘   └──────────┘   └──────────┘   └──────┬───────┘     │
│                                                        │             │
│                              approved ─────────────────┘             │
│                                                                      │
│   ┌──────────────────┐   ┌──────────┐   ┌──────────────────┐        │
│   │  编码实现          │──→│ 测试执行  │──→│  测试审查          │        │
│   │ ┌──────────────┐  │   │ (调 CI)  │   │ ┌──────────────┐  │        │
│   │ │ Coding Agent │  │   │ Workflow │   │ │ Review Agent │  │        │
│   │ │ (Agent #1)   │  │   └──────────┘   │ │ (Agent #2)   │  │        │
│   │ └──────────────┘  │                   │ └──────────────┘  │        │
│   └──────────────────┘                   └────────┬─────────┘        │
│                                                    │                 │
│                                          ┌─────────▼──────────┐      │
│                                          │ 人工 Approval       │      │
│                                          │ (等待审批)          │      │
│                                          └─────────┬──────────┘      │
│                                                    │                 │
│                                   approved ────────┘                 │
│                                                                      │
│                                          ┌──────────┐                │
│                                          │ CI/CD    │──→ 部署完成     │
│                                          │ (调 API) │                │
│                                          └──────────┘                │
└──────────────────────────────────────────────────────────────────────┘
```

**关于 Agent 节点的关系**：

注意图中的 Agent 节点**不是"多 Agent 协作"架构**。它们之间不会互相对话、不共享推理上下文、不存在协调开销。它们是工作流中**独立的节点**，各自接收确定性的输入、产出确定性的产物，由工作流引擎串联。这与 MetaGPT 式的"多 Agent 互相讨论"有本质区别。

### 4.1.1 架构弹性：不绑定"双 Agent"，支持未来灵活增减

上面的图只是**当前阶段的实例化**，不是架构的上限。正确的理解是：

> **底层架构是一条确定性工作流，其中的任意节点都可以是"LLM 单次调用"、"确定性逻辑"或"Agent SubGraph"，三者之间可以按需切换和新增。**

随着平台演进，你可能会发现某些节点需要升级为 Agent，或者某些通用能力需要抽取为独立的可复用 Agent。LangGraph 的 SubGraph 机制天然支持这种弹性：

```
当前（Phase 1）                    未来可能的演进
────────────────────               ────────────────────────────

需求分析: LLM Call                 需求分析: Agent SubGraph
                                      （如果需要自主检索多个知识库、
                                       多轮推理才能拆解复杂需求）

PRD 产出: LLM Call                 PRD 产出: LLM Call（保持不变）

技术方案: LLM + RAG               技术方案: Agent SubGraph
                                      （如果需要自主探索代码库、
                                       参考多个架构模板迭代生成）

编码实现: Agent (Open SWE)         编码实现: Agent（保持不变）

测试执行: Workflow                  测试执行: Workflow（保持不变）

测试审查: Agent (Review Agent)     测试审查: Agent（保持不变）

                                   安全扫描: Agent SubGraph（新增）
                                      （如果需要智能化安全审计）

CI/CD: Workflow                    CI/CD: Workflow（保持不变）
```

**LangGraph 如何支持这种弹性——SubGraph 架构**：

```python
# 任何节点都可以从 LLM Call 升级为 Agent SubGraph，
# 而不改变工作流的其余部分

# ── 阶段一：需求分析是简单的 LLM 调用 ──
async def requirement_analysis_v1(state: PlatformState) -> dict:
    result = await llm.ainvoke(prompt.format(req=state["requirement"]))
    return {"analysis": result.content}

# ── 阶段二：需求分析升级为 Agent SubGraph ──
# 只需替换节点实现，工作流拓扑不变
def build_requirement_agent() -> CompiledGraph:
    """当简单 LLM 调用不够用时，升级为 Agent SubGraph"""
    agent_graph = StateGraph(RequirementAgentState)
    agent_graph.add_node("retrieve_context", retrieve_from_knowledge_base)
    agent_graph.add_node("analyze", llm_analysis)
    agent_graph.add_node("validate", cross_check_with_existing_specs)
    agent_graph.add_conditional_edges("validate", need_more_info, {
        "yes": "retrieve_context",  # 循环：需要更多信息
        "no": END,                  # 完成
    })
    return agent_graph.compile()

# 主工作流中替换节点——一行改动
workflow.add_node("requirement_analysis", build_requirement_agent())
```

**关键设计原则：状态共享的便利性不因 Agent 数量变化而损失**

LangGraph SubGraph 提供两种状态共享模式，可按需选择：

| 模式 | 机制 | 适用场景 | 状态共享 |
|------|------|---------|---------|
| **共享状态键** | SubGraph 与父图使用相同的 State 类型字段 | Agent 需要读写全局上下文（如编码结果、PRD 内容） | 完全共享，自动读写 |
| **隔离 + 适配** | SubGraph 有独立 State，通过包装函数做 I/O 映射 | Agent 有私有推理上下文，只暴露最终产物 | 按需映射，互不污染 |

```python
# 模式一：共享状态——SubGraph 直接读写父图状态
workflow.add_node("coding", open_swe_subgraph)  # 自动共享 PlatformState

# 模式二：隔离 + 适配——SubGraph 有私有状态，只映射输入输出
async def review_node(state: PlatformState) -> dict:
    """Review Agent 有自己的私有推理上下文，只返回最终报告"""
    review_input = {
        "diff": state["coding_result"]["diff"],
        "tech_design": state["tech_design"],
    }
    review_output = await review_subgraph.ainvoke(review_input)
    return {"review_report": review_output["report"]}

workflow.add_node("review", review_node)
```

**无论使用哪种模式**，LangGraph 的 `checkpointer` 保证全链路状态持久化，任何节点（不论是 LLM Call 还是 Agent SubGraph）都可以被 `interrupt()` 暂停、被人工恢复、从断点重试。增加或减少 Agent 节点，不会影响这些基础能力。

### 4.2 为什么这个架构更好

| 维度 | "全 Agent 对话"架构 | "工作流 + 双节点 Agent"架构 |
|------|-------------------|--------------------------|
| **可预测性** | 低 — 每个 Agent 都可能走意外路径 | 高 — 流程确定，只有编码和审查有不确定性 |
| **调试难度** | 极高 — 需要追踪多个 Agent 间的交互 | 低 — Agent 之间不通信，独立调试 |
| **成本** | 高 — 每个 Agent 都在消耗推理 Token 做"决策" | 适中 — 只有 2 个 Agent 消耗推理 Token |
| **延迟** | 高 — Agent 循环决策 + 多 Agent 通信 | 适中 — 确定性节点直接执行，Agent 节点按需运行 |
| **可靠性** | 低 — 级联幻觉风险 | 高 — Agent 间传递的是确定性产物（代码、测试报告） |
| **人工审批** | 复杂 — 需要 Agent 感知审批状态 | 简单 — 工作流原生支持暂停/恢复 |
| **CI/CD 集成** | 复杂 — Agent 需要学会调 API | 简单 — 工作流节点直接调 API |
| **可替换性** | 低 — Agent 紧耦合 | 高 — Coding Agent 和 Review Agent 可独立选型/替换 |

---

## 五、那么什么时候才应该用多 Agent？

多 Agent 架构的正当场景是**任务可并行分解**且**各子任务之间高度独立**。

在你的平台中，**当前不需要多 Agent**，但**未来可能需要**的场景包括：

| 场景 | 说明 | 何时引入 |
|------|------|---------|
| **同时处理多个独立需求** | 多个需求并行进入流水线，各自独立走完全流程 | 这不是多 Agent，这是工作流引擎的**多实例并发**，不需要 Agent 间协作 |
| **编码任务内部的并行** | 一个大需求拆解为多个模块，多个 Coding Agent 并行开发不同模块 | 当单个 Coding Agent 处理一个需求的时间过长时（单 Agent 稳定运行后再考虑） |
| **跨代码库的大型重构** | 一个变更涉及多个微服务仓库 | 当平台成熟、单仓编码验证通过后 |

**关键原则**：先让单个流水线跑通跑稳，再考虑并行化和多 Agent。

---

## 六、技术栈决策

既然确定了"确定性工作流 + 双 Agent 节点"架构，现在做技术选型。

### 6.1 你需要选择三样东西

| 组件 | 作用 | 关键要求 |
|------|------|---------|
| **工作流引擎** | 编排整个流程、状态管理、人工审批、错误恢复、持久化 | 必须支持长时运行、暂停/恢复、故障重试 |
| **Coding Agent** | "编码实现"节点，自主完成代码编写、测试、PR | 可编程集成、沙箱隔离、模型无关 |
| **Review Agent** | "测试审查"节点，对代码变更做智能审查 | 代码库上下文理解、架构合规检查 |

### 6.2 推荐技术栈

#### 工作流引擎：LangGraph

在你的场景下，**LangGraph 是最佳选择**，原因如下：

| 候选 | 结论 | 理由 |
|------|------|------|
| **LangGraph** | **✅ 推荐** | 你的流水线中有 5 个节点需要调用 LLM（需求分析、PRD、技术方案、编码 Agent、审查 Agent），LangGraph 对 LLM 调用是一等公民；Interrupt 机制天然适配人工审批；StateGraph 统一管理全流程状态；LangSmith 提供全链路可观测性 |
| Temporal | ⚠️ 强备选 | 持久化执行和故障恢复是业界最强（Netflix/Stripe/OpenAI 在用），但对 LLM 调用没有原生优化，需要额外封装；运维成本更高 |
| n8n | ❌ 不推荐做生产 | 适合 PoC 快速验证，但复杂的 Agent 集成、状态管理和企业级需求用低代码工具会很痛苦 |
| Airflow/Prefect | ❌ 不适合 | 偏数据管线调度，不适合实时交互式工作流 |

**LangGraph 的关键能力与你的需求对齐**：

```
你的需求                          LangGraph 对应能力
────────────────────────────────────────────────────────────
人工审批关键节点                   → interrupt() 原生暂停/恢复
流程中多处调用 LLM                → LLM 调用是一等公民
编码/审查节点需要 Agent 自主循环   → 支持 SubGraph / 循环图
全流程状态跟踪                    → StateGraph 统一状态
出错后从断点恢复                  → Checkpointer 持久化
生产环境可观测性                  → LangSmith 全链路 Tracing
自托管/内网部署                   → 开源 MIT，可完全自托管
```

**LangGraph 部署方案选择**：

| 方案 | 适合阶段 | 成本 | 特点 |
|------|---------|------|------|
| 自托管（Docker + PostgreSQL Checkpointer） | 开发/PoC/内网部署 | 免费（MIT 开源） | 完全掌控，但需自行运维 |
| LangGraph Platform Plus | 团队协作阶段 | $39/座/月 + 用量费 | 托管运维，减少 DevOps 负担 |
| LangGraph Platform Enterprise | 生产阶段 | 定制报价 | Hybrid/VPC 部署、SSO、RBAC、SLA |

> **建议**：PoC 阶段用自托管（零成本），验证通过后按需升级到 Platform。

#### 语言与基础设施

| 层面 | 选型 | 理由 |
|------|------|------|
| **编程语言** | **Python 3.12+** | LangGraph/LangChain 的原生语言、AI/ML 生态最强、Coding Agent SDK 均为 Python |
| **Web 框架** | **FastAPI** | 异步原生、自动 OpenAPI 文档、性能优秀、与 LangGraph 配合良好 |
| **数据验证** | **Pydantic v2** | LangChain 生态标准，类型安全 |
| **状态持久化** | **PostgreSQL** | LangGraph Checkpointer 原生支持，也是企业标准 |
| **向量数据库** | **pgvector（PostgreSQL 扩展）** | 复用 PostgreSQL 基础设施，技术方案节点的 RAG 检索用 |
| **消息队列** | **Redis** | 异步任务下发、Agent 事件通知 |
| **容器化** | **Docker + K8s** | Agent 沙箱隔离、生产部署标准 |
| **可观测性** | **LangSmith + OpenTelemetry** | LLM 链路追踪 + 基础设施监控 |
| **前端**（管理后台） | **按团队能力选择** | Next.js / Vue 3 / 内部框架均可 |

#### Coding Agent：按场景选择

| 场景 | 推荐 | 集成方式 |
|------|------|---------|
| **与 LangGraph 最紧密集成** | **Open SWE** | LangGraph SubGraph 原生嵌入 |
| **最强编码能力 + 最成熟企业治理** | **OpenHands** | 封装为 LangChain Tool，通过 SDK/MCP 调用 |
| **最大社区支持 + 模型灵活性** | **OpenCode** | 通过 REST API / Python SDK 调用 |

> **建议**：选 Open SWE 起步（与 LangGraph 零摩擦），如编码质量不达标再切 OpenHands。

#### 关于 Deep Agents：不是你的技术栈选择项，而是 Open SWE 的内部依赖

你可能注意到之前的文档提到了 Deep Agents。这里要澄清一个关键问题：**你不需要"选择"是否使用 Deep Agents——它只是 Open SWE 的内部实现细节。**

先搞清楚三者的层次关系：

```
┌──────────────────────────────────────────────────────┐
│  Open SWE                                            │
│  应用层 — 面向"编码"场景的具体实现                      │
│  （沙箱集成、GitHub/Slack 触发、PR 创建等）              │
│                                                      │
│  ┌──────────────────────────────────────────────────┐ │
│  │  Deep Agents                                     │ │
│  │  工具层 — 预装的 Agent "脚手架"                    │ │
│  │  （规划工具、文件系统记忆、子Agent派生、中间件）     │ │
│  │                                                  │ │
│  │  ┌──────────────────────────────────────────────┐│ │
│  │  │  LangGraph                                   ││ │
│  │  │  基础层 — 状态机运行时                         ││ │
│  │  │  （状态管理、持久化、图编排、Interrupt）        ││ │
│  │  └──────────────────────────────────────────────┘│ │
│  └──────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

**对你的意义**：

| 你的角色 | 你直接使用的 | 你不需要直接使用的 |
|---------|------------|------------------|
| **MainAgent 工作流** | LangGraph（StateGraph、Interrupt、Checkpointer） | Deep Agents |
| **Coding Agent 节点** | Open SWE（作为 SubGraph 嵌入你的工作流） | Deep Agents（它是 Open SWE 内部用的） |
| **Review Agent 节点** | LangGraph（自建 SubGraph） | Deep Agents |
| **其他节点** | LangGraph + LLM 调用 | Deep Agents |

**换句话说**：
- **你的 MainAgent 工作流**用 **LangGraph** 编排——这是你直接操作的层
- **Open SWE**用 **Deep Agents** 实现其内部的"规划→编码→审查"循环——这是 Open SWE 的实现细节，对你透明
- 你的其他节点（需求分析、PRD、技术方案、Review Agent）**直接用 LangGraph 构建**，不需要 Deep Agents

**为什么不建议你的 MainAgent 也用 Deep Agents**：

1. **你的 MainAgent 不是"Agent"**——它是一个确定性工作流。Deep Agents 的核心能力（自主规划、子 Agent 派生、文件系统记忆）是为开放性、长时间运行的自主 Agent 设计的，而你的工作流流程是预定义的。

2. **不必要的抽象层**——Deep Agents 在 LangGraph 之上又加了一层抽象。对于确定性工作流，这层抽象只会增加调试难度，不会带来额外价值。Anthropic 的忠告适用于此：*"Start by using LLM APIs directly to avoid unnecessary abstraction layers that can complicate debugging."*

3. **你的 SubGraph Agent 节点大多不需要**——Review Agent 是一个相对简单的"读 diff → 检索上下文 → LLM 评估 → 输出报告"流程，用 LangGraph 原生构建就够了。只有 Coding Agent（Open SWE）因为其复杂性才需要 Deep Agents 的能力，而这已经被 Open SWE 内部封装好了。

> **结论**：Deep Agents 不是你需要做的技术决策。它是 Open SWE 的内部依赖，对你透明。你的技术栈是 **LangGraph（工作流编排）+ Open SWE（编码节点）**，仅此而已。

#### Review Agent：推荐方案

Review Agent 不需要像 Coding Agent 那样复杂（不需要写代码、不需要沙箱），它的核心能力是**理解代码变更并做出判断**。推荐方案：

| 方案 | 描述 | 适合阶段 |
|------|------|---------|
| **自建 LLM + 工具** | 用 LangGraph SubGraph 构建：读取 PR diff → 检索代码库上下文 → LLM 评估 → 产出审查报告 | 推荐（控制力最强） |
| **OpenCode `run` 模式** | `opencode run "Review this PR: {diff}"` — 利用 OpenCode 的代码理解能力 | 快速验证 |

### 6.3 最终推荐技术栈全景

```
┌────────────────────────────────────────────────────────────────────┐
│                      推荐技术栈全景                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  编排层    Python 3.12 + LangGraph + FastAPI                       │
│                                                                    │
│  Agent 层  Coding Agent: Open SWE (首选) / OpenHands (备选)         │
│            Review Agent: 自建 LangGraph SubGraph + LLM             │
│                                                                    │
│  数据层    PostgreSQL (状态持久化 + pgvector RAG)                    │
│            Redis (消息队列 + 缓存)                                  │
│                                                                    │
│  LLM 层   Claude Sonnet/Opus (主力) + DeepSeek (降本)              │
│            本地模型备选 (合规需求)                                    │
│                                                                    │
│  可观测性  LangSmith (LLM Tracing) + OpenTelemetry (基础设施)       │
│                                                                    │
│  基础设施  Docker + Kubernetes                                      │
│            Jenkins/GitLab CI (对接现有 CI/CD)                       │
│                                                                    │
│  前端      管理后台: Next.js / Vue 3                                │
│            审批交互: Slack/钉钉/企业微信 Bot + Web 审批页              │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 七、总结

### 架构决策

**不是"多 Agent 对话架构"，而是"确定性工作流 + 双 Agent 节点"**。

你的 9 个流程环节中，只有"编码实现"和"测试审查"需要 Agent 能力。其余环节用确定性工作流编排 + LLM 单次调用即可。两个 Agent 之间不通信、不协调，由工作流引擎串联，避免了多 Agent 协作的全部陷阱。

### 技术栈决策

| 组件 | 选定 | 核心理由 |
|------|------|---------|
| **语言** | Python 3.12+ | AI/ML 生态最强，LangGraph 原生语言 |
| **工作流引擎** | LangGraph | LLM 调用一等公民、Interrupt 天然适配人工审批、LangSmith 全链路可观测 |
| **Web 框架** | FastAPI | 异步原生、高性能、自动 API 文档 |
| **Coding Agent** | Open SWE（首选）→ OpenHands（备选） | LangGraph 原生集成 / 编码能力最强 |
| **Review Agent** | 自建 LangGraph SubGraph + LLM | 控制力最强，按需定制审查规则 |
| **数据库** | PostgreSQL + pgvector | 状态持久化 + RAG 向量检索复用 |
| **可观测性** | LangSmith + OpenTelemetry | LLM 链路 + 基础设施双覆盖 |

### 核心原则

> **"Only increase complexity when simpler solutions fall short."** — Anthropic
>
> 每增加一个 Agent，就增加了调试复杂度、Token 成本和出错概率。只有当"单次 LLM 调用搞不定这个环节，需要 Agent 循环迭代"时，才把那个环节升级为 Agent。

---

## 附：实施路线

```
Phase 1 — PoC 验证
┌──────────────────────────────────────┐
│  LangGraph + FastAPI + PostgreSQL    │
│  + Open SWE (Coding Agent)          │
│  + 自建 Review Agent (LLM+RAG)      │
│  + Interrupt 人工审批                │
│  + LangSmith Tracing                │
│                                      │
│  目标：单条流水线完整跑通             │
│  验证：编码质量、审查准确性、审批体验  │
└──────────────────┬───────────────────┘
                   │
                   ▼
Phase 2 — 核心打通
┌──────────────────────────────────────┐
│  + 对接现有 CI/CD (Jenkins/GitLab)   │
│  + 对接消息通知 (Slack/钉钉)         │
│  + 企业知识库 RAG (pgvector)         │
│  + Prompt 调优（需求/PRD/方案节点）   │
│  + 多实例并发（多需求并行流水线）     │
│                                      │
│  目标：内部小范围试运行              │
└──────────────────┬───────────────────┘
                   │
                   ▼
Phase 3 — 生产加固
┌──────────────────────────────────────┐
│  + RBAC / 审计日志                   │
│  + 管理后台 (任务面板 + 审批界面)     │
│  + 监控仪表盘 (OpenTelemetry)        │
│  + 沙箱安全加固                      │
│  + 容灾高可用                        │
│  + 评估是否切换/补充 OpenHands        │
│  + (按需) 多 Coding Agent 并行编码   │
│                                      │
│  目标：全团队推广                     │
└──────────────────────────────────────┘
```

---

## 参考资料

| 资源 | 要点 |
|------|------|
| [Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) | Workflow vs Agent 的权威定义，"从简单开始"原则 |
| [Google/MIT — Scaling Agent Systems](https://arxiv.org/abs/2512.08296) | 多 Agent 在顺序任务中性能下降 39%～70% 的实验证据 |
| [Anthropic — 2026 Agentic Coding Trends Report](https://resources.anthropic.com/2026-agentic-coding-trends-report) | 编码 Agent 的行业趋势和最佳实践 |
| [LangChain — Choosing the Right Multi-Agent Architecture](https://blog.langchain.com/choosing-the-right-multi-agent-architecture/) | 多 Agent 架构模式的选择指南 |
| [LangChain — How and When to Build Multi-Agent Systems](https://blog.langchain.com/how-and-when-to-build-multi-agent-systems/) | 何时该用多 Agent，何时不该 |
| [Towards AI — The Orchestration Problem Nobody Talks About](https://pub.towardsai.net/multi-agent-systems-arent-magic-here-s-the-orchestration-problem-nobody-talks-about-547d2e3437ca) | 多 Agent 生产环境的真实陷阱 |
