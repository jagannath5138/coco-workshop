---
name: prd-to-dynamic-table
description: "Analyze PRD-style requirement files (XLSX/CSV) and produce a structured implementation plan for a target Snowflake Dynamic Table. Use when: onboarding new sources, adding business rules, or updating schema for a Silver-layer DT. Triggers: prd, requirements, onboard source, dynamic table plan, silver layer plan."
---

# PRD to Dynamic Table Plan

Turn structured product requirement documents into an actionable implementation plan for a Snowflake Dynamic Table.

## When to Use

- A new source system is being onboarded into an existing Silver-layer Dynamic Table
- Business rules (status mappings, deduplication, normalization) are changing
- Column mappings or schema updates are required for an existing DT
- You have one or more XLSX/CSV files containing source onboarding requests, business rules, or column mappings

## Inputs

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prd_path` | Yes | Path to the PRD file(s) — supports XLSX (multiple sheets) or CSV files. If multiple CSVs, provide a glob or comma-separated list. |
| `target_dynamic_table` | Yes | Fully qualified name of the target Dynamic Table (e.g., `DB.SCHEMA.SILVER_AP_INVOICES`) |

## Workflow

### Step 1: Parse PRD Files

1. Read the PRD file(s) at `prd_path`
2. For XLSX: inspect all sheet names and identify which contain source onboarding, business rules, and column mappings
3. For CSV: infer category from filename patterns (`*source_onboarding*`, `*business_rules*`, `*column_mapping*`)
4. Extract all rows into structured data

**Expected sheet/file categories:**
- **Source Onboarding** — new systems, platforms, regions, delivery mechanisms
- **Business Rules** — status normalization, deduplication, currency handling, data quality
- **Column Mapping** — source-to-target field mappings with data types and transformations

### Step 2: Inspect Target Dynamic Table

1. Run `DESCRIBE TABLE <target_dynamic_table>` to get current schema
2. Run `SELECT DISTINCT SOURCE_SYSTEM FROM <target_dynamic_table>` to identify already-onboarded sources
3. Identify the existing UNION ALL pattern and column structure

**STOP**: Present a summary of what was parsed and what already exists. Confirm understanding before proceeding.

### Step 3: Identify Changes

For each new source or rule change, determine:
- New UNION ALL branches needed (one per new source system)
- New columns to add (nullable, to avoid breaking existing branches)
- CASE statements or transformations required (status normalization, dedup logic)
- Data quality checks to implement (DMFs, alerts — separate from the DT itself)
- Columns to drop at the Bronze-to-Silver boundary

### Step 4: Surface Ambiguities

**CRITICAL: Never guess. Always surface.**

Review the requirements for:
- Contradictions between rules (e.g., "normalize at Silver" vs. "leave raw at Silver")
- Rules marked as "OPEN QUESTION" or "NEEDS DECISION"
- Missing information (e.g., column mapping exists but no data type specified)
- Implicit assumptions (e.g., timezone handling, null semantics, historical data formats)
- Dependencies on external systems not yet available (legal approvals, FX tables, CoA mappings)

Each ambiguity must be classified:
- **BLOCKING** — cannot implement without a decision
- **NON-BLOCKING** — can proceed with a stated default, but flag for review

### Step 5: Produce Implementation Plan

Generate the structured output (see `references/output-template.md`) covering all five required sections.

**STOP**: Present the full plan for user approval before any implementation begins.

## Best Practices

### Surfacing Assumptions

1. **Never infer business logic from column names alone.** If a column is named `STATUS` but no mapping is provided, ask — do not guess the valid values.
2. **Treat "OPEN QUESTION" annotations as blockers.** If the PRD flags something as undecided, carry it forward as a blocking open question.
3. **Distinguish "confirmed" from "recommended."** If a rule says "current recommendation: X", note that this is not yet confirmed.
4. **Flag format inconsistencies.** If the same concept appears in different formats across sources (e.g., `NET30` vs `Net 30`), surface it even if the PRD doesn't mention it.
5. **Call out missing sources.** If the column mapping references a source not in the onboarding sheet (or vice versa), flag the discrepancy.
6. **Separate DT logic from guardrails.** Data quality checks, alerts, and monitoring belong outside the DT definition. Don't conflate them.

### XLSX-Specific Guidance

- Check for hidden sheets — they may contain lookup tables or reference data
- Headers may not be in row 1 — scan for the row containing column headers
- Watch for merged cells — they often indicate hierarchical groupings
- Multiple tabs may represent different pipeline stages (Bronze, Silver, Gold)

## Output

Always produce the five sections defined in `references/output-template.md`:

1. **New Source Systems** — table of sources with platform, region, delivery mechanism, status
2. **Schema Changes** — new/modified columns with types and nullability
3. **Business Rule Logic** — SQL-ready transformations (CASE, QUALIFY, etc.)
4. **Open Questions** — classified as BLOCKING or NON-BLOCKING with context
5. **Verification Plan** — queries to validate the implementation

## Example: AP Invoices Pipeline Update

**Inputs:**
- `prd_path`: `./sample_business_requirements_source_onboarding.csv`, `./sample_business_requirements_business_rules.csv`, `./sample_business_requirements_column_mapping.csv`
- `target_dynamic_table`: `COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES`

**Result summary:**

Two new sources (Baan IV, Workday) need UNION ALL branches. Status normalization requires a CASE statement mapping `POSTED→APPROVED` (Baan) and `In Review→PENDING` (Workday). Baan requires deduplication via `QUALIFY ROW_NUMBER()`. Three blocking open questions identified: payment terms normalization layer, Baan cost-center format handling, and high-value invoice threshold currency conversion timing.
