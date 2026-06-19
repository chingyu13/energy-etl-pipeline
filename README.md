# Australian Energy ETL Pipeline

A multi-source geospatial ETL pipeline that ingests, cleans, geocodes, and stores Australian energy and economic data into a cloud PostgreSQL database — built as part of COMP5339 Data Engineering at the University of Sydney.

---

## What It Does

The pipeline pulls data from three government sources, resolves inconsistencies across them, geocodes thousands of energy facilities, and loads everything into a star-schema relational database ready for downstream analysis.

**Data Sources**
- **NGER** (National Greenhouse and Energy Reporting) — facility-level emissions and electricity production, 2014–2023, via Australian Government REST API
- **LGCS** (Large-scale Generation Certificate Scheme) — accredited renewable power stations, via REST API
- **ECON** (ABS Economic Census) — regional business activity data, via ABS Excel files

---

## Pipeline Stages

**1 · Data Acquisition** — downloads all three datasets programmatically; detects existing cached files to avoid redundant fetches; loads ASGS geospatial reference shapefiles.

**2 · Cleaning & Validation** — harmonises column names across NGER years, coalesces duplicate columns, standardises date formats, classifies ECON rows by ASGS geographic level (SA2–national), validates region codes against the ASGS reference, and flags outliers using IQR bounds (retained rather than dropped, to preserve state-level aggregates).

**3 · Geocoding** — resolves latitude/longitude for ~1,000 NGER facilities and LGCS power stations using a dual-API strategy: OpenStreetMap Nominatim as primary, Google Maps API as fallback for unresolved addresses. Fuzzy-matches inconsistent facility names across datasets using `thefuzz`/`rapidfuzz`. Results are persisted to the DB incrementally so the geocoding step is resumable.

**4 · Database Load** — writes to a Neon cloud PostgreSQL instance with PostGIS enabled. Schema follows a star model:

```
dim_facility  ←─┐
dim_lgcs      ←─┤
dim_fuel      ←─┼── fact_nger
dim_year      ←─┤   fact_econ
geo_regions   ←─┘
```

Geometry columns store point and region shapes for spatial queries.

---

## My Contributions

Sections 1 (data acquisition), 3 (geocoding), and 4 (DB schema + ETL load). Section 2 (data cleaning) was collaborative.

---

## Tech Stack

| Category | Tools |
|---|---|
| Language | Python 3.11+ |
| Data | pandas, geopandas, shapely |
| Geocoding | requests, thefuzz, rapidfuzz, Google Maps API, OpenStreetMap Nominatim |
| Database | PostgreSQL (Neon cloud), psycopg2, PostGIS |
| Visualisation | matplotlib, ipyleaflet |

---

## How to Run

```bash
pip install pandas geopandas shapely psycopg2-binary requests thefuzz rapidfuzz matplotlib ipyleaflet python-dotenv
```

Create a `.env` file with your credentials (see `.env.example`):

```
NEON_HOST=...
NEON_PORT=5432
NEON_DB=neondb
NEON_USER=...
NEON_PASSWORD=...
GOOGLE_MAPS_API_KEY=...
```

Then run `energy-etl-pipeline.ipynb` top to bottom. Geocoding results are cached in the database after the first run.

---

## Notes
- The A2 streaming dashboard (`energy-streaming-dashboard`) queries the `dim_facility` and `geo_regions` tables built by this pipeline.
- ECON data for 2014–2015 is structurally absent from the ABS 2011–2024 release, not a filtering error.
