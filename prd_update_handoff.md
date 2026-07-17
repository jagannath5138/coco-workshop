# PRD-Driven Update: SILVER_AP_INVOICES

## Change Summary

Added Baan IV and Workday as source systems to `COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES`. The Dynamic Table now unions four Bronze sources (SAP, Oracle, Baan, Workday) with status normalization and Baan deduplication.

---

## Artifacts

### 1. PRD Input Files

| File | Purpose |
|------|---------|
| `sample_business_requirements_source_onboarding.csv` | New source system requests (Baan IV, Workday) with contacts, delivery mechanisms, and approval status |
| `sample_business_requirements_business_rules.csv` | 10 business rules (BR-001 through BR-010) governing status mapping, currency, dedup, data quality, and retention |
| `sample_business_requirements_column_mapping.csv` | Field-level source-to-Silver mappings for Baan (16 columns) and Workday (16 columns) with transforms and confirmation status |

### 2. Skill (Reusable)

| Path | Purpose |
|------|---------|
| `.cortex/skills/prd-to-dynamic-table/SKILL.md` | Project skill that parses PRD files and produces a structured implementation plan for any Silver-layer Dynamic Table |
| `.cortex/skills/prd-to-dynamic-table/references/output-template.md` | Enforces consistent 5-section output: New Sources, Schema Changes, Business Rule Logic, Open Questions, Verification Plan |

**To invoke:** `/prd-to-dynamic-table` with inputs `prd_path` and `target_dynamic_table`.

### 3. Executed DDL

**Target:** `COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES`

Key changes from the original two-source DT:

- Added `UNION ALL` branch for Baan with `QUALIFY ROW_NUMBER()` dedup on `BAN_INVOICE_REF`
- Added `UNION ALL` branch for Workday with status CASE mapping (`Approved` → `APPROVED`, `In Review` → `PENDING`)
- Baan status CASE: `POSTED` → `APPROVED`, `APPROVED` → `APPROVED`, `PENDING` → `PENDING`
- Dropped system-specific columns: `BAN_COMPANY`, `WD_TENANT_ID`
- System-specific columns (`SAP_COMPANY_CODE`, `SAP_DOCUMENT_TYPE`, `ORACLE_ORG_ID`, `ORACLE_SOURCE`) are NULL for new sources

### 4. Validation Queries

```sql
-- Row counts by source (expect SAP=15, Oracle=15, Baan=10, Workday=10)
SELECT SOURCE_SYSTEM, COUNT(*) AS ROW_COUNT
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM ORDER BY 1;

-- No duplicate invoice numbers in Baan (expect 0 rows)
SELECT INVOICE_NUMBER, COUNT(*)
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
WHERE SOURCE_SYSTEM = 'BAAN'
GROUP BY 1 HAVING COUNT(*) > 1;

-- Unmapped status values (expect only Oracle VALIDATED as known tech debt)
SELECT SOURCE_SYSTEM, STATUS, COUNT(*)
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
WHERE STATUS NOT IN ('APPROVED', 'PENDING') OR STATUS IS NULL
GROUP BY 1, 2;

-- NULLs in required columns (expect all zeros)
SELECT SOURCE_SYSTEM,
       COUNT_IF(INVOICE_ID IS NULL) AS NULL_ID,
       COUNT_IF(INVOICE_NUMBER IS NULL) AS NULL_NUM,
       COUNT_IF(VENDOR_ID IS NULL) AS NULL_VENDOR,
       COUNT_IF(INVOICE_AMOUNT IS NULL) AS NULL_AMT
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
GROUP BY 1;

-- System-specific columns NULL for Baan/Workday (expect all zeros)
SELECT SOURCE_SYSTEM,
       COUNT_IF(SAP_COMPANY_CODE IS NOT NULL) AS HAS_SAP,
       COUNT_IF(ORACLE_ORG_ID IS NOT NULL) AS HAS_ORA
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
WHERE SOURCE_SYSTEM IN ('BAAN', 'WORKDAY')
GROUP BY 1;
```

---

## Open Items for Engineering Review

| # | Item | Status | Owner |
|---|------|--------|-------|
| 1 | Oracle `VALIDATED` status unmapped (11 rows) — should map to `APPROVED` per BR-001 | Pre-existing tech debt | Engineering |
| 2 | Payment terms normalization (Silver vs Gold) | Undecided (BR-005) | Sarah Chen / David Kim |
| 3 | Baan cost-center format change (`BC-XX` → `BC-XXX`) — both pass through | Assumed pass-through | Engineering |
| 4 | Baan inline dedup vs intermediate view for scale | Assumed inline is fine at current volume | Engineering |
| 5 | High-value invoice threshold ($500K) — currency conversion timing | Deferred to DMF guardrail | Finance Controller |

---

## How to Reuse the PRD Evaluator Skill

1. Place new PRD files (XLSX or CSV) in the repo
2. Invoke `/prd-to-dynamic-table`
3. Provide:
   - `prd_path`: path to the requirement file(s)
   - `target_dynamic_table`: fully qualified DT name (e.g., `DB.SCHEMA.SILVER_*`)
4. The skill parses requirements, inspects the current DT, and returns a structured plan with open questions surfaced — never guessed
