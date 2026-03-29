# Fidelity Stock Plan Account Statement Reference

> Last updated: Epoch 1 (2026-03-30) — based on 3 training + 2 test PDFs (YE2021–YE2025)

## Statement Type

These are **Fidelity Stock Plan Services Year-End Investment Reports** — annual statements covering January 1 through December 31 of a given year. They summarize a Stock Plan Account (RSU vesting, ESPP purchases, stock option exercises, dividends, and a core money-market position).

**Key difference from IBKR/Schwab**: These are annual summaries, not monthly activity statements. Transaction-level detail is limited — most data is aggregated at the annual or quarterly level.

## Fidelity-Specific Extraction Rules

These rules override the general output schema for Fidelity Stock Plan statements:

1. **position_transactions**: Capture ESPP purchases, stock option exercises, dividend reinvestment purchases (DRIP), and share sales. Do NOT create matching `cash_transaction` entries for ESPP buys or option exercises — these are funded externally (payroll deductions / exercise proceeds), not through the core cash account.
2. **cash_transactions**: Capture ONLY dividends, interest, taxes, sales proceeds, and dividend reinvestment purchases. Do NOT capture ESPP purchase cash or deposit entries.
3. **Schema rule 5 exception**: For Fidelity, ESPP buys and option exercises in `position_transactions` do NOT require matching `cash_transaction` entries. The cash for these transactions flows outside the core brokerage account.
4. **Cash reconciliation**: `Starting Core Balance + sum(cash_transactions[].amount) = Ending Core Balance`. The core balance is the FDRXX money market position shown in Holdings.

## Statement Structure

### Page Layout (typical 4–6 page statement)

| Page | Sections |
|------|----------|
| 1 | Title/Header, Participant Info, Stock Plan Account Value Summary, Contact Information |
| 2 | Account Summary — Accounts Included, Other Holdings (ESPP) |
| 3 | Account Summary (detail) — Account Value, Change Summary, Income Summary, Core Account and Credit Balance Cash Flow |
| 4 | Holdings — Core Account, Stocks (with quantities, prices, cost basis, unrealized gain/loss, income earned) |
| 5 | Stock Plans — ESPP Contribution Summary, ESPP Purchase Summary |
| 6 | (blank or disclosures) |

**Note**: Minimal/empty statements (e.g., YE2021 with no holdings) may have only 3–4 pages with no Holdings or Cash Flow sections.

## Metadata Extraction

- **Account ID**: Found as "Participant Number" on page 1 (e.g., `XXXXXXXXX`). Format: letter prefix + digits.
- **Period**: From the header: `January 1, <YYYY> - December 31, <YYYY>`. Always a full calendar year.
  - `period.from` → `YYYY-01-01`
  - `period.to` → `YYYY-12-31`
- **Broker**: `fidelity`

## Section Headers (exact text)

Bold section headers to look for:

- `Account Summary`
- `Account Holdings` (pie chart / allocation table)
- `Income Summary`
- `Core Account and Credit Balance Cash Flow`
- `Holdings`
- `Core Account` (subsection of Holdings)
- `Stocks` (subsection of Holdings)
- `Common Stock` (subsection of Stocks)
- `Stock Plans`
- `Employee Stock Purchase - MICROSOFT ESPP PLAN` (or similar company plan name)
- `Employee Stock Purchase Contribution Summary`
- `Employee Stock Purchase Summary`

## Number Formats

- **Currency prefix**: Dollar sign (e.g., `$47,234.28`)
- **Thousands separator**: comma (e.g., `47,234.28`)
- **Decimal separator**: period
- **Negative amounts**: leading minus sign, sometimes with dollar sign (e.g., `-37.43`, `-$12,741.14`). No parentheses observed.
- **Dash for zero/empty**: A single dash `-` represents zero or no value (not a negative sign)
- **Stock quantities**: decimal precision to 3 places (e.g., `196.487`, `378.206`)
- **Stock prices**: dollar sign prefix with 4 decimal places (e.g., `$239.8200`, `$376.0400`)
- **ESPP purchase prices**: 5 decimal places, dollar sign prefix (e.g., `$277.48000`)
- **Percentages**: shown as `15.000%`

## Date Formats

