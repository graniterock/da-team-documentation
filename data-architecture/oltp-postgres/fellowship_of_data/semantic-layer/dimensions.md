# Business Dimensions - fellowship_of_data

**Purpose**: Define how to slice and dice metrics across business dimensions

**Database**: fellowship_of_data (postgres.grc-ops.com)

**Pattern**: Documentation-based semantic layer (Phase 1) - will be formalized in dbt (Phase 2, Issue #170)

---

## Time Dimensions

### sale_date
**Description**: When a ticket was sold

**Type**: Time dimension (supports multiple granularities)

**Source**: `jwsdatagrc_ironhide.tickets_archive.sale_date`

**Granularities**:
- **Daily**: `sale_date`
- **Weekly**: `DATE_TRUNC('week', sale_date)`
- **Monthly**: `DATE_TRUNC('month', sale_date)`
- **Quarterly**: `DATE_TRUNC('quarter', sale_date)`
- **Yearly**: `DATE_TRUNC('year', sale_date)` or EXTRACT(YEAR FROM sale_date)

**Common Filters**:
```sql
-- Last 30 days
WHERE sale_date >= CURRENT_DATE - INTERVAL '30 days'

-- Specific month
WHERE sale_date BETWEEN '2024-01-01' AND '2024-01-31'

-- Year-to-date
WHERE sale_date >= DATE_TRUNC('year', CURRENT_DATE)

-- Last quarter
WHERE sale_date >= DATE_TRUNC('quarter', CURRENT_DATE) - INTERVAL '3 months'
  AND sale_date < DATE_TRUNC('quarter', CURRENT_DATE)
```

**Performance**: Indexed (composite: source_table + sale_date) - queries <100ms

**Related Metrics**: total_tickets_sold, revenue_by_customer, ticket_revenue_by_type, daily_ticket_sales

---

### posting_date
**Description**: When a GL posting occurred

**Type**: Time dimension (supports multiple granularities)

**Source**:
- `jwsdatagrc_ironhide.gl_posting_archive_y2024.posting_date`
- `jwsdatagrc_ironhide.gl_posting_archive_y2025.posting_date`
- `jde_raw.f0901.posting_date`

**Granularities**:
- **Daily**: `posting_date`
- **Weekly**: `DATE_TRUNC('week', posting_date)`
- **Monthly**: `DATE_TRUNC('month', posting_date)`
- **Fiscal Period**: Map to fiscal calendar (if available)
- **Yearly**: `DATE_TRUNC('year', posting_date)`

**Common Filters**:
```sql
-- Current fiscal year (assuming calendar year)
WHERE posting_date >= '2024-01-01'
  AND posting_date < '2025-01-01'

-- Specific fiscal period
WHERE posting_date BETWEEN '2024-03-01' AND '2024-03-31'

-- Year-over-year comparison
WHERE posting_date BETWEEN :current_year_start AND :current_year_end
   OR posting_date BETWEEN :prior_year_start AND :prior_year_end
```

**Performance**:
- ✅ **Partitioned by year** - Partition pruning automatically applied (10x faster)
- ✅ Consider monthly partitions if most queries filter by month

**Related Metrics**: gl_posting_volume, gl_balance_by_account, monthly_gl_posting_trend, jde_gl_account_activity

---

## Customer Dimensions

### customer_id
**Description**: Unique customer identifier

**Type**: Categorical dimension (primary key)

**Source**: `jwsdatagrc_raw.slcust.customer_id`

**Joins**:
```sql
-- Join tickets to customer master
FROM jwsdatagrc_ironhide.tickets_archive t
JOIN jwsdatagrc_raw.slcust c
    ON t.customer_id = c.customer_id
```

**Attributes**:
- `customer_name` - Display name
- `customer_type` - Classification/category (if available)
- `region` - Geographic region (if available)
- `account_status` - Active, inactive, etc. (if available)

**Performance**: Indexed join (customer_id) - <500ms for typical queries

**Related Metrics**: revenue_by_customer, customer_count_active

---

### customer_name
**Description**: Human-readable customer name

**Type**: Categorical dimension (display label)

**Source**: `jwsdatagrc_raw.slcust.customer_name`

**Usage**: Display dimension for reports, not for grouping large datasets

**Example**:
```sql
SELECT
    c.customer_name,
    COUNT(*) as ticket_count
FROM jwsdatagrc_ironhide.tickets_archive t
JOIN jwsdatagrc_raw.slcust c ON t.customer_id = c.customer_id
GROUP BY c.customer_name
ORDER BY ticket_count DESC
LIMIT 100
```

**Performance**: Use with LIMIT for large result sets

**Related Metrics**: revenue_by_customer

---

## Product/Ticket Dimensions

### ticket_type
**Description**: Type or category of ticket

**Type**: Categorical dimension

**Source**: `jwsdatagrc_ironhide.tickets_archive.ticket_type`

**Cardinality**: Low-to-medium (expected <100 unique values)

**Usage**:
```sql
SELECT
    ticket_type,
    COUNT(*) as tickets_sold,
    SUM(amount) as revenue
FROM jwsdatagrc_ironhide.tickets_archive
WHERE sale_date BETWEEN :start_date AND :end_date
GROUP BY ticket_type
ORDER BY revenue DESC
```

**Performance**: Fast grouping (low cardinality categorical)

**Related Metrics**: ticket_revenue_by_type, total_tickets_sold

---

### source_table
**Description**: Source system/table identifier for ticket data

**Type**: Categorical dimension (technical)

**Source**: `jwsdatagrc_ironhide.tickets_archive.source_table`

**Usage**: Filter by specific source system or track data lineage

**Example**:
```sql
-- Tickets from specific source
WHERE source_table = 'tkhist1'

-- Analyze by source
SELECT
    source_table,
    COUNT(*) as record_count
FROM jwsdatagrc_ironhide.tickets_archive
GROUP BY source_table
```

**Performance**: Indexed (composite: source_table + sale_date)

**Related Metrics**: total_tickets_sold, ticket_revenue_by_type

---

## Financial Dimensions

### account_code
**Description**: General Ledger account number

**Type**: Categorical dimension (hierarchical)

**Source**:
- `jwsdatagrc_ironhide.gl_posting_archive_min.account_code`
- `jde_raw.f0901.account_code`

**Cardinality**: Medium-to-high (100s to 1000s of accounts)

**Hierarchy** (if chart of accounts available):
- **Account Level 1**: Major category (Assets, Liabilities, Revenue, Expenses)
- **Account Level 2**: Subcategory
- **Account Code**: Specific account

**Usage**:
```sql
-- Activity by account
SELECT
    account_code,
    COUNT(*) as transaction_count,
    SUM(amount) as total_amount
FROM jde_raw.f0901
WHERE posting_date BETWEEN :start_date AND :end_date
GROUP BY account_code
ORDER BY total_amount DESC
```

**Performance**:
- 💡 **Needs B-tree index on account_code** (721 MB table, frequent filtering)
- Expected <50ms with index

**Related Metrics**: gl_balance_by_account, jde_gl_account_activity

---

### fiscal_year
**Description**: Fiscal year identifier

**Type**: Time dimension (categorical representation)

**Source**: Derived from `posting_date`

**Calculation**:
```sql
-- Assuming calendar year = fiscal year
EXTRACT(YEAR FROM posting_date) as fiscal_year

-- For custom fiscal year (e.g., starts July 1)
CASE
    WHEN EXTRACT(MONTH FROM posting_date) >= 7
    THEN EXTRACT(YEAR FROM posting_date)
    ELSE EXTRACT(YEAR FROM posting_date) - 1
END as fiscal_year
```

**Usage**: Year-over-year analysis, fiscal reporting

**Performance**: Calculated dimension (no storage overhead)

**Related Metrics**: gl_posting_volume, gl_balance_by_account

---

### fiscal_period
**Description**: Fiscal period/month within fiscal year

**Type**: Time dimension (categorical)

**Source**: Derived from `posting_date`

**Calculation**:
```sql
-- Calendar month (1-12)
EXTRACT(MONTH FROM posting_date) as fiscal_period

-- Fiscal period (for fiscal year starting July 1)
CASE
    WHEN EXTRACT(MONTH FROM posting_date) >= 7
    THEN EXTRACT(MONTH FROM posting_date) - 6
    ELSE EXTRACT(MONTH FROM posting_date) + 6
END as fiscal_period
```

**Usage**: Period-to-period comparisons, month-end reporting

**Performance**: Calculated dimension

**Related Metrics**: gl_posting_volume, monthly_gl_posting_trend

---

## JDE ERP Dimensions

### company_code
**Description**: JD Edwards company identifier

**Type**: Categorical dimension

**Source**: `jde_raw.f0901.company_code` (if available)

**Usage**: Multi-company reporting, consolidations

**Example**:
```sql
SELECT
    company_code,
    SUM(amount) as total_activity
FROM jde_raw.f0901
WHERE posting_date BETWEEN :start_date AND :end_date
GROUP BY company_code
```

**Performance**: Low cardinality (few companies) - fast grouping

**Related Metrics**: jde_gl_account_activity

---

### business_unit
**Description**: JD Edwards business unit identifier

**Type**: Categorical dimension

**Source**: `jde_raw.f0901.business_unit` (if available)

**Usage**: Business unit reporting, cost center analysis

**Performance**: Medium cardinality - consider indexing if frequently filtered

**Related Metrics**: jde_gl_account_activity

---

## Technical Dimensions

### ticket_sale_line_item_primary_key
**Description**: Unique identifier for ticket sale line item

**Type**: Identifier (primary key, not for grouping)

**Source**: `jwsdatagrc_ironhide.tickets_archive.ticket_sale_line_item_primary_key`

**Usage**: Join key, unique record lookup

**Performance**: Primary key index - instant lookup (<1ms)

**Related Metrics**: All ticket metrics (used for counting distinct)

---

### ticket_combined_uniqueid
**Description**: Combined unique identifier for ticket

**Type**: Identifier (composite key)

**Source**: `jwsdatagrc_ironhide.tickets_archive.ticket_combined_uniqueid`

**Usage**: Deduplication, cross-system reconciliation

**Performance**: Indexed (composite with ticket_sale_line_item_primary_key)

**Related Metrics**: All ticket metrics

---

## Dimension Usage Patterns

### Pattern 1: Time-Based Slicing
**Common for**: Trend analysis, period comparisons, forecasting

```sql
SELECT
    DATE_TRUNC('month', sale_date) as month,
    COUNT(*) as tickets_sold
FROM jwsdatagrc_ironhide.tickets_archive
WHERE sale_date BETWEEN :start_date AND :end_date
GROUP BY DATE_TRUNC('month', sale_date)
ORDER BY month
```

**Dimensions**: sale_date, posting_date, fiscal_year, fiscal_period

---

### Pattern 2: Categorical Segmentation
**Common for**: Product analysis, customer segmentation, account classification

```sql
SELECT
    ticket_type,
    COUNT(*) as count,
    AVG(amount) as avg_amount
FROM jwsdatagrc_ironhide.tickets_archive
WHERE sale_date BETWEEN :start_date AND :end_date
GROUP BY ticket_type
ORDER BY count DESC
```

**Dimensions**: ticket_type, customer_id, account_code

---

### Pattern 3: Hierarchical Drill-Down
**Common for**: Top-down analysis, root cause investigation

```sql
-- Level 1: By year
SELECT EXTRACT(YEAR FROM sale_date) as year, SUM(amount)
FROM jwsdatagrc_ironhide.tickets_archive
GROUP BY year

-- Level 2: By month within year
SELECT DATE_TRUNC('month', sale_date) as month, SUM(amount)
FROM jwsdatagrc_ironhide.tickets_archive
WHERE EXTRACT(YEAR FROM sale_date) = 2024
GROUP BY month

-- Level 3: By customer within month
SELECT customer_id, SUM(amount)
FROM jwsdatagrc_ironhide.tickets_archive
WHERE DATE_TRUNC('month', sale_date) = '2024-03-01'
GROUP BY customer_id
```

**Dimensions**: Time (year → month → day), Customer (all → segment → individual)

---

### Pattern 4: Cross-Dimensional Analysis
**Common for**: Multi-factor analysis, correlation discovery

```sql
SELECT
    DATE_TRUNC('month', sale_date) as month,
    ticket_type,
    COUNT(*) as tickets_sold,
    SUM(amount) as revenue
FROM jwsdatagrc_ironhide.tickets_archive
WHERE sale_date BETWEEN :start_date AND :end_date
GROUP BY DATE_TRUNC('month', sale_date), ticket_type
ORDER BY month, revenue DESC
```

**Dimensions**: Combine time + categorical (sale_date + ticket_type)

---

## Dimension Cardinality Guide

**Low Cardinality** (<100 unique values):
- ticket_type, source_table, company_code, fiscal_year
- **Impact**: Fast grouping, suitable for dashboard filters

**Medium Cardinality** (100-10,000 unique values):
- account_code, business_unit, fiscal_period
- **Impact**: Moderate grouping cost, index recommended for filtering

**High Cardinality** (10,000+ unique values):
- customer_id, customer_name
- **Impact**: Use LIMIT, consider aggregation first, index required

**Unique Identifiers** (one per row):
- ticket_sale_line_item_primary_key, ticket_combined_uniqueid
- **Impact**: Not for grouping, use for joins/lookups only

---

## Future Enhancement (Phase 2)

These dimensions will be formalized in dbt semantic layer (Issue #170):
- Defined in `dbt_cloud/models/fellowship_of_data/semantic/`
- Type-safe dimension definitions (time, categorical, identifier)
- Hierarchies explicitly modeled
- Queryable via `mcp__dbt-mcp__query_metrics` with dimension filters

**Timeline**: 5-7 days for dbt semantic layer implementation

---

**Last Updated**: 2025-10-21
**Status**: Documentation-based semantic layer (Phase 1)
**Next Step**: Formalize in dbt (Phase 2, Issue #170)
