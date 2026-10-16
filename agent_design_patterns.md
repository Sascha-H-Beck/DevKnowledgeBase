# Zero to Hero: Agent Design Patterns

## Overview

Agent design patterns are standardized blueprints for handling agent reasoning, planning, and coordination. In 2026, the industry has converged on **7 canonical patterns** organized into single-agent and multi-agent categories. Understanding when to apply each pattern — and when **not** to — is critical for building reliable agentic systems.

An **agent** is not merely an LLM with a prompt. It is a control loop: observe state, reason, act (via tools or other agents), observe results, and repeat until a termination condition is met. Patterns differ in *how* that loop is structured — whether reasoning and action are interleaved, separated into phases, or distributed across specialized actors.

The cost of choosing the wrong pattern is measurable:

| Mistake | Symptom | Fix |
|---|---|---|
| ReAct for rigid pipelines | Skipped steps, wrong order | Plan-and-Execute |
| No reflection on high-stakes output | Hallucinated facts, bad code | Add Reflection or Debate/Critic |
| Multi-agent for simple tasks | 3–10× token cost, latency | Tool Use only |
| Plan-and-Execute for exploratory research | Brittle plans, no mid-run adaptation | ReAct |

This guide covers all seven patterns with architecture diagrams, decision guidance, and runnable code in Python (LangGraph) and Elixir (BEAM).

---

## Key Concepts

### What Makes Something an "Agent Pattern"?

Every pattern implements the same abstract loop with different topology:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Observe   │ ──► │   Reason    │ ──► │    Act      │
│  (context,  │     │  (plan,     │     │  (tools,    │
│   memory)   │     │   decide)   │     │   delegate) │
└─────────────┘     └─────────────┘     └──────┬──────┘
       ▲                                        │
       └────────────────────────────────────────┘
                    (loop until done)
```

Patterns answer three design questions:

1. **Where does reasoning live?** Inline (ReAct), post-hoc (Reflection), upfront (Plan-and-Execute), or distributed (Multi-Agent).
2. **How many actors?** One LLM (single-agent) or orchestrated specialists (multi-agent).
3. **What terminates the loop?** Tool result, quality threshold, plan completion, or supervisor decision.

### The 7 Core Patterns

| Pattern | Category | Best For | Framework Support |
|---|---|---|---|
| ReAct (Reason + Act) | Single-agent | Open-ended, exploratory tasks | LangGraph, LangChain, all frameworks |
| Reflection | Single-agent | Output quality improvement | LangGraph, DSPy |
| Plan-and-Execute | Single-agent | Structured multi-step tasks | LangGraph, custom |
| Tool Use | Single-agent | External actions | All frameworks |
| Supervisor-Worker | Multi-agent | Cross-functional workflows | LangGraph, OpenAI Agents SDK |
| Debate/Critic | Multi-agent | Verification, adversarial review | AutoGen, custom |
| Swarm | Multi-agent | High-volume parallel tasks | OpenAI Swarm, Elixir BEAM |

### Pattern Composition

Patterns are **composable**, not mutually exclusive. Production systems often stack them:

```
User Query
    │
    ▼
┌───────────────────┐
│  Supervisor       │  ← Pattern 5: routes to specialists
└─────────┬─────────┘
          │
    ┌─────┴─────┐
    ▼           ▼
 Research     Coding
 Agent        Agent
 (ReAct)      (ReAct + Reflection)
    │           │
    └─────┬─────┘
          ▼
    Final Answer
```

A coding sub-agent might run ReAct internally, then Reflection before returning to the supervisor. A research pipeline might use Plan-and-Execute at the top level with ReAct executors per step.

### Decision Tree

Use this tree before writing any agent code:

```
Is it a single LLM call + tools? → Tool Use only
Does output need verification? → Add Reflection
Does it need multi-step with dependencies? → Planning (ReAct or Plan-and-Execute)
Does it need specialized roles exceeding single context? → Multi-Agent
```

**Expanded decision logic:**

```
START: What is the task?
│
├─ Can it be solved in ONE generation + optional tool calls?
│   └─ YES → Tool Use (Pattern 4). Stop here.
│
├─ Does the task require iterating based on tool results?
│   └─ YES → ReAct (Pattern 1)
│       │
│       ├─ Are steps strictly ordered with dependencies?
│       │   └─ YES → Consider Plan-and-Execute (Pattern 3) instead
│       │
│       └─ Is output quality the bottleneck (not tool access)?
│           └─ YES → Add Reflection (Pattern 2) after generation
│
├─ Does the task span multiple domains (research + code + review)?
│   └─ YES → Supervisor-Worker (Pattern 5)
│
├─ Is the decision safety-critical (legal, medical, financial)?
│   └─ YES → Add Debate/Critic (Pattern 6) for verification
│
└─ Do you need to process thousands of items in parallel?
    └─ YES → Swarm (Pattern 7)
