# DBS Singapore (Treasures) Statement Reference

> Last updated: Epoch 1 (2026-03-30) — based on 7 training PDFs (Jan–Jun 2025, Feb 2026)

## Statement Structure

DBS Treasures "Investment Statement" — monthly consolidated wealth management statements. Title page has "DBS Treasures" logo and "INVESTMENT STATEMENT".

### Page Layout (typical 8–12 page statement)

| Page | Sections |
|------|----------|
| Cover | DBS Treasures branding, customer address |
| 01 | Overview: Table of Contents + Your Financial Profile (investment risk profile) |
| 02 | Portfolio ESG Rating |
| 03 | Portfolio Summary: asset allocation by currency (SGD equivalent) |
| 04 | Income and Expense Summary + Fixed Income Coupon and Redemption projections |
| 05-06 | Portfolio Details: "Your Total Asset" (cash by currency) + "Your Total Equity" (holdings) |
| 07-08 | **Transactional Details**: Cash Transactions (per currency account) + Equity Transactions |
| 09+ | Additional Information: Exchange rates, disclaimers, special notes |

## Metadata Extraction

- **Broker**: `dbs-sg`
- **Account ID**: Found on every page header → "Portfolio Number" field (e.g., `S-XXXXXX-X`)
- **Period**: From the "Statement as of:" field in section headers
  - Format: `DD-MMM-YYYY` (e.g., "31-JAN-2025")
  - Period **from**: first day of that month (e.g., `2025-01-01`)
  - Period **to**: the "Statement as of" date (e.g., `2025-01-31`)
  - The "Balance carried forward" date is the previous month-end = period_from - 1 day

## Number Formats

- **Thousands separator**: comma (e.g., `20,942.96`, `14,310.91`)
- **Decimal separator**: period
- **Negative amounts**: minus sign prefix (e.g., `-179.20`, `-505.80`) — seen in Unrealized P/L only
- **Currency**: Multi-currency account; amounts shown with currency code prefix in Portfolio Details, plain numbers in Transactional Details
- **Cash transactions use Debit/Credit columns** (not signed amounts):
  - **Debit** column = cash outflow (expenses, fees, purchases)
  - **Credit** column = cash inflow (interest, dividends, sale proceeds)

## Date Formats

- **All dates**: `DD-MMM-YYYY` format (e.g., `31-JAN-2025`, `28-FEB-2025`, `01-MAR-2025`)
- Months abbreviated to 3 uppercase letters: JAN, FEB, MAR, APR, MAY, JUN, JUL, AUG, SEP, OCT, NOV, DEC

## Key Sections for Data Extraction

### Transactional Details — Cash Transaction

The primary section for cash data. Organized **per currency account**.

Section header: `"Your Transactional Details"` with `"Statement as of: DD-MMM-YYYY"`
Sub-header: `"Cash Transaction :"`
Account header: `"Account No. MCSA S-XXXXXX-X-<CCY>-1"` (e.g., `S-XXXXXX-X-SGD-1`, `S-XXXXXX-X-USD-1`)

Table columns:
```
Trans. Date | Value Date | Ref. No. | Transaction Type | Transaction Details | Debit | Credit | Balance
```

Special rows (NOT transactions — do not extract):
- First row: `"Balance carried forward"` — this is the starting cash balance
- Last row: `"Balance in your favour"` — this is the ending cash balance
- `"Total"` row — shows sum of Debit and Credit columns

**Cash Reconciliation** per currency account:
`Balance carried forward + sum(Credits) - sum(Debits) = Balance in your favour`

### Transactional Details — Equity Transaction

Section sub-header: `"Equity Transaction :"`

Table columns:
```
Transaction Date / Transaction Ref. | Value Date / Transaction Type | Security Code | Description | Quantity/Not. Amt. / Price | Currency / Settlement Amt.
```

