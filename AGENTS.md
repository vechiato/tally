# Tally - Agent Instructions

This document provides instructions for AI agents working with Tally.

## Quick Reference

```bash
# Show help
tally

# Initialize a new budget directory (current dir or specified)
tally init
tally init ./my-budget

# Run analysis (uses ./config by default)
tally run
tally run ./path/to/config

# View summary only (find uncategorized transactions)
tally run --summary

# Inspect a CSV file to see its structure and get format suggestions
tally inspect path/to/file.csv

# Discover unknown merchants and get suggested rules
tally discover                    # Human-readable output
tally discover --format csv       # CSV output for import
tally discover --format json      # JSON output for programmatic use
tally discover --limit 50         # Show top 50 by spend
```

## Your Tasks

When working with this budget analyzer, you may be asked to:

1. **Set up a new budget directory** - Use `tally init`
2. **Add merchant categorization rules** - Edit `config/merchant_categories.csv`
3. **Configure data sources** - Edit `config/settings.yaml`
4. **Analyze and fix uncategorized transactions** - Run with `--summary`, then add rules

## Understanding merchant_categories.csv

This is the main file you'll edit. Each row maps a transaction pattern to a category.

**Format:** `Pattern,Merchant,Category,Subcategory`

**Pattern** is a Python regex (case-insensitive) matched against transaction descriptions.

### Pattern Examples

| Pattern | What it matches | Use case |
|---------|-----------------|----------|
| `NETFLIX` | Contains "NETFLIX" | Simple substring |
| `STARBUCKS` | Contains "STARBUCKS" | Simple substring |
| `DELTA\|SOUTHWEST` | "DELTA" OR "SOUTHWEST" | Multiple variations |
| `WHOLE FOODS\|WHOLEFDS` | Either spelling | Handle abbreviations |
| `UBER\s(?!EATS)` | "UBER " NOT followed by "EATS" | Exclude Uber Eats from rideshare |
| `COSTCO(?!.*GAS)` | "COSTCO" without "GAS" | Exclude gas station |
| `APPLE\.COM(?!/BILL)` | Apple.com but not /BILL | Exclude subscriptions |
| `^ATT\s` | Starts with "ATT " | Avoid matching "SEATTLE" |
| `CHICK.FIL.A` | CHICK-FIL-A or CHICKFILA | `.` matches any char |

### Inline Modifiers (Target Specific Transactions)

Add conditions to patterns to match on amount or date:

```csv
# Amount modifiers
COSTCO[amount>200],Costco Bulk,Shopping,Bulk
STARBUCKS[amount<10],Quick Coffee,Food,Coffee
BESTBUY[amount=499.99],TV Purchase,Shopping,Electronics
RESTAURANT[amount:20-100],Dining Out,Food,Restaurant

# Date modifiers
COSTCO[date=2025-01-15],Costco Jan 15,Shopping,Grocery
SUBSCRIPTION[date:2025-01-01..2025-06-30],H1 Subscription,Bills,Subscription
PURCHASE[date:last30days],Recent Purchase,Shopping,Retail
HOLIDAY[month=12],December Shopping,Shopping,Gifts

# Combined (AND logic)
COSTCO(?!GAS)[amount>200][date=2025-01-15],Specific Costco Trip,Shopping,Bulk
```

**Amount modifiers:**
- `[amount>100]` - Greater than $100
- `[amount>=100]` - Greater than or equal to $100
- `[amount<50]` - Less than $50
- `[amount<=50]` - Less than or equal to $50
- `[amount=99.99]` - Exactly $99.99
- `[amount:50-200]` - Between $50 and $200 (inclusive)

**Date modifiers:**
- `[date=2025-01-15]` - Exact date (YYYY-MM-DD format)
- `[date:2025-01-01..2025-01-31]` - Date range (inclusive)
- `[date:last30days]` - Within last N days
- `[month=12]` - Any transaction in December (any year)

Use modifiers to express rules like *"that $500 Best Buy purchase was a gift"* or *"Costco purchases over $200 are bulk shopping"*.

### Adding New Rules

