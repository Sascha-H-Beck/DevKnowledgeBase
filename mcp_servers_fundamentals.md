# Zero to Hero: MCP Servers — Part 1: Fundamentals & Building

## Overview

The Model Context Protocol (MCP) is an open standard (created by Anthropic, November 2024) that provides a universal way for AI applications to connect to external tools, data sources, and workflows. Think of MCP as **"USB-C for AI"** — instead of building custom integrations for every model-tool combination (the N×M problem), you build one MCP server and any compatible AI client can use it (reducing to N+M).

MCP is supported by Claude, ChatGPT, Cursor, VS Code, and many other AI clients. As of mid-2026, the official SDKs for Python and TypeScript have 23K+ and 12K+ GitHub stars respectively, with 87K+ stars on the community servers repository.

## Key Concepts

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        MCP Host                              │
│  (AI Application: Claude, Cursor, ChatGPT, Custom App)       │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │ MCP      │  │ MCP      │  │ MCP      │                   │
│  │ Client 1 │  │ Client 2 │  │ Client 3 │  ← one per server │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                   │
└───────┼──────────────┼──────────────┼────────────────────────┘
        │              │              │
   JSON-RPC       JSON-RPC       JSON-RPC
   (stdio)        (HTTP)         (HTTP)
        │              │              │
   ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐
   │ MCP      │  │ MCP      │  │ MCP      │
   │ Server A │  │ Server B │  │ Server C │
   │ (local)  │  │ (remote) │  │ (remote) │
   │          │  │          │  │          │
   │ Database │  │ GitHub   │  │ Slack    │
   └──────────┘  └──────────┘  └──────────┘
```

| Component | Role |
|---|---|
| **MCP Host** | The AI application (Claude Desktop, Cursor, custom app) that coordinates clients |
| **MCP Client** | A component within the host that maintains a connection to one MCP server |
| **MCP Server** | A program that exposes capabilities (tools, resources, prompts) to clients |

### The Three Primitives

| Primitive | Analogy | Who Controls | Side Effects | Example |
|---|---|---|---|---|
| **Tools** | POST endpoint | Model-invoked | Yes (writes, API calls) | `create_invoice`, `query_database` |
| **Resources** | GET endpoint | User/app-controlled | No (read-only) | `file://config.json`, `db://schema` |
| **Prompts** | Reusable templates | User-selected | No | "Analyze these transactions", "Summarize report" |

### Transport Options

| Transport | Use Case | How It Works |
|---|---|---|
| **stdio** | Local servers (same machine) | Client spawns server as subprocess, communicates via stdin/stdout |
| **Streamable HTTP** | Remote/cloud servers | Client sends HTTP POST, server responds + optional SSE streaming |

### Communication Protocol

MCP uses JSON-RPC 2.0 for all messages:

```json
// Client → Server: Tool invocation
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "create_invoice",
    "arguments": {
      "customer": "Acme Corp",
      "amount": 1500.00,
      "currency": "EUR"
    }
  }
}

// Server → Client: Response
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Invoice INV-2026-0042 created for Acme Corp: €1,500.00"
      }
    ]
  }
}
```

### Discovery Flow

```
Client                          Server
  │                                │
  │  initialize                    │
  │───────────────────────────────▶│
  │◀───────────────────────────────│  server info + capabilities
  │                                │
  │  tools/list                    │
  │───────────────────────────────▶│
  │◀───────────────────────────────│  [{name, description, inputSchema}]
  │                                │
  │  resources/list                │
  │───────────────────────────────▶│
  │◀───────────────────────────────│  [{uri, name, mimeType}]
  │                                │
  │  prompts/list                  │
  │───────────────────────────────▶│
  │◀───────────────────────────────│  [{name, description, arguments}]
  │                                │
  │  tools/call {name, arguments}  │
  │───────────────────────────────▶│
  │◀───────────────────────────────│  {content: [{type, text}]}
  │                                │
```

