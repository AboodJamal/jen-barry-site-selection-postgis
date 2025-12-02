# Jen and Barry's Ice Cream Site Selection Project

## Project Overview

This project automates the site selection process for Jen and Barry's ice cream business using **PostGIS spatial analysis**. The goal is to identify the best locations for opening an ice cream business by applying multiple spatial and demographic criteria to filter candidate cities.

### Objective

Find optimal locations that meet all of Jen and Barry's business requirements:
- Sufficient milk production capacity
- Adequate labor pool
- Low crime rates
- Appropriate population density
- Proximity to educational institutions
- Access to recreation areas
- Access to major transportation routes

---

## Project Scenario

Jen and Barry are looking to open an ice cream business and need to find the best locations based on specific criteria. The selection process involves analyzing county-level demographics, city-level characteristics, and spatial relationships with infrastructure features like interstates and recreation areas.

---

## Data Structure

### Input Shapefiles

The project uses four main spatial datasets:

#### 1. **Counties** (`counties.shp`)
- **Geometry Type:** MultiPolygon
- **Coordinate System:** EPSG:4267 (NAD27)
- **Key Attributes:**
  - `NAME` - County name
  - `NO_FARMS87` - Number of farms (for milk production)
  - `AGE_18_64` - Population aged 18-64 (labor pool)
  - `POP_SQMILE` - Population per square mile
  - `SQ_MILES` - Total square miles

#### 2. **Cities** (`cities.shp`)
- **Geometry Type:** Point
- **Coordinate System:** EPSG:4267 (NAD27)
- **Key Attributes:**
  - `NAME` - City name
  - `POPULATION` - Total population
  - `CRIME_INDE` - Crime index (lower is better)
  - `UNIVERSITY` - Binary flag (1 = has university/college, 0 = none)

#### 3. **Interstates** (`interstates.shp`)
- **Geometry Type:** MultiLineString
- **Coordinate System:** EPSG:4267 (NAD27)
- **Purpose:** Major highway network for transportation access

#### 4. **Recreation Areas** (`recareas.shp`)
- **Geometry Type:** MultiPolygon
- **Coordinate System:** EPSG:4267 (NAD27)
- **Purpose:** Recreational facilities and parks

---

## Selection Criteria

### County-Level Criteria
1. **Greater than 500 farms** (`NO_FARMS87 > 500`)
   - Ensures sufficient milk production capacity
2. **Labor pool of at least 25,000** (`AGE_18_64 >= 25000`)
   - Adequate workforce aged 18-64 years
3. **Population density less than 150 per square mile** (`POP_SQMILE < 150`)
   - Prefers less densely populated areas

### City-Level Criteria
4. **Low crime index** (`CRIME_INDE <= 0.02`)
   - Safety requirement for business location
5. **Located near a university or college** (`UNIVERSITY = 1`)
   - Access to student population and educational community

### Spatial Criteria
6. **At least one recreation area within 10 miles**
   - Proximity to recreational facilities (family-friendly locations)
7. **Interstate within 20 miles**
   - Access to major transportation routes

---

## Solution Approach

### Methodology

The solution uses a **progressive filtering approach** with PostGIS spatial functions:

1. **Data Preparation**
   - Import shapefiles into PostGIS database
   - Transform geometries from EPSG:4267 (NAD27) to EPSG:2271 (PA State Plane South)
   - Create spatial indexes for performance optimization

2. **County-Level Filtering**
   - Apply demographic criteria (farms, labor pool, population density)
   - Create view of suitable counties

3. **City-Level Filtering**
   - Spatial join cities with suitable counties
   - Apply crime and university criteria
   - **Result: 9 candidate cities**

4. **Spatial Proximity Filtering**
   - Filter cities within 20 miles of interstates
   - Filter cities within 10 miles of recreation areas
   - **Final Result: 4 optimal cities**

### Key Technical Components

#### Coordinate System Transformation
- **Source:** EPSG:4267 (NAD27) - Geographic coordinate system
- **Target:** EPSG:2271 (PA State Plane South) - Projected coordinate system in feet
- **Why?** Projected coordinate systems allow accurate distance measurements in linear units (feet/meters) rather than degrees

#### Spatial Functions Used
- `ST_Transform()` - Coordinate system transformation
- `ST_Within()` - Point-in-polygon spatial join
- `ST_DWithin()` - Distance-based spatial join (within specified distance)
- `ST_SRID()` - Get spatial reference system identifier
- `GeometryType()` - Get geometry type

#### Performance Optimization
- **GIST Spatial Indexes:** Created on transformed geometries to speed up spatial queries
- **Views:** Used to organize the filtering process and make queries reusable

---

## Expected Results

### Filtering Progression

1. **Suitable Counties:** Multiple counties meeting demographic criteria
2. **Suitable Cities:** **9 cities** meeting county and city-level criteria
3. **Cities Near Interstate:** Subset of 9 cities within 20 miles of interstates
4. **Final Candidate Cities:** **4 cities** meeting all criteria including recreation area proximity

---

## Project Structure

```
HW1/
├── Project4/                    # Input shapefiles
│   ├── counties.shp
│   ├── cities.shp
│   ├── interstates.shp
│   └── recareas.shp
├── solution.md                  # Complete SQL solution with notes
├── README.md                    # This file
└── fullHWSol.docx              # Original solution document
```

---

## Getting Started

### Prerequisites
- PostgreSQL with PostGIS extension
- QGIS (for data import and visualization)
- pgAdmin (optional, for database management)

### Setup Steps

1. **Create Database**
   - Create a new PostgreSQL database (e.g., `HW1_ice-cream`)
   - Enable PostGIS extension

2. **Import Shapefiles**
   - Use QGIS DB Manager to import all shapefiles
   - Import into schema: `jen_barry`
   - Set SRID: 4267 (NAD27)
   - Create spatial indexes

3. **Run SQL Solution**
   - Execute the SQL statements from `solution.md`
   - Follow the steps sequentially
   - Verify results at each stage

4. **Visualize Results**
   - Add views to QGIS map canvas
   - Style layers appropriately
   - Analyze spatial patterns

---

## Solution File

For the complete SQL solution with detailed comments and test queries, see **[solution.md](./solution.md)**.

---

## Key Learnings

1. **Spatial Data Management:** Working with multiple coordinate systems and understanding when to use projected vs. geographic coordinate systems
2. **Spatial Analysis:** Using PostGIS functions for proximity analysis and spatial joins
3. **Query Optimization:** Creating spatial indexes to improve query performance
4. **Progressive Filtering:** Building complex queries through cascading views
5. **Data Integration:** Combining demographic and spatial criteria for site selection

---

## Notes

- All distance measurements are performed in the projected coordinate system (EPSG:2271) using feet as units
- The solution uses views to organize the filtering process, making it easy to inspect intermediate results
- Spatial indexes are critical for performance when working with large datasets
- The final result identifies 4 cities that meet all selection criteria

---

## Author

Spatial Data Analysis - Homework 1  
Project 4: Jen and Barry's Site Selection in PostGIS

