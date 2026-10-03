# Demo Scenarios

Seven ready-made problems ship with StressBugger. Each is a complete, reproducible session:

- pick one in the dashboard with **New session → Load a demo problem…**;
- they run fully offline with the mock LLM, which replays recorded answers;
- they are the backend's end-to-end tests, run in CI on every push.

Each demo was chosen to prove one property of the system, including the cases where a naive stress tester would
report a **false bug**.

| Demo | The situation | Expected result | What it proves |
|---|---|---|---|
| `01-max-pair-product` | `int * int` overflow | `SOLUTION_BUG` (overflow), shrunk to 2 numbers, verified fix | the overflow strategy and the shrinker |
| `02-range-sum` | prefix-sum loop stops at `n - 1`; the first generated generator is broken | `SOLUTION_BUG` (off-by-one) | self-repair of a broken LLM program |
| `03-distinct-multitest` | global array not reset between test cases | `SOLUTION_BUG` (reset-missing) | the multitest strategy |
| `04-wrong-brute` | the solution is right, **your brute force is wrong** | `BRUTE_BUG`, never `SOLUTION_BUG` | independent oracles out-vote a wrong reference |
| `05-print-any` | several valid answers | `MULTIPLE_VALID_ANSWERS`, no false bug | the checker gate |
| `06-quadratic-tle` | correct but O(n²) | `SOLUTION_TLE`, complexity ≈ n² | the TLE hunter and complexity fit |
| `07-find-maximum` | running maximum starts at 0 (wrong for negative arrays); no brute force given | `SOLUTION_BUG`, shrunk to one number, verified fix | two LLM brute forces with nothing from the user |

## Real sessions on the hosted demo (Gemini)

| Problem | Bug | Minimal input | Fix |
|---|---|---|---|
| Max Pair Product | `a[n-1] * a[n-2]` multiplied as 32-bit `int` | `2` / `7 499999997` (shrunk from 32 to 14 bytes) | not verified: the fix failed 1 of 6 earlier failing tests |
| Distinct Values in Every Test | global `seen` array not reset between test cases | 3 test cases (shrunk from 22 to 18 bytes) | verified |
| Watermelon (Codeforces 4A) | `w = 2` answered `YES` | `2` | verified (`n % 2 == 0 && n > 2`) |

See the [screenshots](../screenshots/README.md) for the full evidence of each.
