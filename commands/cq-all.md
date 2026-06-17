---
description: Full CQ pipeline end-to-end — clear prior reports (preserving Purpose files), ensure purpose baselines, run the per-unit analysis, produce cross-solution summaries, then generate ten reviewable implementation plans. Delegates to /cq-purpose, /cq-analyze, /cq-summary, and /cq-plan.
argument-hint: [subdir1 subdir2 ...]
---

# /cq-all — full CQ review → ten implementation plans pipeline

End-to-end orchestrator. Runs every stage of the workflow in order and stops at reviewable plans — **no production code is changed**. Each stage is a real command you can also run on its own (`/cq-purpose`, `/cq-analyze`, `/cq-summary`, `/cq-plan`); this command just chains them.

> Prefer running the stages individually when you want to **fill in each `## Scale signals` table** between purpose generation and analysis (that calibration is what makes the reviews sharp). `/cq-all` will generate any missing purpose file and then analyze immediately — so on a first run it analyzes against blank Scale-signals tables. Use `/cq-all` for re-runs (where the Purpose files are already curated and preserved), or when you deliberately want an un-calibrated first pass.

## Arguments

`$ARGUMENTS` — optional space-separated list of subdirectories, forwarded verbatim to `/cq-purpose`, `/cq-analyze`, and (as the working dir) the later stages. If empty, operates on the current working directory.

## Steps

### 1. Clear the target directory (preserving Purpose files)

- Confirm `CQ-Reviews/` exists at the cwd. If yes, delete everything under it **except** the per-solution `solutions/<Solution>/Purpose.md` files (curated baselines — expensive to regenerate, rarely stale) and the `scripts/` folder (source, not output).
  - On bash: `Bash find CQ-Reviews -mindepth 1 -type f ! -path 'CQ-Reviews/scripts/*' ! -name 'Purpose.md' -delete; find CQ-Reviews -mindepth 1 -type d -empty -delete`
- If `CQ-Reviews/` does not exist, create it with `Bash mkdir -p CQ-Reviews`.
- Verify only `solutions/*/Purpose.md` files (if any) and `scripts/*` remain (`Bash find CQ-Reviews -type f`).

### 2. Ensure purpose baselines — `/cq-purpose`

Invoke the `/cq-purpose` workflow with the `Skill` tool (`skill: "cq-purpose"`, `args: "$ARGUMENTS"`). It generates a `solutions/<Solution-Name>/Purpose.md` for every solution whose file is missing (existing ones are preserved) and reports each Scale-signals disposition. Wait for it to finish and verify every expected purpose file now exists before continuing; if any is missing, stop.

### 3. Run the analysis — `/cq-analyze`

Invoke `/cq-analyze` with the `Skill` tool (`skill: "cq-analyze"`, `args: "$ARGUMENTS"`). Follow it to completion: discover solutions, classify projects, dispatch the CQ review agents in parallel batches, and wait for **every** spawned agent to return. After it returns, `CQ-Reviews/` should contain per-solution `solutions/<Solution>/Architect.md` (+ `Frontend.md` where applicable) and per-project `projects/<Project>/{CodeReview,Data,TestReview}.md` reports.

### 4. Summarise — `/cq-summary`

Invoke `/cq-summary` with the `Skill` tool (`skill: "cq-summary"`, `args: "$ARGUMENTS"`). It dispatches `CQ-Summary` to write the five `summaries/CQ-Summary-<Domain>.md` files plus the top-level `summaries/CQ-Summary.md`. Wait for it to finish and verify the six files exist before continuing.

### 5. Plan — `/cq-plan`

Invoke `/cq-plan` with the `Skill` tool (`skill: "cq-plan"`, `args: "$ARGUMENTS"`). Its preconditions (analysis files + summaries) are now satisfied by Steps 3–4. It reads every per-unit report and summary and writes the ten `plans/IMPLEMENTATION-PLAN-{DETAILED,SUMMARY}-<Kind>.md` files, then hands off for review.

### 6. Hand off for review

Surface the consolidated result: the ten plan paths grouped by kind, the per-kind finding/theme/task counts, and the top findings by severity. Do **not** begin implementation — wait for the user's explicit go-ahead.

## Constraints

- **Read-only across the production source tree.** The only writes are: the Step 1 deletion of prior reports (Purpose files and `scripts/` preserved), and the report / summary / plan files produced by the delegated stage commands.
- Do not invoke `CQ-Management-Summary` or `CQ-HTML-Publisher` here — those are deliberately separate workflows the user runs after the summaries exist (`/cq-management-summary`, or dispatch `CQ-HTML-Publisher`).
- The plans are proposals: every task must be reversible until the user approves execution. Do not start editing code.
