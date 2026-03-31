# Fidelity Stock Plan Account Statement Reference

> Last updated: Epoch 2 (2026-03-30) — based on 5 train + 4 test monthly/annual PDFs

## Statement Types

Two statement formats exist:

| Format | Title | Period | Detail Level |
|--------|-------|--------|--------------|
| **Monthly** | `STOCK PLAN SERVICES REPORT` | 1–2 months | Individual transactions with dates |
| **Annual** | `YEAR-END INVESTMENT REPORT` | Jan 1 – Dec 31 | Aggregated summaries only |

**Prefer monthly statements** — they provide exact transaction dates, individual dividend/tax entries, and RSU vest details that annual reports aggregate away.

## Fidelity-Specific Extraction Rules

1. **position_transactions**: Capture ESPP purchases, RSU vests, and share sales. Do NOT create matching `cash_transaction` entries for ESPP buys or RSU vests — these are externally funded / employer-granted, not through core cash.
2. **cash_transactions**: Capture dividends, interest, taxes, and sales proceeds. Do NOT capture ESPP purchase cash or "SPP PURCHASE CREDIT" deposit entries.
3. **Schema rule 5 exception**: ESPP buys in `position_transactions` do NOT require matching `cash_transaction` entries.
4. **Cash reconciliation**: `Starting Core Balance + sum(cash_transactions[].amount) = Ending Core Balance`. The core balance is the money market fund (FDRXX/FYIXX) quantity in Holdings.
5. **RSU vests ARE position_transactions** — capture as `buy` with date, quantity, and price from the "SHARES DEPOSITED / Conversion" entry in Activity → Other Activity In. No matching cash_transaction (employer-granted, not market-purchased).

## Metadata Extraction

- **Account ID**: Found as "Participant Number" on page 1. Format: letter prefix + digits.
- **Period**: From the header line (e.g., `February 1, 2022 - March 31, 2022`).
  - Parse start/end dates → `YYYY-MM-DD` format
  - Monthly periods may span 1 or 2 calendar months
  - Annual periods are always Jan 1 – Dec 31
- **Broker**: `fidelity`

## Statement Structure — Monthly

### Page Layout (typical 8-page monthly statement)

| Page | Sections |
|------|----------|
| 1 | Title/Header, Participant Info, Account Value Summary |
| 2 | Account Summary — Accounts Included, Other Holdings |
| 3 | Account Summary (detail) — Account Value, Top Holdings, Income Summary, Core Account Cash Flow |
| 4 | Holdings — Core Account, Stocks (with beginning/ending market values, quantities, prices, cost basis, unrealized gain/loss, EAI/EY) |
| 5 | **Activity** — Securities Bought & Sold, Dividends/Interest/Other Income, Other Activity In, Taxes Withheld, Core Fund Activity |
| 6 | Estimated Cash Flow (rolling 12-month projection — skip, not actual transactions) |
| 7 | Stock Plans — ESPP Contribution Summary, ESPP Purchase Summary |
| 8 | Additional Information and Endnotes |

### Sections to SKIP (not transaction data)
- `Estimated Cash Flow` — forward-looking estimates, not actuals
- `Additional Information and Endnotes` — legal disclosures
- `Top Holdings` — summary, not transactions

## Section Headers (exact text)

### Monthly statement headers
- `Account Summary`
- `Account Holdings`
- `Top Holdings`
- `Income Summary`
- `Core Account and Credit Balance Cash Flow`
- `Holdings`
- `Core Account`
- `Stocks`
- `Common Stock`
- `Activity`
- `Securities Bought & Sold`
- `Dividends, Interest & Other Income`
- `Other Activity In`
- `Taxes Withheld`
- `Core Fund Activity`
- `Estimated Cash Flow`
- `Stock Plans`
- `Employee Stock Purchase Contribution Summary`
- `Employee Stock Purchase Summary`
- `Additional Information and Endnotes`

## Number Formats

- **Currency prefix**: Dollar sign (e.g., `$12,345.67`)
- **Thousands separator**: comma
- **Decimal separator**: period
- **Negative amounts**: leading minus sign, sometimes with dollar sign (e.g., `-4.86`, `-$1,234.56`). No parentheses.
- **Dash for zero/empty**: A single `-` or `--` represents zero or not available
- **Stock quantities**: decimal precision to 3 places (e.g., `10.500`, `150.123`)
- **Stock prices**: dollar sign prefix with 4 decimal places (e.g., `$350.0000`, `$275.5000`)
- **ESPP/activity prices**: 5 decimal places (e.g., `$250.00000`, `$300.50000`)
- **Percentages**: shown as `15.000%` or `0.800%`

