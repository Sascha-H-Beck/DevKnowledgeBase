# Zero to Hero: Prompt Engineering & Context Engineering

## Overview

**Prompt engineering** is the practice of crafting instructions, examples, and constraints so a Large Language Model (LLM) produces useful, reliable output. You write text; the model interprets it as context and generates a continuation.

**Context engineering** (the 2026 evolution) goes further: you design the *entire information system* around the model — what goes into the context window, in what order, from which sources, with what retrieval and tooling, and how outputs are validated and fed back. Prompts are one component; RAG chunks, tool results, memory, user history, and structured schemas are others.

Andrej Karpathy captured the mental model succinctly:

> *"The LLM is a CPU, the context window is RAM, and you are the operating system."*

The model's weights are fixed at inference time (the "CPU instruction set"). Everything you can change per request lives in the context window (the "RAM"). Your job as an AI engineer is to load the right programs (system prompts), data (RAG, user input), and I/O formats (schemas, tools) into that window — efficiently, reliably, and measurably.

In 2026, production systems rarely use a single static prompt. They use **modular context assembly**: a pipeline that selects, ranks, compresses, and injects the minimum high-signal tokens needed for the task. Prompt engineering teaches you to write good instructions; context engineering teaches you to architect the full runtime environment the model sees.

---

## Key Concepts

The table below summarizes core techniques, when to reach for them, token cost, and typical accuracy impact. Numbers are approximate and task-dependent; treat them as planning heuristics, not guarantees.

| Technique | Best For | Token Cost | Accuracy Gain |
|---|---|---|---|
| Zero-shot | Simple, clear tasks | Low | Baseline |
| Few-shot | Pattern-matching, format compliance | Medium | +5–10% |
| Chain of Thought (CoT) | Multi-step reasoning | Medium | +10–18% |
| Self-Consistency | High-accuracy reasoning | High (3–5×) | +12–18% |
| Tree of Thoughts (ToT) | Strategic planning | Highest | +20–70% |
| Role/saschahbeck Prompting | Consistent behavior | Low | +consistency |
| Structured Output | Code/data integration | Low | Format reliability |
| System Prompts | Persistent behavior | Low | Consistency |

### Technique Summaries

**Zero-shot** — Give the model a task with no examples. Works when the task is well-defined and within the model's training distribution (classification, summarization, translation).

**Few-shot** — Provide 3–5 input/output pairs before the real query. In 2026, the primary value is **format alignment** (JSON shape, tone, field naming), not teaching the model new reasoning.

**Chain of Thought (CoT)** — Ask the model to show intermediate reasoning steps before the final answer. Improves multi-step math, logic, and planning. Modern reasoning models (OpenAI o-series, DeepSeek-R1) often internalize this; explicit CoT may be redundant or harmful for them.

**Self-Consistency** — Run CoT multiple times at non-zero temperature, extract answers, take a majority vote. Trades compute for reliability on hard reasoning tasks.

**Tree of Thoughts (ToT)** — Explore multiple reasoning branches, evaluate partial solutions, backtrack when a path fails. Best for puzzles, planning, and search-like problems.

**Role/saschahbeck Prompting** — Assign identity, expertise, and tone ("You are a senior tax attorney…"). Improves consistency and reduces generic answers; does not add factual knowledge.

**Structured Output** — Constrain responses to JSON, XML, or schema-validated objects. Essential for pipelines, databases, and UI integration.

**System Prompts** — Persistent instructions separated from the user's task. Define behavior, constraints, and output format across a conversation.

---

## How It Works

### System Prompts

System prompts set global behavior: persona, tone, constraints, safety rules, and output conventions. They are sent once per conversation (or cached server-side) and apply to every turn.

**Keep under ~800 tokens for instruction adherence.** Research and production experience consistently show that very long system prompts suffer from:

- **Instruction dilution** — later rules compete with earlier ones; the model follows a subset unpredictably.
- **Context rot** — performance degrades as total context grows (see Context Engineering below).
- **Higher cost and latency** — system tokens are paid on every request.

