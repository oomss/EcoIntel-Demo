# EcoIntel OSINT — Economic Intelligence Platform for Romania

EcoIntel is a portfolio-grade **economic OSINT (Open-Source Intelligence) platform** focused on Romania. It turns public economic information into a source-traceable analyst workflow:

`official source → ingestion → normalized observation → evidence → signal → forecast`

The project now includes **real ingestion connectors** for:

- **Eurostat** — current HICP monthly annual-rate dataset `prc_hicp_minr`
- **Banca Națională a României (BNR)** — official daily EUR/RON reference-rate XML feed
- **Institutul Național de Statistică (INS)** — TEMPO Online JSON service, including `SOM103A` unemployment-rate data

Eurostat currently uses `prc_hicp_minr` as the replacement for the older HICP monthly dataset. Eurostat documents its REST service as a programmatic interface to official statistics. INSSE exposes TEMPO through a JSON REST service, and the official TEMPO catalogue lists `SOM103A` as the unemployment-rate table. BNR publishes an official XML exchange-rate feed. See the source links in the application’s source registry.

## What the project demonstrates

- Python data ingestion and normalization
- REST API development with FastAPI
- SQL / SQLite data modeling
- source provenance and authority scoring
- official economic-data collection
- anomaly detection
- transparent time-series forecasting
- risk heuristics
- analyst search and evidence browsing
- Docker deployment
- automated regression testing

## Architecture

```text
                    OFFICIAL PUBLIC SOURCES
        ┌────────────────┬─────────────────┬──────────────┐
        │   Eurostat     │      BNR        │     INS      │
        │ HICP / macro   │  FX reference   │ TEMPO / NIS  │
        └────────┬───────┴────────┬────────┴──────┬───────┘
                 │                │               │
                 └────────────────┼───────────────┘
                                  ▼
                      ┌──────────────────────┐
                      │ Ingestion connectors │
                      │ JSON / XML / REST    │
                      └──────────┬───────────┘
                                 ▼
                      ┌──────────────────────┐
                      │ Normalization layer  │
                      │ period / unit / geo  │
                      │ provenance / series  │
                      └──────────┬───────────┘
                                 ▼
                      ┌──────────────────────┐
                      │ SQLite evidence DB   │
                      │ observations/events  │
                      │ source registry      │
                      └──────────┬───────────┘
                                 ▼
                 ┌───────────────┼───────────────┐
                 ▼               ▼               ▼
            Anomalies       Forecasting       Search/API
                 │               │               │
                 └───────────────┼───────────────┘
                                 ▼
                      ┌──────────────────────┐
                      │ Analyst dashboard    │
                      └──────────────────────┘
```

## Project structure

```text
app/
  analytics.py       # anomaly, forecast and risk calculations
  db.py              # SQLite schema
  ingest.py          # CLI orchestration for live ingestion
  ingestors.py       # Eurostat / BNR / INS connectors
  main.py            # FastAPI routes
  models.py          # domain models
  seed.py            # offline fallback only
  static/
    index.html
    styles.css

tests/
  test_api.py
  test_ingestors.py

Dockerfile
docker-compose.yml
requirements.txt
README.md
```

## Run the live ingestion

### 1. Create and activate a virtual environment

Windows PowerShell:

```powershell
python -m venv .venv
.venv\\Scripts\\Activate.ps1
```

macOS / Linux:

```bash
python -m venv .venv
source .venv/bin/activate
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Pull official data

```bash
python -m app.ingest --source all
```

You can also run sources independently:

```bash
python -m app.ingest --source eurostat
python -m app.ingest --source bnr
python -m app.ingest --source ins
```

The command reports each connector independently, so a temporary failure at one source does not hide which source failed.

### 4. Start the application

```bash
uvicorn app.main:app --reload
```

Open:

```text
http://127.0.0.1:8000
```

Swagger API documentation:

```text
http://127.0.0.1:8000/docs
```

## Offline fallback

The repository also contains a small **synthetic fallback dataset** so the UI and tests can run without network access. Those values are deliberately labeled as offline-demo data and should never be described as current official statistics.

To refresh the database with current official data, simply rerun:

```bash
python -m app.ingest --source all
```

## API

```text
GET /api/summary
GET /api/series?indicator=HICP%20inflation
GET /api/events?limit=10
GET /api/sources
GET /api/risks
GET /api/search?q=inflation
```

## Data model

Each observation stores:

```text
country
indicator
period
value
unit
source_id
publication_date
retrieved_at
series_code
```

This keeps the project closer to an actual intelligence/data-engineering workflow than a chart-only project: the application knows **what was observed, for which period, from which source, and from which official series**.

## Source details

### Eurostat

The live connector uses the current monthly HICP dataset `prc_hicp_minr`, selecting:

```text
geo = RO
coicop18 = TOTAL
unit = RCH_A
```

The result is normalized into the application’s `HICP inflation` series.

### BNR

The live connector parses BNR's official XML feed and extracts the EUR/RON reference rate for each published business day. It targets `curs.bnr.ro`, the subdomain BNR documents for automated/bulk retrieval — the older `www.bnr.ro/nbrfxrates.xml` path has been observed rejecting direct/automated requests (WAF block), so this connector avoids it. Override with the `BNR_FX_URL` env var if BNR changes this again.

### INS

The live connector uses the TEMPO Online JSON service metadata for `SOM103A`, dynamically discovers its dimensions/options, constructs a selection, sends it to the TEMPO pivot endpoint, and normalizes the returned unemployment-rate observations.

The connector does **not** hard-code opaque item IDs beyond the published table code; it reads dimension metadata first. That makes it much more resilient to changes in option numbering.

## Forecasting

The current demo intentionally keeps the forecast model transparent: ordinary least-squares linear trend extrapolation over the observed series.

For a production version, the natural next models are:

- SARIMA / seasonal exponential smoothing
- VAR for multi-indicator relationships
- rolling-origin backtesting
- model comparison and calibration
- scenario analysis with explicit assumptions

The project intentionally avoids presenting a simple statistical forecast as an official forecast.

## OSINT / intelligence design principles

The project is structured around a few analyst-oriented principles:

**Traceability** — every observation links to a source record.

**Source hierarchy** — official statistical and central-bank sources receive high authority scores, but the score is metadata, not a claim of infallibility.

**Reproducibility** — the exact source series is stored alongside normalized observations.

**Separation of fact and interpretation** — the database stores observed values separately from the application’s risk heuristics and forecasts.

**Failure visibility** — ingestion errors are surfaced per source rather than silently replacing missing data with invented values.

## Docker

```bash
docker compose up --build
```

Then open `http://127.0.0.1:8000`.

For live ingestion inside Docker, run:

```bash
docker compose run --rm ecointel python -m app.ingest --source all
```

## Tests

```bash
pytest -q
```

The tests cover both API behavior and the parsing logic used by the Eurostat JSON-stat / INS TEMPO ingestion layers.

## GitHub repository description

> Romania-focused economic OSINT platform using official Eurostat, BNR and INS data for source-traceable macro monitoring, anomaly detection, forecasting and analyst search.

## Resume-ready description

> Built a source-traceable economic OSINT platform in Python/FastAPI and SQLite, integrating official Eurostat HICP, BNR FX and INSSE labour-market data with provenance tracking, anomaly detection, transparent forecasting and an analyst dashboard.

## Official source references

- Eurostat REST API: https://ec.europa.eu/eurostat/api/dissemination/statistics/1.0/data/
- Eurostat HICP dataset: https://ec.europa.eu/eurostat/databrowser/view/prc_hicp_minr/default/table?lang=en
- BNR: https://www.bnr.ro/
- BNR XML reference rates (machine-readable subdomain): https://curs.bnr.ro/nbrfxrates.xml
- INSSE TEMPO: http://statistici.insse.ro:8077/tempo-online/
- INSSE SOM103A: https://statistici.insse.ro/tempoins/index.jsp?ind=SOM103A&lang=ro&page=tempo3
