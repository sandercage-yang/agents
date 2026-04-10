# LangChain SubAgent 与 LangGraph SubGraph 概念辨析

> 更新日期：2026-04-10
> 目的：区分两个容易混淆的概念，明确在你的架构中各自对应什么

---

## 一句话总结

- **LangGraph SubGraph** = 图中嵌套图。是一种**工程结构**，用来把复杂图拆成可复用模块。
- **LangChain SubAgent** = Agent 中派生 Agent。是一种**运行时行为**，由 LLM 自主决定何时派生子 Agent 去完成子任务。

两者的核心区别在于**谁决定调用**：SubGraph 是你在代码中预定义好的，SubAgent 是 LLM 在运行时自己决定派生的。

---

## 二、LangGraph SubGraph —— 图中嵌套图

### 是什么

SubGraph 就是把一个编译好的 `StateGraph` 当作另一个 `StateGraph` 的一个节点（Node）来使用。

本质上是**代码级别的模块化封装**——就像函数调用一样。你在设计时就确定了"这个节点由一个子图来实现"，运行时 SubGraph 按照你预定义的逻辑执行。

### 一个直观的类比

```
普通函数:                       SubGraph:
┌────────────────┐              ┌────────────────┐
│  main()        │              │  主工作流        │
│    │           │              │    │            │
│    ├─ step_1() │              │    ├─ 需求分析   │  ← 普通节点
│    ├─ step_2() │              │    ├─ 编码实现   │  ← SubGraph（内部是一个完整的图）
│    └─ step_3() │              │    └─ 测试       │  ← 普通节点
└────────────────┘              └────────────────┘

step_2() 内部可能很复杂,         "编码实现" SubGraph 内部:
有自己的调用链，但对 main()      有 Planner → Programmer → Reviewer,
来说它就是一个函数调用。         但对主工作流来说它就是一个节点。
```

### 两种使用模式

#### 模式 1：共享状态（SubGraph 直接读写父图状态）

当子图和父图使用**相同的 State 字段**时，直接把编译好的子图当节点加进去：

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

# ── 全局状态 ──
class PlatformState(TypedDict):
    requirement: str
    tech_design: str
    code_result: str

# ── 定义一个 SubGraph：编码实现 ──
def plan_node(state: PlatformState) -> dict:
    return {"code_result": f"计划：基于 {state['tech_design']} 制定编码计划"}

def code_node(state: PlatformState) -> dict:
    return {"code_result": f"代码：已完成实现"}

coding_graph = StateGraph(PlatformState)
coding_graph.add_node("plan", plan_node)
coding_graph.add_node("code", code_node)
coding_graph.add_edge(START, "plan")
coding_graph.add_edge("plan", "code")
coding_graph.add_edge("code", END)
coding_subgraph = coding_graph.compile()   # 编译成可复用组件

# ── 主工作流：直接把 SubGraph 当节点使用 ──
def requirement_node(state: PlatformState) -> dict:
    return {"tech_design": f"基于 {state['requirement']} 生成技术方案"}

main_graph = StateGraph(PlatformState)
main_graph.add_node("requirement", requirement_node)
main_graph.add_node("coding", coding_subgraph)       # ← 直接传入编译好的子图
main_graph.add_edge(START, "requirement")
main_graph.add_edge("requirement", "coding")          # coding 内部会执行 plan → code
main_graph.add_edge("coding", END)

app = main_graph.compile()
result = app.invoke({"requirement": "开发用户认证模块"})
# coding_subgraph 自动读取 tech_design，自动写入 code_result
```

**特点**：SubGraph 和父图共享同一个 State 对象，子图可以直接读取 `tech_design` 并写入 `code_result`。

#### 模式 2：隔离状态（SubGraph 有自己的私有 State）

当子图有自己的内部状态（不想暴露给父图）时，用包装函数做转换：

```python
# ── SubGraph 有自己的私有状态 ──
class ReviewState(TypedDict):
    diff: str                   # 输入：代码变更
    review_notes: list[str]     # 私有：审查过程中的笔记（不暴露给父图）
    report: str                 # 输出：最终审查报告

def analyze_diff(state: ReviewState) -> dict:
    notes = [f"分析 diff: {state['diff'][:50]}..."]
    return {"review_notes": notes}

def generate_report(state: ReviewState) -> dict:
    report = f"审查通过。审查要点：{', '.join(state['review_notes'])}"
    return {"report": report}

review_graph = StateGraph(ReviewState)
review_graph.add_node("analyze", analyze_diff)
review_graph.add_node("report", generate_report)
review_graph.add_edge(START, "analyze")
review_graph.add_edge("analyze", "report")
review_graph.add_edge("report", END)
review_subgraph = review_graph.compile()