```

### Terminology

| Term | Definition |
|---|---|
| **Tool** | A callable function the LLM can invoke (search, DB query, API call) |
| **Observation** | The result returned after a tool call or sub-agent action |
| **Thought / Reasoning trace** | The LLM's intermediate reasoning before acting |
| **Handoff** | Transferring control from one agent to another (OpenAI Agents SDK term) |
| **Subgraph** | A nested LangGraph graph invoked as a node in a parent graph |
| **Checkpoint** | Persisted graph state enabling resume after failure |
| **Human-in-the-loop (HITL)** | Interrupt point where a human approves or edits before continuing |

---

## How It Works

### Pattern 1: ReAct (Reason + Act)

The foundational agent loop, introduced by Yao et al. (2022). Interleaves reasoning traces with tool calls:

```
Thought → Action → Observation → Thought → Action → Observation → Final Answer
```

**Architecture:**

```
┌──────────────────────────────────────────────────────────────┐
│                     ReAct Agent Loop                          │
│                                                               │
│  ┌─────────┐    ┌─────────┐    ┌──────────────┐              │
│  │  LLM    │───►│  Tool   │───►│ Observation  │──┐           │
│  │ (reason)│    │  Call   │    │  (result)    │  │           │
│  └─────────┘    └─────────┘    └──────────────┘  │           │
│       ▲                                          │           │
│       └──────────────────────────────────────────┘           │
│                    (until final answer)                       │
└──────────────────────────────────────────────────────────────┘
```

**When to use:**

- Default for most agents
- Excels at exploratory, open-ended tasks (research, debugging, data investigation)
- Works well when the *path* to the answer is unknown upfront

**When NOT to use:**

- Strict sequential requirements (step 3 must never run before step 1 completes)
- Tasks where mid-stream replanning is expensive and plans are stable

**Strengths:** Adaptive, handles surprises, minimal upfront structure.

**Weaknesses:** Can drift, repeat tool calls, struggle with long-horizon tasks without memory aids.

**LangGraph implementation:** `create_react_agent()` wraps the loop as a stateful graph with automatic tool routing.

---

### Pattern 2: Reflection

The agent reviews its own output against criteria and self-corrects:

```
Generate → Evaluate → Critique → Revise → (repeat until satisfactory)
```

**Architecture:**

```
┌────────────┐     ┌────────────┐     ┌────────────┐
│  Generator │ ──► │  Evaluator │ ──► │  Satisfied?│──► Output
│    LLM     │     │    LLM     │     └─────┬──────┘
└────────────┘     └────────────┘           │ NO
       ▲                                    │
       └────────── Revise ◄─────────────────┘
```

**When to use:**

- Output quality is the bottleneck, not tool access or planning
- Code generation, long-form writing, structured data extraction
- LangChain 2026 Report: **32% cite output quality as top blocker** for production agent adoption

**Variations:**

| Variant | Description |
|---|---|
| **Self-reflection** | Same model generates and critiques |
| **Reflexion** | Persistent memory of past failures across episodes |
| **LLM-as-judge** | Separate (often stronger) model evaluates output |
| **Structured rubric** | Evaluation against explicit JSON criteria |

**Cost trade-off:** Each reflection cycle adds 1–2 LLM calls. Cap iterations (typically 2–3) to control cost.

**When NOT to use:**

- Latency-sensitive paths where single-shot accuracy is already >95%
- Tasks where verification is better done by a separate Debate/Critic pattern

---

### Pattern 3: Plan-and-Execute

Separates planning from execution:

```
Planner LLM → Full Plan → Executor (cheaper model) → Step 1 → Step 2 → ... → Result
```

**Architecture:**

```
┌─────────────┐     ┌─────────────────────────────────────┐
│   Planner   │     │            Executor                  │
│  (strong    │     │  ┌──────┐   ┌──────┐   ┌──────┐     │
│   model)    │────►│  │Step 1│──►│Step 2│──►│Step N│     │
└─────────────┘     │  └──┬───┘   └──┬───┘   └──┬───┘     │
                    │     │ tools     │ tools     │        │
                    └─────┼───────────┼───────────┼────────┘
                          ▼           ▼           ▼
                       Results accumulated in state
```

**When to use:**

- Long, structured tasks where mid-stream drift is costly (ETL pipelines, multi-file refactors, report generation)
- Cost optimization: use GPT-4o/Claude Opus for planning, GPT-4o-mini/Haiku for execution
- Auditable workflows where the plan is a deliverable

**When NOT to use:**

- Exploratory research where the plan will change after step 1
- Tasks requiring tight feedback loops between steps

**Strengths:** Predictable structure, cheaper at scale, easier to debug (inspect the plan).

**Weaknesses:** Brittle when plans need mid-run adaptation; replanning adds complexity.

**Replanning variant:** After each step (or on failure), a lightweight replanner updates remaining steps — hybrid of ReAct flexibility with Plan-and-Execute structure.

---

### Pattern 4: Tool Use

The simplest pattern: a single LLM call with access to external tools.

```
User Query → LLM (+ tool definitions) → [optional tool calls] → Response
```

**Architecture:**

```
┌──────────┐     ┌─────────────────────┐     ┌──────────┐
│   User   │────►│  LLM + Tool Schema  │────►│ Response │
└──────────┘     │  (one generation    │     └──────────┘
                 │   or tool loop)     │
                 └─────────────────────┘
```

**When to use:**

- Task resolves in one generation + zero or one round of tool calls
- Classification with lookup, simple Q&A with search, date/time queries
- Function calling APIs (OpenAI, Anthropic, Gemini) handle this natively

**When NOT to use:**

- Multi-step reasoning where each step depends on prior results
- Any task where you'd write "and then" more than twice in the procedure

**This is not an agent loop** in the strict sense — it is enhanced LLM inference. Many teams over-engineer agents when Tool Use suffices.

---

### Pattern 5: Supervisor-Worker (Multi-Agent)

One orchestrator routes work to specialized sub-agents:

```
Supervisor → Research Agent → Coding Agent → Review Agent → Supervisor → Final
```

**Architecture:**

```
                    ┌─────────────────┐
                    │   Supervisor    │
                    │  (orchestrator) │
                    └────────┬────────┘
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
    ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
    │  Research   │   │   Coding    │   │   Review    │
    │   Worker    │   │   Worker    │   │   Worker    │
    └─────────────┘   └─────────────┘   └─────────────┘
           │                 │                 │
           └─────────────────┼─────────────────┘
                             ▼
                    Results back to Supervisor
