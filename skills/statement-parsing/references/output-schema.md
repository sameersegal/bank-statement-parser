# Output Schema

Every parsed statement MUST produce a JSON file conforming to this schema.

## Top-Level Structure

```json
{
  "broker": "ibkr | schwab",
  "account_id": "string",
  "period": {
    "from": "YYYY-MM-DD",
    "to": "YYYY-MM-DD"
  },
  "position_transactions": [ ... ],
  "cash_transactions": [ ... ]
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

Each entry represents a non-trade cash movement.

| Field       | Type   | Description                                         |
|-------------|--------|-----------------------------------------------------|
| date        | string | Transaction date in `YYYY-MM-DD` format             |
| type        | string | One of: `dividend`, `interest`, `fee`, `deposit`, `withdrawal`, `forex`, `tax`, `other` |
| description | string | Original description from the statement             |
| amount      | number | Signed amount (positive = inflow, negative = outflow) |
| currency    | string | ISO 4217 currency code                              |

## Rules

1. All dates use `YYYY-MM-DD` format.
2. All amounts are numeric (no commas or currency symbols).
3. `position_transactions.amount` sign convention: negative for purchases (cash outflow), positive for sales (cash inflow).
4. `cash_transactions.amount` sign convention: positive for inflows (dividends, deposits, interest), negative for outflows (fees, withdrawals, taxes).
5. Forex transactions record the net cash effect in the target currency.
6. If a field cannot be determined, use `null` rather than omitting it.
7. The JSON file is written alongside the PDF with the same base name and `.json` extension.
