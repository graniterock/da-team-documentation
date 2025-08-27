# OLTP - Postgres

## Data Landing Strategy

All data is landed in Postgres on AWS. 

### Architecture Decision: Postgres vs Iceberg

**Reasoning:** 75%+ of all GRC data is already in structured format. It doesn't make sense to move it from structured (MS-SQL) to unstructured (Iceberg - parquet) back to structured (Snowflake).

This approach maintains data in its natural structured format throughout the pipeline, reducing unnecessary transformations and improving performance.

## Related Documentation

- [Initial Setup](../postgres/initial-setup.md)
- [Apex Tickets (Bronze Layer)](../apex/apex-tickets-bronze-layer.md) 
- [Apex GL Posting (Bronze Layer)](../apex/apex-gl-posting-bronze-layer.md)

## DBT Documentation for Postgres

For detailed dbt model documentation related to Postgres operations, see the dbt documentation section.

---
*Migrated from Confluence: Original page created 2025-07-16*