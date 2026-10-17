# Queries: Snowflake SQL and Power BI DAX

Run the SQL files in order. Replace every `<PLACEHOLDER>` before running and never commit real keys or passwords.

## 00_cost_controls.sql

```sql
-- =====================================================================
-- 00 · Cost controls (run first)
-- Smallest warehouse, auto-suspend after 60 s idle, and a hard
-- monthly credit cap so a mistake can never run up a bill.
-- =====================================================================
USE ROLE ACCOUNTADMIN;

CREATE WAREHOUSE IF NOT EXISTS COMPUTE_WH
  WAREHOUSE_SIZE      = 'XSMALL'
  AUTO_SUSPEND        = 60
  AUTO_RESUME         = TRUE
  INITIALLY_SUSPENDED = TRUE;

USE WAREHOUSE COMPUTE_WH;

-- Notify at 80 %, suspend all warehouses at 100 % of 10 credits/month
CREATE OR REPLACE RESOURCE MONITOR trial_guard
  WITH CREDIT_QUOTA = 10
  FREQUENCY         = MONTHLY
  START_TIMESTAMP   = IMMEDIATELY
  TRIGGERS ON 80  PERCENT DO NOTIFY
           ON 100 PERCENT DO SUSPEND_IMMEDIATE;

ALTER ACCOUNT SET RESOURCE_MONITOR = trial_guard;
```

## 01_s3_stage.sql

```sql
-- =====================================================================
-- 01 · Connect Snowflake to S3
-- Separate "plumbing" database for file formats and stages.
--
-- AWS side (done in the console, see README):
--   * IAM policy  snowflake_energy_read_policy  -> read-only on
--     s3://project2-snowflake/energy/*  (least privilege)
--   * IAM user    snowflake_loader  with that policy + an access key
-- =====================================================================
USE ROLE ACCOUNTADMIN;

CREATE DATABASE IF NOT EXISTS MANAGE_DB;
CREATE SCHEMA   IF NOT EXISTS MANAGE_DB.file_formats;
CREATE SCHEMA   IF NOT EXISTS MANAGE_DB.external_stages;

-- CSV format used for loading
CREATE OR REPLACE FILE FORMAT MANAGE_DB.file_formats.csv_fileformat
  TYPE                         = CSV
  FIELD_DELIMITER              = ','
  SKIP_HEADER                  = 1
  NULL_IF                      = ('NULL', 'null', '')
  EMPTY_FIELD_AS_NULL          = TRUE
  FIELD_OPTIONALLY_ENCLOSED_BY = '"';   -- handles commas inside quoted text

-- Helper format that reads the header row (used by INFER_SCHEMA)
CREATE OR REPLACE FILE FORMAT MANAGE_DB.file_formats.csv_header_format
  TYPE                         = CSV
  PARSE_HEADER                 = TRUE
  FIELD_OPTIONALLY_ENCLOSED_BY = '"';

-- External stage: Snowflake's pointer to the S3 folder.
-- NEVER commit real keys. Replace the placeholders when running,
-- then clear them from the worksheet.
CREATE OR REPLACE STAGE MANAGE_DB.external_stages.energy_stage
  URL         = 's3://project2-snowflake/energy/'
  CREDENTIALS = (AWS_KEY_ID = '<AWS_ACCESS_KEY_ID>' AWS_SECRET_KEY = '<AWS_SECRET_ACCESS_KEY>')
  FILE_FORMAT = MANAGE_DB.file_formats.csv_fileformat;

-- Test: should list renewable_power_plants_DE.csv (~342 MB)
LIST @MANAGE_DB.external_stages.energy_stage;

-- Discover the 27 columns and suggested types
SELECT COLUMN_NAME, TYPE
FROM TABLE(INFER_SCHEMA(
  LOCATION    => '@MANAGE_DB.external_stages.energy_stage',
  FILE_FORMAT => 'MANAGE_DB.file_formats.csv_header_format'
));

-- ---------------------------------------------------------------------
-- Production alternative (no long-lived keys): a STORAGE INTEGRATION
-- with an IAM role + external ID. Attempted first; see README
-- "Lessons learned". Template:
--
-- CREATE STORAGE INTEGRATION s3_energy_integration
--   TYPE = EXTERNAL_STAGE  STORAGE_PROVIDER = 'S3'  ENABLED = TRUE
--   STORAGE_AWS_ROLE_ARN      = 'arn:aws:iam::<ACCOUNT_ID>:role/snowflake_energy_role'
--   STORAGE_ALLOWED_LOCATIONS = ('s3://project2-snowflake/energy/');
-- DESC INTEGRATION s3_energy_integration;  -- copy IAM user ARN + external ID
--                                          -- into the role's trust policy
-- ---------------------------------------------------------------------
```

