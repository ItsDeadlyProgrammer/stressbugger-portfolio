# System Architecture Diagram

![StressBugger system architecture](architecture.png)

## Components

1. **React dashboard (Netlify):** creates sessions over REST and follows them over Server-Sent Events. Pages: New
   session, live session dashboard, counterexample view with fix diff, history, bug stats and status.
2. **REST API + SSE:** session commands, the live event stream (resumable with `Last-Event-ID`), Codeforces import,
   statistics and health.
3. **Queue + worker pool:** sessions wait in a Redis list; a fixed pool of worker threads runs them, each exactly once.
4. **Stress pipeline:** Parse → Generate → Qualify → Stress → Triage → Shrink → Explain → Verify fix.
5. **Oracles:** qualification gates, trust scores, the majority vote and evidence-based triage into 11 verdicts.
6. **Strategies + shrinker:** 8 seeded test strategies, the TLE hunter with complexity fit, and an execution-only
   shrinker.
7. **Explain + fix:** root cause with cited lines, fix verification, bug tags and 256-dimension embeddings.
8. **LLM service:** the only gateway to a model: prompt templates, JSON-schema validation, retries, Redis cache and a
   log of every call.
9. **Execution backend:** one hardened Docker container per session, or a child process on hosts without Docker.
10. **PostgreSQL 16 + pgvector:** the source of truth for problems, sessions, programs, mismatches, counterexamples,
    explanations and the event log.
11. **Redis 7:** the job queue, cancel flags and the LLM answer cache.
12. **Sandbox (untrusted code):** `harness.py` compiles once and runs thousands of tests with time, CPU, memory and
    output limits, as a separate user, with no network.
13. **External services:** the LLM provider (Ollama, Gemini, Anthropic or a mock) and codeforces.com for imports.

## One session

![Pipeline](pipeline.png)

- **Purple steps** (Parse, Generate, Explain + fix) ask the LLM for programs or prose.
- **Every other step** is decided by running code in the sandbox.
- A program becomes a **trusted oracle** only after it compiles, passes every sample, its validator rejects broken
  inputs, and it agrees with the majority on ~150 small inputs.
- A fix is **verified** only after it passes the counterexample, every sample, every earlier failing input and a fresh
  stress run.
