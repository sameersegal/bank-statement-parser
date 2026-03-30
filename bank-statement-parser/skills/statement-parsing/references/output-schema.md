# Output Schema

Every parsed statement MUST produce a JSON file conforming to this schema.

## Top-Level Structure

```json
{
  "broker": "ibkr | schwab | fidelity | dbs-sg",
  "account_id": "string",
  "period": {
    "from": "YYYY-MM-DD",
    "to": "YYYY-MM-DD"
  },
  "position_transactions": [ ... ],
  "cash_transactions": [ ... ],
  "lot_actions": [ ... ]
}
```

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

## Rules

1. All dates use `YYYY-MM-DD` format.
2. All amounts are numeric (no commas or currency symbols).
3. `position_transactions.amount` sign convention: negative for purchases (cash outflow), positive for sales (cash inflow).
4. `cash_transactions.amount` sign convention: positive for inflows (dividends, deposits, sales), negative for outflows (fees, withdrawals, taxes, purchases).
5. Every `position_transaction` must have a corresponding `cash_transaction` with the same date, amount, and type (`buy`/`sell`). **Exception**: For Fidelity Stock Plan accounts, ESPP purchases and stock option exercises are externally funded (payroll deductions / exercise proceeds) and do NOT have matching `cash_transaction` entries. Only transactions that affect the core cash account (FDRXX) appear in `cash_transactions`.
6. Forex transactions record the net cash effect in the target currency.
7. If a field cannot be determined, use `null` rather than omitting it.
8. The JSON file is written alongside the PDF with the same base name and `.json` extension.
9. **Cash reconciliation**: `Starting Cash + sum(cash_transactions[].amount) = Ending Cash`. This is the primary validation check.
10. Lot actions have no cash impact and do not appear in `cash_transactions`. They only change position quantities.