1. Look at the raw transaction description from the bank statement
2. Find a unique substring or pattern that identifies the merchant
3. Add a row: `PATTERN,Clean Name,Category,Subcategory`

**Example:** If you see `"WHOLEFDS MKT 10847"` in a statement:
```csv
WHOLEFDS,Whole Foods,Food,Grocery
```

### Rule Order Matters

Rules are matched top-to-bottom. Put specific rules before general ones:

```csv
# Specific first
UBER\s*EATS,Uber Eats,Food,Delivery
# General second
UBER,Uber,Transport,Rideshare
```

### Standard Categories

Use these categories for consistency:

| Category | Subcategories |
|----------|---------------|
| Food | Grocery, Restaurant, Fast Food, Fast Casual, Coffee, Delivery, Bakery |
| Shopping | Online, Retail, Clothing, Electronics, Home, Kids, Beauty, Books |
| Travel | Airline, Lodging, Car Rental, Agency |
| Transport | Rideshare, Gas, Parking, Tolls, Auto Service |
| Subscriptions | Streaming, Software, News |
| Health | Gym, Pharmacy, Medical, Vision, Fitness |
| Utilities | Mobile, Internet/TV, Electric, Water |
| Entertainment | Movies, Events, Activities, Attractions |
| Transfers | P2P, CC Payment, Investment, Transfer |
| Bills | Mortgage, Insurance, Tax |
| Personal | Childcare, Grooming, Spa |
| Cash | ATM, Check |

## Workflow: Adding Rules for Uncategorized Transactions

### Method 1: Using the discover command (Recommended for agents)

1. Run discover to find unknown merchants sorted by spend:
   ```bash
   tally discover --format json
   ```

2. The output includes:
   - `raw_description`: The original transaction description
   - `suggested_pattern`: A regex pattern to match it
   - `suggested_merchant`: A clean merchant name
   - `count`: Number of transactions
   - `total_spend`: Total amount spent

3. For each unknown merchant:
   - Review the suggested pattern and merchant name
   - Determine the appropriate Category and Subcategory
   - Add to `merchant_categories.csv`

4. Re-run to verify:
   ```bash
   tally run --summary
   ```

### Method 2: Manual inspection

1. Run analysis to find unknown merchants:
   ```bash
   tally run --summary
   ```

2. Look for transactions categorized as "Unknown"

3. For each unknown merchant:
   - Find the raw description in the statement file
   - Create a pattern that uniquely matches it
   - Add to `merchant_categories.csv`

4. Re-run to verify categorization

## Using discover for Bulk Rule Creation

The discover command is designed to help agents efficiently create rules:

```bash
# Get JSON output for programmatic processing
tally discover --format json --limit 0

# Get CSV output ready for import (just needs categories filled in)
tally discover --format csv
```

### JSON Output Structure

```json
[
  {
    "raw_description": "STARBUCKS STORE 12345 SEATTLE WA",
    "suggested_pattern": "STARBUCKS\s*STORE",
    "suggested_merchant": "Starbucks Store",
    "count": 15,
    "total_spend": 87.50,
    "examples": [
      {"date": "2025-01-15", "amount": -5.50, "description": "Starbucks Store"}
    ]
  }
]
```

### Workflow for Agents

1. Run `tally discover --format json --limit 0`
2. Parse the JSON output
3. For each unknown merchant:
   - Use `suggested_pattern` as starting point (may need refinement)
   - Use `suggested_merchant` as the merchant name
   - Determine Category/Subcategory based on merchant type
4. Append rules to `config/merchant_categories.csv`
5. Run `tally run --summary` to verify improvement
6. Repeat until Unknown transactions are minimized

## File Locations

```
my-budget/
├── config/
│   ├── settings.yaml           # Data sources, year, output settings
│   └── merchant_categories.csv # Pattern → Category rules (EDIT THIS)
├── data/                       # Bank/CC statement exports
└── output/                     # Generated reports
```

## Travel Detection

International transactions are automatically classified as travel.
Domestic out-of-state transactions are NOT auto-travel (opt-in via merchant rules).

