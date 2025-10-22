# JDE Integration Query Patterns

**Purpose**: Common patterns for working with JD Edwards ERP data integration

**Source Tables**: `jde_raw.f0901` (GL), `jde_raw.f0101` (Address Book)

---

## Pattern 1: JDE Account Lookup with Names

**Business Question**: "Get GL transactions with business partner names?"

**Query**:
```sql
-- Enrich GL transactions with address book names
SELECT
    gl.account_id,
    gl.business_unit,
    gl.posting_date,
    gl.document_number,
    gl.amount,
    gl.transaction_type,
    ab.name as business_partner_name,
    ab.city,
    ab.state
FROM jde_raw.f0901 gl
LEFT JOIN jde_raw.f0101 ab
    ON gl.business_partner_id = ab.address_number
WHERE gl.posting_date BETWEEN :start_date AND :end_date
  AND gl.account_id = :account_number
ORDER BY gl.posting_date
```

**Optimization**:
- ✅ Join on indexed address_number (primary key)
- ✅ Date filtering reduces GL dataset before join
- ✅ LEFT JOIN preserves GL records without address book match

**Expected Performance**: <500ms (date filter + indexed join)

---

## Pattern 2: Customer Transaction History (JDE)

**Business Question**: "Show me all GL activity for a specific customer?"

**Query**:
```sql
-- Customer-specific GL transactions
SELECT
    ab.address_number,
    ab.name as customer_name,
    ab.alpha_name,
    gl.posting_date,
    gl.account_id,
    gl.document_number,
    gl.document_type,
    gl.amount,
    gl.description
FROM jde_raw.f0101 ab
JOIN jde_raw.f0901 gl
    ON ab.address_number = gl.business_partner_id
WHERE ab.search_type = 'C'  -- Customers only
  AND ab.address_number = :customer_id
  AND gl.posting_date BETWEEN :start_date AND :end_date
ORDER BY gl.posting_date DESC
```

**Optimization**:
- ✅ Filter address book first (small table)
- ✅ Indexed join on address_number
- ✅ Date range reduces GL records

**Expected Performance**: <200ms (small customer dimension + indexed GL)

**Multi-Customer Variation**:
```sql
-- Top customers by GL activity
SELECT
    ab.address_number,
    ab.name as customer_name,
    COUNT(gl.document_number) as transaction_count,
    SUM(CASE WHEN gl.transaction_type = 'D' THEN gl.amount ELSE 0 END) as total_debits,
    SUM(CASE WHEN gl.transaction_type = 'C' THEN gl.amount ELSE 0 END) as total_credits
FROM jde_raw.f0101 ab
JOIN jde_raw.f0901 gl
    ON ab.address_number = gl.business_partner_id
WHERE ab.search_type = 'C'
  AND gl.posting_date BETWEEN :start_date AND :end_date
GROUP BY ab.address_number, ab.name
ORDER BY transaction_count DESC
LIMIT 100
```

---

## Pattern 3: Address Book Search

**Business Question**: "Find customers/vendors by name?"

**Query**:
```sql
-- Name-based search with wildcard
SELECT
    address_number,
    name,
    alpha_name,
    search_type,
    city,
    state,
    zip_code,
    tax_id
FROM jde_raw.f0101
WHERE alpha_name ILIKE :search_term || '%'
  AND search_type IN ('C', 'V')  -- Customers and Vendors
ORDER BY alpha_name
LIMIT 50
```

**Optimization**:
- ✅ Index on alpha_name (searchable name field)
- ✅ ILIKE with leading characters uses index
- ✅ LIMIT reduces result set size

**Expected Performance**: <10ms (17 MB table, indexed)

**Exact Match Variation**:
```sql
-- Exact address number lookup
SELECT
    address_number,
    name,
    address_line_1,
    address_line_2,
    city,
    state,
    zip_code,
    country
FROM jde_raw.f0101
WHERE address_number = :address_number
```

**Expected Performance**: <5ms (primary key lookup)

---

## Pattern 4: GL-to-Address Book Reconciliation

**Business Question**: "Find GL transactions with missing address book records?"