```

**When to use:**

- Cross-domain tasks requiring different tool sets and prompts
- The 2026 production default for complex workflows (customer support triage, software engineering agents, research reports)
- When a single context window would mix incompatible instructions

**Framework implementations:**

| Framework | Mechanism |
|---|---|
| **LangGraph** | Supervisor via subgraphs or `langgraph-supervisor`; workers as compiled subgraphs |
| **OpenAI Agents SDK** | Handoffs between agents with typed transfer |
| **Claude Agent SDK** | Subagents with isolated context and tool sets |
| **CrewAI** | Role-based agents with task delegation |

**Production note (2026):** LangChain recommends **subagents-as-tools** over pure supervisor routing for better type safety and control flow. Supervisors remain excellent for prototyping and LangGraph Studio visualization.

**When NOT to use:**

- Single-domain tasks where one ReAct agent with all tools works fine
- When coordination overhead exceeds the benefit (2 agents is often enough; 10 agents rarely is)

---

### Pattern 6: Debate/Critic

Two or more agents argue opposing positions or one generates while another critiques:

```
Proposer → Critic → Proposer (revised) → Critic → ... → Consensus / Judge
```

**Architecture:**

```
┌──────────┐         ┌──────────┐
│ Proposer │ ◄─────► │  Critic  │
│  Agent   │  debate │  Agent   │
└────┬─────┘         └────┬─────┘
     │                    │
     └────────┬───────────┘
              ▼
       ┌─────────────┐
       │    Judge    │  (optional third agent)
       │   Agent     │
       └─────────────┘
```

**When to use:**

- Verification through adversarial review
- Safety-critical decisions (policy compliance, medical triage support, financial recommendations)
- Reducing sycophancy — critics are prompted to find flaws, not agree

**Variations:**

| Style | Example |
|---|---|
| **Proposer-Critic** | One generates, one finds errors |
| **Multi-perspective debate** | Three agents with assigned viewpoints |
| **Self-consistency voting** | N independent agents, majority vote (related to prompt engineering technique) |

**When NOT to use:**

- Low-stakes creative tasks where debate kills momentum
- When latency budget cannot absorb 2–5× LLM calls

---

### Pattern 7: Swarm

Massive parallel agent execution — many independent agents processing many items simultaneously:

```
                    ┌─── Agent 1 ─── Task A ───┐
Orchestrator ───────├─── Agent 2 ─── Task B ───├─── Aggregate
                    ├─── Agent 3 ─── Task C ───┤
                    └─── Agent N ─── Task N ───┘
```

**Architecture (BEAM ideal):**

```
┌─────────────────────────────────────────────────────────────┐
│                    Swarm Supervisor                          │
│  (DynamicSupervisor or Horde.UniformSupervisor)             │
└──────────────────────────┬──────────────────────────────────┘
                           │ spawns N lightweight processes
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
    ┌─────────┐       ┌─────────┐       ┌─────────┐
    │ Agent   │       │ Agent   │  ...  │ Agent   │
    │ Task 1  │       │ Task 2  │       │ Task N  │
    └─────────┘       └─────────┘       └─────────┘
         │                 │                 │
         └─────────────────┼─────────────────┘
                           ▼
                    Result aggregation
```

**When to use:**

- High-volume parallel tasks: batch document classification, per-row data enrichment, parallel web scraping
- Fan-out/fan-in patterns where tasks are independent

**Platform fit:**

| Platform | Why |
|---|---|
| **Elixir/BEAM** | Millions of lightweight processes, preemptive scheduling, fault isolation — architecturally ideal |
| **OpenAI Swarm** | Python library for lightweight multi-agent handoffs (educational/prototyping; production often moves to Agents SDK) |
| **LangGraph** | Parallel node execution with map-reduce patterns |

**When NOT to use:**

- Tasks with heavy inter-agent dependencies (use Supervisor-Worker instead)
- When aggregate context must be shared in real-time across all agents

---

## Code Example

### Python Setup

All Python examples use LangGraph 0.3+ and LangChain 0.3+. Install dependencies:

```bash
pip install langgraph langchain-openai langgraph-supervisor tavily-python
export OPENAI_API_KEY="sk-..."
export TAVILY_API_KEY="tvly-..."  # for web search examples
```

---

### Example 1: ReAct Agent with Tools (Python / LangGraph)

A research assistant that searches the web and synthesizes answers using the ReAct loop.

```python
"""ReAct agent: interleaved reasoning and tool use."""
from typing import Annotated

from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from tavily import TavilyClient

tavily = TavilyClient()


@tool
def web_search(query: Annotated[str, "Search query for current information"]) -> str:
    """Search the web and return top results as formatted text."""
    response = tavily.search(query=query, max_results=3)
    lines = []
    for result in response.get("results", []):
        lines.append(f"- {result['title']}: {result['content'][:300]}")
    return "\n".join(lines) if lines else "No results found."


