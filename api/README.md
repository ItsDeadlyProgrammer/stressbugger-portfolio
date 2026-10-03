# API Quick Reference

A summary of the StressBugger REST API. For request and response bodies, status codes and the event stream, see the
[full API documentation](../docs/api-documentation.md), or browse the live
[Swagger UI](https://stressbugger-api.onrender.com/swagger-ui.html).

## Sessions
* `POST /api/sessions`: create a stress session (starts it unless `autoStart=false`).
* `GET /api/sessions`: list sessions, newest first.
* `GET /api/sessions/{id}`: one session with settings, summary, programs and runs.
* `POST /api/sessions/{id}/start`: start or restart a session.
* `POST /api/sessions/{id}/cancel`: cancel a running session.
* `POST /api/sessions/{id}/artifacts`: add your own generator, validator, checker, brute force or property checker.

## Results
* `GET /api/sessions/{id}/counterexamples`: shrunk counterexamples with every output, the explanation and the fix.
* `GET /api/sessions/{id}/mismatches`: every disagreement with its verdict, confidence and evidence.
* `GET /api/sessions/{sid}/counterexamples/{cid}/similar`: similar past bugs (pgvector, or tag overlap).
* `GET /api/stats/bugs`: bug statistics by tag, verdict and day.

## Live events
* `GET /api/sessions/{id}/events`: Server-Sent Events; stored events are replayed first.
* `GET /api/sessions/{id}/history`: stored events as JSON.

## Problems
* `GET /api/codeforces/import?problem=1985A`: import a Codeforces problem by id or URL.
* `GET /api/demos`, `GET /api/demos/{slug}`: the 7 bundled demo problems.

## System
* `GET /api/health?toolchains=true`: readiness of every component, compiling a program in each language.
