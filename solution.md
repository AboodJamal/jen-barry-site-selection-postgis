# Jen and Barry's Ice Cream Site Selection - Complete SQL Solution

## Overview
This document contains the complete SQL solution for automating Jen and Barry's ice cream business site selection process using PostGIS spatial analysis.

---

## Database Setup

### STEP 0: Enable PostGIS Extensions
```sql
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;
```

---

## Data Verification

### STEP 1: Verify Data Structure

#### 1.1 Check All Tables Exist
```sql
SELECT table_name 
FROM information_schema.tables 
WHERE table_schema = 'jen_barry' AND table_type = 'BASE TABLE';
```

**Expected Result:**
| table_name |
|------------|
| jen_barry.counties |
| jen_barry.cities |
| jen_barry.interstates |
| jen_barry.recreationareas |

#### 1.2 Check Row Counts
```sql
SELECT 'counties' as table_name, COUNT(*) as row_count FROM jen_barry."jen_barry.counties"
UNION ALL
SELECT 'cities', COUNT(*) FROM jen_barry."jen_barry.cities"
UNION ALL
SELECT 'interstates', COUNT(*) FROM jen_barry."jen_barry.interstates"
UNION ALL
SELECT 'recreationareas', COUNT(*) FROM jen_barry."jen_barry.recreationareas";
```

**Expected Result:**
| table_name | row_count |
|------------|-----------|
| counties | 43 |
| cities | 48 |
| interstates | 7 |
| recreationareas | 110 |

#### 1.3 Check Geometry Types and SRID
```sql
SELECT 'counties' as table_name, GeometryType(geom) as geom_type, ST_SRID(geom) as srid
FROM jen_barry."jen_barry.counties" LIMIT 1;

SELECT 'cities' as table_name, GeometryType(geom) as geom_type, ST_SRID(geom) as srid
FROM jen_barry."jen_barry.cities" LIMIT 1;

SELECT 'interstates' as table_name, GeometryType(geom) as geom_type, ST_SRID(geom) as srid
FROM jen_barry."jen_barry.interstates" LIMIT 1;

SELECT 'recreationareas' as table_name, GeometryType(geom) as geom_type, ST_SRID(geom) as srid
FROM jen_barry."jen_barry.recreationareas" LIMIT 1;
```

**Expected Results:**
| table_name | geom_type | srid |
|------------|-----------|------|
| counties | MULTIPOLYGON | 4267 |
| cities | POINT | 4267 |
| interstates | MULTILINESTRING | 4267 |
| recreationareas | MULTIPOLYGON | 4267 |

#### 1.4 Verify Key Column Value Ranges
```sql
-- Counties key columns
SELECT 
    'NO_FARMS87' as column_name, 
    MIN("NO_FARMS87") as min_val, 
    MAX("NO_FARMS87") as max_val
FROM jen_barry."jen_barry.counties"
UNION ALL
SELECT 
    'AGE_18_64', 
    MIN("AGE_18_64"), 
    MAX("AGE_18_64")
FROM jen_barry."jen_barry.counties"
UNION ALL
SELECT 
    'POP_SQMILE', 
    MIN("POP_SQMILE")::numeric, 
    MAX("POP_SQMILE")::numeric
FROM jen_barry."jen_barry.counties";
```

**Expected Result:**
| column_name | min_val | max_val |
|-------------|---------|---------|
| NO_FARMS87 | 25 | 4775 |
| AGE_18_64 | 2715 | 255417 |
| POP_SQMILE | 11 | 837 |

```sql
-- Cities key columns
SELECT 
    'CRIME_INDE' as column_name, 
    MIN("CRIME_INDE")::text as min_val, 
    MAX("CRIME_INDE")::text as max_val
FROM jen_barry."jen_barry.cities"
UNION ALL
SELECT 
    'UNIVERSITY', 
    MIN("UNIVERSITY")::text, 
    MAX("UNIVERSITY")::text
FROM jen_barry."jen_barry.cities";
```

**Expected Result:**
| column_name | min_val | max_val |
|-------------|---------|---------|
| CRIME_INDE | 0 | 0.098 |
| UNIVERSITY | 0 | 1 |

#### 1.5 List All County Names
```sql
SELECT "NAME" FROM jen_barry."jen_barry.counties" ORDER BY "NAME";
```

