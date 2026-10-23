# WindLens — UK Wind Forecast Intelligence

WindLens is a full-stack pilot dashboard for a UK electricity utility's day-ahead procurement team. It compares national wind generation forecasts with actual generation, preserves forecast publication timestamps, quantifies forecast error, and exposes the interval-level evidence behind the summary metrics.

> **Important:** The application is decision support, not an automated trading recommendation. The assignment restricts the data to Elexon national wind actuals and forecasts; it does not provide consumer demand, thermal availability, or market prices. Therefore the app does not claim to calculate the utility's actual procurement shortfall or optimal purchase volume.

## Features

- January 2026 selected by default; month selector supports other months for investigation.
- Elexon BMRS `FUELHH` actuals filtered to `fuelType = WIND` and `WINDFOR` forecast records.
- Forecast snapshots selected using `publishTime` to avoid using forecasts published after the delivery day began.
- Half-hourly forecast-vs-actual chart, error chart, daily MAE trend, KPI cards and an auditable settlement-period table.
- MAE, RMSE, signed bias, MAPE and interval coverage, with missing intervals retained in the expected half-hourly grid.
- CSV export, API response caching, health endpoint, Docker Compose, unit tests and API error states.
- Optional explicitly labelled synthetic demo mode for UI recording only.

## Architecture

```text
React + TypeScript + Recharts (Vite)
              |
              | GET /api/dashboard?month=2026-01
              v
Python FastAPI  ----  15-minute in-memory cache
       |                       |
       +---- Elexon BMRS public API (no API key)
              |-- datasets/FUELHH/stream
              |-- datasets/WINDFOR/stream
```

The backend normalizes timestamps to UTC, filters actual generation to WIND, retains forecast publication times, constructs a full half-hourly delivery grid, selects the forecast available before each delivery date, joins actuals and forecasts, then calculates metrics. The frontend only visualizes API output and does not recompute the main accuracy metrics.

## Data and evaluation methodology

**Source endpoints**

- Actual generation: <https://bmrs.elexon.co.uk/api-documentation/endpoint/datasets/FUELHH/stream>
- Wind forecasts: <https://bmrs.elexon.co.uk/api-documentation/endpoint/datasets/WINDFOR/stream>
- Elexon developer portal and API notes: <https://developer.data.elexon.co.uk/>

The Elexon API is public and does not require an API key. The service calls the streaming endpoints with a date-time window. The implementation accepts either a bare array or a `{ "data": [...] }` response envelope.

**Forecast snapshot policy:** For each target settlement period, use the latest forecast publication strictly before 00:00 Europe/London on the target delivery date. This prevents a forecast published on the delivery day from leaking into a simulated day-ahead evaluation. Midnight is a transparent proxy, not a confirmed market gate-closure timestamp. A real procurement team should supply the exact decision/gate-closure time and the policy should be changed to that agreed timestamp before operational use.

**Metrics**

- Error = forecast MW − actual MW.
- MAE = mean absolute error, in MW.
- RMSE = square root of mean squared error, in MW.
- Bias = mean signed error, in MW; positive means over-forecast, negative means under-forecast.
- MAPE = mean absolute percentage error over non-zero actual values only.
- Coverage = intervals with both actual and selected forecast / expected half-hourly intervals.

Metrics use only matched actual/forecast pairs. The full expected grid is retained so missing records lower coverage rather than silently disappearing. Elexon timestamps are converted to UK local time for display, while joins are performed in UTC.

## Files and directories

```text
.
├── backend/
│   ├── app/main.py          # FastAPI routes, Elexon client, normalisation, metrics
│   ├── tests/test_metrics.py# Metric, filtering and no-look-ahead tests
│   ├── Dockerfile
│   └── requirements.txt
├── frontend/
│   ├── src/App.tsx          # Dashboard UI, charts, filters, CSV export
│   ├── src/lib/types.ts     # Shared TypeScript data contracts
│   ├── src/lib/demo.ts      # Synthetic fixture, only enabled by explicit env flag
│   ├── src/styles.css       # Responsive dashboard styling
│   ├── Dockerfile
│   ├── .env.example
│   └── vercel.json
├── docker-compose.yml
├── render.yaml              # Render blueprint for FastAPI
└── README.md
```

## Run locally

### Option A — Docker Compose (recommended)

Requires Docker Desktop or Docker Engine with the Compose plugin.

```bash
docker compose up --build
```

- Frontend: <http://localhost:5173>
- Backend health: <http://localhost:8000/health>
- Interactive API docs: <http://localhost:8000/docs>

Stop services with `Ctrl+C`, then `docker compose down`.

### Option B — Run without Docker

Backend (Python 3.11+ recommended):

```bash
cd backend
python -m venv .venv
# macOS/Linux
source .venv/bin/activate
# Windows PowerShell: .venv\Scripts\Activate.ps1
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

Frontend (Node.js 20+):

```bash
cd frontend
cp .env.example .env.local
npm install
npm run dev
```

### Tests

```bash
cd backend
pip install -r requirements.txt
pytest -q
```

## Deployment

The frontend is configured for Vercel; the API can be deployed as a Docker web service on Render. This split deployment keeps the React site static and the Python service independently scalable.

1. Push this repository to a public GitHub repository using a neutral/random repository name.
2. In Render, create a new Blueprint from the repository and use `render.yaml` (or create a Docker web service with `backend/` as its root directory). Set `CORS_ORIGINS` to the exact Vercel frontend origin, e.g. `https://your-random-name.vercel.app`.
3. Deploy `frontend/` as a Vercel project. Set `VITE_API_BASE_URL` to the deployed Render API origin, and `VITE_DEMO_MODE` to `false`. Redeploy after setting environment variables.
4. Verify `/health`, `/api/dashboard?month=2026-01`, and the deployed UI in an incognito window. A free backend may sleep on inactivity; the first API request can take longer.

Do not use a guessed URL in the submission form. Only submit the actual URLs after deployment and incognito verification.

## AI-tool disclosure

AI assistance was used to help draft the application structure, implementation, tests, documentation and UI. The developer should personally run the application, inspect the Elexon response schema and results, verify the metric calculations, and review all generated code before submission.

## Known limitations / next steps

- The upstream Elexon stream is called at request time; availability and historical completeness depend on the upstream service and its supported date filters. Failures surface as a clear API error rather than fabricated operational data.
- The in-memory cache is per process and resets on restart. A production version should use a persistent store and scheduled backfill for reproducible data snapshots.
- The decision cutoff is configurable by design but currently uses midnight UK local time as a proxy. Replace it with the client's confirmed trading decision timestamp.
- The app does not ingest demand, prices, thermal plant costs, imbalance charges, or trading constraints, per the assignment's data limitation.
- Forecast records can be revised. This pilot uses the latest forecast published before the selected cutoff for each interval, rather than averaging versions.
- Add production observability, API retry/backoff, pagination or chunked ingestion if Elexon response sizes/limits require it, plus accessibility and end-to-end tests before operational use.
# WindLens-Wind-Forecast-Intelligence
