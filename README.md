# Meridian AI

**AI-powered document intelligence and purchase-audit platform for finance teams.**

Meridian AI combines two workflows in one service:

1. **Document Q&A (RAG)**: upload policies, contracts and reports as PDFs, then ask questions in plain English. Answers are grounded in your documents; if the answer isn't there, the system says so.
2. **Multi-agent purchase audit**: submit a purchase request and three specialist agents (**risk**, **tax**, **internal control**) review it independently. A **CFO** step consolidates their findings into a single memo with a decision: *approved*, *conditional hold*, or *rejected*.

> The sample company, *Aldermoor Industries*, and all documents in `sample_docs/` are fictional. Audit checks rely on the model's general knowledge (only the FX-rate check uses a live data source), so treat output as decision support, not a compliance system.

---

## Features

- **Retrieval-augmented generation**: PDF ingestion, chunking, embeddings and top-k semantic retrieval with source-grounded answers.
- **Tool-using agents** built on LangChain v1 `create_agent`, with tools such as live FX-rate lookup.
- **Agent guard-rails via middleware**: PII redaction (emails, card numbers), model/tool call limits, automatic retries with backoff, and tool-call logging.
- **Two runtime modes behind one switch** (`APP_MODE`): fully local (Chroma on disk) or cloud (Vertex AI Vector Search and Cloud Storage). No code changes between them.
- **React + TypeScript frontend**: document upload, Q&A, audit and system-status tabs.
- **Structured JSON logging** (structlog) and typed request/response schemas (Pydantic).
- **Containerised and deployable**: single Docker image, GitHub Actions pipeline to Cloud Run.
- **Offline test suite**: runs without accounts or network using fake models.

## Architecture

```
Browser (React)
    │
    ▼
FastAPI backend ──────────────────────────────────────────────┐
    │                                                         │
    ├── Document Q&A (RAG)                                    │
    │     PDF → chunks → embeddings → vector store            │
    │     question → top-k chunks → Gemini → grounded answer  │
    │                                                         │
    └── Purchase audit (agents + middleware)                  │
          Risk ─┐                                             │
          Tax  ─┼─ tools ─► Gemini ─► CFO memo + decision     │
          Control┘                                            │
                                                              │
 Local mode:  Gemini API · Chroma · local disk  ◄─────────────┤
 Cloud mode:  Gemini · Vertex AI Vector Search · Cloud Storage ◄┘
```

Dependency direction: `api → rag, agent → config, logger`.

| Layer | Technology |
|---|---|
| Frontend | React 18, TypeScript, Vite, Tailwind CSS |
| API | FastAPI, Pydantic, structlog |
| LLM and embeddings | Gemini via `langchain-google-genai` |
| Agents | LangChain v1 (`create_agent`, middleware), LangGraph |
| Vector store | Chroma (local) / Vertex AI Vector Search (cloud) |
| Infra | Docker, Cloud Run, Secret Manager, Cloud Storage, GitHub Actions |

## Project structure

```
backend/
├── api/        FastAPI app, endpoints, request/response schemas
├── rag/        LLM, embeddings, vector store, ingestion, retrieval
├── agent/      Agents, prompts, tools, middleware
├── config/     Settings loaded from environment
├── logger/     Structured JSON logging
└── tests/      Pytest suite and runnable try_* scripts
frontend/       React + TypeScript UI
sample_docs/    Fictional PDFs for trying the app
docs/           In-depth guide and runnable examples
.github/workflows/deploy.yml   CI/CD to Cloud Run
```

## Getting started

**Prerequisites:** Python 3.12, Node.js 20+ (for the UI), and a free [Gemini API key](https://aistudio.google.com/apikey). No Google Cloud project is needed in local mode.

```bash
git clone https://github.com/mayank953/meridian-ai-learner.git
cd meridian-ai-learner

python3.12 -m venv .venv
source .venv/bin/activate              # Windows: .venv\Scripts\Activate.ps1
pip install -r requirements-dev.txt

cp .env.example .env                   # set GOOGLE_API_KEY, keep APP_MODE=local
uvicorn api.main:app --app-dir backend --reload --port 8080
```

API docs are at **http://localhost:8080/docs**.

Frontend (second terminal):

```bash
cd frontend && npm install && npm run dev      # http://localhost:3000
```

Docker:

```bash
docker build -t meridian-ai .
docker run -p 8080:8080 --env-file .env -v "$(pwd)/.meridian:/app/.meridian" meridian-ai
```

## API

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/health` | Liveness check |
| `GET` | `/api/status` | Active mode, models and store configuration |
| `POST` | `/api/rag/upload` | Upload a PDF |
| `GET` | `/api/rag/uploads` | List uploaded documents |
| `POST` | `/api/rag/ingest` | Chunk, embed and index uploaded documents |
| `POST` | `/api/rag/ask` | Ask a question over indexed documents |
| `POST` | `/api/agent/audit` | Run the multi-agent purchase audit |

Example audit request:

```bash
curl -X POST http://localhost:8080/api/agent/audit \
  -H "Content-Type: application/json" \
  -d '{"request_text": "Purchase of 200 PLC controllers from Takumi Controls Europe B.V. (Netherlands). Total 480,000 EUR. Origin JP, destination DE. FX rate quoted: 1 EUR = 163.5 JPY."}'
```

The response contains the risk, tax and control reports plus the CFO memo. An audit makes several model calls and can take a minute or more.

Document Q&A: upload the PDFs in `sample_docs/`, then ask, for example, *"What was Aldermoor Industries' total revenue in FY2024?"* (about EUR 2.84 billion).

## Configuration

Settings are read from `.env` (see [.env.example](.env.example)).

| Variable | Purpose |
|---|---|
| `APP_MODE` | `local` (Chroma, disk) or `cloud` (Vertex AI, GCS) |
| `GOOGLE_API_KEY` | Gemini API key (both modes) |
| `VERTEX_LLM_MODEL_NAME`, `LLM_TEMPERATURE` | Chat model and temperature |
| `AGENT_MAX_MODEL_CALLS`, `AGENT_MAX_TOOL_CALLS`, `AGENT_MAX_RETRIES` | Agent guard-rail limits |
| `GCP_PROJECT_ID`, `GCS_BUCKET_NAME`, `VECTOR_SEARCH_INDEX_ID`, `VECTOR_SEARCH_INDEX_ENDPOINT_ID` | Cloud mode only |

## Testing

```bash
pytest backend/tests
```

Tests need no accounts or network. They cover the API, a full local run (PDF → Chroma → answer) and an agent run through the middleware stack, using fake models.

## Deployment

Pushes to `main` trigger [.github/workflows/deploy.yml](.github/workflows/deploy.yml), which enables the required GCP APIs, provisions storage and Vector Search, builds the image and deploys to Cloud Run. Setup steps, required secrets and cost notes are in [DEPLOY.md](DEPLOY.md).

> Vertex AI Vector Search is billed hourly while the index endpoint is deployed, even when idle. Tear it down when not in use.

## Documentation

The [`docs/`](docs/) folder contains deeper material: the architecture, a file-by-file tour, RAG and agent internals, Docker and Cloud Run, troubleshooting, and runnable examples in [`docs/examples/`](docs/examples/README.md).

## License

No license has been specified yet. Add one before accepting external contributions.