## Date Formats

- **Statement period (header)**: `<Month Name> <Day>, <YYYY> - <Month Name> <Day>, <YYYY>`
- **Activity settlement dates**: `MM/DD` (2-digit month/day, **no year**). Infer year from the statement period.
- **ESPP purchase dates in Stock Plans**: `MM/DD/YYYY` (full date)
- **Holdings column headers**: `<Mon> <Day>, <YYYY>` (e.g., `Mar 31, 2022`)

## Extracting Position Transactions (Monthly)

### ESPP Purchases

ESPP purchases appear in **two** locations in monthly statements:

1. **Activity → Securities Bought & Sold**: Shows settlement date, quantity, price, and amount
   ```
   Settlement Date | Security Name        | Symbol/CUSIP | Description | Quantity | Price       | Transaction Cost | Amount
   MM/DD           | <COMPANY> ESPP###    | <CUSIP>      | You Bought  | <qty>    | $<price>    | -                | -$<amount>
   ```

2. **Stock Plans → Employee Stock Purchase Summary**: Shows offering period, purchase date, purchase price, FMV, shares, gain
   ```
   Offering Period | Description       | Purchase Date | Purchase Price | FMV     | Shares | Gain
   MM/DD/YYYY-...  | Employee Purchase | MM/DD/YYYY    | $<price>       | $<fmv>  | <qty>  | $<gain>
   ```

Use the **Activity section** as the primary source (it has the settlement date). Cross-reference with ESPP Purchase Summary for the purchase date.

Map as `buy` position_transaction:
- **date**: Settlement date from Activity (MM/DD → YYYY-MM-DD using statement period year)
- **type**: `buy`
- **ticker**: Infer from security name (e.g., "MICROSOFT CORP ESPP###" → `MSFT`)
- **quantity**: From Activity row
- **price**: From Activity row (5 decimal places)
- **amount**: From Activity row (negative)
- **currency**: `USD`

**No matching cash_transaction** — ESPP purchases are payroll-funded.

**Note**: ESPP settlement date may differ from purchase date. The Q4 ESPP purchase (12/31) typically settles in the following January statement.

### Share Sales

Look for entries in **Activity → Securities Bought & Sold** with "You Sold" description. Map as `sell` position_transaction WITH a matching `sell` cash_transaction.

(No sales observed in training data yet.)

### Stock Option Exercises

Look for option exercise entries in Activity. Map as `buy` position_transactions. **No matching cash_transaction**.

(No option exercises observed in training data yet.)

### RSU Vests (from Activity → Other Activity In)

RSU vests appear as "SHARES DEPOSITED / Conversion":
```
Settlement Date | Security Name                              | Symbol/CUSIP | Description | Quantity  | Price     | Amount
MM/DD           | <COMPANY> SHARES DEPOSITED VALUE OF ...    | <CUSIP>      | Conversion  | <shares>  | $<price>  | -
```

Map as `buy` position_transaction:
- **date**: Settlement date (MM/DD → YYYY-MM-DD)
- **type**: `buy`
- **ticker**: Infer from security name (e.g., `MSFT`)
- **quantity**: From the Quantity column
- **price**: From the Price column
- **amount**: `-(quantity * price)` (compute, since Amount column shows `-`)
- **currency**: `USD`

**No matching cash_transaction** — RSU vests are employer-granted, not market purchases.

### SPP PURCHASE CREDIT — DO NOT EXTRACT

```
MM/DD           | SPP PURCHASE CREDIT                        |              | Journaled   | -         | -         | $<amount>
```

This is the ESPP payroll credit that offsets the ESPP buy. Do NOT extract as a cash_transaction — it's an internal journal entry, not a real cash deposit.

## Extracting Cash Transactions (Monthly)

Monthly statements provide **individual transaction dates** in the Activity section.

### Dividends (from Activity → Dividends, Interest & Other Income)

```
Settlement Date | Security Name              | Symbol/CUSIP | Description       | Amount
MM/DD           | MICROSOFT CORP             | <CUSIP>      | Dividend Received | $<amount>
```

Map as `dividend` cash_transaction with the exact settlement date.

### Interest (from Activity → Dividends, Interest & Other Income)

```
MM/DD           | FIDELITY GOVERNMENT CASH RESERVES | <CUSIP> | Dividend Received | $<amount>
```