**Expected Result:** 43 counties from Bellows to Young

#### 1.6 List All Interstate Routes
```sql
SELECT DISTINCT "NAME" FROM jen_barry."jen_barry.interstates" ORDER BY "NAME";
```

**Expected Result:**
| NAME |
|------|
| I-40 |
| I-50 |
| I-55 |
| I-99 |
| SR-44 |
| SR-97 |

---

## Coordinate Transformation

### STEP 2: Create Transformed Geometry Columns (EPSG:2271)

**Why EPSG:2271?** Pennsylvania State Plane South (NAD83) uses feet as units, allowing accurate distance measurements.

**Distance conversions:**
- 10 miles = 52,800 feet
- 20 miles = 105,600 feet

```sql
-- Add new geometry columns in EPSG:2271 (PA State Plane South, feet)
ALTER TABLE jen_barry."jen_barry.counties" 
    ADD COLUMN IF NOT EXISTS geom_2271 geometry(MultiPolygon, 2271);

ALTER TABLE jen_barry."jen_barry.cities" 
    ADD COLUMN IF NOT EXISTS geom_2271 geometry(Point, 2271);

ALTER TABLE jen_barry."jen_barry.interstates" 
    ADD COLUMN IF NOT EXISTS geom_2271 geometry(MultiLineString, 2271);

ALTER TABLE jen_barry."jen_barry.recreationareas" 
    ADD COLUMN IF NOT EXISTS geom_2271 geometry(MultiPolygon, 2271);
```

### STEP 3: Transform Geometries from EPSG:4267 to EPSG:2271

```sql
UPDATE jen_barry."jen_barry.counties" 
    SET geom_2271 = ST_Transform(geom, 2271);

UPDATE jen_barry."jen_barry.cities" 
    SET geom_2271 = ST_Transform(geom, 2271);

UPDATE jen_barry."jen_barry.interstates" 
    SET geom_2271 = ST_Transform(geom, 2271);

UPDATE jen_barry."jen_barry.recreationareas" 
    SET geom_2271 = ST_Transform(geom, 2271);
```

### STEP 4: Create Spatial Indexes on Transformed Geometries

```sql
CREATE INDEX IF NOT EXISTS idx_counties_geom2271 
    ON jen_barry."jen_barry.counties" USING GIST(geom_2271);

CREATE INDEX IF NOT EXISTS idx_cities_geom2271 
    ON jen_barry."jen_barry.cities" USING GIST(geom_2271);

CREATE INDEX IF NOT EXISTS idx_interstates_geom2271 
    ON jen_barry."jen_barry.interstates" USING GIST(geom_2271);

CREATE INDEX IF NOT EXISTS idx_recreationareas_geom2271 
    ON jen_barry."jen_barry.recreationareas" USING GIST(geom_2271);
```

### STEP 5: Verify Transformations

```sql
SELECT 'counties' as tbl, ST_SRID(geom_2271) as new_srid, COUNT(*) as count 
FROM jen_barry."jen_barry.counties" GROUP BY ST_SRID(geom_2271)
UNION ALL
SELECT 'cities', ST_SRID(geom_2271), COUNT(*) 
FROM jen_barry."jen_barry.cities" GROUP BY ST_SRID(geom_2271)
UNION ALL
SELECT 'interstates', ST_SRID(geom_2271), COUNT(*) 
FROM jen_barry."jen_barry.interstates" GROUP BY ST_SRID(geom_2271)
UNION ALL
SELECT 'recreationareas', ST_SRID(geom_2271), COUNT(*) 
FROM jen_barry."jen_barry.recreationareas" GROUP BY ST_SRID(geom_2271);
```

**Expected Result:** All should show SRID = 2271

---

## VIEW 1: Suitable Counties (County-Level Criteria)

### Criteria:
- Greater than 500 farms (`NO_FARMS87 > 500`)
- Labor pool of at least 25,000 (`AGE_18_64 >= 25000`)
- Population density less than 150 per square mile (`POP_SQMILE < 150`)