- Each equity transaction occupies 2 rows (data stacked vertically)
- Row 1: Transaction Date, Value Date, Security Code, Description (line 1), Quantity/Not. Amt., Currency
- Row 2: Transaction Ref., Transaction Type, (blank), Description (line 2), Price, Settlement Amt.

## Transaction Type Mapping

### Cash Transaction Types → Schema Types

| Transaction Type (in PDF) | → Schema Type | Notes |
|---------------------------|---------------|-------|
| Interest payment | `interest` | Credit; small amounts on cash balances |
| Custody Fee | `fee` | Debit; semi-annual custody fees (SGD account) |
| Sell | `sell` | Credit; cash proceeds from equity sale |
| Buy | `buy` | Debit; cash spent on equity purchase |
| Dividend Cash | `dividend` | Credit; cash dividend received |
| Deposit | `deposit` | Credit; funds deposited |
| Withdrawal | `withdrawal` | Debit; funds withdrawn |
| Forex (Quoted Rate) | `forex` | Debit or Credit; FX conversion at quoted rate |
| Forex (Board Rate) | `forex` | Debit or Credit; FX conversion at board rate |
| Client Money Transfer | `withdrawal` | Debit; transfer out to another account/person |

### Equity Transaction Types → Schema Types

| Transaction Type (in PDF) | → Schema Type | Notes |
|---------------------------|---------------|-------|
| Sell | `sell` → `position_transaction` | Has quantity, price, settlement amount |
| Buy | `buy` → `position_transaction` | Has quantity, price, settlement amount |
| Client transfer IN | `transfer` → `lot_action` | No price, no settlement; shares transferred in |
| Client transfer OUT | `transfer` → `lot_action` | No price, no settlement; shares transferred out |

## Extracting Position Transactions

Only extract from **Equity Transactions** where Transaction Type is `"Sell"` or `"Buy"`.

### Sell Transactions
- **date**: Transaction Date from equity transaction (DD-MMM-YYYY → YYYY-MM-DD)
- **type**: `sell`
- **ticker**: Security Code (e.g., `CRM UN` → use `CRM`; `NVDA UW` → use `NVDA`)
- **quantity**: Quantity/Not. Amt. field (always positive)
- **price**: Price field
- **amount**: Settlement Amt. (positive for sells — cash inflow)
- **currency**: Currency field

### Buy Transactions
- **date**: Transaction Date (DD-MMM-YYYY → YYYY-MM-DD)
- **type**: `buy`
- **ticker**: Security Code → extract base ticker before space
- **quantity**: Quantity/Not. Amt. (always positive)
- **price**: Price field
- **amount**: negative of Settlement Amt. (negative for buys — cash outflow)
- **currency**: Currency field

### Ticker Extraction from Security Code
DBS uses exchange-suffixed security codes:
- `NVDA UW` → `NVDA` (NASDAQ)
- `CRM UN` → `CRM` (NYSE)
- `SHOP CT` → `SHOP` (TSX/Toronto)
- `SHOP UN` → `SHOP` (NYSE)
- `TTD UQ` → `TTD` (NASDAQ)
- `MELI UW` → `MELI` (NASDAQ)
- `ISRG UW` → `ISRG` (NASDAQ)
- `DDOG UW` → `DDOG` (NASDAQ)

Rule: Take the part before the first space as the ticker.

**Important**: The same ticker (e.g., SHOP) can appear on different exchanges (SHOP CT on TSX, SHOP UN on NYSE). Use the full security code in the description to distinguish.

## Extracting Cash Transactions

Extract from **Cash Transactions** section for EACH currency account.

For each transaction row (skip "Balance carried forward", "Balance in your favour", and "Total" rows):

- **date**: Trans. Date (DD-MMM-YYYY → YYYY-MM-DD)
- **type**: Map Transaction Type per table above
- **description**: Transaction Type + " " + Transaction Details (concatenate)
- **amount**:
  - If Credit column has a value: positive (inflow)
  - If Debit column has a value: negative (outflow)
