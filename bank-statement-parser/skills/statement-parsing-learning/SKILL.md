---
name: statement-parsing-learning
description: Iterative learning loop that improves broker-specific parsing by training on PDFs and refining reference guides
user-invocable: true
invocation: /statement-parsing-learning <broker>
argument-description: Broker name (ibkr or schwab) to run a learning epoch for
disable-model-invocation: true
---

# Statement Parsing Learning Skill

You run iterative learning epochs that improve the parsing quality for a specific broker. Each epoch parses PDFs, self-reviews the results, identifies error patterns, and updates the broker reference guide.

## Prerequisites

- Training PDFs exist in `data/<broker>/train/`
- Test PDFs exist in `data/<broker>/test/`
- The `/statement-parsing` skill is available

## Epoch Execution

### Step 1: Determine Epoch Number

Check `data/<broker>/results/` for existing epoch directories. The new epoch is `epoch-<N>` where N is the next integer (starting from 1).

Create the directory: `data/<broker>/results/epoch-<N>/`

### Step 2: Parse Train Set

Invoke the `/statement-parsing` skill on all training PDFs:
- Run `/statement-parsing data/<broker>/train/` to parse every PDF in the train directory.
- Record which files were parsed and any immediate failures.

### Step 3: Self-Review

For EACH training PDF, perform a detailed self-review:

1. Re-read the PDF using the Read tool.
2. Read the generated JSON file.
3. Compare them section by section using `@references/self-review-checklist.md`. Verify that any corporate actions (stock splits, mergers, bonus issues) in the PDF are captured as `lot_actions`.
4. Record every discrepancy as a structured error entry.
5. Write all errors to `data/<broker>/results/epoch-<N>/self-review-errors.json`.

Be meticulous. Read every number, every date, every ticker. The goal is to catch ALL errors, not just obvious ones.

### Step 4: Identify Systematic Error Patterns

Analyze errors across all train files:

- Group by error type (see `@references/accuracy-metrics.md` for categories).
- Look for patterns: Are the same sections consistently misread? Are certain number formats always wrong?
- Identify root causes: Is it a section header issue? A number format issue? A date format issue?
- Write analysis to `data/<broker>/results/epoch-<N>/pattern-analysis.md`.

### Step 5: Update Broker Reference

Edit the broker reference file at `skills/statement-parsing/references/<broker>.md`:

- Document discovered section headers with their exact text from the PDFs.
- Add number format rules (e.g., negatives in parentheses, comma separators).
- Add date format rules for each section.
- Add specific parsing instructions for tricky sections.
- Include examples of correctly parsed entries.
- Remove or correct guidance that caused errors.

**Only add patterns confirmed across multiple files.** Mark single-file observations as tentative.

### Step 6: Present Findings to User

Pause and present a summary to the user:

- Number of errors found
- Top error patterns
- Proposed reference changes
- Ask the user to spot-check 1-2 specific comparisons if needed

Wait for user acknowledgment before continuing.

### Step 7: Re-parse Train Set (Validation)

Run `/statement-parsing` again on the train set to validate improvements:

- Compare new results against previously identified errors.
- Check which errors are now fixed.
- Check for any new regressions.
- Write results to `data/<broker>/results/epoch-<N>/validation-results.md`.

### Step 8: Parse and Review Test Set

Run `/statement-parsing` on `data/<broker>/test/`:

- Parse all test PDFs.
- Perform the same self-review process as Step 3.
- Write errors to `data/<broker>/results/epoch-<N>/test-errors.json`.
- This measures whether learned patterns generalize to unseen statements.

### Step 9: Write Epoch Summary

Create `data/<broker>/results/epoch-<N>/summary.md` with:

1. Epoch number, broker, and date
2. Train set metrics (per `@references/accuracy-metrics.md`)
3. Test set metrics
4. Comparison to previous epoch (if applicable)
5. Errors fixed and errors remaining
6. Hypotheses for remaining errors
7. Recommended focus areas for next epoch

### Step 10: Update Learning Log

Append a one-line entry to `data/<broker>/results/learning-log.md`:

```
| Epoch | Date | Train F1 | Test F1 | Errors Fixed | Notes |
```

See `@references/epoch-workflow.md` for detailed phase descriptions.

## Important Guidelines

- **Be thorough in self-review.** The quality of learning depends on catching all errors.
- **Read actual PDFs.** Never assume what a PDF contains — always re-read it during self-review.
- **Confirm patterns across files.** Don't update the reference based on a single observation.
- **Track everything.** Every error, every change, every metric — write it down in the epoch directory.
- **Present findings clearly.** The user should understand what was learned and what changed.
