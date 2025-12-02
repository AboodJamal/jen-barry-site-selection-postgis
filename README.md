# Jen and Barry's Ice Cream Site Selection Project

## Project Overview

This project automates the site selection process for Jen and Barry's ice cream business using **PostGIS spatial analysis**. The goal is to identify the best locations for opening an ice cream business by applying multiple spatial and demographic criteria to filter candidate cities.

### Objective

Find optimal locations that meet all of Jen and Barry's business requirements:
- Sufficient milk production capacity (farms)
- Adequate labor pool (working-age population)
- Low crime rates
- Appropriate population density
- Proximity to educational institutions (universities)
- Access to recreation areas
- Access to major transportation routes (interstates)

---

## Original Project Requirements

> **Project 4: Jen and Barry's Site Selection in PostGIS**
>
> **Scenario:** If you took GEOG 483, the first project had you finding the best locations for "Jen and Barry" to open an ice cream business. Your task is to import the project shapefiles into a PostGIS schema and then write a series of SQL statements that automate the site selection process.
>
> **Jen and Barry's Selection Criteria:**
> - Greater than 500 farms for milk production
> - A labor pool of at least 25,000 individuals between the ages of 18 and 64 years
> - A low crime index (less than or equal to 0.02)
> - A population of less than 150 individuals per square mile
> - Located near a university or college
> - At least one recreation area within 10 miles
> - Interstate within 20 miles
>
> **Expected Results:** You should narrow the cities down to **9**, based on the county- and city-level criteria. After evaluating the interstate and recreation area criteria, that should get you down to **4 cities**.

---

## Database Information

### Database Details
| Property | Value |
|----------|-------|
| **Database Name** | `jen_barry_db` (or your chosen name) |
| **Schema** | `jen_barry` |
| **PostGIS Version** | 3.x |
| **Original SRID** | 4267 (NAD27) |
| **Projected SRID** | 2271 (PA State Plane South, feet) |

### Source Data Files
Located in the `data/` directory:

| File | Description |
|------|-------------|
| `counties.shp` | County boundaries and demographics |
| `cities.shp` | City locations and characteristics |
| `interstates.shp` | Interstate highway network |
| `recareas.shp` | Recreation areas and parks |

---

## Database Schema

### Table 1: `jen_barry."jen_barry.counties"`

| Property | Value |
|----------|-------|
| **Geometry Type** | MULTIPOLYGON |
| **SRID** | 4267 (NAD27) |
| **Row Count** | **43** |
| **Transformed Column** | `geom_2271` (EPSG:2271) |

#### Columns

| Column Name | Data Type | Description | Value Range |
|-------------|-----------|-------------|-------------|
| `id` | integer | Primary key (auto-increment) | 1, 2, 3... |
| `geom` | geometry | Original geometry (EPSG:4267) | MULTIPOLYGON |
| `AREA` | numeric | County area | - |
| `PERIMETER` | numeric | County perimeter | - |
| `NAME` | character varying | County name | 43 unique values |
| `POP1990` | numeric | Population in 1990 | - |
| `AGE_18_64` | numeric | Working-age population (18-64) | **2,715 - 255,417** |
| `NO_FARMS87` | numeric | Number of farms (1987) | **25 - 4,775** |
| `POP_SQMILE` | bigint | Population per square mile | **11 - 837** |
| `SQ_MILES` | numeric | Total square miles | - |
| `GAVPRIMARY` | bigint | Geographic area value | - |
| `X` | double precision | Centroid X coordinate | - |
| `Y` | double precision | Centroid Y coordinate | - |
| `geom_2271` | geometry | Transformed geometry (EPSG:2271) | MULTIPOLYGON |

#### All County Names (43)
```
Bellows, Bolivar, Box, Briggs, Bright, Center, Crab, Deer, Forest, Forge,
Furrow, Galway, Gnome, Hail, Hook, Jefferson, Jones, King, Krim, Law,
Legg, Lister, Mars, Medici, Melville, Midas, Murphy, Olivier, Otter, Partick,
Phoenix, Pollock, Raccoon, Revere, Shoebill, Step, Taft, Thompson, Turtle,
Viceroy, Victoria, Virgo, Young
```

---

### Table 2: `jen_barry."jen_barry.cities"`

| Property | Value |
|----------|-------|
| **Geometry Type** | POINT |
| **SRID** | 4267 (NAD27) |
| **Row Count** | **48** |
| **Transformed Column** | `geom_2271` (EPSG:2271) |

