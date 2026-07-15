# Knowledge Transfer (KT) — VDV Pipeline
## For AMS Support Team

**Project:** EVS - Vehicle Data Validation (VDV)  
**Date:** 2026-07-15  
**Prepared by:** FLEMUS  
**Environment:** Prod → DB_FCDS.ITDMX_VDV | Dev → MDS_MEXICO.EVS_VDV  

---

## 1. What is VDV?

An automated pipeline that classifies vehicles by their VIN (Vehicle Identification Number). When the Ordering API receives a new VIN, it needs to know:
- **Make/Model/Year**
- **Fuel type** (Gasoline, Diesel, Electric, Hybrid)
- **Engine classification** (engine_class_lvl_id)

VDV resolves this by querying multiple sources in cascade until it finds the answer.

---

## 2. General Flow (how it works)

```
┌─────────────┐     ┌─────────────────────┐     ┌──────────────────────┐
│ Ordering API│────▶│ INSERT VIN into      │────▶│ Serverless Task      │
│  (request)  │     │ VDV_REQUEST_STAGE_HT │     │ runs SP every 5 min  │
└─────────────┘     └─────────────────────┘     └──────────────────────┘
                                                          │
                              ┌────────────────────────────┤
                              ▼                            ▼
                    ┌──────────────────┐       ┌──────────────────┐
                    │ 1. History (RAW) │       │ 2. JATO Catalog  │
                    │   (~350K VINs)   │       │  (~52K models)   │
                    └──────────────────┘       └──────────────────┘
                              │                            │
                              ▼                            ▼
                    ┌──────────────────┐       ┌──────────────────┐
                    │ 3. ML Model      │       │ 4. Suffix Rules  │
                    │  (scikit-learn)  │       │  (BMW E=PHEV)    │
                    └──────────────────┘       └──────────────────┘
                              │
                              ▼
                    ┌──────────────────────────────────────────┐
                    │ CONSISTENCY ENFORCEMENT                   │
                    │ (engine_class always = fuel equivalence)  │
                    └──────────────────────────────────────────┘
                              │
                              ▼
                    ┌──────────────────────────────────────────┐
                    │ NULL-SAFETY GUARANTEE                     │
                    │ (make/model/year never NULL)              │
                    └──────────────────────────────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ API reads result │
                    │ VDV_VW_REQUEST_  │
                    │ STAGE_API (view) │
                    └──────────────────┘
```

---

## 3. Key Objects

### Tables

| Object | Type | Description |
|--------|------|-------------|
| `VDV_REQUEST_STAGE_HT` | Hybrid Table | Main table. Each row = 1 VIN classification request |
| `VDV_VIN_BASE_RAW` | Table | Historical fleet data (~350K already-classified VINs) |
| `QUERYJATO` | Table | JATO vehicle catalog (~52K records) |
| `VDV_MAKE_ALIAS_MAP` | Table (config) | Make code mapping ML → JATO (e.g., MERZ→MERC) |
| `VDV_MODEL_ALIAS_MAP` | Table (config) | Model name mapping ML → JATO (e.g., X TRAIL→X-TRAIL) |
| `VDV_MODEL_ELECTRIFICATION_RULES` | Table (config) | Rules to detect electric type by model name (e.g., BMW I%→BEV) |
| `VDV_ENRICH_PROGRESS` | Table | Execution log for each SP run |

### Views

| Object | Description | Consumer |
|--------|-------------|----------|
| `VDV_VW_REQUEST_STAGE_API` | Main deduplicated view. UNION of stage_data + raw_data | Ordering API |
| `VDV_VW_VIN_BASE_LOOKUP` | 1 record per VIN_BASE, prioritizes SUCCESS | API (fast lookup) |
| `VDV_VW_JATO_CATALOG_MMY` | Normalized JATO catalog with make aliases | SP (internal) |
| `VDV_VW_REQUEST_STAGE_SERVING` | Simple view of HT without deduplication | Reporting |

### Stored Procedures

