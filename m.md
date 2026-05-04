#  Advanced Geospatial Analysis  
## Green Space Accessibility in an Urban Area  

**Course:** M1 Data Science & AI (M1DSAI)  

---

##  Executive Summary  

This project demonstrates an end-to-end spatial data science pipeline. By integrating OpenStreetMap (OSM) data into a PostGIS environment, we quantified the accessibility of green spaces for residents of Algiers.  

The analysis reveals that only **4.5%** of the city's **258,032 buildings** are within a **100-meter "walkable" distance** of a public park.

---

##  2. Infrastructure & Data Engineering  

### 2.1 Optimized Data Ingestion  

Instead of standard imports, we utilized spatial indexing during the `shp2pgsql` process to ensure immediate query readiness.

- **Database:** PostgreSQL 15 + PostGIS 3.x  
- **SRID:** 4326 (WGS84)  
- **Optimization:** Automatic GIST index creation (`-I` flag)  

---

### 2.2 Materialized Views for Performance  

To handle Algiers' large dataset (250k+ entities), we isolated the study area using a spatial envelope (`ST_MakeEnvelope`).

```sql
-- Creating a filtered, high-performance view for Algiers City
CREATE MATERIALIZED VIEW algiers_urban_core AS
SELECT * FROM buildings
WHERE geom && ST_MakeEnvelope(2.85, 36.65, 3.25, 36.85, 4326);

CREATE INDEX idx_algiers_geom 
ON algiers_urban_core USING GIST (geom);
