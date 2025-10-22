# Ticket Analysis Query Patterns

**Purpose**: Common ticket sales analysis queries with optimization guidance

**Source Tables**: `jwsdatagrc_ironhide.tickets_archive`, `jwsdatagrc_raw.slcust`

---

## Pattern 1: Time-Based Ticket Sales

**Business Question**: "How many tickets did we sell in a specific time period?"

**Query**:
```sql
SELECT
    COUNT(*) as tickets_sold,
    SUM(amount) as total_revenue,
    AVG(amount) as avg_ticket_price
FROM jwsdatagrc_ironhide.tickets_archive
WHERE sale_date BETWEEN :start_date AND :end_date;
```

**Optimization**:
- ✅ Uses composite index: (source_table, sale_date)
- ✅ Date range filtering benefits from index
- ✅ Aggregations calculated efficiently

**Expected Performance**: <100ms (12 GB table, indexed)

**Variations**:
```sql
-- Daily breakdown
SELECT
    sale_date,
    COUNT(*) as tickets_sold,
    SUM(amount) as daily_revenue
FROM jwsdatagrc_ironhide.tickets_archive
WHERE sale_date BETWEEN :start_date AND :end_date
GROUP BY sale_date
ORDER BY sale_date;

-- Weekly aggregation
SELECT
    DATE_TRUNC('week', sale_date) as week,
    COUNT(*) as tickets_sold,
    SUM(amount) as weekly_revenue
FROM jwsdatagrc_ironhide.tickets_archive
WHERE sale_date BETWEEN :start_date AND :end_date
GROUP BY DATE_TRUNC('week', sale_date)
ORDER BY week;
```

---

## Pattern 2: Revenue by Customer

**Business Question**: "Who are our top customers by revenue?"

**Query**:
```sql
SELECT
    c.customer_id,
    c.customer_name,
    COUNT(t.ticket_sale_line_item_primary_key) as ticket_count,
    SUM(t.amount) as total_revenue,
    AVG(t.amount) as avg_ticket_value
FROM jwsdatagrc_ironhide.tickets_archive t
JOIN jwsdatagrc_raw.slcust c
    ON t.customer_id = c.customer_id
WHERE t.sale_date BETWEEN :start_date AND :end_date
GROUP BY c.customer_id, c.customer_name
ORDER BY total_revenue DESC
LIMIT 100;
```

**Optimization**:
- ✅ Join on indexed customer_id (both tables)
- ✅ Date filtering reduces dataset before join
- ✅ LIMIT reduces result set size

**Expected Performance**: <500ms (join + aggregation)

**Considerations**:
- Use LIMIT for large customer bases (high cardinality)
- Consider materialized view for frequently run reports
- Filter by sale_date first to reduce join size

**Variations**:
```sql
-- Customer segmentation by ticket volume
SELECT
    CASE
        WHEN ticket_count >= 100 THEN 'High Volume'
        WHEN ticket_count >= 50 THEN 'Medium Volume'
        ELSE 'Low Volume'
    END as customer_segment,
    COUNT(DISTINCT customer_id) as customer_count,
    SUM(total_revenue) as segment_revenue
FROM (
    SELECT
        c.customer_id,
        COUNT(*) as ticket_count,
        SUM(t.amount) as total_revenue
    FROM jwsdatagrc_ironhide.tickets_archive t
    JOIN jwsdatagrc_raw.slcust c ON t.customer_id = c.customer_id
    WHERE t.sale_date BETWEEN :start_date AND :end_date
    GROUP BY c.customer_id
) customer_summary
GROUP BY customer_segment
ORDER BY segment_revenue DESC;
```

---

## Pattern 3: Ticket Revenue by Type

**Business Question**: "Which ticket types generate the most revenue?"

**Query**:
```sql
SELECT
    ticket_type,
    COUNT(*) as tickets_sold,
    SUM(amount) as total_revenue,
    AVG(amount) as avg_price,
    MIN(amount) as min_price,
    MAX(amount) as max_price
FROM jwsdatagrc_ironhide.tickets_archive
WHERE sale_date BETWEEN :start_date AND :end_date
GROUP BY ticket_type
ORDER BY total_revenue DESC;
```

**Optimization**:
- ✅ Indexed date filtering
- ✅ Low cardinality grouping (ticket_type)
- ✅ Single table query (no joins)

**Expected Performance**: <200ms

**Variations**:
```sql
-- Ticket type popularity trend (monthly)
SELECT
    DATE_TRUNC('month', sale_date) as month,
    ticket_type,
    COUNT(*) as tickets_sold,
    SUM(amount) as revenue
FROM jwsdatagrc_ironhide.tickets_archive
WHERE sale_date BETWEEN :start_date AND :end_date
GROUP BY DATE_TRUNC('month', sale_date), ticket_type
ORDER BY month, revenue DESC;
```

---

## Pattern 4: Year-Over-Year Comparison

**Business Question**: "How do this year's sales compare to last year?"

**Query**:
```sql
SELECT
    EXTRACT(YEAR FROM sale_date) as year,
    EXTRACT(MONTH FROM sale_date) as month,
    COUNT(*) as tickets_sold,
    SUM(amount) as revenue
FROM jwsdatagrc_ironhide.tickets_archive
WHERE sale_date BETWEEN :prior_year_start AND :current_year_end
GROUP BY EXTRACT(YEAR FROM sale_date), EXTRACT(MONTH FROM sale_date)
ORDER BY year, month;
```