## How It Works

### Building an MCP Server in Python

Install the official SDK:

```bash
pip install mcp
```

#### Complete Python MCP Server

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("AccountingHelper")


# --- TOOLS (model-invoked, side effects allowed) ---

@mcp.tool()
def create_invoice(
    customer_name: str,
    amount: float,
    currency: str = "EUR",
    due_days: int = 30
) -> str:
    """Create a new invoice for a customer (creditor function)."""
    from datetime import datetime, timedelta

    invoice_id = f"INV-{datetime.now().strftime('%Y%m%d-%H%M%S')}"
    due_date = (datetime.now() + timedelta(days=due_days)).strftime("%Y-%m-%d")

    # In production: insert into your database here
    return (
        f"Invoice {invoice_id} created:\n"
        f"  Customer: {customer_name}\n"
        f"  Amount: {currency} {amount:,.2f}\n"
        f"  Due: {due_date}"
    )


@mcp.tool()
def record_payment(
    invoice_id: str,
    amount: float,
    payment_method: str = "bank_transfer"
) -> str:
    """Record a payment received against an invoice (debitor function)."""
    return (
        f"Payment recorded:\n"
        f"  Invoice: {invoice_id}\n"
        f"  Amount: {amount:,.2f}\n"
        f"  Method: {payment_method}\n"
        f"  Status: Applied"
    )


@mcp.tool()
def get_outstanding_balance(customer_name: str) -> str:
    """Get the outstanding balance for a customer."""
    # In production: query your database
    return f"Outstanding balance for {customer_name}: EUR 4,250.00 (3 open invoices)"


@mcp.tool()
def generate_aging_report() -> str:
    """Generate an accounts receivable aging report."""
    return (
        "Aging Report (as of today):\n"
        "  Current (0-30 days):  EUR 12,500.00  (5 invoices)\n"
        "  31-60 days:           EUR  4,200.00  (2 invoices)\n"
        "  61-90 days:           EUR  1,800.00  (1 invoice)\n"
        "  90+ days:             EUR    950.00  (1 invoice)\n"
        "  Total Outstanding:    EUR 19,450.00"
    )


# --- RESOURCES (read-only data) ---

@mcp.resource("accounting://chart-of-accounts")
def get_chart_of_accounts() -> str:
    """The company's chart of accounts."""
    return """
Chart of Accounts:
  1000 - Assets
    1100 - Cash and Bank
    1200 - Accounts Receivable
    1300 - Inventory
  2000 - Liabilities
    2100 - Accounts Payable
    2200 - Tax Liabilities
  3000 - Equity
    3100 - Share Capital
    3200 - Retained Earnings
  4000 - Revenue
    4100 - Sales Revenue
    4200 - Service Revenue
  5000 - Expenses
    5100 - Cost of Goods Sold
    5200 - Operating Expenses
    5300 - Payroll
"""


@mcp.resource("accounting://payment-terms")
def get_payment_terms() -> str:
    """Standard payment terms and policies."""
    return """
Payment Terms:
  - Net 30: Standard for most customers
  - Net 15: For new customers (first 6 months)
  - Net 60: For key accounts (approved by CFO)
  - 2/10 Net 30: 2% discount if paid within 10 days
Dunning Policy:
  - Day 31: Friendly reminder email
  - Day 45: Second reminder with late fee warning
  - Day 60: Late fee applied (1.5% per month)
  - Day 90: Account sent to collections
"""


# --- PROMPTS (reusable templates) ---

@mcp.prompt()
def analyze_customer_account(customer_name: str) -> str:
    """Analyze a customer's account status and recommend actions."""
    return f"""
Please analyze the account for customer "{customer_name}":

1. Check their outstanding balance using the get_outstanding_balance tool
2. Review the payment terms policy from the accounting://payment-terms resource
3. Generate an aging report to see their position relative to other accounts
4. Recommend specific actions:
   - If overdue: suggest dunning step
   - If current: note good standing
   - If large balance: flag for credit review

Provide a structured summary with recommended next steps.
"""