**Hierarchical prompting** separates concerns:

| Layer | Content | Example |
|---|---|---|
| System | Persona, tone, hard constraints, output format | "You are a concise financial analyst. Never invent figures. Respond in XML." |
| User | Task-specific query, data, ephemeral context | "Should we extend Net-60 terms to Acme Corp? Here is their payment history: …" |
| Assistant (few-shot) | Demonstration turns | Example Q&A pairs showing desired format |

Do not put the user's data or one-off task details in the system prompt unless they are truly global. System = *how* the model behaves; user = *what* it does right now.

**Model-specific tips (2026):**

| Model family | Preference | Example pattern |
|---|---|---|
| **Claude** (Opus/Sonnet) | XML tags for structure | `<instructions>…</instructions>`, `<context>…</context>` |
| **GPT** (4o, 5.x) | Markdown headings and bullet lists | `## Role`, `## Constraints`, `- Rule 1` |
| **Gemini** | Examples-first, clear sections | Lead with 1–2 examples, then rules |
| **Reasoning models** (o3, R1) | Minimal system, clear task | Avoid over-specifying reasoning steps |

---

### Few-Shot Prompting

Few-shot prompting inserts example `(user, assistant)` turns before the actual query. The model pattern-matches on format, style, and edge-case handling.

**3–5 examples is the sweet spot.** Returns diminish sharply after 5–8 examples because:

- Each example consumes context budget.
- Redundant examples add noise without new signal.
- The model may overfit to example specifics.

**Primary value in 2026: format alignment, NOT teaching reasoning.** Frontier models already know how to classify emails or parse dates. Few-shot teaches *your* JSON schema, confidence scale, enum values, and handling of ambiguous cases.

**Show diverse examples covering edge cases:**

- One typical positive case
- One typical negative case
- One ambiguous or borderline case
- One malformed or noisy input (if relevant)

Bad few-shot sets repeat the same pattern five times. Good sets vary the surface form while holding the output structure constant.

---

### Chain-of-Thought (CoT)

CoT elicits step-by-step reasoning before the final answer. The original "Let's think step by step" phrase (Kojima et al., 2022) unlocked latent reasoning in models that were not explicitly trained for it.

**For modern reasoning models (OpenAI o1/o3, DeepSeek-R1, QwQ): explicit CoT can be redundant or harmful.**

These models perform internal chain-of-thought during inference. Adding "think step by step" may:

- Increase latency and token cost without accuracy gain
- Cause over-verbalization or leaked internal reasoning you do not want in output
- Conflict with provider policies on reasoning visibility

**Use structured CoT instead of vague instructions.** Provide a template that separates reasoning from the answer:

```
<analysis>
- List relevant data points
- State assumptions
- Show calculation steps
</analysis>
<answer>
Final answer with confidence
</answer>
```

This gives you parseable output and prevents the answer from appearing before reasoning is complete.

**Reserve CoT for tasks with 3+ logical steps:** multi-hop QA, financial modeling, diagnostic workflows, code debugging. Skip it for simple classification, extraction, or creative writing unless you need auditable reasoning trails.

---

### Self-Consistency

Self-Consistency (Wang et al., 2022) improves CoT by sampling multiple reasoning paths and aggregating answers.

**Algorithm:**

1. Send the same prompt `N` times (typically `N = 5`) with `temperature > 0` (e.g., 0.7–1.0).
2. Parse the final answer from each response.
3. Take the **majority vote** (or weighted vote by confidence if available).

**Reported gain:** +17.9% on GSM8K when combined with CoT vs. single-path CoT.

**When to use:** high-stakes reasoning where latency and cost are acceptable (legal analysis drafts, complex calculations, medical triage support — always with human review).

**When to skip:** real-time chat, high-volume classification, tasks where temperature-0 single shot is already >99% accurate.

---

### Tree of Thoughts

Tree of Thoughts (Yao et al., 2023) treats reasoning as search over a tree of partial solutions.

**Algorithm sketch:**