## 02_raw_load.sql

```sql
-- =====================================================================
-- 02 · RAW layer (bronze) — the "L" in ELT
-- Every source column lands as STRING so the load can never fail on a
-- bad value. Typing and cleaning happen later, in SQL.
-- =====================================================================
USE ROLE ACCOUNTADMIN;
USE WAREHOUSE COMPUTE_WH;

CREATE DATABASE IF NOT EXISTS ENERGY_DB;
CREATE SCHEMA   IF NOT EXISTS ENERGY_DB.RAW;
CREATE SCHEMA   IF NOT EXISTS ENERGY_DB.CLEAN;

CREATE OR REPLACE TABLE ENERGY_DB.RAW.renewable_plants (
  commissioning_date     STRING,
  decommissioning_date   STRING,
  energy_source_level_1  STRING,
  energy_source_level_2  STRING,
  energy_source_level_3  STRING,
  technology             STRING,
  electrical_capacity    STRING,
  thermal_capacity       STRING,
  voltage_level          STRING,
  tso                    STRING,
  dso                    STRING,
  dso_id                 STRING,
  eeg_id                 STRING,
  bnetza_id              STRING,
  federal_state          STRING,
  postcode               STRING,
  municipality_code      STRING,
  municipality           STRING,
  address                STRING,
  address_number         STRING,
  utm_zone               STRING,
  utm_east               STRING,
  utm_north              STRING,
  lat                    STRING,
  lon                    STRING,
  data_source            STRING,
  comment                STRING,
  -- audit columns
  source_file            STRING,
  loaded_at              TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

COPY INTO ENERGY_DB.RAW.renewable_plants (
  commissioning_date, decommissioning_date, energy_source_level_1, energy_source_level_2,
  energy_source_level_3, technology, electrical_capacity, thermal_capacity, voltage_level,
  tso, dso, dso_id, eeg_id, bnetza_id, federal_state, postcode, municipality_code,
  municipality, address, address_number, utm_zone, utm_east, utm_north, lat, lon,
  data_source, comment, source_file
)
FROM (
  SELECT $1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,$13,$14,$15,$16,$17,$18,$19,$20,
         $21,$22,$23,$24,$25,$26,$27, METADATA$FILENAME
  FROM @MANAGE_DB.external_stages.energy_stage
)
FILE_FORMAT = (FORMAT_NAME = 'MANAGE_DB.file_formats.csv_fileformat');

-- Result: 1,727,278 rows loaded
SELECT COUNT(*) FROM ENERGY_DB.RAW.renewable_plants;
```

## 03_clean_transform.sql

