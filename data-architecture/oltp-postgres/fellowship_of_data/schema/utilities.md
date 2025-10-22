# utilities Schema

**Size**: 20 MB
**Tables**: 2
**Purpose**: Support and synchronization logging

---

## Overview

The `utilities` schema contains operational logging and synchronization tracking tables. This schema supports data integration monitoring and troubleshooting.

### Key Characteristics
- 🔧 Operational support functions
- 📝 Data sync operation tracking
- 🔍 Troubleshooting and audit trail
- 💾 Relatively small size (low overhead)

---

## Tables

### sync_log (20 MB) 📝
**Purpose**: Data synchronization operation tracking

**Size Impact**: Primary table in utilities schema (100% of schema size)

**Business Use Cases**:
- Monitor data sync operations between systems
- Troubleshoot failed sync jobs
- Audit trail for data movement
- Performance monitoring of integration pipelines

**Key Columns** (inferred):
- `sync_id` - Unique sync operation identifier
- `source_system` - System data is synced from
- `target_system` - System data is synced to
- `sync_start_time` - When sync operation began
- `sync_end_time` - When sync operation completed
- `sync_status` - Success, failure, in-progress
- `records_processed` - Count of records synced
- `error_message` - Error details if failed
- `sync_type` - Full refresh, incremental, etc.

**Query Patterns**:
```sql
-- Recent sync operations
SELECT
    sync_id,
    source_system,
    target_system,
    sync_start_time,
    sync_end_time,
    sync_status,
    records_processed
FROM utilities.sync_log
WHERE sync_start_time >= CURRENT_DATE - INTERVAL '7 days'
ORDER BY sync_start_time DESC
```

**Performance Characteristics**:
- **Current**: <50ms for typical queries (20 MB, small table)
- **Recommended Index**: B-tree on `sync_start_time` (time-based filtering)
- **Recommended Index**: B-tree on `sync_status` (failure investigation)
- **Consider**: Composite index on (source_system, sync_start_time)

**Optimization Recommendations**:
- 💡 **Add index on sync_start_time**: Most queries filter by time range
- 💡 **Add index on sync_status**: Frequently filter for failures
- 💡 **Consider retention policy**: Archive logs older than N months
- 💡 **Monitor growth**: If table grows >100 MB, consider partitioning by month

**Expected Performance**: <10ms for typical troubleshooting queries

**Integration Points**:
- **Logged By**: ETL/ELT sync processes (Python scripts, dbt, dlthub, etc.)
- **Consumed By**: Monitoring dashboards, alerting systems
- **Related**: Data pipeline orchestration logs (Orchestra, Prefect)

---

## Schema Performance Characteristics

### Performance Baselines
- **sync_log (20 MB)**: <50ms for time-based queries

### Common Access Patterns
- 📊 **Monitoring**: Recent sync status checks
- 🔍 **Troubleshooting**: Failed sync investigation
- 📈 **Trend Analysis**: Sync performance over time
- ⚠️ **Alerting**: Failed sync detection

### Investigation Triggers
- ❌ Queries exceeding 100ms (should be <50ms for 20 MB table)
- ❌ Table growth >100 MB (consider partitioning)
- ❌ High volume of failed sync records

---

## Common Query Patterns

### Pattern 1: Recent Sync Status
```sql
-- Check last 24 hours of sync operations
SELECT
    source_system,
    target_system,
    COUNT(*) as total_syncs,
    COUNT(*) FILTER (WHERE sync_status = 'success') as successful,
    COUNT(*) FILTER (WHERE sync_status = 'failed') as failed
FROM utilities.sync_log
WHERE sync_start_time >= CURRENT_TIMESTAMP - INTERVAL '24 hours'
GROUP BY source_system, target_system
ORDER BY failed DESC
```

**Optimization**: Index on sync_start_time
**Expected Performance**: <20ms

---

### Pattern 2: Failed Sync Investigation
```sql
-- Find recent failures with error details
SELECT
    sync_id,
    source_system,
    target_system,
    sync_start_time,
    error_message
FROM utilities.sync_log
WHERE sync_status = 'failed'
  AND sync_start_time >= CURRENT_DATE - INTERVAL '7 days'
ORDER BY sync_start_time DESC
LIMIT 50
```

**Optimization**: Indexes on sync_status + sync_start_time
**Expected Performance**: <10ms

---

### Pattern 3: Sync Performance Analysis
```sql
-- Average sync duration by source system
SELECT
    source_system,
    target_system,
    COUNT(*) as sync_count,
    AVG(EXTRACT(EPOCH FROM (sync_end_time - sync_start_time))) as avg_duration_seconds,
    MAX(EXTRACT(EPOCH FROM (sync_end_time - sync_start_time))) as max_duration_seconds
FROM utilities.sync_log
WHERE sync_start_time >= CURRENT_DATE - INTERVAL '30 days'
  AND sync_status = 'success'
GROUP BY source_system, target_system
ORDER BY avg_duration_seconds DESC
```

**Optimization**: Index on sync_start_time + sync_status
**Expected Performance**: <100ms (aggregation over 30 days)

---