- **Statement period (header)**: `January 1, <YYYY> - December 31, <YYYY>`
- **Account value dates**: `Jan 1, <YYYY>` and `Dec 31, <YYYY>` (abbreviated month)
- **ESPP offering periods**: `MM/DD/YYYY-MM/DD/YYYY` (e.g., `01/01/2022-03/31/2022`)
- **ESPP purchase dates**: `MM/DD/YYYY` (e.g., `03/31/2022`)
- **Income Summary column header**: `Dec 31, <YYYY>`

## Core Account Cash Reconciliation

The core cash balance is the FDRXX money market position in the Holdings section. Reconcile using:

```
Starting Core Balance (FDRXX from prior year, or 0)
+ Dividends (from Holdings → Stocks → Income Earned)
+ Interest (from Holdings → Core Account → Income Earned)
- Taxes Withheld (from Account Summary → Subtractions)
+ Sales proceeds (if any share sales occurred)
- Dividend reinvestment purchases (if DRIP active)
= Ending Core Balance (FDRXX quantity in Holdings)
```

**Important**: The "Core Account and Credit Balance Cash Flow" section on page 3 shows ALL account value flows (including "Securities Bought" and "Other Activity In" for share movements), but these do NOT all flow through core cash. Only dividends, interest, taxes, sales, and DRIP purchases affect the core cash balance.

**Note**: This section only appears in years with activity. Empty statements (e.g., YE2021) have no Cash Flow section.

## Extracting Position Transactions

### ESPP Purchases (from Stock Plans section)

The **Employee Stock Purchase Summary** table provides quarterly ESPP purchases:

```
Offering Period | Description | Purchase Date | Purchase Price | Purchase Date Fair Market Value | Shares Purchased | Gain from Purchase
```

Map each row as a `buy` position_transaction:
- **date**: Purchase Date column → convert `MM/DD/YYYY` to `YYYY-MM-DD`
- **type**: `buy`
- **ticker**: `MSFT` (inferred from plan name "MICROSOFT ESPP PLAN" and Holdings section)
- **quantity**: Shares Purchased column
- **price**: Purchase Price column (the discounted ESPP price)
- **amount**: `-(quantity * price)` (negative = cash outflow)
- **currency**: `USD`

**No matching cash_transaction** — ESPP purchases are payroll-funded, not from core cash.

### Stock Option Exercises

Stock option exercises may appear in the statement when options are exercised. Look for sections related to stock options. Map as `buy` position_transactions. **No matching cash_transaction** — exercise cash flows outside the core account.

(No stock option exercises observed in training data yet.)

### Share Sales

If shares are sold, they will appear as position_transactions with type `sell` AND a matching `sell` cash_transaction (sales proceeds flow into the core cash account).

(No sales observed in training data yet.)

### RSU Vests / Stock Plan Transfers

The "Other Activity In" line in Cash Flow represents shares deposited from RSU vesting. These are NOT captured as position_transactions or lot_actions — individual vest dates, quantities, and prices are not available in the annual statement. The share growth is visible by comparing Holdings across years.

### Dividend Reinvestments (DRIP)

When dividends are reinvested (indicated by the "D" footnote), the reinvestment is a `buy` position_transaction. However, individual reinvestment dates and quantities are not listed in the annual statement.

**Strategy**: If `Securities Bought` in the Cash Flow exceeds the sum of ESPP purchase amounts, the difference may represent DRIP purchases. Record as a single `buy` cash_transaction (negative, reducing core cash) with date `YYYY-12-31` if the amount is material. The position_transaction cannot be precisely dated.

## Extracting Cash Transactions

Only transactions that affect the **core cash account** (FDRXX) are recorded as cash_transactions:

1. **Dividends** → `dividend` type (from Holdings → Stocks → Income Earned)
2. **Interest** → `interest` type (from Holdings → Core Account → Income Earned)
3. **Taxes Withheld** → `tax` type (negative, from Account Summary → Subtractions)
4. **Sales proceeds** → `sell` type (positive, if any share sales occurred)
5. **DRIP purchases** → `buy` type (negative, if dividend reinvestments are material)

**NOT included as cash_transactions**: ESPP purchases, stock option exercises, RSU vest deposits — these flow outside core cash.

### Date Assignment

Individual transaction dates are generally NOT available in these annual statements. Use these conventions:
- **Dividends**: Use `YYYY-12-31` (period end date)
- **Interest**: Use `YYYY-12-31` (period end date)
- **Taxes Withheld**: Use `YYYY-12-31` (period end date)
- **Sales**: Use specific date if available, otherwise `YYYY-12-31`
- **DRIP purchases**: Use `YYYY-12-31`

