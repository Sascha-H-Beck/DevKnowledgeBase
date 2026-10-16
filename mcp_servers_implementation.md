# Zero to Hero: MCP Servers — Part 2: Implementation, Deployment & Accounting Case Study

## Overview

This document extends [Part 1 (Fundamentals & Building)](mcp_servers_fundamentals.md) with production deployment patterns, security considerations, and a comprehensive real-world case study: embedding an MCP server into an Accounting Department to manage creditor and debitor functions.

## Key Concepts

### Production Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                    Accounting Department AI Stack                  │
│                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │ Claude        │  │ Cursor       │  │ Custom       │  AI Clients │
│  │ Desktop       │  │ IDE          │  │ Phoenix App  │             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│         │                  │                  │                     │
│         └──────────────────┼──────────────────┘                    │
│                            │                                       │
│                     ┌──────▼──────┐                                │
│                     │  Reverse    │  nginx / Caddy                 │
│                     │  Proxy      │  TLS termination               │
│                     └──────┬──────┘                                │
│                            │                                       │
│              ┌─────────────▼─────────────┐                         │
│              │  MCP Server               │                         │
│              │  (Elixir/Phoenix)         │                         │
│              │                           │                         │
│              │  Tools:                   │                         │
│              │    create_invoice         │                         │
│              │    record_payment         │                         │
│              │    approve_payment        │                         │
│              │    aging_report           │                         │
│              │    vendor_lookup          │                         │
│              │    reconcile_bank         │                         │
│              │                           │                         │
│              │  Resources:               │                         │
│              │    chart_of_accounts      │                         │
│              │    payment_terms          │                         │
│              │    tax_rates              │                         │
│              │    vendor_list            │                         │
│              │                           │                         │
│              │  Prompts:                 │                         │
│              │    analyze_customer       │                         │
│              │    monthly_close          │                         │
│              │    dispute_resolution     │                         │
│              └─────────┬─────────────────┘                         │
│                        │                                           │
│              ┌─────────▼─────────────┐                             │
│              │  PostgreSQL           │                              │
│              │  (Invoices, Payments, │                              │
│              │   Vendors, Accounts)  │                              │
│              └───────────────────────┘                              │
└────────────────────────────────────────────────────────────────────┘
```

### Deployment Options

| Strategy | Transport | Pros | Cons |
|---|---|---|---|
| **Local subprocess** | stdio | Zero network config, fast | One machine only, one client |
| **Docker container** | Streamable HTTP | Portable, scalable, CI/CD friendly | Network setup needed |
| **Phoenix embedded** | Streamable HTTP | Full BEAM ecosystem, supervision, LiveView dashboard | Elixir knowledge required |
| **Serverless (Lambda/Cloud Run)** | Streamable HTTP | Auto-scaling, pay-per-use | Cold starts, stateless constraints |

### Security Considerations

| Layer | Implementation |
|---|---|
| **Authentication** | OAuth 2.1 tokens, API keys, or mTLS for server-to-server |
| **Authorization** | Role-based access — e.g., AP clerk can `record_payment` but not `approve_payment` |
| **Input validation** | JSON Schema validation on all tool parameters (built into MCP spec) |
| **Rate limiting** | Limit tool calls per user/session to prevent abuse |
| **Audit logging** | Log every tool invocation with user, timestamp, parameters, and result |
| **Data encryption** | TLS in transit, encrypt sensitive fields at rest |
| **Origin validation** | Restrict which domains can connect (CORS / origin allowlist) |

## How It Works

### Full Accounting Department MCP Server — Elixir + Ecto + PostgreSQL

This is a production-grade implementation connecting the MCP server to real database tables.

#### Database Schema

```elixir
# priv/repo/migrations/001_create_accounting_tables.exs
defmodule MyApp.Repo.Migrations.CreateAccountingTables do
  use Ecto.Migration

  def change do
    create table(:vendors) do
      add :name, :string, null: false
      add :tax_id, :string
      add :payment_terms, :string, default: "net_30"
      add :contact_email, :string
      add :bank_iban, :string
      add :active, :boolean, default: true
      timestamps()
    end

    create table(:invoices) do
      add :invoice_number, :string, null: false
      add :vendor_id, references(:vendors), null: false
      add :direction, :string, null: false  # "receivable" or "payable"
      add :amount, :decimal, null: false
      add :currency, :string, default: "EUR"
      add :tax_amount, :decimal, default: 0
      add :status, :string, default: "open"  # open, partial, paid, overdue, disputed
      add :issue_date, :date, null: false
      add :due_date, :date, null: false
      add :paid_date, :date
      add :description, :text
      add :gl_account, :string
      timestamps()
    end

    create unique_index(:invoices, [:invoice_number])

    create table(:payments) do
      add :invoice_id, references(:invoices), null: false
      add :amount, :decimal, null: false
      add :payment_date, :date, null: false
      add :payment_method, :string, default: "bank_transfer"
      add :reference, :string
      add :approved_by, :string
      add :notes, :text
      timestamps()
    end

    create table(:audit_log) do
      add :action, :string, null: false
      add :entity_type, :string
      add :entity_id, :integer
      add :user, :string
      add :details, :map, default: %{}
      add :mcp_session_id, :string
      timestamps(updated_at: false)
    end
  end
