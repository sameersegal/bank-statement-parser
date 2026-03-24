# Statement Parser

AI-powered brokerage statement parser built as [Claude Code](https://claude.ai/claude-code) skills. Reads PDF statements from multiple brokers and extracts structured JSON with account metadata, position transactions (buys/sells), and cash transactions (dividends, interest, taxes, fees, withdrawals).

## Supported Brokers

| Broker | Status | Formats Tested |
|--------|--------|----------------|
| Interactive Brokers (IBKR) | Supported | Activity Statements (monthly) |
| Charles Schwab | Supported | Schwab One International Brokerage Statements |

## Quick Start

### Prerequisites

- [Claude Code CLI](https://claude.ai/claude-code) installed
- PDF statements placed in the `data/` directory (see structure below)

### Parse a Single Statement

```
/statement-parsing data/ibkr/train/your_statement.pdf
```

### Parse All Statements for a Broker

```
/statement-parsing ibkr
/statement-parsing schwab
```

### Output

Each PDF produces a `.json` file alongside it with the same base name:

```json
{
  "broker": "ibkr",
  "account_id": "UXXXXXXXX",
  "period": { "from": "2025-09-01", "to": "2025-09-30" },
  "position_transactions": [
    { "date": "2025-12-26", "type": "buy", "ticker": "912797TB3", "quantity": 160000, "price": 99.0961, "amount": -158553.76, "currency": "USD" }
  ],
  "cash_transactions": [
    { "date": "2025-09-04", "type": "interest", "description": "USD Credit Interest for Aug-2025", "amount": 1781.70, "currency": "USD" },
    { "date": "2025-09-15", "type": "dividend", "description": "GOOG Cash Dividend USD 0.21 per Share", "amount": 412.23, "currency": "USD" },
    { "date": "2025-09-15", "type": "tax", "description": "GOOG Withholding Tax", "amount": -103.06, "currency": "USD" }
  ]
}
```

## Project Structure

```
bank-statement-parser/
├── data/                          # Statement PDFs, parsed JSON, and results (gitignored)
│   ├── <broker>/
│   │   ├── train/                 # Training statements (used for learning)
│   │   ├── test/                  # Test statements (used for validation)
│   │   └── results/               # Learning epoch results
│   │       ├── epoch-1/
│   │       └── learning-log.md
│
├── .claude-plugin/                # Plugin manifest (cross-tool compatibility)
│   ├── plugin.json
│   └── marketplace.json
│
├── skills/
│   ├── statement-parsing/         # Main parsing skill
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── output-schema.md        # JSON output contract
│   │       ├── ibkr.md                # IBKR format reference (learned)
│   │       └── schwab.md              # Schwab format reference (learned)
│   │
│   └── statement-parsing-learning/    # Meta-learning skill
│       ├── SKILL.md
│       └── references/
│           ├── epoch-workflow.md
│           ├── self-review-checklist.md
│           └── accuracy-metrics.md
│
├── .gitignore
└── README.md
```

## How It Works

The system uses two Claude Code skills:

1. **`/statement-parsing`** — Reads a PDF, identifies the broker, loads the broker-specific reference guide, extracts transactions, and writes JSON. Cross-checks against the statement's cash summary to verify completeness.

2. **`/statement-parsing-learning`** — An iterative learning loop that improves parsing quality. Each epoch: parses training PDFs, self-reviews by re-reading each PDF and comparing against the JSON, identifies systematic error patterns, updates the broker reference guide, then validates on the test set.

The broker reference files (e.g., `ibkr.md`, `schwab.md`) start empty and are populated by the learning skill as it discovers section headers, number formats, date conventions, and transaction type mappings from actual PDFs.

## Adding a New Broker

To teach the parser a new brokerage format:

### 1. Create the Data Directory

```
mkdir -p data/<broker>/train
mkdir -p data/<broker>/test
```

Place 3-6 monthly statements in `train/` and 1-2 in `test/`. Choose statements with variety — months that include dividends, interest, trades, fees, and other transaction types will produce a better reference.

### 2. Create the Broker Reference Skeleton

Create `skills/statement-parsing/references/<broker>.md`:

```markdown
# <Broker Name> Statement Reference

> This file is populated by the `/statement-parsing-learning` skill.
> It starts empty and is incrementally updated as the learning skill
> discovers formats, section headers, and parsing patterns from actual PDFs.

## Status: Not yet populated

Run `/statement-parsing-learning <broker>` to begin discovering and documenting the statement format.
```

### 3. Run the Learning Skill

```
/statement-parsing-learning <broker>
```

This will:
- Read each training PDF and extract transactions (best effort on first pass)
- Self-review: re-read each PDF and compare against the generated JSON
- Identify systematic error patterns across all files
- Update the broker reference with discovered formats and rules
- Present findings for your review
- Validate by re-parsing the training set
- Test on the held-out test set
- Write metrics and a summary

### 4. Iterate if Needed

If the first epoch has errors, run another epoch:

```
/statement-parsing-learning <broker>
```

Each epoch builds on the previous one — the reference gets more precise, and errors should decrease. The learning log tracks F1 score across epochs.

### Tips for Good Results

- **Diverse training data**: Include months with different transaction types (dividends, interest, trades, fees, deposits, withdrawals)
- **Multiple accounts**: If the broker supports different account types (individual, joint, margin), include examples of each
- **Edge cases**: Include statements with stock splits, bond maturities, corporate actions, or other unusual events
- **Minimal test set**: 1-2 statements in `test/` is enough — they should be from a different time period than training

## Transaction Types

### Position Transactions (Trades)

| Type | Description |
|------|-------------|
| `buy` | Stock purchase, bond purchase, T-bill purchase, DRIP reinvestment |
| `sell` | Stock sale, bond redemption/maturity |

### Cash Transactions

| Type | Description |
|------|-------------|
| `dividend` | Cash dividends (qualified, ordinary, reinvested) |
| `interest` | Broker interest, bond coupon payments, purchase accrued interest |
| `tax` | Withholding tax (NRA tax, US tax on dividends/interest) |
| `fee` | Commissions, wire fees, ADR fees, service charges |
| `deposit` | Cash deposits |
| `withdrawal` | Wire transfers out, disbursements |
| `forex` | Foreign exchange transactions |
| `other` | Fee waivers, adjustments, miscellaneous |

## Cross-Tool Compatibility

This repo follows the [`.claude-plugin` convention](https://github.com/cloudflare/skills) — skills live in `skills/` at the root with a `.claude-plugin/` manifest. This structure is discoverable by Claude Code, Codex, and other tools that support the Agent Skills open standard.

## Verification

Every parsed statement is cross-checked against the statement's own cash summary:

```
Starting Cash + Position Transactions + Cash Transactions = Ending Cash
```

This ensures no transactions are missed or double-counted.
