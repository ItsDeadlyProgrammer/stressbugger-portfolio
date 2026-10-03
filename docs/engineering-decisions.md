# Engineering Decisions & Technical Design

The key choices, why they were made, the trade-offs they bring, and the known limitations.

---

## Core principle: the LLM proposes, the sandbox proves

* **Decision:** the LLM writes programs and prose only. Trust, verdicts, shrinking and fix verification are decided by
  executing code.
* **Rationale:** LLMs are good at writing a brute force or a generator from a statement, and unreliable at judging
  whether code is correct. Splitting the roles keeps the creative part and removes the unreliable part from every
  decision.
* **Consequence:** a weak model makes the tool slower (more rejected programs), never wrong. Every claim on screen
  links to the input and outputs behind it.

---

## Architecture

### 1. Self-contained sandbox instead of an external judge
* **Decision:** StressBugger has its own hardened sandbox and batch harness.
* **Rationale:** an early version delegated execution to an existing online judge. It ran code without isolation and
  needed one HTTP call per test, far too slow for thousands of tests. The batch harness runs the whole
  generate → validate → run → compare loop in one call (~1,000 tests per minute).

### 2. Server-Sent Events instead of WebSockets
* **Decision:** live data flows over SSE; commands are plain REST.
* **Rationale:** the dashboard only receives live data, so a one-way stream is enough. SSE works through proxies, has
  built-in reconnection and resumes with `Last-Event-ID`.

### 3. Stored events
* **Decision:** every event except high-volume progress is stored in `session_events`.
* **Rationale:** a dashboard opened late, or after a page reload, replays the whole session and then follows it live.

### 4. One backend instance, Redis queue, PostgreSQL as truth
* **Decision:** jobs go through a Redis list to a pool of worker threads; PostgreSQL holds all state.
* **Rationale:** simple to run on one machine or a free host. On startup, running sessions are marked `FAILED` and
  queued ones are re-queued; a conditional update makes sure each session runs exactly once.
* **Trade-off:** no horizontal scaling of workers (see limitations).

### 5. Plain ids, no JPA associations
* **Decision:** entities store numeric foreign keys, not mapped relationships.
* **Rationale:** a session runs for minutes outside any single transaction; plain ids avoid lazy-loading failures.

---

## Execution

### 6. One sandbox image for all languages
* **Decision:** one image with g++ 13, Java 21 and Python 3.12.
* **Rationale:** a single batch usually mixes languages (Python generator, C++ solution, Java brute force).

### 7. Docker CLI and one container per session
* **Decision:** the backend calls the `docker` CLI; each session has one long-lived container and each batch is one
  `docker exec`.
* **Rationale:** no socket differences on Windows, no extra library, and container start-up is paid once per session.

### 8. Reproducible tests
* **Decision:** every test is `gen <seed> <strategy> key=value…`.
* **Rationale:** any failing input can be regenerated exactly; large inputs are stored as a preview plus the command.

### 9. Time-budgeted stress phase
* **Decision:** the stress phase gets a time budget (default 60 s): 60% small tests, 25% large tests, 15% TLE hunting,
  stopping early once enough distinct bugs are found.
* **Rationale:** predictable session length on any machine, with an optional test count per session.

### 10. Batched shrinking, bisected timeouts
* **Decision:** each shrink round checks up to 24 candidates in one sandbox call; timeouts are bisected by size, not
  shrunk element by element.
* **Rationale:** a `docker exec` is the expensive part; shrinking a TLE input element by element would cost one full
  time limit per candidate.

### 11. A separate user for untrusted code
* **Decision:** untrusted programs run as uid 10002 and are wiped after each run.
* **Rationale:** a program must not be able to tamper with other programs or the recorded evidence.

### 12. Process mode for free hosts
* **Decision:** `STRESSBUGGER_SANDBOX_MODE=process` runs the harness as a child process with the same per-run limits.
* **Rationale:** free container platforms offer no Docker inside. Secrets are moved out of the programs' reach. It is
  meant for a public demo, not as the default.

---

## LLM

### 13. One LLM gateway
* **Decision:** `LlmService` fills prompt templates, demands JSON matching a schema, retries once with the error,
  backs off on overload, caches answers in Redis and logs every call.
* **Rationale:** one place for reliability and observability; providers are interchangeable.

### 14. Self-repair loop
* **Decision:** every LLM-written program goes through compile → check → feedback, up to 3 attempts.
* **Rationale:** most first attempts fail on small details (a wrong format, an off-by-one in a bound); feeding the exact
  error back fixes them cheaply. After 3 attempts the program is rejected and the session continues.

### 15. Local model by default
* **Decision:** Ollama with `qwen2.5-coder:7b`, answers constrained to the JSON schema.
* **Rationale:** free, private and offline. Gemini or Anthropic are one setting away.

### 16. Measured progress, not invented progress
* **Decision:** LLM calls stream a heartbeat with tokens written. Estimates come only from earlier answers to the same
  prompt and are labelled; a stalled call is flagged.
* **Rationale:** on a CPU a single answer can take minutes; the user should see real activity, not a fake bar.

### 17. Cleaning generator plans
* **Decision:** the backend gives exact bounds per strategy, normalises the plan and judges each strategy separately.
* **Rationale:** real 7B-model runs produced plans that ignored arguments or used huge sizes in "small" tests. This cut
  demo 01 from 19+ minutes to 11.5 minutes on a CPU.

---

## Correctness

### 18. Codeforces import by page parsing
* **Decision:** the public problem page is parsed (with a mirror fallback) into the same Markdown a user would paste.
* **Rationale:** the Codeforces API has no statements. Samples are copied verbatim, never rewritten by the LLM.

### 19. One verified property checker per batch
* **Decision:** the LLM may propose up to 3 property checkers; the first one verified against the trusted oracle on
  small inputs is used for large inputs.

### 20. Bug embeddings without an API
* **Decision:** a deterministic 256-dimension feature-hashing embedding stored with pgvector.
* **Rationale:** similar-bug search works offline and with any LLM, at no cost.

### 21. A strict definition of "verified fix"
* **Decision:** counterexample + every sample + every earlier failing input + 300 fresh tests.
* **Rationale:** an LLM fix that only passes the one counterexample is often wrong elsewhere; the dashboard shows
  exactly which check failed.

### 22. Validators get negative tests
* **Decision:** a validator must reject broken inputs (empty, a word instead of a number, out-of-range values), not
  only accept the samples.
* **Rationale:** a validator that accepts everything would let a broken generator produce fake bugs.

---

## Security & operations

### 23. Access token, CORS and queue limit
* One shared token protects the API on a public server, CORS is off unless configured, and the waiting queue is
  bounded (HTTP 429 beyond it).

### 24. Health and reproducibility
* `/api/health` checks every component, including compiling and running a program in each language. Each session
  records toolchain versions, model and prompt versions.

---

## Known limitations

- **Single backend instance:** workers are co-located with the API; scaling means a bigger machine.
- **TLE verdicts depend on machine load** (confidence 0.9).
- **Inputs over 64 KiB** are stored as a preview plus the command that regenerates them.
- **The shrinker understands** ints, arrays, strings, lines, grids and repeats; other layouts fall back to simpler steps.
- **Interactive problems are not supported.**
- **The free hosted demo** is slow (shared CPU), sleeps when idle and uses the weaker process-mode sandbox.

## Future improvements

- Separate worker processes that scale independently of the API.
- gVisor as the default sandbox runtime.
- Interactive problems through a two-process harness.
- A shared bug memory across users, with opt-in.