end
```

#### Ecto Schemas

```elixir
defmodule MyApp.Accounting.Invoice do
  use Ecto.Schema
  import Ecto.Changeset

  schema "invoices" do
    field :invoice_number, :string
    field :direction, :string
    field :amount, :decimal
    field :currency, :string, default: "EUR"
    field :tax_amount, :decimal, default: Decimal.new(0)
    field :status, :string, default: "open"
    field :issue_date, :date
    field :due_date, :date
    field :paid_date, :date
    field :description, :string
    field :gl_account, :string

    belongs_to :vendor, MyApp.Accounting.Vendor
    has_many :payments, MyApp.Accounting.Payment

    timestamps()
  end

  def changeset(invoice, attrs) do
    invoice
    |> cast(attrs, [:invoice_number, :vendor_id, :direction, :amount, :currency,
                    :tax_amount, :status, :issue_date, :due_date, :paid_date,
                    :description, :gl_account])
    |> validate_required([:invoice_number, :vendor_id, :direction, :amount,
                          :issue_date, :due_date])
    |> validate_inclusion(:direction, ["receivable", "payable"])
    |> validate_inclusion(:status, ["open", "partial", "paid", "overdue", "disputed"])
    |> validate_number(:amount, greater_than: 0)
    |> unique_constraint(:invoice_number)
    |> foreign_key_constraint(:vendor_id)
  end
end
```

```elixir
defmodule MyApp.Accounting.Payment do
  use Ecto.Schema
  import Ecto.Changeset

  schema "payments" do
    field :amount, :decimal
    field :payment_date, :date
    field :payment_method, :string, default: "bank_transfer"
    field :reference, :string
    field :approved_by, :string
    field :notes, :string

    belongs_to :invoice, MyApp.Accounting.Invoice

    timestamps()
  end

  def changeset(payment, attrs) do
    payment
    |> cast(attrs, [:invoice_id, :amount, :payment_date, :payment_method,
                    :reference, :approved_by, :notes])
    |> validate_required([:invoice_id, :amount, :payment_date])
    |> validate_number(:amount, greater_than: 0)
    |> foreign_key_constraint(:invoice_id)
  end