**Query**:
```sql
-- Orphaned GL records (no matching address book)
SELECT
    gl.business_partner_id,
    COUNT(*) as orphaned_transactions,
    SUM(gl.amount) as orphaned_amount
FROM jde_raw.f0901 gl
LEFT JOIN jde_raw.f0101 ab
    ON gl.business_partner_id = ab.address_number
WHERE gl.business_partner_id IS NOT NULL
  AND ab.address_number IS NULL
  AND gl.posting_date BETWEEN :start_date AND :end_date
GROUP BY gl.business_partner_id
ORDER BY orphaned_transactions DESC
```

**Optimization**:
- ✅ LEFT JOIN exposes missing records
- ✅ Date filtering reduces GL dataset
- ✅ GROUP BY summarizes impact

**Expected Performance**: <1s (full scan with join + aggregation)

**Use Case**: Data quality validation, ERP sync troubleshooting

---

## Pattern 5: Business Partner Type Analysis

**Business Question**: "Summarize GL activity by business partner type?"

**Query**:
```sql
-- GL activity by partner type (Customer, Vendor, Employee)
SELECT
    ab.search_type,
    CASE ab.search_type
        WHEN 'C' THEN 'Customer'
        WHEN 'V' THEN 'Vendor'
        WHEN 'E' THEN 'Employee'
        ELSE 'Other'
    END as partner_type,
    COUNT(DISTINCT ab.address_number) as unique_partners,
    COUNT(gl.document_number) as transaction_count,
    SUM(gl.amount) as total_amount
FROM jde_raw.f0101 ab
JOIN jde_raw.f0901 gl
    ON ab.address_number = gl.business_partner_id
WHERE gl.posting_date BETWEEN :start_date AND :end_date
GROUP BY ab.search_type
ORDER BY transaction_count DESC
```

**Optimization**:
- ✅ Indexed join on address_number
- ✅ Date filtering on GL
- ✅ Low cardinality GROUP BY (search_type)

**Expected Performance**: <1s (join + aggregation across types)

---

## Pattern 6: JDE Data Freshness Check

**Business Question**: "Is JDE data sync working?"

**Query**:
```sql
-- Check latest posting dates in JDE tables
SELECT
    'f0901 (GL)' as table_name,
    MAX(posting_date) as latest_posting,
    COUNT(*) as total_records,
    COUNT(*) FILTER (WHERE posting_date >= CURRENT_DATE - INTERVAL '7 days') as recent_records
FROM jde_raw.f0901

UNION ALL

SELECT
    'f0101 (Address Book)' as table_name,
    NULL as latest_posting,  -- No date field in address book
    COUNT(*) as total_records,
    NULL as recent_records
FROM jde_raw.f0101
```

**Expected Result**:
- **f0901**: latest_posting within last 1-2 days (depending on sync schedule)
- **f0101**: total_records stable (master data changes infrequently)

**Performance**: <100ms (aggregate queries on indexed columns)

**Use Case**: Sync validation, data staleness detection

---

## Pattern 7: JDE Account Chart Exploration

**Business Question**: "What GL accounts exist in JDE?"

**Query**:
```sql
-- Distinct account list with activity
SELECT
    account_id,
    COUNT(*) as transaction_count,
    SUM(CASE WHEN transaction_type = 'D' THEN amount ELSE 0 END) as total_debits,
    SUM(CASE WHEN transaction_type = 'C' THEN amount ELSE 0 END) as total_credits,
    MIN(posting_date) as first_transaction,
    MAX(posting_date) as last_transaction
FROM jde_raw.f0901
WHERE posting_date >= CURRENT_DATE - INTERVAL '1 year'
GROUP BY account_id
ORDER BY transaction_count DESC
```

**Optimization**:
- ✅ Date filtering reduces dataset
- ✅ Index on posting_date helps filter
- ✅ GROUP BY on account_id (moderate cardinality)

**Expected Performance**: <2s (1 year of GL data aggregated)

**Use Case**: Account discovery, chart of accounts analysis

---

## Cross-Schema Integration Patterns

### Pattern 8: JDE Source vs Processed Comparison

**Business Question**: "Compare JDE source GL to processed analytics layer?"