@tool
def calculate(expression: Annotated[str, "Math expression, e.g. '150000 * 5'"]) -> str:
    """Evaluate a safe arithmetic expression."""
    allowed = set("0123456789+-*/(). ")
    if not all(c in allowed for c in expression):
        return "Error: only basic arithmetic allowed."
    try:
        return str(eval(expression, {"__builtins__": {}}, {}))
    except Exception as exc:
        return f"Error: {exc}"


model = ChatOpenAI(model="gpt-4o-mini", temperature=0)

react_agent = create_react_agent(
    model=model,
    tools=[web_search, calculate],
    prompt=(
        "You are a research assistant. Use web_search for factual questions "
        "about current events. Use calculate for arithmetic. "
        "Think step by step, cite sources, and give concise final answers."
    ),
    name="research_assistant",
)

if __name__ == "__main__":
    result = react_agent.invoke({
        "messages": [{
            "role": "user",
            "content": (
                "What was Apple's revenue in fiscal 2024, and how many times "
                "would it fit into Microsoft's revenue that same year?"
            ),
        }]
    })

    final_message = result["messages"][-1]
    print(final_message.content)
```

**What happens internally:**

1. LLM receives user query + tool schemas.
2. LLM emits a tool call (`web_search`) — the "Action" in ReAct.
3. Tool returns observations (search snippets).
4. LLM reasons over observations, may call `calculate`.
5. LLM produces final answer when no more tools are needed.

The graph handles the loop; you do not write explicit `while` logic.

---

### Example 2: Plan-and-Execute Pattern (Python / LangGraph)

A custom graph that plans steps upfront, then executes each with a cheaper model.

```python
"""Plan-and-Execute: separate planner from step executors."""
import operator
from typing import Annotated, Literal, TypedDict

from langchain_core.messages import HumanMessage, SystemMessage
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from pydantic import BaseModel, Field


# --- Structured plan schema ---

class PlanStep(BaseModel):
    step_number: int = Field(description="Order of execution, starting at 1")
    description: str = Field(description="What this step should accomplish")
    tool_hint: str = Field(description="Which tool or approach to use")


class Plan(BaseModel):
    steps: list[PlanStep] = Field(description="Ordered list of steps to complete the task")


# --- Tools ---

def search_knowledge_base(topic: str) -> str:
    """Simulated internal KB — replace with vector DB / RAG in production."""
    kb = {
        "elixir": "Elixir runs on the BEAM VM with lightweight processes and OTP supervision.",
        "langgraph": "LangGraph builds stateful agent graphs with checkpointing and subgraphs.",
        "react": "ReAct interleaves reasoning traces with tool calls until task completion.",
    }
    for key, value in kb.items():
        if key in topic.lower():
            return value
    return f"No KB entry found for '{topic}'."


def write_summary(content: str, max_words: int = 100) -> str:
    """Format content into a concise summary."""
    words = content.split()
    truncated = " ".join(words[:max_words])
    return f"Summary ({min(len(words), max_words)} words):\n{truncated}"


TOOLS = {"search_knowledge_base": search_knowledge_base, "write_summary": write_summary}


# --- Graph state ---

class PlanExecuteState(TypedDict):
    input: str
    plan: list[PlanStep]
    current_step: int
    step_results: Annotated[list[str], operator.add]
    final_output: str


# --- Models: strong planner, cheap executor ---

planner_model = ChatOpenAI(model="gpt-4o", temperature=0)
executor_model = ChatOpenAI(model="gpt-4o-mini", temperature=0)

planner_chain = (
    ChatPromptTemplate.from_messages([
        ("system", (
            "You are a task planner. Break the user's request into 2-4 concrete, "
            "ordered steps. Each step must be actionable by an executor with "
            "access to: search_knowledge_base, write_summary."
        )),
        ("human", "{input}"),
    ])
    | planner_model.with_structured_output(Plan)
)


def plan_node(state: PlanExecuteState) -> dict:
    plan: Plan = planner_chain.invoke({"input": state["input"]})
    sorted_steps = sorted(plan.steps, key=lambda s: s.step_number)
    return {"plan": sorted_steps, "current_step": 0, "step_results": []}


def execute_step_node(state: PlanExecuteState) -> dict:
    step = state["plan"][state["current_step"]]
    prior = "\n".join(state["step_results"]) or "(none)"

    response = executor_model.invoke([
        SystemMessage(content=(
            "You are a step executor. Complete ONLY the assigned step. "
            "Available tools (call by describing what you'd do, then I'll run it):\n"
            "- search_knowledge_base(topic: str)\n"
            "- write_summary(content: str, max_words: int)\n"
            "Return the step result as plain text."
        )),
        HumanMessage(content=(
            f"Original task: {state['input']}\n"
            f"Prior step results:\n{prior}\n\n"
            f"Execute step {step.step_number}: {step.description}\n"
            f"Tool hint: {step.tool_hint}"
        )),
    ])

    result_text = response.content

    # Simple tool dispatch based on step hint (production: use structured tool calling)
    if "search" in step.tool_hint.lower():
        topic = state["input"]
        kb_result = search_knowledge_base(topic)
        result_text = f"{result_text}\n\n[Tool output]\n{kb_result}"
    elif "summary" in step.tool_hint.lower() or "write" in step.tool_hint.lower():
        source = state["step_results"][-1] if state["step_results"] else result_text
        result_text = write_summary(source)

    return {
        "step_results": [f"Step {step.step_number}: {result_text}"],
        "current_step": state["current_step"] + 1,
    }