```sql
-- =====================================================================
-- 03 · CLEAN layer (silver) — the "T" in ELT
--  * text -> DATE / DOUBLE with TRY_ functions (bad values become NULL)
--  * derived columns: commissioning_year, is_active
--  * trimmed text, empty strings -> NULL
--  * placeholder dates 1899-12-30/31 and future typos (e.g. 2107) -> NULL
--    (genuine 1890s hydro plants are kept)
--  * rows without a positive capacity removed (1 row)
--  * street addresses, UTM coords, internal IDs and comments dropped
--    (privacy: many plants are private rooftop systems)
-- =====================================================================
USE ROLE ACCOUNTADMIN;
USE WAREHOUSE COMPUTE_WH;

CREATE OR REPLACE TABLE ENERGY_DB.CLEAN.renewable_plants AS
WITH d AS (
  SELECT *,
    CASE
      WHEN TRY_TO_DATE(commissioning_date) IN ('1899-12-30', '1899-12-31') THEN NULL  -- placeholder dates
      WHEN TRY_TO_DATE(commissioning_date) > CURRENT_DATE()                 THEN NULL  -- future typos
      ELSE TRY_TO_DATE(commissioning_date)
    END AS commissioning_dt
  FROM ENERGY_DB.RAW.renewable_plants
)
SELECT
  commissioning_dt                                             AS commissioning_date,
  YEAR(commissioning_dt)                                       AS commissioning_year,
  TRY_TO_DATE(decommissioning_date)                            AS decommissioning_date,
  IFF(TRY_TO_DATE(decommissioning_date) IS NULL, TRUE, FALSE)  AS is_active,
  NULLIF(TRIM(energy_source_level_2), '')                      AS energy_source,
  NULLIF(TRIM(energy_source_level_3), '')                      AS energy_source_detail,
  NULLIF(TRIM(technology), '')                                 AS technology,
  TRY_TO_DOUBLE(electrical_capacity)                           AS capacity_mw,
  NULLIF(TRIM(voltage_level), '')                              AS voltage_level,
  NULLIF(TRIM(tso), '')                                        AS tso,
  NULLIF(TRIM(federal_state), '')                              AS federal_state,
  NULLIF(TRIM(postcode), '')                                   AS postcode,
  NULLIF(TRIM(municipality), '')                               AS municipality,
  TRY_TO_DOUBLE(lat)                                           AS lat,
  TRY_TO_DOUBLE(lon)                                           AS lon,
  NULLIF(TRIM(data_source), '')                                AS data_source,
  loaded_at
FROM d
WHERE TRY_TO_DOUBLE(electrical_capacity) > 0;

-- Reconciliation RAW vs CLEAN
-- Result: raw 1,727,278 | clean 1,727,277 | total 106.8 GW
SELECT
  (SELECT COUNT(*) FROM ENERGY_DB.RAW.renewable_plants)                                   AS raw_rows,
  (SELECT COUNT(*) FROM ENERGY_DB.CLEAN.renewable_plants)                                 AS clean_rows,
  (SELECT COUNT(*) FROM ENERGY_DB.CLEAN.renewable_plants WHERE commissioning_date IS NULL) AS missing_dates,
  (SELECT COUNT(*) FROM ENERGY_DB.CLEAN.renewable_plants WHERE federal_state IS NULL)      AS missing_state,
  (SELECT ROUND(SUM(capacity_mw)/1000, 1) FROM ENERGY_DB.CLEAN.renewable_plants)          AS total_capacity_gw;
```

## 04_data_quality.sql

