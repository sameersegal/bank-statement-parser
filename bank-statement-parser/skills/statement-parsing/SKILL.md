---
name: statement-parsing
description: Parse brokerage PDF statements into structured JSON
user-invocable: true
invocation: /statement-parsing <path-or-broker>
argument-description: A PDF file path, a directory of PDFs, or a broker name (ibkr/schwab/fidelity/dbs-sg) to parse all train+test files for that broker
---

# Statement Parsing Skill

You parse brokerage PDF statements and extract structured transaction data into JSON.

## Core Rules

These hold for every broker and override any convenient-looking shortcut:

1. **The document is the source of truth — not the filename.** Never infer the period, account, or
   content from a filename. Real failures seen in production: two statements whose contents were
   swapped relative to their month names; a file named for a single day (`..._20220331_20220331.pdf`)
   that was actually a full-year statement; a folder whose filenames were all offset by one month; and
   byte-identical duplicate PDFs. Parse the body, then record any filename/content mismatch in `_note`.
2. **Never infer an amount's sign from its transaction type.** Read it from the statement's own
   signal: the Debit/Credit column, parentheses, an explicit minus, or the running-balance delta.
   Interest gets *charged* on overdrawn balances, fees get reversed, transfers go both ways.
3. **Exclude pending/unsettled activity.** Anything outside the settled closing balance has not
   happened yet. It settles next period (recorded there) or never posts at all.
4. **An empty statement is a valid parse.** Dormant and quiet periods often have no transaction
   section whatsoever. Emit empty arrays with correct metadata — do not skip the file or call it an error.
5. **Reconcile per currency**, never in aggregate, on multi-currency accounts.
6. **Surface anomalies in `_note`; never silently absorb them.** A residual, an excluded row, a
   duplicate, a document that isn't the type it claims — all of it goes in `_note`.
7. **Keep deposit accounts separate from brokerage accounts.** If one PDF covers both (see DBS),
   emit two records with distinct `account_type`s rather than merging them.

## Input Resolution

The user provides `<path-or-broker>` which can be:

1. **A single PDF file path** — parse that one file
2. **A directory path** — parse all `.pdf`/`.PDF` files in that directory (non-recursive)
3. **A broker name** (`ibkr`, `schwab`, `fidelity`, or `dbs-sg`) — parse all PDFs under `data/<broker>/` (recursive, both train and test)

## Workflow

### Step 1: Identify Target Files

Resolve the input to a list of PDF file paths. If a broker name is given, glob for `data/<broker>/**/*.pdf` and `data/<broker>/**/*.PDF`.

### Step 2: Identify Broker

For each PDF, determine the broker:
- If the path contains `ibkr` → Interactive Brokers
- If the path contains `schwab` → Charles Schwab
- If the path contains `fidelity` → Fidelity Stock Plan Services
- If the path contains `dbs-sg` → DBS Singapore (Treasures)
- Otherwise, read the first page of the PDF and look for broker-identifying text ("Interactive Brokers", "Charles Schwab", "DBS Treasures", etc.)

### Step 3: Load Broker Reference

Read the broker-specific reference file:
- IBKR: `@references/ibkr.md`
- Schwab: `@references/schwab.md`
- Fidelity: `@references/fidelity.md`
- DBS-SG: `@references/dbs-sg.md`

Also load the output schema: `@references/output-schema.md`

If the broker reference has been populated by the learning skill, use its documented section headers, patterns, and parsing rules. If the reference is still empty/skeleton, do your best to parse from first principles.

### Step 4: Read and Parse the PDF

Use the Read tool to read the PDF file. Then extract:

1. **Account metadata**: broker name, account ID, statement period (from/to dates), starting/ending cash
2. **Position transactions**: all buy/sell trades — capture date, type (buy/sell), ticker, quantity, price, amount, currency
3. **Cash transactions**: a complete cash ledger — includes dividends, interest, fees, deposits, withdrawals, forex, taxes, AND the cash impact of every trade (buy/sell). Each position transaction must have a corresponding cash transaction entry with the same amount (dates may differ by a settlement lag).
4. **Lot actions**: corporate actions that change positions without a trade — stock splits, bonus issues, mergers, class reorganizations. No cash impact.

Follow the broker reference guidance for locating sections and interpreting formats.

