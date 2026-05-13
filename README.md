# Aviation Delay Research — Data Ingestion Pipeline

Builds a regression-ready **airport × UTC-hour panel dataset** by combining three years of FAA flight delay records with hourly surface weather observations. The single notebook `aviation_pipeline.ipynb` handles everything from raw download to a clean Parquet export.

| | |
|---|---|
| **Airports** | Top 50 busiest US airports |
| **Period** | January 2022 – December 2024 (36 months) |
| **Granularity** | Airport × UTC hour |
| **Flight source** | FAA BTS On-Time Performance |
| **Weather source** | Meteostat (nearest surface station, ≤ 50 km) |
| **Storage** | DuckDB + Parquet |

---

## Project layout

```
aviation_pipeline.ipynb   ← single notebook, run top-to-bottom
data/
├── raw/
│   ├── flights/          ← flights_{year}_{month}.parquet  (one file per month)
│   └── weather/          ← weather_{IATA}.parquet          (one file per airport)
├── processed/
├── exports/
│   ├── airport_hour_panel.parquet   ← regression-ready dataset
│   ├── panel_sample.csv             ← 500-row inspection sample
│   └── data_dictionary.csv          ← column-level documentation
└── aviation.duckdb                  ← queryable database (flights_raw, weather_raw, airport_hour_panel)
schema/
└── schema.sql                       ← SQL DDL with column comments
```

---

## Setup

```bash
python -m venv .venv
.venv\Scripts\activate        # Windows
pip install pandas pyarrow duckdb "meteostat>=2.1" requests tqdm polars
```

Then open `aviation_pipeline.ipynb` in Jupyter or VS Code and run all cells in order.

> **First run:** downloading 36 months of BTS ZIP files takes roughly 30–90 minutes depending on connection speed. Subsequent runs skip already-cached files.

---

## Notebook sections

| # | Section | What it does |
|---|---------|-------------|
| 0 | Install dependencies | One-time `%pip install` (commented out by default) |
| 1 | Setup & configuration | Paths, date range, constants, logging, DuckDB helpers |
| 2 | Airport list | Top-50 airport table embedded in-notebook — no external download |
| 3 | Flight data ingestion | Downloads BTS On-Time ZIP files month by month, filters to top-50 airports, caches as Parquet |
| 4 | Weather data ingestion | Fetches hourly observations from Meteostat per airport, caches as Parquet |
| 5 | Cleaning & standardisation | `clean_flights`, `clean_weather`, `add_dep_hour` — all pure functions returning cleaned copies |
| 6 | Airport-hour panel | Aggregates flights and weather into one row per airport × UTC hour |
| 7 | Exports | Writes Parquet, CSV sample, schema SQL, and data dictionary |
| 8 | Quick data inspection | Coverage and shape checks — no analysis |

---

## Output schema

Each row in `airport_hour_panel` represents one airport for one UTC hour.

| Column | Type | Description |
|--------|------|-------------|
| `airport_iata` | string | IATA airport code |
| `timestamp_utc` | timestamptz | UTC hour (floored to hour boundary) |
| `hour_of_day` | int | 0–23 |
| `day_of_week` | int | 0 = Monday, 6 = Sunday |
| `month` | int | 1–12 |
| `year` | int | 2022–2024 |
| `weekend_indicator` | bool | True if Saturday or Sunday |
| `total_departures` | int | Scheduled departures in this hour |
| `total_arrivals` | int | Scheduled arrivals in this hour |
| `cancellation_count` | int | Cancelled flights |
| `avg_dep_delay` | float | Mean departure delay (minutes) |
| `avg_arr_delay` | float | Mean arrival delay (minutes) |
| `avg_temperature` | float | Air temperature (°C) |
| `avg_precipitation` | float | Precipitation (mm/hr) |
| `avg_snowfall` | float | Snow depth (mm) |
| `avg_wind_speed` | float | Mean wind speed (km/h) |
| `avg_visibility` | float | Horizontal visibility (km) |
| `avg_pressure` | float | Sea-level pressure (hPa) |

Weather columns are `NULL` for airports outside the 50 km Meteostat station search radius.

---

## Data sources

- **FAA BTS On-Time Performance** — `transtats.bts.gov/PREZIP/` — public domain, no API key required
- **Meteostat** — `meteostat.net` — open weather data via the `meteostat` Python library (v2)
