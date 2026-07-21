# Output Schema

Every parsed statement MUST produce a JSON file conforming to this schema.

## Top-Level Structure

```json
{
  "broker": "ibkr | schwab | fidelity | dbs-sg",
  "account_id": "string",
  "account_type": "brokerage | wealth_management | mcsa_deposit | deposit",
  "period": {
    "from": "YYYY-MM-DD",
    "to": "YYYY-MM-DD"
  },
  "starting_cash": 0.0,
  "ending_cash": 0.0,
  "position_transactions": [ ... ],
  "cash_transactions": [ ... ],
  "lot_actions": [ ... ],
  "_note": "string (optional)"
}
```

### Top-level fields

| Field | Required | Description |
|---|---|---|
| `broker` | yes | Broker slug |
| `account_id` | yes | Account/portfolio number as printed on the statement |
| `account_type` | when a broker splits accounts | See [Account Types](#account-types). Omit for single-account brokers. |
| `period` | yes | `{from, to}` — **always derived from the statement content, never the filename** |
| `starting_cash` | yes (nullable) | Opening cash balance. `null` when the document has no cash section (e.g. an IBKR Trade Confirmation Report) — pair with a `_note`. |
| `ending_cash` | yes (nullable) | Closing cash balance. Same nullability rule. |
| `_note` | optional | Free-text parser note: reconciliation residuals, document-type oddities, filename/content mismatches, duplicates, excluded pending rows. **Use this instead of silently dropping information.** |

### Account Types

Some brokers issue one PDF covering **two distinct accounts** — a securities/brokerage account and a
cash/deposit account. These must be emitted as **separate records**, not merged, because they differ
in nature and tax treatment.

| Value | Meaning |
|---|---|
| `brokerage` | Ordinary securities account (default for Schwab/IBKR/Fidelity) |
| `wealth_management` | DBS custody/brokerage side — holdings, trades, corporate actions |
| `mcsa_deposit` | DBS Multi-Currency Settlement Account — the cash/deposit ledger |
| `deposit` | Any other pure cash/deposit account |

When a statement is split this way, the trade↔cash pairing in Rule 5 is validated **across** the two
records for the same statement, not within one (see Rule 5).

## position_transactions

Each entry represents a buy or sell of a security.

| Field      | Type   | Description                                    |
|------------|--------|------------------------------------------------|
| date       | string | Trade date in `YYYY-MM-DD` format              |
| type       | string | `buy` or `sell`                                |
| ticker     | string | Ticker symbol (e.g., `AAPL`, `MSFT`)          |
| quantity   | number | Number of shares (positive for buy and sell)   |
| price      | number | Price per share                                |
| amount     | number | Total amount (negative for buy, positive for sell) |
| currency   | string | ISO 4217 currency code (e.g., `USD`, `INR`)   |

## cash_transactions

A **complete cash ledger** — every line item that moves cash, including the cash impact of trades. `cash_transactions` must be self-contained: `Starting Cash + sum(cash_transactions[].amount) = Ending Cash`.

| Field       | Type   | Description                                         |
|-------------|--------|-----------------------------------------------------|
| date        | string | Transaction date in `YYYY-MM-DD` format             |
| type        | string | One of: `buy`, `sell`, `dividend`, `interest`, `fee`, `deposit`, `withdrawal`, `forex`, `tax`, `other` |
| description | string | Original description from the statement             |
| amount      | number | Signed amount (positive = inflow, negative = outflow) |
| currency    | string | ISO 4217 currency code                              |

Trade cash impact (`buy`/`sell`) duplicates the `amount` from the corresponding `position_transactions` entry. This redundancy is intentional — it makes `cash_transactions` a standalone cash ledger.

## lot_actions

Each entry represents a corporate action that changes position quantity without a trade.

| Field            | Type   | Description                                         |
|------------------|--------|-----------------------------------------------------|
| date             | string | Action date in `YYYY-MM-DD` format                  |
| type             | string | One of: `split`, `bonus`, `merger`, `reorg`, `transfer` |
| ticker           | string | Ticker symbol affected                              |
| description      | string | Original description from the statement             |
| ratio_from       | number | Split ratio numerator (e.g., 1 in a 4:1 split)     |
| ratio_to         | number | Split ratio denominator (e.g., 4 in a 4:1 split)   |
| currency         | string | ISO 4217 currency code                              |

Lot actions have no cash impact. They change position quantities only.

**The `type` enum is closed.** Use exactly one of the five values — do **not** invent directional
variants such as `transfer_in` / `transfer_out`. Direction belongs in `description` (e.g.
`"Client transfer OUT, …"`), which keeps the enum small and downstream grouping reliable.

**Unknown ratios**: when a statement reports a split by the *quantity credited* without printing the
ratio, set `ratio_from`/`ratio_to` to `null` and put the share count in `description`. Prefer a
statement-grounded `null` over an inferred ratio; only fill the ratio when the document states it or
the pre/post holdings make it unambiguous.

**Cash mergers produce two records**: a `merger` lot_action (position removed, no cash) *and* a cash
credit for the consideration. There is no `merger` type in the cash enum — use `other` and describe it.

## Rules

1. All dates use `YYYY-MM-DD` format.
2. All amounts are numeric (no commas or currency symbols).
3. `position_transactions.amount` sign convention: negative for purchases (cash outflow), positive for sales (cash inflow).
4. `cash_transactions.amount` sign convention: positive for inflows (dividends, deposits, sales), negative for outflows (fees, withdrawals, taxes, purchases).
5. Every `position_transaction` must have a corresponding `cash_transaction` with the same date, amount, and type (`buy`/`sell`).
   - **Exception — Fidelity Stock Plan accounts**: ESPP purchases and stock option exercises are externally funded (payroll deductions / exercise proceeds) and do NOT have matching `cash_transaction` entries. Only transactions that affect the core cash account (FDRXX) appear in `cash_transactions`.
   - **Exception — split brokerage/deposit accounts (DBS)**: when a statement is emitted as two records (`wealth_management` + `mcsa_deposit`), the trade lives on the brokerage record and its settlement cash on the deposit record. Validate the pairing **across** the two records for the same statement.
   - **Settlement lag**: the trade date and its cash settlement date may differ by 1–2 business days. Use the **trade date** on `position_transactions` and the **cash/settlement date** on `cash_transactions` — matching on amount, not date, is the reliable check. Note the lag in `_note`.
6. Forex transactions record the net cash effect in the target currency.
7. If a field cannot be determined, use `null` rather than omitting it.
8. The JSON file is written alongside the PDF with the same base name and `.json` extension.
9. **Cash reconciliation**: `starting_cash + sum(cash_transactions[].amount) = ending_cash`. This is the primary validation check.
   - **Multi-currency accounts reconcile PER CURRENCY**, not in aggregate. Summing across currencies is meaningless. For each currency: `opening[ccy] + sum(cash_transactions where currency == ccy) = closing[ccy]`. Each must balance independently.
   - **Tolerance**: require exactness to the cent. If a residual of ±0.01 remains after re-checking, it is usually the broker's own display rounding (per-line values rounded independently of section totals) — keep the statement's stated `ending_cash` and record the residual and its cause in `_note`. Never silently absorb a discrepancy into an invented transaction.
10. Lot actions have no cash impact and do not appear in `cash_transactions`. They only change position quantities.
11. **Never derive the statement period, account, or content from the filename.** Filenames are frequently wrong: contents swapped between months, a single-day filename holding a full-year statement, month-offset numbering, and byte-identical duplicates. Always read the period from the document body. If the filename disagrees with the content, parse the content and record the mismatch in `_note`.
12. **Never derive an amount's sign from its transaction type.** Take it from the statement's own signal — the Debit/Credit column, parentheses, an explicit minus, or the running-balance delta. Interest can be charged, fees can be reversed, and transfers go both directions.
13. **Exclude pending/unsettled activity.** Rows outside the settled closing balance ("Pending / Open Activity", "Pending settlement transactions") have not occurred yet. They either settle next period — where they are recorded — or never post at all. Including them breaks reconciliation and double-counts.
14. **An empty statement is a valid parse.** Quiet or dormant periods may have no transaction section at all. Emit empty arrays with correct metadata and balances; do not treat it as a failure or skip the file.