```sql
CREATE OR REPLACE VIEW jen_barry.suitable_counties AS
SELECT 
    id,
    "NAME",
    "NO_FARMS87",
    "AGE_18_64",
    "POP_SQMILE",
    "SQ_MILES",
    geom,
    geom_2271
FROM jen_barry."jen_barry.counties"
WHERE 
    "NO_FARMS87" > 500
    AND "AGE_18_64" >= 25000
    AND "POP_SQMILE" < 150;
```

### Test Queries:

```sql
-- Count suitable counties
SELECT COUNT(*) AS count_of_suitable_counties 
FROM jen_barry.suitable_counties;
-- Expected: 11

-- List all suitable counties with criteria values
SELECT "NAME", "NO_FARMS87", "AGE_18_64", "POP_SQMILE" 
FROM jen_barry.suitable_counties 
ORDER BY "NAME";
```

**Expected Result (11 counties):**
| NAME | NO_FARMS87 | AGE_18_64 | POP_SQMILE |
|------|------------|-----------|------------|
| Bellows | 847 | 71214 | 96 |
| Center | 817 | 90058 | 112 |
| Furrow | 541 | 26221 | 69 |
| King | 635 | 27826 | 50 |
| Krim | 701 | 39439 | 130 |
| Olivier | 735 | 42909 | 114 |
| Otter | 1062 | 45413 | 73 |
| Raccoon | 628 | 25505 | 74 |
| Step | 911 | 56594 | 109 |
| Taft | 1043 | 28556 | 47 |
| Victoria | 677 | 27302 | 112 |

---

## VIEW 2: Suitable Cities (City-Level Criteria)

### Criteria:
- Located within a suitable county (spatial join using `ST_Within`)
- Low crime index (`CRIME_INDE <= 0.02`)
- Has a university or college (`UNIVERSITY = 1`)

**Expected Result:** 9 cities

```sql
CREATE OR REPLACE VIEW jen_barry.suitable_cities AS
SELECT 
    c.id AS city_id,
    c."NAME" AS city_name,
    c."POPULATION",
    c."CRIME_INDE",
    c."UNIVERSITY",
    s."NAME" AS county_name,
    s."NO_FARMS87",
    s."AGE_18_64",
    s."POP_SQMILE" AS county_pop_sqmile,
    c.geom,
    c.geom_2271
FROM jen_barry."jen_barry.cities" AS c
JOIN jen_barry.suitable_counties AS s
    ON ST_Within(c.geom_2271, s.geom_2271)
WHERE 
    c."CRIME_INDE" <= 0.02
    AND c."UNIVERSITY" = 1;
```

### Test Queries:

```sql
-- Count suitable cities
SELECT COUNT(*) AS count_of_suitable_cities 
FROM jen_barry.suitable_cities;
-- Expected: 9

-- List all suitable cities with details
SELECT city_name, county_name, "POPULATION", "CRIME_INDE", "UNIVERSITY"
FROM jen_barry.suitable_cities
ORDER BY city_name;
```

**Expected Result (9 cities):**
| city_name | county_name | POPULATION | CRIME_INDE | UNIVERSITY |
|-----------|-------------|------------|------------|------------|
| Ashton | Furrow | 15230 | 0.017 | 1 |
| Driggs | Bellows | 17580 | 0.016 | 1 |
| Frisco | King | 6200 | 0.016 | 1 |
| Geyserville | Taft | 35050 | 0.019 | 1 |
| Huntstown | Taft | 7680 | 0.014 | 1 |
| Nittanytown | Center | 85000 | 0.02 | 1 |
| Saratoga | Krim | 32015 | 0.019 | 1 |
| Shasta | Bellows | 23567 | 0.004 | 1 |
| Whitney | Step | 55600 | 0.006 | 1 |

---

## VIEW 3: Cities Near Interstate (Within 20 Miles)

### Criteria:
- Within 20 miles (105,600 feet) of an interstate

```sql
CREATE OR REPLACE VIEW jen_barry.cities_near_interstate AS
SELECT DISTINCT 
    c.city_id,
    c.city_name,
    c."POPULATION",
    c."CRIME_INDE",
    c."UNIVERSITY",
    c.county_name,
    c."NO_FARMS87",
    c."AGE_18_64",
    c.county_pop_sqmile,
    c.geom,
    c.geom_2271
FROM jen_barry.suitable_cities AS c
JOIN jen_barry."jen_barry.interstates" AS i
    ON ST_DWithin(c.geom_2271, i.geom_2271, 105600);  -- 20 miles in feet
```

