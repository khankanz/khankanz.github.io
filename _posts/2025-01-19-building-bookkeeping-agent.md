---
layout: post
title: "Building a Bookkeeping Agent: What Worked, What Didn't"
date: 2025-01-19
excerpt: "I built tools for a financial agent from scratch while interviewing for an agents role. Calculator failed, DuckDB saved the day, and I found my $70/month Starbucks habit."
tags: [agents, tool-use, claude, duckdb, pydantic, structured-extraction]
---

I was interviewing for a role at a company building financial agents. I wanted to understand the patterns by building, not reading. So I built a bookkeeping agent from scratch—tools for parsing bank statements, categorizing transactions, and answering questions about my spending.

The goal: "Process all bank statements and tell me where my money is going."

---

## The Stack

I used [claudette](https://github.com/AnswerDotAI/claudette), a lightweight wrapper around Anthropic's SDK from Answer.AI. The key feature is `toolloop`—it automates the back-and-forth where Claude requests a tool, you execute it, Claude sees the result, requests another tool, and so on until done.

The pattern is simple: define Python functions with type hints and docstrings, pass them to Claude, and let it figure out what to call based on the user's request.

```python
def get_customer_info(customer_id: str) -> dict:
    "Retrieves customer information and orders"
    return customers.get(customer_id, "Customer not found")
```

Claude uses reflection to inspect the function signature and documentation. You're essentially giving the LLM a menu of capabilities and letting it compose them.

---

## Building Up the Tools

### Stage 1: File I/O

Started basic. Two tools: `list_files` and `read_file`.

```python
def list_files(fp: str) -> list:
    """Given a filepath, returns list of files in it."""
    return Path(fp).ls()

def read_file(fname: str) -> str:
    """Read and return the text content of a file."""
    return Path(fname).read_text()
```

Now the agent can explore the filesystem. Ask "what's in the data folder?" and it calls `list_files`, sees the bank statements, reports back.

### Stage 2: Structured Extraction

Raw text is useless for analysis. I needed structured data.

Built a Pydantic model for statement metadata:

```python
class StatementMetadata(BaseModel):
    source_file: str
    bank_name: str
    account_number: str
    statement_start: str
    statement_end: str
    opening_balance: Optional[float]
    closing_balance: Optional[float]
```

Then a tool that reads a file and extracts structured metadata using Claude's structured outputs:

```python
def extract_statement_metadata(filepath: str) -> dict:
    """Extract structured metadata from a bank statement file."""
    text = Path(filepath).read_text()
    response = anthropic_cli.beta.messages.parse(
        model="claude-sonnet-4-5",
        output_format=StatementMetadata,
        messages=[{"role": "user", "content": f"Extract metadata:\n{text}"}]
    )
    return response.parsed_output.model_dump()
```

Same pattern for transactions—a `Transaction` model with date, description, amount, balance, category.

### Stage 3: Context Window Blowup

Here's where it got interesting.

Large bank statements killed the context window. Stuffing 20 pages of transactions into a single extraction call? Disaster.

**Fix:** Split by pages first, process each page separately.

```python
def split_by_pages(text: str, delimiter: str = '--- Page') -> list[str]:
    """Split text into pages based on delimiter."""
    pages = text.split(delimiter)
    return [p.strip() for p in pages if p.strip()]
```

Then iterate:

```python
for i, page in enumerate(pages):
    if i > 0: time.sleep(2)  # Rate limiting
    response = anthropic_cli.beta.messages.parse(
        output_format=List[Transaction],
        messages=[{"role": "user", "content": f"Extract from page {i}:\n{page}"}]
    )
    all_transactions.extend(response.parsed_output)
```

**Key insight:** Don't return the data to the agent. Save it internally to a CSV and return a summary. Keeps the agent's context clean.

```python
df.to_csv(output_path, index=False)
return f"Parsed {len(df)} transactions. Saved to: {output_path}"
```

### Stage 4: Categorization

Now I had transactions. But "STARBUCKS #12345 TORONTO ON" doesn't tell me I'm bleeding money on coffee.

Here's where my day job bled into the side project. In clinical NLP, we work with ontologies like SNOMED—hierarchical structures where a specific diagnosis rolls up into broader categories. Microcalcifications → Breast Findings → Radiology Observations. Parent-child relationships.

I applied the same pattern:

```python
class ParentCategory(str, Enum):
    INCOME = "Income"
    HOUSING = "Housing"
    TRANSPORTATION = "Transportation"
    DINING = "Dining"
    # ...

class TransactionClassification(BaseModel):
    category: ParentCategory  # Constrained enum
    subcategory: str  # Free-form - let the LLM extract this
    confidence: float
```

The key insight: **constrain the parent, free the child.**

Parent categories are a fixed enum—I control the buckets. But `subcategory` is a free string. The LLM reads "SHELL GAS STN 4521 TORONTO" and extracts "Shell Gas". It reads "PETRO CANADA #1234" and extracts "Petro Canada".

Why does this matter? "Transportation" tells me I spent $400 on getting around. But the subcategory breakdown tells me $280 was Shell and $120 was Petro. Maybe Shell is closer to work but Petro has better prices. That's actionable.

Same with dining—"Dining: $200" is useless. "Starbucks: $70, McDonald's: $45, Thai Palace: $85" tells a story.

Let the LLM do what it's good at: extracting specific merchant names from messy bank descriptions. Constrain what needs consistency (the rollup categories), leave the rest flexible.

Run each transaction through Claude, save the enriched data back to CSV.

---

## What Didn't Work: The Calculator

The agent would say "You spent about $60-70 on Starbucks." Useful, but not exact.

I built a calculator tool:

```python
def calculate(expression: str) -> str:
    """Safely evaluate a mathematical expression."""
    # AST-based safe eval...
    return f"{expression} = {result}"
```

Added a system prompt: "When reporting totals, ALWAYS use the calculate tool to verify your math."

**Problem:** Transcription errors. The agent would read the CSV, try to construct an expression like `2.57 + 2.89 + 3.10 + ...`, but miss values or hallucinate numbers. The math was correct, but the inputs were wrong.

---

## What Worked: DuckDB

The breakthrough was realizing I could just SQL the CSV directly.

```python
import duckdb

result = duckdb.sql(f"""
    SELECT SUM(amount)
    FROM '{filepath}'
    WHERE subcategory LIKE '%Starbucks%'
""")
```

Holy smokes. SQL on a CSV. No loading into a database, no setup, just query.

Wrapped it as a tool:

```python
def query_csv(filepath: str, sql: str) -> str:
    """Run a SQL query on a CSV file using DuckDB."""
    full_sql = sql.replace('FROM data', f"FROM '{filepath}'")
    result = duckdb.sql(full_sql)
    return result.df().to_markdown(index=False)
```

Updated the system prompt: "When reporting totals, ALWAYS use query_csv with SQL to get exact numbers—never estimate."

Now I could ask:

- "How much did I spend on Starbucks?"
- "What's my biggest expense category?"
- "Show me all dining transactions over $50"

And get exact answers backed by real queries.

---

## The Payoff

Ran it on my own bank statements from November-December 2024.

The agent found I was spending **$70/month on Starbucks**.

Two months. $140 on coffee.

I bought a coffee machine on Boxing Day for $40.

---

## What I'd Do Differently

1. **Start with DuckDB.** The calculator detour wasted time. SQL-on-CSV is the right abstraction for financial data.

2. **Better system prompts earlier.** I kept hitting issues where the agent would explore wrong directories or estimate instead of calculating. Clear constraints in the system prompt ("All files are in X. Never estimate.") saved debugging time.

3. **Batch processing resilience.** Rate limits hit hard during categorization. Built in sleep/retry, but should have added checkpointing—resume from where you left off.

---

## Code

The full notebook is here: [Colab](https://colab.research.google.com/gist/khankanz/02d572ad56a40f10771a87abd9decddb/build-an-ai-agent-from-scratch.ipynb)

If you're learning agent patterns, I'd recommend building something similar on your own data. The patterns become obvious when you hit the walls yourself.