To mark domestic locations as travel, add patterns to merchant_categories.csv:
```csv
.*\sHI$,Hawaii Trip,Travel,Hawaii
.*\sCA$,California Trip,Travel,California
```

Configure home in settings.yaml:
```yaml
# Optional: specify home locations (for international exclusions)
home_locations:
  - WA

# Optional: pretty names for travel destinations
travel_labels:
  HI: Hawaii
  GB: United Kingdom
```

If `home_locations` is not specified, it's auto-detected from your most common transaction location.

## Statement Formats and Custom Parsing

The tool supports three ways to parse CSV files:

### 1. Predefined Types (backward compatible)
```yaml
data_sources:
  - name: AMEX
    file: data/amex.csv
    type: amex      # Expects Date,Description,Amount columns
  - name: BOA
    file: data/boa.txt
    type: boa       # Expects "MM/DD/YYYY Description Amount Balance" lines
```

### 2. Custom Format Strings (for any CSV)
Use a format string to specify column mappings:
```yaml
data_sources:
  - name: Chase
    file: data/chase.csv
    format: "{date:%m/%d/%Y}, {_}, {description}, {_}, {_}, {amount}"
  - name: BofA Checking
    file: data/bofa.csv
    format: "{date:%m/%d/%Y}, {description}, {-amount}"  # Bank: negative = expense
```

**Format string syntax:**
- `{date:%m/%d/%Y}` - Date column with strptime format
- `{description}` - Transaction description column
- `{amount}` - Amount column (positive = expense)
- `{-amount}` - Negate amounts (for bank accounts where negative = expense)
- `{location}` - Optional location/state column
- `{_}` - Skip this column

**Sign conventions:**
- Credit cards typically show charges as positive → use `{amount}`
- Bank accounts typically show debits as negative → use `{-amount}`

Position in the string = column index (0-based).

### Discovering CSV Structure

Use the **inspect** command to analyze an unknown CSV:
```bash
tally inspect path/to/file.csv
```

This shows column headers, indices, and sample data rows.

### Workflow: Creating a Format String for Any CSV

**Step 1: Inspect the file**
```bash
tally inspect data/newbank.csv
```

**Step 2: Identify the columns**
Look at the output and find:
- Which column has the **date** (and what format: MM/DD/YYYY, YYYY-MM-DD, etc.)
- Which column has the **description** (merchant name)
- Which column has the **amount**
- Optionally, which column has **location** (state/country code)

**Step 3: Build the format string**
For each column position (0, 1, 2, ...), add:
- `{date:%m/%d/%Y}` if it's the date column (adjust format as needed)
- `{description}` if it's the description column
- `{amount}` if it's the amount column
- `{location}` if it's a location column
- `{_}` for any columns to skip

**Example:** A CSV with columns: Transaction Date, Post Date, Description, Category, Amount

```yaml
format: "{date:%m/%d/%Y}, {_}, {description}, {_}, {amount}"
#         col 0           col 1  col 2       col 3  col 4
```

**Common date formats:**
- `%m/%d/%Y` - 01/15/2024 (US format)
- `%Y-%m-%d` - 2024-01-15 (ISO format)
- `%d/%m/%Y` - 15/01/2024 (European format)
- `%m/%d/%y` - 01/15/24 (2-digit year)

**Step 4: Add to settings.yaml and run**
```bash
tally run
```

Transaction descriptions look like:
- AMEX: `"NETFLIX.COM"`, `"UBER *EATS"`, `"STARBUCKS STORE 12345 SEATTLE WA"`
- BOA: `"NETFLIX.COM DES:RECURRING ID:xxx"`, `"ZELLE TO JOHN DOE"`

## Common Tasks

### Task: User wants to analyze their spending
1. Ensure `config/settings.yaml` has correct data sources
2. Run `tally run`
3. Open the HTML report in `output/`

### Task: User has many "Unknown" transactions
1. Run `tally discover --format json` to get unknowns sorted by spend
2. For each unknown merchant, determine appropriate Category/Subcategory
3. Add patterns to `merchant_categories.csv`
4. Run `tally run --summary` to verify improvement
5. Repeat until unknowns are minimized