### Test Queries:

```sql
-- Count cities near interstate
SELECT COUNT(*) AS count_cities_near_interstate 
FROM jen_barry.cities_near_interstate;
-- Expected: 7

-- List cities near interstate
SELECT city_name, county_name 
FROM jen_barry.cities_near_interstate
ORDER BY city_name;
```

**Expected Result (7 cities):**
| city_name | county_name |
|-----------|-------------|
| Driggs | Bellows |
| Frisco | King |
| Geyserville | Taft |
| Huntstown | Taft |
| Nittanytown | Center |
| Shasta | Bellows |
| Whitney | Step |

---

## VIEW 4: Final Candidate Cities (All Criteria Met)

### Final Criteria:
- Within 10 miles (52,800 feet) of a recreation area

**Expected Result:** 4 cities

```sql
CREATE OR REPLACE VIEW jen_barry.final_candidate_cities AS
SELECT DISTINCT 
    c.city_id,
    c.city_name,
    c."POPULATION",
    c."CRIME_INDE",
    c."UNIVERSITY",
    c.county_name,
    c."NO_FARMS87",
    c."AGE_18_64",
    c.county_pop_sqmile,
    c.geom,
    c.geom_2271
FROM jen_barry.cities_near_interstate AS c
JOIN jen_barry."jen_barry.recreationareas" AS r
    ON ST_DWithin(c.geom_2271, r.geom_2271, 52800);  -- 10 miles in feet
```

### Test Queries:

```sql
-- Count final candidates
SELECT COUNT(*) AS count_final_candidates 
FROM jen_barry.final_candidate_cities;
-- Expected: 4

-- List final candidates
SELECT city_name, county_name, "POPULATION", "CRIME_INDE", "UNIVERSITY"
FROM jen_barry.final_candidate_cities
ORDER BY city_name;
```

**Expected Result (4 cities):**
| city_name | county_name | POPULATION | CRIME_INDE | UNIVERSITY |
|-----------|-------------|------------|------------|------------|
| Driggs | Bellows | 17580 | 0.016 | 1 |
| Geyserville | Taft | 35050 | 0.019 | 1 |
| Nittanytown | Center | 85000 | 0.02 | 1 |
| Whitney | Step | 55600 | 0.006 | 1 |

---

## Comprehensive Summary Report

```sql
-- View filtering progression
SELECT 'suitable_counties' as view_name, COUNT(*) as count FROM jen_barry.suitable_counties
UNION ALL
SELECT 'suitable_cities', COUNT(*) FROM jen_barry.suitable_cities
UNION ALL
SELECT 'cities_near_interstate', COUNT(*) FROM jen_barry.cities_near_interstate
UNION ALL
SELECT 'final_candidate_cities', COUNT(*) FROM jen_barry.final_candidate_cities;
```

**Expected Result:**
| view_name | count |
|-----------|-------|
| suitable_counties | 11 |
| suitable_cities | 9 |
| cities_near_interstate | 7 |
| final_candidate_cities | 4 |

### Detailed Final Report with Criteria Verification:

```sql
SELECT 
    city_name,
    county_name,
    "POPULATION" AS city_population,
    "CRIME_INDE" AS city_crime_index,
    "UNIVERSITY" AS has_university,
    "NO_FARMS87" AS county_farms,
    "AGE_18_64" AS county_labor_force,
    county_pop_sqmile AS county_pop_density,
    -- Verify all criteria are met
    CASE WHEN "NO_FARMS87" > 500 THEN '✓' ELSE '✗' END AS "Farms>500",
    CASE WHEN "AGE_18_64" >= 25000 THEN '✓' ELSE '✗' END AS "Labor≥25K",
    CASE WHEN county_pop_sqmile < 150 THEN '✓' ELSE '✗' END AS "Density<150",
    CASE WHEN "CRIME_INDE" <= 0.02 THEN '✓' ELSE '✗' END AS "Crime≤0.02",
    CASE WHEN "UNIVERSITY" = 1 THEN '✓' ELSE '✗' END AS "Has_Univ"
FROM jen_barry.final_candidate_cities
ORDER BY city_name;
```

---

## Visualization in QGIS

