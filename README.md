# DevFlow — Website & API Reliability Platform

DevFlow checks public websites and JSON API endpoints over HTTP(S), stores each observed result, and opens or resolves incidents as service state changes. The dashboard reports only data collected by the running system. Observed-check uptime is successful checks divided by all recorded checks within the selected period; unobserved periods are excluded.

## Architecture and data flow

```mermaid
flowchart LR
  Browser[React dashboard] -->|REST /api/v1| API[FastAPI]
  API --> DB[(PostgreSQL)]
  Beat[Celery Beat] --> Redis[(Redis broker)]
  Beat -->|periodic due scan| APIQueue[Celery task queue]
  APIQueue --> Worker[Celery worker]
  Worker -->|bounded outbound HTTP check| Target[Public HTTP endpoint]
  Worker --> DB
  DB -->|checks, incidents, uptime| API
  API --> Browser
```

## Features

- Monitor creation, editing, pagination, pause/resume, manual checks and deletion.
- Periodic checks through Celery worker plus one Celery Beat service in Compose.
- HTTP status and latency history, observed 24-hour/7-day/30-day uptime, incident creation and recovery.
- SSRF controls for URL schemes, embedded credentials, local names and resolved non-public addresses. Redirects are disabled.
- React + TypeScript dashboard using React Router, TanStack Query and Recharts.
- SQLAlchemy 2 models and Alembic-managed schema.

## Stack

Python 3.12, FastAPI, Pydantic, SQLAlchemy 2, Alembic, PostgreSQL 16, Redis 7, Celery 5, React 19, TypeScript 5, Vite 6, React Router 7, TanStack Query 5, Recharts 2, pytest, Vitest, Docker Compose.

## Prerequisites

- Docker Desktop with Compose v2 (recommended), or Python 3.12+, Node.js 22+, PostgreSQL 16 and Redis 7 for local development.
- Copy `.env.example` to `.env` for Compose overrides. The checked-in values are development-only placeholders.

## Start with Docker Compose

```powershell
Copy-Item .env.example .env
docker compose up --build
```

Open the frontend at <http://localhost:8080>, API docs at <http://localhost:8000/docs>, and health at <http://localhost:8000/health>. Backend startup runs `alembic upgrade head` before Uvicorn serves requests. PostgreSQL data persists in the `postgres_data` volume. Only the `beat` service runs the scheduler. Stop with `docker compose down`; remove persisted local database data only with `docker compose down -v`.

## Environment configuration

Backend settings use `DATABASE_URL`, `REDIS_URL`, `CORS_ORIGINS`, `ENVIRONMENT`, `CHECK_TIMEOUT_SECONDS` and `MAX_RESPONSE_BYTES`. Defaults are suitable for local development. Compose injects service DNS names. Frontend API base is `VITE_API_BASE_URL`; an empty value uses same-origin `/api/v1` and the Compose Nginx config proxies to FastAPI. Vite development defaults to `http://localhost:8000`.

## Local development

Start PostgreSQL and Redis, then from `backend`:

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
$env:DATABASE_URL = "postgresql+psycopg://devflow:devflow@localhost:5432/devflow"
$env:REDIS_URL = "redis://localhost:6379/0"
alembic upgrade head
uvicorn app.main:app --reload
```

In separate terminals, with the same environment and from `backend`:

```powershell
celery -A app.tasks.celery_app:celery_app worker --loglevel=INFO
celery -A app.tasks.celery_app:celery_app beat --loglevel=INFO --pidfile=
```

Run only one Beat instance for a deployment. For frontend development, from `frontend`:

```powershell
npm install
npm run dev
```

## Tests and builds

```powershell
cd backend
pip install -r requirements.txt
pytest -q
cd ..\frontend
npm install
npm test
npm run build
```

The unit tests use SQLite and HTTPX mock transports. They do not call external websites or require Redis. To exercise the service-to-service deployment smoke path, start Compose and use the API examples below; the Compose worker/Beat share PostgreSQL and Redis. There is no public simulated-target endpoint. For deterministic manual failure and recovery in tests, HTTPX `MockTransport` supplies responses without granting a production worker access to a private target.

## API examples

Create a monitor (only public hosts pass URL validation):

```bash
curl -X POST http://localhost:8000/api/v1/monitors \
  -H 'Content-Type: application/json' \
  -d '{"name":"Public docs","url":"https://www.example.org/","check_interval_seconds":300}'
```

Trigger a real manual check, then inspect recorded checks, uptime, and incidents:

```bash
curl -X POST http://localhost:8000/api/v1/monitors/1/check
curl 'http://localhost:8000/api/v1/monitors/1/checks?limit=50'
curl 'http://localhost:8000/api/v1/monitors/1/uptime?window=24h'
curl 'http://localhost:8000/api/v1/incidents?page=1&page_size=20'
curl http://localhost:8000/api/v1/dashboard/summary
```

Valid uptime windows are `24h`, `7d` and `30d`. The API labels the metric `observed_check_uptime_percent` and returns null when no checks exist.

## Troubleshooting

- If API is unavailable, check `docker compose ps` and `docker compose logs backend db redis`.
- If migrations fail, verify the database is healthy and `DATABASE_URL` points to the Compose Postgres service.
- If monitors remain unchecked, check `docker compose logs worker beat redis` and ensure Beat has only one instance.
- URL validation rejects localhost, private/reserved IPs, credentials in URLs, unsupported schemes and hostnames resolving to non-public addresses.
- The manual check endpoint is rate limited to one request per monitor per 15 seconds.

## Security considerations and limitations

Only GET requests with a fixed User-Agent are sent; callers cannot add headers or select methods. Requests have explicit connect/total timeouts, redirects are not followed, responses are bounded by the configured limit, and user-facing error strings are sanitized. URL validation resolves DNS and rejects any non-global resolved address. A DNS answer can theoretically change between validation and the HTTP client's independent connection lookup (DNS rebinding); production-grade deployments should use a pinned-IP transport or enforce egress policy at the network layer. A hostname changing to private space between those operations is residual risk. Egress filtering should block private, link-local, loopback and metadata ranges. This initial version is single-tenant and has no authentication; do not expose it publicly without an auth and tenancy layer.

`observed_check_uptime_percent` is not continuous-availability SLO math: it weights each check equally and cannot infer failures between checks. Historical graph points only exist for recorded checks; no synthetic history is generated. Redirects are treated as the response received (3xx is successful by definition) and not followed.

## Interview demonstration

1. Start Compose and show the live, empty dashboard: metrics are zero/null and charts explain there is no history.
2. Add a publicly reachable URL from your own environment. Select a 1-minute interval and trigger **Check now**; show the persisted check and actual status/latency.
3. Explain that failure and recovery transitions are deterministic unit tests using mocked HTTP transport. In an interview, run `pytest -q` and show the repeated-failure test creates a single incident and the next healthy result resolves it. The mock is test-only and is not exposed to a deployed worker.
4. Show observed uptime and response-time history derived from the stored check records, then pause the monitor and explain the due-monitor scan skips inactive monitors.

## Project layout

```text
backend/       FastAPI app, SQLAlchemy models, Alembic migration, Celery tasks and pytest tests
frontend/      React/Vite UI, API client, charts, and Vitest tests
docker-compose.yml
.env.example
```
