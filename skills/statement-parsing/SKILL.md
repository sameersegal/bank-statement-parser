---
name: statement-parsing
description: Parse brokerage PDF statements into structured JSON
user-invocable: true
invocation: /statement-parsing <path-or-broker>
argument-description: A PDF file path, a directory of PDFs, or a broker name (ibkr/schwab) to parse all train+test files for that broker
---

# Statement Parsing Skill

You parse brokerage PDF statements and extract structured transaction data into JSON.

## Input Resolution

The user provides `<path-or-broker>` which can be:

1. **A single PDF file path** — parse that one file
2. **A directory path** — parse all `.pdf`/`.PDF` files in that directory (non-recursive)
3. **A broker name** (`ibkr` or `schwab`) — parse all PDFs under `data/<broker>/` (recursive, both train and test)

## Workflow

### Step 1: Identify Target Files

Resolve the input to a list of PDF file paths. If a broker name is given, glob for `data/<broker>/**/*.pdf` and `data/<broker>/**/*.PDF`.

### Step 2: Identify Broker

For each PDF, determine the broker:
- If the path contains `ibkr` → Interactive Brokers
- If the path contains `schwab` → Charles Schwab
- Otherwise, read the first page of the PDF and look for broker-identifying text ("Interactive Brokers", "Charles Schwab", etc.)

### Step 3: Load Broker Reference

Read the broker-specific reference file:
- IBKR: `@references/ibkr.md`
- Schwab: `@references/schwab.md`

Also load the output schema: `@references/output-schema.md`

If the broker reference has been populated by the learning skill, use its documented section headers, patterns, and parsing rules. If the reference is still empty/skeleton, do your best to parse from first principles.

### Step 4: Read and Parse the PDF

Use the Read tool to read the PDF file. Then extract:

1. **Account metadata**: broker name, account ID, statement period (from/to dates)
2. **Position transactions**: all buy/sell trades — capture date, type (buy/sell), ticker, quantity, price, amount, currency
3. **Cash transactions**: all non-trade cash movements — dividends, interest, fees, deposits, withdrawals, forex, taxes

Follow the broker reference guidance for locating sections and interpreting formats.

### Step 5: Write JSON Output

Write the JSON output file alongside the PDF with the same base name and `.json` extension.
For example: `data/ibkr/train/statement_202509.pdf` → `data/ibkr/train/statement_202509.json`

The JSON must conform to `@references/output-schema.md`.

### Step 6: Validate and Report

After writing, perform basic validation:
- All required fields present
- Dates in correct format
- Amounts are numeric
- No duplicate transactions

Report a summary to the user:
```
Parsed: <filename>
  Account: <account_id>
  Period: <from> to <to>
  Position transactions: <count>
  Cash transactions: <count>
  Output: <json_path>
```

## Batch Processing

When processing multiple files, parse them sequentially. After all files are done, print a summary table:

```
| File | Account | Period | Positions | Cash | Status |
|------|---------|--------|-----------|------|--------|
| ...  | ...     | ...    | ...       | ...  | OK/ERR |
```

## Error Handling

- If a PDF cannot be read, log the error and continue to the next file.
- If broker cannot be identified, ask the user.
- If a field cannot be extracted, set it to `null` in the JSON and note it in the summary.