1. **Propose** — generate `k` candidate next steps from the current state.
2. **Evaluate** — score each candidate (LLM-as-judge or heuristic).
3. **Select** — keep top candidates, discard or backtrack from dead ends.
4. **Repeat** until a terminal state (solution) is found or budget is exhausted.

**Best for:** strategic planning (project scheduling, game puzzles, constraint satisfaction), creative exploration with constraints, problems where greedy single-path CoT fails.

**Cost:** highest of all techniques — multiple LLM calls per tree level. Use only when accuracy justifies 5–20× baseline cost.

```
                    [Root: Problem]
                   /       |       \
              Step A    Step B    Step C
              /   \       |         |
           A1    A2      B1       (backtrack)
            |      \
         [Solution]  [Dead end → backtrack to A]
```

---

### Context Engineering (2026 Evolution)

Context engineering treats the prompt as one module in a larger assembly pipeline.

#### Modular Prompt Architecture — 6 Components

```
┌─────────────────────────────────────────────────────────────┐
│                    CONTEXT WINDOW (RAM)                      │
├─────────────────────────────────────────────────────────────┤
│  1. System Instructions    │ persona, rules, output format   │
│  2. Retrieved Documents    │ RAG chunks, ranked by relevance│
│  3. Tool Results           │ API responses, DB query results  │
│  4. Conversation Memory    │ summarized or sliding window   │
│  5. User Input               │ current task and attachments   │
│  6. Output Schema / Tools    │ JSON schema, function defs     │
└─────────────────────────────────────────────────────────────┘
```

Each component should be independently versioned, testable, and replaceable. A change to your RAG chunker should not require rewriting your system prompt.

#### Context Assembly: Minimum Necessary High-Signal Tokens

Principles:

- **Rank, don't dump** — retrieve top-k chunks; do not paste entire documents.
- **Compress history** — summarize older turns instead of sending full chat logs.
- **Deduplicate** — the same fact in system prompt + RAG + user message adds noise.
- **Place strategically** — models attend more strongly to the beginning and end of context ("lost in the middle" effect). Put critical constraints in the system prompt and restate key facts near the user message.

#### Context Rot

**Context rot** is the empirical degradation of model performance as input length grows, even when the model's advertised context window is much larger (128K–1M tokens).

Observed patterns in 2026 production systems:

- Instruction-following accuracy drops when total context exceeds ~3,000 tokens of mixed instructions + data (varies by model and task).
- Needle-in-haystack retrieval tests pass at 128K, but *reasoning over* long contexts often fails earlier.
- More tokens ≠ more accuracy. Irrelevant context actively hurts.

**Practical sweet spot:** **150–300 words** (~200–400 tokens) for system prompts. User messages and RAG should be task-scoped, not encyclopedic.

#### Context Engineering vs. Prompt Engineering

| Prompt Engineering | Context Engineering |
|---|---|
| Wording of instructions | What information enters the window |
| Few-shot examples | RAG retrieval, chunking, reranking |
| CoT templates | Tool orchestration, memory policies |
| Single-request focus | End-to-end pipeline design |

Both are required. A perfect system prompt cannot compensate for irrelevant RAG chunks or 50 turns of un summarized chat history.

---

## Code Example

The examples below use a chat `messages` array format compatible with OpenAI, Anthropic, and most providers. Set `OPENAI_API_KEY` (or equivalent) before running.

### Python — All Techniques

