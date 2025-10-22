# fellowship_of_data Database

**PostgreSQL Production Database** - GraniteRock business systems integration and operational data warehouse

**Host**: postgres.grc-ops.com:5432
**Database**: fellowship_of_data
**Access**: Read-only via `claude_read` user (MCP: dbhub-postgres)

---

## Quick Navigation

- **[Business Metrics](semantic-layer/metrics.md)** - Key performance indicators and business calculations
- **[Business Dimensions](semantic-layer/dimensions.md)** - How to slice data by customer, product, time, etc.
- **[Schema Documentation](schema/)** - Technical table structures and relationships
- **[Query Patterns](query-patterns/)** - Common use cases with optimization guidance

---

## Database Overview

### Purpose
Primary operational data warehouse integrating data from:
- **JWS/DataGRC Systems**: Ticket management, sales tracking, customer data
- **JD Edwards ERP**: General ledger, address book, financial systems
- **Support Systems**: Synchronization logs, utilities

### Size & Performance
- **Total Database**: ~38 GB across 41 tables in 5 schemas
- **Largest Tables**: tickets_archive (12 GB), tkhist1 (7.5 GB), tickets_tkhist1 (5.4 GB)
- **Optimization**: Partitioning on GL data, composite indexes on tickets
- **Expected Query Times**: <100ms (medium tables), <1s (large tables with indexes), <5s (partitioned)

---

## Schema Organization

### 📊 jwsdatagrc_ironhide (26 GB, 12 tables)
**Purpose**: Processed/transformed data from source systems

**Key Tables**:
- `tickets_archive` (12 GB) - Archived ticket sales with composite indexes
- `gl_posting_archive_y2024` / `gl_posting_archive_y2025` - Yearly partitioned GL data
- `tickets_tkhist1` (5.4 GB) - Ticket history processed

**Usage**: Primary analytics layer, heavily indexed, supports year-over-year analysis

[📄 Detailed Schema Documentation](schema/jwsdatagrc_ironhide.md)

---

### 📥 jwsdatagrc_raw (11 GB, 20 tables)
**Purpose**: Raw source system data (pre-transformation)

**Key Tables**:
- `tkhist1` (7.5 GB) - Ticket history raw data
- `grc_glpostingdetail` (2.6 GB) - GL posting detail raw
- `slcust` (18 MB) - Customer master data

**Usage**: ETL source layer, read-heavy, archival candidate for old data

[📄 Detailed Schema Documentation](schema/jwsdatagrc_raw.md)

---

### 🏢 jde_raw (759 MB, 4 tables)
**Purpose**: JD Edwards ERP integration layer

**Key Tables**:
- `f0901` (721 MB) - General Ledger (JDE F0901)
- `f0101` (17 MB) - Address Book Master (JDE F0101)

**Usage**: Financial reporting, ERP sync, read-mostly workload

[📄 Detailed Schema Documentation](schema/jde_raw.md)

---

### 🔧 utilities (20 MB, 2 tables)
**Purpose**: Support and synchronization logging

**Key Tables**:
- `sync_log` (20 MB) - Data sync operation tracking

[📄 Detailed Schema Documentation](schema/utilities.md)

---

### 📦 public (12 MB, 2 tables)
**Purpose**: General-purpose schema

[📄 Detailed Schema Documentation](schema/public.md)

---

## Getting Started

### For Business Questions
1. **Start with metrics**: Check [semantic-layer/metrics.md](semantic-layer/metrics.md) for pre-defined business calculations
2. **Understand dimensions**: Review [semantic-layer/dimensions.md](semantic-layer/dimensions.md) for how to slice data
3. **Find query pattern**: Browse [query-patterns/](query-patterns/) for similar use cases

### For Technical Deep Dives
1. **Review schema docs**: Start with [schema/](schema/) for table structures
2. **Check query patterns**: See [query-patterns/](query-patterns/) for optimization guidance
3. **Use MCP for exploration**: Execute queries via `mcp__dbhub-postgres__execute_sql`
4. **Consult specialist**: Delegate to `postgres-expert` for performance tuning

### For Optimization Work
1. **Understand current indexes**: Review schema docs for existing index strategies
2. **Check partitioning**: GL data uses yearly partitions (consider monthly if needed)
3. **Analyze query performance**: Use `EXPLAIN ANALYZE` via postgres-expert
4. **Consider BRIN indexes**: For time-series data on large tables

---

## Performance Baselines

### Acceptable Query Times
- **Small tables** (<100 MB): <10ms
- **Medium tables** (100 MB - 1 GB): <100ms
- **Large tables** (1-10 GB): <1s with proper indexing
- **Very large tables** (>10 GB): <5s with partitioning + indexing

### When to Investigate
- ❌ Queries exceeding 2x baseline times
- ❌ Sequential scans on tables >1 GB
- ❌ Index bloat >50% of table size
- ❌ Vacuum not running (dead tuples >10%)

---

## MCP Integration

### Available Tools
- **dbhub-postgres**: Direct database query execution via `mcp__dbhub-postgres__execute_sql`
- **postgres-expert**: PostgreSQL optimization specialist (delegates to for tuning)

### Query Execution Pattern
```markdown
1. Check semantic layer for business metric definition
2. Review query pattern for optimization guidance
3. Execute via MCP: mcp__dbhub-postgres__execute_sql
4. If performance issues → Delegate to postgres-expert for analysis
```

---

## Related Documentation

- **PostgreSQL Specialist**: `.claude/agents/specialists/postgres-expert.md`
- **DBHub MCP Configuration**: `knowledge/da-agent-hub/development/dbhub-mcp-configuration.md`
- **Data Engineer Role**: `.claude/agents/roles/data-engineer-role.md` (owns ingestion)
- **Analytics Engineer Role**: `.claude/agents/roles/analytics-engineer-role.md` (owns transformation)

---

**Last Updated**: 2025-10-21
**MCP Connection**: ✅ Validated (L1+L2 tests passed)
**Specialist Agent**: ✅ Production-ready (postgres-expert confidence 0.92)
