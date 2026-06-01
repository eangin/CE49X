**Institution:** Boğaziçi University, Spring 2026  
**Instructor:** Dr. Eyuphan Koc
---
## Project Summary
This project builds an **early-warning data pipeline** for a maritime shipping company that:
1. Collects satellite thermal anomaly data from NASA FIRMS across four conflict-prone regions
2. Gathers war/conflict news from NewsAPI
3. Correlates thermal events with news articles using spatial, temporal, and keyword matching
4. Trains ML classifiers to predict conflict association
5. Synthesises findings in a multi-panel dashboard
An **end-to-end data science pipeline** that helps maritime shipping companies monitor
conflict risk across four strategically critical regions. The pipeline:
1. Collects satellite thermal anomaly data (NASA FIRMS VIIRS) across 6 months
2. Gathers war/conflict news from the GDELT Project (free, no key required)
3. Clusters thermal detections into discrete events with DBSCAN
4. Matches thermal events to conflict news using temporal + keyword overlap
5. Trains ML classifiers (Logistic Regression, Decision Tree) to predict conflict association
6. Produces a 6-panel 300-DPI dashboard summarising findings
**Key results (May 2026 production run):**
| Metric | Value |
|--------|-------|
| FIRMS detections collected | 156,095 |
| News articles collected | 409 (GDELT, Apr 2025) |
| Thermal events (DBSCAN) | 26,459 |
| Conflict-associated events | 5,461 (20.6 %) |
| Decision Tree F1 | **0.878** |
| Decision Tree Accuracy | **0.950** |
| Mann-Whitney U p-value | 0.30 (FRP alone insufficient) |
---
## Repository Structure
```
.
ce49x-conflict-monitoring/
│
├── notebook/
│   ├── conflict_monitoring.ipynb   ← Main project notebook (all 4 tasks)
│   ├── dashboard.png               ← Multi-panel dashboard (300 DPI)
│   └── build_notebook.py           ← Script that generates the notebook
├── requirements.txt                ← Python dependencies
│   ├── conflict_monitoring.ipynb   ← Main deliverable (42 cells, all 4 tasks)
│   ├── build_notebook.py           ← Source script — edit this, then re-run to update .ipynb
│   ├── dashboard.png               ← 6-panel GridSpec dashboard at 300 DPI
│   ├── ml_meta.json                ← ML metrics cached from production run
│   │
│   ├── stage1_firms.py             ← Collect NASA FIRMS data → DB
│   ├── stage2_news.py              ← Collect GDELT conflict news → DB
│   ├── stage3_analyze.py           ← DBSCAN + matching + ML + intermediate plots
│   ├── stage4a_matches.py          ← Store event_matches + ML predictions
│   └── stage4b_dashboard.py        ← Generate dashboard.png from DB
│
├── requirements.txt                ← Python dependencies (pinned)
├── docker-compose.yml              ← Spin up local PostgreSQL for reproduction
└── README.md                       ← This file
```
## Setup Instructions
### 1. Environment Variables
Set the following secrets before running the notebook:
| Variable | Purpose |
|----------|---------|
| `FIRMS_MAP_KEY` | NASA FIRMS API map key |
| `NEWS_API_KEY` | NewsAPI developer key |
| `DATABASE_URL` | PostgreSQL connection string |
### 2. Database (PostgreSQL)
This project uses Replit's managed PostgreSQL instance (available via `DATABASE_URL`).
If running locally with Docker:
```bash
docker run --name ce49x-postgres \
---
## Environment Variables
| Variable | Required | Purpose |
|----------|----------|---------|
| `DATABASE_URL` | **yes** | PostgreSQL connection string |
| `FIRMS_MAP_KEY` | yes (Task 1) | NASA FIRMS API map key — register at https://firms.modaps.eosdis.nasa.gov/api/area/ |
| `NEWS_API_KEY` | optional | NewsAPI key (fallback only — GDELT is used by default) |
On Replit these are stored as **Secrets** (Tools → Secrets).  
Locally, export them in your shell or add to a `.env` file.
---
## Quick Start — Local Reproduction
### 1. Clone the repository
```bash
git clone <repo-url>
cd ce49x-conflict-monitoring
```
### 2. Start the PostgreSQL database with Docker
```bash
docker compose up -d
```
Wait for the health-check to pass (≈5 s), then verify:
```bash
docker compose ps
# ce49x-postgres   running (healthy)
```
Set the connection string:
```bash
export DATABASE_URL=postgresql://ce49x:ce49x@localhost:5432/conflict_monitoring
export FIRMS_MAP_KEY=<your-key>
# NEWS_API_KEY is optional — GDELT requires no key
```
### 3. Install Python dependencies
```bash
# Requires Python 3.11
pip install -r requirements.txt
```
### 4. Run the pipeline (stage scripts)
The cleanest way to re-populate the database from scratch is to run the five stage scripts
**in order**. Each script is self-contained and prints progress to stdout.
```bash
python3 notebook/stage1_firms.py      # ~90 s  — collects 156 K FIRMS detections
python3 notebook/stage2_news.py       # ~20 s  — collects ~400 GDELT articles
python3 notebook/stage3_analyze.py    # ~60 s  — clusters + matches + intermediate plots
python3 notebook/stage4a_matches.py   # ~30 s  — ML training + stores event_matches
python3 notebook/stage4b_dashboard.py # ~10 s  — generates dashboard.png at 300 DPI
```
Expected final state:
```
firms_detections   : 156,095 rows
news_articles      : 409     rows
thermal_events     : 26,459  rows
event_matches      : 26,459  rows
dashboard.png      : ~1.4 MB, 300 DPI
```
### 5. Open the notebook
```bash
jupyter notebook notebook/conflict_monitoring.ipynb
```
Run **Kernel → Restart & Run All**. The notebook reads from the already-populated database,
so it completes much faster than re-fetching the API data.
> **Alternative (Jupyter Lab):**  
> `jupyter lab` then navigate to `notebook/conflict_monitoring.ipynb`
---
## Alternative: docker run (single command)
If you prefer not to use Compose:
```bash
docker run -d \
  --name ce49x-postgres \
  -e POSTGRES_USER=ce49x \
  -e POSTGRES_HOST_AUTH_METHOD=trust \
  -e POSTGRES_PASSWORD=ce49x \
  -e POSTGRES_DB=conflict_monitoring \
  -p 5432:5432 \
  -d postgres:16
