# CLAUDE.md - Instructions for Claude Code

This file provides context for Claude Code when working in this budget directory.

## Project Overview

This is a personal budget analysis directory using the `tally` CLI tool.
The tool categorizes bank/credit card transactions and generates spending reports.

## Key Commands

```bash
# Run analysis (uses ./config by default)
tally run

# Show summary only (good for checking Unknown transactions)
tally run --summary

# Run with specific config directory
tally run ./path/to/config

# Initialize a new budget directory
tally init

# Discover unknown merchants (KEY FOR CLASSIFICATION)
tally discover                # Human-readable
tally discover --format json  # For programmatic use
tally discover --format csv   # Ready to copy into rules

# Inspect a CSV to determine its format
tally inspect data/file.csv
```

## Directory Structure

```
.
├── config/
│   ├── settings.yaml           # Data sources and settings
│   └── merchant_categories.csv # Pattern matching rules (MAIN FILE TO EDIT)
├── data/                       # Statement files (DO NOT commit - contains PII)
└── output/                     # Generated reports
```

## Primary Task: Classifying Unknown Merchants

When asked to improve categorization:

1. Run `tally discover --format json` to find unknown merchants sorted by spend
2. For each unknown merchant:
   - Identify what the merchant is (restaurant, store, subscription, etc.)
   - Determine appropriate Category and Subcategory
   - Create a regex pattern that matches the transaction description
3. Add rules to `config/merchant_categories.csv`
4. Run `tally run --summary` to verify improvement
5. Repeat until Unknown < 5% of total

The `discover` command provides suggested patterns and merchant names to speed up this process.

## Pattern Syntax Quick Reference

The Pattern column uses Python regex (case-insensitive):

| Pattern | Matches |
|---------|---------|
| `NETFLIX` | Contains "NETFLIX" |
| `DELTA\|UNITED` | "DELTA" or "UNITED" |
| `UBER\s(?!EATS)` | "UBER " not followed by "EATS" |
| `COSTCO(?!.*GAS)` | "COSTCO" without "GAS" |
| `^ATT\s` | Starts with "ATT " |

### Inline Modifiers (Target Specific Transactions)

Add conditions to target transactions by amount or date:

```csv
COSTCO[amount>200],Costco Bulk,Shopping,Bulk
BESTBUY[amount=499.99][date=2025-01-15],TV Purchase,Shopping,Electronics
HOLIDAY[month=12],December Shopping,Shopping,Gifts
```

**Amount:** `[amount>N]`, `[amount<N]`, `[amount=N]`, `[amount:MIN-MAX]`
**Date:** `[date=YYYY-MM-DD]`, `[date:START..END]`, `[date:lastNdays]`, `[month=N]`

## Common Categories

- **Food**: Grocery, Restaurant, Fast Food, Coffee, Delivery
- **Shopping**: Online, Retail, Clothing, Electronics, Home
- **Travel**: Airline, Lodging, Car Rental
- **Transport**: Rideshare, Gas, Parking
- **Subscriptions**: Streaming, Software
- **Health**: Gym, Pharmacy, Medical
- **Bills**: Rent, Mortgage, Insurance, Utilities
- **Transfers**: P2P, CC Payment

## Travel Detection

International transactions are automatically classified as travel.
Domestic out-of-state is NOT auto-travel (opt-in via merchant rules).

To mark a domestic location as travel, add to merchant_categories.csv:
```csv
.*\sHI$,Hawaii Trip,Travel,Hawaii
```

## Important Notes

- Statement files in `data/` contain PII - never commit or display raw contents
- First matching rule wins - put specific patterns before general ones
- Test patterns at regex101.com (Python flavor)
- The tool auto-cleans prefixes like APLPAY, SQ*, TST*

## Data Format Requirements

The tool supports multiple data formats:

### Predefined Types
- **AMEX**: CSV with Date,Description,Amount columns (`type: amex`)
- **BOA**: TXT with "MM/DD/YYYY Description Amount Balance" per line (`type: boa`)

### Custom Format Strings
For any other CSV, use the `format` field with a format string:

```yaml
data_sources:
  - name: Chase
    file: data/chase.csv
    format: "{date:%m/%d/%Y}, {_}, {description}, {_}, {amount}"
  - name: BofA Checking
    file: data/bofa.csv
    format: "{date:%m/%d/%Y}, {description}, {-amount}"  # Bank: negative = expense
```

**Format string tokens:**
- `{date:%m/%d/%Y}` - Date column (with strptime format)
- `{description}` - Description/merchant column
- `{amount}` - Amount column (positive = expense)
- `{-amount}` - Negate amounts (for bank accounts where negative = expense)
- `{location}` - Optional location column
- `{_}` - Skip column

**Sign conventions:**
- Credit cards typically show charges as positive → use `{amount}`
- Bank accounts typically show debits as negative → use `{-amount}`

Use `tally inspect <file>` to see the CSV structure before creating a format string.
