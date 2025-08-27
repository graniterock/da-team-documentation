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

### AP GL Codes Views

**View Names:**
- `APGLCODES_FULL`: Full dataset of general ledger codes
- `APGLCODES_INCREMENTAL`: Incremental dataset for GL codes updated on the current date
- `APGLCODES_MYSQL`: Variant for MySQL compatibility, used for Aurora table `DataServ_GLCodes`

**Purpose:** Provides general ledger account details for financial processes, with specific filtering for valid accounts.

**Source Tables:**
- `OCI_JDE_PROD_RAW.DBO.F0901` (GL account master)
- `OCI_JDE_PROD_RAW.DBO.F0902` (GL account balances)
- `OCI_JDE_PROD_RAW.DBO.F0006` (business unit master)

**Key Fields:**
- `COMPANY` (GMCO): Company code
- `BUSINESSUNIT` (GMMCU): Business unit code
- `OBJECTACCOUNT` (GMOBJ): Object account code
- `OBJECTDESCR` (GMDL01): Object account description
- `SUBSIDIARY` (GMSUB): Subsidiary code
- `SUBSIDIARYDESCR` (GMDL01): Subsidiary description
- `POSTINGEDITCODE` (GMPEC): Indicates if posting is allowed ('Y' for 'L', else 'N')

**MySQL View Additional Fields:** `SUBLEDGERREQ`, `AID`, `JDECOMPANY`, `JDEBUSINESSUNIT`, `JDEOBJECTACCOUNT`

**Filters:**
- Business units where `MCPECC <> 'N'` in `F0006`
- Excludes invalid company codes (empty or '99990')
- Ensures `GMOBJ` is numeric and excludes specific accounts (e.g., '4510', '4511')
- Includes specific account ranges (e.g., 5505-5509, 6220-6280) and exceptions for business unit '1'
- Incremental view filters for records updated on the current date (`GMUPMJ`)

**Output:** `APGLCodes070123.csv` (pipe-delimited)

**Note:** `APGLCODES_MYSQL` outputs to Aurora table `DataServ_GLCodes` until the PO process transitions from Talend.

### AP Open Purchase Orders Views

**View Names:**
- `APOPENPOS_FULL`: Full dataset of open POs
- `APOPENPOS_INCREMENTAL`: Incremental dataset for POs updated on the current date

**Purpose:** Tracks open purchase orders for procurement and financial tracking.

**Source Tables:**
- `OCI_JDE_PROD_RAW.DBO.F0101` (address book)
- `OCI_JDE_PROD_RAW.DBO.F4301` (PO header)
- `OCI_JDE_PROD_RAW.DBO.F4311` (PO detail)
- `OCI_JDE_PROD_RAW.DBO.F43121` (receipts)

**Key Fields:**
- `PO_NUMBER` (PHDOCO): Purchase order number
- `PO_LINE_NUMBER` (PDLNID): Line number
- `PO_DATE` (PHTRDJ): PO creation date (converted from Julian)
- `VENDOR_NUMBER` (ABAN8), `VENDOR_NAME` (ABALPH): Vendor details
- `BUYER` (PHANBY), `REQUESTOR`: Buyer and requestor info
- `PO_TYPE` (PHDCTO), `LINE_TYPE` (PDLNTY): PO and line types
- `ITEM_NUMBER` (PDLITM), `ITEM_PRICE` (PDPRRC), `QTY_ORDERED` (PDUORG), `ITEM_EXTD_PRICE` (PDAEXP): Item details
- `OPEN_QTY` (PRUREC), `OPEN_EXTENDED_PRICE`: Open quantities and costs
- `UOM` (PDUOM), `COMPANY` (PDCO), `BUSINESS_UNIT` (PDMCU): Additional details

**Filters:**
- PO type is 'OP'
- Excludes lines with `PDLTTR` between '980' and '998'
- Requires `PRMATC = 1`, `PRUOPN <> 0`, and `PRRCDJ >= 116001`
- Incremental view filters for records updated on the current date (`PHUPMJ`)

**Output:** `APOpenPO070123.csv` (pipe-delimited)

### AP Open Receipts Views

**View Names:**
- `APOPENRECEIPTS_FULL`: Full dataset of open receipts
- `APOPENRECEIPTS_INCREMENTAL`: Incremental dataset for receipts updated on the current date

**Purpose:** Tracks open receipts for inventory and financial reconciliation.

**Source Tables:** Same as Open POs (`F0101`, `F4301`, `F4311`, `F43121`)

**Key Fields:**
- Similar to Open POs, with additions:
- `RECEIPT_NUMBER` (PRDOC): Receipt document number
- `QTY_RECEIVED` (PRUREC), `ITEM_EXT_PRICE`: Received quantities and costs
- `OPEN_QTY` (PRUOPN), `OPEN_EXTD_PRICE`: Open quantities and costs

**Filters:**
- Same as Open POs, with additional filter `PRGLC <> 'FT60'`
- Incremental view filters for records updated on the current date (`PHUPMJ`)

**Output:** `APOpenReceipt070123.csv` (pipe-delimited)

### AP Subledger Views

**View Names:**
- `APSUBLEDGER_FULL`: Full dataset of subledger entries
- `APSUBLEDGER_INCREMENTAL`: Incremental dataset for subledger entries updated on the current date

**Purpose:** Provides subledger data for financial reporting, including work orders and equipment.

**Source Tables:**
- `OCI_JDE_PROD_RAW.DBO.F4801` (work orders)
- `OCI_JDE_PROD_RAW.DBO.F1201` (equipment master)
- `OCI_JDE_PROD_RAW.DBO.F0006` (business unit master)
- `OCI_JDE_PROD_RAW.DBO.F0101` (address book)

