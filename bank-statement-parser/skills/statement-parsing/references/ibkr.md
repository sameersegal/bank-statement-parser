# Interactive Brokers (IBKR) Statement Reference

> Last updated: Epoch 2 (2026-07-20) — Epoch 1 based on 4 training + 2 test PDFs (Sep 2025–Feb 2026);
> Epoch 2 adds learnings from a multi-year production run (FY2020-21 → FY2025-26) covering
> annual statements, trade confirmation reports, stock trades, splits, and an account consolidation.

## ⚠️ Document Type — check this before parsing

Not every IBKR PDF is an Activity Statement, and **the filename does not tell you which is which.**

| Document | How to identify | Cash reconciliation |
|---|---|---|
| **Activity Statement** | Has a **Cash Report** section with Starting/Ending Cash | Yes — full ledger |
| **Trade Confirmation Report** | Trades only; **no Cash Report** | **Not possible** — set `starting_cash`/`ending_cash` to `null` and explain in `_note` |
| Supplementary reports (MTM Summary, Realized Summary, Dividend Report) | Single-topic report | Duplicate data — normally skip |

**The filename lies — read the Statement Period from inside the document.** A real case:
`UXXXXXXX_20220331_20220331.pdf` looks like a single-day snapshot but is the **full-year** Activity
Statement for 2021-04-01 → 2022-03-31, Cash Report included. Meanwhile
`UXXXXXXX_20210401_20220331.pdf` — whose name spans the whole year — is only a Trade Confirmation
Report with no cash ledger. The naming is effectively **inverted** relative to content.

Practical consequence: when a folder holds both, the `*_YYYYMMDD_YYYYMMDD.pdf` file with *identical*
start and end dates is often the authoritative full-period statement. Parse both, take the cash ledger
from whichever actually has the Cash Report, and note in `_note` that the other is a redundant subset.

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

- **Thousands separator**: comma (e.g., `123,456.78`)
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
| Trades (Purchase) | Sum of `buy` cash_transactions |
| Trades (Sales) | Sum of `sell` cash_transactions |
| Ending Cash | Should equal Starting + all above |

**Always verify**: `Starting Cash + sum(cash_transactions[].amount) = Ending Cash` — cash_transactions alone must balance.

### Penny-rounding tolerance

IBKR rounds each displayed line independently of its section totals, so a **±0.01 residual can be the
statement's own arithmetic, not a missed entry.** Observed on a full-year statement: the parsed ledger
summed to one cent *below* the stated Ending Cash, while the Cash Report's own printed lines summed to
one cent *above* it — because the displayed per-trade proceeds and commissions were each rounded a
cent away from the section totals they roll up into. Three mutually inconsistent totals, all printed
by the broker.

Before accepting a residual, verify **every** Cash Report section total (Dividends, Interest,
Withholding Tax, Commissions, Trades Purchase/Sales, Other Fees) ties to your parsed sums. If each
section ties and only the grand total is off by a cent, keep the statement's stated Ending Cash and
record the residual and its cause in `_note`. Never fabricate a balancing entry.

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
Same tabular format as Bonds (confirmed in Epoch 2 — a single fiscal year carried 37 stock trades
alongside Treasury purchases):

```
Symbol | Date/Time | Quantity | T. Price | C. Price | Proceeds | Comm/Fee | Basis | Realized P/L | MTM P/L | Code
```

- **quantity**: share count — **negative for sells**, positive for buys. Use the absolute value in
  `position_transactions.quantity` and let `type` carry the direction.
- **amount**: the Proceeds column (already signed: negative = purchase, positive = sale)
- **Commission**: the Comm/Fee column → a separate negative `fee` cash_transaction on the same date

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
7. **Stock splits / corporate actions**: stock splits, mergers, and other corporate actions that change position quantity without a trade are extracted as `lot_actions` (not position_transactions). Bond maturities/redemptions remain as `sell` trades. Confirmed in Epoch 2: NVDA 10-for-1 (2024-06-10) and ANET 4-for-1 (2024-12-04), both appearing in the **Corporate Actions** section with no cash impact.

   **Always capture the Quantity column into `lot_actions.quantity` (signed).** IBKR prints it on every
   Corporate Actions and Transfers row; leaving it in the description makes positions unreconcilable.

   **Split quantity convention is NOT uniform — read each row literally:**
   - Most splits print the **additional** shares only. A 4-for-1 on a 100-share holding prints `+300`,
     taking the position to 400. Verified across AAPL, GOOGL, TSLA, SHOP, NVDA and BYDDY splits.
   - **The ANET 4-for-1 (2024-12-04) is booked as an ISIN exchange**, i.e. two rows: `ANET.OLD -100`
     (old ISIN `US0404131064`) and `ANET +400` (new ISIN `US0404132054`). Here the positive leg is the
     **full post-split position**, not the increment; the pair nets `+300`. Emit both rows with their
     printed signed quantities. Treating the positive leg as "additional shares" inflates the position ~4x.

8. **Never emit a lot_action for a bond redemption or a merger surrender that is already a trade.**
   IBKR books "Full Call / Early Redemption" and Treasury bill maturities in the Corporate Actions
   section *and* as `sell` trades at par. Likewise the ZNGA cash-and-stock merger surrender appears as
   a sale for the cash consideration. Record these as trades only — emitting the corporate action too
   removes the position twice. Observed duplicates: 3 Treasury full-calls, 1 T-bill maturity, and one
   cash-and-stock merger surrender. The **received** leg of a merger (the acquirer's shares credited,
   often a fractional quantity) is a genuine lot_action and must still be emitted.
8. **Account transfers / consolidation**: positions moved to another IBKR account appear as internal transfers with a market value but **no cash proceeds** — record them as `transfer` lot_actions, never as sells. Any accompanying cash movement is a separate `withdrawal`/`deposit` in the ledger. Observed: 17 positions (16 stocks + 1 bond) transferred out to a successor account over two dates in the same month, alongside two separate cash transfers out. The transferred market value must never enter the cash ledger.
9. **One person, several accounts**: an owner may hold multiple IBKR accounts (e.g. an individual `UXXXXXXX` and a joint `UYYYYYYY`) whose statements sit side by side in the same folder. Always take `account_id` from the Account Information table, never from the folder or filename, and never merge two accounts into one record.
10. **Statement period may end mid-month**: a closing/transitional statement can end on an arbitrary date (e.g. 2021-03-03 rather than 2021-03-31). Use the printed period verbatim rather than normalising to month boundaries.
11. **Dormant post-consolidation shells**: after positions are transferred out, an account can persist for years with a small residual balance, no trades, and no cash flows. Parse these as valid empty statements carrying the residual forward.
12. **Bond full call / early redemption**: recorded as a `sell` at par (price 100.0); the final coupon and the redemption proceeds land in the same statement. Purchase accrued interest is a negative `interest` entry at buy time.
