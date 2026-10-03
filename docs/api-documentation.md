# API Documentation

The backend exposes a JSON REST API plus one Server-Sent Events stream. It is documented with OpenAPI and browsable in
Swagger UI.

![Swagger UI](../screenshots/api-docs.png)

---

## Global specifications

* **Base URL (hosted demo):** `https://stressbugger-api.onrender.com`
* **Swagger UI:** [`/swagger-ui.html`](https://stressbugger-api.onrender.com/swagger-ui.html) · **OpenAPI JSON:** `/v3/api-docs`
* **Content type:** `application/json`
* **Authentication:** optional shared token. When the server sets `STRESSBUGGER_ACCESS_TOKEN`, every endpoint except
  `GET /api/health` needs `Authorization: Bearer <token>`. The SSE stream also accepts
  `?access_token=` because browsers cannot set headers on `EventSource`.
* **Errors:** every non-2xx response has the same body: `{ "error": "NOT_FOUND", "message": "…", "details": … }`.

| Status | When |
|---|---|
| `400` | validation failed (missing statement, limits out of range…) |
| `401` | access token missing or wrong |
| `404` | unknown session, counterexample, demo or Codeforces problem |
| `422` | the Codeforces problem is interactive (not supported) |
| `429` | `QUEUE_FULL`: too many waiting sessions |
| `502` | Codeforces could not be reached (bot check or outage) |
| `503` | `/api/health` when a required component is down |

---

## Endpoint summary

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/sessions` | create a session (starts it unless `autoStart=false`) |
| `GET` | `/api/sessions?limit=50` | list sessions, newest first |
| `GET` | `/api/sessions/{id}` | one session with settings, summary, programs and runs |
| `POST` | `/api/sessions/{id}/start` | start or restart a session |
| `POST` | `/api/sessions/{id}/cancel` | cancel a running session |
| `POST` | `/api/sessions/{id}/artifacts` | add your own generator, validator, checker, brute force or property checker |
| `GET` | `/api/sessions/{id}/counterexamples` | shrunk counterexamples with outputs and explanations |
| `GET` | `/api/sessions/{id}/mismatches` | every disagreement with its verdict and evidence |
| `GET` | `/api/sessions/{id}/events` | **SSE** live events (stored events replayed first) |
| `GET` | `/api/sessions/{id}/history?after=0` | stored events as JSON |
| `GET` | `/api/sessions/{sid}/counterexamples/{cid}/similar` | similar past bugs (pgvector or tag overlap) |
| `GET` | `/api/stats/bugs` | bug statistics: tags, verdicts, timeline, recent bugs |
| `GET` | `/api/codeforces/import?problem=1985A` | import a Codeforces problem by id or URL |
| `GET` | `/api/demos`, `/api/demos/{slug}` | the bundled demo problems |
| `GET` | `/api/health?toolchains=true` | readiness of every component |

---

## Endpoint details

### 1. Create a session
* **`POST /api/sessions`** → `201 Created`

```json
{
  "title": "Max Pair Product",
  "statement": "# Max Pair Product\n\nYou are given n non-negative integers ...\n\n## Examples ...",
  "solution": { "language": "cpp", "source": "#include <bits/stdc++.h>\n..." },
  "brute":    { "language": "python", "source": "n = int(input())\n..." },
  "mode": "STRESS",
  "settings": {
    "timeBudgetSec": 60,
    "timeLimitMs": 2000,
    "memoryLimitMb": 256,
    "compareMode": "auto",
    "maxCounterexamples": 2,
    "explain": true,
    "verifyFix": true
  }
}
```

```json
{ "sessionId": 1, "status": "QUEUED" }
```

| Field | Notes |
|---|---|
| `statement` | required, up to 100,000 characters; samples are read from it exactly |
| `solution.language` | `cpp`, `java` or `python` |
| `brute` | optional; without it the LLM writes two independent brute forces |
| `mode` | `STRESS` ("find my bug", default) or `HACK` (same pipeline, framed as breaking someone else's code) |
| `artifacts` | optional user-written `GENERATOR`, `VALIDATOR`, `CHECKER`, `BRUTE` or `PROPERTY` programs |
| `settings.timeBudgetSec` | 5 – 3600 (default 60) |
| `settings.timeLimitMs` / `memoryLimitMb` | 100 – 20000 ms / 16 – 1024 MB |
| `settings.compareMode` | `auto` (detects float tolerance), `exact` (tokens) or `float` with `absEps` / `relEps`; optional `caseInsensitive` |
| `settings.strategies` | optional subset of `random`, `boundary`, `overflow`, `exhaustive`, `structural`, `adversarial`, `multitest`, `tle` |
| `settings.maxTests` | optional test count, capped by the server |

### 2. List sessions
* **`GET /api/sessions?limit=50`**

```json
[
  {
    "id": 3, "title": "Watermelon", "mode": "STRESS",
    "status": "DONE", "phase": "DONE", "solutionLanguage": "cpp",
    "topVerdict": "SOLUTION_BUG", "counterexamples": 1,
    "createdAt": "2026-10-03T08:12:56.015Z", "finishedAt": "2026-10-03T08:16:12.820Z"
  }
]
```

### 3. Get a session
* **`GET /api/sessions/{id}`**: the problem, settings, extracted constraints, samples, every program with its status
  and trust score, every run, and a summary. The summary makes each result reproducible:

```json
"summary": {
  "tests": 129,
  "disagreements": 5,
  "counterexamples": 1,
  "verdicts": { "SOLUTION_BUG": 5 },
  "topVerdict": "SOLUTION_BUG",
  "verdictSummary": "Your solution has a bug: see the counterexample.",
  "reference": "llm_brute-20",
  "oracles": [
    { "label": "LLM brute #20 (python)", "status": "TRUSTED", "trustScore": 1.0 },
    { "label": "LLM brute #21 (cpp)",    "status": "TRUSTED", "trustScore": 1.0 }
  ],
  "environment": {
    "llm": { "provider": "gemini", "model": "gemini-3.5-flash" },
    "sandbox": { "cpp": "g++ 13.3.0", "java": "openjdk 21.0.12.1", "python": "Python 3.12.3" },
    "limits": { "timeLimitMs": 1000, "memoryLimitMb": 64, "sandboxPidsLimit": 256 },
    "prompts": { "generator": 6, "validator": 5, "brute": 2, "explain": 1, "fix": 2 }
  }
}
```

### 4. Counterexamples
* **`GET /api/sessions/{id}/counterexamples`**

```json
[
  {
    "id": 3,
    "verdict": "SOLUTION_BUG",
    "confidence": 0.95,
    "reasoning": [
      "2 trusted oracles (LLM brute #20 (python), LLM brute #21 (cpp)) agree with each other and disagree with your solution.",
      "The validator accepts the input."
    ],
    "input": "2\n",
    "originalBytes": 2,
    "shrunkBytes": 2,
    "shrinkSteps": [ { "index": 0, "method": "original", "bytes": 2, "description": "2 bytes (exhaustive w=100)" } ],
    "outputs": [
      { "role": "solution",  "label": "your solution",          "result": { "output": "YES\n", "status": "OK", "timeMs": 5,   "exitCode": 0 } },
      { "role": "reference", "label": "LLM brute #20 (python)", "trusted": true, "result": { "output": "NO\n", "status": "OK", "timeMs": 116, "exitCode": 0 } }
    ],
    "explanation": {
      "summary": "The solution incorrectly outputs YES for a watermelon weight of 2.",
      "rootCause": "The code only checks if the weight is even, but fails to handle the edge case ...",
      "citedLines": [9],
      "fixStatus": "VERIFIED",
      "tags": ["edge-case"]
    }
  }
]
```

Each output is recorded twice (`result` and `rerun`) to detect nondeterminism, with hashes of the raw and normalised
output.

### 5. Live events (Server-Sent Events)
* **`GET /api/sessions/{id}/events`** (`text/event-stream`), resumable with `Last-Event-ID` or `?after=<seq>`.

```
id: 2
event: PHASE_CHANGED
data: {"sessionId":3,"seq":2,"type":"PHASE_CHANGED","payload":{"phase":"PARSE"},"at":"2026-10-03T08:12:57.725Z"}
```

Event types: `PHASE_CHANGED`, `PLAN`, `STEP`, `ARTIFACT_READY`, `GATE_PASSED`, `GATE_FAILED`, `STRATEGY_STARTED`,
`PROGRESS`, `MISMATCH_FOUND`, `TRIAGE_RESULT`, `SHRINK_STEP`, `COMPLEXITY_READY`, `EXPLANATION_READY`,
`LLM_ACTIVITY`, `LOG`, `DONE`, `ERROR`. `PROGRESS` and `LLM_ACTIVITY` are live only; the rest are stored and replayed.

### 6. Codeforces import
* **`GET /api/codeforces/import?problem=4A`** (an id or a problem URL)

```json
{
  "id": "4A",
  "url": "https://codeforces.com/problemset/problem/4/A",
  "title": "Watermelon",
  "timeLimitMs": 1000,
  "memoryLimitMb": 64,
  "sampleCount": 1,
  "statement": "# Watermelon\n\nTime limit per test: 1 second. Memory limit per test: 64 megabytes.\n\n..."
}
```

### 7. Bug statistics
* **`GET /api/stats/bugs`**

```json
{
  "days": 90, "totalBugs": 3, "sessions": 3, "vectorSearch": true,
  "byTag": [ { "tag": "edge-case", "count": 1 }, { "tag": "overflow", "count": 1 } ],
  "byVerdict": [ { "verdict": "SOLUTION_BUG", "count": 3 } ],
  "timeline": [ { "day": "2026-10-03", "count": 3 } ],
  "recent": [ { "sessionId": 3, "title": "Watermelon", "verdict": "SOLUTION_BUG", "fixStatus": "VERIFIED", "tags": ["edge-case"] } ]
}
```

### 8. Health
* **`GET /api/health?toolchains=true`**: `UP`, `DEGRADED` (e.g. the LLM model is still downloading) or `DOWN`
  (HTTP 503). With `toolchains=true` it compiles and runs a test program in every language inside the sandbox.

```json
{
  "status": "UP",
  "components": {
    "backend":        { "status": "OK", "detail": "running" },
    "database":       { "status": "OK", "detail": "PostgreSQL reachable" },
    "redis":          { "status": "OK", "detail": "Redis reachable (PONG)" },
    "sandbox_image":  { "status": "OK", "detail": "process sandbox: programs run as child processes (no Docker on this host)" },
    "llm_model":      { "status": "OK", "detail": "gemini-3.5-flash is available" },
    "compiler_cpp":   { "status": "OK", "detail": "g++ 13.3.0 (compiled and ran a test program in the sandbox)" },
    "runtime_java":   { "status": "OK", "detail": "openjdk version \"21.0.12.1\" (compiled and ran a test program in the sandbox)" },
    "runtime_python": { "status": "OK", "detail": "Python 3.12.3 (compiled and ran a test program in the sandbox)" }
  }
}
```

Health reports statuses and versions only, never URLs or keys.