def should_continue(state: PlanExecuteState) -> Literal["execute", "finalize"]:
    if state["current_step"] < len(state["plan"]):
        return "execute"
    return "finalize"


def finalize_node(state: PlanExecuteState) -> dict:
    combined = "\n\n".join(state["step_results"])
    response = executor_model.invoke([
        SystemMessage(content="Combine step results into a polished final answer."),
        HumanMessage(content=f"Task: {state['input']}\n\nStep results:\n{combined}"),
    ])
    return {"final_output": response.content}


# --- Build graph ---

builder = StateGraph(PlanExecuteState)
builder.add_node("plan", plan_node)
builder.add_node("execute", execute_step_node)
builder.add_node("finalize", finalize_node)

builder.add_edge(START, "plan")
builder.add_edge("plan", "execute")
builder.add_conditional_edges("execute", should_continue, {
    "execute": "execute",
    "finalize": "finalize",
})
builder.add_edge("finalize", END)

plan_execute_app = builder.compile()


if __name__ == "__main__":
    result = plan_execute_app.invoke({
        "input": "Explain how ReAct agents work and summarize for a technical blog intro.",
        "plan": [],
        "current_step": 0,
        "step_results": [],
        "final_output": "",
    })
    print("=== PLAN ===")
    for step in result["plan"]:
        print(f"  {step.step_number}. {step.description} [{step.tool_hint}]")
    print("\n=== FINAL OUTPUT ===")
    print(result["final_output"])
```

**Key design points:**

- The planner runs once (strong model, structured output).
- The executor runs per step (cheaper model).
- Conditional edges loop until all steps complete.
- Production systems add a **replan node** on step failure.

---

### Example 3: Supervisor-Worker Multi-Agent (Python / LangGraph)

Cross-functional team: researcher finds facts, writer formats output, supervisor coordinates.

```python
"""Supervisor-Worker: orchestrator delegates to specialized ReAct agents."""
from typing import Annotated

from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from langgraph_supervisor import create_supervisor

model = ChatOpenAI(model="gpt-4o-mini", temperature=0)


# --- Worker tools ---

@tool
def search_web(query: Annotated[str, "Search query"]) -> str:
    """Return simulated search results (swap for Tavily/SerpAPI in production)."""
    results = {
        "langgraph supervisor": (
            "LangGraph supports hierarchical multi-agent systems via supervisor "
            "graphs that route tasks to worker subgraphs with isolated tool sets."
        ),
        "beam concurrency": (
            "The BEAM VM runs millions of lightweight processes with preemptive "
            "scheduling and per-process garbage collection."
        ),
    }
    for key, value in results.items():
        if key in query.lower():
            return value
    return f"No results for: {query}"


@tool
def format_markdown_report(
    title: Annotated[str, "Report title"],
    body: Annotated[str, "Raw content to format"],
) -> str:
    """Format research findings into a markdown report."""
    return f"# {title}\n\n{body}\n\n---\n*Generated by writing agent*"


# --- Specialized workers ---

research_agent = create_react_agent(
    model=model,
    tools=[search_web],
    name="researcher",
    prompt=(
        "You are a research specialist. Search for factual information. "
        "Return concise bullet points with sources mentioned. Never write final reports."
    ),
)

writing_agent = create_react_agent(
    model=model,
    tools=[format_markdown_report],
    name="writer",
    prompt=(
        "You are a writing specialist. Take research findings and format them "
        "into clear markdown reports using format_markdown_report. "
        "Do not search the web yourself."
    ),
)

# --- Supervisor ---

supervisor = create_supervisor(
    agents=[research_agent, writing_agent],
    model=ChatOpenAI(model="gpt-4o", temperature=0),
    prompt=(
        "You are a team supervisor managing a researcher and a writer.\n"
        "Workflow:\n"
        "1. Send research questions to the researcher first.\n"
        "2. Send the researcher's findings to the writer for formatting.\n"
        "3. Return the writer's formatted report as the final answer.\n"
        "Always use both agents in order."
    ),
).compile()


if __name__ == "__main__":
    result = supervisor.invoke({
        "messages": [{
            "role": "user",
            "content": (
                "Write a short technical report on LangGraph supervisor patterns "
                "and BEAM concurrency for agent swarms."
            ),
        }]
    })

    for msg in result["messages"]:
        if hasattr(msg, "name") and msg.name:
            print(f"[{msg.name}] {msg.content[:200]}...")
        elif msg.type == "ai":
            print(f"[assistant] {msg.content}")
```

**Supervisor routing flow:**

```
User: "Write a report on X"
  → Supervisor → Researcher (search_web)
  → Supervisor ← Researcher (bullet points)
  → Supervisor → Writer (format_markdown_report)
  → Supervisor ← Writer (formatted report)
  → Supervisor → User (final answer)
```

For production, add `MemorySaver` checkpointing and subagents-as-tools for typed handoffs.

---

### Example 4: Reflection Loop (Python / LangGraph Bonus)

Stacked on any generator — self-critique before returning output.

```python
"""Reflection: generate → evaluate → revise (max 2 iterations)."""
from typing import Literal, TypedDict

from langchain_core.messages import HumanMessage, SystemMessage
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from pydantic import BaseModel, Field

model = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)
judge_model = ChatOpenAI(model="gpt-4o-mini", temperature=0)


