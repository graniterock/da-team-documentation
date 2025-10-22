# jwsdatagrc_raw Schema

**Size**: 11 GB
**Tables**: 20
**Purpose**: Raw source system data (pre-transformation)

---

## Overview

The `jwsdatagrc_raw` schema contains raw data from source systems before transformation. This is the ETL source layer, optimized for read-heavy workloads during data ingestion and transformation processes.

### Key Characteristics
- 📥 Raw source data (not transformed)
- 📊 Read-heavy workload for ETL processes
- 🗄️ Archival candidate for old data
- 🔍 Primary use: Data quality validation, source investigation

---

## Tables

### tkhist1 (7.5 GB) 📊
**Purpose**: Ticket history raw data

**Size Impact**: Largest table in raw schema - primary bottleneck candidate

**Performance Characteristics**:
- Read-heavy for ETL processes
- Large table size impacts query performance
- May benefit from archival strategies for old data

**Optimization Recommendations**:
- 🔍 **Needs analysis**: Check `pg_stat_statements` for common query patterns
- 💡 **Likely needs**: Composite index on (timestamp + primary key columns)
- 💡 **Consider**: Archival strategy for records older than N years
- 💡 **Consider**: Partitioning by date if not already implemented

**Expected Performance**: <2s with proper indexing (large table)

**Transformed To**: `jwsdatagrc_ironhide.tickets_tkhist1` (processed version)

---

### grc_glpostingdetail (2.6 GB) 📊
**Purpose**: General ledger posting detail raw data

**Size**: Medium-large table
**Usage**: Source for GL transformations

**Performance**: <2s for typical queries
**Optimization**: Consider indexing on posting_date and account_code

**Transformed To**: `jwsdatagrc_ironhide.gl_posting_archive_*` (yearly partitions)

---

### tkohist (939 MB) 📊
**Purpose**: Ticket order history raw data

**Size**: Medium table
**Expected Performance**: <500ms with indexing

**Transformed To**: `jwsdatagrc_ironhide.tickets_tkohist`

---

### grc_ticketaux (90 MB) 📊
**Purpose**: Ticket auxiliary data

**Size**: Small-medium table
**Expected Performance**: <100ms

---

### tkbatch (44 MB) 📊
**Purpose**: Ticket batch data

**Size**: Small table
**Expected Performance**: <50ms

---

### slcust (18 MB) 📊
**Purpose**: Customer master data

**Size**: Small table
**Performance**: <10ms for lookups

**Usage**:
- Customer dimension for ticket analysis
- Frequently joined with `jwsdatagrc_ironhide.tickets_archive`
- Reference data for customer attributes

**Join Pattern**:
```sql
-- Common join to enrich ticket data with customer details
SELECT
    t.*,
    c.customer_name,
    c.customer_type
FROM jwsdatagrc_ironhide.tickets_archive t
JOIN jwsdatagrc_raw.slcust c ON t.customer_id = c.customer_id
```

**Expected Join Performance**: <500ms (small dimension table)

---

## Schema Performance Characteristics

### Performance Baselines
- **Small tables** (<100 MB): <10ms
- **Medium tables** (100 MB - 1 GB): <100ms
- **Large tables** (>1 GB): <2s (raw data, less optimized than processed)

### Common Access Patterns
- 📥 **ETL reads**: Batch extraction for transformation
- 🔍 **Data quality checks**: Validate source data integrity
- 🛠️ **Troubleshooting**: Investigate transformation issues

### Investigation Triggers
- ❌ ETL jobs taking longer than expected
- ❌ Sequential scans on large tables (tkhist1, grc_glpostingdetail)
- ❌ Table growth exceeding storage capacity

---

## Recommended Optimization Strategies

### For tkhist1 (7.5 GB)
**Problem**: Largest raw table, primary bottleneck
**Solutions**:
1. **Indexing**: Composite index on frequently queried columns
2. **Partitioning**: By date range if not already implemented
3. **Archival**: Move old data to cold storage (S3, glacier)
4. **Incremental Load**: Only query recent data in ETL

