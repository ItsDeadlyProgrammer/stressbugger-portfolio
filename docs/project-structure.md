# Project Structure

StressBugger is a monorepo with three runtime parts (backend, frontend, sandbox) plus deployment files, demo problems
and test scripts.

## Repository layout

| Folder | Content |
|---|---|
| `backend/` | Spring Boot 3.5 service (Java 21, Maven) |
| `frontend/` | React 18 dashboard (TypeScript, Vite) |
| `sandbox/` | sandbox image, `harness.py`, output comparator, sandbox attack tests |
| `deployment/` | Render, Netlify and Linux-server files, compose overlays, operations scripts |
| `demo-problems/` | 7 complete scenarios with expected outcomes and recorded LLM answers |
| `scripts/` | `smoke_test.py` and `e2e_demo.py`, run against a running stack |
| `docs/` | architecture, trust and triage, security, decisions, development, troubleshooting |
| `docker-compose.yml` | the whole stack on one machine |
| `setup.ps1`, `setup.sh` | one-command setup for Windows and Linux/macOS |
| `.github/workflows/ci.yml` | CI on every push |

## Backend packages (`com.stressbugger`)

| Package | Responsibility |
|---|---|
| `api` | REST controllers, DTOs, error handling |
| `session` | queue, worker pool, cancellation, pipeline orchestration (`StressPipeline`) |
| `llm` | `LlmService`, the Ollama / Gemini / Anthropic / Mock clients, prompt templates, activity heartbeats |
| `oracle` | qualification gates (`ArtifactPreparer`), trust (`QualificationService`), verdicts (`TriageEngine`) |
| `strategy` | test strategies, TLE hunter, complexity analysis |
| `shrink` | input format model and shrinker |
| `explain` | explanations, fix verification, bug tags, embeddings, similar bugs |
| `execution` | sandbox backends (Docker / process) and the harness protocol |
| `codeforces` | problem import from codeforces.com |
| `events` | event bus, SSE emitters, stored event log |
| `persistence` | JPA entities and repositories |
| `health` | `/api/health` with per-language toolchain checks |
| `demo` | bundled demo problems |
| `config` | security (access token, CORS), OpenAPI, settings |

Resources: `db/migration` (Flyway schema), `db/vector` (optional pgvector migration), `prompts/*.md` (11 versioned
prompt templates) and `prompts/schemas/*.json` (their JSON Schemas).

## Frontend

| Path | Content |
|---|---|
| `pages/NewSession.tsx` | problem import, statement, solution and brute-force editors, advanced settings |
| `pages/SessionDashboard.tsx` | live phases, strategies table, programs, triage, shrinking, log |
| `pages/CounterexampleView.tsx` | verdict, evidence, every output, explanation, fix diff, similar bugs |
| `pages/History.tsx`, `Stats.tsx`, `Status.tsx` | session history, bug statistics, component health |
| `api/useSessionEvents.ts`, `api/liveState.ts` | SSE subscription folded into live state |
| `components/` | phase stepper, verdict badges, code editor, complexity chart, shrink timeline |

## Sandbox

| Path | Content |
|---|---|
| `sandbox/Dockerfile` | g++ 13, Temurin 21, Python 3.12, an unprivileged user for programs |
| `sandbox/harness/harness.py` | compile, run with limits, batch stress loop, shrink batches; JSON in, JSON Lines out |
| `sandbox/harness/compare.py` | output comparison (tokens, floats with tolerance, case-insensitive) |
| `sandbox/test/run_harness_test.py` | harness tests and sandbox attacks |

## Docker Compose services

`postgres` (pgvector), `redis`, `ollama`, `ollama-pull` (one-shot model download), `sandbox` (build-only image),
`backend` and `frontend` (nginx). Data lives in the `pgdata`, `redisdata` and `ollama` volumes.

## Design advantages

- **Clear trust boundary:** only the `execution` package talks to the sandbox, and only the `llm` package talks to a
  model.
- **Provider independence:** swapping Ollama for Gemini or Anthropic is one setting.
- **Same pipeline everywhere:** the laptop, the Linux server and the free hosted demo run the same code; only the
  execution backend and the LLM provider change.