### Task: User wants to track a specific merchant
1. Get the exact description from their statement
2. Create a pattern that matches it
3. Add to `merchant_categories.csv` with appropriate category

### Task: User wants to separate Costco groceries from Costco gas
```csv
COSTCO\s*GAS,Costco Gas,Transport,Gas
COSTCO(?!\s*GAS),Costco,Food,Grocery
```
(Gas rule must come first)

## Tips

- Run `tally` with no args to see help
- Test regex patterns at regex101.com (Python flavor)
- Comments start with `#` in CSV files
- Escape special regex chars: `\.` for literal dot, `\*` for literal asterisk
- The tool cleans common prefixes (APLPAY, SQ*, TST*) automatically

---

## Real-World Workflow Example

Here's a typical workflow when analyzing a new year's spending data:

### 1. Check for existing config
```bash
# Look for existing configs from other years to reuse
ls ../2024/config/ ../2025/config/ 2>/dev/null
```

### 2. Examine statement file formats
The tool expects specific formats. Your files may need transformation:

**Expected AMEX format (CSV):**
```csv
Date,Description,Amount
01/15/2024,AMAZON.COM,-45.99
01/16/2024,STARBUCKS STORE 12345,-6.50
```

**Expected BOA format (TXT, space-separated):**
```
01/15/2024 NETFLIX.COM DES:RECURRING -15.99 1234.56
01/16/2024 ZELLE TO JOHN DOE -100.00 1134.56
```

### 3. Transform data if needed
If your statement files have different formats, transform them with Python:

**Example: Transform multi-line AMEX export:**
```python
import csv
with open('Amex_raw.csv', 'r') as f:
    reader = csv.reader(f)
    # Extract date, description, amount from your specific format
    # Write to clean CSV with Date,Description,Amount columns
```

**Example: Transform BOA CSV to TXT:**
```python
import csv
with open('BOA.csv', 'r') as f:
    reader = csv.DictReader(f)
    with open('data/boa_clean.txt', 'w') as out:
        for row in reader:
            # Format: MM/DD/YYYY Description Amount Balance
            out.write(f"{row['Date']} {row['Description']} {row['Amount']} {row['Balance']}\n")
```

### 4. Copy existing merchant categories
If another year has good patterns, start with those:
```bash
cp ../2025/config/merchant_categories.csv config/
```

### 5. Run analysis and iterate
```bash
# Initial run - will have many "Unknown"
tally run

# Check what's unknown
tally run --summary --category Unknown

# Extract unique unknown patterns for analysis
python3 << 'EOF'
import csv
unknowns = {}
with open('output/transactions.csv') as f:  # or parse from summary
    # Group by description pattern, sum amounts
    pass
# Print top unknowns by spend
EOF
```

### 6. Add patterns in batches
Add patterns for the highest-spend unknowns first:

```csv
# High-value unknowns from 2024
BMWFINANCIAL|BMW FINANCIAL,BMW Financial Services,Bills,Auto Loan
ASHTON BELLEVUE,Ashton Apartments (Rent),Bills,Rent
MICROSOFT DES:EDIPAYMENT,Microsoft Payroll,Income,Salary
```

### 7. Iterate until Unknown < 5%
Re-run after each batch of patterns until categorization rate is acceptable:
```bash
tally run  # Check Unknown total
# Add more patterns
tally run  # Verify improvement
```

### Common Data Issues

**BOA files with summary headers:**
```python
# Skip header rows before the actual data
for row in reader:
    if row['Date'] and '/' in row['Date']:  # Skip summary rows
        # Process transaction
```

**AMEX multi-line format:**
Each transaction may span multiple lines. Look for date patterns to identify record boundaries.

**CHECKCARD prefix in BOA:**
BOA often prefixes with "CHECKCARD": add patterns like `CHECKCARD.*STARBUCKS`

**State/location suffixes:**
Descriptions often end with location: `STARBUCKS SEATTLE WA` - the tool handles this automatically.
