# Apex Tickets (Bronze Layer)

## Data Replication Overview

Data Replication is from GRC to AWS Postgres via DMS.   
[AWS Data Migration Services - Tickets](https://us-west-2.console.aws.amazon.com/dms/v2/home?region=us-west-2#tasks/provisioned/apex-to-postgres-tickets)  

Each ticket table (tkbatch, tkeother, tkhist1, tkohist) have a function and are called by three triggers (insert, update, delete).

## Archiving Strategy

There is a weekly job in Orchestra that Archives the tickets data to a table weekly to keep the table size small for real-time reporting.  
[Orchestra Pipeline - Archive Tickets](https://app.getorchestra.io/pipeline-runs/7b59ed5d-4cd5-4063-b465-2a2a17a7f439/lineage)

## Table Processing Details

### Tkbatch Table

**Source:** `jwsdatagrc_raw.tkbatch`  
**Function:** `update_tickets_tkbatch`  
**Triggers:** 3 triggers (after statement insert, update or delete)

**Steps fired by Insert, Update or Delete:**

1. Execute trigger
2. If successful insert into `jwsdatagrc_ironhide.tickets_tkbatch` and create a record in `utilities.sync_log`
3. If error rollback and create a record in `utilities.error_log`

**Function Logic:**

The function checks `TG_OP` to execute the appropriate logic:

* **For DELETE (`TG_OP = 'DELETE'`):**
  - Deletes records from `tickets_tkbatch` where `ticket_item_level_combined_uniqueid` matches `CONCAT(ticketno, '-', uniqueid, '-', CAST(itemno AS INTEGER))` from `old_table`
  - Captures `deleted_rows` using `GET DIAGNOSTICS`
        
* **For INSERT or UPDATE (`TG_OP = 'INSERT' OR TG_OP = 'UPDATE'`):**
  - Inserts BASE, FREIGHT, and SALES_TAX records into `tickets_tkbatch` using `new_table`
  - Captures `inserted_rows` using `GET DIAGNOSTICS`
        
The `DELETE` precedes the `INSERT` for `UPDATE` to ensure old records are removed before new ones are added.

[Hex Code Reference](https://app.hex.tech/76875139-83bb-420e-9a8e-c55fe6cb1dc8/app/tkbatch---first-030LwVPCnyJg30PVURK4Ok/latest)

### Tkeother Table

**Source:** `jwsdatagrc_raw.tkeother`  
**Function:** `update_tickets_tkeother`  
**Triggers:** 3 triggers (after statement insert, update or delete)

**Processing Logic:** Similar to Tkbatch with same trigger pattern and function logic structure.

[Hex Code Reference](https://app.hex.tech/76875139-83bb-420e-9a8e-c55fe6cb1dc8/app/0197ce77-6e20-7993-a1c5-030e15691956/latest)

### Tkhist1 Table

**Source:** `jwsdatagrc_raw.tkhist1`  
**Function:** `update_tickets_tkhist1`  
**Triggers:** 3 triggers (after statement insert, update or delete)

**Enhanced Processing Steps:**

1. Execute trigger
2. If successful insert into `jwsdatagrc_ironhide.tickets_tkhist1` and create a record in `utilities.sync_log`
3. If an existing record exists in `jwsdatagrc_ironhide.tickets_archive` using `ticket_combined_uniqueid` as the key, then upsert into `jwsdatagrc_ironhide.tickets_archive` and delete from `jwsdatagrc_ironhide.tickets_tkhist1`
4. If successful re-archive then create a record in `utilities.sync_log`
5. If error rollback and create a record in `utilities.error_log`

[Hex Code Reference](https://app.hex.tech/76875139-83bb-420e-9a8e-c55fe6cb1dc8/app/0197cce2-7ee0-7118-a232-561fe4783e08/latest)

### Tkohist Table

**Source:** `jwsdatagrc_raw.tkohist`  
**Function:** `update_tickets_tkohist`  
**Processing Logic:** Similar to Tkhist1 with automatic archiving capabilities

[Hex Code Reference](https://app.hex.tech/76875139-83bb-420e-9a8e-c55fe6cb1dc8/app/0197ce67-af4d-7882-9b55-34589738cb76/latest)

## Archive Functions

There are two functions to archive ticket data:
- `jwsdatagrc_ironhide.archive_tickets_tkhist1` 
- `jwsdatagrc_ironhide.archive_tickets_tkohist`

**Archive Logic:**
1. Functions look at the `MAX(archived_date)` from `jwsdatagrc_ironhide.gl_posting_archive`
2. Select all records using `ticket_sale_line_item_primary_key` from `jwsdatagrc_ironhide.tickets_tkohist` & `jwsdatagrc_ironhide.tickets_tkhist1`
3. Insert the records into `jwsdatagrc_ironhide.gl_posting_archive`
4. Remove them from the corresponding table

[Archive Hex Code Reference](https://app.hex.tech/76875139-83bb-420e-9a8e-c55fe6cb1dc8/app/0197d1bb-3269-7007-a5d7-57e74294bb70/latest)

## Architecture Diagram

*[Architecture diagram would be included here - migrated from original Confluence page]*

---
*Migrated from Confluence: Original page created 2025-07-16*