@mcp.prompt()
def monthly_close_checklist() -> str:
    """Guide through the monthly accounting close process."""
    return """
Monthly Close Checklist:

1. Review all open invoices — use generate_aging_report
2. Match payments to invoices — check for unapplied payments
3. Verify accounts receivable balances
4. Review accounts payable for missed bills
5. Reconcile bank statements
6. Post adjusting journal entries
7. Generate trial balance
8. Review P&L and Balance Sheet

For each step, use the available tools to gather data and flag any discrepancies.
"""


if __name__ == "__main__":
    mcp.run(transport="stdio")
```

#### Running and Testing

```bash
# Run with stdio (for Claude Desktop, Cursor)
python accounting_mcp.py

# Run with HTTP (for remote access)
python -c "
from accounting_mcp import mcp
mcp.run(transport='streamable-http', host='0.0.0.0', port=8080)
"

# Test with MCP Inspector
npx @modelcontextprotocol/inspector python accounting_mcp.py
```

#### Claude Desktop Configuration

```json
{
  "mcpServers": {
    "accounting": {
      "command": "python",
      "args": ["/path/to/accounting_mcp.py"]
    }
  }
}
```

#### Cursor Configuration

```json
{
  "mcpServers": {
    "accounting": {
      "command": "python",
      "args": ["/path/to/accounting_mcp.py"]
    }
  }
}
```

### Building an MCP Server in Elixir

Several Elixir MCP libraries exist as of June 2026:

| Library | Hex Package | Approach |
|---|---|---|
| **Anubis MCP** | `anubis_mcp` ~> 1.0 | Component-based, Phoenix/Plug integration, Streamable HTTP |
| **Hermes MCP** | `hermes_mcp` ~> 0.14 | Predecessor to Anubis, similar API |
| **ConduitMCP** | `conduit_mcp` ~> 0.9 | Three modes (DSL, manual, component), full spec compliance |
| **MCPKit** | `mcp_kit` ~> 0.2 | Phoenix-native, policy-aware runtime |
| **NexusMCP** | `nexus_mcp` ~> 0.2 | Per-session GenServer, lightweight DSL |

#### Complete Elixir MCP Server (using Anubis MCP)

```elixir
# mix.exs
defp deps do
  [
    {:anubis_mcp, "~> 1.0"},
    {:plug_cowboy, "~> 2.7"},
    {:jason, "~> 1.4"}
  ]
end
```

##### Tool Components

```elixir
defmodule MyApp.MCP.Tools.CreateInvoice do
  use Anubis.Server.Component, type: :tool

  @impl true
  def name, do: "create_invoice"

  @impl true
  def description, do: "Create a new invoice for a customer (creditor function)"

  schema do
    field :customer_name, :string, required: true, description: "Customer or company name"
    field :amount, :number, required: true, description: "Invoice amount"
    field :currency, :string, description: "Currency code (default: EUR)"
    field :due_days, :integer, description: "Days until due (default: 30)"
  end

  @impl true
  def execute(%{customer_name: name, amount: amount} = params, _frame) do
    currency = Map.get(params, :currency, "EUR")
    due_days = Map.get(params, :due_days, 30)

    invoice_id = "INV-#{Date.utc_today() |> Date.to_iso8601()}-#{System.unique_integer([:positive])}"
    due_date = Date.utc_today() |> Date.add(due_days) |> Date.to_iso8601()

    # In production: insert into your Ecto repo here
    {:ok, """
    Invoice #{invoice_id} created:
      Customer: #{name}
      Amount: #{currency} #{:erlang.float_to_binary(amount, decimals: 2)}
      Due: #{due_date}
    """}
  end
