# llm-budget-cap

**Inline, function-level budget enforcement for LLM API calls.**

Stop runaway spend *before* it happens — not after a platform billing alert
emails you that you've already burned $500 on an infinite loop or a single
heavy user query.

## The problem

Standard API rate limits and platform-side billing alerts are:
- **Reactive** — you find out after the money is spent.
- **Coarse** — they apply account-wide, not per-feature, per-user, or per-call.
- **Slow** — billing dashboards often lag by hours.

## The idea

`llm-budget-cap` is a small middleware library that wraps any LLM-calling
function with a decorator. Before the call executes, it checks one or more
**budget caps** (cost and/or token limits, optionally scoped per user/tenant,
optionally on a rolling time window). If a cap would be breached, it raises
(or routes to a fallback) — **the LLM call never happens**. After a successful
call, actual usage is recorded against every cap that applies.

## Install

```bash
pip install llm-budget-cap
```

(Or, until published to PyPI, install directly from GitHub — see below.)

## Quick start

```python
from llm_budget_cap import BudgetManager, BudgetCap, enforce_budget, BudgetExceededError

budgets = BudgetManager()

# Tier 1: no single call over 2 cents
budgets.add_cap(BudgetCap(name="per_call", max_cost_usd=0.02))

# Tier 2: $0.50/day per user (rolling 24h window)
budgets.add_cap(BudgetCap(
    name="per_user_daily",
    max_cost_usd=0.50,
    window_seconds=86400,
    key_fn=lambda user_id, **_: user_id,
))

# Tier 3: $5.00 hard global ceiling
budgets.add_cap(BudgetCap(name="global_total", max_cost_usd=5.00))


@enforce_budget(budgets, caps=["per_call", "per_user_daily", "global_total"], model="gpt-4o-mini")
def ask(user_id, prompt):
    return openai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
    )

try:
    ask(user_id="alice", prompt="Hello!")
except BudgetExceededError as e:
    print("Blocked:", e)
```

Token usage is auto-extracted from OpenAI- and Anthropic-shaped response
objects (`.usage.prompt_tokens` / `.usage.input_tokens`, etc.). For other
SDKs, pass `usage_extractor=...`.

### Async clients

```python
from llm_budget_cap import enforce_budget_async

@enforce_budget_async(budgets, caps=["per_call"], model="claude-sonnet-4-6")
async def ask(prompt):
    return await async_anthropic_client.messages.create(...)
```

### Pre-call estimation (block *before* spending anything)

```python
@enforce_budget(
    budgets,
    caps=["per_call"],
    model="gpt-4o",
    estimate_input_tokens=lambda prompt: len(prompt) // 4,
    estimate_output_tokens=500,  # your requested max_tokens
)
def ask(prompt): ...
```

### Graceful fallback instead of raising

```python
@enforce_budget(
    budgets, caps=["global_total"], model="gpt-4o",
    on_exceeded=lambda e: "Sorry, I'm over budget for now.",
)
def ask(prompt): ...
```

### Inspecting current spend

```python
budgets.status("per_user_daily", {"user_id": "alice"})
# {'cap': 'per_user_daily', 'key': 'alice', 'cost_usd': 0.0012, 'max_cost_usd': 0.5, ...}
```

## How caps work

| Field            | Meaning                                                              |
|------------------|-----------------------------------------------------------------------|
| `max_cost_usd`   | Hard ceiling on cumulative USD spend within the window                |
| `max_tokens`     | Hard ceiling on cumulative tokens within the window                   |
| `window_seconds` | `None` = lifetime cumulative; otherwise a rolling window (e.g. 86400) |
| `key_fn`         | Scope the cap per key (e.g. per `user_id`); omit for a global cap     |

A function can be wrapped with multiple caps at once — every one must pass.

## Pricing

A small built-in price table (`PRICE_TABLE`, USD per 1K tokens) covers common
OpenAI and Anthropic models. Prices drift — pass your own table for accuracy:

```python
budgets = BudgetManager(price_table={"my-custom-model": (0.001, 0.002), "_default": (0.001, 0.002)})
```

## Distributed / multi-process use

The default `BudgetManager` is in-memory and per-process. For shared limits
across workers, swap the internal `_state` store for a Redis/DB-backed
implementation — `check()` and `record()` are the only two methods that need
a backing store, making this a small adapter to write.

## Running tests

```bash
pip install -e ".[dev]"
pytest
```

## License

MIT