```python
"""
Prompt engineering techniques — runnable examples.
Requires: pip install openai
Usage: export OPENAI_API_KEY=sk-...
"""

import json
import os
import re
from collections import Counter
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
MODEL = "gpt-4o"


def chat(messages: list[dict], **kwargs) -> str:
    response = client.chat.completions.create(
        model=kwargs.get("model", MODEL),
        messages=messages,
        temperature=kwargs.get("temperature", 0.0),
    )
    return response.choices[0].message.content


# ---------------------------------------------------------------------------
# 1. Zero-shot
# ---------------------------------------------------------------------------
def classify_email_zero_shot(email: str) -> str:
    messages = [
        {
            "role": "user",
            "content": (
                "Classify this email as 'spam' or 'not_spam'. "
                "Reply with only one word.\n\n"
                f"Email: {email}"
            ),
        }
    ]
    return chat(messages)


# ---------------------------------------------------------------------------
# 2. Few-shot (3 examples for format compliance)
# ---------------------------------------------------------------------------
def classify_email_few_shot(email: str) -> dict:
    messages = [
        {
            "role": "system",
            "content": (
                "Classify emails. Respond with JSON: "
                '{"class": "spam|not_spam", "confidence": 0.0-1.0}'
            ),
        },
        {"role": "user", "content": "Win a free iPhone!"},
        {"role": "assistant", "content": '{"class": "spam", "confidence": 0.95}'},
        {"role": "user", "content": "Meeting at 3pm tomorrow"},
        {"role": "assistant", "content": '{"class": "not_spam", "confidence": 0.98}'},
        {
            "role": "user",
            "content": "Your account has been compromised, click here",
        },
        {"role": "assistant", "content": '{"class": "spam", "confidence": 0.92}'},
        {"role": "user", "content": email},
    ]
    raw = chat(messages)
    return json.loads(raw)


# ---------------------------------------------------------------------------
# 3. System prompt + role/persona
# ---------------------------------------------------------------------------
def financial_advisor_query(question: str, customer_data: str) -> str:
    messages = [
        {
            "role": "system",
            "content": """You are a senior financial analyst at a B2B SaaS company.

## Constraints
- Never invent revenue figures or payment dates
- Flag missing data explicitly
- Use conservative recommendations

## Output
Respond in markdown with: Summary, Risk Factors, Recommendation""",
        },
        {
            "role": "user",
            "content": f"Question: {question}\n\nCustomer data:\n{customer_data}",
        },
    ]
    return chat(messages)


# ---------------------------------------------------------------------------
# 4. Structured Chain-of-Thought (CoT)
# ---------------------------------------------------------------------------
def credit_decision_structured_cot(question: str, context: str) -> str:
    messages = [
        {
            "role": "system",
            "content": """You are a financial analyst. For each question:
<analysis>
- List relevant data points
- Identify assumptions
- Show calculation steps
</analysis>
<answer>
Final answer with confidence level (high/medium/low)
</answer>""",
        },
        {
            "role": "user",
            "content": f"{question}\n\nContext:\n{context}",
        },
    ]
    return chat(messages)


# ---------------------------------------------------------------------------
# 5. Structured output (JSON mode)
# ---------------------------------------------------------------------------
def extract_invoice_fields(invoice_text: str) -> dict:
    messages = [
        {
            "role": "system",
            "content": "Extract invoice fields. Return valid JSON only.",
        },
        {
            "role": "user",
            "content": (
                "Extract vendor_name, invoice_number, total_amount, due_date "
                f"from:\n\n{invoice_text}"
            ),
        },
    ]
    response = client.chat.completions.create(
        model=MODEL,
        messages=messages,
        response_format={"type": "json_object"},
        temperature=0.0,
    )
    return json.loads(response.choices[0].message.content)


# ---------------------------------------------------------------------------
# 6. Self-Consistency (majority vote over CoT paths)
# ---------------------------------------------------------------------------
def extract_answer(text: str) -> str:
    """Parse <answer> block or fall back to last line."""
    match = re.search(r"<answer>(.*?)</answer>", text, re.DOTALL | re.IGNORECASE)
    if match:
        return match.group(1).strip()
    return text.strip().split("\n")[-1]


def self_consistent_answer(question: str, n_samples: int = 5) -> str:
    messages = [
        {
            "role": "system",
            "content": """Solve the problem step by step in <analysis>, then give
the final numeric answer in <answer> on its own line.""",
        },
        {"role": "user", "content": question},
    ]
    answers = []
    for _ in range(n_samples):
        text = chat(messages, temperature=0.8)
        answers.append(extract_answer(text))
    vote = Counter(answers).most_common(1)[0][0]
    return vote


# ---------------------------------------------------------------------------
# 7. Tree of Thoughts (simplified: propose → evaluate → select)
# ---------------------------------------------------------------------------
def tree_of_thoughts_planning(problem: str, branches: int = 3) -> str:
    # Step 1: Propose branches
    propose_messages = [
        {
            "role": "system",
            "content": (
                f"Generate {branches} distinct high-level strategies to solve "
                "the problem. Number each strategy."
            ),
        },
        {"role": "user", "content": problem},
    ]
    strategies = chat(propose_messages, temperature=0.9)

    # Step 2: Evaluate each strategy
    eval_messages = [
        {
            "role": "system",
            "content": (
                "Score each strategy 1-10 on feasibility and impact. "
                "Recommend the best one with a concrete action plan."
            ),
        },
        {
            "role": "user",
            "content": f"Problem:\n{problem}\n\nStrategies:\n{strategies}",
        },
    ]
    return chat(eval_messages, temperature=0.2)


# ---------------------------------------------------------------------------
# Demo
# ---------------------------------------------------------------------------
if __name__ == "__main__":
    print("Zero-shot:", classify_email_zero_shot("Quarterly report attached."))
    print("Few-shot:", classify_email_few_shot("ACTUAL EMAIL TO CLASSIFY"))
    print(
        "Structured CoT:",
        credit_decision_structured_cot(
            "Should we extend Net-60 terms to this customer?",
            "Customer: Acme Corp. Avg days to pay: 72. Annual revenue: $2.1M.",
        )[:200],
        "...",
    )
    print("Self-consistency:", self_consistent_answer("Roger has 5 tennis balls. He buys 2 cans of 3. How many does he have?"))
```