#### Columns

| Column Name | Data Type | Description | Value Range |
|-------------|-----------|-------------|-------------|
| `id` | integer | Primary key (auto-increment) | 1, 2, 3... |
| `geom` | geometry | Original geometry (EPSG:4267) | POINT |
| `ID` | integer | City identifier | - |
| `NAME` | character varying | City name | 48 unique values |
| `POPULATION` | numeric | Total population | - |
| `TOTAL_CRIM` | numeric | Total crime count | - |
| `CRIME_INDE` | numeric | Crime index (lower = safer) | **0 - 0.098** |
| `UNIVERSITY` | numeric | Has university (1=Yes, 0=No) | **0, 1** |
| `geom_2271` | geometry | Transformed geometry (EPSG:2271) | POINT |

---

### Table 3: `jen_barry."jen_barry.interstates"`

| Property | Value |
|----------|-------|
| **Geometry Type** | MULTILINESTRING |
| **SRID** | 4267 (NAD27) |
| **Row Count** | **7** |
| **Transformed Column** | `geom_2271` (EPSG:2271) |

#### Columns

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| `id` | integer | Primary key (auto-increment) |
| `geom` | geometry | Original geometry (EPSG:4267) |
| `FNODE_` | bigint | From node identifier |
| `TNODE_` | bigint | To node identifier |
| `LPOLY_` | bigint | Left polygon identifier |
| `RPOLY_` | bigint | Right polygon identifier |
| `LENGTH` | numeric | Segment length |
| `INTERSTATE` | bigint | Interstate route number |
| `INTERSTATE_1` | bigint | Alternate identifier |
| `NAME` | character varying | Route name |
| `TYPE` | character varying | Road type |
| `geom_2271` | geometry | Transformed geometry (EPSG:2271) |

#### Route Names (7)
| Route |
|-------|
| I-40 |
| I-50 |
| I-55 |
| I-99 |
| SR-44 |
| SR-97 |

---

### Table 4: `jen_barry."jen_barry.recreationareas"`

| Property | Value |
|----------|-------|
| **Geometry Type** | MULTIPOLYGON |
| **SRID** | 4267 (NAD27) |
| **Row Count** | **110** |
| **Transformed Column** | `geom_2271` (EPSG:2271) |

#### Columns

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| `id` | integer | Primary key (auto-increment) |
| `geom` | geometry | Original geometry (EPSG:4267) |
| `AREA` | double precision | Area of recreation area |
| `PERIMETER` | double precision | Perimeter |
| `NCREC_` | bigint | Recreation area identifier |
| `NCREC_ID` | bigint | Alternate ID |
| `geom_2271` | geometry | Transformed geometry (EPSG:2271) |

---

## Selection Criteria

### County-Level Criteria

| # | Criterion | SQL Condition | Reason |
|---|-----------|---------------|--------|
| 1 | Greater than 500 farms | `NO_FARMS87 > 500` | Sufficient milk production capacity |
| 2 | Labor pool ≥ 25,000 | `AGE_18_64 >= 25000` | Adequate workforce (ages 18-64) |
| 3 | Population density < 150/sq mi | `POP_SQMILE < 150` | Prefer less densely populated areas |

### City-Level Criteria

| # | Criterion | SQL Condition | Reason |
|---|-----------|---------------|--------|
| 4 | Low crime index | `CRIME_INDE <= 0.02` | Safety for business location |
| 5 | Near university/college | `UNIVERSITY = 1` | Access to student population |

### Spatial Criteria

| # | Criterion | SQL Function | Distance |
|---|-----------|--------------|----------|
| 6 | Near interstate | `ST_DWithin()` | Within 20 miles (105,600 feet) |
| 7 | Near recreation area | `ST_DWithin()` | Within 10 miles (52,800 feet) |

---

## Results Summary

### Filtering Progression

| Step | View Name | Count | Criteria Applied |
|------|-----------|-------|------------------|
| 1 | `suitable_counties` | **11** | Farms > 500, Labor ≥ 25K, Density < 150 |
| 2 | `suitable_cities` | **9** | Within suitable county + Crime ≤ 0.02 + University = 1 |
| 3 | `cities_near_interstate` | **7** | Within 20 miles of interstate |
| 4 | `final_candidate_cities` | **4** | Within 10 miles of recreation area |

### Suitable Counties (11)