end
```

#### Context Module (Business Logic)

```elixir
defmodule MyApp.Accounting do
  import Ecto.Query
  alias MyApp.Repo
  alias MyApp.Accounting.{Invoice, Payment, Vendor}

  # --- Creditor Functions (Accounts Receivable) ---

  def create_receivable_invoice(attrs) do
    %Invoice{}
    |> Invoice.changeset(Map.put(attrs, :direction, "receivable"))
    |> Repo.insert()
  end

  def get_customer_balance(vendor_id) do
    from(i in Invoice,
      where: i.vendor_id == ^vendor_id and i.direction == "receivable"
             and i.status in ["open", "partial", "overdue"],
      select: %{
        count: count(i.id),
        total: sum(i.amount),
        overdue: sum(fragment("CASE WHEN due_date < CURRENT_DATE THEN amount ELSE 0 END"))
      }
    )
    |> Repo.one()
  end

  def aging_report(direction) do
    today = Date.utc_today()

    from(i in Invoice,
      where: i.direction == ^direction and i.status in ["open", "partial", "overdue"],
      select: %{
        current: sum(fragment("CASE WHEN due_date >= ? THEN amount ELSE 0 END", ^today)),
        days_31_60: sum(fragment(
          "CASE WHEN due_date < ? AND due_date >= ? THEN amount ELSE 0 END",
          ^today, ^Date.add(today, -30)
        )),
        days_61_90: sum(fragment(
          "CASE WHEN due_date < ? AND due_date >= ? THEN amount ELSE 0 END",
          ^Date.add(today, -30), ^Date.add(today, -60)
        )),
        over_90: sum(fragment("CASE WHEN due_date < ? THEN amount ELSE 0 END",
          ^Date.add(today, -60)
        )),
        total: sum(i.amount),
        invoice_count: count(i.id)
      }
    )
    |> Repo.one()
  end

  # --- Debitor Functions (Accounts Payable) ---

  def create_payable_invoice(attrs) do
    %Invoice{}
    |> Invoice.changeset(Map.put(attrs, :direction, "payable"))
    |> Repo.insert()
  end

  def record_payment(invoice_id, attrs) do
    Repo.transaction(fn ->
      invoice = Repo.get!(Invoice, invoice_id)

      payment =
        %Payment{}
        |> Payment.changeset(Map.put(attrs, :invoice_id, invoice_id))
        |> Repo.insert!()

      total_paid =
        from(p in Payment, where: p.invoice_id == ^invoice_id, select: sum(p.amount))
        |> Repo.one()
        |> Kernel.||(Decimal.new(0))

      new_status =
        cond do
          Decimal.compare(total_paid, invoice.amount) in [:eq, :gt] -> "paid"
          Decimal.compare(total_paid, Decimal.new(0)) == :gt -> "partial"
          true -> invoice.status
        end

      invoice
      |> Invoice.changeset(%{
        status: new_status,
        paid_date: if(new_status == "paid", do: Date.utc_today())
      })
      |> Repo.update!()

      payment
    end)
  end

  def overdue_invoices(direction) do
    from(i in Invoice,
      join: v in Vendor, on: v.id == i.vendor_id,
      where: i.direction == ^direction
             and i.status in ["open", "partial"]
             and i.due_date < ^Date.utc_today(),
      order_by: [asc: i.due_date],
      select: %{
        invoice_number: i.invoice_number,
        vendor_name: v.name,
        amount: i.amount,
        due_date: i.due_date,
        days_overdue: fragment("CURRENT_DATE - due_date")
      }
    )
    |> Repo.all()
  end
end
```

#### MCP Tool Components (Database-Backed)

```elixir
defmodule MyApp.MCP.Tools.CreateInvoice do
  use Anubis.Server.Component, type: :tool

  alias MyApp.Accounting

  @impl true
  def name, do: "create_invoice"

  @impl true
  def description, do: "Create a new invoice (receivable or payable)"

  schema do
    field :vendor_id, :integer, required: true, description: "Vendor/Customer ID"
    field :amount, :number, required: true, description: "Invoice amount"
    field :direction, :string, required: true,
      description: "Type: 'receivable' (customer owes us) or 'payable' (we owe vendor)"
    field :description, :string, description: "Invoice description"
    field :due_days, :integer, description: "Days until due (default: 30)"
    field :gl_account, :string, description: "General ledger account code"
  end

  @impl true
  def execute(params, _frame) do
    due_days = Map.get(params, :due_days, 30)

    attrs = %{
      invoice_number: "INV-#{Date.utc_today() |> Calendar.strftime("%Y%m%d")}-#{System.unique_integer([:positive]) |> rem(10000)}",
      vendor_id: params.vendor_id,
      direction: params.direction,
      amount: Decimal.new("#{params.amount}"),
      issue_date: Date.utc_today(),
      due_date: Date.utc_today() |> Date.add(due_days),
      description: Map.get(params, :description),
      gl_account: Map.get(params, :gl_account)
    }

    case Accounting.create_receivable_invoice(attrs) do
      {:ok, invoice} ->
        {:ok, """
        Invoice created successfully:
          Number: #{invoice.invoice_number}
          Direction: #{invoice.direction}
          Amount: #{invoice.currency} #{invoice.amount}
          Due: #{invoice.due_date}
          Status: #{invoice.status}
        """}

      {:error, changeset} ->
        errors = Ecto.Changeset.traverse_errors(changeset, fn {msg, _} -> msg end)
        {:error, "Failed to create invoice: #{inspect(errors)}"}
    end
  end
