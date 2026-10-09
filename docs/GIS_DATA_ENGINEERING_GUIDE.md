# 🗺️ Geospatial Data Engineering & Spatial Pipeline Architecture

Comprehensive technical guide and system specification for high-throughput spatial data processing, coordinate transformations, and database indexing within the TerraScale platform.

---

## Table of Contents
1. [Core Concept: What Makes Spatial Data Unique?](#1-core-concept-what-makes-spatial-data-unique)
2. [Coordinate Systems & Projections (The #1 GIS Gotcha)](#2-coordinate-systems--projections-the-1-gis-gotcha)
3. [The Python Geospatial Stack: Layer by Layer](#3-the-python-geospatial-stack-layer-by-layer)
   - [Shapely: The Geometric Engine](#a-shapely-the-geometry-engine)
   - [Pandas: The Tabular Data Engine](#b-pandas-the-tabular-data-engine)
   - [GeoPandas: The Bridge Between Tables and Space](#c-geopandas-the-bridge-between-tables-and-space)
   - [NumPy: Vectorized Math & Realistic Distributions](#d-numpy-vectorized-math--distributions)
4. [The Spatial Database Layer: PostGIS & GiST Indexing](#4-the-spatial-database-layer-postgis--gist-indexing)
5. [The ETL Pipeline Anatomy (Code-by-Code Explanation)](#5-the-etl-pipeline-anatomy)
6. [Quick Reference Glossary of Methods & Parameters](#6-quick-reference-glossary)

---

## 1. Core Concept: What Makes Spatial Data Unique?

In standard web applications, data is **one-dimensional**:
- IDs, emails, prices, timestamps.
- Sorting is straightforward: $1 < 2 < 3$, or alphabetical $A \rightarrow Z$.

In GIS (Geographic Information Systems), spatial data is **multi-dimensional** and represents physical shapes on the surface of the Earth.

### The 4 Basic Geometry Primitives
Every vector map in the world is composed of four building blocks:

1. **Point**: A single coordinate pair representing a discrete location.
   - *Example:* A fire hydrant, a tree, a GPS ping `(x, y)`.
2. **LineString**: An ordered list of connected points forming a path.
   - *Example:* A road, a pipeline, a subway route `[(x1, y1), (x2, y2), ...]`.
3. **Polygon**: A closed ring of at least 4 coordinate vertices where the first and last point are identical, enclosing an area.
   - *Example:* A building footprint, a municipal tax lot parcel, a lake.
4. **MultiPolygon**: A collection of multiple separate polygons treated as a single entity.
   - *Example:* The state of Hawaii (multiple separate islands, but one state).

### Formats for Storing Geometries:
- **GeoJSON**: Human-readable JSON containing `"type": "Feature"`, `"geometry"`, and `"properties"`.
- **WKT (Well-Known Text)**: Text representation like `POLYGON((-74.0 40.7, -73.9 40.7, ...))`.
- **WKB (Well-Known Binary)**: Packed binary format stored inside PostgreSQL/PostGIS for speed and disk compression.

---

## 2. Coordinate Systems & Projections (The #1 GIS Gotcha)

### The "Orange Peel" Problem
The Earth is an irregular 3-dimensional ellipsoid. Your computer screen is a flat 2-dimensional plane.
Just like you cannot flatten an orange peel onto a table without tearing or stretching it, **you cannot flatten the Earth onto a flat map without distorting shape, area, or distance.**

A **CRS (Coordinate Reference System)** defines the mathematical rules for how coordinates map to real locations. Every CRS has a standard **EPSG code** (European Petroleum Survey Group).

### The Three EPSG Codes You Must Know:

| EPSG Code | Name | Units | What it is used for |
|---|---|---|---|
| **EPSG:4326** | **WGS 84** | **Degrees** ($\text{Longitude}, \text{Latitude}$) | Global GPS standard. Best for storing geometries in databases. |
| **EPSG:3857** | **Web Mercator** | **Meters** ($X, Y$) | The web map standard (MapLibre, Mapbox, Google Maps). Projects the world onto a 2D square. |
| **EPSG:2263** | **State Plane (NY Long Island)** | **Feet** ($X, Y$) | Local surveyor grid used by NYC government agencies for millimeter-level engineering accuracy. |

### The Critical Difference: `set_crs()` vs `to_crs()`
A fundamental architectural distinction in spatial data pipelines:

```python
# ❌ INCORRECT USE CASE FOR set_crs:
# If data is in NY State Plane (feet) and you do:
gdf = gdf.set_crs("EPSG:4326")
# Result: Disaster! You just told the computer that 1,000,000 feet is 1,000,000 degrees!
# The coordinates did not change; only the metadata tag changed.
```

```python
# ✅ CORRECT USE:
# to_crs() performs mathematical trigonometry on every coordinate vertex:
gdf = gdf.to_crs("EPSG:4326")
# Result: Converts coordinate (985000 ft, 210000 ft) into (-73.985°, 40.748°).
```

* **`set_crs()`**: Applies a label when the data arrived with *no* coordinate system tag.
* **`to_crs()`**: Recalculates the actual coordinate numbers into a new projection.

---

## 3. The Python Geospatial Stack: Layer by Layer

```
┌─────────────────────────────────────────────────────────────┐
│                 GeoPandas (GeoDataFrame)                    │
│      Combines Tabular Data + Geometry Operations in RAM     │
└───────────────┬─────────────────────────────┬───────────────┘
                │                             │
                ▼                             ▼
┌───────────────────────────────┐ ┌───────────────────────────┐
│     Pandas (DataFrames)       │ │     Shapely (Geometries)  │
│  Fast tabular attributes,     │ │  Point, Polygon, box()    │
│  filtering, column operations │ │  C++ GEOS wrapper         │
└───────────────────────────────┘ └───────────────────────────┘
                │                             │
                └───────────────┬─────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────┐
│               NumPy (Vectorized C-Arrays)                   │
│      Underlying high-speed memory blocks & array math       │
└─────────────────────────────────────────────────────────────┘
```

---

### A. Shapely: The Geometry Engine

Shapely is a Python wrapper around **GEOS** (Geometry Engine Open Source), which is the battle-tested C++ library powering PostGIS, QGIS, and Google Earth.

#### Why we use it:
Python natively only understands lists, strings, and dictionaries. It has no idea what a "polygon" is. Shapely creates real geometric objects.

#### Key Functions:
```python
from shapely.geometry import box, Point, Polygon

# 1. box(minx, miny, maxx, maxy):
# Creates a 4-corner polygon (rectangle) from bounding coordinates.
my_parcel = box(-74.01, 40.70, -74.00, 40.71)

# 2. Point(x, y):
hydrant = Point(-74.005, 40.705)

# 3. Spatial predicates (returns True/False):
print(my_parcel.contains(hydrant))     # True!
print(my_parcel.intersects(hydrant))   # True!
print(my_parcel.area)                 # Area of the polygon
```

---

### B. Pandas: The Tabular Data Engine

#### What is a `DataFrame` (`df`)?
In JavaScript, 100,000 records are stored as an array of 100,000 objects:
```javascript
[{ id: 1, val: 500 }, { id: 2, val: 700 }, ...]
```
JavaScript must allocate memory for each separate object. This is slow and memory-heavy.

In Pandas, data is stored **column-wise** in contiguous C memory blocks:
```
Column 'id':    [1, 2, 3, 4, ...]
Column 'value': [500, 700, 300, 950, ...]
```

#### Why we use it:
Operations on 100,000 rows happen simultaneously in compiled C code (called **Vectorization**) rather than slow Python `for` loops.

#### Core Operations:
```python
import pandas as pd

# Creating a DataFrame from a dictionary
df = pd.DataFrame({
    "parcel_id": ["PRCL-01", "PRCL-02"],
    "assessed_value": [500000, 1200000]
})

# Filtering rows (instant in C):
expensive = df[df["assessed_value"] > 1000000]

# Transforming a column:
df["tax"] = df["assessed_value"] * 0.02
```

---

### C. GeoPandas: The Bridge Between Tables and Space

#### What is a `GeoDataFrame` (`gdf`)?
A `GeoDataFrame` is a standard Pandas `DataFrame`, but with **one special designated column called `geometry`**.
Each row in the `geometry` column contains a Shapely geometric object (`Polygon`, `Point`, etc.).

#### Why we use it:
It allows you to perform both **attribute queries** (like SQL `WHERE`) and **spatial calculations** at the same time:

```python
import geopandas as gpd

# Reading any spatial file (GeoJSON, Shapefile, GeoPackage) into a table:
gdf = gpd.read_file("parcels.geojson")

# Inspecting spatial metadata:
print(gdf.crs)        # Output: EPSG:4326
print(len(gdf))       # Number of rows (e.g., 45,000)

# Filtering by both space and attributes:
commercial_large = gdf[(gdf["land_use"] == "Commercial") & (gdf.geometry.area > 0.001)]

# Exporting directly to PostGIS database:
gdf.to_postgis(name="parcels", con=engine, if_exists="replace")
```

---

### D. NumPy: Vectorized Math & Distributions

NumPy (`import numpy as np`) is the backbone of all numerical computing in Python.

#### Methods Used in Our Pipeline:

1. **`np.linspace(start, stop, num)`**:
   - Generates `num` evenly spaced points between `start` and `stop`.
   - Used to calculate even street grid divisions across the city bounding box.

2. **`np.random.choice(a, p=[...])`**:
   - Randomly chooses values from list `a` according to probabilities `p`.
   - Example: Assigning 45% of parcels to Residential and only 5% to Industrial.

3. **`np.random.lognormal(mean, sigma)`**:
   - Generates values with a **log-normal distribution**.
   - **Why this matters:** In the real world, property values, incomes, and building heights are not a standard bell curve. Most parcels have modest values ($500k–$1M), but a small percentage of parcels (Manhattan skyscrapers) are worth billions. A log-normal distribution mirrors real urban economics.

---

## 4. The Spatial Database Layer: PostGIS & GiST Indexing

### What is PostGIS?
PostgreSQL by itself stores text, numbers, and dates.
**PostGIS** is an extension that turns PostgreSQL into a full spatial database engine by adding:
1. Native spatial data types: `GEOMETRY` and `GEOGRAPHY`.
2. Over 1,000 spatial functions: `ST_Intersects`, `ST_Distance`, `ST_Area`, `ST_AsMVT`.

---

### Why Standard Database Indexes (B-Trees) Fail on Space

In a standard database, columns like `user_id` or `created_at` use a **B-Tree index**.
B-Trees work by sorting values in a **1-dimensional line**:
$$1 \rightarrow 2 \rightarrow 3 \rightarrow 4 \rightarrow 5$$

**The Spatial Problem:**
Geometries exist in **2D (or 3D) space**. You cannot sort polygons in a single line because:
- Is polygon $A$ "greater than" polygon $B$?
- A polygon can be North of $B$, but East of $C$.

If you query `WHERE ST_Intersects(geom, my_box)` on 100,000 parcels without a spatial index, PostgreSQL must perform a **Sequential Scan** (checking all 100,000 geometries one-by-one), which takes 2–5 seconds and consumes 100% CPU.

---

### The Solution: GiST (Generalized Search Tree / R-Tree)

PostGIS uses **GiST indexes**, which implement an **R-Tree (Rectangle Tree)**:

```
[ Root Bounding Box: Entire Manhattan ]
       ├── [ Sub-Box: Lower Manhattan ]
       │        ├── [ Box: Financial District ] ──> [ Parcel 1, Parcel 2 ]
       │        └── [ Box: Battery Park ]       ──> [ Parcel 3, Parcel 4 ]
       └── [ Sub-Box: Midtown Manhattan ]
                └── [ Box: Times Square ]        ──> [ Parcel 5, Parcel 6 ]
```

#### How the Two-Stage Spatial Query Works:
1. **Stage 1 (Index Filter - Fast Bounding Box Check):**
   When you zoom in or draw a polygon, PostGIS checks the GiST R-Tree. It instantly eliminates 99% of parcels whose bounding boxes don't touch your query box in **< 2 milliseconds**.
2. **Stage 2 (Refinement - Exact Geometry Math):**
   PostGIS only runs the expensive polygon intersection algorithm on the few parcels that passed Stage 1.

```sql
-- Creating the spatial index:
CREATE INDEX idx_parcels_geom ON parcels USING GIST (geometry);

-- Updating PostgreSQL's query optimizer planner:
ANALYZE parcels;
```

---

## 5. The ETL Pipeline Anatomy

Here is the exact architecture of [scripts/ingest_parcels.py](file:///d:/code/terrascale/scripts/ingest_parcels.py) mapped to data engineering stages:

### Stage 1: Extract (E)
```python
# Check local disk cache first to avoid redundant downloads
if not os.path.exists(GEOJSON_CACHE_PATH):
    download_with_progress(api_url, GEOJSON_CACHE_PATH)

# Load into in-memory GeoDataFrame
gdf = gpd.read_file(GEOJSON_CACHE_PATH)
```

### Stage 2: Transform (T)
```python
# 1. Standardize column names (lowercase)
gdf.columns = [c.lower() for c in gdf.columns]

# 2. Prune unnecessary columns (keep only what the map needs)
target_columns = ["bbl", "land_use", "assessed_value", "num_floors", "geometry"]
gdf = gdf[[c for c in target_columns if c in gdf.columns]]

# 3. Reproject coordinates to WGS 84 (EPSG:4326)
if gdf.crs.to_string() != "EPSG:4326":
    gdf = gdf.to_crs("EPSG:4326")
```

### Stage 3: Load (L)
```python
# Open SQLAlchemy connection pool
engine = create_engine(DB_URL)

# Bulk insert in batches of 5,000 rows
gdf.to_postgis(
    name="parcels",
    con=engine,
    if_exists="replace",
    index=True,
    index_label="id",
    chunksize=5000
)

# Build GiST index and update planner
with engine.connect() as conn:
    conn.execute(text("CREATE INDEX idx_parcels_geom ON parcels USING GIST (geometry);"))
    conn.execute(text("ANALYZE parcels;"))
    conn.commit()
```

---

## 6. Quick Reference Glossary

| Symbol / Method | Library | Meaning & Purpose |
|---|---|---|
| `gpd.read_file(path)` | GeoPandas | Reads vector files (GeoJSON, Shapefile, etc.) into a `GeoDataFrame`. |
| `gdf.to_crs("EPSG:...")` | GeoPandas | Reprojects all geometry coordinates to a new coordinate reference system. |
| `gdf.set_crs("EPSG:...")` | GeoPandas | Attaches a CRS label to unprojected data without changing numbers. |
| `gdf.to_postgis(...)` | GeoPandas | Generates PostGIS table and writes all rows into the database. |
| `chunksize=5000` | GeoPandas | Splits database inserts into batches of 5,000 rows to prevent network buffer overflow. |
| `box(minx, miny, maxx, maxy)` | Shapely | Constructs a 5-point rectangular closed Polygon from 4 boundary coordinates. |
| `create_engine(url)` | SQLAlchemy | Configures a database connection pool. |
| `with engine.connect() as conn:` | SQLAlchemy | Opens a safe database session that automatically closes when finished. |
| `text("SQL...")` | SQLAlchemy | Wraps raw SQL statements for safe execution. |
| `USING GIST (geometry)` | PostgreSQL / PostGIS | Builds an R-Tree spatial index on 2D geometries for sub-millisecond queries. |
| `ANALYZE table_name;` | PostgreSQL | Re-scans table row distributions so the query planner uses the spatial index. |

---

*TerraScale Architecture & Spatial Engineering Technical Documentation.*