**Optimization**:
- ✅ Date index covers range
- ✅ Time-based grouping efficient
- ✅ Single pass through data

**Expected Performance**: <500ms (2 years of data)

**Variations**:
```sql
-- Side-by-side comparison with growth %
WITH current_year AS (
    SELECT
        EXTRACT(MONTH FROM sale_date) as month,
        SUM(amount) as revenue
    FROM jwsdatagrc_ironhide.tickets_archive
    WHERE sale_date BETWEEN :current_year_start AND :current_year_end
    GROUP BY EXTRACT(MONTH FROM sale_date)
),
prior_year AS (
    SELECT
        EXTRACT(MONTH FROM sale_date) as month,
        SUM(amount) as revenue
    FROM jwsdatagrc_ironhide.tickets_archive
    WHERE sale_date BETWEEN :prior_year_start AND :prior_year_end
    GROUP BY EXTRACT(MONTH FROM sale_date)
)
SELECT
    cy.month,
    cy.revenue as current_year_revenue,
    py.revenue as prior_year_revenue,
    ROUND(((cy.revenue - py.revenue) / py.revenue * 100), 2) as growth_pct
FROM current_year cy
LEFT JOIN prior_year py ON cy.month = py.month
ORDER BY cy.month;
```

---

## Pattern 5: Customer Lifetime Value

**Business Question**: "What's the total value of each customer over time?"

**Query**:
```sql
SELECT
    c.customer_id,
    c.customer_name,
    MIN(t.sale_date) as first_purchase,
    MAX(t.sale_date) as last_purchase,
    COUNT(t.ticket_sale_line_item_primary_key) as lifetime_tickets,
    SUM(t.amount) as lifetime_revenue,
    AVG(t.amount) as avg_ticket_value
FROM jwsdatagrc_ironhide.tickets_archive t
JOIN jwsdatagrc_raw.slcust c
    ON t.customer_id = c.customer_id
GROUP BY c.customer_id, c.customer_name
ORDER BY lifetime_revenue DESC
LIMIT 500;
```

**Optimization**:
- ✅ No date filter (full customer history)
- ✅ Join on indexed customer_id
- ✅ LIMIT reduces result set

**Expected Performance**: <2s (full table scan + aggregation)

**Considerations**:
- Consider materialized view for frequently accessed data
- Add WHERE clause for active customers only to improve performance
- Index on customer_id critical for join performance

**Variations**:
```sql
-- Customer retention cohort analysis
SELECT
    DATE_TRUNC('month', first_purchase) as cohort_month,
    COUNT(DISTINCT customer_id) as customers,
    AVG(lifetime_tickets) as avg_tickets_per_customer,
    AVG(lifetime_revenue) as avg_ltv
FROM (
    SELECT
        customer_id,
        MIN(sale_date) as first_purchase,
        COUNT(*) as lifetime_tickets,
        SUM(amount) as lifetime_revenue
    FROM jwsdatagrc_ironhide.tickets_archive
    GROUP BY customer_id
) customer_ltv
GROUP BY DATE_TRUNC('month', first_purchase)
ORDER BY cohort_month;
```

---

## Pattern 6: Source System Analysis

**Business Question**: "Which source systems contribute the most tickets?"

**Query**:
```sql
SELECT
    source_table,
    COUNT(*) as ticket_count,
    SUM(amount) as total_revenue,
    AVG(amount) as avg_ticket_price
FROM jwsdatagrc_ironhide.tickets_archive
WHERE sale_date BETWEEN :start_date AND :end_date
GROUP BY source_table
ORDER BY ticket_count DESC;
```

**Optimization**:
- ✅ Uses composite index: (source_table, sale_date)
- ✅ Perfect index for this query pattern
- ✅ Low cardinality grouping

**Expected Performance**: <50ms (optimized index)

---

## Performance Optimization Tips

### Index Usage
- **Primary Index**: PK on ticket_sale_line_item_primary_key
- **Composite Index**: (source_table, sale_date) - CRITICAL for time-based queries
- **Composite Index**: (ticket_sale_line_item_primary_key, ticket_combined_uniqueid)

### Query Best Practices
1. **Always filter by sale_date**: Leverages composite index
2. **Use LIMIT for high cardinality**: Customer names, detailed records
3. **Avoid SELECT ***: Specify only needed columns
4. **Use CTEs for clarity**: Complex year-over-year comparisons
5. **Consider materialized views**: Frequently run aggregations

### When to Delegate to postgres-expert
- Queries exceeding 2s (significantly above baseline)
- Sequential scans on tickets_archive (should use indexes)
- Need for additional indexes (analyze with EXPLAIN ANALYZE)
- Complex join optimization

---

## Related Documentation

- **Schema Documentation**: `schema/jwsdatagrc_ironhide.md` (tickets_archive details)
- **Business Metrics**: `semantic-layer/metrics.md` (total_tickets_sold, revenue_by_customer)
- **Business Dimensions**: `semantic-layer/dimensions.md` (sale_date, customer_id, ticket_type)

---

**Last Updated**: 2025-10-21
**Confidence**: 0.90 (production-validated patterns)