end
```

```elixir
defmodule MyApp.MCP.Tools.RecordPayment do
  use Anubis.Server.Component, type: :tool

  alias MyApp.Accounting

  @impl true
  def name, do: "record_payment"

  @impl true
  def description, do: "Record a payment against an invoice and update its status"

  schema do
    field :invoice_id, :integer, required: true, description: "Invoice database ID"
    field :amount, :number, required: true, description: "Payment amount"
    field :payment_method, :string, description: "bank_transfer, credit_card, cash, check"
    field :reference, :string, description: "Payment reference or transaction ID"
  end

  @impl true
  def execute(params, _frame) do
    attrs = %{
      amount: Decimal.new("#{params.amount}"),
      payment_date: Date.utc_today(),
      payment_method: Map.get(params, :payment_method, "bank_transfer"),
      reference: Map.get(params, :reference)
    }

    case Accounting.record_payment(params.invoice_id, attrs) do
      {:ok, payment} ->
        {:ok, """
        Payment recorded:
          Invoice ID: #{params.invoice_id}
          Amount: #{payment.amount}
          Method: #{payment.payment_method}
          Reference: #{payment.reference || "N/A"}
        """}

      {:error, reason} ->
        {:error, "Failed to record payment: #{inspect(reason)}"}
    end
  end
end
```

```elixir
defmodule MyApp.MCP.Tools.AgingReport do
  use Anubis.Server.Component, type: :tool

  alias MyApp.Accounting

  @impl true
  def name, do: "aging_report"

  @impl true
  def description, do: "Generate an accounts receivable or payable aging report from the database"

  schema do
    field :direction, :string, required: true,
      description: "'receivable' for AR aging or 'payable' for AP aging"
  end

  @impl true
  def execute(%{direction: direction}, _frame) do
    report = Accounting.aging_report(direction)

    label = if direction == "receivable", do: "Accounts Receivable", else: "Accounts Payable"

    {:ok, """
    #{label} Aging Report (#{Date.utc_today() |> Date.to_iso8601()}):
      Current (not yet due):  EUR #{format_decimal(report.current)}
      31-60 days overdue:     EUR #{format_decimal(report.days_31_60)}
      61-90 days overdue:     EUR #{format_decimal(report.days_61_90)}
      90+ days overdue:       EUR #{format_decimal(report.over_90)}
      ─────────────────────────────────
      Total Outstanding:      EUR #{format_decimal(report.total)}
      Open Invoices:          #{report.invoice_count}
    """}
  end

  defp format_decimal(nil), do: "0.00"
  defp format_decimal(d), do: Decimal.round(d, 2) |> Decimal.to_string()
end
```

```elixir
defmodule MyApp.MCP.Tools.OverdueInvoices do
  use Anubis.Server.Component, type: :tool

  alias MyApp.Accounting

  @impl true
  def name, do: "list_overdue_invoices"

  @impl true
  def description, do: "List all overdue invoices with days overdue and vendor details"

  schema do
    field :direction, :string, required: true,
      description: "'receivable' or 'payable'"
  end

  @impl true
  def execute(%{direction: direction}, _frame) do
    invoices = Accounting.overdue_invoices(direction)

    if Enum.empty?(invoices) do
      {:ok, "No overdue #{direction} invoices found."}
    else
      lines =
        Enum.map_join(invoices, "\n", fn inv ->
          "  #{inv.invoice_number} | #{inv.vendor_name} | EUR #{inv.amount} | " <>
          "Due: #{inv.due_date} | #{inv.days_overdue} days overdue"
        end)

      {:ok, "Overdue #{direction} invoices:\n#{lines}"}
    end
  end