The core money market fund "dividends" are actually **interest**. Map as `interest` cash_transaction.

**Note**: FDRXX interest may appear as "Dividend Received" in the Activity section but should be typed as `interest`, not `dividend`.

### Taxes Withheld (from Activity → Taxes Withheld)

```
Date  | Security         | Description       | Amount
MM/DD | MICROSOFT CORP   | Non-Resident Tax  | -$<amount>
```

Map as `tax` cash_transaction with the exact date. Amount is negative.

### Core Fund Activity — DO NOT EXTRACT separately

The **Core Fund Activity** section shows FDRXX buy/sell/reinvestment entries. These are the mechanical settlement of cash flowing into/out of the money market fund. Do NOT extract these as separate transactions — they're the plumbing behind the dividend/tax entries already captured.

## Core Account Cash Reconciliation

For monthly statements, use the **Core Account and Credit Balance Cash Flow** section's "This Period" column:

```
Beginning Balance (This Period)
+ Dividends, Interest & Other Income (This Period)
- Taxes Withheld (This Period)
= Ending Balance
```

The "Securities Bought" and "Other Activity In" amounts cancel out (ESPP/RSU offset) and do NOT affect core cash.

**Verify**: FDRXX quantity in Holdings at period end = Ending Balance in Cash Flow.

## Holdings Section (Monthly)

Monthly Holdings include **beginning and ending market values**:

### Core Account
```
Description | Beginning Market Value | Quantity (end) | Price Per Unit | Ending Market Value | Total Cost Basis | Unrealized Gain/Loss | EAI/EY
```
- Money market fund at $1.0000/unit
- May show `unavailable` for beginning market value on first statement
- Quantity = core cash balance

### Stocks
```
Description | Beginning Market Value | Quantity (end) | Price Per Unit | Ending Market Value | Total Cost Basis | Unrealized Gain/Loss | EAI/EY
```
- Shows stock positions with fractional shares
- "t" footnote = "Third-party provided" cost basis

## Lot Actions

No stock splits, mergers, or corporate actions observed in training data. The `lot_actions` array should be empty unless detected.

## Currency

- All amounts in USD
- Single-currency statements

## Edge Cases

1. **Empty/minimal statements**: First statement (account opening) may have zero holdings and no Activity section. Output empty arrays.
2. **Dash/double-dash as zero**: `-` means zero/not applicable; `--` means "not available" (used in Estimated Cash Flow).
3. **Settlement date format**: Activity dates are `MM/DD` without year. Infer year from the statement period header. For periods spanning Dec–Jan, dates in January belong to the new year.
4. **ESPP settlement vs. purchase date**: A Q4 ESPP purchase (e.g., 12/31) may settle in January of the following year. The buy appears in the January monthly statement, not the December one.
5. **Multi-month periods**: Some monthly statements cover 2 months (e.g., Feb 1 – Mar 31, May 1 – Jun 30, Aug 1 – Sep 30). All transactions within the period should be extracted.
6. **"SPP PURCHASE CREDIT / Journaled"**: This is the ESPP payroll credit in Other Activity In. It exactly offsets the ESPP buy amount. Do NOT extract as a cash_transaction.
7. **RSU vests ("SHARES DEPOSITED / Conversion")**: Appear in Other Activity In with quantity, price, and a "VALUE OF TRANSACTION" note. Extract as `buy` position_transactions. The Amount column shows `-` — compute amount as `-(quantity * price)`.
8. **FDRXX interest labeled as "Dividend Received"**: The core money market fund pays interest but Fidelity labels it "Dividend Received" in Activity. Classify as `interest`, not `dividend`.
9. **Core fund may change**: FDRXX → FYIXX across years. Use the actual fund name from Holdings.
10. **No matching cash_transaction for ESPP buys**: By design — externally funded.
11. **"ESPP###" suffix**: ESPP purchases in Activity show the security name as "MICROSOFT CORP ESPP###" with the CUSIP. The ticker is still `MSFT`.
12. **This Period vs. Year-to-Date**: Cash Flow and Income Summary show both columns. Use "This Period" for the current statement's transactions. "Year-to-Date" is cumulative.
13. **Income on positions no longer held**: May appear in annual statements as a separate line. Include in dividend total.
14. **Income Summary vs. Holdings total discrepancy**: Small rounding differences may exist. Use Activity section amounts (which have exact per-transaction values) over summary totals.
15. **Account # vs. Participant Number**: Page 2 may show an "Account #" (e.g., for a Trust account). Use the **Participant Number** from page 1 as the account_id.