### Elixir — Instructor (Structured Output + System Prompts)

Instructor maps Ecto schemas to LLM responses with automatic validation and retries. Ideal for structured output and few-shot-like format enforcement via schema.

```elixir
# mix.exs: {:instructor, "~> 0.1"}
defmodule MyApp.Prompts.EmailClassification do
  use Ecto.Schema
  use Instructor

  @llm_doc """
  Classify emails for a B2B SaaS company.
  Respond with class (spam or not_spam), confidence 0.0-1.0, and brief reason.
  """

  @primary_key false
  embedded_schema do
    field :class, Ecto.Enum, values: [:spam, :not_spam]
    field :confidence, :float
    field :reason, :string
  end

  @impl true
  def validate_changeset(changeset) do
    changeset
    |> Ecto.Changeset.validate_number(:confidence,
      greater_than_or_equal_to: 0.0,
      less_than_or_equal_to: 1.0
    )
  end
end

defmodule MyApp.Prompts.InstructorExamples do
  alias MyApp.Prompts.EmailClassification

  @system_prompt """
  You are an email security classifier for a B2B SaaS platform.
  Be conservative: marketing noise is not_spam; phishing is spam.
  """

  # Few-shot via message history + schema enforcement
  def classify(email_text) do
    Instructor.chat_completion(
      model: "gpt-4o",
      response_model: EmailClassification,
      max_retries: 3,
      messages: [
        %{role: "system", content: @system_prompt},
        # Few-shot turn 1
        %{role: "user", content: "Win a free iPhone!"},
        %{
          role: "assistant",
          content: ~s({"class": "spam", "confidence": 0.95, "reason": "Unsolicited prize"})
        },
        # Few-shot turn 2
        %{role: "user", content: "Meeting at 3pm tomorrow"},
        %{
          role: "assistant",
          content: ~s({"class": "not_spam", "confidence": 0.98, "reason": "Internal scheduling"})
        },
        # Actual query
        %{role: "user", content: "Classify: #{email_text}"}
      ]
    )
  end

  # Structured CoT via schema fields that force reasoning before answer
  defmodule CreditAnalysis do
    use Ecto.Schema
    use Instructor

    @llm_doc """
    Analyze credit term extension requests.
    Fill reasoning_steps BEFORE recommendation.
    """

    @primary_key false
    embedded_schema do
      field :reasoning_steps, {:array, :string}
      field :assumptions, {:array, :string}
      field :recommendation, :string
      field :confidence, Ecto.Enum, values: [:high, :medium, :low]
    end
  end

  def credit_decision(question, customer_data) do
    Instructor.chat_completion(
      model: "gpt-4o",
      response_model: CreditAnalysis,
      max_retries: 3,
      messages: [
        %{
          role: "system",
          content: """
          <role>Senior financial analyst</role>
          <constraints>Never invent figures. List assumptions explicitly.</constraints>
          """
        },
        %{
          role: "user",
          content: "Question: #{question}\n\nCustomer data:\n#{customer_data}"
        }
      ]
    )
  end
end
```