end
```

#### Audit Logging

```elixir
defmodule MyApp.MCP.AuditLogger do
  alias MyApp.Repo
  alias MyApp.Accounting.AuditLog

  def log_tool_call(tool_name, params, result, session_id) do
    %AuditLog{}
    |> AuditLog.changeset(%{
      action: "mcp_tool_call:#{tool_name}",
      details: %{
        params: sanitize_params(params),
        result_preview: String.slice(to_string(result), 0, 500),
        session_id: session_id
      }
    })
    |> Repo.insert()
  end

  defp sanitize_params(params) do
    params
    |> Map.drop([:password, :secret, :token, :api_key])
    |> Map.new(fn {k, v} -> {to_string(k), inspect(v)} end)
  end
end
```

### Full Accounting MCP Server — Python + SQLAlchemy + PostgreSQL

```python
from mcp.server.fastmcp import FastMCP
from sqlalchemy import create_engine, text
from datetime import date, timedelta
from decimal import Decimal

DATABASE_URL = "postgresql://user:password@localhost:5432/accounting"
engine = create_engine(DATABASE_URL)

mcp = FastMCP("AccountingDepartment")


@mcp.tool()
def create_invoice(
    vendor_id: int,
    amount: float,
    direction: str,
    description: str = "",
    due_days: int = 30,
    gl_account: str = "1200"
) -> str:
    """Create a receivable or payable invoice in the database.
    direction: 'receivable' (customer owes us) or 'payable' (we owe vendor)
    """
    inv_number = f"INV-{date.today().strftime('%Y%m%d')}-{abs(hash(str(amount) + str(vendor_id))) % 10000:04d}"
    due_date = date.today() + timedelta(days=due_days)

    with engine.begin() as conn:
        conn.execute(text("""
            INSERT INTO invoices (invoice_number, vendor_id, direction, amount,
                                  issue_date, due_date, description, gl_account,
                                  status, currency, tax_amount, inserted_at, updated_at)
            VALUES (:inv, :vid, :dir, :amt, :issue, :due, :desc, :gl,
                    'open', 'EUR', 0, NOW(), NOW())
        """), {
            "inv": inv_number, "vid": vendor_id, "dir": direction,
            "amt": amount, "issue": date.today(), "due": due_date,
            "desc": description, "gl": gl_account
        })

    return f"Invoice {inv_number} created: {direction}, EUR {amount:.2f}, due {due_date}"


@mcp.tool()
def record_payment(
    invoice_id: int,
    amount: float,
    payment_method: str = "bank_transfer",
    reference: str = ""
) -> str:
    """Record a payment against an invoice and update its status."""
    with engine.begin() as conn:
        conn.execute(text("""
            INSERT INTO payments (invoice_id, amount, payment_date, payment_method,
                                  reference, inserted_at, updated_at)
            VALUES (:iid, :amt, :pd, :pm, :ref, NOW(), NOW())
        """), {
            "iid": invoice_id, "amt": amount,
            "pd": date.today(), "pm": payment_method, "ref": reference
        })

        result = conn.execute(text("""
            SELECT i.amount as invoice_amount, COALESCE(SUM(p.amount), 0) as total_paid
            FROM invoices i LEFT JOIN payments p ON p.invoice_id = i.id
            WHERE i.id = :iid GROUP BY i.amount
        """), {"iid": invoice_id}).fetchone()

        if result:
            new_status = "paid" if result.total_paid >= result.invoice_amount else "partial"
            conn.execute(text(
                "UPDATE invoices SET status = :st, updated_at = NOW() WHERE id = :iid"
            ), {"st": new_status, "iid": invoice_id})

    return f"Payment of EUR {amount:.2f} recorded against invoice #{invoice_id}. Status: {new_status}"