- **currency**: From the account header (e.g., `S-XXXXXX-X-SGD-1` → `SGD`, `S-XXXXXX-X-USD-1` → `USD`)

### Trade Cash Impact
When an equity Sell or Buy appears in cash transactions, it already has the settlement amount. This is the cash impact of the trade. Extract it as a `sell` or `buy` type cash_transaction.

**Date alignment**: The cash Trans. Date may differ from the equity Transaction Date by 1–2 business days (settlement lag). For the cash_transaction entry, use the **cash Trans. Date** (this ensures cash reconciliation works). For the position_transaction, use the **equity Transaction Date**. Document that these may not match exactly.

## Extracting Lot Actions

"Client transfer IN" and "Client transfer OUT" equity transactions map to lot_actions:

- **date**: Transaction Date (YYYY-MM-DD)
- **type**: `transfer`
- **ticker**: Security Code → base ticker
- **description**: "Client transfer IN/OUT, " + Description from equity transaction
- **ratio_from**: `null` (not applicable for transfers)
- **ratio_to**: `null`
- **currency**: Currency field

## Currency Scope

**Extract USD and SGD transactions only.** DBS is a multi-currency account with SGD, USD, CAD, and other currency ledgers. We track the two active accounts:
- `MCSA S-XXXXXX-X-USD-1` — primary investment account
- `MCSA S-XXXXXX-X-SGD-1` — holds custody fees, SGD interest, forex credits

Skip zero-balance currencies (AUD, CAD, EUR, GBP, HKD, JPY) entirely.

## Cash Reconciliation

Cash reconciliation is verified **per currency account**:

- `USD Balance carried forward + sum(cash_transactions[currency=USD].amount) = USD Balance in your favour`
- `SGD Balance carried forward + sum(cash_transactions[currency=SGD].amount) = SGD Balance in your favour`

Both must reconcile independently. The SGD account can go negative (e.g., when custody fees exceed the SGD balance).

## Income and Expense Summary (Cross-Check Only)

Page 04 shows an Income and Expense Summary with:
- Cash Dividend Received (Month to Date / Year to Date)
- Interest Received
- Custody Account and Account Fees
- Transaction Cost
- Interest Paid

Use this as a cross-check against extracted cash_transactions, but do NOT extract data from here — the Transactional Details section is the authoritative source.

## Edge Cases

1. **Multi-currency accounts**: Each currency has its own cash ledger. Extract transactions from ALL currency accounts.
2. **Client transfers**: These are in-kind transfers of securities with no cash impact and no price. Map to `lot_actions` with type `transfer`.
3. **Settlement lag**: Equity trade dates may be 1–2 business days before the cash settlement date. Use equity Transaction Date for position_transactions and cash Trans. Date for cash_transactions.
4. **Settlement amount vs quantity × price**: The settlement amount may differ from quantity × price due to commissions/fees embedded in the settlement. Use the Settlement Amt. as the amount (it's the actual cash impact).
5. **Months with no SGD/other currency transactions**: If a currency account shows only "Balance carried forward" = "Balance in your favour" with no transaction rows, skip that account entirely.
6. **Same ticker on different exchanges**: SHOP appears as both SHOP CT (Toronto) and SHOP UN (NYSE). Both map to ticker `SHOP` but with different currencies (CAD vs USD).
7. **Transaction Details column for Sell/Buy in cash**: Shows "Ordinary Share, Issuer: <Company Name>" — use this as the description.
8. **Custody Fee description**: Includes the portfolio number and period covered (e.g., "S-XXXXXX-X (Client Portfolio) &lt;ACCOUNT_HOLDER&gt; (Period covered from 01-JUL-2024 to 31-DEC-2024)").
9. **No Debit/Credit on balance rows**: "Balance carried forward" and "Balance in your favour" rows have only the Balance column filled — skip these.
10. **February 2026 has no SGD account section**: When a currency account has zero balance and no activity, it may be omitted entirely from the Transactional Details.
