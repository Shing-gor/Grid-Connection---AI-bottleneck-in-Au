# GridConnection NSW — AI Infrastructure Siting Intelligence Platform

![Python](https://img.shields.io/badge/python-3.11-blue?logo=python&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL+PostGIS-16-316192?logo=postgresql&logoColor=white)
![dbt](https://img.shields.io/badge/dbt-FF694B?logo=dbt&logoColor=white)
![scikit-learn](https://img.shields.io/badge/scikit--learn-1.4-orange?logo=scikit-learn&logoColor=white)
![Streamlit](https://img.shields.io/badge/Streamlit-FF4B4B?logo=streamlit&logoColor=white)
![Status](https://img.shields.io/badge/status-🚧%20in%20progress-yellow)

> **Business question:** Where should hyperscalers build AI factories in NSW — and how long will the grid actually let them wait?

---

## The Business Case

Australia is on the verge of a data centre construction boom. Projected AI infrastructure demand is expected to reach **3.2 GW of installed capacity by 2035**, representing up to **$135 billion in investment** — equivalent to building a new mid-sized city's worth of infrastructure in under a decade.

The single biggest constraint is not land, capital, or permits. It is **electricity grid capacity**.

Every AI factory needs a reliable, high-voltage grid connection delivering between 50 MW and 500+ MW continuously. In NSW, every such connection must flow through Ausgrid's distribution network — approximately 180 zone substations serving Sydney, the Hunter, and the Central Coast. These substations have fixed rated capacities. Many are already heavily loaded. The queue of applicants waiting to connect stretches years into the future.

This creates a trillion-dollar information problem: **which sites can actually support an AI factory, at what cost, and how soon?**

Hyperscalers (AWS, Google, Microsoft, NextDC) are making billion-dollar site selection decisions right now, with incomplete information. Ausgrid's own planning team needs to forecast where demand will surge so they can schedule augmentation works years in advance. Property developers and investors need to understand which industrial land parcels carry a hidden grid premium.

**This project builds the analytical tool that answers all three questions simultaneously** — a data-driven site suitability intelligence platform that ranks every zone in NSW's distribution network against five measurable dimensions.

---

## The Three Numbers That Define This Project

### 1. MW Headroom per Zone Substation
```
headroom_mw = rated_capacity_mva − peak_recorded_load_mva
```
This is the raw physical capacity available at each of Ausgrid's ~180 zone substations right now. An AI factory needs a minimum of 50 MW to be viable. A hyperscale facility needs 200 MW+. A zone substation with 5 MW of headroom is effectively unavailable — regardless of how cheap the land next to it is.

**Source:** Ausgrid Zone Substation Load Data (data.nsw.gov.au) — published quarterly, 15-minute intervals since 2005.

**Why it matters:** This is the hard constraint. No amount of money or political will changes the physics of a substation's rated capacity on a short timeline. Headroom is either there or it isn't.

---

### 2. Estimated Grid Connection Wait Time (months)
```
wait_time = f(requested_mw, substation_load_factor, queue_position, augmentation_required)
```
Even a substation with sufficient headroom may have 8 other projects already queued ahead. The real question is not "can this substation handle it" but "when will it actually be energised." This is the ML model at the core of Stage 4 — trained on AEMO's historical connection queue to predict wait time for a new applicant at each zone.

**Source:** AEMO Transmission Connection Queue (aemo.com.au) — updated monthly.

**Why it matters:** A 2-year wait is manageable for most hyperscalers. A 7-year wait is a deal-breaker — the AI market moves too fast. This number is what separates a "theoretically viable" site from an "operationally viable" one.

---

### 3. Renewable Energy Score (0–100)
```
renewable_score = w1 × solar_proximity + w2 × wind_proximity + w3 × rooftop_density + w4 × ppa_availability
```
Hyperscalers have public net-zero commitments. AWS, Google, and Microsoft have all pledged 100% renewable energy matching for their data centres. A site powered primarily by coal-heavy baseload — even if grid-connected immediately — may be ruled out by the hyperscaler's own ESG policy before any technical analysis is done.

**Source:** CSIRO/AEMO renewable energy zones, BOM solar irradiance grids, Ausgrid solar home electricity data (rooftop density proxy).

**Why it matters:** This is the filter that comes before MW headroom in many hyperscalers' decision frameworks. A site that scores below 40/100 on renewable proximity will not appear in a shortlist regardless of grid capacity.

---

## The Five-Dimensional Suitability Index

Every NSW zone substation / LGA is scored on five dimensions, combined into a single ranked suitability score:

| Dimension | What it measures | Data source | Weight |
|---|---|---|---|
| Grid capacity | MW headroom at nearest substation | Ausgrid load data | 30% |
| Connection timeline | Estimated wait time (ML model) | AEMO connection queue | 25% |
| Renewable energy | Proximity to solar/wind + rooftop density | CSIRO / BOM / Ausgrid solar | 20% |
| Cooling efficiency | Climate suitability for data centre cooling (PUE proxy) | BOM temperature/humidity | 15% |
| Total cost | Land value + connection cost + electricity tariff | NSW Valuer General / Ausgrid pricing | 10% |

---

## Project Structure

```
gridconnection_nsw/
│
├── data/
│   ├── raw/                         # Downloaded source files — never modified
│   │   ├── ausgrid_substation_load/ # Quarterly 15-min load CSVs
│   │   ├── ausgrid_outages/         # Past outage records
│   │   ├── ausgrid_solar/           # Solar home electricity data
│   │   ├── aemo_connection_queue/   # Monthly queue snapshots
│   │   ├── bom_climate/             # Temperature and humidity grids
│   │   └── nsw_spatial/             # Land zoning, LGA boundaries (GeoJSON)
│   └── processed/                   # Outputs from dbt / Python transforms
│
├── ingestion/
│   ├── 01_download_ausgrid.py       # Fetches all Ausgrid open datasets
│   ├── 02_download_aemo.py          # Fetches AEMO connection queue
│   ├── 03_download_bom.py           # Fetches BOM climate data
│   └── 04_download_nsw_spatial.py   # Fetches NSW spatial data
│
├── notebooks/
│   ├── 0_EDA.py                     # Exploratory analysis across all sources
│   ├── 1_grid_capacity.py           # MW headroom calculation and mapping
│   ├── 2_connection_timeline.py     # ML model: wait time prediction
│   ├── 3_renewable_score.py         # Renewable proximity scoring
│   ├── 4_cooling_cost_score.py      # PUE estimate + cost modelling
│   └── 5_composite_index.py         # Combine all scores → ranked site list
│
├── dbt/
│   ├── models/
│   │   ├── staging/                 # stg_substations, stg_queue, stg_climate
│   │   └── marts/
│   │       ├── mart_grid_capacity.sql
│   │       ├── mart_connection_queue.sql
│   │       └── mart_suitability_index.sql
│   └── dbt_project.yml
│
├── dashboard/
│   └── app.py                       # Streamlit app — interactive NSW map
│
├── utils.py                         # Shared helpers
├── requirements.txt
├── .gitignore
└── README.md
```

---

## Detailed Stage-by-Stage Derivation

### Stage 1 — Data Collection

**Goal:** Download all source data and store in `./data/raw/`. Every source is free and publicly available.

| Dataset | Source URL | Format | Update frequency |
|---|---|---|---|
| Zone substation load (2005–present) | data.nsw.gov.au → Ausgrid | CSV, 15-min intervals | Quarterly |
| Past outages (50+ customers, 5+ min) | ausgrid.com.au/data-to-share | CSV | Quarterly |
| Solar home electricity data | ausgrid.com.au/data-to-share | CSV | Annual |
| AEMO connection queue | aemo.com.au/energy-systems/electricity/national-electricity-market-nem/nem-forecasting-and-planning/connection-enquiries | CSV/XLSX | Monthly |
| BOM climate grids (temperature, humidity) | bom.gov.au/climate/data | NetCDF / CSV | Monthly |
| NSW LGA boundaries | data.nsw.gov.au → spatial | GeoJSON | Annual |
| NSW land zoning | spatial.industry.nsw.gov.au | GeoJSON | Annual |
| NSW Valuer General land values | valuergeneral.nsw.gov.au | CSV | Annual |

**Derivation steps:**
1. Write one Python ingestion script per source using `requests` + `pandas`
2. Save raw files to `./data/raw/<source>/` with timestamp in filename
3. Log each download (source, date, row count, file size) to `./data/raw/ingestion_log.csv`
4. Never modify files in `./data/raw/` — all transformations happen downstream

---

### Stage 2 — Database Design and dbt Modelling

**Goal:** Load raw data into PostgreSQL + PostGIS, clean and join in dbt, produce one mart table per analytical layer.

**Schema design:**
```sql
-- Core tables
stg_substations        -- substation_id, name, lat, lon, rated_mva, geometry
stg_substation_load    -- substation_id, timestamp, load_mva (15-min intervals)
stg_outages            -- substation_id, date, duration_min, customers_affected, cause
stg_connection_queue   -- project_id, substation_id, requested_mw, application_date,
                       -- status, energisation_date (if complete)
stg_solar_zones        -- postcode, avg_daily_generation_kwh, n_solar_customers
stg_climate            -- lat, lon, month, avg_temp_c, avg_humidity_pct
stg_land_parcels       -- parcel_id, lga, zoning, land_value_sqm, geometry

-- Mart tables (one per score dimension)
mart_grid_capacity     -- substation_id + headroom_mw + load_factor + capacity_score
mart_connection_queue  -- substation_id + queue_depth_mw + predicted_wait_months
mart_renewable_score   -- substation_id + renewable_score_0_100
mart_suitability_index -- substation_id + all five scores + composite_rank
```

**dbt model logic:**
- `mart_grid_capacity`: joins `stg_substations` with aggregated peak load from `stg_substation_load`; calculates headroom and normalises to 0–100 score
- `mart_connection_queue`: joins queue data to substations; aggregates total MW queued per substation; enriched with ML model output in Python
- `mart_suitability_index`: joins all five mart tables; applies weighted composite formula; produces final ranked list

---

### Stage 3 — Spatial Analysis and Scoring

**Goal:** Produce a 0–100 score per zone substation on five dimensions. All spatial operations use `geopandas` and `PostGIS`.

**3a. Grid Capacity Score**
```python
# Per substation:
peak_load_mva = substation_load_df.groupby('substation_id')['load_mva'].quantile(0.95)
headroom_mva  = rated_capacity_mva − peak_load_mva
load_factor   = peak_load_mva / rated_capacity_mva

# Normalise headroom to 0–100
capacity_score = minmax_scale(headroom_mva) × 100
```

**3b. Reliability Score**
```python
# Per substation, from outage data:
outage_freq     = outages.groupby('substation_id')['date'].count()       # events/year
avg_duration    = outages.groupby('substation_id')['duration_min'].mean()
saidi_proxy     = outage_freq × avg_duration                              # System Average Interruption Duration Index proxy

# Invert so higher = more reliable
reliability_score = 100 − minmax_scale(saidi_proxy) × 100
```

**3c. Renewable Energy Score**
```python
# Distance from each substation to nearest committed renewable project (AEMO REZ map)
dist_to_solar = gpd.sjoin_nearest(substations_gdf, solar_farms_gdf, how='left')['distance_km']
dist_to_wind  = gpd.sjoin_nearest(substations_gdf, wind_farms_gdf,  how='left')['distance_km']

# Rooftop solar density in surrounding postcode (Ausgrid solar data)
rooftop_density = solar_zones_df['n_solar_customers'] / solar_zones_df['area_sqkm']

# Weighted composite
renewable_score = (
    0.4 × (100 − minmax_scale(dist_to_solar) × 100) +
    0.4 × (100 − minmax_scale(dist_to_wind)  × 100) +
    0.2 × minmax_scale(rooftop_density) × 100
)
```

**3d. Cooling Efficiency Score (PUE Proxy)**
```python
# PUE (Power Usage Effectiveness): 1.0 = perfect, 1.5+ = hot/inefficient
# Driven primarily by ambient temperature and humidity

avg_summer_temp = climate_df.query("month in [12,1,2]").groupby('substation_id')['avg_temp_c'].mean()
pue_estimate    = 1.0 + (avg_summer_temp / 100)   # simplified linear proxy

# Invert: lower PUE = better score
cooling_score = 100 − minmax_scale(pue_estimate) × 100
```

**3e. Cost Score**
```python
# Two components: land value and grid connection cost proxy
land_value_sqm   = land_parcels_df.groupby('substation_id')['land_value_sqm'].median()
dist_to_substation = gpd.sjoin_nearest(sites_gdf, substations_gdf)['distance_km']

# Connection cost proxy: distance × $/km for HV cable installation (~$1M/km typical)
connection_cost_proxy = dist_to_substation × 1_000_000

# Lower cost = higher score
cost_score = 100 − minmax_scale(land_value_sqm + connection_cost_proxy) × 100
```

---

### Stage 4 — ML Model: Connection Timeline Prediction

**Goal:** Train a regression model on historical AEMO queue data to predict estimated wait time (months) for a new connection application at each zone substation.

**Feature set:**
| Feature | Description |
|---|---|
| `requested_mw` | Size of the proposed connection |
| `substation_load_factor` | How loaded the substation already is (0–1) |
| `queue_position` | How many projects are ahead in the queue |
| `queue_depth_mw` | Total MW already queued at this substation |
| `augmentation_required` | Binary flag: does substation need upgrade? |
| `project_type` | Solar farm, wind, data centre, industrial, etc. |
| `application_year` | Year of application (captures regulatory environment) |

**Target variable:**
```
wait_months = (energisation_date − application_date).days / 30
```
(Only calculable for completed/energised projects — these form the training set)

**Model pipeline:**
```python
from sklearn.ensemble import GradientBoostingRegressor
from sklearn.model_selection import cross_val_score

model = GradientBoostingRegressor(n_estimators=300, max_depth=5, learning_rate=0.05)
cv_rmse = cross_val_score(model, X_train, y_train, cv=5, scoring='neg_root_mean_squared_error')

# Apply to current queue entrants and unqueued zone substations
# Output: predicted_wait_months per substation
```

---

### Stage 5 — Composite Index and Dashboard

**Goal:** Combine all five scores into a ranked list, build an interactive Streamlit dashboard, and write an executive report.

**Composite formula:**
```python
suitability_score = (
    0.30 × capacity_score    +
    0.25 × timeline_score    +   # 100 − minmax_scale(predicted_wait_months) × 100
    0.20 × renewable_score   +
    0.15 × cooling_score     +
    0.10 × cost_score
)
```

**Dashboard features (Streamlit + Folium):**
- Choropleth map of NSW coloured by suitability score
- Sidebar sliders to adjust dimension weights (investor vs. hyperscaler vs. ESG-focused view)
- Click any substation → detailed profile card (all five sub-scores, recommended action, estimated cost)
- Top 10 sites ranked table with exportable CSV

**Executive report structure:**
1. Market context — $135B opportunity, grid as bottleneck
2. Methodology — five dimensions, data sources, ML model
3. Top 5 recommended sites — ranked profiles with grid-ready date, cost estimate, risk rating
4. Strategic recommendations for Ausgrid — which substations need priority augmentation
5. Limitations and future work

---

## Setup and Reproduction

```bash
git clone https://github.com/Shing-gor/gridconnection_nsw.git
cd gridconnection_nsw

pip install -r requirements.txt

# Stage 1: download all data
python ingestion/01_download_ausgrid.py
python ingestion/02_download_aemo.py
python ingestion/03_download_bom.py
python ingestion/04_download_nsw_spatial.py

# Stage 2: set up database and run dbt
createdb gridconnection
psql gridconnection -c "CREATE EXTENSION postgis;"
dbt run

# Stage 3–5: run notebooks in order
jupytext --to notebook notebooks/0_EDA.py && jupyter notebook

# Dashboard
streamlit run dashboard/app.py
```

---

## Tech Stack

| Layer | Tools |
|---|---|
| Data ingestion | Python · requests · pandas |
| Spatial analysis | geopandas · PostGIS · Shapely |
| Database | PostgreSQL 16 + PostGIS |
| Transformation | dbt |
| Machine learning | scikit-learn · GradientBoostingRegressor |
| Visualisation | Matplotlib · Seaborn · Folium · Plotly |
| Dashboard | Streamlit |
| Version control | Git · GitHub |

---

*GridConnection NSW · Personal project · 2025*
*Data sources: Ausgrid (CC BY 4.0) · AEMO (public) · BOM (public) · NSW Government (CC BY 4.0)*
