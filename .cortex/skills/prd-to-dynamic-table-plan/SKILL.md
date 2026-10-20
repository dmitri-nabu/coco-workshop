---
name: prd-to-dynamic-table-plan
description: "Turn PRD-style requirement files (XLSX) into an implementation plan for changing a target Snowflake Dynamic Table. Use when: a PRD, requirements workbook, source onboarding sheet, business rules sheet, or column mapping must be applied to a Silver/Gold dynamic table. Triggers: PRD to dynamic table, plan from requirements, requirements xlsx, onboard new source into DT, apply business rules to silver, change plan for SILVER_*, impact of PRD on pipeline."
---

# PRD → Dynamic Table Change Plan

Reads requirement workbooks, compares them with the current definition of a target
Dynamic Table, and returns a change plan that is grounded in evidence and easy to review.
**The skill plans; it does not implement.** It never runs DDL/DML.

## When to Use
- A stakeholder hands over PRD-style requirements (source onboarding, business rules,
  column mappings) that will change a Dynamic Table.
- You need a gap analysis of what the requirements ask for versus what the DT does today.
- Do NOT use this skill to build or alter the DT. For that, hand the approved plan to the
  `dynamic-tables` skill.

## Inputs

| Input | Required | Description |
|---|---|---|
| `prd_path` | yes | Path to one `.xlsx` file or a directory of `.xlsx` files. Read every sheet. |
| `target_dynamic_table` | yes | Fully qualified name, e.g. `DB.SCHEMA.SILVER_AP_INVOICES`. If only a bare name is given, resolve it (Step 2) and confirm the FQN. |
| `layer` | no | `silver` (default) or `gold`. Determines which rules are in scope. |
| `connection` | no | Snowflake connection. Defaults to the active one. |

**Supported formats:** `.xlsx` only (v1). If you get `.csv`/`.docx`/`.pdf`, say that the
format is not supported yet. Only continue if the user explicitly agrees, and then label
the plan "non-XLSX input, best effort".

## Workflow

### Step 1: Ingest the PRD
1. List the `.xlsx` files under `prd_path`. If there are none, stop and report it.
2. For each workbook, enumerate its sheets and read each one with `read` (or with
   `read_tabular` plus `sheet_name` for large sheets). Don't assume the first row is the
   header. Look for a header row and pass `skip_rows` if there is one.
3. Classify each sheet by its content, not its name: **source onboarding**, **business
   rules**, **column mapping**, or **other**. Keep every row's ID (e.g. `SRC-…`, `BR-…`) so
   you can cite it later.
4. Capture the decision/status column for each row exactly as written ("Confirmed",
   "NEEDS DECISION", "Pending — legal review"). This text decides whether a row becomes a
   change or an open question.

### Step 2: Inspect the target Dynamic Table
Run read-only queries:
```sql
SHOW DYNAMIC TABLES LIKE '<NAME>' IN SCHEMA <DB>.<SCHEMA>;   -- target_lag, warehouse, refresh_mode
SELECT GET_DDL('TABLE', '<FQN>');                              -- current query text
DESCRIBE TABLE <FQN>;                                          -- current columns
```
- Parse the current `SOURCE_SYSTEM` branches, the CASE mappings, QUALIFY/dedup logic, and
  the column list.
- If the DT doesn't exist or you lack access, **don't invent its definition**. Continue
  with a requirements-only plan and add the blocker to Open Questions.
- (Optional) Run `cortex lineage <FQN> --direction both --distance 1` to list the
  upstream Bronze tables and downstream consumers that the change affects.

### Step 3: Classify every requirement
Put each PRD row into exactly one bucket:

| Bucket | Criteria |
|---|---|
| **In scope: change DT** | Confirmed, applies to this layer, and the DT doesn't do it yet |
| **In scope: already satisfied** | The current DDL already implements it. Cite the line. |
| **Out of scope for this DT** | Explicitly assigned to another layer or object (Gold, DMF, Bronze, Phase 2) |
| **Blocked / open** | Marked undecided, pending, or depends on something missing |

### Step 4: Detect conflicts and gaps
Check every time:
- Do any rules **contradict each other**? (e.g. a USD threshold when Silver does no FX conversion)
- Does a rule's **scope** work in a UNION? (e.g. dedup that partitions on a key shared across sources)
- Is a **mapping incomplete**? (status/enum values that are not listed, with no default)
- Do **column names differ** between the rule text and the mapping sheet? (raw vs renamed)
- Does a source have **format drift or history**? (e.g. old and new code formats)
- Is a **column dropped** that some requirement still needs? (entity, region, tenant)
- Do the **refresh cadence and TARGET_LAG** work with source arrival times?
- Is any **go-live dependency** still unresolved? (legal, connector, access)

### Step 5: Assemble the plan → ⚠️ STOP
Produce the output below and present it. **Wait for the user to resolve or accept the
open questions.** Do not draft final DDL or hand off to `dynamic-tables` before that.

## Surfacing Assumptions, Not Guessing
- **Every statement has a citation**: a PRD row ID (`BR-003`), a sheet/cell reference, or the
  current DDL. If you can't cite it, it's an assumption and goes in the Assumptions list.
- **Never fill a gap silently.** An unmapped value, an unknown column name, or a missing
  default becomes an open question with a *recommended* answer. It never becomes code.
- **Keep the PRD's own uncertainty.** A row that says "NEEDS DECISION" stays open even if it
  includes a recommendation. Report the recommendation as a recommendation.
- **Separate "the PRD says" from "I infer".** Use a `Basis` column whose values are `Stated`
  or `Inferred`.
- **Each open question names an owner** (from the PRD's owner/decision columns where
  possible) and says what is **blocked** until it's answered.
- **Rank open questions** by impact: `Blocker` (wrong data or a failed build), `High` (rework
  likely), or `Low` (cosmetic or deferrable).
- **Any SQL is illustrative**, labeled `-- DRAFT, not executed`, and only covers in-scope,
  confirmed items.

## Output (always return all 8 sections, in this order)

```
# Change Plan: <target_dynamic_table>
Inputs: <files + sheets read> | DT inspected: yes/no (<reason>) | Generated: <date>

## 1. Summary
2–4 sentences: what changes, how many requirements, how many are blocked.

## 2. New Source Systems
| Request ID | System | Region | Ingestion / arrival | Go-live | Readiness | Blocking items |

## 3. Changes to the Dynamic Table
| # | Change | Type (column/mapping/filter/dedup/branch/config) | Source rows | Basis | Current state |

## 4. Out of Scope / Handled Elsewhere
| Rule ID | Rule | Where it belongs |

## 5. Conflicts & Gaps
| # | Issue | Rows involved | Risk if ignored |

## 6. Open Questions
| # | Priority | Question | Recommended answer | Owner | Blocks |

## 7. Assumptions
Numbered list. Each one says what it's based on and how to confirm it.

## 8. Next Steps
Ordered checklist: resolve blockers → draft DDL → validate → hand off to `dynamic-tables`.
(Optional appendix: illustrative SQL skeleton, labeled DRAFT.)
```

If a section has nothing in it, write "None identified". Never leave a section out.

## Stopping Points
- ⚠️ Step 2, if only a bare DT name was given: confirm the resolved FQN.
- ⚠️ Step 5: present the plan and wait for decisions on the open questions.

## Example
See `references/example-ap-invoices.md`.