### Income Breakdown

The **Income Summary** section and the **Holdings** table together provide the income split:

| Source | Location | Type |
|--------|----------|------|
| MSFT dividends | Holdings → Stocks → Income Earned | `dividend` |
| FDRXX interest | Holdings → Core Account → Income Earned | `interest` |

Example from YE2023: Total Income $788.42 = MSFT $772.50 (dividend) + FDRXX $15.92 (interest)

## Holdings Section

The **Holdings** page provides end-of-period positions:

### Core Account
```
Description | Quantity | Price Per Unit | Total Market Value | Total Cost Basis | Unrealized Gain/Loss | Income Earned
```
- Money market fund at $1.0000/unit. The fund may change over time:
  - `FIDELITY GOVERNMENT CASH RESERVES (FDRXX)` (observed 2022–2024)
  - `FID TREASURY ONLY MMKT FUND CL OUS (FYIXX)` (observed 2025)
- Quantity = cash balance in the core account
- Income Earned = interest earned during the period

### Stocks
```
Description | Quantity | Price Per Unit | Total Market Value | Total Cost Basis | Unrealized Gain/Loss | Income Earned
```
- Shows `MICROSOFT CORP (MSFT)` with fractional shares
- Income Earned = dividends received during the period
- "t" footnote means "Third-party provided" (cost basis from employer)

## Lot Actions

No stock splits, mergers, or corporate actions have been observed in the training data. The `lot_actions` array should be empty unless such events are detected.

## Currency

- All amounts in USD
- Single-currency statements

## Edge Cases

1. **Empty/minimal statements**: YE2021 shows zero holdings, zero activity, no Cash Flow section. Output should have empty `position_transactions`, `cash_transactions`, and `lot_actions` arrays.
2. **Dash as zero**: A single `-` in value columns means zero/not applicable, not a negative amount.
3. **"Other Activity In"**: Large positive amounts represent RSU vesting or stock plan deposits. These are NOT captured — individual vest events are not itemized in the annual statement.
4. **Dividend reinvestment (DRIP)**: The "D" footnote on "Dividends, Interest & Other Income" indicates dividend reinvestments. Record gross dividend as `dividend` cash_transaction. If DRIP is active, the reinvestment buy reduces core cash — record as `buy` cash_transaction if material.
5. **ESPP purchases vs. Securities Bought**: "Securities Bought" in Cash Flow includes ESPP purchases + DRIP + other buys. Only ESPP purchases are individually documented in the ESPP Purchase Summary.
6. **No individual transaction dates**: Annual reports don't list individual dates for dividends, interest, or taxes. Use `YYYY-12-31` as fallback.
7. **Core Account balance = Starting Cash**: The FDRXX quantity in Holdings IS the core cash balance. Use for cash reconciliation.
8. **Year-over-year share quantity changes**: Share growth comes from ESPP + RSU vests + DRIP. Only ESPP purchases are individually documented.
9. **Statement period always full year**: Jan 1 – Dec 31.
11. **ESPP purchase dates vary**: Q1 may be 03/28, 03/31, etc. Always use the exact Purchase Date from the table, not the offering period end date.
12. **Core fund may change**: The money market fund can switch between FDRXX and FYIXX (or others) across years. Always use the fund shown in Holdings for descriptions.
13. **Income on positions no longer held**: YE2025 introduced a line "Total income earned on positions no longer held: $12.50". This is dividend/interest from securities sold during the year. Include in the dividend cash_transaction total.
14. **Income Summary vs. Holdings total discrepancy**: The Income Summary total may differ slightly from the sum of Holdings Income Earned columns (by $0.02–$0.11) due to rounding or income on sold positions. Use Income Summary as the authoritative total, and derive the dividend amount as: `Income Summary Total - Core Account Interest`.
10. **Taxes Withheld sources**: "Subtractions → Taxes Withheld" in Account Summary matches "Taxes Withheld" in Cash Management Activity. Use either.
11. **No matching cash_transaction for ESPP/option buys**: This is by design — these are externally funded. Schema rule 5 does not apply to Fidelity ESPP and option exercise position_transactions.
12. **Stock options**: May appear in future statements. Exercise events should be captured as `buy` position_transactions without matching cash_transactions.