class Evaluation(BaseModel):
    passes: bool = Field(description="True if output meets all criteria")
    critique: str = Field(description="Specific issues to fix, empty if passes")
    score: int = Field(description="Quality score 1-10", ge=1, le=10)


class ReflectionState(TypedDict):
    task: str
    draft: str
    critique: str
    iteration: int
    max_iterations: int
    final: str


def generate_node(state: ReflectionState) -> dict:
    messages = [
        SystemMessage(content="You are a technical writer. Be accurate and concise."),
        HumanMessage(content=state["task"]),
    ]
    if state.get("critique"):
        messages.append(HumanMessage(content=f"Revise based on this critique:\n{state['critique']}"))
    response = model.invoke(messages)
    return {"draft": response.content, "iteration": state["iteration"] + 1}


def evaluate_node(state: ReflectionState) -> dict:
    evaluation: Evaluation = judge_model.with_structured_output(Evaluation).invoke([
        SystemMessage(content=(
            "Evaluate the draft against the task. Criteria: factual accuracy, "
            "completeness, clarity. Score >= 8 and no major errors means pass."
        )),
        HumanMessage(content=f"Task: {state['task']}\n\nDraft:\n{state['draft']}"),
    ])
    if evaluation.passes:
        return {"critique": "", "final": state["draft"]}
    return {"critique": evaluation.critique}


def route(state: ReflectionState) -> Literal["generate", "done"]:
    if state.get("final"):
        return "done"
    if state["iteration"] >= state["max_iterations"]:
        return "done"
    if state.get("critique"):
        return "generate"
    return "generate"


builder = StateGraph(ReflectionState)
builder.add_node("generate", generate_node)
builder.add_node("evaluate", evaluate_node)
builder.add_edge(START, "generate")
builder.add_edge("generate", "evaluate")
builder.add_conditional_edges("evaluate", route, {"generate": "generate", "done": END})

reflection_app = builder.compile()
```

---

### Example 5: ReAct Loop with GenServer (Elixir)

A GenServer implements the ReAct loop on the BEAM — one process, sequential thought/action/observation cycles.

```elixir
# mix.exs deps: {:req, "~> 0.5"}, {:jason, "~> 1.4"}
defmodule LearnAgents.ReactAgent do
  @moduledoc """
  ReAct agent as a GenServer: Thought → Action → Observation → loop.
  """
  use GenServer

  @max_iterations 8

  defstruct [
    :task,
    :messages,
    :tools,
    :api_key,
    iteration: 0,
    status: :running,
    result: nil
  ]

  # --- Public API ---

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def run(task) do
    GenServer.call(__MODULE__, {:run, task}, :infinity)
  end

  # --- GenServer callbacks ---

  @impl true
  def init(opts) do
    state = %__MODULE__{
      task: nil,
      messages: [],
      tools: Keyword.get(opts, :tools, default_tools()),
      api_key: Keyword.fetch!(opts, :api_key)
    }

    {:ok, state}
  end

  @impl true
  def handle_call({:run, task}, _from, state) do
    messages = [
      %{"role" => "system", "content" => system_prompt(state.tools)},
      %{"role" => "user", "content" => task}
    ]

    case react_loop(%{state | task: task, messages: messages, iteration: 0, status: :running}) do
      {:ok, result, _final_state} -> {:reply, {:ok, result}, state}
      {:error, reason, _final_state} -> {:reply, {:error, reason}, state}
    end
  end

  # --- ReAct loop ---

  defp react_loop(%{iteration: n} = state) when n >= @max_iterations do
    {:error, :max_iterations, state}
  end

  defp react_loop(state) do
    case call_llm(state) do
      {:ok, %{"content" => content}} when is_binary(content) and content != "" ->
        {:ok, content, %{state | result: content, status: :done}}

      {:ok, %{"tool_calls" => [tool_call | _]}} ->
        observation = execute_tool(state.tools, tool_call)
        tool_call_id = tool_call["id"]
        fn_name = get_in(tool_call, ["function", "name"])
        fn_args = Jason.decode!(tool_call["function"]["arguments"])

        new_messages =
          state.messages ++
            [
              %{
                "role" => "assistant",
                "tool_calls" => [tool_call]
              },
              %{
                "role" => "tool",
                "tool_call_id" => tool_call_id,
                "content" => observation
              }
            ]

        IO.puts("[ReAct] Action: #{fn_name}(#{inspect(fn_args)}) → #{String.slice(observation, 0, 80)}...")

        react_loop(%{state | messages: new_messages, iteration: state.iteration + 1})

      {:error, reason} ->
        {:error, reason, state}
    end
  end

  defp call_llm(state) do
    body = %{
      "model" => "gpt-4o-mini",
      "messages" => state.messages,
      "tools" => Enum.map(state.tools, &tool_schema/1)
    }

    case Req.post("https://api.openai.com/v1/chat/completions",
           json: body,
           headers: [{"authorization", "Bearer #{state.api_key}"}]
         ) do
      {:ok, %{status: 200, body: %{"choices" => [%{"message" => message}]}}} ->
        {:ok, message}

      {:ok, %{body: body}} ->
        {:error, {:api_error, body}}

      {:error, reason} ->
        {:error, reason}
    end
  end

  defp execute_tool(tools, %{"function" => %{"name" => name, "arguments" => args_json}}) do
    args = Jason.decode!(args_json)
    tool = Enum.find(tools, &(&1.name == name))

    case tool do
      nil -> "Unknown tool: #{name}"
      %{handler: handler} -> handler.(args)
    end
  end

  defp system_prompt(tools) do
    tool_list = Enum.map_join(tools, ", ", & &1.name)

    """
    You are a ReAct agent. Reason about the task, call tools when needed (#{tool_list}),
    observe results, and continue until you can answer directly without tools.
    """
  end

  defp default_tools do
    [
      %{
        name: "search_docs",
        description: "Search internal documentation by keyword",
        parameters: %{
          "type" => "object",
          "properties" => %{"query" => %{"type" => "string"}},
          "required" => ["query"]
        },
        handler: fn %{"query" => query} ->
          # Replace with RAG / DB lookup in production
          if String.contains?(String.downcase(query), "otp") do
            "OTP provides supervision trees, GenServers, and fault-tolerant processes."
          else
            "No documentation found for '#{query}'."
          end
        end
      },
      %{
        name: "calculate",
        description: "Evaluate a basic arithmetic expression",
        parameters: %{
          "type" => "object",
          "properties" => %{"expression" => %{"type" => "string"}},
          "required" => ["expression"]
        },
        handler: fn %{"expression" => expr} ->
          {result, _} = Code.eval_string(expr, [], __ENV__)
          to_string(result)
        rescue
          _ -> "Invalid expression"
        end
      }
    ]
  end

  defp tool_schema(%{name: name, description: desc, parameters: params}) do
    %{
      "type" => "function",
      "function" => %{
        "name" => name,
        "description" => desc,
        "parameters" => params
      }
    }
  end
