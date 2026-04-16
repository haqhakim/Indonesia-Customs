# Indonesia Trade Analytics Dashboard
### End-to-end data pipeline: The Observatory of Economic Complexity Data → BigQuery → Tableau Prep → Tableau Public

> **Live Dashboard →** [View on Tableau Public](https://public.tableau.com/authoring/TradeIndonesia/Dashboard1#3)

---

## 📌 Project Overview

This project builds a bilateral trade analytics dashboard for Indonesia, analyzing export and import flows between Indonesia and its two largest trading partners — China and the United States — across 2022–2024.

The full pipeline runs from raw OEC data through BigQuery SQL validation, Tableau Prep cleaning, and final visualization on Tableau Public.

| | |
|---|---|
| **Data source** | [Observatory of Economic Complexity (OEC)](https://oec.world/en/profile/country/idn) |
| **Classification** | HS22 — Harmonized System 2022 |
| **Period** | 2022 - 2024 |
| **Trade partners** | China, United States |
| **Trade directions** | Export (Indonesia → partner) / Import (partner → Indonesia) |

---

## 📊 Key Findings

| Metric | Value |
|---|---|
| Total Export Value (2022–2024) | **IDR292.1B** |
| Total Import Value (2022–2024) | **IDR236.2B** |
| Overall Trade Balance | **+IDR55.8B surpluss** |
| Export / Import Ratio | **1.2x** |
| Active HS4 product codes | **1.218** |

**Top export surplus sections (2024):**
- Metals — IDR15.9B
- Mineral Products — IDR15.8B
- Animal and Vegetable Bi-Products — IDR7.1B

---

## Data Pipeline
```
OEC Website
    │
    │  Manual download (CSV)
    ▼
Google BigQuery
    │
    │  Upload 4 raw CSVs as individual tables
    │  SQL: UNION ALL + SAFE_CAST + data validation
    │  Output: cleaned_trade table
    ▼
Tableau Prep Builder
    │
    │  Live BigQuery connection
    │  Additional cleaning + column renaming
    │  Output: Trade table Indonesia.csv
    ▼
Tableau Public (Published)
```

---

## 🔧 Step-by-Step Process

### 1. Data Collection

Downloaded 4 CSV files from [OEC](https://oec.world/en/profile/country/idn) — bilateral trade data at HS4 product level:

| File | Direction | Partner |
|---|---|---|
| `Export_to_China.csv` | Export | China |
| `Import_from_China.csv` | Import | China |
| `Export_to_US.csv` | Export | United States |
| `Import_from_US.csv` | Import | United States |

---

### 2. BigQuery — Upload & Schema

Created Google Cloud project: `indonesia-trade-analytics`
Created dataset: `indonesia_trade`

Uploaded each CSV as a separate table with **manual schema** to ensure correct types (BigQuery auto-detect reads Trade Value as FLOAT64).

**Validation checks run before cleaning:**

```sql

-- Null check
SELECT
  COUNTIF(`Trade Value` IS NULL) AS null_trade_value,
  COUNTIF(Year IS NULL)          AS null_year,
  COUNTIF(HS2 IS NULL)           AS null_hs2,
  COUNTIF(HS4 IS NULL)           AS null_hs4
FROM `indonesia_trade.export_china`

-- Duplicate check
SELECT HS4, Year, COUNT(*) AS occurrences
FROM `indonesia_trade.export_china`
GROUP BY HS4, Year
HAVING COUNT(*) > 1

-- Year range check
SELECT Year, COUNT(*) AS rows
FROM `indonesia_trade.export_china`
GROUP BY Year
ORDER BY Year
```

**All checks passed:** 0 nulls, 0 duplicates, years 2022–2024 only.

---

### 3. BigQuery — Cleaning & Master Table

```sql
CREATE OR REPLACE TABLE `indonesia-trade-analytics.indonesia_trade.cleaned_trade` AS

WITH combined AS (

  SELECT
    'export'                           AS trade_type,
    'US'                               AS country,
    SAFE_CAST(HS2 AS STRING)           AS hs2,
    SAFE_CAST(HS4 AS STRING)           AS hs4,
    SAFE_CAST(Section AS STRING)       AS section,
    SAFE_CAST(`Trade Value` AS FLOAT64) AS trade_value,
    SAFE_CAST(Year AS INT64)           AS year
  FROM `indonesia-trade-analytics.indonesia_trade.export_US`

  UNION ALL

  SELECT 'export', 'China',
    SAFE_CAST(HS2 AS STRING),
    SAFE_CAST(HS4 AS STRING),
    SAFE_CAST(Section AS STRING),
    SAFE_CAST(`Trade Value` AS FLOAT64),
    SAFE_CAST(Year AS INT64)
  FROM `indonesia-trade-analytics.indonesia_trade.export_china`

  UNION ALL

  SELECT 'import', 'US',
    SAFE_CAST(HS2 AS STRING),
    SAFE_CAST(HS4 AS STRING),
    SAFE_CAST(Section AS STRING),
    SAFE_CAST(`Trade Value` AS FLOAT64),
    SAFE_CAST(Year AS INT64)
  FROM `indonesia-trade-analytics.indonesia_trade.import_US`

  UNION ALL

  SELECT 'import', 'China',
    SAFE_CAST(HS2 AS STRING),
    SAFE_CAST(HS4 AS STRING),
    SAFE_CAST(Section AS STRING),
    SAFE_CAST(`Trade Value` AS FLOAT64),
    SAFE_CAST(Year AS INT64)
  FROM `indonesia-trade-analytics.indonesia_trade.import_china`

),

cleaned AS (
  SELECT
    trade_type,
    country,
    hs2,
    hs4,
    TRIM(section)                              AS section,
    CASE
      WHEN trade_value < 0 THEN NULL
      ELSE trade_value
    END                                        AS trade_value,
    CASE
      WHEN trade_type = 'export' THEN  trade_value
      ELSE                            -trade_value
    END                                        AS signed_trade_value,
    year
  FROM combined
)

SELECT *
FROM cleaned
WHERE
  hs2         IS NOT NULL
  AND hs4     IS NOT NULL
  AND trade_value IS NOT NULL
  AND year BETWEEN 2000 AND 2030;
```

**Cleaning decisions:**

| Issue | Action | Reason |
|---|---|---|
| Negative trade values | → NULL | OEC data shouldn't have negatives; indicates data error |
| Whitespace in Section | `TRIM()` | Prevents duplicate filter values in Tableau |
| Type safety | `SAFE_CAST` not `CAST` | Returns NULL instead of crashing on bad values |
| Year range guard | `BETWEEN 2000 AND 2030` | Catches corrupt year values (0, 9999, etc.) |
| Added `signed_trade_value` | Export = positive, Import = negative | Powers trade balance charts directly |

**Output table:** `cleaned_trade` — 12,410 rows, 8 columns

---

### 4. Tableau Prep Builder

Connected Tableau Prep directly to BigQuery (live connection — no CSV download required at this stage).

**Flow:**
```
[Input: cleaned_trade (BigQuery)] → [Clean 1] → [Output: Trade table Indonesia.csv]
```

**Clean step operations:**
- Renamed columns: `trade_type` → `Trade Type`, `hs2` → `HS Code`, `hs4` → `Hs4`, `section` → `Section`
- Verified field types: Trade Type (String), Country (Geographic), HS Code (String), Hs4 (String), Section (String)
- Confirmed 0 null values across all fields in profile pane

**Output:** `Trade table Indonesia.csv` — saved to Tableau Prep Datasources folder


---

### 5. Tableau Desktop / Public

Connected to `Trade table Indonesia.csv` as a Text File connection.

**Data source:** 7 fields, 12,410 rows

**Sheets built:**

| Sheet | Chart type | Key dimensions | Key measure |
|---|---|---|---|
| KPI Cards | BANs (text) | — | Total Export, Import, Balance, Ratio, HS4 count |
| Export vs Import over time | Grouped bar | Year, Country, Trade Type | SUM(Trade Value) |
| Trade Balance per Section | Horizontal bar | Section, Year | SUM(Trade Value) |
| Export to Import Top 10 Section | 100% stacked bar | Section | % of Total Trade Value |
| Export/Import Ratio Table | Crosstab | Section × Year | Ratio calculated field |

**Calculated fields used in Tableau:**

```
// Trade Balance
SUM(IF [Trade Type] = 'export' THEN [Trade Value] ELSE -[Trade Value] END)

// Export/Import Ratio
SUM(IF [Trade Type] = 'export' THEN [Trade Value] END)
/
SUM(IF [Trade Type] = 'import' THEN [Trade Value] END)

// Active HS4 Count
COUNTD(IF NOT ISNULL([Hs4]) THEN [Hs4] END)
```

**Dashboard layout:** Fixed 1200×900px, 5 KPI cards + 4 charts in 2×2 grid
