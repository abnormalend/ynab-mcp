# YNAB MCP Server — Test Suite

Run each test in order. For each, report: what tool was called, what parameters were used, what was returned, and whether it PASSED or FAILED.

---

## Test 1 — Memo display in `list-transactions`

Call `list-transactions` with a real account ID and a `since_date` of 30+ days ago to get a reasonable sample (at least 10 transactions if possible).

**Pass criteria:**
- Transactions WITH a memo show `| Memo: <text>` before the `(ID: ...)` portion
- Transactions WITHOUT a memo do NOT show `| Memo:` at all — the ID comes right after the amount
- No transaction line shows `| Memo:  (ID:` with a blank after "Memo:"

Pick one transaction that has a memo and record its ID and memo text for Test 3.

---

## Test 2 — `list-transactions` with `account_id` + `month` + `limit`

Call `list-transactions` with:
- An `account_id`
- `month` set to the current month (`2026-03-01`)
- `limit` set to `3`

**Pass criteria:**
- Returns at most 3 transactions
- All returned transactions have dates in March 2026

---

## Test 3 — Write a memo via `bulk-manage-transactions` update

Using the transaction ID from Test 1 (pick one with no existing memo, or overwrite one):

Call `bulk-manage-transactions` with:
```json
{
  "action": "update",
  "update_transactions": [
    {
      "id": "<transaction_id>",
      "memo": "TEST MEMO - cowork verification"
    }
  ]
}
```

Then immediately call `list-transactions` to retrieve that transaction again.

**Pass criteria:**
- The update call returns success
- The follow-up `list-transactions` shows `| Memo: TEST MEMO - cowork verification` on that transaction line

Then clean up by updating the memo back to its original value (or empty string `""` to clear it).

---

## Test 4 — `manage-financial-overview` get action

Call `manage-financial-overview` with `action: "get"`.

**Pass criteria:**
- Returns the financial overview data (even if it's mostly empty/default)
- Does NOT return a tool-not-found error or "read-only" rejection

---

## Test 5 — Unit sanity check

Call `list-categories` and look at the Budgeted/Spent/Balance values for any category you know the approximate dollar amount for.

**Pass criteria:**
- The numbers shown are dollar amounts (e.g. `45.00`), NOT raw milliunits (e.g. `45000`)
- Confirm: a category budgeted at ~$100 shows roughly `100.00`, not `100000`

---

## Test 6 — `bulk-manage-transactions` create with `payee_name`

Create a single test transaction using a payee name instead of a payee ID:

```json
{
  "action": "create",
  "create_transactions": [
    {
      "account_id": "<any_checking_account_id>",
      "date": "2026-03-23",
      "amount": -1000,
      "payee_name": "COWORK TEST PAYEE",
      "memo": "automated test - delete me",
      "approved": false
    }
  ]
}
```

**Pass criteria:**
- Returns success with a transaction ID
- Immediately delete it using `bulk-manage-transactions` with `action: "delete"` and that ID

---

## Summary

| Test | Tool | Pass/Fail | Notes |
|------|------|-----------|-------|
| 1 | list-transactions (memo display) | | |
| 2 | list-transactions (account+month+limit) | | |
| 3 | bulk-manage-transactions update (write memo) | | |
| 4 | manage-financial-overview get | | |
| 5 | list-categories (unit check) | | |
| 6 | bulk-manage-transactions create+delete (payee_name) | | |

---

> **Note:** If Tests 1 or 4 fail immediately with wrong output format or tool errors (not API errors), the MCP server may not have been restarted yet. Report that as a likely restart issue rather than a code bug.