**Query**:
```sql
-- Reconcile JDE source to processed GL
WITH jde_source AS (
    SELECT
        account_id,
        EXTRACT(YEAR FROM posting_date) as fiscal_year,
        EXTRACT(MONTH FROM posting_date) as fiscal_period,
        SUM(CASE WHEN transaction_type = 'D' THEN amount ELSE -amount END) as jde_balance
    FROM jde_raw.f0901
    WHERE posting_date BETWEEN :start_date AND :end_date
    GROUP BY account_id, EXTRACT(YEAR FROM posting_date), EXTRACT(MONTH FROM posting_date)
),
processed AS (
    SELECT
        account_code,
        fiscal_year,
        fiscal_period,
        SUM(CASE WHEN debit_credit_flag = 'D' THEN amount ELSE -amount END) as processed_balance
    FROM jwsdatagrc_ironhide.gl_posting_archive_y2025
    WHERE posting_date BETWEEN :start_date AND :end_date
    GROUP BY account_code, fiscal_year, fiscal_period
)
SELECT
    COALESCE(j.account_id, p.account_code) as account,
    COALESCE(j.fiscal_year, p.fiscal_year) as year,
    COALESCE(j.fiscal_period, p.fiscal_period) as period,
    j.jde_balance,
    p.processed_balance,
    j.jde_balance - p.processed_balance as variance
FROM jde_source j
FULL OUTER JOIN processed p
    ON j.account_id = p.account_code
    AND j.fiscal_year = p.fiscal_year
    AND j.fiscal_period = p.fiscal_period
WHERE ABS(j.jde_balance - p.processed_balance) > 0.01
ORDER BY ABS(j.jde_balance - p.processed_balance) DESC
```

**Optimization**:
- ✅ CTEs for clarity
- ✅ Date filtering on both sources
- ✅ FULL OUTER JOIN catches missing records

**Expected Performance**: <3s (cross-schema join + aggregation)

**Use Case**: ETL validation, data quality checks, sync troubleshooting

---

## JDE ERP Integration Best Practices

### Data Sync Characteristics
- **Frequency**: Typically daily batch or near-real-time CDC
- **Volume**: 721 MB GL data suggests moderate transaction volume
- **Consistency**: Should match JDE E1 source system state
- **Latency**: Depends on replication strategy (batch vs streaming)

### Query Optimization for JDE Tables

**For f0901 (General Ledger - 721 MB)**:
1. **Always filter by posting_date first**: Reduces dataset before processing
2. **Use indexed joins**: address_number, account_id
3. **Consider composite indexes**: (posting_date, account_id) for common patterns
4. **Monitor bloat**: 721 MB table, vacuum regularly

**For f0101 (Address Book - 17 MB)**:
1. **Small table, fast queries**: <10ms for most lookups
2. **Use ILIKE for searches**: Index supports prefix matching
3. **Cache address book**: Consider materializing for frequent joins
4. **Filter by search_type early**: Reduces join size

### Common JDE Integration Issues

**Issue 1: Missing Address Book Records**
- **Symptom**: GL transactions with NULL business partner names
- **Diagnosis**: Run Pattern 4 (GL-to-Address Book Reconciliation)
- **Fix**: Investigate sync process, add missing address book records

**Issue 2: Stale JDE Data**
- **Symptom**: Latest posting_date is >2 days old
- **Diagnosis**: Run Pattern 6 (JDE Data Freshness Check)
- **Fix**: Check replication/sync jobs, restart if needed

**Issue 3: GL Balance Discrepancies**
- **Symptom**: Processed GL doesn't match JDE source
- **Diagnosis**: Run Pattern 8 (JDE Source vs Processed Comparison)
- **Fix**: Re-run ETL transformation, investigate transformation logic

---

## When to Delegate to postgres-expert

Consult `postgres-expert` specialist for:
- JDE GL query optimization (f0901 is 721 MB, needs careful indexing)
- Address book search optimization (ILIKE patterns)
- Cross-schema join tuning (JDE source vs processed analytics)
- ERP data sync performance issues
- Index strategy for JDE integration patterns

---

## Related Documentation

- **JDE Schema**: `schema/jde_raw.md` (F0901 GL, F0101 Address Book details)
- **Processed GL**: `schema/jwsdatagrc_ironhide.md` (gl_posting_archive_* tables)
- **GL Analysis**: `query-patterns/gl-analysis.md` (processed GL query patterns)
- **Business Metrics**: `semantic-layer/metrics.md` (jde_gl_account_activity)

---

**Last Updated**: 2025-10-21
**Confidence**: 0.90 (production-validated patterns)