**Key Fields:**
- `SUBLEDGER_TYPE`: 'W' for work orders, 'E' for equipment
- `SUBLEDGER`: Work order number (`WADOCO`) or equipment number (`FANUMB`)

**Filters:**
- Work orders: `WASRST` in (' ', '05', '10', '20', '30', '40', '99') and `WASBLI = ' '`
- Equipment: `FADSP = 0`
- Incremental view filters for records updated on the current date (`WAUPMJ` or `FAUPMJ`)

**Output:** `APSubledger070123.csv` (pipe-delimited)

### AP Vendors Views

**View Names:**
- `APVENDORS_FULL`: Full dataset of vendor details
- `APVENDORS_INCREMENTAL`: Incremental dataset for vendors updated on the current date

**Purpose:** Provides vendor master data for procurement and payment processes.

**Source Tables:**
- `OCI_JDE_PROD_RAW.DBO.F0101` (address book)
- `OCI_JDE_PROD_RAW.DBO.F0111` (address book - who's who)
- `OCI_JDE_PROD_RAW.DBO.F0116` (address by date)
- `OCI_JDE_PROD_RAW.DBO.F0401` (supplier master)

**Key Fields:**
- `VENDORNUMBER` (ABAN8), `VENDORNAME` (ABALPH): Vendor identifiers
- `ADDRESS1` (ALADD1), `ADDRESS2` (ALADD2), `CITY` (ALCTY1), `STATE` (ALADDS), `POSTALCODE` (ALADDZ), `COUNTRY` (ALCTR): Address details
- `PAYMENTTERMS` (A6TRAP): Payment terms

**Filters:**
- Vendor types in ('V', 'P', 'E', 'C', 'O', 'SH', 'TA', 'REF', 'GOV', 'NP')
- Excludes `WWMLNM = 'Insurance'`, `WWIDLN = '0'`, and specific vendor numbers (1, 2, 9000001)
- Uses the latest address record (`ALEFTB`)
- Incremental view filters for records updated on the current date (`ABUPMJ`)

**Output:** `APVendorMaster070123.csv` (pipe-delimited)

### AP Payments Views

**View Names:**
- `APPAYMENTS_FULL`: Full dataset of payment records
- `APPAYMENTS_INCREMENTAL`: Incremental dataset for payments updated on the current date

**Purpose:** Tracks accounts payable payment transactions.

**Source Tables:**
- `OCI_JDE_PROD_RAW.DBO.F0411` (AP ledger)
- `OCI_JDE_PROD_RAW.DBO.F0413` (AP payment header)
- `OCI_JDE_PROD_RAW.DBO.F0414` (AP payment detail)

**Key Fields:**
- `DIN` (RPURRF): Document identifier
- `PAYMENT_NUMBER` (RMDOCM): Payment number
- `PAYMENT_AMOUNT` (RNPAAP): Payment amount (negative, divided by 100)
- `PAYMENT_DATE` (RMDMTJ): Payment date (converted from Julian)

**Filters:**
- Matches payment documents across tables
- Excludes empty `RPURRF` and zero `RPAG`
- Payment types in ('PK', 'PT')
- Excludes voided payments (`RMVDGJ = 0`)
- Incremental view filters for records updated on the current date (`RPUPMJ`)

**Output:** `APPaymentData070123.csv` (pipe-delimited)

### AP Validation Views

**View Names:**
- `APVALIDATION_FULL`: Full dataset of AP validation records
- `APVALIDATION_INCREMENTAL`: Incremental dataset for AP validations updated on the current date

**Purpose:** Validates accounts payable invoices for financial accuracy.

**Source Tables:**
- `OCI_JDE_PROD_RAW.DBO.F0411` (AP ledger)
- `OCI_JDE_PROD_RAW.DBO.F0101` (address book)

**Key Fields:**
- `DIN` (RPURRF): Document identifier
- `INVOICE_NUMBER` (RPVINV): Invoice number
- `PO_NUMBER` (RPPO): Associated PO number
- `INVOICE_DATE` (RPDIVJ), `GL_DATE` (RPDGJ), `DATE_ENTERED` (RPUPMJ): Date fields (converted from Julian)
- `INVOICE_TOTAL` (RPAG): Total invoice amount (divided by 100)
- `VENDOR_NUMBER` (RPAN8), `VENDOR_NAME` (ABALPH): Vendor details

**Filters:**
- Uses the latest `ELT_LOADED_ON` for each `RPURRF`
- Excludes voided invoices (`RPVOD <> 'V'`) and non-PV documents (`RPDCT = 'PV'`)
- Excludes empty `RPURRF`
- Limits to records within the last 380 days (`RPUPMJ`)
- Incremental view filters for records updated on the current date (`RPUPMJ`)

**Output:** `APValidation070123.csv` (pipe-delimited)

## Technical Notes

### Output Format
All views generate pipe-delimited CSV files with names like `<ViewName>070123.csv`.

### Incremental Updates
Incremental views filter for records updated on the current date using the `CONVERT_JULIAN_DATE` function on fields like `FAUPMJ`, `PHUPMJ`, etc.

### MySQL Compatibility
The `APGLCODES_MYSQL` view is tailored for Aurora compatibility, with trimmed fields and additional columns.

### Dependencies
Views rely on the `CONVERT_JULIAN_DATE` function for date conversions and join multiple JDE tables for comprehensive data extraction.

## Maintenance

### Code Repository
[GitHub Link](https://github.com/graniterock/old_snowflake_pipelines/blob/dev/dataserv/views_for_dataserv.sql)

### Updates
Monitor the source tables (`F1201`, `F0901`, etc.) for schema changes or data inconsistencies.

### Performance
Incremental views are optimized for daily updates; full views may require performance tuning for large datasets.

---
*Migrated from Confluence: Original page created 2025-07-29*