```
Then set:
```
DATABASE_URL=postgresql://ce49x@localhost:5432/conflict_monitoring
```
### 3. Install Dependencies
```bash
pip install -r requirements.txt
```
### 4. Run the Notebook
```bash
jupyter notebook notebook/conflict_monitoring.ipynb
```
Or via Jupyter Lab:
```bash
jupyter lab notebook/conflict_monitoring.ipynb
```
The notebook must run **top-to-bottom without errors** (Kernel → Restart & Run All).
## Required Database Tables
| Table | Contents |
|-------|---------|
| `firms_detections` | Raw FIRMS thermal detection records (cleaned) |
| `news_articles` | Collected conflict news articles with metadata |
| `thermal_events` | DBSCAN-clustered thermal events with computed features |
| `event_matches` | Thermal event–news article matching results + ML predictions |
Verify tables after running the notebook:
```bash
# If using Docker:
  -v ce49x_pg_data:/var/lib/postgresql/data \
  postgres:16-alpine
export DATABASE_URL=postgresql://ce49x:ce49x@localhost:5432/conflict_monitoring
```
---
## Database Tables
All tables are created automatically by the stage scripts (`if_exists='replace'`).
| Table | Rows | Key columns |
|-------|------|-------------|
| `firms_detections` | 156,095 | `latitude`, `longitude`, `brightness`, `frp`, `acq_date`, `region`, `confidence`, `daynight` |
| `news_articles` | 409 | `title`, `source`, `published_date`, `url`, `description`, `region`, `loc_keyword`, `conf_keyword` |
| `thermal_events` | 26,459 | `event_id`, `region`, `centroid_lat/lon`, `start_date`, `end_date`, `total_frp`, `max_brightness`, `n_detections`, `duration_days`, `daynight_ratio`, `month` |
| `event_matches` | 26,459 | `event_id`, `region`, `n_matching_articles`, `conflict_associated`, `ml_prediction`, all event features |
Inspect via psql:
```bash
# Docker Compose
docker exec -it ce49x-postgres psql -U ce49x -d conflict_monitoring -c "\dt"
# If using Replit DB (psql via environment):
docker exec -it ce49x-postgres psql -U ce49x -d conflict_monitoring \
  -c "SELECT region, COUNT(*) FROM firms_detections GROUP BY region ORDER BY 2 DESC;"
