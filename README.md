# CIOrch

CIOrch is an AI-assisted CI/CD incident console. It collects failed workflow or deployment logs, investigates likely causes with repository context, streams diagnosis progress to a dashboard, and can prepare a pull request with proposed fixes.

## Features

- Diagnose GitHub Actions failures using a repository and workflow run ID.
- Analyze supported Vercel deployment failures from signed webhooks.
- Use Groq to classify failures, explain root causes, and suggest fixes and validation steps.
- Inspect relevant repository files when log analysis needs code context.
- Stream agent progress over WebSockets and keep diagnosis history in SQLite.
- Preview proposed code changes and create a GitHub pull request.
- Use the included React dashboard or the lightweight dashboard served by the backend.

## Architecture

- `cicd-ai-backend/`: FastAPI API, orchestration and diagnosis pipeline, GitHub/Vercel clients, SQLite storage, and a built-in static dashboard.
- `cicd-ai-frontend/`: React and Vite dashboard. Its development server proxies API and WebSocket requests to the backend.

## Requirements

- Python 3.10 or newer
- Node.js 18 or newer and npm
- A Groq API key
- A GitHub token for authenticated repository/workflow access and pull request creation

Vercel credentials are only needed when processing Vercel webhooks.

## Setup

From the repository root, create and activate a Python virtual environment, then install the backend dependencies:

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
cd cicd-ai-backend
Copy-Item .env.example .env
pip install -r requirements.txt
```

On macOS or Linux, activate the environment with `source .venv/bin/activate`, then run `cd cicd-ai-backend`, `cp .env.example .env`, and `pip install -r requirements.txt` from the repository root as shown above.

Set the values in `cicd-ai-backend/.env`:

| Variable | Required | Description |
| --- | --- | --- |
| `GROQ_API_KEY` | Yes | API key used for AI diagnosis. |
| `GITHUB_TOKEN` | For GitHub access | Token used to read workflow/repository data and create pull requests. Grant only the repository permissions the app needs. |
| `GROQ_MODEL` | No | Groq model name; defaults to `llama-3.3-70b-versatile`. |
| `VERCEL_TOKEN` | For Vercel | Token used to fetch deployment details and events. |
| `VERCEL_WEBHOOK_SECRET` | For Vercel | Secret used to validate Vercel webhook signatures. |
| `DATABASE_PATH` | No | SQLite database path; defaults to `diagnosis_history.db` in the backend working directory. |

Start the API from `cicd-ai-backend/`:

```powershell
uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

In another terminal, start the React dashboard from the repository root:

```powershell
cd cicd-ai-frontend
npm ci
npm run dev
```

Open the Vite URL shown in the terminal (normally <http://localhost:5173>). The backend's built-in dashboard is also available at <http://localhost:8000>. Interactive API documentation is at <http://localhost:8000/docs>.

## Trigger a diagnosis

Post a repository and GitHub Actions run ID to the webhook endpoint. The API returns a diagnosis ID while the analysis runs asynchronously.

```sh
curl -X POST http://localhost:8000/api/webhook \
	-H "Content-Type: application/json" \
	-d '{"source":"github","repo":"owner/repository","run_id":"123456789","ref":"main"}'
```

The dashboard displays saved diagnoses and live progress. The backend also exposes:

| Endpoint | Purpose |
| --- | --- |
| `GET /health` | Health check. |
| `POST /api/webhook` | Queue a CI diagnosis. |
| `POST /api/vercel/webhook` | Receive signed Vercel deployment events. |
| `GET /api/diagnoses` | List diagnosis history. |
| `GET /api/diagnosis/{diagnosis_id}` | Retrieve one diagnosis. |
| `GET /api/events/{diagnosis_id}` | Read events for a diagnosis. |
| `WS /ws` | Stream diagnosis progress. |
| `POST /api/diagnosis/{diagnosis_id}/preview-pr` | Preview proposed pull request changes. |
| `POST /api/diagnosis/{diagnosis_id}/create-pr` | Create a pull request from the diagnosis. |

## Security

Keep `.env` files and access tokens out of Git. Use narrowly scoped credentials, verify webhook signatures, and avoid exposing this development server directly to the public internet. Diagnosis results may contain excerpts from CI logs and repository files, so handle them according to your organization's data policies.