# public Schema

**Size**: 12 MB
**Tables**: 2
**Purpose**: General-purpose schema (PostgreSQL default)

---

## Overview

The `public` schema is PostgreSQL's default schema. In this database, it contains a small amount of general-purpose data or utility tables.

### Key Characteristics
- 📦 Default PostgreSQL schema
- 💾 Small size (12 MB total)
- 🔧 General-purpose or miscellaneous tables
- ✅ Low performance impact

---

## Tables

The specific table names and purposes are not detailed in the discovery output, but based on the schema size (12 MB total across 2 tables), these are likely:

- **Small utility or reference tables** (<10 MB each)
- **Possible uses**: Configuration, lookup tables, temporary staging

**Performance Characteristics**:
- **Size**: 12 MB across 2 tables (6 MB average per table)
- **Expected Performance**: <10ms for typical queries (small tables)
- **Indexing**: Depends on table purpose and access patterns

---

## General Guidance

### Query Performance
- **Small tables** (<10 MB): <10ms expected
- **No special optimization**: Basic indexes sufficient
- **Low priority**: Other schemas more critical for analytics

### When to Use
- ✅ General-purpose or utility functions
- ✅ Configuration lookups
- ✅ Temporary staging (if applicable)

### When to Use Other Schemas
- ❌ Business analytics → Use `jwsdatagrc_ironhide`
- ❌ Raw source data → Use `jwsdatagrc_raw`
- ❌ Financial reporting → Use `jde_raw`
- ❌ Sync monitoring → Use `utilities`

---

## Schema Discovery Needed

To fully document this schema, run:

```sql
-- List tables in public schema
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Get column details for each table
SELECT
    table_name,
    column_name,
    data_type,
    is_nullable
FROM information_schema.columns
WHERE table_schema = 'public'
ORDER BY table_name, ordinal_position;
```

---

## Delegation to postgres-expert

Consult `postgres-expert` specialist for:
- Detailed table discovery in public schema
- Determining purpose of public schema tables
- Optimization if queries become slow

---

## Related Documentation

- **Database Overview**: `README.md` (main database navigation)
- **Schema Discovery**: `../README.md` (automated discovery patterns)

---

**Last Updated**: 2025-10-21
**Discovery Source**: Automated PostgreSQL schema discovery (partial)
**Confidence**: 0.70 (limited information available, needs detailed discovery)
