# Self-Review Checklist

Use this checklist when comparing a generated JSON file against its source PDF. Go through every section systematically.

## 1. Metadata Checks

- [ ] **Broker**: Correct broker identified?
- [ ] **Document type**: Is this actually a full statement? (A trade-confirmation or supplementary report has no cash ledger — cash fields should be `null` with a `_note`, not zeros.)
- [ ] **Account ID**: Matches the account number in the PDF header? (Never taken from the folder or filename.)
- [ ] **Account type**: If the broker splits brokerage vs deposit accounts, is `account_type` set and are the two emitted as separate records rather than merged?
- [ ] **Period from**: Matches the statement start date **as printed inside the document**?
- [ ] **Period to**: Matches the statement end date?
- [ ] **Filename vs content**: Does the filename agree with the parsed period? Swapped, offset, single-day-named-but-full-year, and duplicate filenames all occur — content wins, and the mismatch belongs in `_note`.
- [ ] **Starting/ending cash**: Both present (or explicitly `null` with a reason)?

## 2. Position Transaction Checks

- [ ] **Completeness**: Are ALL buy/sell transactions from the PDF present in the JSON?
- [ ] **No extras**: Are there any JSON entries that don't correspond to a PDF transaction?
- [ ] **Dates**: Every transaction date matches the PDF exactly?
- [ ] **Type**: Buy/sell correctly identified for each transaction?
- [ ] **Ticker**: Correct ticker symbol for each transaction? (Watch for ticker vs. full name confusion)
- [ ] **Quantity**: Share counts match exactly? (Watch for decimal shares)
- [ ] **Price**: Per-share price matches? (Watch for rounding)
- [ ] **Amount**: Total amount matches? (Watch for sign convention: negative for buys, positive for sells)
- [ ] **Currency**: Correct currency for each transaction?

## 3. Cash Transaction Checks

- [ ] **Completeness**: Are ALL cash transactions (dividends, interest, fees, deposits, withdrawals, forex, taxes) present?
- [ ] **No extras**: Any JSON entries without a corresponding PDF entry?
- [ ] **Type classification**: Each transaction correctly categorized (dividend vs interest vs fee, etc.)?
- [ ] **Dates**: All dates correct?
- [ ] **Amounts**: All amounts match with correct signs? (positive = inflow, negative = outflow)
- [ ] **Sign derivation**: Was each sign taken from the statement's own signal (Debit/Credit column, parentheses, minus, running-balance delta) rather than assumed from the transaction type? Spot-check any **interest that is negative** (charged on an overdrawn balance), **fees that are positive** (reversals/waivers), and **transfers** (which go both directions) — these are the classic sign failures.
- [ ] **Currency**: Correct currency for each entry?
- [ ] **Description**: Description captures the original text from the statement?
- [ ] **Pending excluded**: Are unsettled rows ("Pending / Open Activity", "Pending settlement transactions") left out? They must not appear until the period in which they actually settle — and some never settle at all.

## 4. Lot Action Checks

- [ ] **Completeness**: Every corporate action that changed quantity without a trade is present (splits, mergers, reorgs, in-kind transfers/journals)?
- [ ] **No cash impact**: No lot_action has a corresponding cash_transaction. In-kind transfers move securities, not money — a large market value must never enter the ledger.
- [ ] **Type enum**: Only `split`, `bonus`, `merger`, `reorg`, `transfer` used? (No invented variants like `transfer_in`; direction belongs in the description.)
- [ ] **Split ratios**: Ratio stated in the document, or `null` with the credited quantity in the description? (Not silently inferred.)
- [ ] **Split double-counting**: A split printed as two lines (old lot out, new lot in) recorded as **one** split?
- [ ] **Cash mergers**: Recorded as a `merger` lot_action **plus** a separate cash credit for the consideration?

## 5. Cross-Checks

- [ ] **No duplicates**: Same transaction not recorded twice?
- [ ] **Trade cash impact present**: Every `position_transaction` HAS a matching `cash_transaction` of the same amount and type. This duplication is **required** by the schema so that `cash_transactions` stands alone as a complete ledger — it is not an error. (Exceptions: Fidelity stock-plan externally-funded transactions, and split brokerage/deposit accounts where the match lives in the sibling record.)
- [ ] **Settlement lag**: Trade date on the position, settlement date on the cash entry — matched on amount, not date?
- [ ] **Multi-currency**: If the statement has multiple currencies, are transactions correctly attributed, and does **each currency reconcile independently**? (Never sum across currencies.)
- [ ] **Reconciliation**: `starting_cash + sum(cash) = ending_cash`? If off by ±0.01, was it verified as broker display rounding and recorded in `_note` — rather than patched with an invented entry or an adjusted amount?
- [ ] **Section boundaries**: Transactions are not mixed between sections (e.g., a dividend listed as a trade)
- [ ] **Empty is valid**: If the statement genuinely has no transaction section, are the arrays empty with correct metadata — rather than the file being skipped or flagged as an error?

## 6. Structural Observations

When reviewing, also note (for pattern analysis):

- Section headers and their exact text (for updating the broker reference)
- Number formatting conventions (commas, parentheses for negatives, decimal places)
- Date formatting conventions used in different sections
- Any unusual layouts or edge cases encountered
- Transactions that span multiple lines
- Summary/total rows that should NOT be parsed as transactions
- Closing-row label variants (e.g. "Balance" vs "Balance in your favour") that a parser must match
- Whether flat text extraction preserved column alignment, or coordinate-based extraction was needed
- Sections that are **absent** in quiet periods (their absence is itself a pattern worth documenting)
- Any second account bundled into the same PDF, and whether it is in scope

## 7. Anti-Patterns — flag these as errors

- **Balancing entries**: a transaction with no counterpart in the PDF, added to force reconciliation.
- **Adjusted amounts**: a real transaction's value altered to make totals tie.
- **Pre-booked pending items**: unsettled rows recorded as though they had settled.
- **Filename-derived metadata**: period or account taken from the filename instead of the document.
- **Type-assumed signs**: interest forced positive, fees forced negative, transfers forced outbound.
- **Merged accounts**: a deposit ledger folded into a brokerage record (or vice versa).
- **Silent discrepancies**: any residual, exclusion, or oddity resolved without a `_note`.
