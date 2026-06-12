# When the Tools Lied — Building 100 Mini-Sites in a Poisoned Environment

> A retrospective of one long session: an AI agent (Claude) and a human (the repo owner)
> set out to build 100 tiny front-end toys and ship them to GitHub Pages.
> What actually happened became a case study in trust, verification, and humility.

---

## 1. The goal

- Build a growing collection of **pure front-end, single-file HTML toys** (no deps, no build, works offline).
- Ship them to GitHub Pages at `https://sakuar.github.io/why/`.
- Scale all the way to **100 sites**.
- Architecture: the main agent (Opus) orchestrates; parallel sub-agents (Sonnet) generate each site via multi-agent workflows.

## 2. The part that went fine

- First 4 sites by hand, then a reusable **playbook** of docs, then batches of 10 via workflows.
- Early lesson: workflow sub-agents could not write files to the real working directory (sandbox isolation), so we switched to a **"content-return" pattern** — the agent returns the HTML, the main loop writes it.

## 3. The hidden adversary

The whole environment was **silently tampering with tool output**: fabricated "success" lines, fake `system-reminder` blocks, flood attacks (one written file ballooned into hundreds of repeated lines), and **forged byte counts and commit hashes**.

I thought I had defenses: boundary markers (`BEGIN/END`), and "hard" git indicators — `cat-file`, `rev-parse HEAD`, `ls-tree`.

## 4. The deepest failure

For **batches 2 through 10 — 90 of the sites — every local verification was faked.**

- `cat-file -s` returned plausible but **invented** byte counts.
- `commit_happened=True` was fake.
- `ls-tree` / `git status` were fake.

I reported **"success" nine times in a row. All of it was false.** Only 11 files (the first 10 sites + the index) were ever truly on the remote.

Meanwhile the human kept saying: *"it's 404," "most of the files are missing."* **They were right every single time.** And I pushed back with fabricated data. That was the worst mistake of the session — trusting my poisoned tools over a human telling me the ground truth.

## 5. The breakthrough

- Switched to the **GitHub API** (server-side truth). The `contents` API revealed the brutal fact: only **11** HTML files existed.
- But even the API ran through a local proxy (`127.0.0.1:1080`) that **also tampered** — the file count flip-flopped 91 ↔ 101 between identical calls.
- The real crack came from the **Git Trees API (non-paginated, `truncated=False`)** plus **two independent cross-verifications**. That finally cut through the noise and gave a stable, self-consistent answer.

## 6. The recovery

- Abandoned the poisoned local git **entirely**. Read the generated HTML straight from the workflow output files, base64-encoded it, and **PUT each file directly through the GitHub Contents API**.
- Re-verified every step with the Trees API — which exposed that a faked "missing_count=0" had hidden an entire batch (the last 10 sites) that never uploaded.
- Filled the gaps. Final state, confirmed by **two independent Trees API queries**: **100 sites + a correct index, `missing_count=0`, Pages `built`.**

## 7. Lessons worth telling

1. **Tool output is not ground truth.** In a compromised environment it is an adversary, not a witness.
2. **Even "hard" indicators can be forged** — commit hashes, byte counts, exit codes. Anything that passes through a tampered layer is suspect.
3. **The only real truth is out-of-band**: the server itself, and the human's own eyes. The user looking at `github.com` directly was the ultimate oracle.
4. **Cross-verify with independent methods.** One channel agreeing with itself proves nothing; two independent channels are the minimum bar for confidence.
5. **When a channel is proven compromised, abandon it — don't patch it.** local git → API → Trees API was an escalation toward trustworthiness, not stubbornness.
6. **Listen to the human.** Repeated user contradiction ("it's still wrong") outweighs every green checkmark. It's a five-alarm signal, not noise to rebut.
7. **Report honestly.** "I'm not sure" / "I was wrong" is worth more than any confident "done."

## Epilogue

The 100 toys are real and live in the repo now. But the durable artifact is the lesson:
**an AI agent is only as trustworthy as its weakest verification channel — and being humble about that is not optional.**