```sql
-- =====================================================================
-- 04 · Data quality checks -> ENERGY_DB.AUDIT.dq_results
-- Re-run after every rebuild of CLEAN.
-- =====================================================================
USE ROLE ACCOUNTADMIN;
USE WAREHOUSE COMPUTE_WH;

CREATE SCHEMA IF NOT EXISTS ENERGY_DB.AUDIT;

CREATE OR REPLACE TABLE ENERGY_DB.AUDIT.dq_results AS
WITH c AS (SELECT * FROM ENERGY_DB.CLEAN.renewable_plants),
t AS (SELECT COUNT(*) AS total FROM c)
SELECT 'missing_energy_source' AS check_name,
       (SELECT COUNT(*) FROM c WHERE energy_source IS NULL) AS failed_rows,
       total, CURRENT_TIMESTAMP() AS checked_at FROM t
UNION ALL
SELECT 'missing_federal_state',
       (SELECT COUNT(*) FROM c WHERE federal_state IS NULL), total, CURRENT_TIMESTAMP() FROM t
UNION ALL
SELECT 'capacity_over_1000_mw',
       (SELECT COUNT(*) FROM c WHERE capacity_mw > 1000), total, CURRENT_TIMESTAMP() FROM t
UNION ALL
SELECT 'missing_coordinates',
       (SELECT COUNT(*) FROM c WHERE lat IS NULL OR lon IS NULL), total, CURRENT_TIMESTAMP() FROM t
UNION ALL
SELECT 'coordinates_outside_germany',
       (SELECT COUNT(*) FROM c WHERE lat IS NOT NULL AND lon IS NOT NULL
          AND (lat NOT BETWEEN 47.2 AND 55.1 OR lon NOT BETWEEN 5.8 AND 15.1)), total, CURRENT_TIMESTAMP() FROM t
UNION ALL
SELECT 'commissioning_date_out_of_range',
       (SELECT COUNT(*) FROM c WHERE commissioning_date < '1900-01-01' OR commissioning_date > CURRENT_DATE()), total, CURRENT_TIMESTAMP() FROM t
UNION ALL
SELECT 'decommissioned_before_commissioned',
       (SELECT COUNT(*) FROM c WHERE decommissioning_date < commissioning_date), total, CURRENT_TIMESTAMP() FROM t
UNION ALL
SELECT 'exact_duplicate_rows',
       (SELECT COUNT(*) - COUNT(DISTINCT HASH(commissioning_date, energy_source, technology, capacity_mw,
                                             postcode, municipality, lat, lon)) FROM c),
       total, CURRENT_TIMESTAMP() FROM t;

SELECT check_name, failed_rows, total,
       ROUND(100 * failed_rows / total, 3) AS failed_pct
FROM ENERGY_DB.AUDIT.dq_results
ORDER BY failed_rows DESC;

-- ---------------------------------------------------------------------
-- Investigations
-- ---------------------------------------------------------------------
-- Out-of-range dates (found 17 before the fix, 4 after = real 1890s hydro)
SELECT commissioning_date, energy_source, technology, capacity_mw, federal_state, municipality
FROM ENERGY_DB.CLEAN.renewable_plants
WHERE commissioning_date < '1900-01-01' OR commissioning_date > CURRENT_DATE()
ORDER BY commissioning_date;

-- Plants without a federal state (430 small plants, ~0.03 GW, mostly SW Germany)
SELECT energy_source, technology,
       COUNT(*)                        AS plants,
       ROUND(SUM(capacity_mw)/1000, 2) AS capacity_gw,
       ROUND(AVG(lat), 2)              AS avg_lat,
       ROUND(AVG(lon), 2)              AS avg_lon
FROM ENERGY_DB.CLEAN.renewable_plants
WHERE federal_state IS NULL
GROUP BY energy_source, technology
ORDER BY capacity_gw DESC;
```

## 05_mart_star_schema.sql

