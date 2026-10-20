# Example: AP Invoices (Baan + Workday onboarding)

## Invocation
```
$prd-to-dynamic-table-plan
prd_path: ./prd/ap_invoices_phase2.xlsx
target_dynamic_table: FINANCE_DB.SILVER.SILVER_AP_INVOICES
layer: silver
```
The workbook has three sheets: `Source Onboarding` (SRC-2025-003/004), `Business Rules`
(BR-001…BR-010), and `Column Mapping`.

## Expected output (abridged)

# Change Plan: FINANCE_DB.SILVER.SILVER_AP_INVOICES
Inputs: ap_invoices_phase2.xlsx (3 sheets) | DT inspected: yes | Generated: 2026-10-20

## 1. Summary
Add BAAN and WORKDAY branches to the UNION ALL, extend the status CASE, and add Baan
dedup. 10 rules: 5 in scope, 4 handled elsewhere, 1 blocked (BR-005). Workday go-live
is blocked on legal sign-off (DPA-2025-0041).

## 2. New Source Systems
| Request ID | System | Region | Ingestion / arrival | Go-live | Readiness | Blocking items |
|---|---|---|---|---|---|---|
| SRC-2025-003 | Baan IV | EMEA | Nightly CSV → S3, ~02:00 UTC | Q3 2025 | Approved | Mapping in progress |
| SRC-2025-004 | Workday FM | Americas + London (WD-T2) | Hourly connector | Q3 2025 | Pending | Legal DPA review |

## 3. Changes to the Dynamic Table
| # | Change | Type | Source rows | Basis | Current state |
|---|---|---|---|---|---|
| 1 | Add BAAN branch, `'BAAN' AS SOURCE_SYSTEM` | branch | BR-007, SRC-2025-003 | Stated | Absent |
| 2 | Add WORKDAY branch, `'WORKDAY' AS SOURCE_SYSTEM` | branch | BR-007, SRC-2025-004 | Stated | Absent |
| 3 | Status CASE: POSTED→APPROVED; Approved→APPROVED; In Review→PENDING | mapping | BR-001 | Stated | SAP/Oracle only |
| 4 | Baan dedup via QUALIFY ROW_NUMBER() | dedup | BR-003 | Stated | Absent |
| 5 | Drop BAN_COMPANY, WD_TENANT_ID | column | BR-008 | Stated | n/a |
| 6 | Keep original currency, GL code, payment terms | column | BR-002, BR-006, BR-005 | Stated | Matches pattern |

## 4. Out of Scope / Handled Elsewhere
| Rule ID | Rule | Where it belongs |
|---|---|---|
| BR-002 | FX conversion | Gold (Treasury FX table) |
| BR-004 | >$500K review flag | DMF in Guardrails Pack |
| BR-006 | Unified chart of accounts | Gold, Phase 2 |
| BR-010 | History retention | Bronze / Phase 2 backlog |

## 5. Conflicts & Gaps
| # | Issue | Rows involved | Risk if ignored |
|---|---|---|---|
| 1 | Dedup partitions on INVOICE_NUMBER only, so it could merge invoices from different sources in the UNION | BR-003 | Silent row loss |
| 2 | Threshold is "USD equivalent", but there's no FX at Silver and the FX table is out of scope | BR-004 vs BR-002 | DMF can't be computed |
| 3 | Only some status values are mapped (no Rejected/Cancelled/Draft) | BR-001 | NULL or wrong status |
| 4 | Baan cost center format changed (BC-XX → BC-XXX) | SRC-2025-003 | Inconsistent joins |
| 5 | WD_TENANT_ID dropped, but the London entity sits inside the Americas feed | BR-008, SRC-2025-004 | Lost entity/region info |

## 6. Open Questions
| # | Priority | Question | Recommended answer | Owner | Blocks |
|---|---|---|---|---|---|
| 1 | Blocker | Scope the dedup to the Baan branch, or partition by (SOURCE_SYSTEM, INVOICE_NUMBER)? | Baan branch only | David Kim | Change #4 |
| 2 | Blocker | Default for unmapped statuses? | Pass through + DMF alert | Sarah Chen | Change #3 |
| 3 | High | Normalize payment terms at Silver or Gold? | Gold (per PRD rec.) | Sarah Chen / David Kim | BR-005 |
| 4 | High | Which FX source does the $500K DMF use? | Static rate table until Treasury FX exists | Tom Walsh | BR-004 |
| 5 | High | Keep entity/tenant as a standard column? | Add ENTITY_CODE | Jennifer Okafor | Change #5 |
| 6 | Low | Normalize the old BC-XX cost centers? | Keep as-is at Silver | Karen van der Berg | — |

## 7. Assumptions
1. Bronze Baan/Workday tables already exist. Inferred from the PRD; confirm with `SHOW TABLES`.
2. Dedup tie-breaker = BAN_INVOICE_ID DESC. Inferred; confirm with Karen.

## 8. Next Steps
- [ ] Resolve open questions 1–2 (blockers)
- [ ] Draft the DDL for changes 1–6
- [ ] Validate in a dev clone and diff row counts per SOURCE_SYSTEM
- [ ] Hand off to the `dynamic-tables` skill to apply
