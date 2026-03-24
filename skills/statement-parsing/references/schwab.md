# Charles Schwab Statement Reference

> Last updated: Epoch 1 (2026-03-24) — based on 8 training PDFs (Sep–Dec 2025, 2 accounts)

## Statement Structure

Schwab statements are titled "Schwab One International® Account of" with the period shown in the header as:
`<Month> <Day>-<Day>, <Year>` (e.g., "September 1-30, 2025") or `<Month> <Day>, <Year>`

### Page Layout (typical 6-page statement)

| Page | Sections |
|------|----------|
| 1 | Account Summary (ending/beginning value, chart, summary table) |
| 2 | Asset Allocation, Top Account Holdings, Gain or (Loss) Summary, Income Summary |
| 3 | Positions - Summary, Cash and Cash Investments, Positions - Equities (start) |
| 4 | Positions - Equities (cont.), Transactions - Summary, Transaction Details (start) |
| 5-6 | Transaction Details (cont.), Pending/Open Activity, Endnotes, Terms and Conditions |

Statements with more transactions (sales, withdrawals, etc.) may be 8 pages.

## Metadata Extraction

- **Account ID**: Found in page header → "Account Number" field (e.g., `XXXX-XXXX`)
  - Format: `XXXX-XXXX`
- **Period**: From the "Statement Period" field in the header
  - Format: `<Month> <Day>-<Day>, <Year>` → parse start and end dates
  - Example: "September 1-30, 2025" → from `2025-09-01`, to `2025-09-30`
  - Example: "December 1-31, 2025" → from `2025-12-01`, to `2025-12-31`

## Number Formats

- **Thousands separator**: comma (e.g., `254,582.80`)
- **Decimal separator**: period
- **Negative amounts**: parentheses (e.g., `(87.86)` means -87.86)
- **Currency**: Always USD, amounts shown with `$` prefix in summary sections but plain numbers in Transaction Details

## Date Formats

- **Statement period**: `<Month> <Day>-<Day>, <Year>`
- **Transaction dates**: `MM/DD` format (e.g., `09/11`, `12/29`)
  - **Year must be inferred** from the statement period
  - All transaction dates use the year from the statement period

## Key Sections for Data Extraction

### Transactions - Summary (Cross-Check)

Provides a single-line reconciliation:
```
Beginning Cash + Deposits + Withdrawals + Purchases + Sales/Redemptions + Dividends/Interest + Expenses = Ending Cash
```

**Always verify**: `Starting Cash + sum(cash_transactions[].amount) = Ending Cash` — cash_transactions alone must balance. Purchases and Sales/Redemptions appear as `buy`/`sell` type cash_transactions.

### Transaction Details

The primary data extraction table:

```
Date | Category | Action | Symbol/CUSIP | Description | Quantity | Price/Rate per Share($) | Charges/Interest($) | Amount($) | Realized Gain/(Loss)($)
```

## Category + Action Mapping

This is the critical mapping from Schwab's Category/Action columns to our schema types:

### Cash Transactions

| Category | Action | → Schema Type | Notes |
|----------|--------|---------------|-------|
| Dividend | Qual. Dividend | `dividend` | Regular qualified dividend |
| Dividend | Qual Div Reinvest | `dividend` | Dividend that will be reinvested (DRIP) |
| Dividend | NRA Tax | `tax` | Non-Resident Alien withholding tax on dividends |
| Interest | Credit Interest | `interest` | Schwab One account interest |
| Interest | NRA Tax | `tax` | NRA withholding tax on interest |
| Withdrawal | Funds Paid | `withdrawal` | Wire transfer out |
| Expense | Service Fee | `fee` | Wire transfer fee, service charges |
| Expense | Misc Cash Entry | `other` | Fee waivers (positive amount = credit) |

### Position Transactions

| Category | Action | → Schema Type | Notes |
|----------|--------|---------------|-------|
| Purchase | Reinvested Shares | `buy` | DRIP — dividend reinvestment purchase |
| Purchase | *(direct purchase)* | `buy` | T-bill or stock purchase |
| Sale | *(stock sale)* | `sell` | Quantity is negative in PDF, use absolute value |

### Lot Actions (Corporate Actions)

