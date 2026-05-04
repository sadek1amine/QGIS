#  TP 7
### Djireb Sadek Amine 

## Analysis of Green Spaces in an Urban Area  
### Using PostGIS and QGIS  



---

##  Objective  

This practical work aims to master the full spatial data processing pipeline:  
from importing OpenStreetMap shapefiles into a PostGIS database, to performing advanced spatial queries, and finally visualizing results using QGIS.

The study area is the city of **Algiers**, focusing on:  
 Identifying buildings located near green spaces.

---

##  1. Environment and Data  

### 1.1 Tools Used  

- PostgreSQL 15 with PostGIS 3.x extension  
- QGIS 3.x for cartographic visualization  
- OpenStreetMap data — Algeria (source: Geofabrik)  

---

### 1.2 Spatial Database Creation  

Creation of the database and activation of PostGIS:

```bash
createdb -U postgres green_spaces
psql -U postgres -d green_spaces -c "CREATE EXTENSION postgis;"
psql -U postgres -d green_spaces -c "SELECT PostGIS_Version();"
```

---

### 2. Importing Geographic Layers

The three OSM layers were imported using shp2pgsql, specifying SRID 4326 (WGS84) and automatically creating a spatial index (-I):

```bash
shp2pgsql -I -s 4326 ".../gis_osm_buildings_a_free_1.shp" buildings | psql -U postgres -d green_spaces
shp2pgsql -I -s 4326 ".../gis_osm_roads_free_1.shp" roads | psql -U postgres -d green_spaces
shp2pgsql -I -s 4326 ".../gis_osm_landuse_a_free_1.shp" landuse | psql -U postgres -d green_spaces
```

Verification of created tables:

```bash
psql -U postgres -d green_spaces -c "\dt"
```

---

### 3. Extraction of Green Spaces

#### 3.1 Creating the Parks Table

Green spaces are extracted from the landuse layer by filtering the classes `'park'` and `'recreation_ground'`:

```sql
CREATE TABLE parks AS
SELECT * FROM landuse
WHERE fclass IN ('park', 'recreation_ground');

CREATE INDEX parks_geom_idx ON parks USING GIST (geom);
```

#### 3.2 Parks Larger Than 1 Hectare

Geodesic area (in m²) is calculated using ST_Area with a geography cast, then converted into hectares:

```sql
SELECT name,
  ROUND((ST_Area(geom::geography) / 10000)::numeric, 2) AS surface_ha
FROM parks
WHERE ST_Area(geom::geography) > 10000
ORDER BY surface_ha DESC
LIMIT 20;
```

---

## 📍 4. Restricting to Algiers Area

To optimize performance (large national dataset), a spatial filter using a bounding box (ST_MakeEnvelope, SRID 4326) is applied:

```sql
CREATE TABLE buildings_city AS 
SELECT * FROM buildings
WHERE geom && ST_MakeEnvelope(2.85, 36.65, 3.25, 36.85, 4326);

CREATE TABLE parks_city AS 
SELECT * FROM parks
WHERE geom && ST_MakeEnvelope(2.85, 36.65, 3.25, 36.85, 4326);

CREATE TABLE roads_city AS 
SELECT * FROM roads
WHERE geom && ST_MakeEnvelope(2.85, 36.65, 3.25, 36.85, 4326);
```

| Layer          | Number of Features |
| -------------- | ------------------ |
| buildings_city | 258,032            |
| parks_city     | 208                |
| roads_city     | 45,539             |

---

##  5. Proximity Analysis: Buildings Within 100m of a Park

### 5.1 Query with Spatial Prefiltering

A spatial index prefilter (`&&`) is combined with `ST_DWithin` (metric distance via geography cast) to efficiently identify nearby buildings:

```sql
SELECT COUNT(DISTINCT b.osm_id)
FROM buildings_city b
JOIN parks_city p
  ON b.geom && ST_Expand(p.geom, 0.001)
WHERE ST_DWithin(b.geom::geography, p.geom::geography, 100);
```

### 5.2 Results

| Indicator                       | Value   |
| ------------------------------- | ------- |
| Buildings within 100m of a park | 11,546  |
| Total buildings in Algiers      | 258,032 |
| Relative share                  | ~4.5%   |

```sql
CREATE TABLE buildings_near_parks AS
SELECT DISTINCT b.*
FROM buildings_city b
JOIN parks_city p
  ON b.geom && ST_Expand(p.geom, 0.001)
WHERE ST_DWithin(b.geom::geography, p.geom::geography, 100);

CREATE INDEX buildings_near_parks_geom_idx
ON buildings_near_parks USING GIST (geom);
```

---

##  7. Visualization in QGIS

### 7.1 PostgreSQL/PostGIS Connection

The four layers were loaded from the PostGIS database:

- parks_city  
- roads_city  
- buildings_city  
- buildings_near_parks  

### 7.2 Applied Styles

| Layer                | Color      | Notes                |
| -------------------- | ---------- | -------------------- |
| parks_city           | Green      | Label: `name` field  |
| buildings_near_parks | Red        | Buildings near parks |
| buildings_city       | Light Grey | Other buildings      |
| roads_city           | Grey/Black | Road network         |

### 7.3 Final Layout

- **Title:** Urban Green Space Analysis in Algiers  
- Clear legend with all layers  
- Scale bar and North arrow  
- Park names displayed using QGIS labels  