end
```

**Usage:**

```elixir
# In application supervisor:
# {LearnAgents.ReactAgent, api_key: System.get_env("OPENAI_API_KEY")}

{:ok, answer} = LearnAgents.ReactAgent.run(
  "Search docs for OTP and calculate 2^10"
)
IO.puts(answer)
```

Each GenServer handles one agent instance. The loop is sequential within the process — perfect for a single user's conversational agent.

---

### Example 6: BEAM Processes as Agent Swarms (Elixir)

Swarm pattern: spawn one lightweight process per task, aggregate results fault-tolerantly.

```elixir
defmodule LearnAgents.Swarm do
  @moduledoc """
  Swarm pattern: parallel agent tasks via Task.async_stream.
  Each task runs in its own BEAM process — ideal for batch enrichment.
  """
  use GenServer

  @default_concurrency 50
  @default_timeout :timer.seconds(30)

  defstruct api_key: nil

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def enrich_batch(items, opts \\ []) when is_list(items) do
    GenServer.call(__MODULE__, {:enrich_batch, items, opts}, :infinity)
  end

  @impl true
  def init(opts) do
    {:ok, %__MODULE__{api_key: Keyword.fetch!(opts, :api_key)}}
  end

  @impl true
  def handle_call({:enrich_batch, items, opts}, _from, state) do
    concurrency = Keyword.get(opts, :concurrency, @default_concurrency)
    timeout = Keyword.get(opts, :timeout, @default_timeout)

    results =
      items
      |> Task.async_stream(
        &process_item(&1, state.api_key),
        max_concurrency: concurrency,
        timeout: timeout,
        ordered: false,
        on_timeout: :kill_task
      )
      |> Enum.map(fn
        {:ok, result} -> result
        {:exit, reason} -> {:error, reason}
      end)

    {:reply, {:ok, results}, state}
  end

  defp process_item(%{id: id, text: text}, api_key) do
    # Each item gets its own process — isolated failure domain
    case classify_and_extract(text, api_key) do
      {:ok, data} -> {:ok, id, data}
      {:error, reason} -> {:error, id, reason}
    end
  end

  defp classify_and_extract(text, api_key) do
    body = %{
      "model" => "gpt-4o-mini",
      "messages" => [
        %{
          "role" => "system",
          "content" => ~s(
            Classify the text and extract entities. Respond ONLY with JSON:
            {"category": "...", "entities": ["..."], "sentiment": "positive|neutral|negative"}
          )
        },
        %{"role" => "user", "content" => text}
      ],
      "response_format" => %{"type" => "json_object"}
    }

    with {:ok, %{status: 200, body: body}} <-
           Req.post("https://api.openai.com/v1/chat/completions",
             json: body,
             headers: [{"authorization", "Bearer #{api_key}"}]
           ),
         [%{"message" => %{"content" => content}}] <- body["choices"],
         {:ok, parsed} <- Jason.decode(content) do
      {:ok, parsed}
    else
      error -> {:error, error}
    end
  end
end
```

**Why BEAM excels at swarms:**

| BEAM Property | Agent Swarm Benefit |
|---|---|
| Lightweight processes (~2 KB each) | Spawn thousands of agent workers |
| Preemptive scheduling | No single slow agent blocks others |
| Process isolation | One failed classification doesn't crash the batch |
| Supervision trees | `DynamicSupervisor` restarts failed agents |
| Distribution | Same code runs agents across cluster nodes |

**Scaling beyond `Task.async_stream`:**

```elixir
# For persistent agent pools, use Horde.UniformSupervisor or DynamicSupervisor:

children = [
  {DynamicSupervisor, name: LearnAgents.AgentSupervisor, strategy: :one_for_one}
]

