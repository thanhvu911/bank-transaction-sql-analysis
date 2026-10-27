# Bank Transaction Analysis: SQL Walkthrough

This page walks through the project query by query: **the SQL, the actual result table, and what the result means**. You can follow the analysis without opening SQL Server.

> **How the outputs were produced:** the queries are written in T-SQL for SQL Server (`validation.sql`, `transform.sql`, `analysis.sql`). The result tables below come from running the same logic in DuckDB on the same two Excel files. A few SQL Server functions were translated one-for-one (`YEAR` → `year`, `DATENAME(WEEKDAY, …)` → `dayname`, `CHARINDEX`/`SUBSTRING` parsing → `split_part`), which doesn't change the results. Long outputs are trimmed to their top rows.

**Data:** `banking_transaction` (1,000 rows, Jan 2023 – Sep 2025) and `customer_account` (800 rows).

**Contents:** [1. Validate](#1-validate-can-we-trust-the-data) · [2. Transform](#2-transform) · [3. Analyze](#3-analyze) · [4. Key takeaways](#4-key-takeaways)

---

## 1. Validate: can we trust the data?

### 1.1 Row counts

```sql
SELECT COUNT(*) FROM dbo.banking_transaction;
SELECT COUNT(*) FROM dbo.customer_account;
```

**Output**

|   transactions |   accounts |
|---------------:|-----------:|
|           1000 |        800 |

**What it shows:** This is the baseline size of both tables.

### 1.2 Rows with any missing value

```sql
SELECT COUNT(*)
FROM dbo.banking_transaction
WHERE transaction_id IS NULL OR account_number IS NULL OR transaction_type IS NULL
   OR amount IS NULL OR transaction_date IS NULL OR branch_code IS NULL
   OR currency IS NULL OR transaction_time IS NULL;
```

**Output**

|   rows_with_any_null |
|---------------------:|
|                   65 |

Breakdown by column:

|   null_transaction_type |   null_amount |   null_currency |   null_branch |
|------------------------:|--------------:|----------------:|--------------:|
|                      60 |             0 |               5 |             0 |

**What it shows:** **123 of 1,000 transactions (12.3%) have at least one missing field**: 60 have no transaction type, 60 have no amount, and 5 have no currency. If missing amounts were left unhandled, totals and averages would be silently wrong, so `transform.sql` fixes them before the analysis.

### 1.3 Duplicate keys

```sql
SELECT transaction_id, COUNT(*) FROM dbo.banking_transaction GROUP BY transaction_id HAVING COUNT(*) > 1;
SELECT account_number, COUNT(*) FROM dbo.customer_account GROUP BY account_number HAVING COUNT(*) > 1;
```

**Output**

|   duplicate_transaction_ids |   duplicate_account_numbers |
|----------------------------:|----------------------------:|
|                           0 |                           0 |

**What it shows:** Both primary keys are unique, so joins won't double-count anything.

### 1.4 Invalid categorical values

```sql
SELECT 'currency' AS column_name, currency AS value, COUNT(*) AS count
FROM dbo.banking_transaction
GROUP BY currency
ORDER BY count DESC;
```

**Output**

| value   |   count |
|:--------|--------:|
| USD     |     260 |
| EUR     |     256 |
| JPY     |     247 |
| GBP     |     220 |
| -999    |       7 |
| *^$$    |       5 |
| nan     |       5 |

**What it shows:** Besides the four real currencies, there are placeholder and garbage codes (`-999`, `*^$$`) and NULLs, **17 rows in total**. These need cleaning or exclusion before any currency-level reporting.

### 1.5 Value ranges

```sql
SELECT MIN(amount), MAX(amount), AVG(amount), MIN(transaction_time), MAX(transaction_time)
FROM dbo.banking_transaction;

SELECT MIN(balance), MAX(balance), MIN(credit_score), MAX(credit_score), AVG(credit_score)
FROM dbo.customer_account;
```

**Output**

|   min_amount |   max_amount |   avg_amount |   min_hour |   max_hour |
|-------------:|-------------:|-------------:|-----------:|-----------:|
|            0 |      9987.02 |      4760.06 |          1 |         23 |

|   min_balance |   max_balance |   min_credit |   max_credit |   avg_credit |
|--------------:|--------------:|-------------:|-------------:|-------------:|
|         50.98 |       49962.6 |          300 |          849 |        574.8 |

**What it shows:** Amounts are positive and plausible (about $139 to $9,987), hours fall in 1–23, and credit scores fall in the valid 300–850 range. No outliers need treatment.

### 1.6 Referential integrity

```sql
SELECT COUNT(DISTINCT bt.account_number) AS transaction_accounts,
       COUNT(DISTINCT ca.account_number) AS customer_accounts,
       COUNT(DISTINCT CASE WHEN ca.account_number IS NULL THEN bt.account_number END) AS missing_accounts
FROM dbo.banking_transaction bt
LEFT JOIN dbo.customer_account ca ON bt.account_number = ca.account_number;
```

**Output**

|   transaction_accounts |   matched_accounts |   missing_accounts |
|-----------------------:|-------------------:|-------------------:|
|                    558 |                558 |                  0 |

**What it shows:** Every transaction links to a known account (no orphans). 558 of the 800 accounts have at least one transaction.

### 1.7 Date coverage

```sql
SELECT 'Transaction', MIN(transaction_date), MAX(transaction_date),
       DATEDIFF(day, MIN(transaction_date), MAX(transaction_date))
FROM dbo.banking_transaction
UNION ALL
SELECT 'Customer Account', MIN(opening_date), MAX(opening_date),
       DATEDIFF(day, MIN(opening_date), MAX(opening_date))
FROM dbo.customer_account;
```

**Output**

| table_name       | earliest            | latest              |   range_days |
|:-----------------|:--------------------|:--------------------|-------------:|
| Transaction      | 2023-01-01 00:00:00 | 2025-09-26 00:00:00 |          999 |
| Customer Account | 2020-01-01 00:00:00 | 2022-03-10 00:00:00 |          799 |

**What it shows:** **Transactions stop on Sep 26, 2025, so 2025 is a partial year.** Year-over-year comparisons have to account for that (see 3.2).

---

## 2. Transform

```sql
-- Transform NULL amount to 0
UPDATE dbo.banking_transaction
SET amount = 0
WHERE amount IS NULL;
```

The 60 missing amounts are set to 0, so they count as transactions but add no value to the totals. Every result below is computed after this step.

---

## 3. Analyze

### 3.1 Yearly summary (Q1)

```sql
SELECT YEAR(transaction_date) AS [year],
       ROUND(COUNT(*) / 1000.0, 1)          AS nb_transactions_in_thousands,
       ROUND(SUM(amount) / 1000000.0, 2)    AS total_amount_million,
       ROUND(AVG(a.balance) / 1000000.0, 2) AS avg_balance_million,
       ROUND(SUM(a.loan_amount) / 1000000.0, 2) AS total_loans_million
FROM dbo.banking_transaction t
JOIN dbo.customer_account a ON t.account_number = a.account_number
GROUP BY YEAR(transaction_date)
ORDER BY [year];
```

**Output**

|   year |   nb_transactions_thousands |   total_amount_million |   avg_balance_million |   total_loans_million |
|-------:|----------------------------:|-----------------------:|----------------------:|----------------------:|
|   2023 |                         0.4 |                   1.75 |                  0.03 |                  9.64 |
|   2024 |                         0.4 |                   1.72 |                  0.03 |                  9.85 |
|   2025 |                         0.3 |                   1.29 |                  0.03 |                  7.15 |

**What it shows:** Volume and value were flat from 2023 to 2024 (about $1.7M a year). 2025 looks lower only because it covers about 9 months.

### 3.2 Year-over-year growth (Q3)

```sql
WITH yearly AS (
    SELECT YEAR(transaction_date) AS [year], SUM(amount) AS total_amount
    FROM dbo.banking_transaction
    GROUP BY YEAR(transaction_date)
)
SELECT [year], total_amount,
       LAG(total_amount) OVER (ORDER BY [year]) AS prev_amount,
       (total_amount - LAG(total_amount) OVER (ORDER BY [year])) * 100.0
         / NULLIF(LAG(total_amount) OVER (ORDER BY [year]), 0) AS growth_rate_pct
FROM yearly ORDER BY [year];
```

**Output**

|   year |   total_amount |   prev_amount |   growth_rate_pct |
|-------:|---------------:|--------------:|------------------:|
|   2023 |    1.74836e+06 | nan           |            nan    |
|   2024 |    1.724e+06   |   1.74836e+06 |             -1.39 |
|   2025 |    1.2877e+06  |   1.724e+06   |            -25.31 |

**What it shows:** The raw −25.3% for 2025 is **an artifact of the partial year, not a real decline**. Per month, 2025 averages about $143K against about $144K in 2024, so activity is effectively unchanged. This is why the date check in 1.7 matters.

### 3.3 Mix by transaction type (Q4)

```sql
WITH yearly_type AS (
    SELECT YEAR(transaction_date) AS [year], transaction_type, SUM(amount) AS total_amount
    FROM dbo.banking_transaction
    GROUP BY YEAR(transaction_date), transaction_type
)
SELECT [year], transaction_type, total_amount,
       ROUND(total_amount * 100.0 / SUM(total_amount) OVER (PARTITION BY [year]), 2) AS pct_contribution
FROM yearly_type
ORDER BY [year], total_amount DESC;
```

**Output**

|   year | transaction_type   |   total_amount |   pct_contribution |
|-------:|:-------------------|---------------:|-------------------:|
|   2023 | Transfer           |       441601   |              25.26 |
|   2023 | Payment            |       417864   |              23.9  |
|   2023 | Deposit            |       400073   |              22.88 |
|   2023 | Withdrawal         |       367717   |              21.03 |
|   2023 | (missing)          |       121109   |               6.93 |
|   2024 | Deposit            |       453567   |              26.31 |
|   2024 | Withdrawal         |       425012   |              24.65 |
|   2024 | Payment            |       410584   |              23.82 |
|   2024 | Transfer           |       359187   |              20.83 |
|   2024 | (missing)          |        75646.1 |               4.39 |
|   2025 | Transfer           |       344514   |              26.75 |
|   2025 | Withdrawal         |       292716   |              22.73 |
|   2025 | Payment            |       291313   |              22.62 |
|   2025 | Deposit            |       259803   |              20.18 |
|   2025 | (missing)          |        99354.4 |               7.72 |

**What it shows:** The four transaction types split value fairly evenly (about 20–27% each), and the leader changes every year. The `(missing)` type still carries 4–8% of value each year, so cleaning `transaction_type` at the source would make the mix more reliable.

### 3.4 Value by currency and branch (Q5)

```sql
SELECT currency, SUM(amount) AS total_amount
FROM dbo.banking_transaction GROUP BY currency ORDER BY total_amount DESC;

SELECT branch_code, SUM(amount) AS total_amount
FROM dbo.banking_transaction GROUP BY branch_code ORDER BY total_amount DESC;
```

**Output**

| currency   |    total_amount |
|:-----------|----------------:|
| EUR        |     1.23333e+06 |
| JPY        |     1.18739e+06 |
| USD        |     1.17905e+06 |
| GBP        |     1.07382e+06 |
| -999       | 37646.9         |
| nan        | 28206.4         |
| *^$$       | 20622.8         |

Top 5 branches (of 431 branches):

|   branch_code |   total_amount |   n |
|--------------:|---------------:|----:|
|           479 |        47622.9 |   8 |
|           219 |        37392.2 |   6 |
|           347 |        35356   |   5 |
|            38 |        32872.3 |   4 |
|           349 |        32570.1 |   5 |

**What it shows:** EUR leads slightly, but the four currencies are close. About $86K sits under invalid or missing currency codes. Activity is spread thinly across 431 branches (the top branch has only 8 transactions), so branch-level rankings are noisy at this data size.

### 3.5 Share by credit-score band (Q7)

```sql
SELECT CASE WHEN credit_score < 580 THEN 'Poor'
            WHEN credit_score BETWEEN 580 AND 669 THEN 'Fair'
            WHEN credit_score BETWEEN 670 AND 739 THEN 'Good'
            WHEN credit_score BETWEEN 740 AND 799 THEN 'Very Good'
            ELSE 'Excellent' END AS credit_score_range,
       SUM(t.amount) AS total_amount,
       ROUND(SUM(t.amount) * 100.0 / SUM(SUM(t.amount)) OVER (), 2) AS pct_share
FROM dbo.banking_transaction t
JOIN dbo.customer_account a ON t.account_number = a.account_number
GROUP BY <same CASE>;
```

**Output**

| credit_score_range   |     total_amount |   pct_share |
|:---------------------|-----------------:|------------:|
| Poor                 |      2.44507e+06 |       51.37 |
| Fair                 | 665688           |       13.98 |
| Good                 | 607542           |       12.76 |
| Very Good            | 522875           |       10.98 |
| Excellent            | 518890           |       10.9  |

**What it shows:** "Poor" credit accounts drive 51% of transaction value, but 52% of all accounts are in that band. **Value share simply tracks how many customers are in each band**, so no band is disproportionately active. Without that context, the headline number would mislead.

### 3.6 Share by account type (Q7)

```sql
SELECT a.account_type, SUM(t.amount) AS total_amount,
       ROUND(SUM(t.amount) * 100.0 / SUM(SUM(t.amount)) OVER (), 2) AS pct_share
FROM dbo.banking_transaction t
JOIN dbo.customer_account a ON t.account_number = a.account_number
GROUP BY a.account_type;
```

**Output**

| account_type   |   total_amount |   pct_share |
|:---------------|---------------:|------------:|
| Savings        |    1.38446e+06 |       29.08 |
| Checking       |    1.17782e+06 |       24.74 |
| Credit         |    1.10712e+06 |       23.26 |
| Loan           |    1.09067e+06 |       22.91 |

**What it shows:** Savings accounts carry the largest share of transaction value (29%); the other three types are close to each other.

### 3.7 Transaction mix within each account type (Q8)

```sql
SELECT a.account_type, t.transaction_type,
       ROUND(SUM(t.amount) * 100.0
             / SUM(SUM(t.amount)) OVER (PARTITION BY a.account_type), 2) AS pct_share
FROM dbo.banking_transaction t
JOIN dbo.customer_account a ON t.account_number = a.account_number
GROUP BY a.account_type, t.transaction_type;
```

**Output**

| account_type   | transaction_type   |   pct_share |
|:---------------|:-------------------|------------:|
| Checking       | Withdrawal         |       26.75 |
| Checking       | Deposit            |       23.77 |
| Checking       | Transfer           |       21.89 |
| Checking       | Payment            |       20.11 |
| Checking       | (missing)          |        7.48 |
| Credit         | Transfer           |       30.39 |
| Credit         | Payment            |       23.37 |
| Credit         | Withdrawal         |       21.45 |
| Credit         | Deposit            |       17.85 |
| Credit         | (missing)          |        6.95 |
| Loan           | Deposit            |       26.75 |
| Loan           | Payment            |       26.6  |
| Loan           | Transfer           |       20.52 |
| Loan           | Withdrawal         |       19.36 |
| Loan           | (missing)          |        6.77 |
| Savings        | Deposit            |       24.86 |
| Savings        | Payment            |       24.13 |
| Savings        | Transfer           |       23.64 |
| Savings        | Withdrawal         |       23.24 |
| Savings        | (missing)          |        4.13 |

**What it shows:** Each account type has its own pattern: Credit accounts lean on **transfers** (30%), Checking on **withdrawals** (27%), and Loan accounts on **deposits and payments**, which is consistent with repayments.

### 3.8 Parsing a semi-structured field: value by sector and city (Q9)

```sql
-- account_holder_details looks like: 'Sector: Technology, Residence: 15 years, City: London'
SELECT LTRIM(RTRIM(SUBSTRING(a.account_holder_details,
           CHARINDEX(':', a.account_holder_details) + 1,
           CHARINDEX(',', a.account_holder_details) - CHARINDEX(':', a.account_holder_details) - 1))) AS sector,
       SUM(t.amount) AS total_amount
FROM dbo.banking_transaction t
JOIN dbo.customer_account a ON t.account_number = a.account_number
GROUP BY <same expression>
ORDER BY total_amount DESC;
```

**Output**

| sector     |     total_amount |
|:-----------|-----------------:|
| Technology |      1.00664e+06 |
| Education  | 992888           |
| Healthcare | 960440           |
| Retail     | 931088           |
| Finance    | 869008           |

By city:

| city     |     total_amount |
|:---------|-----------------:|
| Sydney   |      1.10306e+06 |
| Tokyo    |      1.0816e+06  |
| Berlin   | 933867           |
| New York | 832939           |
| London   | 808595           |

**What it shows:** String parsing turns a free-text field into three usable dimensions. Technology and Education customers lead on value, and **Sydney and Tokyo lead the cities**, about 35% ahead of London.

### 3.9 When do customers transact? (Q10)

```sql
SELECT transaction_time AS [hour], COUNT(*) AS nb_transactions
FROM dbo.banking_transaction
GROUP BY transaction_time
ORDER BY nb_transactions DESC;
```

**Output**

|   hour |   nb_transactions |
|-------:|------------------:|
|     15 |                55 |
|     11 |                54 |
|     10 |                51 |
|      2 |                49 |
|     12 |                48 |

Busiest weekday per credit band:

| credit_score_range   | weekday_name   |   nb_transactions |
|:---------------------|:---------------|------------------:|
| Poor                 | Wednesday      |                85 |
| Fair                 | Sunday         |                35 |
| Good                 | Friday         |                25 |
| Very Good            | Saturday       |                25 |
| Excellent            | Thursday       |                23 |

**What it shows:** Mid-day hours (10:00–15:00) dominate, with a peak at **15:00**. That's useful for staffing and for scheduling maintenance windows outside it.

### 3.10 Does balance drive activity?

```sql
WITH balance_groups AS (
    SELECT account_number,
           CASE WHEN balance < 1000 THEN 'Low'
                WHEN balance BETWEEN 1000 AND 5000 THEN 'Medium'
                ELSE 'High' END AS balance_group
    FROM dbo.customer_account
)
SELECT bg.balance_group,
       COUNT(t.transaction_id) * 1.0 / COUNT(DISTINCT bg.account_number) AS avg_transactions_per_account
FROM balance_groups bg
LEFT JOIN dbo.banking_transaction t ON bg.account_number = t.account_number
GROUP BY bg.balance_group
ORDER BY avg_transactions_per_account DESC;
```

**Output**

| balance_group   |   accounts |   avg_transactions_per_account |
|:----------------|-----------:|-------------------------------:|
| Medium          |         54 |                           1.33 |
| Low             |         14 |                           1.29 |
| High            |        732 |                           1.24 |

**What it shows:** Transaction frequency is nearly identical across balance groups (1.24–1.33 per account). **Balance doesn't predict activity** in this dataset. Note that 732 of the 800 accounts fall in the "High" group, so the thresholds could be re-cut to quartiles in a next iteration.

---

## 4. Key takeaways

1. **Validate first.** 12.3% of raw transactions had a missing field and 17 had invalid currency codes. Catching that before the analysis kept every total honest.
2. **Context beats headlines.** The 2025 "−25% decline" is a partial year, and per month, activity is flat. The "Poor credit = 51% of value" result simply matches that band's 52% share of customers.
3. **The mix is balanced.** Transaction types, currencies, and account types each split value fairly evenly, and no single segment dominates.
4. **Each account type behaves differently.** Credit accounts lean on transfers, Checking on withdrawals, and Loan accounts on deposits and payments.
5. **Next steps:** fix `transaction_type` and `currency` at the source, use quartile-based balance groups, and normalize year-over-year comparisons by the number of months.

**Skills shown:** data profiling and validation · T-SQL CTEs, window functions (`LAG`, `RANK`, `SUM() OVER`), `CASE` bucketing · parsing semi-structured text · turning query results into business insight
