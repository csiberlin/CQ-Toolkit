---
description: Produce the cross-solution CQ summaries — five per-domain summaries (Architecture, Frontend, CodeReview, Data, TestReview) plus a top-level brief — from the per-unit analysis files written by /cq-analyze, via the CQ-Summary agent. Run after /cq-analyze; required before /cq-management-summary and /cq-plan.
argument-hint: [working-subdir]
---

# /cq-summary — cross-solution per-domain summaries

**Stage 3a** of the workflow (`/cq-purpose` → `/cq-analyze` → **`/cq-summary`** → `/cq-plan`). Rolls the per-unit analysis files up into one summary per domain plus a top-level cross-domain brief. These summaries are the **prerequisite** for both `/cq-management-summary` (exec briefs by quality attribute) and `/cq-plan` (the SUMMARY implementation plans).

## Arguments

`$ARGUMENTS` — optional subdirectory to treat as the working directory. Default to the current directory. The reports live under `<working-directory>\CQ-Reviews\`.

## Preconditions

- The per-unit analysis files from `/cq-analyze` must already exist under `<working-directory>\CQ-Reviews\` — `solutions/<Solution>/Architect.md` (and `Frontend.md` where present) and `projects/<Project>/{CodeReview,Data,TestReview}.md`. `Glob` for them first; if none are present, **stop** and tell the user to run `/cq-analyze` (or `/cq-all`) before summarising. Do not regenerate the analysis here.
- Read-only except for the summary files the agent writes into `summaries\`.

## Steps

1. **Resolve the working directory.** Default to cwd; if `$ARGUMENTS` names a subdirectory, treat it as the working directory.

2. **Confirm inputs exist.** `Glob` `CQ-Reviews/solutions/*/Architect.md`, `CQ-Reviews/solutions/*/Frontend.md`, and `CQ-Reviews/projects/*/{CodeReview,Data,TestReview}.md`. If the analysis set is empty, stop per Preconditions.

3. **Dispatch the agent.** Dispatch a single `CQ-Summary` agent via the `Agent` tool. Brief it to produce **all five** per-domain summaries plus the top-level cross-domain summary, **using the file naming `CQ-Summary-<Domain>.md` and `CQ-Summary.md`** (NOT the agent's default `CQ-<Domain>-Summary.md` — be explicit; the agent honours an explicit override). All files land in `CQ-Reviews/summaries/`:
   - `summaries/CQ-Summary-Architecture.md` (from `solutions/*/Architect.md`)
   - `summaries/CQ-Summary-Frontend.md` (from `solutions/*/Frontend.md`)
   - `summaries/CQ-Summary-CodeReview.md` (from `projects/*/CodeReview.md`)
   - `summaries/CQ-Summary-Data.md` (from `projects/*/Data.md` — produce even when only one project has findings; the others are 0-finding attestations)
   - `summaries/CQ-Summary-TestReview.md` (from `projects/*/TestReview.md`)
   - `summaries/CQ-Summary.md` (top-level cross-domain summary consuming all five)
   - Wait for the agent to return before proceeding.

4. **Verify & report.** `Glob CQ-Reviews/summaries/*.md` and confirm all six files are present. Report the six absolute paths and, for each per-domain summary, the count of promoted themes. If any file is missing, surface the error and stop — do not claim success without the `Glob` proof. Point the user at the next steps: `/cq-management-summary` for executive briefs, `/cq-plan` for implementation plans.

## Constraints

- Consumes the existing per-unit analysis files only — never re-run `/cq-analyze` or regenerate them here.
- Writes only the six summary files under `CQ-Reviews\summaries\`. No production source files are modified.
- The summary naming is the `/cq-plan`-compatible `CQ-Summary-<Domain>.md` convention so the downstream plan and management-summary commands find the files regardless of producer.
