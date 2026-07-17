# Output Template

Every invocation of this skill MUST produce all five sections below. If a section has no findings, state "None identified" — never omit a section.

---

## 1. New Source Systems

| Source System | Platform | Region | Delivery Mechanism | Cadence | Status | Blocking Issues |
|---------------|----------|--------|--------------------|---------|--------|-----------------|
| _name_ | _ERP/platform_ | _geography_ | _CDC/batch/API_ | _real-time/hourly/nightly_ | _Approved/Pending/Blocked_ | _legal, access, etc._ |

For each source, note:
- Contact person and approval status
- Any infrastructure prerequisites (connectors, landing zones, DPAs)
- Expected go-live sequencing relative to other sources

---

## 2. Schema Changes

### New Columns

| Column Name | Data Type | Nullable | Source(s) | Notes |
|-------------|-----------|----------|-----------|-------|
| _COLUMN_ | _VARCHAR/NUMBER/etc._ | _YES/NO_ | _which sources populate it_ | _default for sources that don't have it_ |

### Dropped Columns (Bronze → Silver boundary)

| Column Name | Source System | Reason |
|-------------|--------------|--------|
| _COLUMN_ | _source_ | _system-specific, not needed at Silver_ |

### Modified Columns

| Column Name | Change | Rationale |
|-------------|--------|-----------|
| _COLUMN_ | _type change / rename / new mapping_ | _why_ |

---

## 3. Business Rule Logic

For each rule, provide:

### Rule: _[Rule ID] — [Short Name]_

- **Applies to**: _which source(s)_
- **Layer**: _Silver DT / DMF / Gold_
- **Logic** (SQL-ready):

```sql
-- Example:
CASE
  WHEN SOURCE_SYSTEM = 'BAAN' AND status = 'POSTED' THEN 'APPROVED'
  WHEN SOURCE_SYSTEM = 'WORKDAY' AND status = 'In Review' THEN 'PENDING'
  ...
END AS STATUS
```

- **Confirmed by**: _person and date, or "UNCONFIRMED"_

---

## 4. Open Questions

### BLOCKING (cannot implement without resolution)

| # | Question | Context | Owner | Deadline |
|---|----------|---------|-------|----------|
| 1 | _question_ | _why it matters_ | _who decides_ | _when_ |

### NON-BLOCKING (can proceed with stated default)

| # | Question | Default Assumption | Risk if Wrong |
|---|----------|--------------------|---------------|
| 1 | _question_ | _what we'll do if no answer_ | _impact_ |

---

## 5. Verification Plan

### Row Counts

```sql
-- Expected total after adding new sources
SELECT SOURCE_SYSTEM, COUNT(*) AS row_count
FROM <target_dynamic_table>
GROUP BY SOURCE_SYSTEM;
```

### Schema Validation

```sql
DESCRIBE TABLE <target_dynamic_table>;
-- Confirm new columns exist with correct types
```

### Business Rule Spot Checks

```sql
-- Verify no unmapped status values
SELECT DISTINCT SOURCE_SYSTEM, STATUS
FROM <target_dynamic_table>
WHERE STATUS NOT IN ('APPROVED', 'PENDING');
-- Should return 0 rows
```

### Deduplication Verification

```sql
-- Confirm no duplicate invoice numbers per source
SELECT SOURCE_SYSTEM, INVOICE_NUMBER, COUNT(*)
FROM <target_dynamic_table>
GROUP BY 1, 2
HAVING COUNT(*) > 1;
-- Should return 0 rows
```

### Data Quality

```sql
-- Check for NULLs in required columns
SELECT SOURCE_SYSTEM,
       COUNT_IF(INVOICE_ID IS NULL) AS null_invoice_id,
       COUNT_IF(INVOICE_NUMBER IS NULL) AS null_invoice_number,
       COUNT_IF(VENDOR_ID IS NULL) AS null_vendor_id
FROM <target_dynamic_table>
GROUP BY 1;
```
