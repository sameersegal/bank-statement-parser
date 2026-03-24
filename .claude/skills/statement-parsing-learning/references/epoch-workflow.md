# Epoch Workflow

Each learning epoch consists of 7 phases. All results are written to `.claude/results/<broker>/epoch-<N>/`.

## Phase 1: Parse (Train Set)

Run `/statement-parsing` on every PDF in `data/<broker>/train/`.

- This generates JSON files alongside each PDF.
- Record the list of files parsed and any immediate errors.

## Phase 2: Self-Review

For each train PDF, perform a careful self-review:

1. Re-read the PDF using the Read tool.
2. Read the generated JSON file.
3. Walk through the PDF content section by section, comparing against the JSON.
4. Use the checklist from `@references/self-review-checklist.md`.
5. Record every discrepancy as a structured error:
   ```json
   {
     "file": "filename.pdf",
     "field": "position_transactions[2].amount",
     "error_type": "wrong_amount",
     "expected": -1500.00,
     "actual": -150.00,
     "context": "The PDF shows 'Total: -1,500.00' in the trades section"
   }
   ```
6. Write all errors to `epoch-<N>/self-review-errors.json`.

## Phase 3: Pattern Analysis

Analyze errors across ALL train files to find systematic patterns:

- Group errors by `error_type`.
- Look for broker-specific formatting that caused repeated mistakes.
- Identify section headers, delimiters, or layouts that were misinterpreted.
- Identify any categories of transactions that were consistently missed.
- Write findings to `epoch-<N>/pattern-analysis.md`.

## Phase 4: Reference Update

Update the broker reference file (`.claude/skills/statement-parsing/references/<broker>.md`):

- Add discovered section headers and their exact text.
- Document number formats (e.g., negative amounts shown in parentheses).
- Document date formats used in different sections.
- Add parsing rules for tricky cases.
- Add examples of correctly parsed entries.
- Remove or correct any previous guidance that led to errors.

**Important**: Only add patterns confirmed across multiple files. Flag single-file observations as tentative.

## Phase 5: Validation (Re-parse Train Set)

Re-run `/statement-parsing` on the train set using the updated reference.

- Compare new JSON against previous self-review to check if identified errors are fixed.
- Record which errors are resolved and which persist.
- Write results to `epoch-<N>/validation-results.md`.

## Phase 6: Test Evaluation

Run `/statement-parsing` on `data/<broker>/test/` PDFs.

- Perform the same self-review process as Phase 2.
- Record errors to `epoch-<N>/test-errors.json`.
- This measures generalization — whether learned patterns apply to unseen statements.

## Phase 7: Reporting

Write `epoch-<N>/summary.md` containing:

1. **Epoch number and broker**
2. **Train metrics**: error counts by type, accuracy rate (see `@references/accuracy-metrics.md`)
3. **Test metrics**: error counts by type, accuracy rate
4. **Improvements from previous epoch** (if not epoch 1)
5. **Remaining issues**: errors that persist and hypotheses for fixing them
6. **Next steps**: what the next epoch should focus on

Update `.claude/results/<broker>/learning-log.md` with a one-line summary of this epoch.