# ── 在父图中用包装函数调用，做状态转换 ──
async def review_node(state: PlatformState) -> dict:
    # 从父图状态 → 提取子图需要的输入
    review_input = {"diff": state["code_result"], "review_notes": [], "report": ""}
    
    # 调用子图
    review_output = await review_subgraph.ainvoke(review_input)
    
    # 从子图输出 → 映射回父图状态
    # 注意：review_notes 是私有的，不会传回父图
    return {"review_report": review_output["report"]}

main_graph.add_node("review", review_node)  # 通过包装函数加入
```

**特点**：`review_notes`（审查过程中的中间笔记）是子图的私有状态，父图永远看不到它。父图只能看到最终的 `report`。

### SubGraph 的关键特性

| 特性 | 说明 |
|------|------|
| **确定性** | 你在代码中预定义了子图的结构，运行时按设计执行 |
| **状态持久化** | 自动继承父图的 Checkpointer，支持 `interrupt()`、断点恢复 |
| **可调试** | LangSmith 中可以展开看到子图内部的每一步执行 |
| **可复用** | 同一个编译好的 SubGraph 可以被多个父图复用 |
| **可独立测试** | SubGraph 可以单独 `.invoke()` 测试，不需要父图 |

---

## 三、LangChain SubAgent —— Agent 中派生 Agent

### 是什么

SubAgent 是 **Deep Agents 框架**中的概念。它指的是一个正在运行的 Agent（父 Agent）通过调用 `task` 工具，**在运行时动态派生**一个子 Agent 去执行特定任务。

关键区别：**不是你在代码中预定义的，而是 LLM 在运行过程中自己决定的**。

### 一个直观的类比

```
函数调用 (SubGraph):                           委派任务 (SubAgent):
┌─────────────────────────┐                   ┌─────────────────────────┐
│  经理按照流程办事          │                   │  经理遇到问题，自己决定    │
│                          │                   │  派一个人去调查           │
│  1. 先做 A  ← 预定义     │                   │                          │
│  2. 再做 B  ← 预定义     │                   │  "这个安全问题比较复杂，   │
│  3. 最后做 C ← 预定义    │                   │   让安全专家去查一下"      │
│                          │                   │   ↓                      │
│  每一步都是提前确定的      │                   │   task("分析这段代码的     │
└─────────────────────────┘                   │        安全漏洞")         │
                                              │   ↓                      │
                                              │  子Agent独立工作，         │
                                              │  只返回最终结论            │
                                              └─────────────────────────┘
```

### SubAgent 的工作机制

```python
# 这不是你写的调用代码！
# 这是 Agent 在运行时，LLM 自己决定生成的 tool call：

# Agent 的思考过程（不可见）：
# "这个任务很复杂，我需要把安全分析部分委托给一个子Agent..."

# Agent 产生的 tool call：
task(
    description="分析 auth_service.py 中的安全漏洞，检查 SQL 注入和 XSS 风险",
    # 子Agent 会获得自己独立的上下文窗口
    # 子Agent 完成后，只把最终结论返回给父Agent
)
```

### SubAgent 的关键特性

| 特性 | 说明 |
|------|------|
| **动态的** | LLM 在运行时自主决定是否、何时、派生什么子 Agent |
| **上下文隔离** | 子 Agent 有独立的上下文窗口，防止父 Agent 的上下文被撑爆 |
| **不可预测** | 你无法提前确定会派生几个子 Agent、每个做什么 |
| **需要 Deep Agents** | 这是 Deep Agents 框架提供的能力，原生 LangGraph 没有 `task` 工具 |

---

## 四、对比总结

| 维度 | LangGraph SubGraph | LangChain SubAgent (Deep Agents) |
|------|-------------------|--------------------------------|
| **本质** | 代码结构（图中嵌套图） | 运行时行为（Agent 派生 Agent） |
| **谁决定调用** | 开发者在代码中预定义 | LLM 在运行时自主决定 |
| **确定性** | 高 — 结构和流程预定义 | 低 — LLM 动态决策 |
| **状态关系** | 可共享父图状态 / 可隔离 | 始终隔离（独立上下文窗口） |
| **可调试性** | 高 — LangSmith 展开查看 | 中 — 需要追踪动态派生链 |
| **可预测成本** | 高 — 执行路径确定 | 低 — LLM 可能派生任意数量的子 Agent |
| **依赖** | LangGraph（原生能力） | Deep Agents 框架 |
| **适合** | 确定性工作流中的模块化封装 | 开放性 Agent 中的自主任务分解 |
| **类比** | 函数调用 | 经理临时委派任务 |

---

## 五、在你的架构中，用到的是哪个？

### 你的 MainAgent 工作流 → 用 SubGraph

你的主工作流是确定性的：需求→PRD→方案→Review→编码→测试→审查→Approve→CI/CD。每个节点是什么、顺序如何，都是你在代码中预定义好的。这里用的是 **SubGraph**。

```python
# 你的架构中 SubGraph 的使用方式：

main_workflow = StateGraph(PlatformState)

main_workflow.add_node("requirement", requirement_llm_call)    # 普通节点
main_workflow.add_node("prd", prd_llm_call)                    # 普通节点
main_workflow.add_node("tech_design", tech_design_rag_call)    # 普通节点
main_workflow.add_node("human_review", human_review_gate)      # 审批节点
main_workflow.add_node("coding", open_swe_subgraph)            # ← SubGraph
main_workflow.add_node("test_execute", trigger_ci_pipeline)    # 普通节点
main_workflow.add_node("test_review", review_agent_subgraph)   # ← SubGraph
main_workflow.add_node("human_approve", human_approval_gate)   # 审批节点
main_workflow.add_node("cicd", deploy_pipeline)                # 普通节点
```

### Open SWE 的内部 → 用了 SubAgent

Open SWE 内部（你不需要直接操作的部分），Coding Agent 在执行复杂编码任务时，可能会通过 Deep Agents 的 `task` 工具动态派生子 Agent。比如：

```
Open SWE Coding Agent 运行中：
  "这个任务涉及前端和后端两个部分..."
  → task("实现后端 API 接口")       ← LLM 自主决定派生
  → task("实现前端组件")            ← LLM 自主决定派生
  每个子 Agent 独立完成，结果汇总
```

这是 Open SWE 内部的行为，对你的 MainAgent 来说是透明的。你只需要知道：把技术方案扔进 `coding` 节点，过一段时间拿到编码结果。

### 视觉化总结

```
┌─────────────────────────────────────────────────────────────┐
│  你的 MainAgent 工作流  (LangGraph StateGraph)               │
│                                                             │
│  需求分析 ──→ PRD ──→ 技术方案 ──→ 人工Review                │
│   (Node)     (Node)   (Node)      (Node)                   │
│                                     │                       │
│                              ┌──────▼──────┐                │
│                              │ 编码实现     │                │
│                              │ (SubGraph)  │ ← LangGraph SubGraph     │
│                              │ ┌─────────┐ │                │
│                              │ │Open SWE │ │                │
│                              │ │ 内部用了  │ │ ← Deep Agents SubAgent  │
│                              │ │SubAgent │ │   (对你透明)              │
│                              │ └─────────┘ │                │
│                              └──────┬──────┘                │
│                                     │                       │
│  测试执行 ──→ 测试审查 ──→ 人工Approve ──→ CI/CD            │
│   (Node)    (SubGraph)    (Node)         (Node)             │
│                                                             │
└─────────────────────────────────────────────────────────────┘

图例:
  (Node)     = 普通节点（LLM 调用 / 确定性逻辑）
  (SubGraph) = LangGraph SubGraph — 你在代码中预定义的模块化子图
  SubAgent   = Deep Agents SubAgent — Open SWE 内部 LLM 动态派生的（对你透明）
```

---

## 六、什么时候 SubGraph 会不够用，需要 SubAgent？

当你的某个节点内部的任务**高度开放、无法预定义执行路径**时，才需要 SubAgent。在你当前的架构中：

| 节点 | 用 SubGraph 够了吗 | 理由 |
|------|-------------------|------|
| 需求分析 | ✅ 够了 | 流程确定：检索→分析→输出 |
| PRD 产出 | ✅ 够了 | 一次 LLM 调用 |
| 技术方案 | ✅ 够了 | 检索+生成，流程确定 |
| **编码实现** | ⚠️ 不够 | 需要自主决定读哪些文件、改哪些代码、何时跑测试、如何修复错误 → **已由 Open SWE 内部用 SubAgent 解决** |
| 测试审查 | ✅ 够了 | 读 diff→检索规范→评估→输出报告 |
| 未来的安全扫描 | ✅ 大概率够 | 扫描→检索漏洞库→评估→输出报告 |

**结论**：在你的架构中，唯一需要 SubAgent 能力的地方（编码实现）已经被 Open SWE 封装了。你自己构建的所有节点，用 LangGraph SubGraph 就足够了。
