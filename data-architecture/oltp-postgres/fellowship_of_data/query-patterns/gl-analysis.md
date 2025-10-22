# General Ledger Analysis Query Patterns

**Purpose**: Common GL analysis queries with optimization guidance for financial reporting

**Source Tables**: `jwsdatagrc_ironhide.gl_posting_archive_*`, `jde_raw.f0901`, `jde_raw.f0101`

---

## Pattern 1: Account Balance as of Date

**Business Question**: "What's the balance of this account as of [date]?"

**Query**:
```sql
-- Current year account balance
SELECT
    account_code,
    SUM(CASE WHEN debit_credit_flag = 'D' THEN amount ELSE -amount END) as balance
FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
WHERE account_code = :account_code
  AND posting_date <= :as_of_date
GROUP BY account_code
```

**Optimization**:
- ✅ Uses composite index: (account_code, posting_date)
- ✅ Partition pruning if querying single year
- ✅ Date filtering benefits from index

**Expected Performance**: <50ms (current year partition with index)

**Multi-Year Variation**:
```sql
-- Account balance across years (union partitions)
SELECT
    account_code,
    SUM(balance) as total_balance
FROM (
    SELECT account_code,
           SUM(CASE WHEN debit_credit_flag = 'D' THEN amount ELSE -amount END) as balance
    FROM jwsdatagrc_ironhide.gl_posting_archive_y2024
    WHERE account_code = :account_code AND posting_date <= :as_of_date
    GROUP BY account_code

    UNION ALL

    SELECT account_code,
           SUM(CASE WHEN debit_credit_flag = 'D' THEN amount ELSE -amount END) as balance
    FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
    WHERE account_code = :account_code AND posting_date <= :as_of_date
    GROUP BY account_code
) all_years
GROUP BY account_code
```