| Category | Action | → Schema | Notes |
|----------|--------|----------|-------|
| Other Activity | Stock Split | `lot_action` type=split | Extract ticker and description; no cash impact |

## Extracting Position Transactions

### Stock/DRIP Purchases
- **ticker**: Symbol/CUSIP column (e.g., `MSFT`, `GOOGL`, `AAPL`)
- **date**: MM/DD → combine with year from statement period
- **quantity**: Quantity column (may be fractional for DRIP, e.g., `0.0154`)
- **price**: Price/Rate per Share column
- **amount**: Amount column (negative in parentheses = cash outflow for purchases)
- **type**: `buy`

### Stock Sales
- **ticker**: Symbol/CUSIP column (e.g., `XYZ`)
- **date**: MM/DD → combine with year
- **quantity**: Absolute value of Quantity (shown as negative, e.g., `(39.0000)`)
- **price**: Price/Rate per Share column
- **amount**: Amount column (positive = cash inflow for sales)
- **type**: `sell`
- **Note**: Charges/Interest column may show an "Industry Fee" — this is already embedded in the Amount, do NOT record separately

### T-Bill Purchases
- **ticker**: Symbol/CUSIP column (e.g., `912797SC2`)
- **date**: MM/DD → combine with year
- **quantity**: Par value (e.g., `254,000.0000`)
- **price**: Price as percentage of par (e.g., `99.1420`)
- **amount**: Amount column (negative)
- **type**: `buy`
- May have multiple partial fills (same CUSIP, same price, different quantities)

## Extracting Cash Transactions

### Dividends
- Description column has the stock name (e.g., `MICROSOFT CORP`)
- Symbol/CUSIP has the ticker (e.g., `MSFT`)
- Amount is always positive
- For DRIP accounts, Action is "Qual Div Reinvest" instead of "Qual. Dividend"

### NRA Tax (Withholding)
- Always paired with a dividend or interest entry on the same date
- Amount is always negative (shown in parentheses)
- Applies to both dividend income AND interest income for non-resident accounts

### Interest
- Description: `SCHWAB1 INT <date_range>` (e.g., `SCHWAB1 INT 08/28-09/28`)
- Action: "Credit Interest"
- Always followed by an NRA Tax entry on the same date

### Withdrawals
- Action: "Funds Paid"
- Description: "WIRED FUNDS DISBURSED"
- Amount is negative (in parentheses)

### Fees and Waivers
- Wire fees: Category "Expense", Action "Service Fee", description "WIRED FUNDS FEE", amount -15.00
- Fee waivers: Category "Expense", Action "Misc Cash Entry", description "WAIVE WIRE FEE", amount +15.00
- These always come in pairs (fee + waiver) and net to zero

## DRIP (Dividend Reinvestment) Pattern

In DRIP-enabled accounts (e.g., account 925):
1. Dividend received → `dividend` cash_transaction (positive)
2. NRA Tax withheld → `tax` cash_transaction (negative)
3. Net dividend immediately used to buy shares → `buy` position_transaction (negative, equals dividend minus tax)

The Transactions Summary shows: Purchases = -(Dividends/Interest), so cash balance stays the same.

## Pending / Open Activity

A "Pending / Open Activity" section may appear showing unsettled dividends. These are NOT included in the account value or cash balance. **Do NOT extract pending transactions** — only extract settled transactions from the "Transaction Details" section.

## Edge Cases

1. **Stock splits**: Category "Other Activity", Action "Stock Split" — no cash impact, no amount. Extract as a `lot_action` with type `split`. Parse ticker from Symbol/CUSIP and description from the Description column.
2. **Industry Fee on sales**: The Charges/Interest column may show a small fee (e.g., $0.01). This is already deducted from the Amount column. Do NOT record as a separate fee.
3. **Wire fee waivers**: Always paired with the fee, netting to zero. Record both individually.
4. **Multiple accounts**: Schwab statements are per-account. Different accounts (individual vs joint) may have different features (e.g., DRIP enabled only on the joint account).
5. **Dates spanning months**: Interest descriptions may show date ranges crossing month boundaries (e.g., "08/28-09/28" in a September statement).
6. **T-bill CUSIP as ticker**: T-bills use CUSIP (e.g., `912797SC2`) rather than a human-readable ticker.