| Object | Description |
|--------|-------------|
| `VDV_SP_ENRICH_STAGE()` | Main SP. Processes pending VINs in batches of 500. Includes: RAW lookup → ML → JATO → Rules → Consistency → Null-safety |

### Tasks

| Object | Schedule | Normal State |
|--------|----------|--------------|
| `VDV_TASK_ENRICH_STAGE` | Every 5 min (serverless) | **STARTED** |

### UDFs

| Object | Description |
|--------|-------------|
| `VDV_DECODE_VIN(VIN)` | Python UDF using ML model to predict MAKE/MODEL/YEAR from a VIN |

---

## 4. State Machine

### PROCESS_STATUS (internal SP lifecycle)

```
NEW → PROCESSING → PROCESSED (success)
                 → ERROR (technical failure)
```

### STATUS (result visible to API/business)

| STATUS | STATUS_ID | Meaning | When Assigned |
|--------|-----------|---------|---------------|
| PENDING | 1 | Just inserted, awaiting processing | On INSERT |
| SUCCESS | 2 | Classification complete and reliable | SRC=JATO/RULE (always) or ML score >= 0.70 |
| REJECTED | 3 | Could not classify with confidence | ML score < 0.70 and no JATO/Rule match |
| ERROR | 4 | Technical failure | Invalid VIN (len!=17) or UDF crash |
| FINAL | — | Manually approved (legacy) | Historical records only |

### SRC (resolution source)

| SRC | Meaning | Reliability |
|-----|---------|-------------|
| HISTORY | Found in VDV_VIN_BASE_RAW | High (actual historical data) |
| JATO | Matched JATO catalog (exact, alias, or prefix) | High (validated external source) |
| RULE | Detected by electrification rule (model name pattern) | Medium-High |
| MODEL | ML model prediction only | Depends on score |
| JATO_LEGACY | Matched legacy specs table | Medium |

---

## 5. Fuel Equivalence Table (CRITICAL)

Every fuel combination MUST exist in this table. If it doesn't, it's a **bug**.

| fuel_ctgy | fuel_subctgy | engine_class_lvl_id | prim_fuel_typ_cd | secondary_fuel_typ_cd | Type |
|---|---|---|---|---|---|
| G | L | 8 | 3 | NULL | Gasoline |
| D | D | 8 | 1 | NULL | Diesel |
| E | E | 5 | 2 | NULL | BEV (battery electric) |
| E | R | 7 | 2 | 3 | EREV (extended range) |
| I | M | 1 | 3 | 2 | MHEV (mild hybrid) |
| I | H | 3 | 3 | 2 | HEV (full hybrid) |
| I | P | 4 | 3 | 2 | PHEV (plug-in hybrid) |
| N | NULL | 9 | 15 | 15 | Non-fuel (electric forklifts, etc.) |
| A | A | 8 | 11 | NULL | Methanol |
| B | B | 8 | 1 | NULL | Biodiesel |
| C | N | 8 | 4 | NULL | Compressed natural gas |
| F | NULL | 8 | 3 | 12 | Flexible (gas/ethanol) |
| J | C | 6 | 14 | NULL | Fuel cell (hydrogen) |

**Golden rule:** `engine_class_lvl_id` is ALWAYS derived from `fuel_ctgy + fuel_subctgy`. Never from any other source.

---

## 6. System Guarantees

The SP has two blocks at the end of each iteration that MUST NEVER be removed:

### CONSISTENCY ENFORCEMENT
Forces `engine_class_lvl_id`, `prim_fuel_typ_cd`, and `secondary_fuel_typ_cd` to always be consistent with `fuel_ctgy/fuel_subctgy` per the equivalence table.

### NULL-SAFETY GUARANTEE
Ensures there are NEVER NULLs in: `make`, `make_full`, `model`, `model_full`, `year`, `engine_class_lvl_id`, `fuel_ctgy`, `fuel_subctgy`, `prim_fuel_typ_cd`.

Defaults when data is unavailable:
- make/make_full = `UNKN` / `UNKNOWN`
- model/model_full = `UNKN` / `UNKNOWN`
- year = `0`
- fuel = G/L/8/3/NULL (gasoline ICE by default)

---

