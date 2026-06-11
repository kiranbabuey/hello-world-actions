# Data Lookup Transformation Pipeline - Complete Documentation

**Document Version:** 1.0  
**Last Updated:** June 11, 2026  
**Audience:** Business & Technical Teams  
**Purpose:** Cross-border SKU classification and HS Code assignment system

---

## Executive Summary

This Databricks notebook implements an automated **SKU Classification and HS Code Lookup Pipeline** for cross-border tariff and regulatory compliance. It transforms raw forecast and product data from multiple sources into a standardized classification dataset with assigned Harmonized System (HS) Codes and tariff codes (CFL).

### Key Objectives
- ✅ Classify SKUs across international borders (origin-destination lanes)
- ✅ Assign Harmonized System (HS) codes based on product hierarchy and attributes
- ✅ Determine tariff classification flags (CFL) per regulatory requirements
- ✅ Track processing status through a multi-stage gate system (Gates 1-5)
- ✅ Prevent duplicate processing via incremental filtering

### Output Deliverables
- **final_output_lookup.csv** - Complete classified dataset (all SKUs)
- **final_output_lookup_one.csv** - Filtered dataset (SKUs with assigned HS codes only)
- **SKU_Tracking_Report_*.xlsx** - Audit trail of SKU journey through pipeline

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Input Data Sources](#input-data-sources)
3. [Processing Pipeline - 12 Steps](#processing-pipeline---12-steps)
4. [Data Quality & Audit Trail](#data-quality--audit-trail)
5. [Output Specifications](#output-specifications)
6. [Error Handling](#error-handling)
7. [Business Use Cases](#business-use-cases)
8. [Integration Points](#integration-points)
9. [Technical Glossary](#technical-glossary)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    INPUT DATA SOURCES (Multiple Formats)                 │
├─────────────────────────────────────────────────────────────────────────┤
│ • Cross Border DSR - SKU Detail (Monthly Forecast)                      │
│ • ESB Product Catalogs (Beauty, Home Care, Ice Cream, Nutrition, PC)    │
│ • RTVA Import/Export Views (Material descriptions & origins)            │
│ • Reference HS Code Mapping (Brazil NCM → HS Code conversion)           │
│ • GTS Origin Uploads (Existing processed SKUs - incremental filter)     │
│ • Gate 4 & 5 Completed (Previous classification results)                │
│ • Waves Data (Country-based wave allocation)                            │
│ • Country Scope Configuration (In-scope vs out-of-scope markets)        │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                         PROCESSING PIPELINE                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                         OUTPUT GENERATION                                │
├─────────────────────────────────────────────────────────────────────────┤
│ • Write to Azure Blob Storage (_processed & _archived folders)          │
│ • Upload classified data to GTS database via REST API                   │
│ • Generate SKU tracking report (Excel/CSV) for audit trail              │
│ • Send success/failure notification emails                              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Input Data Sources

| File Name | Format | Purpose | Key Columns |
|-----------|--------|---------|-------------|
| Cross Border DSR-SKU detail_MM-YYYY.xlsx | Excel | Monthly forecast | SKU, Lane, Report quantities |
| Beauty_*.xlsx | Excel | ESB catalog | DU MRDR, DU Description, Business Group |
| Home Care_*.xlsx | Excel | ESB catalog | (same structure) |
| Ice Cream_*.xlsx | Excel | ESB catalog | (same structure) |
| Nutrition_*.xlsx | Excel | ESB catalog | (same structure) |
| Personal Care_*.xlsx | Excel | ESB catalog | (same structure) |
| CROSS BORDER - IMPORT VIEW.csv | CSV | RTVA imports | MaterialNumber, SourceCountry, DestinationCountry |
| CROSS BORDER - EXPORT VIEW.csv | CSV | RTVA exports | (same structure) |
| REFERENCE_HS_CODE.xlsx | Excel | BR & generic HS mapping | SKU, NCM, CFL, Descrição |
| origin_gts_uploads.csv | CSV (via API) | GTS reference | SKU, HS_CODE |
| Gate 4 Completed_DD-MM-YYYY.xlsx | Excel | Previous classifications | SKU, Origin Country, Destination Country, HS Code |
| Gate 5 Completed_DD-MM-YYYY.xlsx | Excel | Previous classifications | (same structure) |
| waves.csv | CSV | Wave allocation | COUNTRY, WAVE |

---

## Processing Pipeline - 12 Steps

### STEP 1: File Discovery & Incremental Locking

**Purpose:** Ensure files are processed exactly once by locking them immediately

**Key Actions:**
- Scan input folder for unprocessed Excel files (.xlsx)
- SKIP files in _archived folder (already processed - source of truth)
- Convert Excel → CSV for processing
- Move original file to _archived/{date}/ subfolder (prevents re-processing)

**Business Impact:**
- ✅ No duplicate processing
- ✅ Transparent audit trail
- ✅ Safe failure handling

---

### STEP 2: Forecast Transformation

**Purpose:** Prepare forecast data and establish base SKU-Lane structure

**Input:** `Cross Border DSR-SKU detail.csv`

**Key Transformations:**
1. Extract Report Date from filename (MM-YYYY format)
2. Remove unnecessary columns (source plants, actuals, deviations)
3. Parse "Lane" field → Origin & Destination Countries
4. Rename SKU Details → SKU
5. Deduplicate by latest report date per lane
6. Create composite keys (SKU-Sales Country, Origin-Destination-SKU)
7. Remove aggregate rows (where Origin = "TOTAL")

**Output Schema:**
| Column | Type | Example |
|--------|------|---------|
| SKU | string | 34567890 |
| Origin Country | string | BRAZIL |
| Destination Country | string | USA |
| Report Date | date | 2026-03-31 |

---

### STEP 3: Incremental Filter (GTS Check)

**Purpose:** Prevent reprocessing of already-classified SKUs

**Process:**
1. Download GTS file (origin_gts_uploads) via REST API
2. Build SKU → HS_CODE mapping (strip leading zeros)
3. Apply filtering logic:
   - SKIP if SKU in GTS AND HS_CODE value exists
   - ALLOW if SKU not in GTS OR HS_CODE is empty
4. Track skipped SKUs for audit

**Example:**
```
Forecast: SKU-A, SKU-B, SKU-C (3 SKUs)
GTS has:  SKU-A (HS=123456), SKU-B (HS=NULL)
Result:   SKIP SKU-A, ALLOW SKU-B and SKU-C (2 to process)
```

**Business Impact:**
- ✅ Avoid redundant processing
- ✅ Incremental pipeline (only new SKUs processed)
- ⚠️ If GTS fails: Full dataset processed

---

### STEP 4: ESB Data Consolidation

**Purpose:** Enrich SKUs with product hierarchy and marketing data

**Inputs:** 5 ESB product catalogs (Beauty, Home Care, Ice Cream, Nutrition, Personal Care)

**Key Transformations:**
1. Consolidate all 5 catalogs into single dataset
2. Extract: DU MRDR (SKU), Description, Business Group, Category, Segment, Product Line
3. Explode country field (comma-separated → separate rows)
4. Build Product Hierarchy: [BG]-[Category]-[SubCat]-[Segment]-[ProdLine]
5. Deduplicate by SKU, Country, Hierarchy

**Output:** Enriched product data with hierarchy and marketing info

---

### STEP 5: RTVA Cross-Border Data Join

**Purpose:** Add material descriptions and validate cross-border movements

**Inputs:** 
- CROSS BORDER - IMPORT VIEW.csv
- CROSS BORDER - EXPORT VIEW.csv

**Process:**
1. Load both import and export views
2. Union into single dataset
3. Clean material numbers (strip leading zeros)
4. Join with Forecast+ESB on SKU + Origin + Destination
5. Enrich with RTVA material descriptions

**Result:** Complete SKU data with cross-border compliance info

---

### STEP 6: HS Code Reference Mapping

**Purpose:** Build lookup tables for HS code classification

**Inputs:**
- Brazil NCM → HS Code mapping
- Generic product → HS Code mapping

**Key Features:**
- Parse CFL field (Format: "95/10" = Origin/Destination)
- Calculate HS CODE from NCM (special handling for Brazil)
- Create multi-level composite keys (Lane, SKU, Origin-Country, Dest-Country)
- Deduplicate by lane

---

### STEP 7: HS Code Calculation (4-Step Lookup Hierarchy)

**Purpose:** Assign HS codes using intelligent hierarchy matching

**Priority Order:**

```
TIER 1: LANE-SPECIFIC (Most Specific)
├─ Condition: (SKU + Origin Country + Destination Country)
├─ Match Rate: ~5-15%
└─ Example: SKU-A from BRAZIL to USA → Exact lane match

TIER 2: SKU-ONLY
├─ Condition: (SKU) regardless of country
├─ Match Rate: ~20-30%
└─ Example: SKU-A → Same product, consistent HS globally

TIER 3: PRODUCT HIERARCHY
├─ Condition: (Product Hierarchy string exact match)
├─ Match Rate: ~25-35%
└─ Example: "Beauty-Skin Care-Face Wash-Daily-Premium" → HS code

TIER 4: PRODUCT DESCRIPTION (Fallback)
├─ Condition: (Product Description exact match)
├─ Match Rate: ~10-20%
└─ Example: Full product name → HS code from RTVA

RESULT: HS CODE = COALESCE(Tier1, Tier2, Tier3, Tier4, NULL)
```

**Wave-Based Adjustment:**
- Wave ≤2 countries: Use FULL HS Code
- Wave >2 countries: Use FIRST 6 DIGITS ONLY

---

### STEP 8: Gate Assignment (Multi-Gate Classification)

**Purpose:** Classify SKUs based on HS code lookup source

**Gate Logic:**

| Gate | Condition | CFL Value | Meaning |
|------|-----------|-----------|---------|
| 1 | Brazil export rule (CFL=95) | 95 | Special Brazil customs |
| 2 | Product hierarchy matched | 90 | Known product family |
| 3 | Product description matched | 88 | Description-based match |
| 4 | HS code exists, no hierarchy/desc | 86 | SKU only, incomplete data |
| 5 | Wave countries, SKU match only | (none) | Alternative processing |
| Unclassified | No HS code found | (empty) | Needs manual review |

---

### STEP 9: Wave Assignment

**Purpose:** Determine delivery waves (time-based market entry strategy)

**Process:**
1. Load Waves table (Country → Wave number mapping)
2. Join with forecast on Origin & Destination countries
3. Calculate final wave = MIN(origin_wave, dest_wave)
4. Flag is_wave = TRUE if any country is Wave ≤2

**Business Context:**
- Wave 1-2: Key markets (full tariff detail needed)
- Wave 3+: Standard markets (6-digit code sufficient)

---

### STEP 10: CFL Code Assignment

**Purpose:** Assign final tariff classification flag based on HS code presence

**Logic:**
```
IF hs_code is NOT NULL:
    cfl = carry forward HS code value
ELSE IF gate = 1:
    cfl = "95"
ELSE IF gate = 2:
    cfl = "90"
ELSE IF gate = 3:
    cfl = "88"
ELSE IF gate = 4 AND hs_code exists:
    cfl = "86"
ELSE IF gate = 5:
    cfl = NULL
ELSE:
    cfl = "" (empty - unclassified)
```

**Flag Lookup:**
- 1 = HS code exists (ready for processing)
- 0 = No HS code (needs manual review)

---

### STEP 11: Country Scope Classification

**Purpose:** Determine regulatory scope and assign classification IDs per market

**Process:**
1. Fetch country scope config from API
2. Determine: origin_country_in_scope, destination_country_in_scope (boolean)
3. Assign Classification IDs based on scope + gate:

**Classification ID Rules:**

| Scenario | ID | Meaning |
|----------|----|---------| 
| No HS code + In Scope | 5S | Unclassified, In-Scope market |
| No HS code + Out-of-Scope | 5N | Unclassified, Out-of-Scope market |
| Gate 3 + In Scope | 30AS | High priority, In-Scope |
| Gate 2 + In Scope | 30BS | Standard priority, In-Scope |
| Gate 3 + Out-of-Scope | 10AN | High priority, Out-of-Scope |
| Gate 2 + Out-of-Scope | 10BN | Standard priority, Out-of-Scope |

---

### STEP 12: Final Output Preparation & Column Selection

**Purpose:** Standardize and export final classified dataset

**Output Columns:**

**Identifiers:**
- sku, lane, origin_country, destination_country, origin_destination_country

**Product Information:**
- product_description, product_hierarchy, brand, business_group, category, sub_category, product_image_link

**Tariff & Classification:**
- hs_code, hs_code_six_plus_origin, hs_code_six_plus_destination, reference_hs_code, cfl, cfl_origin, cfl_destination

**Processing Gates:**
- gate (1-5 or NULL), wave (wave number)

**Compliance & Scope:**
- origin_classification_id, destination_classification_id
- origin_country_scope_status, destination_country_scope_status

**Metadata & Audit:**
- report_date, flag_lookup (1=HS code found, 0=not found)
- trigger_id (unique run identifier)
- classification_rational, remarks, meta_data (empty for future use)

---

## Data Quality & Audit Trail

### SKU Tracking Report

**Generated:** `SKU_Tracking_Report_*.xlsx`

**Sheet 1: Summary**
```
Metric | Count
──────────────────────────────────────────────────
📥 INPUT - Total SKUs | N
🔄 TRANSFORMATION - SKUs after dedup | Y
❌ TRANSFORMATION - SKUs skipped | X-Y
📊 GTS CHECK - Total SKUs in GTS | Z
✅ GTS CHECK - SKUs with HS_CODE | Z1
⚠️  GTS CHECK - SKUs without HS_CODE | Z2
🎯 FINAL - SKUs with HS Code | F1
⚠️  FINAL - SKUs without HS Code | F2
```

**Sheet 2: SKU Details**
- SKU, Status, Reason, Count
- Status values: PROCESSED, TRANSFORMED_NO_HS, SKIPPED_TRANSFORMATION, SKIPPED_GTS_CHECK

---

## Output Specifications

### Output Files

| File | Records | Purpose |
|------|---------|---------|
| final_output_lookup.csv | All classified SKUs | Complete reference dataset |
| final_output_lookup_one.csv | Only HS-coded SKUs | Primary for downstream processing |
| SKU_Tracking_Report_*.xlsx | Summary + Details | Audit trail and QA |

### Output Schema

```
id (INT) - Row identifier
sku (VARCHAR 20) - Product identifier
lane (VARCHAR 50) - Origin-Destination-SKU
origin_country (VARCHAR 50) - Source country
destination_country (VARCHAR 50) - Target country
product_description (VARCHAR 500) - Product name
product_hierarchy (VARCHAR 500) - Category hierarchy
brand (VARCHAR 100) - Marketing brand
business_group (VARCHAR 100) - Portfolio group
category (VARCHAR 100) - Product category
sub_category (VARCHAR 100) - Sub-category
hs_code (VARCHAR 20) - Assigned HS code
cfl (VARCHAR 10) - Tariff classification flag
gate (INT) - Assignment gate (1-5)
wave (INT) - Market wave number
origin_classification_id (VARCHAR 10) - Regulatory ID
destination_classification_id (VARCHAR 10) - Regulatory ID
origin_country_scope_status (VARCHAR 20) - In/Out of Scope
destination_country_scope_status (VARCHAR 20) - In/Out of Scope
report_date (DATE) - Data report date
flag_lookup (INT) - 1=HS code found, 0=not found
trigger_id (VARCHAR 100) - Unique run identifier
```

---

## Error Handling

### Common Issues & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| JWT Token Expired | Token has expired | Refresh token and re-run |
| File Encoding Error | Special characters in CSV | Auto-retry with different encodings |
| No CSV matching 'pattern' | File not uploaded | Check file naming conventions |
| GTS download failed (401) | Authentication invalid | Verify jwt_token is fresh |
| No HS code generated | All lookup tiers failed | Check reference data; may need manual classification |
| Excel conversion fails | File corrupted | Validate Excel file structure |

### Failure Notifications

When processing fails:
```
Email Notification Sent:
├─ Type: Lookup Flow Failure
├─ Recipient: Configured user
├─ Content: Error message + failing step
└─ Action: Review logs and retry with corrected inputs
```

---

## Business Use Cases

### Use Case 1: Monthly Forecast Processing

**Scenario:** New monthly forecast with demand updates

**Process Flow:**
```
1. Upload new Cross Border DSR-SKU detail_06-2026.xlsx
2. Run notebook with fresh jwt_token
3. Pipeline automatically:
   ✅ Identifies new file
   ✅ Locks in _archived (prevents re-run)
   ✅ Loads and transforms forecast
   ✅ Filters out already-processed SKUs
   ✅ Enriches with product data
   ✅ Classifies new SKUs
   ✅ Uploads to GTS database
   ✅ Sends success notification
4. Output: Ready for supply chain & customs teams
```

---

### Use Case 2: New Product Launch

**Scenario:** New product (SKU=987654) to 3 countries

**Result:**
```
Product added to ESB catalogs
↓
Forecast includes new lanes
↓
Pipeline classifies via hierarchy match (Gate 2)
↓
HS code assigned automatically
↓
Product ready for market without manual delay
```

---

### Use Case 3: Brazil Export Compliance

**Scenario:** Products exported from Brazil need CFL=95

**Result:**
```
Forecast includes Brazil → USA lanes
↓
Reference mapping finds CFL=95
↓
Gate 1 triggered (Brazil special processing)
↓
Customs team receives flagged records for special routes
```

---

## Integration Points

### Input APIs

```
GET /countries
├─ Authorization: Bearer {jwt_token}
├─ Returns: [{country_name, country_in_scope: boolean}, ...]
└─ Used In: Step 11 (Country Scope Classification)

GET /gts/origin-gts/download
├─ Authorization: Bearer {jwt_token}
├─ Returns: CSV with SKU, HS_CODE columns
└─ Used In: Step 3 (Incremental Filter)
```

### Output APIs

```
POST /core/upload
├─ Authorization: Bearer {jwt_token}
├─ Payload: final_output_lookup_one.csv
├─ Purpose: Insert classified SKUs into GTS database
└─ Response: Success/failure status

POST /api/email/send-lookup-notification
├─ Authorization: Bearer {jwt_token}
├─ Params: status ("succeeded"/"failed"), message
└─ Purpose: Alert stakeholders of processing status
```

---

## Technical Glossary

| Term | Definition | Example |
|------|-----------|---------|
| SKU | Stock Keeping Unit | 12345678 |
| Lane | Origin-Destination movement | BRAZIL-USA |
| HS Code | Harmonized System tariff code | 62044200 |
| CFL | Tariff Classification Flag | 90, 95, 88, 86 |
| Gate | Classification confidence (1=highest) | Gate 2 |
| Wave | Market rollout phase | Wave 1 (priority) |
| ESB | Product catalog system | Beauty, HC, PC |
| RTVA | Supply chain tracking | Import/Export views |
| NCM | Brazilian tariff code | 62044200 |
| In Scope | Subject to classification | 30AS classification |
| Not In Scope | Exempt from normal rules | 10AN classification |
| flag_lookup | Binary HS code indicator | 1=found, 0=not found |

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-06-11 | Data Engineering | Initial documentation |

---

**For questions or clarifications, contact the Data Engineering team.**