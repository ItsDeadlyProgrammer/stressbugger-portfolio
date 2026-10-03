# Trust & Triage

A mismatch between your solution and a brute force only means that *someone* is wrong. StressBugger never assumes the
brute force is right:

- every program must **prove itself by execution** before it is trusted;
- every disagreement is **classified from evidence**, with a confidence value and its reasoning;
- the LLM writes programs and explanations, but **never decides a verdict**.

![Pipeline and trust gates](../architecture/pipeline.png)

## 1. Qualification gates

Every program passes these checks before any stress test.

| Gate | Passes when | On failure |
|---|---|---|
| **Parse** | samples are read exactly from the statement; any sample the LLM proposes must appear verbatim in it | invented samples are dropped with a warning |
| **Compile** | the program compiles in the sandbox | LLM programs get the compiler error back (up to 3 attempts); a solution that does not compile fails the session |
| **Validator** | it accepts every sample and **rejects broken inputs** (empty, a word instead of a number, out-of-range values) | repaired by the LLM; without a validator, confidence is lowered |
| **Generator** | each strategy runs on its own (25 seeds for small strategies) and prints valid, varied inputs | bad strategies are dropped with a warning; rejected only if no useful small strategy is left |
| **Checker** (for "print any" problems) | it accepts the official answers and rejects corrupted outputs | repaired; without a checker, differing outputs are reported as `MULTIPLE_VALID_ANSWERS` with low confidence |
| **Samples** | the solution and every brute force produce the sample answers | your brute force: **untrusted**; LLM brute force: repaired; your solution: already a `SOLUTION_BUG` (0.99) |

## 2. Several independent oracles

- **No brute force from you:** the LLM writes 2 independent ones, with different approaches and languages.
- **Your brute force given:** the LLM still writes 1 more to cross-check it.
- **Cross-check:** every oracle runs on ~150 small inputs. The largest agreeing group is the majority. On a tie, the
  group that matches your solution wins (your solution is evidence, never the judge).
- **Trust score** = 0.4 × (samples passed) + 0.6 × (agreement with the majority).
  - **Trusted:** passes every sample and agrees on at least 95% of the inputs.
  - **Verified:** trusted, and agreed on 100% of at least 20 inputs.
- **Reference:** the best trusted oracle is what stress tests compare against. The others still run on every
  disputed input.
- **Property checkers:** for large inputs, where no brute force can run, the LLM may propose up to 3 property checkers
  (for example "the product equals the two largest values multiplied"). The first one verified against the trusted
  oracle on small inputs is used.

## 3. Verdicts

For every disagreement, every oracle runs on the input, and the solution and the reference each run twice.

| Verdict | Meaning | Confidence |
|---|---|---|
| `INVALID_INPUT` | the validator rejects the input (a generator bug; the input is discarded) | 0.99 |
| `NONDETERMINISTIC` | the solution (or reference) gives different outputs on two runs | 0.95 (0.9 for the reference) |
| `SOLUTION_TLE` / `SOLUTION_RE` / `SOLUTION_MLE` | the solution hits a time, crash or memory limit | 0.9 TLE, 0.95 RE/MLE; 0.7 without a validator; ≤ 0.5 if the oracles fail too |
| `REFERENCE_TLE` / `REFERENCE_RE` | the reference is too slow or crashes, the solution finishes | 0.9 |
| `MULTIPLE_VALID_ANSWERS` | outputs differ but the checker accepts both | 0.95; 0.4 without a checker |
| `BRUTE_BUG` | your brute force disagrees, all other trusted oracles agree with your solution | 0.95 with ≥ 2 agreeing oracles, 0.8 with 1 |
| `SOLUTION_BUG` | all trusted oracles agree with each other and disagree with your solution | see below |
| `ORACLE_DISAGREEMENT` | the trusted oracles disagree among themselves | 0.3, all outputs shown for you to decide |

`SOLUTION_BUG` confidence:

- **0.95** with ≥ 2 agreeing trusted oracles, or 1 verified oracle;
- **0.6** with 1 unverified oracle, **0.5** if only untrusted oracles agree;
- capped at **0.75** without a validator;
- **0.8** for a property violation on a large input (property checkers are first verified on small inputs).

A high-confidence `SOLUTION_BUG` is therefore **never claimed on the word of a single unverified brute force**. This
rule is covered by unit tests of the triage engine.

## 4. What happens next

- Findings that blame **your solution** are shrunk and explained, one per verdict type (up to `maxCounterexamples`).
- A **`BRUTE_BUG`** is shrunk and explained too, pointing at your brute force.
- **Disagreements, multiple valid answers and reference failures** are stored with all outputs for review, but never
  shown as bugs in your solution.

## 5. A "verified" fix

The LLM's proposed fix is labelled **verified** only if it:

1. compiles,
2. gives the trusted answer on the counterexample,
3. passes every official sample,
4. passes every earlier failing input,
5. survives a fresh stress run of 300 tests.

Otherwise the dashboard shows **"fix not verified"** with the check that failed. The screenshot below is a real case
from the hosted demo: the fix for an overflow bug passed the counterexample but failed 1 of 6 earlier failing tests, so
it was not called verified.

![Fix not verified](../screenshots/counterexample-overflow.png)