end
```

```elixir
defmodule MyApp.MCP.Tools.RecordPayment do
  use Anubis.Server.Component, type: :tool

  @impl true
  def name, do: "record_payment"

  @impl true
  def description, do: "Record a payment received against an invoice (debitor function)"

  schema do
    field :invoice_id, :string, required: true, description: "Invoice ID (e.g., INV-2026-001)"
    field :amount, :number, required: true, description: "Payment amount"
    field :payment_method, :string, description: "Payment method (default: bank_transfer)"
  end

  @impl true
  def execute(%{invoice_id: inv_id, amount: amount} = params, _frame) do
    method = Map.get(params, :payment_method, "bank_transfer")

    # In production: update invoice status in your database
    {:ok, """
    Payment recorded:
      Invoice: #{inv_id}
      Amount: #{:erlang.float_to_binary(amount, decimals: 2)}
      Method: #{method}
      Status: Applied
    """}
  end
end
```

```elixir
defmodule MyApp.MCP.Tools.AgingReport do
  use Anubis.Server.Component, type: :tool

  @impl true
  def name, do: "generate_aging_report"

  @impl true
  def description, do: "Generate an accounts receivable aging report"

  schema do
  end

  @impl true
  def execute(_params, _frame) do
    # In production: query from your database
    {:ok, """
    Aging Report (as of #{Date.utc_today() |> Date.to_iso8601()}):
      Current (0-30 days):  EUR 12,500.00  (5 invoices)
      31-60 days:           EUR  4,200.00  (2 invoices)
      61-90 days:           EUR  1,800.00  (1 invoice)
      90+ days:             EUR    950.00  (1 invoice)
      Total Outstanding:    EUR 19,450.00
    """}
  end
end
```

##### Server Definition

```elixir
defmodule MyApp.MCPServer do
  use Anubis.Server,
    name: "Accounting Helper",
    version: "1.0.0",
    capabilities: [:tools, :resources, :prompts]

  # Register tool components
  component MyApp.MCP.Tools.CreateInvoice
  component MyApp.MCP.Tools.RecordPayment
  component MyApp.MCP.Tools.AgingReport

  @impl true
  def init(_client_info, frame) do
    # Register resources
    frame = register_resource(frame, "accounting://chart-of-accounts",
      name: "Chart of Accounts",
      description: "The company's chart of accounts",
      mime_type: "text/plain"
    )

    frame = register_resource(frame, "accounting://payment-terms",
      name: "Payment Terms",
      description: "Standard payment terms and dunning policy",
      mime_type: "text/plain"
    )

    # Register prompts
    frame = register_prompt(frame, "analyze_customer_account",
      description: "Analyze a customer's account status and recommend actions",
      arguments: [
        %{name: "customer_name", description: "Customer to analyze", required: true}
      ]
    )

    frame = register_prompt(frame, "monthly_close_checklist",
      description: "Guide through the monthly accounting close process"
    )

    {:ok, frame}
  end

  @impl true
  def handle_read_resource("accounting://chart-of-accounts", _frame) do
    {:ok, """
    Chart of Accounts:
      1000 - Assets
        1100 - Cash and Bank
        1200 - Accounts Receivable
        1300 - Inventory
      2000 - Liabilities
        2100 - Accounts Payable
        2200 - Tax Liabilities
      3000 - Equity
      4000 - Revenue
      5000 - Expenses
    """}
  end

  @impl true
  def handle_read_resource("accounting://payment-terms", _frame) do
    {:ok, """
    Payment Terms:
      - Net 30: Standard for most customers
      - Net 15: For new customers (first 6 months)
      - Net 60: Key accounts (CFO approval required)
      - 2/10 Net 30: 2% discount if paid within 10 days
    Dunning Policy:
      - Day 31: Friendly reminder email
      - Day 45: Second reminder with late fee warning
      - Day 60: Late fee applied (1.5% per month)
      - Day 90: Account sent to collections
    """}
  end

  @impl true
  def handle_get_prompt("analyze_customer_account", %{"customer_name" => name}, _frame) do
    {:ok, [
      %{role: "user", content: %{type: "text", text: """
      Please analyze the account for customer "#{name}":

      1. Check their outstanding balance using the get_outstanding_balance tool
      2. Review the payment terms policy from the accounting://payment-terms resource
      3. Generate an aging report to see their position
      4. Recommend specific actions (dunning, credit review, etc.)
      """}}
    ]}
  end

  @impl true
  def handle_get_prompt("monthly_close_checklist", _args, _frame) do
    {:ok, [
      %{role: "user", content: %{type: "text", text: """
      Monthly Close Checklist:
      1. Review all open invoices — use generate_aging_report
      2. Match payments to invoices
      3. Verify accounts receivable balances
      4. Review accounts payable
      5. Reconcile bank statements
      6. Post adjusting journal entries
      7. Generate trial balance
      8. Review P&L and Balance Sheet
      """}}
    ]}
  end
