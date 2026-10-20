# Decisions: SILVER_AP_INVOICES Baan + Workday onboarding
Deployed 2026-10-20 to `COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES` (ACCOUNTADMIN, connection DEMO).

| # | Question | Decision |
|---|---|---|
| Q1 | Payment terms at Silver | Keep `payment_terms_raw` and add normalized `payment_terms` / `payment_terms_days` (existing table pattern) |
| Q2 | System-specific columns (BR-008) | Keep `company_code_raw` / `source_doc_type`; NULL for Baan and Workday. Removal is a separate change |
| Q3 | Target lag | Unchanged: 1 hour |
| Q4 | Unmapped statuses | `approval_status` = NULL; caught by validation |
| Q5 | Workday tenant | `WD_TENANT_ID` dropped per mapping |
| — | Workday legal sign-off | Proceed with the Workday branch included (user decision) |

## Results
- Refresh: inserted 50 rows, deleted 30; refresh_mode stays INCREMENTAL; state ACTIVE
- Counts: SAP 15, ORACLE 15, BAAN 10, WORKDAY 10. All invoice_key values unique
- Null approval_status / payment_terms / required fields: 0
- Bronze reconciliation: Baan 257,600.00 = Bronze; Workday 265,300.00 = Bronze
- SAP/Oracle rows vs `SILVER_AP_INVOICES_PRE_CLONE`: 0 differences

Rollback: `ddl/00_current.sql`. Drop `SILVER_AP_INVOICES_PRE_CLONE` once sign-off is complete.