**Priority**: HIGH (largest performance impact)

---

### For grc_glpostingdetail (2.6 GB)
**Problem**: Medium-large table without visible optimization
**Solutions**:
1. **Indexing**: posting_date, account_code
2. **Incremental ETL**: Query only new/modified records
3. **Consider**: If monthly GL processing, filter by month efficiently

**Priority**: MEDIUM

---

### For Other Tables
**Strategy**: Monitor and optimize reactively
- Most tables <1 GB, acceptable performance
- Add indexes as access patterns emerge
- Review quarterly with `pg_stat_statements`

---

## Common Query Patterns

### Pattern 1: Full Table Extraction (ETL)
```sql
-- Load all data since last ETL run
SELECT *
FROM jwsdatagrc_raw.tkhist1
WHERE last_modified_date > :last_etl_timestamp
```

**Optimization**: Index on last_modified_date (if column exists)
**Expected Performance**: Depends on volume, aim for <5s

---

### Pattern 2: Data Quality Validation
```sql
-- Check for null customer IDs in raw data
SELECT COUNT(*)
FROM jwsdatagrc_raw.tkhist1
WHERE customer_id IS NULL
  AND sale_date >= CURRENT_DATE - INTERVAL '7 days'
```

**Optimization**: Partial index on NULL values (if frequent check)
**Expected Performance**: <1s

---

### Pattern 3: Customer Lookup
```sql
-- Get customer details for enrichment
SELECT customer_id, customer_name, customer_type
FROM jwsdatagrc_raw.slcust
WHERE customer_id IN (:customer_ids)
```

**Optimization**: Primary key index on customer_id
**Expected Performance**: <10ms (small table)

---

## Archival Strategy Recommendations

### Candidates for Archival
1. **tkhist1** (7.5 GB)
   - Archive records older than 3 years
   - Estimated savings: 30-50% table size
   - Restore on-demand for historical analysis

2. **grc_glpostingdetail** (2.6 GB)
   - Archive records older than 5 years
   - Keep current + prior 2 years for audits
   - Cold storage for compliance retention

### Archival Targets
- **AWS S3**: Long-term storage, query via Athena if needed
- **PostgreSQL Partitioning**: Move old partitions to separate tablespace
- **Data Lake**: Integrate with broader data lake strategy

**Benefit**: Reduce database size by 30-40%, improve query performance

---

## Usage Guidelines

### When to Query This Schema
- ✅ ETL source validation
- ✅ Data quality checks
- ✅ Investigating transformation logic
- ✅ Raw data comparisons

### When to Query jwsdatagrc_ironhide Instead
- ❌ Business analytics (use processed/indexed data)
- ❌ Historical trend analysis (partitioned for performance)
- ❌ BI reporting (optimized for queries)

### Delegation to postgres-expert
Consult `postgres-expert` specialist for:
- Large table query optimization (tkhist1, grc_glpostingdetail)
- Archival strategy implementation
- ETL performance tuning
- Index strategy for raw data access

---

## Transformation Mapping

**Raw → Processed**:
- `tkhist1` → `jwsdatagrc_ironhide.tickets_tkhist1`
- `grc_glpostingdetail` → `jwsdatagrc_ironhide.gl_posting_archive_*`
- `tkohist` → `jwsdatagrc_ironhide.tickets_tkohist`
- `slcust` → Referenced directly (dimension table)

---

## Related Documentation

- **Processed Schema**: `jwsdatagrc_ironhide.md` (transformation target)
- **ETL Patterns**: See data-engineer-role documentation
- **Data Quality**: See testing-patterns.md

---

**Last Updated**: 2025-10-21
**Discovery Source**: Automated PostgreSQL schema discovery
**Confidence**: 0.90 (production-validated via MCP)