### Elixir — LangChain (System Prompts, CoT, Self-Consistency, Context Assembly)

LangChain Elixir (`langchain ~> 0.8`) composes messages, chains, and tools — a natural fit for context engineering pipelines.

```elixir
# mix.exs: {:langchain, "~> 0.8"}
defmodule MyApp.Prompts.LangChainExamples do
  alias LangChain.Chains.LLMChain
  alias LangChain.ChatModels.ChatOpenAI
  alias LangChain.Message

  @system_prompt """
  ## Role
  You are a concise technical writer for Elixir documentation.

  ## Constraints
  - Use code examples where helpful
  - Max 3 paragraphs unless asked otherwise
  - Never invent API functions
  """

  def zero_shot(user_query) do
    {:ok, chain} =
      %{llm: ChatOpenAI.new!(%{model: "gpt-4o", temperature: 0.0})}
      |> LLMChain.new!()
      |> LLMChain.add_message(Message.new_user!(user_query))
      |> LLMChain.run()

    chain.last_message.content
  end

  def with_system_and_user(user_query) do
    {:ok, chain} =
      %{llm: ChatOpenAI.new!(%{model: "gpt-4o", temperature: 0.0})}
      |> LLMChain.new!()
      |> LLMChain.add_message(Message.new_system!(@system_prompt))
      |> LLMChain.add_message(Message.new_user!(user_query))
      |> LLMChain.run()

    chain.last_message.content
  end

  def structured_cot(financial_question) do
    cot_system = """
    For each question respond with exactly two XML blocks:

    <analysis>
    - List relevant data points
    - Identify assumptions
    - Show calculation steps
    </analysis>
    <answer>
    Final answer with confidence level
    </answer>
    """

    {:ok, chain} =
      %{llm: ChatOpenAI.new!(%{model: "gpt-4o", temperature: 0.0})}
      |> LLMChain.new!()
      |> LLMChain.add_message(Message.new_system!(cot_system))
      |> LLMChain.add_message(Message.new_user!(financial_question))
      |> LLMChain.run()

    chain.last_message.content
  end

  # Self-consistency: run N times, majority vote on parsed <answer>
  def self_consistent(financial_question, n \\ 5) do
    llm = ChatOpenAI.new!(%{model: "gpt-4o", temperature: 0.8})

    answers =
      for _ <- 1..n do
        {:ok, chain} =
          %{llm: llm}
          |> LLMChain.new!()
          |> LLMChain.add_message(Message.new_system!(structured_cot_system()))
          |> LLMChain.add_message(Message.new_user!(financial_question))
          |> LLMChain.run()

        extract_xml_answer(chain.last_message.content)
      end

    answers
    |> Enum.frequencies()
    |> Enum.max_by(fn {_answer, count} -> count end)
    |> elem(0)
  end

  defp structured_cot_system do
    """
    Respond with <analysis> reasoning then <answer> final decision.
    Keep <answer> to one sentence.
    """
  end

  defp extract_xml_answer(content) do
    case Regex.run(~r/<answer>(.*?)<\/answer>/s, content) do
      [_, answer] -> String.trim(answer)
      _ -> String.trim(content)
    end
  end

  # Context engineering: assemble modular context before the LLM call
  defmodule ContextAssembler do
    @doc """
    Builds a message list from modular components.
    Each component is independently testable and versioned.
    """
    def build(opts) do
      system = Keyword.fetch!(opts, :system_instructions)
      rag_chunks = Keyword.get(opts, :rag_chunks, [])
      tool_results = Keyword.get(opts, :tool_results, [])
      user_input = Keyword.fetch!(opts, :user_input)

      rag_section =
        if rag_chunks == [] do
          ""
        else
          """
          <retrieved_context>
          #{Enum.map_join(rag_chunks, "\n\n", &wrap_chunk/1)}
          </retrieved_context>
          """
        end

      tool_section =
        if tool_results == [] do
          ""
        else
          """
          <tool_results>
          #{Jason.encode!(tool_results, pretty: true)}
          </tool_results>
          """
        end

      user_content = """
      #{rag_section}
      #{tool_section}
      <task>
      #{user_input}
      </task>
      """

      [
        Message.new_system!(system),
        Message.new_user!(String.trim(user_content))
      ]
    end

    defp wrap_chunk(chunk), do: "<document>\n#{chunk}\n</document>"
  end

  def context_engineered_query(user_input, rag_chunks) do
    messages =
      ContextAssembler.build(
        system_instructions: @system_prompt,
        rag_chunks: rag_chunks,
        user_input: user_input
      )

    {:ok, chain} =
      %{llm: ChatOpenAI.new!(%{model: "gpt-4o", temperature: 0.0})}
      |> LLMChain.new!()
      |> LLMChain.add_messages(messages)
      |> LLMChain.run()

    chain.last_message.content
  end
end
```