# Spawn per-task agents:
{:ok, pid} = DynamicSupervisor.start_child(
  LearnAgents.AgentSupervisor,
  {LearnAgents.ReactAgent, api_key: api_key}
)
```

**Usage — process 1,000 support tickets in parallel:**

```elixir
tickets =
  for i <- 1..1000 do
    %{id: i, text: "Customer report ##{i}: issue description..."}
  end

{:ok, results} = LearnAgents.Swarm.enrich_batch(tickets, concurrency: 100)

successes = Enum.count(results, &match?({:ok, _, _}, &1))
IO.puts("Processed #{successes}/1000 tickets")
```

---

## Practical Application

### Choosing the Right Pattern — Decision Matrix

| Task Profile | Recommended Pattern | Example |
|---|---|---|
| Single lookup + format | Tool Use | "What's the weather in Berlin?" |
| Exploratory debugging | ReAct | "Why is this API returning 500?" |
| Blog post draft | ReAct + Reflection | Generate → critique → revise |
| Multi-file code migration | Plan-and-Execute | Plan files → execute per file |
| Research + write + review | Supervisor-Worker | Report generation pipeline |
| Compliance check on contract | Debate/Critic | Proposer vs. legal critic |
| Classify 10K documents | Swarm | Parallel classification agents |

### Framework Comparison (2026)

| Framework | Strengths | Best For |
|---|---|---|
| LangGraph | Stateful graphs, checkpointing, durable execution, subgraphs | Mission-critical, deterministic workflows |
| CrewAI | Role-based teams, intuitive API, quick setup | Rapid prototyping, demo-to-MVP |
| AutoGen/AG2 | Conversation-based, flexible agent chat | Research, experimentation, debate patterns |
| OpenAI Agents SDK | Native OpenAI integration, handoffs, tracing | OpenAI-first stacks |
| Claude Agent SDK | Subagents, tool orchestration, MCP native | Anthropic-first stacks |
| LangChain (agents) | Broad ecosystem, `create_agent` abstraction | General-purpose, multi-provider |

### LangGraph Production Checklist

When deploying LangGraph agents to production:

1. **Checkpointing** — use `MemorySaver` (dev) or Postgres/Redis (prod) for resumable runs.
2. **Human-in-the-loop** — `interrupt_before` on sensitive tool calls.
3. **Timeouts** — set per-node and global graph timeouts.
4. **Observability** — LangSmith or OpenTelemetry traces for every node transition.
5. **Subagents-as-tools** — prefer typed tool wrappers over free-form supervisor routing.
6. **Token budgets** — cap iterations and message history length.

### Cost Optimization by Pattern

| Pattern | Cost Driver | Optimization |
|---|---|---|
| ReAct | Unbounded tool loops | Max iterations, cache tool results |
| Reflection | 2–3× generation cost | Cap revisions, use cheaper judge model |
| Plan-and-Execute | Planner quality | Strong planner + cheap executor |
| Supervisor-Worker | Coordination overhead | Minimize supervisor model size |
| Swarm | N × per-item cost | Batch API, local models for classification |

### When NOT to Use Agents

Agents add latency, token cost, and failure modes. Skip them when:

| Scenario | Better Approach |
|---|---|
| Simple classification | Single LLM call with structured output |
| Entity extraction from text | Instructor / structured output API |
| Summarization | Direct prompt, no tools |
| Deterministic ETL | Code, not LLM |
| Keyword search | Elasticsearch/pgvector, not agent loop |

**Rule of thumb:** If you can describe the solution as a function with no branching based on intermediate LLM reasoning, you probably do not need an agent.

### Anti-Patterns to Avoid

1. **Agent soup** — 8 agents where 2 would suffice. Every handoff loses context.
2. **ReAct for pipelines** — Use Plan-and-Execute when step order is contractual.
3. **No termination condition** — Always set max iterations, token limits, or timeouts.
4. **Shared mutable state across workers** — Pass results explicitly through supervisor state.
5. **Reflection on every response** — Reserve for quality-critical paths only.

### Combining Patterns with MCP

Agent tools often come from [MCP servers](../agents/mcp_servers_fundamentals.md). The typical stack:

```
Agent (ReAct or Supervisor)
    │
    ├── MCP Client → Database Server (tools: query, insert)
    ├── MCP Client → GitHub Server (tools: list_prs, create_issue)
    └── MCP Client → Internal API Server (tools: create_invoice)
```

MCP decouples tool implementation from agent logic — swap servers without changing agent graphs.

---

## Related Topics

- [MCP Servers — Fundamentals](../agents/mcp_servers_fundamentals.md) — Tool protocol that powers agent Tool Use and worker specializations
- [MCP Servers — Implementation](../agents/mcp_servers_implementation.md) — Building production MCP servers for agent tool access
- [Zero to Hero: Prompt Engineering](../prompt_engineering/zero_to_hero_prompt_engineering.md) — System prompts, CoT, and context engineering for agent instructions
- [AI Security and Guardrails](../evaluation/ai_security_and_guardrails.md) — Guardrails for agent tool calls, output filtering, and red-teaming
- [Zero to Hero: AI Engineering with Elixir](../api_integration/elixir_ai_zero_to_hero.md) — Full Elixir AI stack including LangChain agents and Arcana RAG
- [Implementing LLMs in Elixir](../code_examples/elixir_llm_implementation.md) — Structured outputs, API clients, and production patterns on the BEAM
- [Zero to Hero: RAG Pipelines](../rag/zero_to_hero_rag_pipelines.md) — Retrieval patterns commonly embedded in research worker agents
