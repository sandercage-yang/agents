# 企业级自动化研发平台 — Coding Agent 集成方案

## 一、需求分析

### 平台定位
构建一个企业级自动化研发平台，核心流程为：

```
需求分析 → PRD产出 → 技术方案 → 编码实现 → 自动化测试 → CI/CD集成
                          ↕ (人工Review & Approval)
```

### 对 Coding Agent 的核心要求

| 维度 | 要求 |
|------|------|
| **可编程性** | 必须提供 Python SDK / API，能被 MainAgent (LangChain) 编程调用 |
| **沙箱隔离** | 代码执行必须在隔离环境中运行 (Docker/K8s)，保证安全 |
| **模型无关** | 支持切换不同 LLM（Claude、GPT、DeepSeek、本地模型等） |
| **人工介入** | 支持 Human-in-the-Loop，关键节点可暂停等待审批 |
| **企业级特性** | RBAC、审计日志、可观测性 |
| **开源可控** | 可自托管，数据不出企业内网 |

---

## 二、开源 Coding Agent 横向对比

### 2.1 候选项目

| 项目 | 技术栈 | SDK/API | 沙箱 | 多Agent协作 | 适合场景 |
|------|--------|---------|------|-------------|----------|
| **OpenHands** | Python | ✅ Python SDK | ✅ Docker | ✅ 多Agent委派 | 企业级全流程自动化 |
| **Open SWE (LangChain)** | Python/LangGraph | ✅ LangGraph原生 | ✅ Modal/Daytona | ✅ 深度Agent架构 | LangChain生态无缝集成 |
| **SWE-Agent** | Python | ⚠️ 仅CLI | ✅ Docker | ❌ | 研究/CI自动修复 |
| **Aider** | Python | ⚠️ 仅CLI | ❌ | ❌ | 个人开发者辅助 |
| **Cline** | TypeScript | ⚠️ VSCode扩展 | ❌ | ❌ | IDE内交互式编码 |

### 2.2 推荐排名

#### 🥇 首选：Open SWE (langchain-ai/open-swe)

**推荐理由：**
- **与你的技术栈天然契合**：你的 MainAgent 使用 LangChain，Open SWE 基于 LangGraph 构建，同属 LangChain 生态，集成零摩擦
- **异步深度Agent架构**：Manager → Planner → Programmer → Reviewer 的多Agent工作流，与你的平台流程（需求→方案→编码→测试）高度吻合
- **Human-in-the-Loop 原生支持**：Planner 阶段支持人工审批，与你"关键节点Review"的需求完美匹配
- **沙箱执行**：每个任务在隔离环境中运行（支持 Modal、Daytona、Runloop 等）
- **可深度定制**：可替换沙箱提供商、模型、工具、系统提示词和中间件

#### 🥈 备选：OpenHands

**推荐理由：**
- **最成熟的企业级开源方案**：内置 RBAC、审计日志
- **独立的 Python SDK**：`openhands-sdk` 包提供完整的编程接口
- **强大的沙箱**：成熟的 Docker 沙箱方案
- **模型无关**：支持 75+ LLM 提供商

**劣势：**
- 与 LangChain 生态不是原生集成，需要额外的适配层

---

## 三、推荐集成方案

### 方案 A：集成 Open SWE（首选 — LangChain 原生）

Open SWE 基于 LangGraph Deep Agents 构建，可作为你 MainAgent 的一个 SubGraph 直接嵌入。

#### 核心集成代码

```python
# === 方案A: 将 Open SWE 作为 LangGraph SubAgent 集成到 MainAgent ===

from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from open_swe.agent import create_coding_agent  # Open SWE 的核心构建函数

# 1. 构建 Coding SubAgent（基于 Open SWE）
coding_agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-20250514",
    system_prompt="你是一个专业的编码Agent，根据技术方案完成代码实现...",
    tools=[http_request, fetch_url, commit_and_open_pr],
    backend=sandbox_backend,  # Modal / Daytona / 自定义沙箱
)

# 2. 定义 MainAgent 的工作流状态
class PlatformState(TypedDict):
    requirement: str        # 原始需求
    prd: str               # PRD文档
    tech_design: str       # 技术方案
    code_result: str       # 编码结果
    test_result: str       # 测试结果
    approval_status: dict  # 审批状态

# 3. 在 MainAgent 中调用 Coding Agent
async def coding_node(state: PlatformState) -> PlatformState:
    """MainAgent 流程中的编码节点 — 委托给 Open SWE"""
    from langgraph_sdk import get_client
    
    client = get_client()  # 连接 LangGraph Platform
    
    # 创建一个 Coding Agent 会话
    thread = await client.threads.create()
    run = await client.runs.create(
        thread_id=thread["thread_id"],
        assistant_id="open-swe-coding-agent",
        input={
            "messages": [{
                "role": "user",
                "content": f"根据以下技术方案进行编码实现:\n{state['tech_design']}"
            }]
        },
    )
    
    # 等待执行完成（支持人工审批中断）
    result = await client.runs.join(thread["thread_id"], run["run_id"])
    
    return {**state, "code_result": result["output"]}

# 4. 构建完整的 MainAgent 工作流
workflow = StateGraph(PlatformState)
workflow.add_node("requirement_analysis", requirement_analysis_node)
workflow.add_node("prd_generation", prd_generation_node)
workflow.add_node("tech_design", tech_design_node)
workflow.add_node("human_review", human_review_node)      # 人工审批节点
workflow.add_node("coding", coding_node)                   # ← Open SWE
workflow.add_node("testing", testing_node)
workflow.add_node("cicd_integration", cicd_node)

workflow.add_edge(START, "requirement_analysis")
workflow.add_edge("requirement_analysis", "prd_generation")
workflow.add_edge("prd_generation", "tech_design")
workflow.add_edge("tech_design", "human_review")
workflow.add_conditional_edges("human_review", check_approval, {
    "approved": "coding",
    "rejected": "tech_design",
})
workflow.add_edge("coding", "testing")
workflow.add_edge("testing", "cicd_integration")
workflow.add_edge("cicd_integration", END)

app = workflow.compile(checkpointer=MemorySaver())
```

