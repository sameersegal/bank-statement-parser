# Self-Review Checklist

Use this checklist when comparing a generated JSON file against its source PDF. Go through every section systematically.

## 1. Metadata Checks

- [ ] **Broker**: Correct broker identified?
- [ ] **Account ID**: Matches the account number in the PDF header?
- [ ] **Period from**: Matches the statement start date?
- [ ] **Period to**: Matches the statement end date?

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
- [ ] **Currency**: Correct currency for each entry?
- [ ] **Description**: Description captures the original text from the statement?

## 4. Cross-Checks

- [ ] **No duplicates**: Same transaction not recorded twice?
- [ ] **Trade vs. cash separation**: Trade-related cash flows (settlement amounts) are NOT duplicated in cash_transactions
- [ ] **Multi-currency**: If the statement has multiple currencies, are transactions correctly attributed?
- [ ] **Section boundaries**: Transactions are not mixed between sections (e.g., a dividend listed as a trade)

## 5. Structural Observations

When reviewing, also note (for pattern analysis):

- Section headers and their exact text (for updating the broker reference)
- Number formatting conventions (commas, parentheses for negatives, decimal places)
- Date formatting conventions used in different sections
- Any unusual layouts or edge cases encountered
- Transactions that span multiple lines
- Summary/total rows that should NOT be parsed as transactions
