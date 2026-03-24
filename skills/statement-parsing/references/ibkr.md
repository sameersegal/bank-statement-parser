# Interactive Brokers (IBKR) Statement Reference

> Last updated: Epoch 1 (2026-03-24) — based on 4 training + 2 test PDFs (Sep 2025–Feb 2026)

## Statement Structure

IBKR Activity Statements are titled "Activity Statement" with the period shown as:
`<Month> <Day>, <Year> - <Month> <Day>, <Year>` (e.g., "September 1, 2025 - September 30, 2025")

### Page Layout (typical 7-page statement)

| Page | Sections |
|------|----------|
| 1 | Account Information, Net Asset Value, Change in NAV, Mark-to-Market Performance Summary (start) |
| 2 | Mark-to-Market (cont.), Realized & Unrealized Performance Summary |
| 3 | Cash Report, Open Positions (start) |
| 4 | Interest Accruals, Interest, Withholding Tax, Dividends, Fees, Change in Dividend Accruals |
| 5+ | Financial Instrument Information, Codes, Notes/Legal Notes |

When trades exist, a **Trades** section appears between Open Positions and Interest Accruals (pages 4-5), and the statement may be longer (8-9 pages).

## Metadata Extraction

- **Account ID**: Found in "Account Information" table → "Account" row (e.g., `UXXXXXXXX`)
- **Period**: From the page header subtitle: "Activity Statement" line shows `<StartDate> - <EndDate>`
  - Parse as: month name + day + year → `YYYY-MM-DD`

## Section Headers (exact text)

These are the bold section headers to look for:

- `Account Information`
- `Net Asset Value`
- `Mark-to-Market Performance Summary`
- `Realized & Unrealized Performance Summary`
- `Cash Report`
- `Open Positions`
- `Trades` (only present when trades occurred)
- `Corporate Actions | Glossary` (only present for bond maturities/redemptions/calls)
- `Interest Accruals`
- `Interest`
- `Withholding Tax`
- `Dividends`
- `Fees` (only present when fees occurred)
- `Change in Dividend Accruals`
- `Financial Instrument Information`

## Number Formats

- **Thousands separator**: comma (e.g., `553,410.91`)
- **Decimal separator**: period
- **Negative amounts**: leading minus sign (e.g., `-206.12`). No parentheses observed.
- **Bond prices**: percentage of face value (e.g., `99.4790` means 99.479%)
- **Bond quantities**: face value amounts (e.g., `200,000`)

## Date Formats

- **Statement period**: `<Month Name> <Day>, <Year>` (e.g., "September 1, 2025")
- **Transaction dates**: `YYYY-MM-DD` (e.g., `2025-09-15`)
- **Trade timestamps**: `YYYY-MM-DD, HH:MM:SS` (e.g., `2025-12-26, 11:07:24`)

## Cash Report (Cross-Check)

The **Cash Report** section under "Base Currency Summary" provides line-item totals that should reconcile:

| Line Item | Maps To |
|-----------|---------|
| Starting Cash | Starting balance |
| Dividends | Sum of all dividend cash_transactions |
| Broker Interest Paid and Received | Sum of broker interest cash_transactions |
| Bond Interest Paid and Received | Sum of purchase accrued interest (negative = paid at purchase) |
| Other Fees | Sum of ADR fees and similar |
| Withholding Tax | Sum of all tax cash_transactions |
| Commissions | Sum of all commission fees |
| Trades (Purchase) | Sum of all position_transactions amounts |
| Ending Cash | Should equal Starting + all above |

**Always verify**: `Starting Cash + sum(all cash & position txns) = Ending Cash`

## Extracting Position Transactions (Trades)

Look for the **Trades** section. It has subsections:

### Bonds
```
Symbol | Date/Time | Quantity | T. Price | C. Price | Proceeds | Comm/Fee | Basis | Realized P/L | MTM P/L | Code
```

