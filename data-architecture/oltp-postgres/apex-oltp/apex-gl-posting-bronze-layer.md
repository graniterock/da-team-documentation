# Apex GL Posting (Bronze Layer)

## Data Replication Overview

Data Replication is from GRC to AWS Postgres via DMS.   
[AWS Data Migration Services - Apex GL](https://us-west-2.console.aws.amazon.com/dms/v2/home?region=us-west-2#tasks/provisioned/apex-to-postgres-gl)

## Archiving Strategy

There is a weekly job in Orchestra that Archives the GL data to a table weekly to keep the table size small for real-time reporting.  
[Orchestra Pipeline - Run Archive](https://app.getorchestra.io/pipeline-runs/7b59ed5d-4cd5-4063-b465-2a2a17a7f439/lineage)

## GL Posting Data Architecture

The GL Posting data is split into 3 parts:

### 1. VW_GL_POSTING
**View:** `jwsdatagrc_ironhide.vw_gl_posting`  
**Purpose:** View of all GL Postings

Straight join on both tables with transformation logic in view:

```sql
FROM jwsdatagrc_raw.grc_glpostingbatch c
JOIN jwsdatagrc_raw.grc_glpostingdetail t2 ON c.batchid = t2.batchid
```

[Hex Code Reference](https://app.hex.tech/76875139-83bb-420e-9a8e-c55fe6cb1dc8/app/GL-Posting-030LPYeqTmvxbI21mtOKw9/latest)

### 2. MV_GL_POSTING
**View:** `jwsdatagrc_ironhide.mv_gl_posting`  
**Purpose:** Materialized view of all GL Postings that aren't final and older than 7 days (already archived)

Same logic as above view except filtered for all data not already archived. This refreshes daily and can be manually triggered to refresh the view.

```sql
FROM jwsdatagrc_raw.grc_glpostingbatch c
JOIN jwsdatagrc_raw.grc_glpostingdetail t2 ON c.batchid = t2.batchid
WHERE NOT (c.isproof = 'N' AND t2.batchenddate < CURRENT_DATE - INTERVAL '7 days')
```

[Orchestra Pipeline - Refresh View](https://app.getorchestra.io/pipeline-runs/f442c6f8-f29d-4200-8cd7-97aad61dcc6e/lineage)

### 3. GL_POSTING_ARCHIVE
**Table:** `jwsdatagrc_ironhide.gl_posting_archive`  
**Purpose:** Archive table of all GL Postings that are final and older than 7 days

Reverse logic as materialized view. Data is archived to this table weekly via a stored procedure `jwsdatagrc_ironhide.archive_gl_posting` that is kicked off via Orchestra.

```sql
FROM jwsdatagrc_ironhide.vw_gl_posting
WHERE is_proof = 'N' AND AGE(CURRENT_DATE, batch_end_date) > INTERVAL '7 days'
```

[Orchestra Pipeline - Archive GL Postings](https://app.getorchestra.io/pipeline-runs/7b59ed5d-4cd5-4063-b465-2a2a17a7f439/lineage)

## Architecture Diagram

*[Architecture diagram would be included here - migrated from original Confluence page]*

---
*Migrated from Confluence: Original page created 2025-07-16*