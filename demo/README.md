# 🎬 Product Walkthrough & Live Demo

---

## 🌐 Live Deployments

*   **Dashboard:** [https://stressbugger.netlify.app](https://stressbugger.netlify.app)
    *   *Architecture note:* a static React 18 + TypeScript build served by Netlify. It talks to the API over REST and
        follows each session over Server-Sent Events.
*   **Backend API:** [https://stressbugger-api.onrender.com/swagger-ui.html](https://stressbugger-api.onrender.com/swagger-ui.html)
    *   *Architecture note:* the Spring Boot service, Redis and the sandbox harness in one container on Render's free
        tier, with PostgreSQL + pgvector on Neon and Gemini as the LLM.
*   **Health:** [https://stressbugger-api.onrender.com/api/health?toolchains=true](https://stressbugger-api.onrender.com/api/health?toolchains=true)

> Free tiers: the first request after a quiet period takes about a minute while the server wakes up.

---

## 📺 Walkthrough

<p align="center">
  <img src="./StressBugger.gif" width="100%" alt="StressBugger walkthrough">
</p>

[**Download / view the video (MP4) →**](./StressBugger.mp4)

---

## 📝 Try it yourself

1.  Open the [dashboard](https://stressbugger.netlify.app) and go to **New session**.
2.  Type a Codeforces id such as `4A` and press **Import**, or pick **Load a demo problem…**.
3.  Paste a solution with a bug. For `4A`, this one is wrong for `w = 2`:

    ```cpp
    #include <bits/stdc++.h>
    using namespace std;
    int main() { int w; cin >> w; cout << (w % 2 == 0 ? "YES" : "NO") << "\n"; }
    ```

4.  Press **Start stress testing** and watch the phases. In a few minutes you get the minimal input `2`, the output of
    every program, an explanation and a verified fix.

---

## 🔍 What to look for

1.  **Generate:** the LLM writes a generator, a validator and two independent brute forces in different languages.
2.  **Qualify:** each program has to compile, pass the samples and agree with the others before it is trusted (see the
    *Programs* table and trust scores).
3.  **Stress:** test batches per strategy (`exhaustive`, `boundary`, `overflow`, `random`, `tle`…) with live counters.
4.  **Triage:** every disagreement gets a verdict and a confidence, with the reasoning.
5.  **Shrink:** the failing input shrinks step by step; each step must still fail the same way.
6.  **Explain + fix:** the root cause cites lines in your code; the fix is shown as a diff and labelled *verified* only
    after the sandbox re-runs it.
