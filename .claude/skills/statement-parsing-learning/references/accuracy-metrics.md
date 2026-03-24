# Accuracy Metrics

## Error Categories

Each discrepancy found during self-review is classified into one of these categories:

| Error Type             | Description                                              |
|------------------------|----------------------------------------------------------|
| `missing_transaction`  | A transaction in the PDF is absent from the JSON         |
| `extra_transaction`    | A JSON entry has no corresponding PDF transaction        |
| `wrong_amount`         | Amount value is incorrect                                |
| `wrong_date`           | Date is incorrect                                        |
| `wrong_type`           | Transaction type is wrong (e.g., buy vs sell, dividend vs interest) |
| `wrong_ticker`         | Ticker symbol is incorrect                               |
| `wrong_quantity`       | Share quantity is incorrect                               |
| `wrong_price`          | Price per share is incorrect                             |
| `wrong_currency`       | Currency code is incorrect                               |
| `wrong_sign`           | Amount has the wrong sign (positive vs negative)         |
| `wrong_description`    | Cash transaction description is significantly wrong      |
| `wrong_metadata`       | Account ID or period dates are incorrect                 |
| `duplicate`            | Same transaction appears more than once                  |

## Computed Metrics

### Per-File Metrics

For each PDF file:

- **Total expected transactions**: Count of all transactions visible in the PDF (position + cash)
- **Total extracted transactions**: Count of all transactions in the JSON
- **Correct transactions**: Extracted transactions with no errors
- **Error count**: Number of individual field-level errors
- **File accuracy**: `correct_transactions / total_expected_transactions`

### Per-Epoch Aggregate Metrics

Across all files in a set (train or test):

- **Transaction precision**: `correct_transactions / total_extracted_transactions` — measures how many extracted transactions are correct (penalizes extras and field errors)
- **Transaction recall**: `correct_transactions / total_expected_transactions` — measures how many real transactions were correctly captured (penalizes misses)
- **F1 score**: `2 * (precision * recall) / (precision + recall)` — harmonic mean of precision and recall
- **Error rate by type**: Count of each error category across all files
- **Overall accuracy**: `total_correct / total_expected` across all files

## Reporting Format

Each epoch summary should include a metrics table:

```
### Train Set Metrics

| Metric                | Value  |
|-----------------------|--------|
| Files parsed          | N      |
| Total expected txns   | N      |
| Total extracted txns  | N      |
| Correct txns          | N      |
| Precision             | X.XX%  |
| Recall                | X.XX%  |
| F1 Score              | X.XX%  |

### Error Breakdown

| Error Type            | Count  |
|-----------------------|--------|
| missing_transaction   | N      |
| wrong_amount          | N      |
| ...                   | ...    |
```

## Improvement Tracking

When comparing across epochs, report:

- **Errors fixed**: Errors from previous epoch that no longer appear
- **Errors introduced**: New errors not present in previous epoch
- **Net improvement**: `errors_fixed - errors_introduced`
- **Accuracy delta**: Change in F1 score from previous epoch
