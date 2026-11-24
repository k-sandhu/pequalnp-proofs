# Local Project Instructions — P vs NP Attack (`pequalnp`)

## Contributor Credit & Git Identity
- **GitHub handle:** `k-sandhu`
- **Git commit email (repo-local, respects GitHub email privacy):** `23510890+k-sandhu@users.noreply.github.com`
- **PR description credit line** (include verbatim in every pull request body to `open-goat/pequalnp-proofs`):

```markdown
**LinkedIn profile**: https://www.linkedin.com/in/kamal-sandhu-mba-78a390a1/
```

## Proof & Submission Checklist
1. Claim a gap from `https://pequalnp.com/desk/next?handle=k-sandhu` (add `&another=1` when running parallel worktrees, up to 5 leases), or monitor fresh drops via cache-busted `https://data.pequalnp.com/data/desk/gaps.json?t=<unix_ts>` (send a `User-Agent` header).
2. Renew active leases every 10 minutes via `POST https://pequalnp.com/desk/beat?lease=<lease_id>` (a lease expires 20 minutes after its last renewal, 6 hours max in total; a `410` means the gap was settled or withdrawn — stop work at once).
3. Work on a clean branch/worktree based on latest `upstream/main` so the PR adds **exactly one new file** (`lean/PEqualNP/Outside/L_<gap>.lean`, plain mode `100644`, status `A` — never modify an existing file) and changes nothing else.
4. Follow `pequalnp-proofs/CONTRIBUTING.md` and the Own-Namespace guard rules:
   - Exact verbatim statement match; allowed imports only at top of file (prefer `import PEqualNP.Defs` only when permitted).
   - Put **every** helper (`def`, `lemma`, `theorem`, `structure`, `inductive`, `instance`, `section`, `mutual`) inside `namespace PEqualNP.Outside.L_<gap> ... end PEqualNP.Outside.L_<gap>`, with matching explicit `end PEqualNP.Outside.L_<gap>`.
   - Place `theorem PEqualNP.Outside.L_<gap>` **last** at top level, outside the namespace.
   - File size ≤ 512 KiB; no `"`, no `«»`, no `sorry`, no `admit`, no `native_decide`, no extra axioms, no `add_` lines.
5. Verify locally before pushing (automated by `submit_proof.py`):
   - `node .github/scripts/outside-guard.mjs imports <file> <imports.json> lean/PEqualNP/Outside/L_<gap>.lean`
   - `lake build <module>`
   - `lake env .lake/build/bin/checklemma.exe <module> <statement.txt> check`
6. Push branch to `origin`, open PR against `open-goat/pequalnp-proofs` (`main`) with the LinkedIn credit line above, and notify the desk via `POST https://pequalnp.com/desk/release?lease=<lease_id>&reason=submitted`.
   - **Repo canonical name:** Always query/target `open-goat/pequalnp-proofs` (the repo moved from `brilliant-hq/pequalnp-proofs`, and GitHub Search API returns 422 / 0 results on the old name).
   - **Reduction (`R_`) 5-slot cap:** Each contributor may hold at most **5 open reductions** simultaneously. Additional `R_` PRs pass the checker and are then auto-closed until one of your existing 5 reductions closes (all its sub-steps checked) or is withdrawn. Prioritize direct `L_` proofs (uncapped) whenever all 5 reduction slots are full.

---

## High-Merge Targeting Playbook (Lessons Learned from PRs `#1`–`#3754`)

Almost no PR fails on Lean math (`< 1%` `REJECTED`). Nearly **90%** of closed PRs fail due to **losing a duplicate race**, **guard refusals**, **stood-down gaps**, or **20-minute CI timeouts**. Every agent MUST follow these rules:

### 1. Strict 0-Live-PR Race Gate & Dead-Zombie Overtakes
- **Double live pre-flight check:** Check live GitHub PRs (both open and merged) via `pick_targets.py --check <GAP_ID>` **before starting work** AND **again immediately before pushing** (`submit_proof.py` enforces this automatically). Do not rely on `gaps.json` alone — it lags GitHub merges by 1–3 minutes.
- **Never submit into a live race:** Merge rate collapses from `~75–100%` at `0` earlier live PRs to `~9%` at `1`, `1.8%` at `2`, and **`0.0%` at `3+`**.
- **Distinguish live PRs from dead zombies:** An open PR whose latest bot comment contains `REFUSED`, `REJECTED`, or `FAULT` (e.g., `exit 124`) is **dead** (~19% of open PRs; accounts for 87.5% of all successful overtakes). Only count *live* (unchecked or `ACCEPTED`) open PRs as blockers; immediately overtake gaps whose only open PRs are dead.
- **Pause on race loss, keep draft on standby:** If a live external PR opens on a gap while a prover is mid-flight, stop spending CPU immediately, save the draft under `d:\projects\pequalnp\drafts\L_<gap>.lean`, and monitor for an overtake in case the competing PR fails CI.
- **Never force-push a valid 1st-in-line PR:** CI dispatches out-of-order during congestion; force-pushing resets your queue position.

### 2. Gap Selection, Weight Tiers & Dependency Chaining
- **Never touch `weight = 1`:** These are stood-down backlog steps that auto-close in seconds.
- **Inspect `context` (`SKETCH:` + `Size:`), never raw `weight` alone:** Weight semantics shift across waves (e.g., active `SKETCH:` routes moved from `w=15–30` into `w=60–95`). Skip unsketched open conjectures (`Hard and open`, `conjecture`, `no sketch`).
- **Avoid crowded `≤ 20 min` / `< 200`-line drops** unless you can land first within ~90 seconds (30% of multi-PR gaps get a 2nd PR within 60s, 68% within 5 min; `0–50`-line PRs merge only `13%`).
- **Prioritize the `45–400 min` `SKETCH:` sweet spot (`600–2,500+` lines):** Substantial proofs filter out fast-shot bots and merge at `36–49%` overall (`73–81%` when first-mover). Non-`meta` families (`pcx`, tractable `gct`) also have much lower competition.
- **Chain dependent lemmas immediately (highest alpha):** Whenever you prove a foundational lemma (e.g., `FL14 → FL18 → FL19`, `AS21 → AS22`, `K1/K2/K3 → K4`), immediately assign the same prover (with its verified Lean file in context) to its downstream dependent before other bots solve the base step.
- **Decide paired `Probe:` / `Search:` twins computationally first:** Spend `< 30s` in Python/SAT/counterexample search to confirm which twin is true before formalizing anything in Lean.

### 3. Build Speed & CI Timeout Prevention (`FAULT exit 124`)
- **Enforce a `≤ 2 min` local single-threaded build budget (`lake env lean -j 1`):** CI runners are slower and enforce a hard 20-minute `exit 124` timeout (`FAULT`s surged 9× on heavy proofs). If a proof takes `> 3–4 min` locally, optimize it (unboxed `BitVec`/`UInt64`, tighter search/BST, avoid heavy `decide` over large domains) before submitting.
- **Keep `pequalnp-proofs` synced and `PEqualNP.Defs` pre-warmed:** Pre-building `PEqualNP.Defs` on latest `upstream/main` cuts `submit_proof.py` end-to-end time from ~2 min to ~20–30s, which decides close wave races.

### 4. Multi-Agent, Workspace & Windows Tooling Hygiene
- **Single-writer rule on `pequalnp-proofs`:** Only the coordinator runs `git` write operations, `lake build`, or `submit_proof.py` inside `d:\projects\pequalnp\pequalnp-proofs` — and **never** runs two `submit_proof.py` calls concurrently.
- **Subagent sandbox convention:** Assign strictly disjoint gap IDs. Subagents write drafts only to `d:\projects\pequalnp\drafts\L_<gap>.lean` and typecheck read-only from `d:\projects\pequalnp\pequalnp-proofs\lean` via:
  ```powershell
  $env:PATH = "$env:USERPROFILE\.elan\bin;$env:PATH"
  lake env lean d:\projects\pequalnp\drafts\L_<gap>.lean
  ```
- **PowerShell quoting:** Do not use escaped `\"` inside double-quoted `python -c "..."` one-liners in PowerShell; write a small `.py` script instead.