### 方案 B：集成 OpenHands SDK（备选 — 独立SDK调用）

如果你更倾向于将 Coding Agent 作为一个独立的微服务来使用，OpenHands SDK 提供了更简洁的编程接口。

#### 核心集成代码

```python
# === 方案B: 将 OpenHands 作为 LangChain Tool 集成到 MainAgent ===

from langchain_core.tools import tool
from pydantic import SecretStr
from openhands.sdk import LLM, Agent, Conversation
from openhands.tools.file_editor import FileEditorTool
from openhands.tools.terminal import TerminalTool

# 1. 封装 OpenHands 为 LangChain Tool
@tool
def coding_agent_tool(task_description: str, workspace_path: str) -> str:
    """调用 OpenHands Coding Agent 完成编码任务。
    
    Args:
        task_description: 编码任务描述（技术方案 + 实现要求）
        workspace_path: 代码仓库工作目录路径
    """
    llm = LLM(
        model="anthropic/claude-sonnet-4-20250514",
        api_key=SecretStr("YOUR_API_KEY"),
    )
    
    agent = Agent(
        llm=llm,
        tools=[TerminalTool(), FileEditorTool()],
    )
    
    conversation = Conversation(agent=agent, workspace=workspace_path)
    conversation.send_message(task_description)
    result = conversation.run()
    
    return result.final_output

# 2. 在 MainAgent 中注册该 Tool
from langchain.agents import create_tool_calling_agent
from langchain_anthropic import ChatAnthropic

main_llm = ChatAnthropic(model="claude-sonnet-4-20250514")
tools = [
    requirement_analysis_tool,
    prd_generation_tool,
    tech_design_tool,
    coding_agent_tool,       # ← OpenHands Coding Agent
    testing_tool,
    cicd_trigger_tool,
]

main_agent = create_tool_calling_agent(main_llm, tools, prompt_template)
```

---

## 四、方案对比与建议

| 维度 | 方案A: Open SWE | 方案B: OpenHands SDK |
|------|-----------------|---------------------|
| **与LangChain集成度** | ⭐⭐⭐⭐⭐ 原生 | ⭐⭐⭐ 需适配 |
| **多Agent协作** | ⭐⭐⭐⭐⭐ LangGraph原生 | ⭐⭐⭐⭐ 事件流架构 |
| **企业级成熟度** | ⭐⭐⭐ 较新但活跃 | ⭐⭐⭐⭐⭐ 成熟稳定 |
| **沙箱安全性** | ⭐⭐⭐⭐ 云沙箱 | ⭐⭐⭐⭐⭐ Docker成熟方案 |
| **Human-in-the-Loop** | ⭐⭐⭐⭐⭐ LangGraph Interrupt | ⭐⭐⭐⭐ 事件驱动 |
| **可观测性** | ⭐⭐⭐⭐⭐ LangSmith原生 | ⭐⭐⭐⭐ MLflow集成 |
| **部署复杂度** | 中等 (需要 LangGraph Platform) | 低 (pip install) |

### 最终建议

> **推荐方案A (Open SWE) 作为首选**，理由：
> 1. 你的 MainAgent 已经选定 LangChain，Open SWE 同属一个生态，集成成本最低
> 2. LangGraph 的 StateGraph + Interrupt 机制天然适配你的"人工审批"需求
> 3. Deep Agent 架构的 Planner → Programmer → Reviewer 流程与你的平台流程高度吻合
> 4. LangSmith 提供开箱即用的 Tracing/Observability，方便调试和监控
>
> **保留方案B (OpenHands) 作为备选**，适用于以下场景：
> - 需要更强的沙箱隔离（OpenHands 的 Docker 方案更成熟）
> - 需要离线/内网部署（OpenHands 对本地模型支持更好）
> - 需要 RBAC 等更成熟的企业治理能力

---

## 五、快速启动步骤

### Open SWE 集成启动

```bash
# 1. 安装依赖
pip install langgraph langgraph-sdk open-swe

# 2. 克隆 Open SWE 进行定制
git clone https://github.com/langchain-ai/open-swe.git
cd open-swe

# 3. 配置环境变量
export ANTHROPIC_API_KEY="your-key"
export GITHUB_TOKEN="your-token"

# 4. 本地开发启动
langgraph dev
```

### OpenHands SDK 集成启动

```bash
# 1. 安装 SDK
pip install openhands-sdk

# 2. 启动本地 Agent
python -c "
from openhands.sdk import LLM, Agent, Conversation
from openhands.tools.terminal import TerminalTool

llm = LLM(model='anthropic/claude-sonnet-4-20250514')
agent = Agent(llm=llm, tools=[TerminalTool()])
conv = Conversation(agent=agent, workspace='./my-project')
conv.send_message('List all Python files and summarize the project structure')
conv.run()
"
```

---

## 六、参考资源

- [Open SWE GitHub](https://github.com/langchain-ai/open-swe)
- [Open SWE 官方博客](https://blog.langchain.com/introducing-open-swe-an-open-source-asynchronous-coding-agent/)
- [OpenHands GitHub](https://github.com/All-Hands-AI/OpenHands)
- [OpenHands SDK 文档](https://docs.openhands.dev/sdk)
- [LangGraph 文档](https://langchain-ai.github.io/langgraph/)
- [LangSmith 可观测性](https://smith.langchain.com/)