## 7. Monitoring Procedures

### 7.1 Daily Health Check
```sql
-- Is the task running?
SHOW TASKS LIKE 'VDV_TASK_ENRICH_STAGE' IN SCHEMA DB_FCDS.ITDMX_VDV;
-- state must be 'started'

-- Are there accumulated pending records?
SELECT COUNT(*) AS PENDING 
FROM DB_FCDS.ITDMX_VDV.VDV_REQUEST_STAGE_HT 
WHERE PROCESS_STATUS = 'NEW' AND ACTIVE = TRUE;
-- Normal: < 50. Alert if > 500

-- Last successful execution?
SELECT * FROM DB_FCDS.ITDMX_VDV.VDV_ENRICH_PROGRESS 
ORDER BY STARTED_AT DESC LIMIT 1;
-- status should be 'DONE'
```

### 7.2 Integrity Verification
```sql
-- Any NULLs? (must be 0)
SELECT COUNT(*) FROM DB_FCDS.ITDMX_VDV.VDV_REQUEST_STAGE_HT 
WHERE ACTIVE = TRUE AND (MAKE_FULL IS NULL OR MODEL_FULL IS NULL OR YEAR IS NULL);

-- Any fuel descriptions instead of codes? (must be 0)
SELECT COUNT(*) FROM DB_FCDS.ITDMX_VDV.VDV_REQUEST_STAGE_HT 
WHERE ACTIVE = TRUE AND FUEL_CTGY IS NOT NULL AND LENGTH(FUEL_CTGY) > 1;

-- Any invalid combinations? (must return 0 rows)
WITH valid AS (
  SELECT * FROM (VALUES ('G','L',8,3,NULL),('D','D',8,1,NULL),('E','E',5,2,NULL),
    ('E','R',7,2,3),('I','M',1,3,2),('I','H',3,3,2),('I','P',4,3,2),('N',NULL,9,15,15)
  ) AS t(fc,fs,ecl,pf,sf)
)
SELECT FUEL_CTGY, FUEL_SUBCTGY, ENGINE_CLASS_LVL_ID, PRIM_FUEL_TYP_CD, SECONDARY_FUEL_TYP_CD, COUNT(*)
FROM DB_FCDS.ITDMX_VDV.VDV_REQUEST_STAGE_HT a
LEFT JOIN valid v ON a.FUEL_CTGY=v.fc AND (a.FUEL_SUBCTGY=v.fs OR (a.FUEL_SUBCTGY IS NULL AND v.fs IS NULL))
  AND a.ENGINE_CLASS_LVL_ID=v.ecl AND a.PRIM_FUEL_TYP_CD=v.pf 
  AND (a.SECONDARY_FUEL_TYP_CD=v.sf OR (a.SECONDARY_FUEL_TYP_CD IS NULL AND v.sf IS NULL))
WHERE a.STATUS='SUCCESS' AND a.ACTIVE=TRUE AND v.fc IS NULL
GROUP BY 1,2,3,4,5;
```

### 7.3 Status Distribution (reporting)
```sql
SELECT STATUS, SRC, COUNT(*) AS CNT
FROM DB_FCDS.ITDMX_VDV.VDV_REQUEST_STAGE_HT
WHERE ACTIVE = TRUE
GROUP BY 1, 2 ORDER BY 3 DESC;
```

---

## 8. Incident Runbooks

### 8.1 Task stopped / not processing

**Symptom:** Pending records accumulate, API doesn't classify new VINs.

**Diagnosis:**
```sql
SHOW TASKS LIKE 'VDV_TASK_ENRICH_STAGE' IN SCHEMA DB_FCDS.ITDMX_VDV;
-- If state = 'suspended':
```

**Fix:**
```sql
ALTER TASK DB_FCDS.ITDMX_VDV.VDV_TASK_ENRICH_STAGE RESUME;
```

### 8.2 Records stuck in PROCESSING

**Symptom:** `PROCESS_STATUS = 'PROCESSING'` for more than 10 minutes.