---

## Practical Application

### Decision Tree: Which Technique for Which Task?

```
START: What kind of task?
│
├─ Simple classification / extraction / translation?
│   └─ Zero-shot + Structured Output (JSON schema)
│       └─ Format errors? → Add 3-5 few-shot examples
│
├─ Need consistent persona / tone across sessions?
│   └─ System Prompt (150-300 words) + Role prompting
│
├─ Multi-step reasoning (math, logic, diagnostics)?
│   ├─ Using reasoning model (o3, R1)? → Minimal prompt, no explicit CoT
│   └─ Standard model? → Structured CoT template
│       └─ Accuracy still insufficient? → Self-Consistency (5 samples)
│
├─ Strategic planning / puzzles / search problems?
│   └─ Tree of Thoughts (if budget allows 5-20× cost)
│
├─ Production pipeline integrating with code/DB/UI?
│   └─ Structured Output (JSON mode / Instructor / response_format)
│       + System prompt for constraints
│       + Context assembly (RAG, tools) — NOT one giant prompt
│
└─ Long-running agent / chat product?
    └─ Context Engineering full stack:
        modular components, memory summarization, RAG, tool results
```

### Production Checklist

**Version control**

- [ ] Store prompts in git (YAML, Markdown, or `.ex` modules) — not hardcoded strings scattered in code
- [ ] Tag prompt versions (`prompts/email_classifier/v2.1.yaml`)
- [ ] Link prompt version to eval results and deployment

**Testing**

- [ ] Maintain a golden dataset (50–200 examples) with expected outputs
- [ ] Run regression evals on every prompt change (exact match, JSON schema validation, LLM-as-judge)
- [ ] Track metrics: accuracy, format compliance, latency p95, cost per request
- [ ] Test edge cases explicitly (empty input, adversarial input, non-English)

**CI/CD for prompts**

- [ ] PR checks run eval suite against changed prompts
- [ ] Block merge if accuracy drops below threshold (e.g., >2% regression)
- [ ] Canary deploy: 5% traffic on new prompt version before full rollout
- [ ] Log prompt version + model + token counts on every request for debugging

**Context pipeline**

- [ ] Measure total context tokens per request; alert if p95 > 4,000
- [ ] Monitor context rot: does accuracy drop as conversation length grows?
- [ ] Summarize or truncate conversation history after N turns
- [ ] RAG: rerank chunks; do not inject low-relevance results

**Security**

- [ ] Separate untrusted user content from system instructions (delimiter tags, role separation)
- [ ] Never put secrets in prompts
- [ ] Red-team for prompt injection on any user-controlled input field

### Anti-Patterns to Avoid

