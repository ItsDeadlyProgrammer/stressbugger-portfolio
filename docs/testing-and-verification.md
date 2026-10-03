# Testing & Verification

A tool that tells people their code is wrong has to be right itself. StressBugger is tested at every layer, including
attacks on its own sandbox and end-to-end runs of real problems.

## Test suites

| Layer | Suite | What it covers |
|---|---|---|
| Sandbox | `sandbox/harness/test_compare.py` | the output comparator against shared test vectors |
| Sandbox | `sandbox/test/run_harness_test.py` | every harness command, plus attacks: memory hogs, endless output, infinite loops, a fork bomb, a detached daemon, file tampering, network access |
| Backend | `mvnw test` | **97 unit tests**: triage rules, shrinker, input format, generator plans, complexity fit, sample parsing, validator negative tests, LLM clients and prompt templates, Codeforces parsing, embeddings, access token |
| Backend | `mvnw verify` | **22 integration tests** with real PostgreSQL, Redis and the sandbox (Testcontainers) |
| End-to-end | `DemoProblemsIT`, `SessionApiIT` | all 7 demo problems through the REST API with the mock LLM: verdicts, shrinking, tags, verified fixes |
| Frontend | `npm test` | **36 component tests** (Vitest + Testing Library) and the TypeScript type check |
| Stack | `scripts/smoke_test.py` | a running stack: health, a session, its events |
| Real LLM | `scripts/e2e_demo.py`, `RealLlmDemoIT` | demos against a real model (Ollama, Gemini or Anthropic) |

## Continuous integration

GitHub Actions runs on every push: the sandbox tests, script checks, `mvnw verify` (unit, integration and demo
end-to-end tests) and the frontend tests with the type check.

## Rules that are pinned by tests

- A high-confidence `SOLUTION_BUG` is **never** claimed on the word of a single unverified brute force.
- When your brute force is wrong and your solution is right, the verdict is `BRUTE_BUG`, never `SOLUTION_BUG`
  (demo 04).
- A "print any valid answer" problem produces `MULTIPLE_VALID_ANSWERS`, not a false bug (demo 05).
- A broken first generator is repaired by the self-repair loop (demo 02).
- A sample invented by the LLM is dropped.
- An untrusted program cannot overwrite another program or the recorded outputs.

## Measured results

| Area | Result |
|---|---|
| Backend build | `mvnw verify` (2026-10-03): 97 unit and 22 integration tests passing |
| All 7 demos | expected verdicts, shrink sizes, tags and fix status reached through the REST API |
| Real local LLM | `qwen2.5-coder:7b` on a CPU: demo 07 found the `SOLUTION_BUG`, explained it and verified the fix in 414 s |
| Free hosting | Render + Neon + Netlify: health `UP` with all toolchains; demo 01 found and shrank the overflow in ~4 minutes under 512 MB |
| Hosted sessions | 3 real sessions, 3 bugs found (overflow, reset-missing, edge-case), 2 fixes verified, 1 correctly reported as *not verified* |

## Not verified (stated honestly)

- The Linux-server path (`install.sh`, Caddy and Let's Encrypt) on a real server: syntax-checked only.
- The NVIDIA GPU overlay: no GPU was available.
- The Anthropic provider against the real API.