@mcp.tool()
def aging_report(direction: str = "receivable") -> str:
    """Generate aging report from the database. direction: 'receivable' or 'payable'"""
    with engine.connect() as conn:
        result = conn.execute(text("""
            SELECT
                COALESCE(SUM(CASE WHEN due_date >= CURRENT_DATE THEN amount ELSE 0 END), 0) as current_amt,
                COALESCE(SUM(CASE WHEN due_date < CURRENT_DATE
                    AND due_date >= CURRENT_DATE - 30 THEN amount ELSE 0 END), 0) as d30,
                COALESCE(SUM(CASE WHEN due_date < CURRENT_DATE - 30
                    AND due_date >= CURRENT_DATE - 60 THEN amount ELSE 0 END), 0) as d60,
                COALESCE(SUM(CASE WHEN due_date < CURRENT_DATE - 60 THEN amount ELSE 0 END), 0) as d90,
                COALESCE(SUM(amount), 0) as total,
                COUNT(*) as cnt
            FROM invoices
            WHERE direction = :dir AND status IN ('open', 'partial', 'overdue')
        """), {"dir": direction}).fetchone()

    label = "Accounts Receivable" if direction == "receivable" else "Accounts Payable"
    return f"""{label} Aging Report ({date.today()}):
  Current:        EUR {result.current_amt:>12,.2f}
  1-30 days:      EUR {result.d30:>12,.2f}
  31-60 days:     EUR {result.d60:>12,.2f}
  60+ days:       EUR {result.d90:>12,.2f}
  ────────────────────────────────
  Total:          EUR {result.total:>12,.2f}  ({result.cnt} invoices)"""


@mcp.tool()
def list_overdue_invoices(direction: str = "receivable") -> str:
    """List all overdue invoices with vendor name and days overdue."""
    with engine.connect() as conn:
        rows = conn.execute(text("""
            SELECT i.invoice_number, v.name as vendor_name, i.amount,
                   i.due_date, (CURRENT_DATE - i.due_date) as days_overdue
            FROM invoices i JOIN vendors v ON v.id = i.vendor_id
            WHERE i.direction = :dir AND i.status IN ('open', 'partial')
                  AND i.due_date < CURRENT_DATE
            ORDER BY i.due_date ASC
        """), {"dir": direction}).fetchall()

    if not rows:
        return f"No overdue {direction} invoices."

    lines = "\n".join(
        f"  {r.invoice_number} | {r.vendor_name} | EUR {r.amount:,.2f} | "
        f"Due: {r.due_date} | {r.days_overdue} days overdue"
        for r in rows
    )
    return f"Overdue {direction} invoices:\n{lines}"


@mcp.resource("accounting://chart-of-accounts")
def chart_of_accounts() -> str:
    """Company chart of accounts."""
    return """1000 Assets | 1100 Cash | 1200 Accounts Receivable | 1300 Inventory
2000 Liabilities | 2100 Accounts Payable | 2200 Tax Liabilities
3000 Equity | 4000 Revenue | 5000 Expenses"""


@mcp.resource("accounting://tax-rates")
def tax_rates() -> str:
    """Current tax rates for invoice calculations."""
    return """Germany (2026):
  Standard VAT: 19%
  Reduced VAT: 7% (food, books, transport)
  Intra-EU B2B: 0% (reverse charge applies)
  Export (non-EU): 0%"""


@mcp.prompt()
def dunning_workflow(customer_name: str) -> str:
    """Guide through the dunning process for a specific customer."""
    return f"""
Dunning workflow for "{customer_name}":
1. Use list_overdue_invoices(direction="receivable") to find their overdue items
2. Check the aging_report to see the overall AR position
3. Review accounting://payment-terms for the dunning policy stages
4. Based on days overdue, recommend the appropriate dunning action:
   - 1-30 days: Friendly payment reminder email
   - 31-60 days: Formal reminder with late fee warning
   - 61-90 days: Final notice, apply 1.5% monthly late fee
   - 90+ days: Escalate to collections or legal
5. Draft the appropriate communication for the customer
"""