**Diagnosis:**
```sql
SELECT COUNT(*) FROM DB_FCDS.ITDMX_VDV.VDV_REQUEST_STAGE_HT 
WHERE PROCESS_STATUS = 'PROCESSING' 
AND UPDATED_AT < DATEADD('minute', -10, CURRENT_TIMESTAMP());
```

**Fix:** The SP already has auto-recovery (resets to NEW after 5 min). If it persists:
```sql
UPDATE DB_FCDS.ITDMX_VDV.VDV_REQUEST_STAGE_HT
SET PROCESS_STATUS = 'NEW', UPDATED_AT = CURRENT_TIMESTAMP()
WHERE PROCESS_STATUS = 'PROCESSING';
```

### 8.3 API returns NULLs in make/model/year

**Probable cause:** An external flow inserted data without going through the SP.

**Immediate fix:**
```sql
UPDATE DB_FCDS.ITDMX_VDV.VDV_REQUEST_STAGE_HT
SET MAKE = COALESCE(MAKE, 'UNKN'), MAKE_FULL = COALESCE(MAKE_FULL, MAKE, 'UNKNOWN'),
    MODEL = COALESCE(MODEL, 'UNKN'), MODEL_FULL = COALESCE(MODEL_FULL, MODEL, 'UNKNOWN'),
    YEAR = COALESCE(YEAR, 0), UPDATED_AT = CURRENT_TIMESTAMP()
WHERE ACTIVE = TRUE AND (MAKE_FULL IS NULL OR MODEL_FULL IS NULL OR YEAR IS NULL);
```

**Permanent fix:** The `VDV_TASK_NORMALIZE_DATA` task (if deployed) runs hourly and auto-corrects this. If not deployed, escalate to development.

### 8.4 Invalid fuel combinations (engine_class doesn't match)

**Probable cause:** Historical RAW data with incorrect engine_class overrode the JATO value.

**Fix:**
```sql
-- Run DATAFIX_CONSISTENCY_ENFORCEMENT.sql
-- Or manually:
UPDATE DB_FCDS.ITDMX_VDV.VDV_REQUEST_STAGE_HT
SET ENGINE_CLASS_LVL_ID = CASE 
    WHEN FUEL_CTGY = 'I' AND FUEL_SUBCTGY = 'M' THEN 1
    WHEN FUEL_CTGY = 'I' AND FUEL_SUBCTGY = 'H' THEN 3
    WHEN FUEL_CTGY = 'I' AND FUEL_SUBCTGY = 'P' THEN 4
    WHEN FUEL_CTGY = 'E' AND FUEL_SUBCTGY = 'E' THEN 5
    WHEN FUEL_CTGY = 'E' AND FUEL_SUBCTGY = 'R' THEN 7
    WHEN FUEL_CTGY = 'N' THEN 9
    ELSE 8 END,
  updated_at = CURRENT_TIMESTAMP()
WHERE process_status IN ('PROCESSED','ERROR') AND FUEL_CTGY IS NOT NULL;
```

### 8.5 ML Model returns very low scores (many REJECTED)

**Symptom:** High % of REJECTED for new VINs.

**Diagnosis:**
```sql
SELECT MAKE_FULL, MODEL_FULL, YEAR, AVG(MODEL_SCORE_TOP1) AS AVG_SCORE, COUNT(*) AS CNT
FROM DB_FCDS.ITDMX_VDV.VDV_REQUEST_STAGE_HT
WHERE STATUS = 'REJECTED' AND CREATED_AT > DATEADD('day', -1, CURRENT_TIMESTAMP())
GROUP BY 1, 2, 3 ORDER BY 5 DESC LIMIT 20;
```

**Actions:**
1. If they are new models → Update `QUERYJATO` with more recent JATO data
2. If unrecognized makes → Add alias in `VDV_MAKE_ALIAS_MAP`
3. If model predicted with trim/version → Add in `VDV_MODEL_ALIAS_MAP`
4. If it's a new 100% electric brand → Add rule in `VDV_MODEL_ELECTRIFICATION_RULES`

### 8.6 Invalid VIN (length != 17)

**This is expected behavior.** The system marks as ERROR, assigns defaults (UNKN/UNKNOWN/0) and continues. No action required.

