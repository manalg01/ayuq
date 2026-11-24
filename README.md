# SUGARFREE

SUGARFREE is a diabetes risk support app with:

- live vitals logging
- rule + ML hybrid risk scoring
- critical alerts and AI triage summaries
- explainability panels (why the score changed)
- doctor-sharing workflow for urgent cases
- AI chat assistant (with voice + report upload support)

## Repository

- GitHub: [https://github.com/mohammedtalha15/cse-inception](https://github.com/mohammedtalha15/cse-inception)

## Quick Start (Local)

### 1) Prerequisites

- Node.js 20+
- npm
- Python 3.10+ (3.11+ recommended)

### 2) Install frontend deps

```bash
npm install
```

### 3) Configure environment

Copy the template:

```bash
cp .env.local.example .env.local
```

Optional keys:

- `GEMINI_API_KEY` for AI explanations/chat
- `DATABASE_URL` only when you want PostgreSQL locally (otherwise SQLite is used automatically)

### 4) Run everything (web + API)

```bash
npm run dev
```

This single command starts:

- Next.js app at `http://localhost:3000`
- FastAPI backend at `http://127.0.0.1:8000`

## Experience Links (Local)

After `npm run dev`, open:

- Home: [http://localhost:3000](http://localhost:3000)
- Log vitals: [http://localhost:3000/enter-data](http://localhost:3000/enter-data)
- Live dashboard: [http://localhost:3000/dashboard](http://localhost:3000/dashboard)
- Alerts & AI: [http://localhost:3000/alerts](http://localhost:3000/alerts)
- Predictor: [http://localhost:3000/predict](http://localhost:3000/predict)

## How Data Flows

1. User submits vitals in `Log vitals`.
2. Frontend posts to backend `/reading`.
3. Backend computes:
   - rule score
   - ML score
   - hybrid score
   - alert type + explanation
4. Dashboard reads latest readings.
5. Alerts page shows critical/early events and AI guidance.

## Important Behavior

- Critical alert band is triggered at risk basis `>= 40`.
- Alerts and dashboard are patient-ID based (`P001`, `P002`, etc.).
- If API is unreachable, the dashboard shows an API connection error and does not substitute generated readings.
- When `model.pkl` and `scaler.pkl` are absent, the risk engine uses its deterministic score stub. This is not a trained model.

## Troubleshooting

- If backend is not reachable:
  - ensure `npm run dev` is running
  - check terminal output for FastAPI startup errors
- If AI text is missing:
  - set `GEMINI_API_KEY` in `.env.local`
- If dashboard/alerts look empty:
  - log at least one vitals entry for the same patient ID

## Scripts

- `npm run dev` - run Next.js + FastAPI together
- `npm run dev:web` - run only frontend
- `npm run dev:api` - run only backend
- `npm run build` - production build
- `npm run start` - run production frontend

## Persistent production deployment (Vercel + Render)

The Next.js frontend and FastAPI API are separate services. `render.yaml` provisions a Render FastAPI web service and a persistent Render PostgreSQL database. The backend runs `Base.metadata.create_all()` at startup for the existing `readings`, `alerts`, `profiles`, and `simulator_state` tables. `DATABASE_URL` is read only by FastAPI; it is never placed in browser code. Locally, leaving it unset uses `backend/sugarfree.db`.

### Render backend and database

1. Push this repository to GitHub and create a Render Blueprint from the repository, using its root `render.yaml`.
2. The Blueprint creates the `ayuq-api` service and `ayuq-postgres` database. PostgreSQL is provisioned on the paid `basic-256mb` plan so the database is persistent; review current Render pricing before provisioning.
3. Set the Render service environment variable `CORS_ORIGINS` to the Vercel site origins if browser clients will call FastAPI directly (for example, `https://ayuq.vercel.app`). The current frontend calls FastAPI through its same-origin Next.js proxy, so CORS is not used for the normal app flow.
4. Optionally set `GEMINI_API_KEY` on Render for Gemini explanations/chat. Without it, the existing explanation fallback is used.
5. Deploy and verify `https://<render-service>.onrender.com/health`; a ready response includes `{"ok":true,"service":"sugarfree-api","database":"ok"}`. A database failure returns HTTP 503.

Required backend host environment:

- `DATABASE_URL`: supplied by the Blueprint from the private Render PostgreSQL connection string. If creating services manually, use the database's **internal** PostgreSQL URL here.
- `CORS_ORIGINS`: optional for the same-origin proxy flow; otherwise comma-separated exact frontend origins.
- `GEMINI_API_KEY`: optional.
- `PORT`: supplied by Render; the start command binds to it.

### Vercel frontend

Import this repository into Vercel and set the project Root Directory to the repository root. In **Settings → Environment Variables**, set:

- `BACKEND_URL` = the FastAPI service base URL, such as `https://ayuq-api.onrender.com` (no `/health` suffix).

Apply it to Production and Preview as needed, then redeploy; the proxy reads `BACKEND_URL` at request time. Do not set `DATABASE_URL` or any PostgreSQL credential in Vercel. No `NEXT_PUBLIC_*` backend URL is required; browser requests use `/api/backend/*` on the Vercel origin.

### Local run (PowerShell)

From the repository root, install dependencies:

```powershell
npm ci
py -3 -m venv backend\venv
backend\venv\Scripts\python.exe -m pip install -r backend\requirements.txt
```

Start FastAPI in terminal 1:

```powershell
Set-Location backend
.\venv\Scripts\python.exe -m uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

Start Next.js in terminal 2 from the repository root:

```powershell
npm run dev:web
```

Then open `http://localhost:3000`. With no `DATABASE_URL`, SQLite data persists in `backend/sugarfree.db`. Verify the API with `Invoke-RestMethod http://127.0.0.1:8000/health`.

For local PostgreSQL testing, set `DATABASE_URL` in the backend process environment before starting FastAPI. Keep that value in a backend-only environment file or secret store; do not add it to `NEXT_PUBLIC_*` variables or frontend configuration.