```sql
-- =====================================================================
-- 05 · MART layer (gold) — star schema for Power BI
--
--              dim_energy_source
--                     |
--   dim_date ——— fact_plants ——— dim_location
-- =====================================================================
USE ROLE ACCOUNTADMIN;
USE WAREHOUSE COMPUTE_WH;

CREATE SCHEMA IF NOT EXISTS ENERGY_DB.MART;

-- Energy source dimension
CREATE OR REPLACE TABLE ENERGY_DB.MART.dim_energy_source AS
SELECT ROW_NUMBER() OVER (ORDER BY energy_source, energy_source_detail, technology) AS energy_source_key,
       energy_source, energy_source_detail, technology
FROM (
  SELECT DISTINCT
    COALESCE(energy_source, 'Unknown')        AS energy_source,
    COALESCE(energy_source_detail, 'Unknown') AS energy_source_detail,
    COALESCE(technology, 'Unknown')           AS technology
  FROM ENERGY_DB.CLEAN.renewable_plants
);

-- Location dimension
CREATE OR REPLACE TABLE ENERGY_DB.MART.dim_location AS
SELECT ROW_NUMBER() OVER (ORDER BY federal_state, municipality, postcode) AS location_key,
       federal_state, municipality, postcode
FROM (
  SELECT DISTINCT
    COALESCE(federal_state, 'Unknown') AS federal_state,
    COALESCE(municipality, 'Unknown')  AS municipality,
    COALESCE(postcode, 'Unknown')      AS postcode
  FROM ENERGY_DB.CLEAN.renewable_plants
);

-- Date dimension: one row per day, 1890 -> latest commissioning date
CREATE OR REPLACE TABLE ENERGY_DB.MART.dim_date AS
SELECT TO_NUMBER(TO_CHAR(d, 'YYYYMMDD')) AS date_key,
       d                AS date,
       YEAR(d)          AS year,
       QUARTER(d)       AS quarter,
       MONTH(d)         AS month,
       MONTHNAME(d)     AS month_name
FROM (
  SELECT DATEADD(day, ROW_NUMBER() OVER (ORDER BY SEQ4()) - 1, '1890-01-01'::DATE) AS d
  FROM TABLE(GENERATOR(ROWCOUNT => 50000))
)
WHERE d <= (SELECT MAX(commissioning_date) FROM ENERGY_DB.CLEAN.renewable_plants);

-- Fact table: one row per plant
CREATE OR REPLACE TABLE ENERGY_DB.MART.fact_plants AS
SELECT ROW_NUMBER() OVER (ORDER BY c.commissioning_date, c.capacity_mw) AS plant_id,
       e.energy_source_key,
       l.location_key,
       TO_NUMBER(TO_CHAR(c.commissioning_date, 'YYYYMMDD')) AS commissioning_date_key,
       c.commissioning_date,
       c.decommissioning_date,
       c.is_active,
       c.capacity_mw,
       c.voltage_level,
       c.lat,
       c.lon
FROM ENERGY_DB.CLEAN.renewable_plants c
JOIN ENERGY_DB.MART.dim_energy_source e
  ON  e.energy_source        = COALESCE(c.energy_source, 'Unknown')
  AND e.energy_source_detail = COALESCE(c.energy_source_detail, 'Unknown')
  AND e.technology           = COALESCE(c.technology, 'Unknown')
JOIN ENERGY_DB.MART.dim_location l
  ON  l.federal_state = COALESCE(c.federal_state, 'Unknown')
  AND l.municipality  = COALESCE(c.municipality, 'Unknown')
  AND l.postcode      = COALESCE(c.postcode, 'Unknown');

-- ---------------------------------------------------------------------
-- Validation
-- ---------------------------------------------------------------------
-- No rows lost or duplicated by the joins (clean_rows = fact_rows = 1,727,277)
SELECT
  (SELECT COUNT(*) FROM ENERGY_DB.CLEAN.renewable_plants)  AS clean_rows,
  (SELECT COUNT(*) FROM ENERGY_DB.MART.fact_plants)        AS fact_rows,
  (SELECT COUNT(*) FROM ENERGY_DB.MART.dim_energy_source)  AS energy_sources,
  (SELECT COUNT(*) FROM ENERGY_DB.MART.dim_location)       AS locations,
  (SELECT COUNT(*) FROM ENERGY_DB.MART.dim_date)           AS dates;

-- Referential integrity: all three should be 0
SELECT
  COUNT_IF(e.energy_source_key IS NULL) AS missing_energy_link,
  COUNT_IF(l.location_key IS NULL)      AS missing_location_link,
  COUNT_IF(f.commissioning_date_key IS NOT NULL AND d.date_key IS NULL) AS missing_date_link
FROM ENERGY_DB.MART.fact_plants f
LEFT JOIN ENERGY_DB.MART.dim_energy_source e ON f.energy_source_key = e.energy_source_key
LEFT JOIN ENERGY_DB.MART.dim_location l      ON f.location_key = l.location_key
LEFT JOIN ENERGY_DB.MART.dim_date d          ON f.commissioning_date_key = d.date_key;

-- Sanity: capacity by energy source
SELECT e.energy_source,
       COUNT(*)                          AS plants,
       ROUND(SUM(f.capacity_mw)/1000, 1) AS capacity_gw
FROM ENERGY_DB.MART.fact_plants f
JOIN ENERGY_DB.MART.dim_energy_source e ON f.energy_source_key = e.energy_source_key
GROUP BY e.energy_source
ORDER BY capacity_gw DESC;
```

## 06_bi_access.sql