**First, confirm the document is what it claims to be.** Some files are not full statements — e.g. an
IBKR *Trade Confirmation Report* has trades but no Cash Report, so cash cannot reconcile. Identify the
document type from its own headings before parsing, set unavailable fields to `null`, and say so in
`_note`. If a sibling file in the same folder *is* the full statement, note that too.

**When the text layer scrambles columns**, fall back to coordinate-based extraction (e.g. `pdfplumber`
word objects with x-positions), bucketing numbers by their right-edge x-coordinate. Flat text
extraction routinely merges Debit/Credit/Balance into an ambiguous run — which silently inverts signs.
Locate the header row and reuse its x-positions for the rows beneath rather than hardcoding offsets.

### Step 5: Write JSON Output

Write the JSON output file alongside the PDF with the same base name and `.json` extension.
For example: `data/ibkr/train/statement_202509.pdf` → `data/ibkr/train/statement_202509.json`

The JSON must conform to `@references/output-schema.md`.

**If a JSON sibling already exists, verify before overwriting.** Parse the PDF independently, compare
against the existing file, and leave it untouched if it matches. Only rewrite when you find a genuine
discrepancy — and say what changed. Blindly rewriting churns files and can silently discard correct
prior work (including hand-made corrections).

**Write working files to a scratch directory, never into the statement folders.** Intermediate text
dumps, diagnostics, and scratch scripts must not land next to the source data — they pollute the
dataset and get picked up by inventory scans. Only the `.json` outputs belong beside the PDFs.

### Step 6: Validate and Report

After writing, perform basic validation:
- All required fields present
- Dates in correct format
- Amounts are numeric
- No duplicate transactions
- **Cash reconciliation**: `starting_cash + sum(cash_transactions[].amount) = ending_cash`
  — **per currency** on multi-currency accounts; each currency must balance independently
- Every position_transaction has a matching cash_transaction (same amount and type; dates may differ
  by a settlement lag, and on split brokerage/deposit accounts the match is in the sibling record)

**On a reconciliation failure**, re-read the statement for missed entries first — most failures are a
sign error (see Core Rule 2) or a pending row wrongly included (Core Rule 3). If a residual of ±0.01
survives, it is usually the broker's own display rounding; keep the statement's stated ending balance
and record the residual and its cause in `_note`. Never invent a balancing transaction, and never
adjust a real amount to force a reconciliation.

Report a summary to the user:
```
Parsed: <filename>
  Account: <account_id>
  Period: <from> to <to>
  Position transactions: <count>
  Cash transactions: <count>
  Lot actions: <count>
  Output: <json_path>
```

## Batch Processing

When processing multiple files, parse them sequentially. After all files are done, print a summary table:

```
| File | Account | Period | Positions | Cash | Lot Actions | Recon | Status |
|------|---------|--------|-----------|------|-------------|-------|--------|
| ...  | ...     | ...    | ...       | ...  | ...         | OK/FAIL (per ccy) | OK/ERR |
```

### Sizing a batch

Reading PDFs consumes context quickly, and it scales with **page count**, not file size — a 12-page
DBS statement costs several times a 4-page Schwab one even though most DBS pages are boilerplate.

- Keep a batch to roughly **6–12 statements** for page-heavy brokers (DBS), or **up to ~15** for short,
  quiet statements. Overlong batches run out of context mid-run and lose completed work.
- Split large jobs along natural boundaries (account, financial year, sub-account) so a failed batch
  is cheap to retry and easy to identify.
- **Write each JSON as you finish that statement**, not in one pass at the end. If the run is
  interrupted, everything already parsed survives.
- If a run dies partway, **check the filesystem before re-running** — completed outputs are often
  already on disk, and re-parsing them wastes budget and risks overwriting good work.

### Cross-file checks worth doing

- **Balance continuity**: each statement's ending cash should equal the next statement's opening cash.
  A break points to a missing statement or a misparse — both worth flagging.
- **Duplicate detection**: hash the PDFs. Byte-identical duplicates of the same period do occur; flag
  them rather than treating them as two periods.
- **Coverage gaps**: a "missing" month is often a quarterly statement covering three months, or a
  filename offset — verify against the parsed periods before reporting a gap.

## Error Handling

- If a PDF cannot be read, log the error and continue to the next file.
- If broker cannot be identified, ask the user.
- If a field cannot be extracted, set it to `null` in the JSON and note it in the summary.