### Recommended Layer Order (Bottom to Top)

| Order | Layer | Source | Style Suggestion |
|-------|-------|--------|------------------|
| 1 | All Counties | `jen_barry.counties.geom` | Light gray fill |
| 2 | Suitable Counties | `suitable_counties.geom` | Light blue fill |
| 3 | Recreation Areas | `jen_barry.recreationareas.geom` | Light green fill |
| 4 | Interstates | `jen_barry.interstates.geom` | Red lines, 2px |
| 5 | Suitable Cities | `suitable_cities.geom` | Orange circles |
| 6 | Cities Near Interstate | `cities_near_interstate.geom` | Yellow circles |
| 7 | Final Candidates | `final_candidate_cities.geom` | Red stars ⭐ |

### Layers to AVOID Adding:
- ❌ `jen_barry.cities.geom` - Shows ALL 48 cities, not filtered
- ❌ Any `geom_2271` layers - Used for calculations only

---

## Complete SQL Script (Copy-Paste Ready)

```sql
-- ============================================================================
-- JEN AND BARRY'S ICE CREAM SITE SELECTION - COMPLETE SOLUTION
-- Database: jen_barry_db
-- Schema: jen_barry
-- ============================================================================

-- ===================
-- STEP 0: Setup
-- ===================
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;

-- ===================
-- STEP 1: Verify Data
-- ===================
SELECT 'counties' as table_name, COUNT(*) as row_count FROM jen_barry."jen_barry.counties"
UNION ALL SELECT 'cities', COUNT(*) FROM jen_barry."jen_barry.cities"
UNION ALL SELECT 'interstates', COUNT(*) FROM jen_barry."jen_barry.interstates"
UNION ALL SELECT 'recreationareas', COUNT(*) FROM jen_barry."jen_barry.recreationareas";

-- ===================
-- STEP 2: Add Transformed Geometry Columns
-- ===================
ALTER TABLE jen_barry."jen_barry.counties" ADD COLUMN IF NOT EXISTS geom_2271 geometry(MultiPolygon, 2271);
ALTER TABLE jen_barry."jen_barry.cities" ADD COLUMN IF NOT EXISTS geom_2271 geometry(Point, 2271);
ALTER TABLE jen_barry."jen_barry.interstates" ADD COLUMN IF NOT EXISTS geom_2271 geometry(MultiLineString, 2271);
ALTER TABLE jen_barry."jen_barry.recreationareas" ADD COLUMN IF NOT EXISTS geom_2271 geometry(MultiPolygon, 2271);

-- ===================
-- STEP 3: Transform Geometries
-- ===================
UPDATE jen_barry."jen_barry.counties" SET geom_2271 = ST_Transform(geom, 2271);
UPDATE jen_barry."jen_barry.cities" SET geom_2271 = ST_Transform(geom, 2271);
UPDATE jen_barry."jen_barry.interstates" SET geom_2271 = ST_Transform(geom, 2271);
UPDATE jen_barry."jen_barry.recreationareas" SET geom_2271 = ST_Transform(geom, 2271);

-- ===================
-- STEP 4: Create Spatial Indexes
-- ===================
CREATE INDEX IF NOT EXISTS idx_counties_geom2271 ON jen_barry."jen_barry.counties" USING GIST(geom_2271);
CREATE INDEX IF NOT EXISTS idx_cities_geom2271 ON jen_barry."jen_barry.cities" USING GIST(geom_2271);
CREATE INDEX IF NOT EXISTS idx_interstates_geom2271 ON jen_barry."jen_barry.interstates" USING GIST(geom_2271);
CREATE INDEX IF NOT EXISTS idx_recreationareas_geom2271 ON jen_barry."jen_barry.recreationareas" USING GIST(geom_2271);

-- ===================
-- VIEW 1: Suitable Counties
-- Criteria: Farms > 500, Labor >= 25000, Density < 150
-- ===================
CREATE OR REPLACE VIEW jen_barry.suitable_counties AS
SELECT id, "NAME", "NO_FARMS87", "AGE_18_64", "POP_SQMILE", "SQ_MILES", geom, geom_2271
FROM jen_barry."jen_barry.counties"
WHERE "NO_FARMS87" > 500 AND "AGE_18_64" >= 25000 AND "POP_SQMILE" < 150;

-- Test VIEW 1
SELECT COUNT(*) AS suitable_counties FROM jen_barry.suitable_counties; -- Expected: 11

-- ===================
-- VIEW 2: Suitable Cities
-- Criteria: Within suitable county + Crime <= 0.02 + University = 1
-- ===================
CREATE OR REPLACE VIEW jen_barry.suitable_cities AS
SELECT c.id AS city_id, c."NAME" AS city_name, c."POPULATION", c."CRIME_INDE", c."UNIVERSITY",
       s."NAME" AS county_name, s."NO_FARMS87", s."AGE_18_64", s."POP_SQMILE" AS county_pop_sqmile,
       c.geom, c.geom_2271
FROM jen_barry."jen_barry.cities" AS c
JOIN jen_barry.suitable_counties AS s ON ST_Within(c.geom_2271, s.geom_2271)
WHERE c."CRIME_INDE" <= 0.02 AND c."UNIVERSITY" = 1;

-- Test VIEW 2
SELECT COUNT(*) AS suitable_cities FROM jen_barry.suitable_cities; -- Expected: 9

-- ===================
-- VIEW 3: Cities Near Interstate (within 20 miles = 105600 feet)
-- ===================
CREATE OR REPLACE VIEW jen_barry.cities_near_interstate AS
SELECT DISTINCT c.city_id, c.city_name, c."POPULATION", c."CRIME_INDE", c."UNIVERSITY",
       c.county_name, c."NO_FARMS87", c."AGE_18_64", c.county_pop_sqmile, c.geom, c.geom_2271
FROM jen_barry.suitable_cities AS c
JOIN jen_barry."jen_barry.interstates" AS i ON ST_DWithin(c.geom_2271, i.geom_2271, 105600);

-- Test VIEW 3
SELECT COUNT(*) AS cities_near_interstate FROM jen_barry.cities_near_interstate; -- Expected: 7

-- ===================
-- VIEW 4: Final Candidates (within 10 miles = 52800 feet of recreation area)
-- ===================
CREATE OR REPLACE VIEW jen_barry.final_candidate_cities AS
SELECT DISTINCT c.city_id, c.city_name, c."POPULATION", c."CRIME_INDE", c."UNIVERSITY",
       c.county_name, c."NO_FARMS87", c."AGE_18_64", c.county_pop_sqmile, c.geom, c.geom_2271
FROM jen_barry.cities_near_interstate AS c
JOIN jen_barry."jen_barry.recreationareas" AS r ON ST_DWithin(c.geom_2271, r.geom_2271, 52800);

-- Test VIEW 4
SELECT COUNT(*) AS final_candidates FROM jen_barry.final_candidate_cities; -- Expected: 4

-- ===================
-- FINAL RESULTS
-- ===================
SELECT city_name, county_name, "POPULATION", "CRIME_INDE", "UNIVERSITY"
FROM jen_barry.final_candidate_cities ORDER BY city_name;

-- Filtering progression summary
SELECT 'suitable_counties' as step, COUNT(*) as count FROM jen_barry.suitable_counties
UNION ALL SELECT 'suitable_cities', COUNT(*) FROM jen_barry.suitable_cities
UNION ALL SELECT 'cities_near_interstate', COUNT(*) FROM jen_barry.cities_near_interstate
UNION ALL SELECT 'final_candidate_cities', COUNT(*) FROM jen_barry.final_candidate_cities;
```

---

## Summary

### Final Results: 4 Candidate Cities

| City | County | Population | Crime Index | Why Selected |
|------|--------|------------|-------------|--------------|
| **Driggs** | Bellows | 17,580 | 0.016 | All criteria met ✓ |
| **Geyserville** | Taft | 35,050 | 0.019 | All criteria met ✓ |
| **Nittanytown** | Center | 85,000 | 0.020 | All criteria met ✓ |
| **Whitney** | Step | 55,600 | 0.006 | All criteria met ✓ |

### Filtering Funnel

```
Counties (43) → Suitable Counties (11) → Filter by demographics
Cities (48) → Suitable Cities (9) → Filter by crime + university
         → Cities Near Interstate (7) → Filter by 20-mile proximity
         → Final Candidates (4) → Filter by 10-mile recreation proximity
```