---

## 9. How to Add New Rules (no code changes needed)

### Add a new make alias
```sql
INSERT INTO DB_FCDS.ITDMX_VDV.VDV_MAKE_ALIAS_MAP (MAKE_ML, MAKE_JATO) 
VALUES ('NEWX', 'NEW');
-- NEWX = code the ML uses, NEW = code in JATO
```

### Add a model alias
```sql
INSERT INTO DB_FCDS.ITDMX_VDV.VDV_MODEL_ALIAS_MAP (MAKE, MODEL_ML, MODEL_JATO) 
VALUES ('TOYO', 'COROLLA LE HEV', 'COROLLA');
-- SP will now map "COROLLA LE HEV" → "COROLLA" when searching JATO
```

### Add an electrification rule
```sql
INSERT INTO DB_FCDS.ITDMX_VDV.VDV_MODEL_ELECTRIFICATION_RULES 
(RULE_TYPE, MAKE_PATTERN, MODEL_PATTERN, ENGINE_CLASS_LVL_ID, FUEL_CTGY, FUEL_SUBCTGY, PRIM_FUEL_TYP_CD, SECONDARY_FUEL_TYP_CD, PRIORITY, DESCRIPTION)
VALUES ('PREFIX', 'RIVE', '%', 5, 'E', 'E', 2, NULL, 5, 'Rivian models are all BEV');
-- RULE_TYPE: SUFFIX (ends with), PREFIX (starts with), CONTAINS (contains)
-- PRIORITY: lower number = higher priority (applied first)
```

### Update JATO catalog
1. Get new `QueryJato.csv` from provider
2. Upload to stage: `@DB_FCDS.ITDMX_VDV.STG_JATO_LOAD`
3. Execute:
```sql
TRUNCATE TABLE DB_FCDS.ITDMX_VDV.QUERYJATO;
COPY INTO DB_FCDS.ITDMX_VDV.QUERYJATO FROM @STG_JATO_LOAD/QueryJato.csv
FILE_FORMAT = (TYPE='CSV' SKIP_HEADER=1 FIELD_DELIMITER='\t' FIELD_OPTIONALLY_ENCLOSED_BY='"' ENCODING='UTF8')
ON_ERROR = 'CONTINUE';
```
The `VDV_VW_JATO_CATALOG_MMY` view updates automatically.

---

## 10. VIN Decoding (reference)

Character 10 of the VIN indicates the model year:

| Char | Year | | Char | Year | | Char | Year |
|---|---|---|---|---|---|---|---|
| 9 | 2009 | | L | 2020 | | 1 | 2031 |
| A | 2010 | | M | 2021 | | 2 | 2032 |
| B | 2011 | | N | 2022 | | 3 | 2033 |
| C | 2012 | | P | 2023 | | 4 | 2034 |
| D | 2013 | | R | 2024 | | 5 | 2035 |
| E | 2014 | | S | 2025 | | 6 | 2036 |
| F | 2015 | | T | 2026 | | 7 | 2037 |
| G | 2016 | | V | 2027 | | 8 | 2038 |
| H | 2017 | | W | 2028 | | | |
| J | 2018 | | X | 2029 | | | |
| K | 2019 | | Y | 2030 | | | |

If the character doesn't match → year = 0 (default).

---

## 11. Contacts and Escalation

| Level | Who | When |
|-------|-----|------|
| L1 - Monitoring | AMS | Daily health checks, task resume |
| L2 - Configuration | AMS / Development | Add aliases, rules, update JATO |
| L3 - Code | FLEMUS | SP changes, UDF, ML model |

---

## 12. SLA and Thresholds

| Metric | Normal | Warning | Critical |
|--------|--------|---------|----------|
| Pending (NEW) | < 50 | > 500 | > 5000 |
| Task state | started | suspended > 30 min | suspended > 2 hrs |
| NULLs in critical fields | 0 | > 0 | > 100 |
| Invalid fuel combos | 0 | > 0 | > 50 |
| PROCESSING stuck | 0 | > 0 for > 10 min | > 100 |