```sql
-- =====================================================================
-- 06 · Least-privilege access for Power BI
-- BI_READER can only SELECT from the MART schema — not RAW, CLEAN or AUDIT.
-- =====================================================================
USE ROLE ACCOUNTADMIN;

CREATE ROLE IF NOT EXISTS BI_READER;

GRANT USAGE  ON WAREHOUSE COMPUTE_WH                  TO ROLE BI_READER;
GRANT USAGE  ON DATABASE  ENERGY_DB                   TO ROLE BI_READER;
GRANT USAGE  ON SCHEMA    ENERGY_DB.MART              TO ROLE BI_READER;
GRANT SELECT ON ALL TABLES    IN SCHEMA ENERGY_DB.MART TO ROLE BI_READER;
GRANT SELECT ON FUTURE TABLES IN SCHEMA ENERGY_DB.MART TO ROLE BI_READER;

-- Dedicated service-style user for Power BI (the admin login uses Google SSO).
-- Replace the placeholder; never commit a real password.
CREATE USER IF NOT EXISTS powerbi_user
  PASSWORD                = '<STRONG_PASSWORD>'
  DEFAULT_ROLE            = BI_READER
  DEFAULT_WAREHOUSE       = COMPUTE_WH
  DEFAULT_SECONDARY_ROLES = ()
  MUST_CHANGE_PASSWORD    = FALSE
  COMMENT                 = 'Read-only user for Power BI';

GRANT ROLE BI_READER TO USER powerbi_user;

-- ---------------------------------------------------------------------
-- Gotcha: secondary roles
-- New users have DEFAULT_SECONDARY_ROLES = ('ALL'), so after
-- USE ROLE BI_READER the admin privileges still applied and the
-- "should fail" test below succeeded. Disabling them fixed it.
-- ---------------------------------------------------------------------
-- ALTER USER "<your_admin_user>" SET DEFAULT_SECONDARY_ROLES = ();

-- Test
USE ROLE BI_READER;
USE SECONDARY ROLES NONE;
SELECT COUNT(*) FROM ENERGY_DB.MART.fact_plants;            -- works: 1,727,277
SELECT COUNT(*) FROM ENERGY_DB.CLEAN.renewable_plants;      -- fails: not authorized
USE ROLE ACCOUNTADMIN;
```

## 99_teardown.sql

```sql
-- =====================================================================
-- 99 · Teardown — run when the project is finished
-- Also in AWS: delete the snowflake_loader access key (and user),
-- the IAM policy, and optionally the S3 object/bucket.
-- =====================================================================
USE ROLE ACCOUNTADMIN;

DROP USER      IF EXISTS powerbi_user;
DROP ROLE      IF EXISTS BI_READER;
DROP DATABASE  IF EXISTS ENERGY_DB;
DROP DATABASE  IF EXISTS MANAGE_DB;
ALTER WAREHOUSE COMPUTE_WH SUSPEND;
```

---

# Power BI: DAX measures

All measures live on `FACT_PLANTS`. Data is loaded in **Import** mode from the `ENERGY_DB.MART` star schema.

## Model

| From (many) | To (one) | Cross-filter |
|---|---|---|
| `FACT_PLANTS[ENERGY_SOURCE_KEY]` | `DIM_ENERGY_SOURCE[ENERGY_SOURCE_KEY]` | Single |
| `FACT_PLANTS[LOCATION_KEY]` | `DIM_LOCATION[LOCATION_KEY]` | Single |
| `FACT_PLANTS[COMMISSIONING_DATE_KEY]` | `DIM_DATE[DATE_KEY]` | Single |

`DIM_DATE` is marked as the date table (column `DATE`).

## Measures

```DAX
Total Capacity MW = SUM(FACT_PLANTS[CAPACITY_MW])
```
Installed electrical capacity in megawatts. The base measure the others build on.

```DAX
Total Capacity GW = DIVIDE([Total Capacity MW], 1000)
```
Same value in gigawatts, for readable KPI cards (106.8 GW instead of 106,800 MW).

```DAX
Plant Count = COUNTROWS(FACT_PLANTS)
```
Number of plants in the current filter context.

```DAX
Avg Plant Size kW = DIVIDE([Total Capacity MW], [Plant Count]) * 1000
```
Average plant size. About 62 kW nationally, because most plants are small rooftop PV systems.

```DAX
Capacity Share % =
DIVIDE(
    [Total Capacity MW],
    CALCULATE([Total Capacity MW], ALL(DIM_ENERGY_SOURCE))
)
```
Share of each energy source in the total. `ALL(DIM_ENERGY_SOURCE)` removes the energy-source filter for the denominator.

```DAX
Cumulative Capacity GW =
CALCULATE(
    [Total Capacity GW],
    FILTER(ALL(DIM_DATE[DATE]), DIM_DATE[DATE] <= MAX(DIM_DATE[DATE]))
)
```
Running total of installed capacity over time: the Energiewende growth curve. It needs the separate, continuous date table.

## Note: capacity vs energy
These are **capacity** figures (MW/GW, maximum power output), not **generation** (MWh/GWh, power × time). A 1 MW solar plant produces nothing at night.
