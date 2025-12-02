# Jen and Barry's Ice Cream Site Selection - Solution

## Overview
This document contains the complete SQL solution for automating Jen and Barry's ice cream business site selection process using PostGIS spatial analysis.

---

## Solution Steps

### STEP 0: Enable PostGIS Extensions
```sql
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;
```

---

### STEP 1: Create Transformed Geometry Columns (EPSG:2271 - PA State Plane South)

**Why EPSG:2271?** Pennsylvania State Plane South (NAD83) uses feet as units, which allows us to measure distances accurately in feet.

**Distance conversions:**
- 10 miles = 52,800 feet
- 20 miles = 105,600 feet

```sql
-- Add new geometry columns in EPSG:2271 (PA State Plane South, feet)
ALTER TABLE jen_barry."jen_barry.counties" 
    ADD COLUMN geom_2271 geometry(MultiPolygon, 2271);

ALTER TABLE jen_barry."jen_barry.cities" 
    ADD COLUMN geom_2271 geometry(Point, 2271);

ALTER TABLE jen_barry."jen_barry.interstates" 
    ADD COLUMN geom_2271 geometry(MultiLineString, 2271);

ALTER TABLE jen_barry."jen_barry.recreationareas" 
    ADD COLUMN geom_2271 geometry(MultiPolygon, 2271);
```

---

### STEP 2: Transform Geometries from EPSG:4267 (NAD27) to EPSG:2271

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

---

### STEP 3: Create Spatial Indexes on Transformed Geometries

**Note:** Spatial indexes significantly improve query performance for spatial operations.

```sql
CREATE INDEX idx_counties_geom2271 
    ON jen_barry."jen_barry.counties" USING GIST(geom_2271);

CREATE INDEX idx_cities_geom2271 
    ON jen_barry."jen_barry.cities" USING GIST(geom_2271);

CREATE INDEX idx_interstates_geom2271 
    ON jen_barry."jen_barry.interstates" USING GIST(geom_2271);

CREATE INDEX idx_recreationareas_geom2271 
    ON jen_barry."jen_barry.recreationareas" USING GIST(geom_2271);
```

---

### STEP 4: Verify Transformed Geometries

Check that all geometries were transformed correctly:

```sql
SELECT 
    'counties' AS tbl, 
    GeometryType(geom_2271) AS geom_type, 
    ST_SRID(geom_2271) AS srid, 
    COUNT(*) AS count 
FROM jen_barry."jen_barry.counties"
GROUP BY GeometryType(geom_2271), ST_SRID(geom_2271)

UNION ALL

SELECT 
    'cities' AS tbl, 
    GeometryType(geom_2271), 
    ST_SRID(geom_2271), 
    COUNT(*) 
FROM jen_barry."jen_barry.cities"
GROUP BY GeometryType(geom_2271), ST_SRID(geom_2271)

UNION ALL

SELECT 
    'interstates' AS tbl, 
    GeometryType(geom_2271), 
    ST_SRID(geom_2271), 
    COUNT(*) 
FROM jen_barry."jen_barry.interstates"
GROUP BY GeometryType(geom_2271), ST_SRID(geom_2271)

UNION ALL

SELECT 
    'recreationareas' AS tbl, 
    GeometryType(geom_2271), 
    ST_SRID(geom_2271), 
    COUNT(*) 
FROM jen_barry."jen_barry.recreationareas"
GROUP BY GeometryType(geom_2271), ST_SRID(geom_2271);
```

---

## VIEW 1: Suitable Counties (County-Level Criteria)

**Criteria:**
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

**Test Query:**
```sql
SELECT COUNT(*) AS count_of_suitable_counties 
FROM jen_barry.suitable_counties;

SELECT "NAME", "NO_FARMS87", "AGE_18_64", "POP_SQMILE" 
FROM jen_barry.suitable_counties 
ORDER BY "NAME";
```

---

## VIEW 2: Suitable Cities (City-Level Criteria within Suitable Counties)

**Criteria:**
- Located within a suitable county (spatial join)
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

**Test Query:**
```sql
SELECT COUNT(*) AS count_of_suitable_cities 
FROM jen_barry.suitable_cities;

SELECT city_name, county_name, "POPULATION", "CRIME_INDE", "UNIVERSITY"
FROM jen_barry.suitable_cities
ORDER BY city_name;
```

---

## VIEW 3: Cities Near Interstate (Within 20 Miles)

**Criteria:**
- Within 20 miles (105,600 feet) of an interstate

**Note:** Uses `ST_DWithin` for efficient distance-based spatial join.

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

**Test Query:**
```sql
SELECT COUNT(*) AS count_cities_near_interstate 
FROM jen_barry.cities_near_interstate;

SELECT city_name, county_name 
FROM jen_barry.cities_near_interstate
ORDER BY city_name;
```

---

## VIEW 4: Final Candidate Cities (All Criteria Met)

**Final Criteria:**
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

**Test Query:**
```sql
SELECT COUNT(*) AS count_final_candidates 
FROM jen_barry.final_candidate_cities;

SELECT city_name, county_name 
FROM jen_barry.final_candidate_cities
ORDER BY city_name;
```

---

## BONUS: Comprehensive Summary Report

Detailed view of final candidates with all criteria verification:

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

### Add Layers to Map Canvas

#### 1. Base Layers First
In Browser Panel, expand: `PostgreSQL → jen_barry_db → jen_barry schema`

Drag these to your map canvas:
- ✅ `jen_barry.counties.geom` - All counties (gray background)
- ✅ `suitable_counties.geom` - Suitable counties (light blue)

#### 2. Feature Layers (Middle)
- ✅ `jen_barry.recreationareas.geom` - Recreation areas (light green)
- ✅ `jen_barry.interstates.geom` - Highways (red lines)

#### 3. City Layers (Top - Most Important)
- ✅ `suitable_cities.geom` - 9 cities (orange circles)
- ✅ `cities_near_interstate.geom` - Cities near highways (yellow circles)
- ✅ `final_candidate_cities.geom` - 4 FINAL CITIES (red stars ⭐)

### Layers You Should NOT Add:
- ❌ `jen_barry.cities.geom` - (This shows ALL cities, not just suitable ones)
- ❌ Any `geom_2271` layers - (These are for calculations only)

---

## Key Notes

1. **Coordinate System Transformation:** The original data is in EPSG:4267 (NAD27), but we transform to EPSG:2271 (PA State Plane South) to work with feet as units for accurate distance measurements.

2. **Spatial Indexes:** Creating GIST indexes on the transformed geometries significantly improves query performance for spatial operations like `ST_DWithin` and `ST_Within`.

3. **Progressive Filtering:** The solution uses a cascading view approach:
   - First filter counties by demographic criteria
   - Then filter cities within those counties by crime and university criteria
   - Then filter by proximity to interstates
   - Finally filter by proximity to recreation areas

4. **Distance Calculations:** All distance-based queries use the transformed geometry (`geom_2271`) in feet, making it easy to convert miles to feet (1 mile = 5,280 feet).

