# YNAB MCP Server — Memo Field Fix

## Context

This is a fix for [Jtewen/ynab-mcp](https://github.com/Jtewen/ynab-mcp), a Python MCP server that connects Claude to YNAB. The PyPI package name is `ynab-mcp-server`.

The YNAB API returns a `memo` field on every transaction object, but the server never includes it in any output. This means Claude cannot read memos through the MCP — it has to fall back to browser automation to read them from the YNAB web UI. That's the bug.

---

## Files to Change

### 1. `src/ynab_mcp_server/server.py`

#### Fix 1: `list-transactions` output (line ~248)

**Current code:**
```python
transaction_list = "\n".join(
    f"- {t.var_date}: {t.payee_name or 'N/A'} | "
    f"{t.category_name or 'N/A'} | {t.amount / 1000:.2f} (ID: {t.id})"
    for t in transactions
)
```

**Replace with:**
```python
transaction_list = "\n".join(
    f"- {t.var_date}: {t.payee_name or 'N/A'} | "
    f"{t.category_name or 'N/A'} | {t.amount / 1000:.2f} | "
    f"Memo: {t.memo or ''} (ID: {t.id})"
    for t in transactions
)
```

---

### 2. `src/ynab_mcp_server/tool_models.py`

The `bulk-manage-transactions` update action uses `SaveTransactionWithIdOrImportId`. Verify that the model used for update transactions includes a `memo` field. If there's a `TransactionUpdate` or similar Pydantic model defined in this file for update payloads, add `memo: str | None = None` to it so Claude can write memos via the MCP (not just read them).

Check for a model like:
```python
class UpdateTransaction(BaseModel):
    id: str
    category_id: str | None = None
    # memo is likely missing here — add it:
    memo: str | None = None
```

---

## What to Verify After the Fix

1. Run `list-transactions` on an account with known memos and confirm the memo text appears in the output.
2. Run a `bulk-manage-transactions` update that sets a memo and confirm it saves in YNAB.
3. Make sure transactions with `None` memos don't show "None" — the `or ''` handles this.

---

## Why This Matters

Amazon orders sync to YNAB with the product name(s) in the memo field. Without memo support, Claude can't read what was purchased and can't auto-categorize Amazon transactions — which is the main use case for using Claude with YNAB at all.