| Anti-pattern | Why it fails | Do instead |
|---|---|---|
| **ALL-CAPS emphasis** (`YOU MUST ALWAYS…`) | Models do not respond better to shouting; adds noise | Clear, specific constraints in normal case |
| **"YOU MUST" stacking** | 20 "must" rules → model follows ~5 unpredictably | Prioritize 3–5 hard rules; move rest to validation code |
| **Overly long system prompts** (2,000+ tokens) | Instruction dilution, context rot, cost | 150–300 words; split task-specific detail to user message |
| **Few-shot for reasoning** | Wastes tokens; models already know how to reason | Few-shot for *format* only |
| **Explicit CoT on reasoning models** | Redundant cost/latency; may degrade output | Let the model reason internally; constrain output format only |
| **Dumping full documents into context** | Context rot; lost-in-the-middle failures | RAG with chunking, reranking, top-k retrieval |
| **Duplicating instructions** in system + user + RAG | Conflicting versions; wasted tokens | Single source of truth per rule |
| **No evals** ("it looks fine") | Silent regressions on every model update | Automated golden-set regression tests |
| **Temperature > 0 for extraction** | Non-deterministic JSON; parse failures | `temperature: 0.0` for structured tasks |
| **Trusting LLM arithmetic** | Models hallucinate calculations | Code execution or calculator tool for numbers |

### Model-Specific Tips (2026)

**Claude (Anthropic)**

- Use XML tags for sections: `<instructions>`, `<context>`, `<example>`, `<output_format>`
- Place long documents in `<document>` tags; instructions outside
- Claude follows nuanced constraints well — prefer precise language over emphatic language
- Cache system prompts server-side (`cache_control`) for cost savings on repeated calls

**GPT (OpenAI)**

- Markdown structure works well: headings, bullet lists, numbered steps
- Use `response_format: { type: "json_object" }` or strict JSON schema for structured output
- Reasoning models (o3): use developer message for instructions; avoid "think step by step"
- Keep system + developer messages focused; put variable data in user message

**Gemini (Google)**

- Lead with 1–2 concrete examples, then state rules
- Gemini 2.x handles long context well for retrieval but still benefits from focused instructions
- Use explicit output format at the end of the prompt (recency bias)

**Open-weight (Llama 4, Qwen 3, Mistral)**

- Often benefit from explicit CoT and few-shot more than frontier closed models
- Use chat templates exactly as documented (special tokens matter)
- Smaller models: shorter system prompts, more examples, lower temperature

---

## Related Topics

- [What is an LLM?](../llm_fundamentals/what_is_an_llm.md) — transformer architecture, tokenization, inference fundamentals
- [RAG (Retrieval-Augmented Generation)](../rag/) — chunking, embeddings, vector search for context assembly
- [Agents](../agents/) — tool use, MCP servers, multi-step autonomous workflows
- [Elixir LLM Implementation](../code_examples/elixir_llm_implementation.md) — Instructor schemas, streaming, production CV extraction pipeline
- [Elixir AI Zero to Hero](../api_integration/elixir_ai_zero_to_hero.md) — LangChain, Instructor, multi-provider routing

### Key Papers and References

- Kojima et al. (2022) — "Large Language Models are Zero-Shot Reasoners" (CoT prompting)
- Wei et al. (2022) — "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models"
- Wang et al. (2022) — "Self-Consistency Improves Chain of Thought Reasoning in Language Models"
- Yao et al. (2023) — "Tree of Thoughts: Deliberate Problem Solving with Large Language Models"
- Liu et al. (2023) — "Lost in the Middle: How Language Models Use Long Contexts"

### External Tools (2026)

- [LangChain Elixir](https://hexdocs.pm/langchain) — chains, agents, tool calling on the BEAM
- [Instructor (Elixir)](https://hexdocs.pm/instructor) — Ecto schema ↔ LLM structured outputs
- [OpenAI Prompt Engineering Guide](https://platform.openai.com/docs/guides/prompt-engineering) — provider-specific best practices
- [Anthropic Prompt Engineering](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) — Claude-specific patterns