| County Name | Farms (NO_FARMS87) | Labor Pool (AGE_18_64) | Pop Density (POP_SQMILE) |
|-------------|-------------------|------------------------|--------------------------|
| Bellows | 847 | 71,214 | 96 |
| Center | 817 | 90,058 | 112 |
| Furrow | 541 | 26,221 | 69 |
| King | 635 | 27,826 | 50 |
| Krim | 701 | 39,439 | 130 |
| Olivier | 735 | 42,909 | 114 |
| Otter | 1,062 | 45,413 | 73 |
| Raccoon | 628 | 25,505 | 74 |
| Step | 911 | 56,594 | 109 |
| Taft | 1,043 | 28,556 | 47 |
| Victoria | 677 | 27,302 | 112 |

### Suitable Cities (9)

| City Name | County | Population | Crime Index | Has University |
|-----------|--------|------------|-------------|----------------|
| Ashton | Furrow | 15,230 | 0.017 | Yes |
| Driggs | Bellows | 17,580 | 0.016 | Yes |
| Frisco | King | 6,200 | 0.016 | Yes |
| Geyserville | Taft | 35,050 | 0.019 | Yes |
| Huntstown | Taft | 7,680 | 0.014 | Yes |
| Nittanytown | Center | 85,000 | 0.020 | Yes |
| Saratoga | Krim | 32,015 | 0.019 | Yes |
| Shasta | Bellows | 23,567 | 0.004 | Yes |
| Whitney | Step | 55,600 | 0.006 | Yes |

### 🏆 Final Candidate Cities (4)

| City Name | County | Population | Crime Index | Has University |
|-----------|--------|------------|-------------|----------------|
| **Driggs** | Bellows | 17,580 | 0.016 | ✓ |
| **Geyserville** | Taft | 35,050 | 0.019 | ✓ |
| **Nittanytown** | Center | 85,000 | 0.020 | ✓ |
| **Whitney** | Step | 55,600 | 0.006 | ✓ |

---

## Technical Implementation

### Coordinate System Transformation

| Property | Source | Target |
|----------|--------|--------|
| **SRID** | 4267 (NAD27) | 2271 (PA State Plane South) |
| **Type** | Geographic | Projected |
| **Units** | Degrees | Feet |
| **Purpose** | Storage | Distance calculations |

**Distance Conversions:**
- 10 miles = 52,800 feet
- 20 miles = 105,600 feet

### Spatial Functions Used

| Function | Purpose | Example |
|----------|---------|---------|
| `ST_Transform()` | Convert coordinate systems | `ST_Transform(geom, 2271)` |
| `ST_Within()` | Point-in-polygon test | City within county |
| `ST_DWithin()` | Distance-based filter | Within X feet of feature |
| `ST_SRID()` | Get coordinate system | Verify SRID |
| `GeometryType()` | Get geometry type | Verify data type |

### Performance Optimization

- **GIST Spatial Indexes** on all `geom_2271` columns
- **Views** for progressive filtering (debuggable pipeline)
- **DISTINCT** to prevent duplicate results from spatial joins

---

## Project Structure

```
HW1/
├── Project4/                    # Input shapefiles
│   ├── counties.shp/.dbf/.prj/.shx
│   ├── cities.shp/.dbf/.prj/.shx
│   ├── interstates.shp/.dbf/.prj/.shx
│   └── recareas.shp/.dbf/.prj/.shx
├── solution.md                  # Complete SQL solution
├── README.md                    # This file
└── model-builder.html           # Visual workflow diagram
```

---

## Getting Started

### Prerequisites
- PostgreSQL 12+ with PostGIS 3.x extension
- QGIS 3.x (for data import and visualization)
- pgAdmin 4 (optional, for database management)

### Setup Steps

1. **Create Database**
   ```sql
   CREATE DATABASE jen_barry_db;
   \c jen_barry_db
   CREATE EXTENSION postgis;
   CREATE EXTENSION postgis_topology;
   CREATE SCHEMA jen_barry;
   ```

2. **Import Shapefiles via QGIS**
   - Open QGIS → Database → DB Manager
   - Connect to PostgreSQL database
   - Import each shapefile to `jen_barry` schema
   - Set SRID: 4267 (NAD27)
   - Check "Create spatial index"

3. **Run SQL Solution**
   - Open `solution.md` in pgAdmin
   - Execute statements sequentially
   - Verify results at each step

4. **Visualize in QGIS**
   - Add views as layers
   - Style appropriately
   - Analyze spatial patterns

---

## Solution File

For the complete SQL solution with detailed comments and test queries, see **[solution.md](./solution.md)**.

---

## Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        JEN & BARRY'S SITE SELECTION                         │
└─────────────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
    │   COUNTIES   │         │    CITIES    │         │  INTERSTATES │
    │   (43 rows)  │         │   (48 rows)  │         │   (7 routes) │
    └──────┬───────┘         └──────┬───────┘         └──────┬───────┘
           │                        │                        │
           ▼                        │                        │
    ┌──────────────┐                │                        │
    │   WHERE:     │                │                        │
    │ • Farms>500  │                │                        │
    │ • Labor≥25K  │                │                        │
    │ • Density<150│                │                        │
    └──────┬───────┘                │                        │
           │                        │                        │
           ▼                        │                        │
    ┌──────────────┐                │                        │
    │   SUITABLE   │                │                        │
    │   COUNTIES   │◄───────────────┤                        │
    │  (11 rows)   │   ST_Within    │                        │
    └──────┬───────┘                │                        │
           │                        ▼                        │
           │                 ┌──────────────┐                │
           │                 │   WHERE:     │                │
           │                 │ • Crime≤0.02 │                │
           │                 │ • Univ = 1   │                │
           │                 └──────┬───────┘                │
           │                        │                        │
           │                        ▼                        │
           │                 ┌──────────────┐                │
           │                 │   SUITABLE   │◄───────────────┤
           │                 │    CITIES    │  ST_DWithin    │
           │                 │   (9 rows)   │   (20 miles)   │
           │                 └──────┬───────┘                │
           │                        │                        │
           │                        ▼                        │
           │                 ┌──────────────┐    ┌──────────────┐
           │                 │ CITIES NEAR  │    │  RECREATION  │
           │                 │  INTERSTATE  │    │    AREAS     │
           │                 │   (7 rows)   │    │  (110 rows)  │
           │                 └──────┬───────┘    └──────┬───────┘
           │                        │                   │
           │                        ▼                   │
           │                 ┌──────────────┐           │
           │                 │  ST_DWithin  │◄──────────┘
           │                 │  (10 miles)  │
           │                 └──────┬───────┘
           │                        │
           │                        ▼
           │                 ┌──────────────┐
           │                 │    FINAL     │
           │                 │  CANDIDATES  │
           │                 │   (4 rows)   │
           │                 │  ⭐ RESULT   │
           └─────────────────┴──────────────┘
```

---

## Project Structure

```
HW1/
├── README.md                    # Project overview and documentation
├── solution.md                  # Complete SQL solution with step-by-step guide
├── jen-barry-site-selection.qgz # QGIS project file
├── data/                        # Source shapefiles
│   ├── counties.shp (+ .dbf, .prj, .shx)
│   ├── cities.shp (+ .dbf, .prj, .shx)
│   ├── interstates.shp (+ .dbf, .prj, .shx)
│   └── recareas.shp (+ .dbf, .prj, .shx)
├── docs/                        # Documentation and resources
│   ├── model-builder.html               # Interactive workflow visualization
│   ├── model-builder.png                # Workflow diagram screenshot
│   ├── solution-explanation.md          # Detailed step-by-step explanation
│   ├── coordinate-transformation-alternatives.md  # Transform vs geography discussion
│   ├── original-solution.docx           # Original solution document
│   └── lesson-4-advanced-postgres-postgis-topics.pdf  # Course reference material
└── visual-outputs/              # QGIS screenshots
    ├── canvas-layers-names.png
    └── canvas-layers-visual.png
```

---

## Additional Documentation

| Document | Description |
|----------|-------------|
| [solution.md](solution.md) | Complete SQL solution with all queries and expected results |
| [docs/model-builder.html](docs/model-builder.html) | Interactive workflow visualization diagram |
| [docs/solution-explanation.md](docs/solution-explanation.md) | Detailed explanation of each step and why it's needed |
| [docs/coordinate-transformation-alternatives.md](docs/coordinate-transformation-alternatives.md) | Discussion on using geography type vs coordinate transformation |

---

## Key Learnings

1. **Coordinate System Management** - Transform from geographic (degrees) to projected (feet) for accurate distance calculations
2. **Progressive Filtering** - Build complex queries through cascading views
3. **Spatial Joins** - Combine attribute and spatial criteria using `ST_Within()` and `ST_DWithin()`
4. **Performance Optimization** - Create GIST indexes on geometry columns
5. **Data Validation** - Verify transformations and results at each step

---

## Author

**Spatial Data Analysis - Homework 1**  
Project 4: Jen and Barry's Site Selection using PostGIS
