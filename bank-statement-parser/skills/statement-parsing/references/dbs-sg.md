# DBS Singapore (Treasures) Statement Reference

> Last updated: Epoch 2 (2026-07-20) — Epoch 1 based on 7 training PDFs (Jan–Jun 2025, Feb 2026);
> Epoch 2 adds learnings from a 306-statement production run across 4 portfolios and 6 financial years
> (FY2020-21 → FY2025-26), covering active, dormant, joint, and wound-down accounts.

## ⚠️ Read First — the three rules that break parses

1. **Never infer an amount's sign from its transaction type.** Interest is *not* always a credit —
   DBS charges interest as a **Debit** whenever the ledger balance is negative/overdrawn. Derive
   every sign from which column the number sits in (Debit vs Credit) or from the running-balance
   delta. See [Amount Signs](#amount-signs-critical).
2. **Two accounts share one PDF.** The Wealth Management (brokerage) account and the MCSA
   (multi-currency deposit) account are *different accounts* and must not be merged into one record.
   See [Account Model](#account-model-wealth-management-vs-mcsa).
3. **Exclude pending settlement rows.** Anything under "Pending settlement transactions" (below
   "Balance in your favour") settles next month; including it double-counts.
   See [Pending Settlement](#pending-settlement-transactions).

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

## Account Model: Wealth Management vs MCSA

A single DBS Treasures PDF is a **consolidated statement covering two linked but distinct accounts**:

| | Wealth Management Account | MCSA (Multi-Currency Settlement Account) |
|---|---|---|
| **Nature** | Brokerage / custody — holds securities | **Deposit account** — holds cash, per currency |
| **Statement sections** | "Your Total Equity" (holdings), "Equity Transaction" | "Your Total Asset" (cash by currency), "Cash Transaction" |
| **Produces** | `position_transactions` (buy/sell), `lot_actions` (transfers, splits) | `cash_transactions` (per currency) |
| **Account number** | the Portfolio Number `S-XXXXXX-X` | per-currency sub-accounts `S-XXXXXX-X-<CCY>-1` |

**Do not merge these into one record.** They are different account types with different tax and
reporting treatment (interest/fees on a deposit account vs capital gains on a brokerage account).
Emit them as two records, distinguished by `account_type`:
- Wealth Management → `account_type: "wealth_management"` (positions + lot actions, no cash ledger)
- MCSA → `account_type: "mcsa_deposit"` (cash ledger only, currency-tagged)

**Consequence for validation:** the usual "every position_transaction has a matching cash_transaction"
rule does **not** hold *within* a single record here. A buy/sell is recorded on the Wealth Management
side; its settlement cash lands in the MCSA record for the same statement (often 1–2 days later, see
[Settlement lag](#trade-cash-impact)). Validate the pairing **across** the two records, not inside one.

### Out of scope: the bundled banking account

Some statements (notably joint portfolios) additionally bundle a **retail banking statement** for a
**DBS eMulti-Currency Autosave account, numbered `113-XXXXXX-X`**, with its own small interest lines.
This is a *savings* account, distinct from the investment MCSA, and is **not** part of this parse.
Only extract the MCSA ledgers found under "Your Transactional Details". Do not confuse the two
just because both are "multi-currency".

## Metadata Extraction

- **Broker**: `dbs-sg`
- **Account ID**: Found on every page header → "Portfolio Number" field (e.g., `S-XXXXXX-X`)
- **Period**: From the "Statement as of:" field in section headers
  - Format: `DD-MMM-YYYY` (e.g., "31-JAN-2025")
  - Period **from**: first day of that month (e.g., `2025-01-01`)
  - Period **to**: the "Statement as of" date (e.g., `2025-01-31`)
  - The "Balance carried forward" date is the previous month-end = period_from - 1 day

## Number Formats

- **Thousands separator**: comma (e.g., `12,345.67`, `98,765.43`)
- **Decimal separator**: period
- **Negative amounts**: minus sign prefix (e.g., `-179.20`, `-505.80`) — seen in Unrealized P/L only
- **Currency**: Multi-currency account; amounts shown with currency code prefix in Portfolio Details, plain numbers in Transactional Details
- **Cash transactions use Debit/Credit columns** (not signed amounts):
  - **Debit** column = cash outflow → negative amount
  - **Credit** column = cash inflow → positive amount

## Amount Signs (CRITICAL)

The Debit/Credit columns carry the sign — the *transaction type does not*. Deriving signs from type
assumptions is the single most common cause of failed reconciliation on DBS statements.

**Rule: determine every amount's sign from (a) which column the number occupies, cross-checked
against (b) the running-balance delta and (c) the "Total" row's Debit/Credit subtotals. Never from
the transaction type.**

Real cases that violate naive type-based assumptions:

| Case | Naive assumption | Reality |
|---|---|---|
| "Interest payment" on an **overdrawn/negative balance** | credit (income) | **Debit** — interest *charged*. Seen repeatedly on SGD ledgers after a custody fee pushes the balance negative, and on USD ledgers during negative-rate periods. |
| "Client Money Transfer" | withdrawal (outflow) | **Either direction.** Credit = funding the account (`deposit`); Debit = transfer out (`withdrawal`). |
| "Custody Fee" | always a debit | Usually a debit, but **reversals and waivers post as credits** (positive `fee`). |

A robust implementation reads the running `Balance` column and takes
`amount = balance[row] - balance[row-1]`, then sanity-checks that against the Debit/Credit column
placement. This self-corrects for every case above.

## Extraction Technique

**Flat text extraction scrambles the Debit / Credit / Balance columns** — the three numbers collapse
into an ambiguous run and it becomes impossible to tell an inflow from an outflow. Use
**coordinate-based extraction** (e.g. `pdfplumber` word objects with x-positions) and bucket each
number by its right-edge x-coordinate.

Column x-positions vary slightly by statement vintage; derive them per file rather than hardcoding.
Observed right-edge clusters: Debit ≈ 627–645, Credit ≈ 706–726, Balance ≈ 775–802. Locate them by
finding the header row ("Debit", "Credit", "Balance") and reusing those x-positions for the rows below.

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
- Closing row: `"Balance in your favour"` — this is the ending cash balance
- `"Total"` row — shows sum of Debit and Credit columns
- `"Balance (including pending settlement transactions)"` — a *different*, non-authoritative closing
  figure. Do **not** use it as ending cash. See [Pending Settlement](#pending-settlement-transactions).

**Closing-row variant (multi-currency statements):** when a statement contains more than one currency
sub-account, the **first** sub-account may close with a row labelled simply `"Balance"` rather than
`"Balance in your favour"`. Handle both labels or you will fail to find the closing balance.

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

**The "Direction" column below tells you where to *look*, never what to assume.** Always take the
sign from the Debit/Credit column — see [Amount Signs](#amount-signs-critical).

| Transaction Type (in PDF) | → Schema Type | Direction | Notes |
|---------------------------|---------------|-----------|-------|
| Interest payment | `interest` | **Either** | Credit when earned; **Debit when the balance is negative** (interest charged) or during negative-rate periods. Do not assume. |
| Custody Fee | `fee` | **Either** | Semi-annual custody fee (accrued Jan–Jun and Jul–Dec, debited in Aug and Feb). Reversals and "Waiver" credits post as **positive** `fee`. Charged in SGD or USD depending on portfolio. |
| Sell | `sell` | Credit | Cash proceeds from equity sale |
| Buy | `buy` | Debit | Cash spent on equity purchase |
| Dividend Cash | `dividend` | Credit | Cash dividend received |
| Deposit | `deposit` | Credit | Funds deposited |
| Withdrawal | `withdrawal` | Debit | Funds withdrawn |
| Forex (Quoted Rate) | `forex` | Either | FX conversion at quoted rate — posts as a **pair** (debit in one currency, credit in the other) |
| Forex (Board Rate) | `forex` | Either | FX conversion at board rate — also a pair |
| Client Money Transfer | `deposit` **or** `withdrawal` | **Either** | **Direction-dependent.** Credit ("By order of …", funding the account) → `deposit`. Debit (transfer out to another account/person) → `withdrawal`. Choose from the column, not the label. |
| Transfer … (OET … Commission Rebate Promotion) | `other` | Credit | Promotional brokerage commission rebate; no better schema fit |
| Transfer … (Waiver) | `fee` | Credit | Fee waiver credit that clears an overdrawn balance |
| Cash merger / buyout proceeds | `other` | Credit | Cash consideration from a cash merger (e.g. ZEN at $77.50/sh). Pair with a `merger` **lot_action**. See [Corporate Actions](#corporate-actions-in-equity-transactions). |

### Equity Transaction Types → Schema Types

| Transaction Type (in PDF) | → Schema Type | Notes |
|---------------------------|---------------|-------|
| Sell | `sell` → `position_transaction` | Has quantity, price, settlement amount |
| Buy | `buy` → `position_transaction` | Has quantity, price, settlement amount |
| Client transfer IN | `transfer` → `lot_action` | No price, no settlement; shares in → **positive** `quantity` |
| Client transfer OUT | `transfer` → `lot_action` | No price, no settlement; shares out → **negative** `quantity` |
| Receive free of payment | `transfer` → `lot_action` | Inbound in-kind delivery (FOP); same treatment as Client transfer IN |
| Delivery free of payment | `transfer` → `lot_action` | Outbound in-kind delivery; negative `quantity` |
| Split | `split` → `lot_action` | Quantity credited, **no price and no settlement amount**. See below. |

**DBS prints no sign on Equity Transaction rows** — the Quantity column is unsigned for both directions.
Derive the sign from the Transaction Type (IN/receive → positive, OUT/delivery → negative) and confirm
against the holding in "Your Total Equity" before and after. Always populate `lot_actions.quantity`;
never leave the count only in the description.

## Corporate Actions in Equity Transactions

### Stock splits
Splits appear as an Equity Transaction row with Transaction Type `Split`, a **quantity** (the shares
*added*), and **no price / no settlement amount** — that absence is how you distinguish a split from a
trade. Map to a `split` lot_action with no cash impact.

**The statement usually does not print the ratio.** Set `ratio_from`/`ratio_to` to `null` and record
the credited share count in `quantity`, unless the ratio is unambiguous from the holdings
(pre/post quantity in "Your Total Equity"). Prefer statement-grounded nulls over inferred ratios.

DBS splits print the **additional** shares credited — a 4-for-1 on a 100-share holding prints `+300`,
taking the position to 400. Always confirm the increment against the pre/post holding in
"Your Total Equity". Verified across TTD, NVDA, ISRG and SHOP splits.

### Multi-listed securities — never merge on ISIN

The same ISIN can be held as **separate positions on different exchanges**, each with its own
quantity, holding row and transaction reference. Shopify (ISIN `CA82509L1076`) appears as
`SHOP CT` (Toronto), `SHOP UN` (NYSE) and `SHOP UQ` (Nasdaq) — the June-2022 10-for-1 split posted as
**two independent rows with distinct transaction references**, each crediting the increment for its
own listing. Keep the full exchange-suffixed code in `security_code` and treat each listing as its own position.
Merging them corrupts both the split and the transfer arithmetic. Note also that a Toronto row may
settle in **CAD** while every other row on the page is USD.

### Merger quirks seen in DBS statements

- **Not always 1:1.** The June-2020 Match Group reorganisation printed the surrendered leg labelled
  with the raw ISIN `US57665R1068` and issuer "Match Group Inc/old" (no suffixed ticker), and a
  slightly **larger** received quantity as `MTCH UW` — e.g. 100 out, 103 in. Record both legs exactly
  as printed; do not normalise the asymmetry away.
- **A cash acquisition has only one leg.** The November-2022 Zendesk buyout printed a single `Merger`
  row removing the entire `ZEN UN` position with no security received; the consideration arrives as a
  cash credit (type `other`). Confirm the direction from the holdings table — the position is present
  the prior month and absent afterwards.
- DBS prints no sign on merger rows either; infer direction from the old/new issuer naming and verify
  against "Your Total Equity".

Observed: TTD 10-for-1 (Jun 2021), NVDA 4-for-1 (Jul 2021), ISRG 3-for-1 (Oct 2021),
SHOP 10-for-1 (Jun 2022, hitting **both** SHOP CT and SHOP UN lots), NVDA 10-for-1 (Jun 2024).

### Cash mergers
A cash buyout produces **two** records: a `merger` lot_action (position removed, no cash) *and* a
cash-ledger credit for the consideration. The cash side has no `merger` type in the cash enum — use
`other` and describe it. Observed: ZEN (Zendesk) taken private Nov 2022 at $77.50/share.

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

**Important**: The same ticker (e.g., SHOP) can appear on different exchanges (SHOP CT on TSX, SHOP UN on NYSE). Use the full security code in the description to distinguish. When a corporate action or transfer touches both listings, expect **two separate rows** that both map to ticker `SHOP`.

**Security Code may be an ISIN instead of a ticker.** Some instruments print a bare ISIN with no
exchange-suffixed symbol — e.g. Match Group as `US57665R1068`. Record the ISIN as the ticker as-is and
note it in the description rather than guessing a symbol. (Match Group later appears as `MTCH` after
its 2020 reorganisation, so the same holding can change identifier mid-history.)

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

### Pending Settlement Transactions

Some statements append a **`"Pending settlement transactions:"`** block *after* the
"Balance in your favour" row, followed by a `"Balance (including pending settlement transactions)"`
line. These rows have a **value date in the following month** and have **not** settled.

**Exclude them.** They are outside "Balance in your favour", so including them breaks reconciliation —
and they reappear as settled rows in the *next* month's ledger, so including them here also
double-counts. When the row settles next month, record it there using its **cash Trans. Date** (which
may be back-dated into the prior month — that is correct and expected).

How to tell you have this case: the ledger reconciles to "Balance in your favour" only when the
pending rows are excluded, and next month's "Balance carried forward" equals *that* figure — **not**
the "including pending" figure.

Observed: pending NVDA dividends (Jun 2021, Jun 2022, Mar 2026), pending custody fees
(Jan 2022, Aug 2023). A month whose only activity is pending therefore has **zero** settled
transactions for that currency.

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
10. **A currency ledger may be omitted entirely**: when a currency account has zero balance and no activity, DBS drops that section from the Transactional Details. Common for SGD in quiet months. Absence ≠ error; just extract the currencies that are present.
11. **A whole statement may have NO "Transactional Details" section.** Quiet and dormant months stop after "Portfolio Details". This is very common (in the production run, ~9 of 15 months on a wound-down account). Emit **empty** `cash_transactions`/`position_transactions`/`lot_actions` — an empty statement is a valid parse, not a failure. Reconciliation is trivially satisfied: confirm the Portfolio Details cash figure is unchanged month-over-month and record that in `_note` (there are no ledger rows to sum).
12. **Fully dormant accounts**: a portfolio can sit at a static token balance for years (e.g. USD 0.09 across 60+ consecutive statements) with no holdings and no ledger. Parse them anyway — the continuity is itself evidence.
13. **Overdrawn ledgers are normal**: a custody fee can push a currency ledger negative and it may stay negative for months, accruing charged interest (a Debit). Do not treat a negative "Balance in your favour" as a parse error.
14. **Account-consolidation events**: an in-kind account move appears as a batch of `Client transfer OUT` rows in the source portfolio and a matching batch of `Client transfer IN` rows in the destination portfolio's statement for the same month, plus a separate cash `Client Money Transfer`. Securities carry **no cash impact**; only the cash transfer line does.
15. **Statement titling can change mid-history**: a portfolio may be retitled (individual → joint, or joint holder list changing) while keeping the **same Portfolio Number**. Key on the Portfolio Number, not the names.
16. **Filenames are unreliable** — see the cross-broker rule in `SKILL.md`. Observed on DBS: files labelled one month early (`2024 01(1).pdf` = January, `2024 01.pdf` = February, with no `2024 02` file), and byte-identical duplicate PDFs of the same month. **Always take the period from the "Statement as of" date inside the document**, and flag suspected duplicates in `_note` rather than silently overwriting.
