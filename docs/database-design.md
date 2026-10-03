# Database Design

**PostgreSQL 16** is the single source of truth. The schema is managed by **Flyway** migrations and accessed through
Spring Data JPA. Flexible payloads (settings, outputs, reasoning, shrink steps) are stored as `jsonb`; everything that
is queried or joined is a typed column.

Redis holds only the job queue, cancel flags and the LLM cache, so losing it loses no results.

## Entity-relationship diagram

```mermaid
erDiagram
    PROBLEMS ||--o{ SESSIONS : "tested in"
    SESSIONS ||--o{ ARTIFACTS : "programs"
    SESSIONS ||--o{ RUNS : "stress batches"
    RUNS ||--o{ MISMATCHES : "found"
    SESSIONS ||--o{ MISMATCHES : ""
    MISMATCHES ||--o| COUNTEREXAMPLES : "shrunk to"
    SESSIONS ||--o{ COUNTEREXAMPLES : ""
    COUNTEREXAMPLES ||--o{ EXPLANATIONS : "explained by"
    EXPLANATIONS ||--o{ BUG_TAGS : "tagged"
    SESSIONS ||--o{ LLM_CALLS : "logged"
    SESSIONS ||--o{ SESSION_EVENTS : "live log"

    PROBLEMS {
        bigint id PK
        varchar title
        text statement
        boolean interactive
        boolean multi_test
        jsonb constraints
        jsonb samples
        jsonb format_spec
    }
    SESSIONS {
        bigint id PK
        bigint problem_id FK
        varchar mode "STRESS or HACK"
        varchar status
        varchar phase
        jsonb settings
        jsonb summary
        timestamptz started_at
        timestamptz finished_at
    }
    ARTIFACTS {
        bigint id PK
        bigint session_id FK
        varchar kind "SOLUTION, BRUTE, LLM_BRUTE, GENERATOR, VALIDATOR, CHECKER, PROPERTY, FIX"
        varchar language
        text source
        varchar origin "USER or LLM"
        varchar status
        double trust_score
        int attempts
        jsonb stats
    }
    RUNS {
        bigint id PK
        bigint session_id FK
        varchar strategy
        varchar tier "SMALL or LARGE"
        jsonb gen_args
        bigint seed_start
        int tests
        int mismatches
        bigint elapsed_ms
    }
    MISMATCHES {
        bigint id PK
        bigint session_id FK
        bigint run_id FK
        bigint seed
        text input
        jsonb outputs
        varchar verdict
        double confidence
        jsonb reasoning
    }
    COUNTEREXAMPLES {
        bigint id PK
        bigint session_id FK
        bigint mismatch_id FK
        varchar verdict
        text input
        int original_bytes
        int shrunk_bytes
        jsonb shrink_steps
        jsonb outputs
    }
    EXPLANATIONS {
        bigint id PK
        bigint counterexample_id FK
        text summary
        text root_cause
        jsonb cited_lines
        text fix_source
        varchar fix_status "NONE, VERIFIED, FAILED, UNVERIFIED"
        vector embedding "256 dims, pgvector"
    }
    BUG_TAGS {
        bigint id PK
        bigint explanation_id FK
        varchar tag
    }
    LLM_CALLS {
        bigint id PK
        bigint session_id FK
        varchar template_id
        int template_version
        varchar model
        int input_tokens
        int output_tokens
        bigint latency_ms
        boolean cache_hit
    }
    SESSION_EVENTS {
        bigint id PK
        bigint session_id FK
        int seq
        varchar type
        jsonb payload
    }
```

## Tables

| Table | Content | Notes |
|---|---|---|
| `problems` | statement, constraints, samples, input format model | samples are copied verbatim from the statement |
| `sessions` | mode, status, phase, settings, summary, timings | the summary records toolchain versions, model, prompt versions and limits |
| `artifacts` | every program: solution, brute forces, generator, validator, checker, property checkers, fixes | status, trust score and number of LLM attempts |
| `runs` | one stress batch: strategy, tier, generator arguments, seed range, counts, timing | makes every test reproducible |
| `mismatches` | every disagreement with all outputs, its verdict, confidence and reasoning | inputs over 64 KiB are stored as a preview plus the command that regenerates them |
| `counterexamples` | shrunk input, every shrink step, re-verified outputs | `original_bytes → shrunk_bytes` |
| `explanations` | root cause, cited lines, fix, fix status and verification details | optional `embedding vector(256)` |
| `bug_tags` | tags per explanation | drives the bug statistics page |
| `llm_calls` | every LLM call: template, version, model, tokens, latency, cost, cache hit, errors | observability and cost tracking |
| `session_events` | the live event log, `UNIQUE (session_id, seq)` | replayed to dashboards opened late |

All child tables use `ON DELETE CASCADE` from `sessions`, so deleting a session removes its evidence cleanly.

## Design choices

- **Plain ids, no JPA associations.** Entities hold numeric foreign-key columns instead of mapped relationships, so
  the long-running pipeline never hits lazy-loading problems outside a transaction.
- **`jsonb` for evidence.** Outputs, reasoning and shrink steps vary by verdict; storing them as JSON keeps the full
  evidence without a dozen side tables, and the API returns them as they are.
- **Optional pgvector.** The `embedding` column is added by a separate migration that runs only when pgvector is
  enabled. Without it, similar bugs fall back to tag overlap.
- **Deterministic embeddings.** Similar-bug search uses a 256-dimension feature-hashing embedding computed in Java, so
  it works offline and with any LLM, with no embedding API.
- **Indexes** on `session_id` for every child table, `sessions (created_at DESC)` for the history page and
  `bug_tags (tag, created_at)` for statistics.