# Direct psql (any environment)
psql $DATABASE_URL -c "\dt"
```
psql $DATABASE_URL -c "SELECT region, COUNT(*), AVG(frp)::numeric(8,2) AS mean_frp FROM firms_detections GROUP BY region;"
```
---
## Pipeline Overview
### Task 1 — Data Collection (30 pts)
**FIRMS thermal data:**  
- Instrument: VIIRS/Suomi-NPP Standard Processing (`VIIRS_SNPP_SP`), 375 m resolution  
- Date range: 2024-11-01 → 2025-04-30 (6 months)  
- API: `https://firms.modaps.eosdis.nasa.gov/api/area/csv/{key}/{source}/{bbox}/{days}/{date}`  
- Collection: 5-day chunks × 4 regions = 144 API calls  
- Column note: VIIRS returns `bright_ti4` (not `brightness`) — renamed immediately after fetch  
**Conflict news:**  
- Source: GDELT Project DOC API (free, no key) — `https://api.gdeltproject.org/api/v2/doc/doc`  
- Fallback: NewsAPI `everything` endpoint (requires `NEWS_API_KEY`; limited to 100 req/day on free plan)  
- Date range: 2025-04-01 → 2025-04-30  
- Queries: one broad OR-query per region (4 total API calls)  
### Task 2 — Spatial & Temporal Analysis (25 pts)
- DBSCAN on `[lat_rad, lon_rad, time_rad]` with ε = 7 km, Δt = 2 days, `min_samples = 3`  
- Temporal axis scaled: `time_rad = days × (eps_km / eps_days) / EARTH_R`  
- 4 visualisations: monthly event count, FRP trend + day/night ratio, world map, regional hotspot maps  
### Task 3 — Thermal–News Correlation & Classification (30 pts)
- Matching: same region + article published within ±7 days of event start + conflict keyword in title/description  
- Hypothesis test: Mann-Whitney U on FRP (conflict vs non-conflict), two-sided  
- ML: Logistic Regression + Decision Tree, 80/20 stratified split, 5-fold cross-validation  
- Evaluation: accuracy, precision, recall, F1, confusion matrices, feature importances  
- Justification: **recall** is the primary metric — a false negative (missed conflict) is more dangerous for shipping than a false alarm  
### Task 4 — Dashboard & Discussion (15 pts)
- 6-panel `GridSpec` figure saved as `notebook/dashboard.png` at 300 DPI  
- Panels: (A) world map, (B) conflict-rate bar, (C) monthly event time-series,  
  (D) FRP boxplot with Mann-Whitney annotation, (E) ML performance bars, (F) keyword heatmap  
- Written discussion: findings, shipping implications, limitations, critical reflection  
---
## Regions Monitored
| Region | Bounding Box (W,S,E,N) | Shipping Relevance |
|--------|----------------------|-------------------|
| Ukraine | 22,44,41,53 | Grain corridor; energy infrastructure strikes |
| Iraq/Syria | 35,29,50,38 | Major oil producer; proxy conflicts near pipelines |
| Yemen/Red Sea | 42,11,56,20 | Bab-el-Mandeb chokepoint; Houthi shipping attacks |
| Gaza/Israel | 34,29,36,33 | Eastern Mediterranean; Suez Canal approach |
## Pipeline Overview (4 Tasks)
### Task 1 — Data Collection (30 pts)
- NASA FIRMS VIIRS/Suomi-NPP, 5-day chunks over 6 months
- NewsAPI conflict keyword × location cross-queries (≥1,000 unique articles)
- PostgreSQL storage via SQLAlchemy
### Task 2 — Spatial & Temporal Analysis (25 pts)
- DBSCAN clustering (ε=7 km, Δt=2 days, min_samples=3)
- Temporal visualisations: monthly event count, FRP trend, day/night ratio
- Spatial visualisations: world map + regional hotspot maps
### Task 3 — Thermal–News Correlation & Classification (30 pts)
- Matching: temporal ±7 days + region + conflict keyword
- Mann-Whitney U hypothesis test: FRP conflict vs non-conflict
- ML: Logistic Regression + Decision Tree (stratified 80/20 split)
- Confusion matrices, classification reports, feature importances
### Task 4 — Dashboard & Discussion (15 pts)
- 6-panel GridSpec dashboard saved as `dashboard.png` at 300 DPI
- Written discussion: key findings, shipping implications, limitations, reflection
## Data Sources
- **NASA FIRMS API:** https://firms.modaps.eosdis.nasa.gov/api/area/  
  Instrument: VIIRS/Suomi-NPP Standard Processing (VIIRS_SNPP_SP)  
  Accessed: May 2026
- **NewsAPI:** https://newsapi.org/v2/everything  
  Language...
[truncated]
[truncated]
[truncated]