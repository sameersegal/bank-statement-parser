# Charles Schwab Statement Reference

> Last updated: Epoch 2 (2026-07-20) — Epoch 1 based on 8 training PDFs (Sep–Dec 2025, 2 accounts);
> Epoch 2 adds learnings from a ~150-statement production run (FY2019-20 → FY2025-26) across
> individual, joint, DRIP, and wound-down accounts.

## ⚠️ Period comes from the statement, never the filename

Two real failure modes seen in production:

1. **Swapped contents**: a file named for April contained the May statement and vice versa. The
   parsed periods were correct only because they were read from the document header.
2. **Quarterly statements**: a file dated `2022-06-30` covered **April 1 – June 30**, not just June.
   This looked like "missing April and May statements" until the period was read from the body — a
   coverage gap that did not actually exist.

Always parse the "Statement Period" from the header, and reconcile apparent gaps against parsed
periods before reporting them. Record any filename/content mismatch in `_note`.

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

- **Thousands separator**: comma (e.g., `123,456.78`)
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
| Other Activity | Forward Split | `lot_action` type=split | Same as Stock Split. Often printed as **two lines** (old lot removed, new lot added) — record as **one** split, not two |
| Other Activity | Journaled Shares | `lot_action` type=transfer | In-kind transfer to/from another Schwab account; **no cash impact** |
| Other Activity | Security Transfer | `lot_action` type=transfer | Same treatment as Journaled Shares |
| Other Activity | Adjust Position | `tax` (usually) | NRA withholding **corrections/refunds** — see below |

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
- **quantity**: Par value (e.g., `100,000.0000`)
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

**A pending item may never post at all.** One December statement listed a pending dividend payable
in early January; the January statement's income summary and transaction detail showed interest only —
it never appeared. So do not "carry forward" pending rows on faith: record them only when they
actually show up as settled, and never pre-book them to make a balance work.

**Trade-date vs settlement-date across a period boundary.** A purchase with a trade date of 07/29 that
settles 08/01 belongs to the **July** statement under the trade-date convention, even though the cash
moves in August. Expect a trade whose date precedes the statement period and reconcile against the
statement's own Transactions Summary rather than assuming the trade is misfiled.

## Adjust Position — NRA withholding corrections

Schwab periodically reprocesses non-resident withholding. These appear as `Other Activity` /
`Adjust Position` rows and are genuinely confusing:

- They can carry **transaction dates spanning several prior months** while all being *processed* on a
  single date (e.g. 13 corrections dated Jan–May, all processed 05/31).
- They may be **refunds** (positive) as well as additional withholding (negative).
- On sales in the same period, the sale `Amount` may be shown **net of tax withheld**, with the tax
  refunded separately — so the trade amount and the cash entry can look inconsistent until the
  adjustments are included.

Record each as a `tax` cash_transaction with its stated transaction date, preserve the sign as shown,
and note the processed-date grouping in `_note`.

## Edge Cases

1. **Stock splits**: Category "Other Activity", Action "Stock Split" — no cash impact, no amount. Extract as a `lot_action` with type `split`. Parse ticker from Symbol/CUSIP and description from the Description column. **Always capture the Quantity column into `lot_actions.quantity` (signed)** — never leave the share count only in the description.

   Single-row splits print the **additional** shares — a 5-for-1 on a 100-share holding prints `+400`,
   taking it to 500. Verified across TSLA, TTD, NVDA, ANET, AMZN, SHOP, GOOGL, BYDDY and NFLX splits,
   at holding sizes from single shares to five figures.

   **"Forward Split" is often booked as TWO rows** — a removal of the pre-split lot (Symbol/CUSIP
   column **blank**, quantity in parentheses) and an addition of the post-split lot. Example, a
   4-for-1: `(50.0000)` then `200.0000`, netting `+150`. Emit **both** rows as `split` with their
   signed quantities. The blank-symbol negative row is **not** a transfer out — infer its ticker from
   the paired addition.

2. **Cash-in-lieu rows have a BLANK Quantity column** — the dollar figure sits in Total Amount. Check
   the Realized Gain/(Loss) section to see whether the fraction was already netted out of the holding.
   Observed on a cash-and-stock merger: the acquirer's shares were delivered with a fractional
   remainder cashed in lieu, and the merger row printed the **net** whole-share figure while the
   fractional disposal was booked separately. Whichever convention the statement uses, the lot_action
   and the trades together must net to the holding — record the gross when the fraction is separately
   disposed of.

3. **Ticker changes** (e.g. SQ → XYZ for Block Inc) move no shares but split one position across two
   symbols in the trade history. Emit a `reorg` lot_action with `quantity: 0` naming both symbols, or
   positions will break on both.
2. **Industry Fee on sales**: The Charges/Interest column may show a small fee (e.g., $0.01). This is already deducted from the Amount column. Do NOT record as a separate fee.
3. **Wire fee waivers**: Always paired with the fee, netting to zero. Record both individually.
4. **Multiple accounts**: Schwab statements are per-account. Different accounts (individual vs joint) may have different features (e.g., DRIP enabled only on the joint account).
5. **Dates spanning months**: Interest descriptions may show date ranges crossing month boundaries (e.g., "08/28-09/28" in a September statement).
6. **T-bill CUSIP as ticker**: T-bills use CUSIP (e.g., `912797SC2`) rather than a human-readable ticker.
7. **T-bill maturity proceeds include interest**: a bill redeeming at par pays out par + accrued interest in a **single** line, so the redemption amount exceeds the face value. Record the full cash amount and identify the interest component in the description so income can be separated later.
8. **Quarterly and multi-month statements**: some periods are issued quarterly (Apr–Jun in a file dated 06-30) or spanning two months (Jan–Feb). Never assume one statement equals one month.
9. **Account wind-down**: when an account is closed out, the full portfolio is journaled out as `lot_action` transfers (no cash impact) and the cash swept out as a `withdrawal` — e.g. 8 securities journaled to a successor account plus a single cash journal, leaving a sub-dollar residual. The securities' market value can be orders of magnitude larger than the cash journal; it is **not** cash and must not enter the ledger.
10. **Dormant statements**: after a wind-down the account still issues monthly statements with a static residual, no positions, and **no Transaction Details section at all**. These are valid empty parses — beginning cash equals ending cash.
11. **DRIP triples net to zero**: dividend (+), NRA tax (−), reinvestment buy (−) on the same date leave cash unchanged. Record all three; do not collapse them.
12. **Same statement can appear under two fiscal years**: a Jan–Mar statement may be filed in both the ending and beginning FY folders. Identical parses in two places are expected, not duplicates to dedupe blindly — key on account + period.
