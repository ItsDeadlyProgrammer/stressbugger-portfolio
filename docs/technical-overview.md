# Technical Overview

## The problem

In competitive programming, a solution that passes the samples can still fail on a hidden test. The standard way to
find the failing input is **stress testing**: write a random test generator and a slow but obviously correct brute
force, run both on thousands of inputs, and stop at the first difference.

It works, but it is tedious, and it has a trap: if the brute force or the generator is wrong, the "bug" it reports is
not real. StressBugger automates the whole loop and closes that trap.

## The approach

| Step | Who does it | Why |
|---|---|---|
| Write the generator, validator, checker and brute forces | the LLM | creative work that changes with every problem |
| Decide whether those programs can be trusted | execution | compile, samples, negative tests, cross-agreement |
| Run thousands of tests | execution | a Python harness in a hardened sandbox |
| Decide what each disagreement means | a deterministic triage engine | evidence and confidence, never the model's opinion |
| Shrink the failing input | execution | each candidate must still fail the same way |
| Explain the bug and propose a fix | the LLM | prose and a patch |
| Decide whether the fix works | execution | counterexample, samples, earlier failures, fresh stress run |

**The LLM proposes. The sandbox proves.** A weak or wrong model can only produce rejected programs or an unverified
fix; it cannot produce a wrong verdict.

## Technology stack

### Backend: Java 21 + Spring Boot 3.5
- **Spring Web MVC** for the REST API and `SseEmitter` for the live event stream.
- **Spring Data JPA + Flyway** for a versioned PostgreSQL schema.
- **Spring AI 1.1** for the Anthropic client; Ollama and Gemini have small dedicated HTTP clients. Ollama answers
  are streamed, so the dashboard can show tokens as they are written.
- **networknt json-schema-validator** checks every LLM answer against a JSON schema.
- **Jsoup** parses Codeforces problem pages.
- **springdoc OpenAPI** publishes the Swagger UI.
- **Worker pool**: a fixed number of worker threads (`STRESSBUGGER_WORKERS`) take sessions from a Redis list.

### Frontend: React 18 + TypeScript
- **Vite** build, **Tailwind CSS 4** styling, a dark UI designed for long debugging sessions.
- **Monaco editor** for code input and side-by-side diffs of suggested fixes.
- **TanStack Query** for server state; a custom `useSessionEvents` hook folds the SSE stream into live state.
- **Recharts** for the complexity chart and bug statistics.

### Data
- **PostgreSQL 16** is the single source of truth: problems, sessions, every program, every mismatch, counterexamples,
  explanations and the live event log.
- **pgvector** stores a 256-dimension embedding per explanation for "similar past bugs".
- **Redis 7** holds only the job queue, cancel flags and the LLM answer cache. Losing it loses no results.

### Sandbox
- One **Docker image for all languages**: g++ 13 (`-O2 -std=c++17`), Eclipse Temurin 21 and CPython 3.12, because one
  batch usually mixes languages (Python generator, C++ solution, Java brute force).
- A **Python harness** receives one JSON job on stdin and streams JSON Lines results. It runs the whole fast loop
  (generate → validate → run solution → run reference → compare) without a round trip to the backend.

### LLM providers
| Provider | Use |
|---|---|
| Ollama (`qwen2.5-coder:7b` by default, 3b/14b optional) | local, free, offline; the default |
| Gemini | free tier; used by the hosted demo |
| Anthropic | optional cloud provider |
| Mock | replays recorded answers for the 7 demo problems: offline demos and end-to-end tests |

## Engineering highlights

- **Trust is earned by execution.** LLM-written oracles pass qualification gates and a majority vote before they are
  used ([trust and triage](trust-and-triage.md)).
- **Reproducible tests.** Every test is described as `gen <seed> <strategy> key=value…`, so any failing input can be
  regenerated exactly.
- **Measured, honest progress.** Long LLM calls stream a heartbeat with elapsed time and tokens written. Time estimates
  come only from earlier answers to the same prompt and are labelled as estimates.
- **Defence in depth.** Container limits, a separate user for untrusted code, an access token, CORS allow-list and a
  queue limit ([security](sandbox-and-security.md)).
- **Runs anywhere.** One `setup` script for a laptop, a Linux server overlay with automatic HTTPS, and a free hosted
  mode for platforms without Docker ([deployment](setup-and-deployment.md)).