@mcp.prompt()
def vendor_payment_batch() -> str:
    """Prepare a batch of vendor payments for approval."""
    return """
Vendor Payment Batch Preparation:
1. Use list_overdue_invoices(direction="payable") to find due vendor invoices
2. Check each vendor's payment terms
3. Prioritize payments:
   - First: invoices with early-payment discounts about to expire
   - Second: invoices past due
   - Third: invoices due within 7 days
4. For each payment, verify:
   - Invoice amount matches purchase order
   - Goods/services were received
   - No duplicate payments exist
5. Generate a payment batch summary with total amount and bank details
"""


if __name__ == "__main__":
    mcp.run(transport="stdio")
```

## Practical Application

### Usage Scenarios in the Accounting Department

#### Scenario 1: AI-Assisted Invoice Processing

User says to Claude/Cursor:
> "Create a receivable invoice for customer ID 42, amount EUR 3,500 for consulting services, net 30"

The AI calls `create_invoice(vendor_id=42, amount=3500, direction="receivable", description="Consulting services", due_days=30)` and returns the confirmation.

#### Scenario 2: Automated Dunning

User says:
> "Which customers have overdue invoices? Start the dunning process for the worst ones."

The AI:
1. Calls `list_overdue_invoices(direction="receivable")`
2. Calls `aging_report(direction="receivable")`
3. Reads `accounting://payment-terms` for dunning policy
4. Recommends specific actions per customer based on days overdue

#### Scenario 3: Month-End Close

User selects the "monthly_close_checklist" prompt, and the AI walks through each step, calling tools to gather data and flagging discrepancies.

#### Scenario 4: Vendor Payment Approval

User says:
> "Prepare today's vendor payment batch"

The AI:
1. Calls `list_overdue_invoices(direction="payable")`
2. Groups by vendor and payment priority
3. Checks for early-payment discounts
4. Generates a summary for CFO approval

### Deployment Checklist

```
Pre-Production:
  [ ] Database migrations applied
  [ ] Environment variables configured (DB_URL, API keys)
  [ ] OAuth 2.1 or API key authentication enabled
  [ ] Origin allowlist configured for known AI clients
  [ ] Audit logging enabled and tested
  [ ] Rate limiting configured per user/session
  [ ] TLS certificate installed (reverse proxy or direct)

Testing:
  [ ] All tools tested via MCP Inspector
  [ ] Error handling verified (invalid inputs, DB errors)
  [ ] Concurrent access tested (multiple AI sessions)
  [ ] Recovery tested (server restart, DB reconnect)

Monitoring:
  [ ] Telemetry events for tool calls
  [ ] Alert on high error rates
  [ ] Dashboard for tool usage patterns
  [ ] Cost tracking (if using paid LLM APIs)
```

### Docker Deployment

```dockerfile
# Dockerfile for Python MCP Server
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY accounting_mcp.py .
EXPOSE 8080
CMD ["python", "-c", "from accounting_mcp import mcp; mcp.run(transport='streamable-http', host='0.0.0.0', port=8080)"]
```

```yaml
# docker-compose.yml
services:
  mcp-server:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/accounting
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: accounting
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  pgdata:
```

### Remote Client Configuration

```json
{
  "mcpServers": {
    "accounting": {
      "url": "https://mcp.yourcompany.com/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_OAUTH_TOKEN"
      }
    }
  }
}
```

## Related Topics

- [MCP Servers Part 1: Fundamentals & Building](mcp_servers_fundamentals.md) — architecture, primitives, basic examples
- [What is an LLM?](../llm_fundamentals/what_is_an_llm.md) — foundational LLM concepts
- [Elixir AI Zero to Hero](../api_integration/elixir_ai_zero_to_hero.md) — full Elixir AI stack
- [Elixir LLM Implementation](../code_examples/elixir_llm_implementation.md) — calling LLMs from Elixir
- [Authentication](../authentication/zero_to_hero_authentication.md) — OAuth2 for securing MCP servers

### References

- [MCP Specification](https://modelcontextprotocol.io/)
- [Official Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [Anubis MCP (Elixir)](https://github.com/zoedsoupe/anubis-mcp)
- [ConduitMCP (Elixir)](https://github.com/nyo16/conduit_mcp)
- [MCP Inspector](https://github.com/modelcontextprotocol/inspector)
