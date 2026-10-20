# PRD-Driven Update: Artifacts to Save
Change: onboard Baan + Workday into `COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES`
Status: plan drafted, DDL compiled only (not deployed). Open questions Q1–Q5 unresolved.

Legend: ✅ exists in repo · ⬜ to be saved

## 1. Inputs (PRD source of truth)
| | Path | Purpose |
|---|---|---|
| ✅ | `sample_csv/sample_business_requirements_source_onboarding.csv` | SRC-2025-003 (Baan), SRC-2025-004 (Workday) |
| ✅ | `sample_csv/sample_business_requirements_business_rules.csv` | BR-001…BR-010 |
| ✅ | `sample_csv/sample_business_requirements_column_mapping.csv` | Column-level Baan/Workday → Silver mapping |
| ⬜ | `prd/ap_invoices_phase2.xlsx` | Same three sheets combined into one workbook (the skill's supported format) |

## 2. Review artifacts
| | Path | Purpose |
|---|---|---|
| ⬜ | `changes/silver_ap_invoices/PLAN.md` | Full skill output: summary, mapping, conflicts, open questions, assumptions, next steps |
| ⬜ | `changes/silver_ap_invoices/DECISIONS.md` | Q1–Q5 with owner, decision, date. Blocks deployment until filled in |
| ⬜ | `changes/silver_ap_invoices/ddl/00_current.sql` | `GET_DDL` of the live table before the change (baseline for diffs and rollback) |
| ⬜ | `changes/silver_ap_invoices/ddl/01_silver_ap_invoices.sql` | Proposed `CREATE OR ALTER` DDL, labeled DRAFT |
| ⬜ | `changes/silver_ap_invoices/ddl/99_rollback.sql` | Re-apply `00_current.sql`, or swap with the pre-change clone |

## 3. Rerunnable checks
| | Path | Purpose |
|---|---|---|
| ⬜ | `changes/silver_ap_invoices/checks/pre_bronze_profile.sql` | Status/terms/currency profile, Baan duplicates, cross-source number collisions |
| ⬜ | `changes/silver_ap_invoices/checks/pre_clone.sql` | `CREATE ... CLONE` of Silver before deploying, used for diffs |
| ⬜ | `changes/silver_ap_invoices/checks/post_validation.sql` | Validation queries 1–8 (counts 15/15/10/10, key uniqueness, NULL status/terms, required fields, Bronze reconciliation, refresh history) |
| ⬜ | `changes/silver_ap_invoices/checks/expected_results.md` | Expected value for each check plus the actual result after the run, so reviewers can compare |

Run order: `pre_bronze_profile` → `pre_clone` → deploy `01` → `post_validation` → `data_diff`
of the clone vs Silver on `invoice_key` for SAP/Oracle (expect no changes).

## 4. Reusable skill
| | Path | Purpose |
|---|---|---|
| ✅ | `.cortex/skills/prd-to-dynamic-table-plan/SKILL.md` | The PRD Evaluator skill (invoke as `$prd-to-dynamic-table-plan`) |
| ✅ | `.cortex/skills/prd-to-dynamic-table-plan/references/example-ap-invoices.md` | Worked example. Update it with the real FQN (`COCO_WORKSHOP.PIPELINE_LAB`) |
| ⬜ | Skill update: CSV support | This run used CSV as best effort. Add CSV to the supported formats |
| ⬜ | Skill update: drift check | Add a step that compares the live table with the PRD (this run found drift on payment terms, retained system-specific columns, and lag) |

## 5. Handoff notes for the next engineer
- Inputs used: `prd_path=sample_csv/...column_mapping.csv` (plus the rules and onboarding CSVs), `target_dynamic_table=SILVER_AP_INVOICES`
- Connection `DEMO`, role ACCOUNTADMIN, warehouse COMPUTE_WH
- Do not deploy until DECISIONS.md resolves Q1 (payment terms) and Q2 (system-specific columns)
- The Workday branch depends on legal sign-off (DPA-2025-0041). Deploy without it if that's still pending