- **ticker**: Use the Symbol column (e.g., "T 0 1/2 02/28/26")
- **date**: Parse the Date/Time column, take just the date part (`YYYY-MM-DD`)
- **quantity**: The Quantity column (face value for bonds)
- **price**: The T. Price column (trade price as % of face value)
- **amount**: The Proceeds column (negative = purchase)
- **type**: Negative proceeds = `buy`, positive = `sell`
- **Commission**: Comm/Fee column → record as separate `fee` cash_transaction

### Treasury Bills
Same format as Bonds. May have multiple partial executions for the same security.

### Stocks
Not observed in training data yet, but expected same tabular format.

## Extracting Cash Transactions

### Interest Section
```
Date | Description | Amount
```
- Under **USD** currency header
- Monthly pattern: `USD Credit Interest for <Month>-<Year>`
- Purchase accrued interest: `Purchase Accrued Interest <bond name>` (negative amounts)

### Dividends Section
```
Date | Description | Amount
```
- Format: `<TICKER>(<CUSIP>) Cash Dividend USD <rate> per Share (Ordinary Dividend)`
- Amount is always positive (gross dividend)

### Withholding Tax Section
```
Date | Description | Amount | Code
```
- Format: `<TICKER>(<CUSIP>) Cash Dividend USD <rate> per Share - US Tax`
- Amount is always negative

### Fees Section
```
Date | Description | Other Fees (Amount)
```
- ADR fee format: `<TICKER>(<CUSIP>) ADR Fee USD <rate> per Share`
- Amount column header may say "Other Fees"
- Amount is negative
- **Note**: Fee dates may fall outside the statement period

## Commissions

Commissions appear in the **Trades** section's `Comm/Fee` column, not as standalone entries in the Fees section. They are also totaled in the Cash Report as a separate "Commissions" line.

Record each trade's commission as a separate `fee` cash_transaction with:
- Same date as the trade
- Description: derived from the trade (no standalone text in statement)
- Amount: the Comm/Fee value (negative)

## Currency

- These statements are single-currency (USD base currency)
- All amounts in USD
- Currency header "USD" appears above transaction lists in each section

## Corporate Actions (Bond Maturity / Early Redemption)

A **Corporate Actions | Glossary** section appears when bonds mature or are called. Format:

```
Report Date | Date/Time | Description | Quantity | Proceeds | Value | Realized P/L | Code
```

- **Report Date**: The date recorded in this statement (use as the transaction date)
- **Quantity**: Negative (position decreasing, e.g., `-200,000`)
- **Proceeds**: Positive (cash received, e.g., `200,000.00`)
- **Description**: e.g., `(US9128286A35) Full Call / Early Redemption for USD 1.00 per Bond (T 2 5/8 01/31/26, ...)`

Map as a `sell` position_transaction:
- **ticker**: The bond symbol (e.g., "T 2 5/8 01/31/26")
- **quantity**: Absolute value of the Quantity column
- **price**: 100.0 (redeemed at par/face value)
- **amount**: Proceeds value (positive = cash inflow)

The Cash Report lists this under "Trades (Sales)".

## Bond Coupon Payments

Bond coupon payments appear in the **Interest** section:
- Format: `Bond Coupon Payment (<bond symbol> - <full bond name>)`
- Amount is positive (cash inflow)
- Map as `interest` type cash_transaction

The Cash Report lists this under "Bond Interest Paid and Received" (positive when received).

## Edge Cases

1. **Out-of-period dates**: Some transactions (particularly ADR fees) may have dates before the statement start date but are included in the current period's Cash Report.
2. **Multiple partial fills**: Treasury bill purchases may be split across multiple executions (same date/price, different quantities). Record each as a separate position_transaction.
3. **No Trades section**: When no trades occurred in the period, the Trades section is entirely absent.
4. **No Fees section**: When no fees occurred, the Fees section is absent.
5. **Corporate action date vs. event date**: Bond redemptions may have a Date/Time in the prior month (e.g., 2026-01-30) but a Report Date in the current month (e.g., 2026-02-02). Use the Report Date.
6. **Bond coupon + redemption same day**: When a bond matures, both the final coupon payment and the redemption proceeds appear in the same statement.