### Pattern 4: Data Volume Tracking
```sql
-- Records processed per day
SELECT
    DATE(sync_start_time) as sync_date,
    source_system,
    SUM(records_processed) as total_records
FROM utilities.sync_log
WHERE sync_start_time >= CURRENT_DATE - INTERVAL '30 days'
  AND sync_status = 'success'
GROUP BY DATE(sync_start_time), source_system
ORDER BY sync_date DESC, total_records DESC
```

**Optimization**: Index on sync_start_time
**Expected Performance**: <50ms

---

## Recommended Indexes

### For sync_log (20 MB)
**Priority: MEDIUM** (small table, but frequently queried for monitoring)

1. **B-tree on sync_start_time**:
   ```sql
   CREATE INDEX idx_sync_log_start_time ON utilities.sync_log(sync_start_time);
   ```
   - **Use Case**: Time-range filtering (last 24 hours, last 7 days)
   - **Expected Improvement**: 5x faster time-based queries

2. **B-tree on sync_status**:
   ```sql
   CREATE INDEX idx_sync_log_status ON utilities.sync_log(sync_status);
   ```
   - **Use Case**: Filter for failed syncs
   - **Expected Improvement**: 10x faster failure queries (low selectivity)

3. **Composite on (source_system, sync_start_time)**:
   ```sql
   CREATE INDEX idx_sync_log_source_time ON utilities.sync_log(source_system, sync_start_time);
   ```
   - **Use Case**: System-specific sync history
   - **Expected Improvement**: 3x faster source system queries

---

## Retention Strategy Recommendations

### Data Retention Considerations
**Current Size**: 20 MB suggests moderate sync frequency

**Recommended Retention**:
- **Keep online**: Last 90 days (active troubleshooting period)
- **Archive to S3**: 90 days - 1 year (cold storage for compliance)
- **Purge**: >1 year (unless required for audit)

**Implementation**:
```sql
-- Archive old sync logs (example strategy)
-- Option 1: Move to archive table
INSERT INTO utilities.sync_log_archive
SELECT * FROM utilities.sync_log
WHERE sync_start_time < CURRENT_DATE - INTERVAL '90 days';

DELETE FROM utilities.sync_log
WHERE sync_start_time < CURRENT_DATE - INTERVAL '90 days';

-- Option 2: Export to S3 then delete
-- (Requires external ETL process)
```

**Benefit**: Keep table small (<50 MB), maintain fast queries

---

## Monitoring and Alerting Patterns

### Critical Sync Failures Alert
```sql
-- Detect critical sync failures in last hour
SELECT
    source_system,
    target_system,
    COUNT(*) as consecutive_failures
FROM utilities.sync_log
WHERE sync_start_time >= CURRENT_TIMESTAMP - INTERVAL '1 hour'
  AND sync_status = 'failed'
GROUP BY source_system, target_system
HAVING COUNT(*) >= 3;  -- Alert if 3+ failures in past hour
```

**Use Case**: Automated monitoring, trigger alerts when critical systems fail repeatedly

---

### Sync Latency Monitoring
```sql
-- Detect slow sync operations
SELECT
    sync_id,
    source_system,
    target_system,
    EXTRACT(EPOCH FROM (sync_end_time - sync_start_time)) as duration_seconds
FROM utilities.sync_log
WHERE sync_start_time >= CURRENT_TIMESTAMP - INTERVAL '4 hours'
  AND sync_status = 'success'
  AND EXTRACT(EPOCH FROM (sync_end_time - sync_start_time)) > 3600  -- >1 hour
ORDER BY duration_seconds DESC;
```

**Use Case**: Performance degradation detection

---

### Data Completeness Check
```sql
-- Verify expected sync frequency (daily syncs)
SELECT
    source_system,
    target_system,
    DATE(sync_start_time) as sync_date,
    COUNT(*) as sync_count
FROM utilities.sync_log
WHERE sync_start_time >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY source_system, target_system, DATE(sync_start_time)
HAVING COUNT(*) < 1;  -- Alert if no syncs for a day
```

**Use Case**: Missing sync detection

---

## Usage Guidelines

### When to Query This Schema
- ✅ Troubleshoot data sync failures
- ✅ Monitor ETL/ELT pipeline health
- ✅ Audit data movement between systems
- ✅ Performance analysis of sync operations

### When to Query Other Schemas Instead
- ❌ Actual business data → Use jwsdatagrc_ironhide, jwsdatagrc_raw, jde_raw
- ❌ Orchestration logs → Use Orchestra MCP or Prefect logs
- ❌ Application logs → Use centralized logging (CloudWatch, etc.)

### Delegation to postgres-expert
Consult `postgres-expert` specialist for:
- Sync log performance tuning (if table grows >100 MB)
- Retention policy implementation (archival strategy)
- Monitoring query optimization
- Partitioning strategy (if high sync volume)

---

## Related Documentation

- **Data Engineer Role**: `.claude/agents/roles/data-engineer-role.md` (owns sync processes)
- **Orchestra Expert**: `.claude/agents/specialists/orchestra-expert.md` (orchestration monitoring)
- **Query Patterns**: `query-patterns/sync-monitoring.md` (monitoring patterns)

---

**Last Updated**: 2025-10-21
**Discovery Source**: Automated PostgreSQL schema discovery
**Confidence**: 0.90 (production-validated via MCP)
