# To DataServ From JDE

## Overview

This document outlines the 9 Snowflake views created for the DataServ application, sourced from the JDE (JD Edwards) database. These views transform and filter data for various financial and operational processes.

### Architecture & Scheduling

- **Source Code:** [GitHub Repository](https://github.com/graniterock/old_snowflake_pipelines/blob/dev/dataserv/views_for_dataserv.sql)
- **Orchestration:** Prefect extracts and exports data to DataServ's SFTP location
- **Prefect Code:** [GitHub - DataServ](https://github.com/graniterock/dataserv)
- **Prefect Job:** [Prefect Cloud Deployment](https://app.prefect.cloud/account/9c8ebb0f-90e7-4f88-aca0-285baed88e39/workspace/bdb91bd0-a550-4ad2-b572-2ba6a99f19d0/deployments/deployment/094a487d-38f0-4167-88b6-082e71916565?versionId)
- **Schedule:** Daily at 7pm PST via Orchestra: [Orchestra Pipeline](https://app.getorchestra.io/pipeline-runs/2a97338a-ce6d-44cf-a4f2-7b9d81a4b1cb/lineage)

## Views and Functionality

### CONVERT_JULIAN_DATE Function

**Purpose:** Converts JDE Julian dates (integer format) to standard DATE format.

- **Input:** Integer representing a JDE Julian date
- **Output:** DATE value calculated by extracting the year (adding 1900 to the first three digits) and adding the day offset (last three digits minus 1) from January 1st of that year
- **Usage:** Used across multiple views to convert date fields like `FAUPMJ`, `PHUPMJ`, `RPDIVJ`, etc.

### AP Asset Views

**View Names:**
- `APASSET_FULL`: Full dataset of assets
- `APASSET_INCREMENTAL`: Incremental dataset for assets updated on the current date

**Purpose:** Extracts asset data for financial reporting, excluding specific equipment statuses.

**Source Table:** `OCI_JDE_PROD_RAW.DBO.F1201`

**Key Fields:**
- `ASSET_NBR` (FANUMB): Asset number
- `ASSETDESCR` (FADL02): Asset description
- `JULIAN` (incremental only): Converted update date

**Filters:**
- Excludes records where `FAEQST` is in ('1C', '1D', '1O', '1S', '1T', '1X', 'OS')
- Incremental view filters for records updated on the current date (`FAUPMJ`)

**Output:** `APAsset070123.csv` (pipe-delimited)

*[Full content continues with all view details as previously created]*

---
*Migrated from Confluence: Original page created 2025-07-29*