**Optimization**: Manual UNION ALL for partition pruning (PostgreSQL doesn't auto-partition across years)

---

## Pattern 2: Trial Balance Report

**Business Question**: "Show me a trial balance for [fiscal period]?"

**Query**:
```sql
-- Trial balance for fiscal period
SELECT
    account_code,
    account_description,
    SUM(CASE WHEN debit_credit_flag = 'D' THEN amount ELSE 0 END) as total_debits,
    SUM(CASE WHEN debit_credit_flag = 'C' THEN amount ELSE 0 END) as total_credits,
    SUM(CASE WHEN debit_credit_flag = 'D' THEN amount ELSE -amount END) as net_balance
FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
WHERE fiscal_year = :fiscal_year
  AND fiscal_period = :fiscal_period
GROUP BY account_code, account_description
ORDER BY account_code
```

**Optimization**:
- ✅ Partition pruning (single year)
- ✅ Composite index on (fiscal_year, fiscal_period)
- ✅ Single table scan with grouping

**Expected Performance**: <500ms (full period aggregation)

**Period-to-Date Variation**:
```sql
-- Trial balance for fiscal year-to-date
SELECT
    account_code,
    account_description,
    SUM(CASE WHEN debit_credit_flag = 'D' THEN amount ELSE 0 END) as ytd_debits,
    SUM(CASE WHEN debit_credit_flag = 'C' THEN amount ELSE 0 END) as ytd_credits,
    SUM(CASE WHEN debit_credit_flag = 'D' THEN amount ELSE -amount END) as ytd_balance
FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
WHERE fiscal_year = :fiscal_year
  AND fiscal_period <= :current_period
GROUP BY account_code, account_description
ORDER BY account_code
```

---

## Pattern 3: GL Activity by Account (Time-Based)

**Business Question**: "Show me all activity for this account in [date range]?"

**Query**:
```sql
-- Account transaction detail
SELECT
    posting_date,
    document_number,
    document_type,
    description,
    debit_credit_flag,
    amount,
    fiscal_year,
    fiscal_period
FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
WHERE account_code = :account_code
  AND posting_date BETWEEN :start_date AND :end_date
ORDER BY posting_date, document_number
```

**Optimization**:
- ✅ Composite index: (account_code, posting_date)
- ✅ Perfect for this query pattern
- ✅ Partition pruning if single year

**Expected Performance**: <100ms (indexed range scan)

**Multi-Account Variation**:
```sql
-- Multiple accounts activity
SELECT
    account_code,
    posting_date,
    document_number,
    description,
    debit_credit_flag,
    amount
FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
WHERE account_code IN (:account_list)
  AND posting_date BETWEEN :start_date AND :end_date
ORDER BY account_code, posting_date
```

**Optimization**: Index supports IN clause for multiple accounts

---

## Pattern 4: Fiscal Period Summary

**Business Question**: "Summarize GL activity by fiscal period for [year]?"

**Query**:
```sql
-- Fiscal period rollup
SELECT
    fiscal_year,
    fiscal_period,
    COUNT(*) as transaction_count,
    SUM(CASE WHEN debit_credit_flag = 'D' THEN amount ELSE 0 END) as period_debits,
    SUM(CASE WHEN debit_credit_flag = 'C' THEN amount ELSE 0 END) as period_credits,
    SUM(CASE WHEN debit_credit_flag = 'D' THEN amount ELSE -amount END) as period_net
FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
WHERE fiscal_year = :fiscal_year
GROUP BY fiscal_year, fiscal_period
ORDER BY fiscal_period
```

**Optimization**:
- ✅ Partition pruning (single year)
- ✅ Composite index on (fiscal_year, fiscal_period)
- ✅ Efficient aggregation

**Expected Performance**: <200ms (12 periods aggregated)

**Account-Level Breakdown**:
```sql
-- Fiscal period summary by account
SELECT
    fiscal_year,
    fiscal_period,
    account_code,
    COUNT(*) as transaction_count,
    SUM(CASE WHEN debit_credit_flag = 'D' THEN amount ELSE -amount END) as period_balance
FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
WHERE fiscal_year = :fiscal_year
  AND account_code LIKE :account_pattern  -- e.g., '1000-%' for all cash accounts
GROUP BY fiscal_year, fiscal_period, account_code
ORDER BY fiscal_period, account_code
```

---

## Pattern 5: Year-Over-Year GL Comparison

**Business Question**: "Compare this year's GL activity to last year?"

**Query**:
```sql
-- Year-over-year comparison (requires UNION across partitions)
WITH current_year AS (
    SELECT
        fiscal_period,
        SUM(CASE WHEN debit_credit_flag = 'D' THEN amount ELSE -amount END) as current_balance
    FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
    WHERE fiscal_year = 2025
      AND account_code = :account_code
    GROUP BY fiscal_period
),
prior_year AS (
    SELECT
        fiscal_period,
        SUM(CASE WHEN debit_credit_flag = 'D' THEN amount ELSE -amount END) as prior_balance
    FROM jwsdatagrc_ironhide.gl_posting_archive_y2024
    WHERE fiscal_year = 2024
      AND account_code = :account_code
    GROUP BY fiscal_period
)
SELECT
    cy.fiscal_period,
    cy.current_balance,
    py.prior_balance,
    cy.current_balance - py.prior_balance as variance,
    ROUND(((cy.current_balance - py.prior_balance) / NULLIF(py.prior_balance, 0) * 100), 2) as variance_pct
FROM current_year cy
LEFT JOIN prior_year py ON cy.fiscal_period = py.fiscal_period
ORDER BY cy.fiscal_period
```

**Optimization**:
- ✅ CTEs for clarity
- ✅ Partition pruning (separate queries per year)
- ✅ Index on (account_code) for both partitions

**Expected Performance**: <300ms (2 partition scans + join)

---

## Pattern 6: GL Reconciliation (Source vs Processed)

**Business Question**: "Reconcile processed GL data back to JDE source?"

**Query**:
```sql
-- Reconcile processed GL to JDE source
WITH processed_total AS (
    SELECT
        fiscal_year,
        fiscal_period,
        account_code,
        SUM(CASE WHEN debit_credit_flag = 'D' THEN amount ELSE -amount END) as processed_balance
    FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
    WHERE fiscal_year = :fiscal_year
      AND fiscal_period = :fiscal_period
    GROUP BY fiscal_year, fiscal_period, account_code
),
source_total AS (
    SELECT
        EXTRACT(YEAR FROM posting_date) as fiscal_year,
        EXTRACT(MONTH FROM posting_date) as fiscal_period,
        account_id as account_code,
        SUM(CASE WHEN transaction_type = 'D' THEN amount ELSE -amount END) as source_balance
    FROM jde_raw.f0901
    WHERE posting_date >= :period_start_date
      AND posting_date <= :period_end_date
    GROUP BY EXTRACT(YEAR FROM posting_date), EXTRACT(MONTH FROM posting_date), account_id
)
SELECT
    p.account_code,
    p.processed_balance,
    s.source_balance,
    p.processed_balance - s.source_balance as variance
FROM processed_total p
FULL OUTER JOIN source_total s
    ON p.fiscal_year = s.fiscal_year
    AND p.fiscal_period = s.fiscal_period
    AND p.account_code = s.account_code
WHERE ABS(p.processed_balance - s.source_balance) > 0.01  -- Show only variances
ORDER BY ABS(p.processed_balance - s.source_balance) DESC
```

**Optimization**:
- ✅ Partition pruning (processed table)
- ✅ Date filtering (source table)
- ✅ FULL OUTER JOIN catches missing records in either system

**Expected Performance**: <2s (cross-schema join + aggregation)

**Use Case**: Data quality validation, ETL troubleshooting

---

## Pattern 7: High-Volume Account Analysis

**Business Question**: "Which accounts have the most transactions?"

**Query**:
```sql
-- High-volume account identification
SELECT
    account_code,
    account_description,
    COUNT(*) as transaction_count,
    SUM(amount) as total_amount,
    AVG(amount) as avg_transaction_amount,
    MIN(posting_date) as first_transaction,
    MAX(posting_date) as last_transaction
FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
WHERE fiscal_year = :fiscal_year
GROUP BY account_code, account_description
HAVING COUNT(*) > 1000  -- High-volume threshold
ORDER BY transaction_count DESC
LIMIT 50
```

**Optimization**:
- ✅ Partition pruning (single year)
- ✅ HAVING clause reduces result set
- ✅ LIMIT reduces output

**Expected Performance**: <1s (full partition scan with aggregation)

**Use Case**: Performance tuning, archival planning, account activity monitoring

---

## Performance Optimization Tips

### Partition Strategy
**Current Implementation**: Yearly partitions (gl_posting_archive_y2024, gl_posting_archive_y2025)

**Query Best Practices**:
1. **Filter by fiscal_year first**: Enables partition pruning
2. **Avoid cross-partition queries when possible**: Query single year at a time
3. **Use UNION ALL for multi-year**: Explicit partition access faster than auto-pruning
4. **Consider fiscal_period filtering**: Reduces scan size within partition

**Example - Partition-Aware Query**:
```sql
-- BAD (cross-partition scan)
SELECT * FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
WHERE posting_date BETWEEN '2024-12-01' AND '2025-01-31';

-- GOOD (explicit partition access)
SELECT * FROM jwsdatagrc_ironhide.gl_posting_archive_y2024
WHERE posting_date BETWEEN '2024-12-01' AND '2024-12-31'
UNION ALL
SELECT * FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
WHERE posting_date BETWEEN '2025-01-01' AND '2025-01-31';
```

### Index Usage
**Current Indexes**:
- **Composite**: (account_code, posting_date) - Most common query pattern
- **Composite**: (fiscal_year, fiscal_period) - Period-based reporting
- **Composite**: (document_number, fiscal_year) - Document lookup

**Query Optimization**:
1. **Lead with indexed columns**: account_code or fiscal_year first in WHERE
2. **Use exact matches when possible**: = instead of LIKE
3. **Avoid function calls on indexed columns**: Don't UPPER(account_code)
4. **BETWEEN for date ranges**: Leverages index efficiently

### Common Anti-Patterns to Avoid

❌ **Anti-Pattern 1: Function on Indexed Column**
```sql
-- BAD (can't use index)
WHERE EXTRACT(YEAR FROM posting_date) = 2025

-- GOOD (uses index)
WHERE posting_date >= '2025-01-01' AND posting_date < '2026-01-01'
```

❌ **Anti-Pattern 2: OR Instead of UNION**
```sql
-- BAD (table scan)
WHERE fiscal_year = 2024 OR fiscal_year = 2025

-- GOOD (partition pruning)
SELECT * FROM gl_posting_archive_y2024 WHERE fiscal_year = 2024
UNION ALL
SELECT * FROM gl_posting_archive_y2025 WHERE fiscal_year = 2025
```

❌ **Anti-Pattern 3: Implicit Cross-Partition**
```sql
-- BAD (PostgreSQL scans all partitions)
SELECT * FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
WHERE account_code = '1000-100'  -- No year filter

-- GOOD (explicit partition)
SELECT * FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
WHERE account_code = '1000-100' AND fiscal_year = 2025
```

---

## When to Delegate to postgres-expert

Consult `postgres-expert` specialist for:
- GL queries exceeding 2s (significantly above baseline)
- Need for custom indexes on GL partitions
- Partition maintenance strategies (adding new years)
- Cross-schema join optimization (processed vs JDE source)
- Query plan analysis with EXPLAIN ANALYZE

---

## Related Documentation

- **Schema Documentation**: `schema/jwsdatagrc_ironhide.md` (GL partition details)
- **Business Metrics**: `semantic-layer/metrics.md` (gl_posting_volume, account_activity)
- **Business Dimensions**: `semantic-layer/dimensions.md` (posting_date, fiscal_year, account_code)
- **JDE Integration**: `schema/jde_raw.md` (source system F0901 table)

---

**Last Updated**: 2025-10-21
**Confidence**: 0.90 (production-validated patterns)
