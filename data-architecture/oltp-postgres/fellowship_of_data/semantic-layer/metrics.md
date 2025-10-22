# Business Metrics - fellowship_of_data

**Purpose**: Consistent business metric definitions for AI-driven analytics

**Database**: fellowship_of_data (postgres.grc-ops.com)

**Pattern**: Documentation-based semantic layer (Phase 1) - will be formalized in dbt (Phase 2, Issue #170)

---

## Ticket Sales Metrics

### total_tickets_sold
**Description**: Count of all ticket sales transactions

**Business Question**: "How many tickets did we sell?"

**Source Table**: `jwsdatagrc_ironhide.tickets_archive`

**Calculation**:
```sql
SELECT COUNT(*) as total_tickets_sold
FROM jwsdatagrc_ironhide.tickets_archive
WHERE sale_date BETWEEN :start_date AND :end_date
```

**Common Dimensions**:
- `sale_date` - When ticket was sold (supports daily, weekly, monthly, yearly)
- `customer_id` - Which customer bought the ticket
- `ticket_type` - Type/category of ticket

**Performance**: <100ms (indexed on sale_date + ticket_sale_line_item_primary_key)

---

### revenue_by_customer
**Description**: Total revenue aggregated by customer

**Business Question**: "Which customers generate the most revenue?"

**Source Tables**:
- `jwsdatagrc_ironhide.tickets_archive` (ticket sales)
- `jwsdatagrc_raw.slcust` (customer master)

**Calculation**:
```sql
SELECT
    c.customer_id,
    c.customer_name,
    SUM(t.amount) as total_revenue
FROM jwsdatagrc_ironhide.tickets_archive t
JOIN jwsdatagrc_raw.slcust c ON t.customer_id = c.customer_id
WHERE t.sale_date BETWEEN :start_date AND :end_date
GROUP BY c.customer_id, c.customer_name
ORDER BY total_revenue DESC
```

**Common Dimensions**:
- `customer_id` - Customer identifier
- `customer_name` - Customer display name
- `sale_date` - Time period for revenue
- `region` - Geographic region (if available)

**Performance**: <500ms (join indexed tables)

---

### ticket_revenue_by_type
**Description**: Revenue segmented by ticket type/category

**Business Question**: "Which ticket types generate the most revenue?"

**Source Table**: `jwsdatagrc_ironhide.tickets_archive`

**Calculation**:
```sql
SELECT
    ticket_type,
    COUNT(*) as ticket_count,
    SUM(amount) as total_revenue,
    AVG(amount) as avg_revenue_per_ticket
FROM jwsdatagrc_ironhide.tickets_archive
WHERE sale_date BETWEEN :start_date AND :end_date
GROUP BY ticket_type
ORDER BY total_revenue DESC
```

**Common Dimensions**:
- `ticket_type` - Ticket category
- `sale_date` - Time period
- `customer_segment` - Customer grouping (if available)

**Performance**: <200ms (indexed on source_table + sale_date)

---

## General Ledger Metrics

### gl_posting_volume
**Description**: Count of GL posting transactions

**Business Question**: "How many GL postings occurred?"

**Source Tables**:
- `jwsdatagrc_ironhide.gl_posting_archive_y2024` (yearly partition)
- `jwsdatagrc_ironhide.gl_posting_archive_y2025` (yearly partition)

**Calculation**:
```sql
-- For 2024 data
SELECT COUNT(*) as posting_volume
FROM jwsdatagrc_ironhide.gl_posting_archive_y2024
WHERE posting_date BETWEEN :start_date AND :end_date

-- For cross-year queries, use UNION ALL across partitions
```

**Common Dimensions**:
- `posting_date` - When GL entry was posted
- `account_code` - GL account number
- `department` - Department/cost center (if available)

**Performance**: <1s with partition pruning (10x faster than unpartitioned)

---

### gl_balance_by_account
**Description**: GL account balances aggregated by account code

**Business Question**: "What's the balance for each GL account?"

**Source Tables**:
- `jwsdatagrc_ironhide.gl_posting_archive_min` (minimum archive)
- `jde_raw.f0901` (JDE General Ledger)

**Calculation**:
```sql
SELECT
    account_code,
    SUM(debit_amount) as total_debits,
    SUM(credit_amount) as total_credits,
    SUM(debit_amount) - SUM(credit_amount) as net_balance
FROM jwsdatagrc_ironhide.gl_posting_archive_min
WHERE posting_date BETWEEN :start_date AND :end_date
GROUP BY account_code
ORDER BY net_balance DESC
```

**Common Dimensions**:
- `account_code` - GL account identifier
- `posting_date` - Time period
- `fiscal_year` - Fiscal year
- `fiscal_period` - Fiscal period/month

**Performance**: <2s (3.2 GB table, consider account_code index)

---

## JDE ERP Metrics

### jde_gl_account_activity
**Description**: JD Edwards GL account activity (debits, credits, balance)

**Business Question**: "What's the activity for JDE GL accounts?"

**Source Table**: `jde_raw.f0901` (721 MB)

**Calculation**:
```sql
SELECT
    account_code,
    COUNT(*) as transaction_count,
    SUM(CASE WHEN transaction_type = 'D' THEN amount ELSE 0 END) as total_debits,
    SUM(CASE WHEN transaction_type = 'C' THEN amount ELSE 0 END) as total_credits
FROM jde_raw.f0901
WHERE posting_date BETWEEN :start_date AND :end_date
GROUP BY account_code
ORDER BY transaction_count DESC
```

**Common Dimensions**:
- `account_code` - JDE account number
- `posting_date` - Transaction date
- `company_code` - JDE company identifier
- `business_unit` - JDE business unit

**Performance**: <500ms (needs account_code B-tree index)

---

## Customer Metrics

### customer_count_active
**Description**: Count of active customers (with transactions in period)

**Business Question**: "How many active customers do we have?"

**Source Tables**:
- `jwsdatagrc_raw.slcust` (18 MB) - Customer master
- `jwsdatagrc_ironhide.tickets_archive` (12 GB) - Ticket sales

**Calculation**:
```sql
SELECT COUNT(DISTINCT c.customer_id) as active_customer_count
FROM jwsdatagrc_raw.slcust c
JOIN jwsdatagrc_ironhide.tickets_archive t
    ON c.customer_id = t.customer_id
WHERE t.sale_date BETWEEN :start_date AND :end_date
```

**Common Dimensions**:
- `sale_date` - Activity period
- `customer_type` - Customer classification (if available)
- `region` - Geographic region

**Performance**: <1s (join on indexed customer_id)

---

## Time Series Metrics

### daily_ticket_sales
**Description**: Ticket sales aggregated by day

**Business Question**: "What's our daily sales trend?"

**Source Table**: `jwsdatagrc_ironhide.tickets_archive`

**Calculation**:
```sql
SELECT
    sale_date,
    COUNT(*) as tickets_sold,
    SUM(amount) as daily_revenue,
    AVG(amount) as avg_ticket_price
FROM jwsdatagrc_ironhide.tickets_archive
WHERE sale_date BETWEEN :start_date AND :end_date
GROUP BY sale_date
ORDER BY sale_date
```

**Common Dimensions**:
- `sale_date` - Day granularity
- `week_of_year` - Weekly aggregation
- `month` - Monthly aggregation
- `year` - Yearly aggregation

**Performance**: <500ms (indexed on sale_date)

**Optimization**: Consider BRIN index if queries are always time-ordered

---

### monthly_gl_posting_trend
**Description**: GL posting activity aggregated by month

**Business Question**: "What's our monthly GL activity trend?"

**Source Tables**:
- `jwsdatagrc_ironhide.gl_posting_archive_y2024`
- `jwsdatagrc_ironhide.gl_posting_archive_y2025`

**Calculation**:
```sql
SELECT
    DATE_TRUNC('month', posting_date) as month,
    COUNT(*) as posting_count,
    SUM(amount) as total_amount
FROM jwsdatagrc_ironhide.gl_posting_archive_y2024
WHERE posting_date BETWEEN :start_date AND :end_date
GROUP BY DATE_TRUNC('month', posting_date)
ORDER BY month

-- For cross-year, UNION ALL with gl_posting_archive_y2025
```

**Common Dimensions**:
- `month` - Monthly granularity
- `fiscal_year` - Fiscal year
- `account_category` - GL account grouping

**Performance**: <1s with partition pruning (yearly partitions)

**Optimization**: Consider monthly partitions if most queries filter by month

---

## Usage Guidelines

### For AI Agents
1. **Start here first**: Check metrics before writing custom SQL
2. **Business questions**: Match user question to pre-defined metric
3. **Combine metrics**: Build complex analysis from metric building blocks
4. **Dimension slicing**: Use common dimensions to filter/group metrics

### For Custom Metrics
If needed metric doesn't exist:
1. **Define clearly**: What business question does it answer?
2. **Specify tables**: Which source tables contain the data?
3. **Write SQL**: Create calculation with named parameters
4. **Document dimensions**: What dimensions make sense for slicing?
5. **Test performance**: Check query execution time
6. **Consider adding**: If reusable, add to this file

### Performance Expectations
- **Simple metrics** (single table, indexed): <100ms
- **Join metrics** (2-3 tables, indexed): <500ms
- **Complex metrics** (multiple tables, aggregations): <2s
- **If slower**: Consult postgres-expert for optimization

---

## Future Enhancement (Phase 2)

These metrics will be formalized as dbt semantic layer metrics (Issue #170):
- Defined in `dbt_cloud/models/fellowship_of_data/metrics/`
- Queryable via `mcp__dbt-mcp__query_metrics`
- Type-safe, validated, with lineage tracking

**Timeline**: 5-7 days for dbt semantic layer implementation

---

**Last Updated**: 2025-10-21
**Status**: Documentation-based semantic layer (Phase 1)
**Next Step**: Formalize in dbt (Phase 2, Issue #170)