end
```

##### Application Supervisor and Router

```elixir
# lib/my_app/application.ex
defmodule MyApp.Application do
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      {MyApp.MCPServer, transport: :streamable_http},
      {Plug.Cowboy, scheme: :http, plug: MyApp.Router, options: [port: 4000]}
    ]

    Supervisor.start_link(children, strategy: :one_for_one)
  end
end

# lib/my_app/router.ex
defmodule MyApp.Router do
  use Plug.Router

  plug :match
  plug :dispatch

  forward "/mcp", to: Anubis.Server.Transport.StreamableHTTP.Plug,
    init_opts: [server: MyApp.MCPServer]

  match _ do
    send_resp(conn, 404, "Not found")
  end
end
```

#### Running the Elixir Server

```bash
# Start the server
mix run --no-halt

# The MCP endpoint is now available at http://localhost:4000/mcp

# Test with MCP Inspector
npx @modelcontextprotocol/inspector --url http://localhost:4000/mcp
```

## Practical Application

### When to Build an MCP Server

| Scenario | MCP Server? | Why |
|---|---|---|
| Internal tool for one AI app | Maybe | Consider a direct API integration first |
| Tool used by multiple AI clients | **Yes** | Build once, connect everywhere |
| Department-specific workflows | **Yes** | Encapsulate domain logic, expose via prompts |
| Database access for AI | **Yes** | Controlled access with schema validation |
| File system operations | Use existing | Community servers already exist |

### Testing Your MCP Server

The official MCP Inspector provides a visual testing interface:

```bash
# Local server (stdio)
npx @modelcontextprotocol/inspector python your_server.py

# Remote server (HTTP)
npx @modelcontextprotocol/inspector --url http://localhost:8080/mcp
```

The Inspector lets you:
- List all tools, resources, and prompts
- Call tools with custom arguments
- Read resources
- View JSON-RPC messages for debugging

## Related Topics

- [MCP Servers Part 2: Implementation, Deployment & Accounting Case Study](mcp_servers_implementation.md)
- [What is an LLM?](../llm_fundamentals/what_is_an_llm.md) — foundational LLM concepts
- [Elixir AI Zero to Hero](../api_integration/elixir_ai_zero_to_hero.md) — full Elixir AI stack
- [Authentication](../authentication/zero_to_hero_authentication.md) — OAuth2 for securing MCP servers

### References

- [MCP Specification](https://modelcontextprotocol.io/)
- [Official Python SDK](https://github.com/modelcontextprotocol/python-sdk) — 23K+ stars
- [Official TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) — 12K+ stars
- [Anubis MCP (Elixir)](https://github.com/zoedsoupe/anubis-mcp)
- [ConduitMCP (Elixir)](https://github.com/nyo16/conduit_mcp)
- [MCP Inspector](https://github.com/modelcontextprotocol/inspector)
- [Community MCP Servers](https://github.com/modelcontextprotocol/servers) — 87K+ stars
